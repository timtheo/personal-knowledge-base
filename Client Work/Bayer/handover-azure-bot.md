# Handover: Azure Bot Service

**Author:** Tim Kroll  
**Date:** 2026-05-07  
**Audience:** Colleagues taking over administration of the ITF DWP Agentic System

---

## 1. What Is Azure Bot Service?

**Azure Bot Service** is the managed Azure resource that registers a bot and connects it to communication channels such as Microsoft Teams. It acts as a relay between the channel (Teams) and the bot application code running in the backend — it does not contain any application logic itself.

The key responsibilities of the Azure Bot resource are:

- **Identity** — holds the reference to the Entra ID App Registration that authenticates the bot
- **Messaging endpoint** — knows the HTTPS URL where the bot application receives messages
- **Channel connections** — manages which channels (Teams, Web Chat, etc.) the bot is active on
- **Token service** — handles the OAuth token exchange between Teams and the bot backend

The relationship between the components is:

```
Microsoft Teams
      │
      │  sends message
      ▼
Azure Bot Service          ← Azure resource (relay + auth)
      │
      │  POST /api/messages
      ▼
Agents Backend             ← Container App (application logic)
(Container App)
```

---

## 2. How to Find the Azure Bot Resource

**Portal:** https://portal.azure.com

1. Search for **Azure Bot** in the top search bar, or navigate to the resource group directly
2. The bot resource is located in the same resource group as the other ITF DWP resources
3. The resource type is **Azure Bot** (not "Bot Channels Registration" — that is the deprecated predecessor)

---

## 3. Key Configuration Areas in the Azure Portal

### 3.1 Overview

The Overview blade shows:

| Field | What it means |
|---|---|
| **Resource name** | The internal Azure name of the bot resource |
| **Microsoft App ID** | The client ID of the linked Entra ID App Registration (ITF DWP Agentic System) |
| **Subscription / Resource Group** | Where the resource lives |

### 3.2 Configuration

This is the most important blade for day-to-day maintenance.

| Setting | Description |
|---|---|
| **Messaging endpoint** | The HTTPS URL where Teams delivers messages to the bot (e.g. `https://<container-app-url>/api/messages`). **Must be updated if the backend URL changes.** |
| **Microsoft App ID** | Client ID of the Entra ID App Registration. Links the bot to its identity. |
| **App Tenant ID** | The Entra ID tenant the bot belongs to. Must match the App Registration's tenant. |
| **App type** | Set to `UserAssignedMSI` — the bot uses a Managed Identity, not a client secret. |
| **Manage** link | Opens the linked App Registration directly in the Entra ID portal. |

> **If messages stop arriving:** The first thing to check is whether the messaging endpoint is correct and the Container App is running.

### 3.3 Channels

Lists all active channel connections. Our bot uses:

- **Microsoft Teams** — the primary channel

To view or modify the Teams channel:
1. **Channels** → select **Microsoft Teams**
2. The **Messaging** tab shows the cloud environment (must be set to `Commercial`)
3. The **Publish** tab shows the current publishing state of the Teams app
4. **Get bot embed code** — provides a direct Teams chat link for testing (`https://teams.microsoft.com/l/chat/0/0?users=28:...`)

> **Do not delete the Teams channel.** Doing so generates new keys and invalidates all stored user and conversation IDs in the bot backend.

### 3.4 Identity (Managed Identity Assignment)

The Azure Bot resource uses a **User-Assigned Managed Identity** for authentication. This means no client secret is stored — Azure manages the credential automatically.

To verify the Managed Identity is correctly assigned:
1. Navigate to the **App Service** (Container App) hosting the agents backend
2. **Settings** → **Identity** → **User assigned** tab
3. Confirm the Managed Identity for **ITF DWP Agentic System** is listed

If the Managed Identity is missing from the App Service, the bot will fail to authenticate with the Bot Framework and messages will not be processed.

---

## 4. Bot Identity — How Authentication Works

The Azure Bot authenticates using the **ITF DWP Agentic System** Entra ID App Registration via a User-Assigned Managed Identity. The flow is:

```
Container App (bot code)
    │
    │  requests token using Managed Identity
    ▼
Azure Instance Metadata Service (IMDS)
    │
    │  returns access token for the App Registration
    ▼
Bot Framework Connector Service
    │
    │  validates token, delivers/sends messages
    ▼
Microsoft Teams
```

The bot's configuration file (`config.py` or equivalent) must contain:

| Variable | Value |
|---|---|
| `MicrosoftAppType` | `UserAssignedMSI` |
| `MicrosoftAppId` | Client ID of ITF DWP Agentic System |
| `MicrosoftAppTenantId` | Our Entra ID tenant ID |
| `MicrosoftAppPassword` | *(leave empty — not used with Managed Identity)* |

> These values are injected as environment variables into the Container App at deploy time by `deploy-components.yml`.

---

