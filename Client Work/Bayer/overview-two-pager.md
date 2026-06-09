# ITF DWP Agentic System — Handover Overview

**Author:** Tim Kroll · **Date:** 2026-05-07

---

## Part 1 — Infrastructure Architecture

### System Components

```
Users / Microsoft Teams
        │
        ▼
┌─────────────────────────┐     ┌──────────────────────────┐
│  Azure Front Door (CDN) │────▶│  UI Container App        │
│  Public HTTPS entry     │     │  (React/Next.js frontend) │
└─────────────────────────┘     └──────────────────────────┘
        │
        ▼
┌─────────────────────────┐     ┌──────────────────────────┐
│  Azure Bot Service      │────▶│  Agents Backend          │
│  Teams channel bridge   │     │  Container App (Python)  │
└─────────────────────────┘     └──────────────────────────┘
                                         │
                                         ▼
                                ┌──────────────────────────┐
                                │  Azure Function          │
                                │  (secured via Easy Auth) │
                                └──────────────────────────┘
```

### Entra ID App Registrations

Three App Registrations control identity and access — full details in [handover-entra-id-app-registrations.md](handover-entra-id-app-registrations.md).

| Registration | Purpose | Credential |
|---|---|---|
| **ITF DWP Agentic System** | Identity of the Azure Bot (Bot Service ↔ Teams) | User-Assigned Managed Identity — no secret needed |
| **ITF DWP M365 Agentic System** | Graph API & M365 permissions for the bot | Client secret / certificate — **check expiry regularly** |
| **ITF DWP Azure Function** | Easy Auth guard on the Azure Function HTTP endpoints | Client secret stored as app setting — **check expiry regularly** |

> Entra ID admin center: https://entra.microsoft.com → App registrations → search `ITF DWP`

### Teams App (Microsoft 365 Developer Portal)

The bot is surfaced in Teams through a **Teams App Package** (`manifest.json` + icons). It is managed in the Developer Portal — full details in [handover-teams-bot-developer-portal.md](handover-teams-bot-developer-portal.md).

| Task | Where |
|---|---|
| Edit app name, description, icons | https://dev.teams.microsoft.com → Apps → Configure |
| Update bot endpoint after redeployment | Developer Portal → Tools → Bot management |
| Add / remove colleagues as app owners | Configure → Advanced → Owners |
| Republish after changes | Configure → Publish → Publish to org |

### Terraform State & Configuration

| Item | Detail |
|---|---|
| Terraform working directory | `configuration/` |
| State backend | Azure Blob Storage (account: `AZURE_TF_BACKEND_NAME`, container: `AZURE_TF_BACKEND_CONTAINER_NAME`) |
| State key per env | `<PROJECT_NAME>-<env>.terraform.tfstate` |
| Environments | `sandbox` · `dev` · `test` · `qa` · `prod` |
| Key outputs | `frontdoor_endpoint_hostname`, `container_app_ui_url`, `resource_name_suffix` |

---

## Part 2 — CI/CD

### Workflow Map

```
GitHub Actions
├── deploy-infrastructure.yml   ← Terraform plan / apply / destroy
│       └── (optional) calls deploy-components.yml after apply
├── deploy-components.yml       ← Build images + update Container Apps
├── pr-check.yml / pr-plan.yml  ← Terraform plan on pull requests
├── security-checks.yml         ← SAST / dependency scanning
└── sonarqube.yml               ← Code quality gate
```

---

### `deploy-infrastructure.yml` — Terraform Lifecycle

**Trigger:** Manual (`workflow_dispatch`) or called by another workflow (`workflow_call`)

**Inputs:**

| Input | Options | Notes |
|---|---|---|
| `environment` | sandbox / dev / test / qa / prod | Required |
| `terraform_action` | **plan** · **apply** · **destroy** | Default: `plan` |
| `replace_targets` | space-separated resource addresses | Optional force-replace |
| `deploy_components` | true / false | Chain component deploy after apply |
| `skip_manual_approval` | true / false | Use with caution |

**Job flow:**

```
terraform-plan  ──▶  manual-approval (hitl-approvals env)  ──▶  terraform-apply
                                                            └──▶  terraform-destroy
```

**Key behaviours:**
- Authentication to Azure via **OIDC federation** — no stored Azure password
- Plan artifact is uploaded and re-used by apply (exact plan is applied)
- `apply` and `destroy` always require a human to approve in the `hitl-approvals` GitHub environment (unless `skip_manual_approval` is set)
- Role assignments are removed from state before `destroy` so they are never accidentally deleted
- After apply, Terraform outputs (Front Door URL, Container App URLs) are written to the job summary

**Required repository variables:** `PROJECT_NAME`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP`, `AZURE_TF_BACKEND_NAME`, `AZURE_TF_BACKEND_CONTAINER_NAME`, `AZURE_APIM_GATEWAY_URL`, `GRAFANA_OTLP_ENDPOINT`

**Required secrets:** `ORG_REPOS_INTERNAL_READ_ONLY`, `APIM_SUBSCRIPTION_KEY`

---

### `deploy-components.yml` — Application Deployment

**Trigger:** Manual (`workflow_dispatch`) or called after infrastructure apply

**Components (can be deployed individually or all at once):**

| Flag | Component | Container App name pattern |
|---|---|---|
| `deploy_agents_backend` | Python bot / agents backend | `ca-agents-backend-<suffix>` |
| `deploy_agents_frontend` | Agents frontend service | `ca-agents-frontend-<suffix>` |
| `deploy_ui` | React UI application | `ca-uiapp-<suffix>` |
| `deploy_all` | Deploys all three above | — |

**Job flow per component:**

```
preflight  ──▶  build-<component>  ──▶  deploy-<component>
(validates Azure targets,               (updates Container App
 reads Terraform outputs,                image + secrets + env vars)
 generates version tag)
```

**Image versioning:**
- If `version` input is empty: auto-generates `YYYY.MM.DD-<short-sha>-<run-id>`
- Images are pushed to **JFrog Artifactory** (`ARTIFACTORY_REPOSITORY_NAME`)

**Secrets injected into Container Apps at deploy time:**

| Container App | Secrets set |
|---|---|
| Agents Backend | `AZURE_CLIENT_SECRET`, `CLIENT_SECRET`, `MONGODB_URI` |
| Agents Frontend | `REDIS_URL` |
| UI | _(none beyond registry credentials)_ |

**Required secrets:** `ARTIFACTORY_USER_TOKEN` · `AZURE_CLIENT_SECRET` · `CLIENT_SECRET` · `MONGODB_URI` · `REDIS_URL`

---

### How the Two Workflows Relate

```
Option A — Infrastructure only:
  deploy-infrastructure.yml (apply)

Option B — Full deployment:
  deploy-infrastructure.yml (apply, deploy_components=true)
      └──▶ deploy-components.yml (deploy_all=true)

Option C — Code change only (infra unchanged):
  deploy-components.yml (select individual components)
```

---

### Document Index

| Document | Content |
|---|---|
| [handover-teams-bot-developer-portal.md](handover-teams-bot-developer-portal.md) | Teams Developer Portal navigation, app config, bot management, publishing |
| [handover-entra-id-app-registrations.md](handover-entra-id-app-registrations.md) | All three Entra ID App Registrations, secret rotation, Graph API permissions |
| [deploy-infrastructure.yml](.github/workflows/deploy-infrastructure.yml) | Terraform plan / apply / destroy workflow |
| [deploy-components.yml](.github/workflows/deploy-components.yml) | Docker build + Container App update workflow |
