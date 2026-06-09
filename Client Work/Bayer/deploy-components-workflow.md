# Deploy Components Workflow

**File:** `.github/workflows/deploy-components.yml`

This document describes every aspect of the **Deploy Components (Multi-Agent)** GitHub Actions workflow — how to trigger it, what it needs, how it works internally, and what your colleagues should watch out for when maintaining or extending it.


---

## Overview

This workflow builds Docker images for up to three application components and deploys them to **Azure Container Apps**. It is designed as a **selective deployment pipeline** — you can deploy all components at once or target individual ones in a single run.

```
Trigger (manual or called)
       │
       ▼
  [preflight]  ──── validates Azure targets, reads Terraform outputs, sets version
       │
       ├──── [build-agents-backend]  ──── [deploy-agents-backend]
       │
       ├──── [build-agents-frontend] ──── [deploy-agents-frontend]
       │
       └──── [build-ui]              ──── [deploy-ui]
                                              │
                                              ▼
                                    [deployment-summary]  (always runs)
```

All build jobs run **in parallel** after `preflight` completes. Each deploy job waits only for its own build job — the three component pipelines are fully independent of each other.

**Runner:** All jobs run on `external-k8s-v2` (Kubernetes-based self-hosted runners). Deploy jobs use Ubuntu 24.04 containers pulled from `official-dockerhub-images.artifactory.bayer.com`.

---

## Trigger Mechanisms

The workflow supports two trigger types:

### `workflow_dispatch` (Manual via GitHub UI)

Navigate to **Actions → Deploy Components (Multi-Agent) → Run workflow** in the GitHub UI and fill in the form inputs.

The run name is displayed as:
```
Deploy Components → <environment> (#<run_number>)
```

### `workflow_call` (Called from Another Workflow)

The workflow can be orchestrated by a parent workflow (e.g., a full-stack deployment pipeline). The caller passes the same inputs and must forward the required secrets:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy-components.yml
    with:
      environment: dev
      deploy_all: true
    secrets:
      ARTIFACTORY_USER_TOKEN: ${{ secrets.ARTIFACTORY_USER_TOKEN }}
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      CLIENT_SECRET: ${{ secrets.CLIENT_SECRET }}
      MONGODB_URI: ${{ secrets.MONGODB_URI }}
      REDIS_URL: ${{ secrets.REDIS_URL }}
