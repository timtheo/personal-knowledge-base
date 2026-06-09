# Deploy Infrastructure Workflow

**File:** `.github/workflows/deploy-infrastructure.yml`

This document describes every aspect of the **Deploy Infrastructure** GitHub Actions workflow — how to trigger it, what it needs, how it provisions and tears down Azure infrastructure with Terraform, and what your colleagues must understand before running it in any environment.

---

## Table of Contents

1. [Overview](#overview)
2. [Trigger Mechanisms](#trigger-mechanisms)
3. [Inputs Reference](#inputs-reference)
4. [Secrets Reference](#secrets-reference)
5. [Environment Variables](#environment-variables)
6. [Job Architecture](#job-architecture)
7. [Job Details](#job-details)
   - [terraform-plan](#1-terraform-plan)
   - [manual-approval](#2-manual-approval)
   - [terraform-apply](#3-terraform-apply)
   - [terraform-destroy](#4-terraform-destroy)
8. [Terraform Variables Reference](#terraform-variables-reference)
9. [Terraform State Management](#terraform-state-management)
10. [Authentication: OIDC](#authentication-oidc)
11. [Plan Exit Codes](#plan-exit-codes)
12. [Force-Replace Targets](#force-replace-targets)
13. [Protected Resources on Destroy](#protected-resources-on-destroy)
14. [Post-Apply: Terraform Outputs](#post-apply-terraform-outputs)
15. [Post-Apply: Auto-Commit](#post-apply-auto-commit)
16. [Chaining with Deploy Components](#chaining-with-deploy-components)
17. [GitHub Environment Configuration](#github-environment-configuration)
18. [Required GitHub Configuration](#required-github-configuration)
19. [Service Principal RBAC Requirements](#service-principal-rbac-requirements)
20. [Troubleshooting](#troubleshooting)

---

## Overview

This workflow provisions, updates, and tears down Azure infrastructure using **Terraform**. It is the entry point for all infrastructure changes and follows a strict plan-then-apply pattern with a mandatory human-in-the-loop approval gate before any destructive or mutating operation.

```
Trigger (manual or called)
        │
        ▼
 [terraform-plan]  ──── always runs — generates & uploads plan artifact
        │
        ▼
 [manual-approval]  ──── gates apply / destroy (skippable with flag)
        │
        ├──── [terraform-apply]    (if action == apply)
        │
        └──── [terraform-destroy]  (if action == destroy)
```

**The `plan` action stops after `terraform-plan`.** No apply or destroy job is triggered. This makes `plan` safe to run at any time, on any environment, without side effects.

**Runner:** All jobs run on `external-k8s-v2` (Kubernetes-based self-hosted runners) with Ubuntu 24.04 containers from `official-dockerhub-images.artifactory.bayer.com`.

**Terraform version:** `1.14.0` (pinned).

**Terraform working directory:** `configuration/` (relative to the repo root).

---

## Trigger Mechanisms

### `workflow_dispatch` (Manual via GitHub UI)

Navigate to **Actions → Deploy Infrastructure → Run workflow** and fill in the form. The run name is:

```
<PROJECT_NAME> Infra: <action> → <environment> (#<run_number>)
```

Example: `myapp Infra: apply → dev (#42)`

### `workflow_call` (Called from Another Workflow)

The workflow can be orchestrated by a parent pipeline (e.g., a release workflow):

```yaml
jobs:
  provision:
    uses: ./.github/workflows/deploy-infrastructure.yml
    with:
      environment: dev
      terraform_action: apply
    secrets:
      ORG_REPOS_INTERNAL_READ_ONLY: ${{ secrets.ORG_REPOS_INTERNAL_READ_ONLY }}
      APIM_SUBSCRIPTION_KEY: ${{ secrets.APIM_SUBSCRIPTION_KEY }}
      GRAFANA_OTLP_HEADERS: ${{ secrets.GRAFANA_OTLP_HEADERS }}
      GH_PAT_RUNNER: ${{ secrets.GH_PAT_RUNNER }}
```

> **Important:** Secrets are not automatically inherited by called workflows. The caller must forward each secret explicitly.

---

## Inputs Reference

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `environment` | choice / string | Yes | `sandbox` | Target Azure environment. Must match a GitHub Environment name. |
| `terraform_action` | choice / string | Yes | `plan` | One of `plan`, `apply`, or `destroy`. Controls which job runs after the plan. |
| `replace_targets` | string | No | _(empty)_ | Space-separated list of Terraform resource addresses to force-replace (adds `-replace=` flags). See [Force-Replace Targets](#force-replace-targets). |
| `deploy_components` | boolean | No | `false` | Reserved for chaining into `deploy-components.yml` after a successful apply. Currently commented out — see [Chaining with Deploy Components](#chaining-with-deploy-components). |
| `skip_manual_approval` | boolean | No | `false` | Skips the `manual-approval` job. **Use with caution** — only appropriate for automated pipelines where upstream approval already happened. |

### Available Environments

`sandbox` · `dev` · `test` · `qa` · `prod`

Each must be configured as a GitHub Environment in the repository settings.

---

## Secrets Reference

| Secret | Required | Used by | Purpose |
|---|---|---|---|
| `ORG_REPOS_INTERNAL_READ_ONLY` | **Yes** | All jobs | GitHub PAT for authenticating `git` when Terraform fetches private module sources. Rewrites `github.com` URLs to use OAuth. |
| `APIM_SUBSCRIPTION_KEY` | No | Plan, Apply, Destroy | Passed as `TF_VAR_azure_apim_sub_key` — the APIM subscription key for the OpenAI gateway. Required if Terraform manages APIM resources. |
| `GRAFANA_OTLP_HEADERS` | No | Plan, Apply, Destroy | Passed as `TF_VAR_grafana_otlp_headers` — authentication headers for the Grafana OTLP exporter. |
| `GH_PAT_RUNNER` | No | Plan, Apply, Destroy | Passed as `TF_VAR_github_pat` — used by Terraform if it provisions a self-hosted runner. |

> **Note:** `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are read from GitHub **Variables** (not secrets) with a secret fallback for backward compatibility. Prefer Variables for these identifiers.

---

## Environment Variables

Defined at the workflow level, available to all jobs:

| Variable | Source | Purpose |
|---|---|---|
| `AZURE_CLIENT_ID` | `vars` → `secrets` fallback | Service Principal client ID for OIDC authentication. |
| `AZURE_TENANT_ID` | `vars` → `secrets` fallback | Azure AD tenant ID. |
| `AZURE_SUBSCRIPTION_ID` | `vars` → `secrets` fallback | Target Azure subscription. |
| `AZURE_TF_BACKEND_NAME` | `vars` → `secrets` fallback | Storage account name for Terraform remote state. |
| `AZURE_TF_BACKEND_CONTAINER_NAME` | `vars` → `secrets` fallback | Blob container for Terraform state files. |
| `ENV_NAME` | `inputs.environment` → `github.ref_name` fallback | Resolved environment name, used in the Terraform state key and as a `-var` value. |
| `TERRAFORM_DIR` | hardcoded | `configuration` — Terraform root module path. |

Each job that runs Terraform also sets these ARM-prefixed variables for the Terraform Azure provider:

```
ARM_CLIENT_ID, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
ARM_USE_OIDC=true
ARM_USE_CLI=false
```

---

## Job Architecture

```
terraform-plan
  └─ outputs: plan_exitcode (0 = no changes, 2 = changes pending, 1 = error)

manual-approval
  └─ needs: terraform-plan
  └─ runs when: action is apply or destroy AND skip_manual_approval != true
  └─ environment: hitl-approvals  (must have required reviewers configured)

terraform-apply
  └─ needs: terraform-plan, manual-approval
  └─ runs when: action == apply AND plan succeeded AND approval succeeded or was skipped
  └─ outputs: frontdoor_endpoint_hostname, frontdoor_agent_urls,
              container_app_ui_url, container_app_agent_urls

terraform-destroy
  └─ needs: terraform-plan, manual-approval
  └─ runs when: action == destroy AND plan succeeded AND approval succeeded or was skipped
```

All four jobs use `always()` in the `if:` condition for apply and destroy so that a skipped `manual-approval` (when `skip_manual_approval=true`) does not block execution.

---

## Job Details

### 1. `terraform-plan`

**Always runs** regardless of the chosen action.

**Steps:**

1. **Checkout** the repository.
2. **Install packages** — `unzip`, `curl`, `git`, `jq` (no Azure CLI needed for plan).
3. **Print pipeline parameters** to the GitHub Actions step summary.
4. **Setup Terraform** (v1.14.0, wrapper disabled so exit codes pass through correctly).
5. **Configure Git** to use `ORG_REPOS_INTERNAL_READ_ONLY` for private module sources.
6. **Terraform Init** — connects to the remote backend:
   ```
   terraform init \
     -backend-config="resource_group_name=<AZURE_RESOURCE_GROUP>" \
     -backend-config="storage_account_name=<AZURE_TF_BACKEND_NAME>" \
     -backend-config="container_name=<AZURE_TF_BACKEND_CONTAINER_NAME>" \
     -backend-config="key=<PROJECT_NAME>-<ENV_NAME>.terraform.tfstate" \
     -upgrade -reconfigure
   ```
7. **Terraform Validate** — checks configuration syntax before touching state.
8. **Terraform Plan** — generates a plan with all `TF_VAR_*` variables set, optional `-replace=` flags, and `-detailed-exitcode`. The plan binary is saved as `tfplan`. See [Plan Exit Codes](#plan-exit-codes).
9. **Display plan** — runs `terraform show` and a `jq` summary (add/change/destroy counts).
10. **Upload `tfplan` artifact** — retained for 5 days. The apply job downloads this exact file, guaranteeing that what was reviewed is what gets applied.
11. **Plan Summary** — writes a human-readable result to the step summary, including next steps if changes were detected.

**Key output:** `plan_exitcode` — consumed by downstream jobs to understand the plan result.

---

### 2. `manual-approval`

**Runs when:**
- Action is `apply` or `destroy`, **AND**
- `terraform-plan` succeeded, **AND**
- `skip_manual_approval` is not `true`

**Environment:** `hitl-approvals`

This job uses the [GitHub Environment Protection](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) mechanism. The `hitl-approvals` environment must be configured with **required reviewers** — the workflow pauses here and waits for a named reviewer to approve or reject in the GitHub UI.

The environment URL is set to the Azure Portal resource group overview, giving reviewers quick access to the current state of the target environment while making their approval decision.

**The job itself does nothing other than pause.** There is no shell command to approve — the approval happens entirely in the GitHub UI under the **Environments** tab of the run.

> **Rejecting the approval** cancels the entire workflow run. Neither apply nor destroy will execute.

---

### 3. `terraform-apply`

**Runs when:** action == `apply` AND plan succeeded AND approval was either granted or skipped.

**Steps:**

1. Checkout, install packages, setup Terraform, configure Git — identical to plan.
2. **Terraform Init** — re-initialises the backend (same state key as plan).
3. **Download `tfplan` artifact** — retrieves the exact plan binary produced in `terraform-plan`. This is critical: the apply runs against the reviewed plan, not a fresh one. No drift is possible.
4. **Terraform Apply** — applies the downloaded plan with `-auto-approve`:
   ```
   terraform apply -var-file="terraform.tfvars" \
     -var="env_name=<ENV>" \
     -var="product_name=<PROJECT_NAME>" \
     -auto-approve tfplan
   ```
   All `TF_VAR_*` variables are set identically to the plan step.
5. **Apply Summary** — writes timestamp and target details to the step summary.
6. **Capture Terraform Outputs** — reads all outputs from state and exposes them as job outputs and step summary rows. See [Post-Apply: Terraform Outputs](#post-apply-terraform-outputs).
7. **Commit and Push Changes** — if Terraform modified `api-gateway/config.json`, commits and pushes it back. See [Post-Apply: Auto-Commit](#post-apply-auto-commit).

**Required permission:** `contents: write` (for the auto-commit step).

---

### 4. `terraform-destroy`

**Runs when:** action == `destroy` AND plan succeeded AND approval was either granted or skipped.

> **Warning:** This permanently deletes all Terraform-managed resources in the target environment. It cannot be undone. The `hitl-approvals` gate exists specifically to prevent accidental destroy runs.

**Steps:**

1. Checkout, install packages, setup Terraform, configure Git — identical to plan and apply.
2. **Terraform Init** — same state key as plan.
3. **Remove protected resources from state** — before destroy, role assignments are removed from the Terraform state so they are not deleted. See [Protected Resources on Destroy](#protected-resources-on-destroy).
4. **Terraform Destroy** — runs `terraform destroy -auto-approve` with the same `-var` and `TF_VAR_*` set as plan/apply. **Note:** Destroy does not use a pre-generated plan file; it generates and applies a destroy plan in a single step.
5. **Destroy Summary** — writes what was destroyed (and what was preserved) to the step summary.

---

## Terraform Variables Reference

These variables are passed to every Terraform operation (plan, apply, destroy) via environment variables in the `TF_VAR_` convention:

| TF Variable | Source | Default | Purpose |
|---|---|---|---|
| `github_pat` | `secrets.GH_ORG_REPOS_INTERNAL_READ_ONLY` | — | GitHub PAT passed to Terraform (e.g., for provisioning a runner). |
| `subscription_id` | `env.AZURE_SUBSCRIPTION_ID` | — | Azure subscription to deploy into. |
| `resource_group_name` | `vars.AZURE_RESOURCE_GROUP` | — | Target resource group. |
| `env_name` | `env.ENV_NAME` | — | Environment identifier (sandbox / dev / test / qa / prod). |
| `product_name` | `vars.PROJECT_NAME` | — | Short product identifier, used in resource naming. |
| `azure_apim_base_url` | `vars.AZURE_APIM_GATEWAY_URL` | — | APIM gateway base URL for the OpenAI proxy. |
| `azure_apim_sub_key` | `secrets.APIM_SUBSCRIPTION_KEY` | — | APIM subscription key. |
| `azure_apim_openai_deployment_name` | `vars.AZURE_APIM_OPENAI_DEPLOYMENT_NAME` | `gpt-5-mini` | OpenAI model deployment name configured in APIM. |
| `azure_apim_openai_api_version` | `vars.AZURE_APIM_OPENAI_API_VERSION` | `2024-12-01-preview` | OpenAI API version. |
| `grafana_otlp_endpoint` | `vars.GRAFANA_OTLP_ENDPOINT` | — | Grafana OTLP collector endpoint URL. |
| `grafana_otlp_headers` | `secrets.GRAFANA_OTLP_HEADERS` | — | Auth headers for Grafana OTLP (e.g., `Authorization=Bearer <token>`). |
| `auth_tenant_id` | `vars.AUTH_TENANT_ID` → `vars.AZURE_TENANT_ID` fallback | — | Tenant ID for application-level authentication (may differ from the infrastructure tenant). |

In addition, a `terraform.tfvars` file in the `configuration/` directory is always loaded via `-var-file`. This file should contain static, non-sensitive defaults that do not change between runs.

---

## Terraform State Management

Each environment has its own isolated state file in Azure Blob Storage:

```
Storage Account:  <AZURE_TF_BACKEND_NAME>
Container:        <AZURE_TF_BACKEND_CONTAINER_NAME>
Blob (state key): <PROJECT_NAME>-<ENV_NAME>.terraform.tfstate
```

Example for the `dev` environment of project `myapp`:
```
myapp-dev.terraform.tfstate
```

**State locking:** Azure Blob Storage provides native state locking via blob leases. Two concurrent Terraform operations against the same state file will not corrupt each other — the second will wait or fail with a lock error.

**Backend initialisation flags:**
- `-upgrade`: updates provider versions within the constraints declared in `terraform.tf`
- `-reconfigure`: forces re-reading of backend config, preventing stale configuration from a previous init in the same workspace

> **If you need to unlock a stuck state**, use the `force-unlock.yml` workflow with the lock ID shown in the error output. Do not manually delete blobs or edit state files.

---

## Authentication: OIDC

Azure authentication uses **OpenID Connect (OIDC)** — no client secret is stored. The Terraform Azure provider is configured to use OIDC via:

```
ARM_USE_OIDC=true
ARM_USE_CLI=false
```

Each job authenticates independently. The OIDC token is scoped to the job and expires when the job ends.

**Federated credential requirements on the Service Principal:**

The Azure Service Principal must have a Federated Credential for:
- **Repository:** `<org>/<repo>`
- **Entity type:** `Environment`
- **Environment name:** matching each GitHub Environment used (e.g., `dev`, `prod`, `hitl-approvals`)

> **Common mistake:** Setting the entity type to "Branch" instead of "Environment". Jobs that specify `environment:` receive an OIDC subject scoped to the environment — a branch-scoped federated credential will not match and authentication will fail.

---

## Plan Exit Codes

Terraform's `-detailed-exitcode` flag produces three distinct exit codes:

| Exit Code | Meaning | Workflow Behaviour |
|---|---|---|
| `0` | No changes detected — infrastructure is already up to date. | Plan succeeds. Apply / destroy jobs still run if selected, but apply will complete instantly (no-op). |
| `1` | Terraform error — invalid configuration, provider error, etc. | Plan fails. All downstream jobs are blocked. |
| `2` | Changes detected — resources will be created, updated, or destroyed. | Plan succeeds. Step summary shows the change counts and prompts to trigger an apply. |

The exit code is captured and published as the `plan_exitcode` job output, visible in the step summary.

---

## Force-Replace Targets

The `replace_targets` input allows forcing one or more resources to be destroyed and re-created, even if Terraform considers them up to date. This is useful for:

- Resources that have drifted outside Terraform (e.g., manually modified in the portal)
- Resources that must be cycled to pick up a new provider version's defaults
- Rotated certificates or secrets that the provider cannot detect as changed

**Format:** space-separated resource addresses, exactly as they appear in `terraform state list`.

**Example:**
```
module.agentcell_stack.terraform_data.approve_frontdoor_private_links
```

**Multiple targets:**
```
module.agentcell_stack.azurerm_container_app.backend module.agentcell_stack.azurerm_container_app.frontend
```

Internally, the workflow builds `-replace=<address>` flags and appends them to the `terraform plan` command. The replace instructions are baked into the `tfplan` artifact, so the apply step will perform the replacement automatically.

> **Tip:** Always run with `plan` first to review the replace impact before triggering `apply`.

---

## Protected Resources on Destroy

Before running `terraform destroy`, the workflow removes **role assignments** from the Terraform state. This means they will not be destroyed — they are preserved on the resource group even after everything else is gone.

**Why:** Role assignments grant the Service Principal and managed identities access to the subscription and resource group. Destroying them would make it impossible to re-provision the environment without manual intervention by an Azure administrator.

**What is preserved:**
- All `azurerm_role_assignment` resources in the state

**What is destroyed:**
- Everything else managed by Terraform in that environment

**Implementation:**
```bash
role_assignments=$(terraform state list | grep 'azurerm_role_assignment' || true)
echo "$role_assignments" | while read -r resource; do
  terraform state rm "$resource" || true
done
```

The `|| true` guards make the loop idempotent — it is safe to run against a state that has already had role assignments removed.

> **Note:** The resource group itself is declared as a `data` source in this configuration (not a `resource`), so it is never managed or destroyed by Terraform.

---

## Post-Apply: Terraform Outputs

After a successful apply, the workflow reads all Terraform outputs from state and:

1. Exposes them as **job outputs** (available to downstream `workflow_call` callers)
2. Writes them as a **step summary table** in the GitHub Actions UI

| Output | Job Output Key | Description |
|---|---|---|
| `frontdoor_endpoint_hostname` | `frontdoor_endpoint_hostname` | Front Door hostname for the UI endpoint (e.g., `xyz.azurefd.net`). Empty if Front Door is not deployed. |
| `frontdoor_agent_urls` | `frontdoor_agent_urls` | JSON map of agent name → Front Door URL for each agent. |
| `container_app_ui_url` | `container_app_ui_url` | Direct Container App URL for the UI (bypasses Front Door). |
| `container_app_agent_urls` | `container_app_agent_urls` | JSON map of agent name → direct Container App URL. |

If `frontdoor_endpoint_hostname` is non-empty, the step summary shows both Front Door and direct URLs. If empty, only direct URLs are shown.

---

## Post-Apply: Auto-Commit

After a successful apply, the workflow checks whether Terraform modified a file at:

```
configuration/api-gateway/config.json
```

If this file exists and was changed during the apply (detected via `git status --porcelain`), the workflow commits it back to the branch:

```
Add config.json (automated)
```

Commit author: `GitHub Actions <github-actions[bot]@users.noreply.github.com>`

**When does this happen?** Only when a Terraform resource writes to that file as a side effect (e.g., an API gateway configuration export). If the file does not exist or was not modified, this step is a no-op and prints a skip message.

**Required permission:** The `terraform-apply` job declares `contents: write` for this reason.

> If you do not use `api-gateway/config.json` in your setup, this step is harmless and will always be skipped.

---

## Chaining with Deploy Components

A `deploy-components` job is defined in the workflow file but is **currently commented out**. When uncommented, it would automatically trigger `deploy-components.yml` after a successful apply if `deploy_components == true`.

The intended shape when re-enabled:

```yaml
deploy-components:
  name: Deploy All Components
  needs: [terraform-apply]
  if: >
    needs.terraform-apply.result == 'success' &&
    (inputs.deploy_components == true || github.event.inputs.deploy_components == 'true')
  uses: ./.github/workflows/deploy-components.yml
  with:
    environment: ${{ inputs.environment || github.event.inputs.environment }}
    product_name: ${{ vars.PROJECT_NAME }}
    deploy_all: true
  secrets:
    ORG_REPOS_INTERNAL_READ_ONLY: ${{ secrets.ORG_REPOS_INTERNAL_READ_ONLY }}
```

To enable it: uncomment the block and add the remaining required secrets (`ARTIFACTORY_USER_TOKEN`, `AZURE_CLIENT_SECRET`, `CLIENT_SECRET`, `MONGODB_URI`, `REDIS_URL`).

---

## GitHub Environment Configuration

This workflow uses **two distinct GitHub Environments** per deployment:

### Target Environment (e.g., `dev`, `prod`)

- Activated by: `terraform-plan`, `terraform-apply`, `terraform-destroy`
- Holds all environment-scoped Variables and Secrets
- Optional: add required reviewers here for additional protection on prod

### `hitl-approvals` (Human-In-The-Loop)

- Activated by: `manual-approval` job only
- **Must have required reviewers configured** — this is the approval gate
- The environment URL is set to the Azure Portal resource group overview for reviewer convenience
- Rejecting here cancels the entire workflow run

> The `hitl-approvals` environment is shared across all target environments. Reviewers configured there can approve applies and destroys for any environment.

---

## Required GitHub Configuration

### Repository / Environment Variables

Must be set per environment in **Settings → Environments → \<environment\> → Variables**:

| Variable | Description |
|---|---|
| `AZURE_CLIENT_ID` | Service Principal client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target Azure subscription ID |
| `AZURE_RESOURCE_GROUP` | Resource group for the deployment |
| `AZURE_TF_BACKEND_NAME` | Storage account name for Terraform state |
| `AZURE_TF_BACKEND_CONTAINER_NAME` | Blob container for Terraform state |
| `PROJECT_NAME` | Short product identifier (used in state key and resource naming) |
| `AZURE_APIM_GATEWAY_URL` | APIM gateway base URL |
| `AZURE_APIM_OPENAI_DEPLOYMENT_NAME` | OpenAI deployment name in APIM (default: `gpt-5-mini`) |
| `AZURE_APIM_OPENAI_API_VERSION` | OpenAI API version (default: `2024-12-01-preview`) |
| `GRAFANA_OTLP_ENDPOINT` | Grafana OTLP endpoint URL |
| `AUTH_TENANT_ID` | App-level auth tenant (defaults to `AZURE_TENANT_ID` if unset) |

### Repository Secrets

| Secret | Scope | Description |
|---|---|---|
| `ORG_REPOS_INTERNAL_READ_ONLY` | Repository | GitHub PAT for private Terraform module access |
| `APIM_SUBSCRIPTION_KEY` | Environment | APIM subscription key |
| `GRAFANA_OTLP_HEADERS` | Environment | Grafana OTLP auth headers |
| `GH_PAT_RUNNER` | Environment | PAT for self-hosted runner provisioning (optional) |

---

## Service Principal RBAC Requirements

The federated Service Principal used for OIDC requires the following Azure RBAC assignments:

| Role | Scope | Reason |
|---|---|---|
| `Contributor` | Subscription | Required to purge soft-delete protected resources (Key Vault, Cognitive Services) which need subscription-level access |
| `User Access Administrator` | Resource Group | Required to create and manage role assignments for managed identities |

> **Why Subscription-level Contributor?** Resources like Azure Key Vault and Azure Cognitive Services (AI Foundry) use soft-delete with purge protection. Purging them (to allow re-creation with the same name) requires `Microsoft.KeyVault/locations/deletedVaults/purge/action`, which is only grantable at subscription scope.

---

## Troubleshooting

### Plan fails at `terraform init` with "state not found"

The Terraform state file does not exist yet. This is expected on first deployment. Run with `apply` — Terraform will create the state file automatically on the first successful apply.

### Plan fails with "backend configuration changed"

The `-reconfigure` flag is set, so this should not normally occur. If it does, check that `PROJECT_NAME` and `ENV_NAME` variables are set correctly and that no backend variable has been renamed between runs.

### `manual-approval` is skipped but apply/destroy does not run

Check that `skip_manual_approval` was set to `true` explicitly. If it was left at the default `false`, the approval job should have run. If `terraform-plan` failed (exit code 1), the approval job is also skipped — check the plan job logs first.

### Apply fails with "plan file is stale"

The `tfplan` artifact was generated by a different Terraform provider version or configuration than what is currently initialised. This can happen if:
- A provider version was updated in `terraform.tf` between the plan and apply runs
- The artifact expired (retention is 5 days)

Re-run the full workflow starting from `plan`.

### Destroy fails to delete a resource (dependency error)

Some Azure resources have deletion protection or dependencies that Terraform cannot automatically resolve. Common examples: Key Vault with purge protection enabled, resources with active private endpoints. Check the destroy logs for the specific resource address, handle the dependency manually in the Azure portal, then re-run the destroy.

### Role assignments still exist after destroy (expected)

This is intentional. See [Protected Resources on Destroy](#protected-resources-on-destroy). Role assignments are explicitly excluded from the destroy run to preserve access for future re-provisioning.

### Auto-commit step fails with `403 Permission denied`

The `GITHUB_TOKEN` for the job does not have write access, or branch protection rules block direct pushes. Either:
- Grant the workflow `contents: write` at the repository level (already set on `terraform-apply`)
- Add an exception to the branch protection rule for `github-actions[bot]`

### OIDC authentication fails with "subject does not match"

The Federated Credential on the Service Principal is configured for the wrong entity type. Ensure it is set to **Environment**, not Branch, and that the environment name matches exactly (case-sensitive).