## 5. Relationship to the Teams Developer Portal

The Azure Bot Service and the Teams App in the Developer Portal are two separate but linked things:

| | Azure Bot Service | Teams Developer Portal |
|---|---|---|
| **What it is** | Azure infrastructure resource | Teams app package configuration |
| **What it controls** | Bot identity, messaging endpoint, channel connections | App name, icons, scopes, manifest |
| **Where managed** | https://portal.azure.com | https://dev.teams.microsoft.com |
| **Bot ID referenced** | Holds the App Registration client ID | References the same Bot ID in `manifest.json` |

Both must be in sync: the **Bot ID** in the Teams app manifest must match the **Microsoft App ID** on the Azure Bot resource. A mismatch breaks message delivery.

---

## 6. Common Maintenance Tasks

### 6.1 Update the Messaging Endpoint

Required when the agents backend Container App URL changes (e.g. after infrastructure redeployment).

1. Azure Portal → Azure Bot resource → **Configuration**
2. Update the **Messaging endpoint** field with the new HTTPS URL (must end in `/api/messages`)
3. Click **Apply**
4. Also update the endpoint in the Teams Developer Portal: **Tools** → **Bot management** → select the bot → update endpoint

### 6.2 Verify the Bot Is Receiving Messages

1. In the Azure Bot resource → **Channels** → **Microsoft Teams** → copy the **embed code** URL
2. Open the URL in a browser — this starts a 1:1 chat with the bot in Teams
3. Send a test message and confirm the bot responds
4. If there is no response: check the Container App logs and verify the messaging endpoint is correct

### 6.3 Check the Teams Channel Is Active

1. Azure Bot → **Channels**
2. Confirm **Microsoft Teams** is listed with status **Running**
3. If missing: **Add a channel** → **Microsoft Teams** → agree to terms → **Apply**

### 6.4 Rotate or Verify Bot Identity

Since a Managed Identity is used, there is no secret to rotate. If authentication errors appear (`401 Unauthorized` from the Bot Framework):

1. Confirm the Managed Identity is still assigned to the Container App (see Section 3.4)
2. Confirm `MicrosoftAppId` in the app config matches the client ID of **ITF DWP Agentic System**
3. Confirm `MicrosoftAppTenantId` matches the Entra tenant ID

### 6.5 Test the Bot Endpoint Directly

To confirm the messaging endpoint is reachable without going through Teams:

```bash
curl -I https://<container-app-url>/api/messages
```

Expected response: `HTTP 405 Method Not Allowed` (GET is not allowed; POST is). Any other response (404, 502) indicates the app is not running or the URL is wrong.

---

## 7. Architecture Summary

```
┌──────────────────────────────────────────────────────────────┐
│                     Microsoft Teams                          │
│          (user sends message to ITF DWP bot)                 │
└───────────────────────────┬──────────────────────────────────┘
                            │  HTTPS via Teams channel
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                   Azure Bot Service                          │
│  Microsoft App ID  =  ITF DWP Agentic System (client ID)    │
│  App Type          =  UserAssignedMSI                        │
│  Messaging endpoint → https://<frontdoor>/api/messages       │
│  Channel           → Microsoft Teams (Commercial)            │
└───────────────────────────┬──────────────────────────────────┘
                            │  POST /api/messages
                            ▼
┌──────────────────────────────────────────────────────────────┐
│           Azure Front Door  (WAF + routing)                  │
└───────────────────────────┬──────────────────────────────────┘
                            │  private endpoint
                            ▼
┌──────────────────────────────────────────────────────────────┐
│        Agents Backend Container App  (Python bot code)       │
│  Auth: Managed Identity → ITF DWP Agentic System             │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Important Values

| Item | Where to find it |
|---|---|
| **Microsoft App ID** | Azure Portal → Azure Bot → Overview or Configuration |
| **App Tenant ID** | Azure Portal → Azure Bot → Configuration |
| **Messaging endpoint** | Azure Portal → Azure Bot → Configuration |
| **Teams embed code** | Azure Portal → Azure Bot → Channels → Microsoft Teams → Get bot embed code |
| **Managed Identity client ID** | Azure Portal → Managed Identities → ITF DWP Agentic System → Overview |

---

## 9. Links & References

| Resource | URL |
|---|---|
| Azure Portal | https://portal.azure.com |
| Teams Developer Portal | https://dev.teams.microsoft.com |
| MS Learn: Connect bot to Teams | https://learn.microsoft.com/en-us/azure/bot-service/channel-connect-teams?view=azure-bot-service-4.0 |
| MS Learn: Register a bot with Azure | https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration?view=azure-bot-service-4.0 |
| MS Learn: Managed Identity for bots | https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-authentication-federated-credential?view=azure-bot-service-4.0 |
| Entra ID handover doc | handover-entra-id-app-registrations.md |
| Teams Developer Portal handover doc | handover-teams-bot-developer-portal.md |
