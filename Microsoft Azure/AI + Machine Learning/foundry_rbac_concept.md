# Azure AI Foundry — RBAC Reference

## Important: Two Role Layers

There are two distinct role layers for AI Foundry. They are **not interchangeable**:

| Layer | Roles | When to use |
|---|---|---|
| **Foundry roles** | Foundry User, Foundry Project Manager, Foundry Account Owner, Foundry Owner | Access to Foundry hubs and projects (build agents, use playgrounds, manage projects) |
| **Cognitive Services roles** | Cognitive Services OpenAI User/Contributor, Cognitive Services Contributor | Direct access to the underlying `azurerm_cognitive_account` endpoint (API calls, model deployments, key management) |

---

## Foundry-Specific Built-In Roles

Assigned at the **Foundry resource or project scope**.

> **Note:** These roles were recently renamed. Use the GUIDs in Terraform/CLI to avoid issues during the portal rollout.

| Role | Old Name | GUID | Data Plane (build) | Control Plane (manage) |
|---|---|---|---|---|
| **Foundry User** | Azure AI User | `53ca6127-db72-4b80-b1b0-d745d6d5456d` | YES | Reader only |
| **Foundry Project Manager** | Azure AI Project Manager | `eadc314b-1a2d-4efa-be10-5d325db5065e` | YES | Create projects, publish agents, assign Foundry User |
| **Foundry Account Owner** | Azure AI Account Owner | `e47c6f54-e4a2-4754-9501-8e0985b135e1` | NO | Full control plane (create accounts, deploy models) |
| **Foundry Owner** | Azure AI Owner | `c883944f-8b7b-4483-af10-35834be79c4a` | YES | Full control plane + full data plane |

Key points:
- **Foundry Account Owner does not grant data-plane access** — a platform admin who also needs to build must assign themselves Foundry User as well, or use Foundry Owner instead.
- **Foundry Owner** is the most privileged role — use sparingly.
- Do **not** use the legacy "Azure AI Developer" role for Foundry work — it targets Azure Machine Learning workspaces, not Foundry projects.

---

## Recommended Assignments by Persona

| Persona | Role | Scope |
|---|---|---|
| IT Admin / Platform Team | Owner | Subscription |
| Platform Engineer / Manager | Foundry Account Owner | Foundry resource |
| Team Lead | Foundry Project Manager | Foundry resource |
| Developer / Data Scientist | Foundry User | Foundry project |
| CI/CD Service Principal | Foundry Owner (or Account Owner + User) | Foundry resource |
| Read-Only / Auditor | Reader | Resource group or Foundry resource |
| Fine-Tuning User | Foundry Owner (combines both needed roles in one) | Foundry resource |

---

## Cognitive Services Roles (Direct API Access)

Assigned at the **`azurerm_cognitive_account` resource scope** (or resource group for Contributor).

These apply when accessing the AI Services endpoint directly — e.g. an application calling the OpenAI API with Entra ID auth, bypassing the Foundry project layer.

| Role | Entra ID inference | Deploy models | Manage keys | Create resources |
|---|---|---|---|---|
| **Cognitive Services OpenAI User** | YES | NO | NO | NO |
| **Cognitive Services OpenAI Contributor** | YES | YES | NO | NO |
| **Cognitive Services Contributor** | NO (key-based only) | YES | YES | YES |
| **Cognitive Services Usages Reader** | — | — | — | — (quota visibility only, subscription scope) |

Common combinations:

| Goal | Assign |
|---|---|
| App calling the API with Entra ID | Cognitive Services OpenAI User |
| Developer who also manages deployments | Cognitive Services OpenAI Contributor |
| Platform admin managing the resource | Cognitive Services Contributor (resource group scope) |
| Anyone needing quota visibility in the portal | + Cognitive Services Usages Reader (subscription scope) |

> **Note:** Cognitive Services roles do **not** grant access to Foundry project data-plane features. Assign Foundry roles for that.

---

## Managed Identity Role Assignments (Resource-to-Resource)

This module deploys a system-assigned identity on both the `azurerm_cognitive_account` and the `azurerm_ai_foundry` hub. The following role assignments are required for the Foundry workflow to function correctly.

> These are **not yet in the Terraform module** — they need to be added as `azurerm_role_assignment` resources.

| Identity | Role | Scope | Purpose |
|---|---|---|---|
| `azurerm_cognitive_account` (Foundry account) | Storage Blob Data Contributor | Storage account | Allows Foundry to read/write blobs (fine-tuning datasets, "on your data") |
| `azurerm_ai_foundry` (hub) | Storage Blob Data Contributor | Storage account | Hub access to the associated storage |
| `azurerm_ai_foundry` (hub) | Key Vault Secrets User | Key Vault | Hub access to secrets (connections, credentials) |
| AI Foundry project managed identity | Foundry User | Foundry resource (cognitive account) | Allows the project runtime to call Foundry APIs |

Reference the principal IDs from module outputs:
- `module.ai_foundry.ai_foundry_account_principal_id`
- `module.ai_foundry.ai_foundry_hub_principal_id`

---

## Sources

- [RBAC in Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/rbac-ai-foundry)
- [Azure OpenAI RBAC](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/role-based-access-control)