```

> **Important:** When called via `workflow_call`, secrets are **not** automatically inherited — the caller must explicitly pass each secret in the `secrets:` block.

---

## Inputs Reference

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `environment` | choice / string | Yes | `sandbox` | Target Azure environment. Controls which GitHub Environment (and its protection rules / variable sets) is activated. |
| `product_name` | string | No | `dwp` | Must match the `product_name` Terraform variable used during infrastructure provisioning. Used for display only in this workflow. |
| `version` | string | No | _(empty)_ | Docker image version tag. Leave empty to auto-generate from date + commit SHA + run ID (see [Version Tagging Strategy](#version-tagging-strategy)). |
| `deploy_all` | boolean | No | `false` | When `true`, all three components are built and deployed regardless of the individual toggles below. |
| `deploy_agents_backend` | boolean | No | `false` | Deploy only the Agents Backend component. |
| `deploy_ui` | boolean | No | `false` | Deploy only the UI component. |
| `deploy_agents_frontend` | boolean | No | `false` | Deploy only the Agents Frontend component. |

**Selection logic:** A component's build+deploy jobs run when:
```
deploy_all == true  OR  deploy_<component> == true
```
If none of the flags are set to `true`, no component is built or deployed — only the `preflight` and `deployment-summary` jobs run.

---

## Secrets Reference

Secrets are only relevant when the workflow is called via `workflow_call`. For `workflow_dispatch`, secrets are read directly from the GitHub Environment.

| Secret | Required | Used by | Purpose |
|---|---|---|---|
| `ARTIFACTORY_USER_TOKEN` | **Yes** | All build + deploy jobs | Authentication token for JFrog Artifactory (image push and pull). |
| `AZURE_CLIENT_SECRET` | No | `deploy-agents-backend` | Injected into the container app as `AZURE_CLIENT_SECRET` (secretref). |
| `CLIENT_SECRET` | No | `deploy-agents-backend` | Injected into the container app as `CLIENT_SECRET` (secretref). |
| `MONGODB_URI` | No | `deploy-agents-backend` | Connection string for MongoDB, injected as `MONGODB_URI` (secretref). |
| `REDIS_URL` | No | `deploy-agents-frontend` | Redis connection URL, injected as `REDIS_URL` (secretref). |
| `ORG_REPOS_INTERNAL_READ_ONLY` | No | `preflight` | GitHub PAT for accessing private Terraform modules during `terraform init`. |

> **Note on OIDC:** Azure authentication does **not** use a stored client secret in the runner. It uses OpenID Connect (OIDC) — the runner exchanges a short-lived token with Azure AD. `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are expected to be GitHub **Variables** (not secrets). See [Authentication: OIDC](#authentication-oidc).

---

## Environment Variables

These are defined at the workflow level and available to all jobs:

| Variable | Source | Value / Purpose |
|---|---|---|
| `AZURE_CLIENT_ID` | `vars` (fallback `secrets`) | Service Principal / Managed Identity client ID for OIDC login. |
| `AZURE_TENANT_ID` | `vars` (fallback `secrets`) | Azure AD tenant ID. |
| `AZURE_SUBSCRIPTION_ID` | `vars` (fallback `secrets`) | Azure subscription to deploy into. |
| `AZURE_TF_BACKEND_NAME` | `vars` (fallback `secrets`) | Storage account name for the Terraform remote state backend. |
| `AZURE_TF_BACKEND_CONTAINER_NAME` | `vars` (fallback `secrets`) | Blob container name for the Terraform state file. |
| `ARTIFACTORY_REPOSITORY_NAME` | `vars` | Fully qualified Artifactory hostname (e.g., `dwp-docker-dev-dwpagents.artifactory.bayer.com`). |
| `ARTIFACTORY_REPOSITORY_FOLDER` | hardcoded | `dwp-docker-dev-dwpagents` — the folder/repo path within Artifactory. |
| `ARTIFACTORY_USER_NAME` | `vars` | Artifactory service account username. |
| `ARTIFACTORY_USER_TOKEN` | `secrets` | Artifactory API token. |
| `PRODUCT_NAME` | `inputs.product_name` | Passed-through from workflow input. |
| `ENV_NAME` | `inputs.environment` | Passed-through from workflow input. |
| `TERRAFORM_DIR` | hardcoded | `configuration` — path to the Terraform root module relative to repo root. |

> **Fallback pattern:** `${{ vars.X || secrets.X }}` means the value is first looked up in GitHub Variables, then in Secrets. Prefer Variables for non-sensitive identifiers.

---

## Job Architecture

```
preflight
  └─ outputs: version, resource_name_suffix, unique_id, frontdoor_url

build-agents-backend  (if deploy_all OR deploy_agents_backend)
  └─ needs: preflight

deploy-agents-backend  (if deploy_all OR deploy_agents_backend)
  └─ needs: preflight, build-agents-backend

build-agents-frontend  (if deploy_all OR deploy_agents_frontend)
  └─ needs: preflight

deploy-agents-frontend  (if deploy_all OR deploy_agents_frontend)
  └─ needs: preflight, build-agents-frontend

build-ui  (if deploy_all OR deploy_ui)
  └─ needs: preflight

deploy-ui  (if deploy_all OR deploy_ui)
  └─ needs: preflight, build-ui

deployment-summary  (always, even on failure)
  └─ needs: preflight, deploy-agents-backend, deploy-agents-frontend, deploy-ui
```

---

## Job Details

### 1. `preflight` — Validate & Prepare

**Purpose:** Central validation and information-gathering step. All subsequent jobs depend on its outputs.

**What it does:**

1. **Installs tooling** — Azure CLI and supporting packages (`jq`, `curl`, `unzip`) into the Ubuntu container.
2. **Prints pipeline parameters** to the GitHub Actions step summary (environment, product, resource group, components selected).
3. **Resolves the version tag** — uses `inputs.version` if provided, otherwise generates:
   ```
   YYYY.MM.DD-<7-char-SHA>-<run_id>
   ```
4. **Logs in to Azure** via OIDC (`azure/login` action).
5. **Sets up Terraform** (v1.14.0, wrapper disabled) and initialises the backend:
   ```
   terraform init \
     -backend-config="resource_group_name=<RESOURCE_GROUP_NAME>" \
     -backend-config="storage_account_name=<AZURE_TF_BACKEND_NAME>" \
     -backend-config="container_name=<AZURE_TF_BACKEND_CONTAINER_NAME>" \
     -backend-config="key=<PROJECT_NAME>-<ENV>.terraform.tfstate"
   ```
6. **Reads Terraform outputs** (`resource_name_suffix`, `resource_name_suffix_compact`, `unique_id`, `frontdoor_endpoint_hostname`) and exposes them as job outputs consumed by all deploy jobs.
7. **Validates Azure targets** — confirms the resource group exists and pings Artifactory.

**Job outputs consumed downstream:**

| Output key | Used in | Purpose |
|---|---|---|
| `version` | All build + deploy jobs | Image tag applied to all pushed images. |
| `resource_name_suffix` | All deploy jobs | Appended to container app names to find the right resource. |
| `unique_id` | Available (not currently used) | Terraform-generated unique ID for resource disambiguation. |
| `frontdoor_url` | Available (not currently used) | Front Door hostname for post-deploy verification. |

---

### 2. `build-agents-backend`

**Condition:** `deploy_all == true OR deploy_agents_backend == true`

**Source directory:** `images/agents-backend/`

**Dockerfile:** `images/agents-backend/Dockerfile`

**Image name:** `agents-backend`

**Steps:**
1. Checks out the repository.
2. Sets up Docker Buildx for multi-arch capable builds.
3. Logs in to JFrog Artifactory.
4. Builds and pushes the image to:
   ```
   <ARTIFACTORY_REPOSITORY_NAME>/<ARTIFACTORY_REPOSITORY_FOLDER>/agents-backend:<version>
   ```
   Platform: `linux/amd64` only.

---

### 3. `deploy-agents-backend`

**Condition:** Same as build — runs only after its build job succeeds.

**Target resource:**
```
ca-agents-backend-<resource_name_suffix>
```

**Steps:**
1. Installs Azure CLI.
2. Logs in to Azure (OIDC).
3. **Validates** the container app exists (`az containerapp show`). Fails fast if the resource is missing — Terraform must be applied first.
4. **Configures the Artifactory registry** on the container app (stores credentials so the app can pull future images).
5. **Sets secrets** on the container app:
   - `azure-client-secret` ← `secrets.AZURE_CLIENT_SECRET`
   - `client-secret` ← `secrets.CLIENT_SECRET`
   - `mongodb-uri` ← `secrets.MONGODB_URI`
6. **Sets environment variables**, referencing secrets by `secretref:`:
   - `AGENT_API_BASE_URL` ← `vars.AGENT_API_BASE_URL`
   - `CLIENT_ID` ← `vars.CLIENT_ID`
   - `OAUTH_SCOPE` ← `vars.OAUTH_SCOPE`
   - `SERVICENOW_INSTANCE` ← `vars.SERVICENOW_INSTANCE`
   - `AZURE_CLIENT_SECRET` → `secretref:azure-client-secret`
   - `CLIENT_SECRET` → `secretref:client-secret`
   - `MONGODB_URI` → `secretref:mongodb-uri`
7. **Updates the container app** to the new image tag.

> **Important:** `az containerapp secret set` replaces the entire secret collection. Any secrets not listed in this step will be removed from the app.

---

### 4. `build-agents-frontend`

**Condition:** `deploy_all == true OR deploy_agents_frontend == true`

**Source directory:** `images/agents-frontend/`

**Dockerfile:** `images/agents-frontend/Dockerfile`

**Image name:** `agents-frontend`

Identical build pattern to agents-backend. Platform: `linux/amd64`.

---

### 5. `deploy-agents-frontend`

**Condition:** Same as build — runs after its own build job.

**Target resource:**
```
ca-agents-frontend-<resource_name_suffix>
```

**Secrets injected:**
- `redis-url` ← `secrets.REDIS_URL`

**Environment variables set:**
- `AAD_APP_CLIENT_ID` ← `vars.AAD_APP_CLIENT_ID`
- `AGENT_API_BASE_URL` ← `vars.AGENT_API_BASE_URL`
- `BOT_DOMAIN` ← `vars.BOT_DOMAIN`
- `BOT_ID` ← `vars.BOT_ID`
- `REDIS_URL` → `secretref:redis-url`

---

### 6. `build-ui`

**Condition:** `deploy_all == true OR deploy_ui == true`

**Source directory:** `agent-ui/`

**Dockerfile:** `agent-ui/Dockerfile`

**Build target:** `chat` (multi-stage Dockerfile — only the `chat` stage is built)

**Image name:** `ui-app`

**Additional build arg:**
```
BUILD_VERSION=<version>
```
This is the only component that injects the version into the image at build time via a build argument.

---

### 7. `deploy-ui`

**Condition:** Same as build — runs after its own build job.

**Target resource:**
```
ca-uiapp-<resource_name_suffix>
```

**Steps:** Configures the Artifactory registry and updates the image. Unlike the other components, the UI deploy does **not** set any secrets or environment variables — these are managed separately (e.g., via Terraform or the Azure portal).

---

### 8. `deployment-summary`

**Condition:** Always runs (`if: always()`), even when all component jobs were skipped or failed.

**Purpose:** Writes a final Markdown summary to the GitHub Actions step summary, including:
- Configuration (environment, product, resource group, version, registry path)
- Status of each component (`success`, `failure`, or `skipped`)
- Full image path of deployed images

This job has no Azure credentials and does not interact with any external system.

---

## Component-to-Resource Mapping

Container app names are constructed by combining a static prefix with the `resource_name_suffix` output from Terraform:

| Component | Container App Name |
|---|---|
| Agents Backend | `ca-agents-backend-<resource_name_suffix>` |
| Agents Frontend | `ca-agents-frontend-<resource_name_suffix>` |
| UI | `ca-uiapp-<resource_name_suffix>` |

The `resource_name_suffix` is read from the Terraform state at runtime. If the Terraform state does not contain the expected output, the `preflight` job will fail before any deployment begins.

> **If a container app is not found:** The `Validate Container App Exists` step will fail with an error. This means Terraform has not been applied for that environment yet. Run the `deploy.yml` or `validate-and-deploy.yml` workflow first to provision the infrastructure.

---

## Version Tagging Strategy

**Manual version** (when `inputs.version` is set):
```
<whatever you typed>
```

**Auto-generated version** (when `inputs.version` is empty):
```
YYYY.MM.DD-<7-char-git-SHA>-<github_run_id>
```

Example: `2025.05.06-a3f1c2d-12345678`

All three components built in a single workflow run always receive the **same version tag**. There is no per-component versioning within a single run.

The version is:
- Set in `preflight` once
- Passed to all build jobs via `needs.preflight.outputs.version`
- Applied identically to all Docker image tags

---

## Authentication: OIDC

Azure authentication uses **OpenID Connect (OIDC)** — no client secret is stored in GitHub. The flow:

1. GitHub generates a short-lived OIDC token for the job.
2. The `azure/login` action exchanges that token for an Azure access token.
3. The access token is used for all `az` CLI calls in that job.

**Required configuration on the Azure side:**
- A Service Principal (or Managed Identity) with **Federated Credentials** configured to trust tokens from this GitHub repository, for the specific branch and environment.
- The SP must have sufficient RBAC permissions on the subscription and resource group (at minimum: `Contributor` on the resource group, `AcrPull` on the registry if applicable).

**Required GitHub Variables** (not secrets):
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

Each job that touches Azure re-authenticates independently — the OIDC token is scoped per-job.

ARM environment variables are written explicitly in `preflight` to support Terraform's Azure provider:
```
ARM_CLIENT_ID, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID, ARM_USE_OIDC=true
```

---

## Terraform Integration

The workflow does **not** apply Terraform changes. It only reads outputs from an already-applied state. This is an intentional read-only integration.

**What is read:**

| Terraform Output | Workflow Output | Purpose |
|---|---|---|
| `resource_name_suffix` | `resource_name_suffix` | Appended to container app names. |
| `resource_name_suffix_compact` | _(printed to summary only)_ | Compact variant of the suffix. |
| `unique_id` | `unique_id` | Exposed but not currently used downstream. |
| `frontdoor_endpoint_hostname` | `frontdoor_url` | Exposed but not currently used downstream (graceful fallback to empty string if not present). |

**State file key pattern:**
```
<PROJECT_NAME>-<environment>.terraform.tfstate
```
`PROJECT_NAME` is read from the `vars.PROJECT_NAME` GitHub Variable.

**Terraform version:** `1.14.0` (pinned via `hashicorp/setup-terraform`).

**Private modules:** If Terraform modules are sourced from private GitHub repositories, the step `Configure Git for Private Modules` rewrites the Git remote URL to use the `ORG_REPOS_INTERNAL_READ_ONLY` token. If no private modules are used, this secret can be left empty.

---

## Artifactory Registry

All Docker images are pushed to and pulled from **JFrog Artifactory**.

**Full image path pattern:**
```
<ARTIFACTORY_REPOSITORY_NAME>/<ARTIFACTORY_REPOSITORY_FOLDER>/<IMAGE_NAME>:<VERSION>
```

Example:
```
dwp-docker-dev-dwpagents.artifactory.bayer.com/dwp-docker-dev-dwpagents/agents-backend:2025.05.06-a3f1c2d-9876543
```

**Authentication in build jobs:** via `docker/login-action` using `ARTIFACTORY_USER_NAME` / `ARTIFACTORY_USER_TOKEN`.

**Authentication in deploy jobs:** the same credentials are registered on the Azure Container App via `az containerapp registry set`. This enables the Container App's managed environment to pull updated images on future restarts without needing to re-run this workflow.

---

## GitHub Environment Protection

Every job in this workflow declares `environment: ${{ inputs.environment }}`. This activates the corresponding [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) and its protection rules.

**Consequences:**
- **Required reviewers** on `staging` or `prod` environments will pause the entire workflow until approved.
- **Environment-scoped Variables and Secrets** override repository-level ones within jobs that reference that environment.
- Deployment history is tracked per environment in the GitHub UI under **Environments**.

**Environment-scoped Variables that must exist per environment:**

| Variable | Description |
|---|---|
| `AZURE_RESOURCE_GROUP` | Resource group containing all container apps. |
| `AZURE_CLIENT_ID` | OIDC client ID (can also be a repo-level variable). |
| `AZURE_TENANT_ID` | Azure AD tenant ID. |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID. |
| `AZURE_TF_BACKEND_NAME` | Terraform backend storage account name. |
| `AZURE_TF_BACKEND_CONTAINER_NAME` | Terraform backend blob container name. |
| `ARTIFACTORY_REPOSITORY_NAME` | Artifactory registry hostname. |
| `ARTIFACTORY_USER_NAME` | Artifactory username. |
| `PROJECT_NAME` | Used to construct the Terraform state key. |
| `AGENT_API_BASE_URL` | Backend API base URL injected into containers. |
| `CLIENT_ID` | (Agents Backend) OAuth client ID. |
| `OAUTH_SCOPE` | (Agents Backend) OAuth scope string. |
| `SERVICENOW_INSTANCE` | (Agents Backend) ServiceNow instance hostname. |
| `AAD_APP_CLIENT_ID` | (Agents Frontend) AAD app client ID. |
| `BOT_DOMAIN` | (Agents Frontend) Bot domain. |
| `BOT_ID` | (Agents Frontend) Bot ID. |

---

## Required GitHub Configuration

### Repository / Environment Variables

All variables listed in the table above must be set per environment in **Settings → Environments → \<environment\> → Variables**.

### Repository Secrets

| Secret | Where to set |
|---|---|
| `ARTIFACTORY_USER_TOKEN` | Environment or Repository |
| `AZURE_CLIENT_SECRET` | Environment |
| `CLIENT_SECRET` | Environment |
| `MONGODB_URI` | Environment |
| `REDIS_URL` | Environment |
| `ORG_REPOS_INTERNAL_READ_ONLY` | Repository (shared across environments) |

### Self-Hosted Runner

All jobs require the `external-k8s-v2` runner label. Ensure at least one runner with this label is online and has network access to:
- `management.azure.com` (Azure API)
- `packages.microsoft.com` (Azure CLI packages)
- `<ARTIFACTORY_REPOSITORY_NAME>` (Docker registry)
- GitHub API (for `actions/checkout`)

---

## Adding a New Component

To add a fourth deployable component (e.g., `data-processor`):

1. **Add input flags** to both `workflow_dispatch.inputs` and `workflow_call.inputs`:
   ```yaml
   deploy_data_processor:
     description: 'Deploy Data Processor'
     required: false
     type: boolean
     default: false
   ```

2. **Add a build job**, copying the pattern from `build-agents-backend`. Change:
   - `IMAGE_NAME: data-processor`
   - `context:` and `file:` to the correct source directory

3. **Add a deploy job**, copying the pattern from `deploy-agents-backend`. Change:
   - `IMAGE_NAME: data-processor`
   - `CONTAINER_APP_NAME: "ca-data-processor-${{ needs.preflight.outputs.resource_name_suffix }}"`
   - Secrets and environment variables specific to this component

4. **Update the `if:` conditions** on both jobs:
   ```yaml
   if: ${{ inputs.deploy_all == true || inputs.deploy_data_processor == true }}
   ```

5. **Add the deploy job to `deployment-summary`'s `needs:`** list so its status is reported.

6. **Update the Terraform module** to output a container app named `ca-data-processor-<suffix>`.

---

## Troubleshooting

### `preflight` fails at "Read Terraform Naming Outputs"

The Terraform state either does not exist for the target environment or does not contain the expected outputs. Verify:
- The `PROJECT_NAME`, `AZURE_TF_BACKEND_NAME`, `AZURE_TF_BACKEND_CONTAINER_NAME` variables are correct for the environment.
- Terraform has been successfully applied at least once for this environment.
- The Terraform root module in `configuration/` exports `resource_name_suffix` and `unique_id` outputs.

### `Validate Container App Exists` fails

The Azure Container App named `ca-<component>-<suffix>` does not exist in the resource group. Run the infrastructure provisioning workflow (e.g., `deploy.yml`) first.

### OIDC authentication fails

Check that the **Federated Credential** on the Azure Service Principal is configured for:
- Repository: `<org>/<repo>`
- Entity: `Environment: <environment>` (not just branch)

If the entity type is set to "Branch" instead of "Environment", jobs that specify `environment:` will receive a different OIDC subject and authentication will fail.

### Image push to Artifactory fails

Verify `ARTIFACTORY_USER_TOKEN` is not expired and that the service account has write permission on `ARTIFACTORY_REPOSITORY_FOLDER` in Artifactory.

### Secrets are missing from the container app after deploy

`az containerapp secret set` performs a **full replacement** of the secret collection. If a secret exists on the app but is not listed in the `--secrets` argument in the workflow, it will be deleted. Always include all required secrets in the step, even if their values have not changed.

### A component was skipped unexpectedly

If `deploy_all` is `false` and the individual toggle was not explicitly set to `true`, the job is skipped. Check the `Display Pipeline Parameters` step in `preflight` — it prints the effective values of all toggles to the step summary.
