# Handover: Microsoft Entra ID App Registrations

**Author:** Tim Kroll  
**Date:** 2026-05-07  
**Audience:** Colleagues taking over administration of the ITF DWP Agentic System

---

## 1. What Is an App Registration?

An **App Registration** in Microsoft Entra ID (formerly Azure Active Directory) is the identity record for an application. It defines:

- **Who the application is** (a unique Application/Client ID and Tenant ID)
- **How it authenticates** (client secrets, certificates, or federated credentials / Managed Identity)
- **What it is allowed to access** (API permissions — e.g. Microsoft Graph, other APIs)
- **Who is allowed to call it** (Expose an API / App Roles)

Every application that needs to interact with Azure or Microsoft 365 services needs an App Registration. Crucially, the App Registration is a **configuration record only** — it does not run code. The actual code runs elsewhere (App Service, Azure Function, etc.) and uses the credentials from the App Registration to authenticate.

### App Registration vs. Enterprise Application

When an App Registration is created, Entra ID automatically creates a linked **Enterprise Application** (also called a Service Principal) in the same tenant. The distinction:

| | App Registration | Enterprise Application |
|---|---|---|
| Purpose | Define the app's identity, credentials, permissions | Control who in the tenant can use the app; consent management |
| Where to find | Entra ID → App registrations | Entra ID → Enterprise applications |
| Who manages | App developer / DevOps | Tenant admin |

You will mostly work in **App registrations** for the tasks in this document.

---

## 2. How to Navigate to App Registrations

**Portal:** https://entra.microsoft.com (or https://portal.azure.com → Microsoft Entra ID)

1. Open the [Microsoft Entra admin center](https://entra.microsoft.com)
2. In the left sidebar: **Applications** → **App registrations**
3. Use the **All applications** tab to find registrations you do not own
4. Search by name (e.g. `ITF DWP`) to filter

---

## 3. Our Three App Registrations

### Overview of All Three

| App Registration | Role | Identity mechanism | Who / what uses it |
|---|---|---|---|
| **ITF DWP Agentic System** | Bot identity for the Azure Bot Service | User-Assigned Managed Identity | The Azure Bot resource + the bot application code (App Service) |
| **ITF DWP M365 Agentic System** | Permission delegation for Microsoft 365 / Graph API | Client secret or certificate | Developer-facing: Graph API calls, Teams/M365 integrations |
| **ITF DWP Azure Function** | Authentication guard for the Azure Function | Client secret (Easy Auth) | Azure Function App — protects its HTTP endpoints |

---

## 4. ITF DWP Agentic System

### What It Does

This App Registration is the **identity of the Azure Bot**. The Azure Bot Service uses it to:

- Authenticate outbound requests from the bot to the Bot Framework (proving "I am this bot")
- Receive inbound messages from Teams via the Bot Framework connector

It is linked to a **User-Assigned Managed Identity**, which means **no client secret is stored anywhere** — Azure manages the credential rotation automatically. The App Registration's client ID matches the `MicrosoftAppId` setting in the bot application configuration.

### Configuration Details

| Field | Where to find it | Used in |
|---|---|---|
| **Application (client) ID** | App registration → Overview | Bot config: `MicrosoftAppId` |
| **Directory (tenant) ID** | App registration → Overview | Bot config: `MicrosoftAppTenantId` |
| **Managed Identity resource** | Azure Portal → Managed Identities | Assigned to the App Service hosting the bot |

### Key Sections in the Portal

#### Overview
Shows the **Application (client) ID** and **Directory (tenant) ID**. These are the values wired into the Azure Bot resource and the application config.

#### Authentication
- **Application type:** Single-tenant (only our tenant's users can interact with the bot directly)
- No redirect URIs needed for a bot-only app (bots don't have browser login flows)

#### Certificates & Secrets
Because a Managed Identity is used, **no client secret or certificate should be present here** for production. The Managed Identity provides the credential transparently.

> **If you ever see an expiring client secret here:** It is either a remnant from initial setup or something was added manually. Investigate before deleting — check with the team first.

#### API Permissions
Minimal by design. The bot itself does not need Graph API permissions — those live in **ITF DWP M365 Agentic System**.

#### How the Managed Identity Connects

```
Azure App Service (bot code runs here)
    │
    ├── User Identity assigned: [Managed Identity resource]
    │         │
    │         └── client ID = Application (client) ID of ITF DWP Agentic System
    │
Azure Bot resource
    │
    └── Microsoft App ID = Application (client) ID of ITF DWP Agentic System
```

The App Service fetches tokens automatically using the Managed Identity — no secret, no rotation needed.

### Maintenance Tasks

| Task | Steps |
|---|---|
| Find the App ID | App registrations → ITF DWP Agentic System → Overview → Application (client) ID |
| Verify Managed Identity is still assigned | Azure Portal → App Service → Settings → Identity → User assigned tab |
| Check if bot config matches | App Service → Settings → Environment variables → `MicrosoftAppId` must match client ID above |

---

## 5. ITF DWP M365 Agentic System

### What It Does

This App Registration provides **elevated Microsoft 365 / Graph API permissions** used by the system to interact with Teams, SharePoint, Exchange, or other M365 services. It is the identity used when the bot (or a background service) needs to call Graph API on behalf of the application (not on behalf of a specific user).

Examples of what it might be used for:
- Sending proactive messages to Teams channels or users via Graph
- Reading/writing data in SharePoint or Exchange
- Managing Teams memberships or calendar entries

### Permission Model: Application vs. Delegated

| Type | What it means | When used |
|---|---|---|
| **Application permissions** | The app acts as itself, no user involved | Background jobs, proactive messaging |
| **Delegated permissions** | The app acts on behalf of a signed-in user | User-facing features with SSO |

> Most bot-to-Graph scenarios use **Application permissions** with admin consent.

### Key Sections in the Portal

#### Overview
Application (client) ID and Directory (tenant) ID — used in the code that makes Graph API calls (e.g., `AZURE_CLIENT_ID` or equivalent environment variable).

#### Certificates & Secrets
This registration uses a **client secret** (or certificate) to authenticate.

> **Critical: Client secrets expire.** The default maximum lifetime is 24 months, but Microsoft recommends 12 months or less. If the secret expires, all Graph API calls from the bot will silently fail.

To view existing secrets:
1. App registrations → ITF DWP M365 Agentic System → **Certificates & secrets**
2. The **Client secrets** tab shows the description, creation date, and expiry date
3. The **Value** of the secret is only visible immediately after creation — it cannot be retrieved later

**Secret rotation procedure:**
1. Create a **new** secret (do not delete the old one yet)
2. Update the secret value in the application config / Key Vault
3. Verify the app is working correctly with the new secret
4. Delete the old secret

> **Best practice:** Store the secret in **Azure Key Vault** and reference it from the App Service config as a Key Vault reference (`@Microsoft.KeyVault(...)`). Never store it in plain text in environment variables or source code.

#### API Permissions

This is where the Graph API access is configured. Each permission row shows:

| Column | Meaning |
|---|---|
| **API / Permissions name** | E.g. `Microsoft Graph / User.Read.All` |
| **Type** | `Delegated` (acts as user) or `Application` (acts as itself) |
| **Status** | `Granted for [tenant]` = admin has consented; `Not granted` = needs admin action |

**To add a new permission:**
1. App registrations → ITF DWP M365 Agentic System → **API permissions** → **Add a permission**
2. Select **Microsoft Graph** (or another API)
3. Choose **Application permissions** or **Delegated permissions**
4. Search for and select the required permission → **Add permissions**
5. Click **Grant admin consent for [tenant]** — this requires a Global Admin or Privileged Role Admin

> **Never add more permissions than needed.** Principle of least privilege applies here.

#### Authentication
For application-only (background) scenarios: **no redirect URIs** needed. If delegated/SSO flows are used, the redirect URIs of the app (e.g., the bot's reply URL) must be listed here.

---

## 6. ITF DWP Azure Function

### What It Does

This App Registration protects the **Azure Function** using Entra ID authentication, commonly known as **Easy Auth**. When a caller hits an HTTP endpoint of the Azure Function, Azure validates the bearer token in the request against this App Registration before the function code even runs.

This means:
- Only callers who can obtain a valid token for this App Registration are allowed through
- The function code does not need to implement its own authentication logic
- Access control is enforced at the Azure platform level

### How Easy Auth Works

```
Caller (browser / API client / another Azure service)
    │
    │  HTTP request + Authorization: Bearer <token>
    ▼
Azure Function App (Easy Auth layer)
    │
    ├── Validates token against ITF DWP Azure Function App Registration
    │       (checks: correct audience, valid signature, not expired, tenant match)
    │
    ├── Token valid → forwards request to function code
    └── Token invalid → returns HTTP 401 Unauthorized
```

### Key Sections in the Portal

#### Overview
- **Application (client) ID** — this is the **audience** (`aud` claim) that valid tokens must target
- **Application ID URI** — often in the format `api://<client-id>` — this is what callers must request a token for

#### Authentication
- **Platform:** Web
- **Redirect URI:** `https://<function-app-name>.azurewebsites.net/.auth/login/aad/callback`
  - Required for browser-based login flows (if any)
  - The path `/.auth/...` is the Easy Auth built-in endpoint

#### Certificates & Secrets
Easy Auth uses a client secret stored as an **application setting** (`MICROSOFT_PROVIDER_AUTHENTICATION_SECRET`) in the Azure Function App.

> This secret **will expire** — check the expiry date and rotate it before it does. The rotation procedure is the same as for ITF DWP M365 Agentic System (see Section 5).

When you rotate the secret:
1. Create a new secret here in the App Registration
2. Update the value in the Azure Function App: **Authentication** → **Edit** the Microsoft identity provider → update the client secret → **Save**

#### Expose an API
If other applications need to obtain tokens **for** this function (as a resource), their allowed client IDs are listed here under **Authorized client applications**. Check this if a service account or another app is supposed to call the function but is getting 401 errors.

#### App Roles (optional)
If the function uses role-based access control (e.g., only `Admin` role can call certain endpoints), roles are defined here. Assignments are managed in the Enterprise Application linked to this registration.

---

## 7. Key Concepts Reference

### Credentials Comparison

| Credential type | Security | Notes |
|---|---|---|
| **Client secret** | Low–Medium | Expires; easy to set up; avoid in production if possible |
| **Certificate** | High | Recommended for production; must manage cert lifecycle |
| **Managed Identity** | Highest | No credential at all; Azure manages everything; only usable within Azure |

### What Happens When a Secret Expires

The application will receive HTTP 401 / `AADSTS7000215: Invalid client secret` errors. The fix is to rotate the secret (create new → update config → delete old). There is **no automatic renewal** — you must proactively monitor expiry dates.

### Admin Consent

**Application permissions** in Graph API require a Global Admin (or Privileged Role Admin) to grant consent. If a new permission is added but not consented, the app will receive `403 Forbidden` / `AADSTS65001` errors when trying to use that permission.

---

## 8. Common Maintenance Tasks

### 8.1 Find an App Registration

1. https://entra.microsoft.com → **App registrations** → **All applications**
2. Search for `ITF DWP`

### 8.2 Check All Secret Expiry Dates (do this periodically)

1. Open each App Registration → **Certificates & secrets** → **Client secrets**
2. Note the **Expires** column
3. Rotate any secret expiring within 30 days (see 8.3)

### 8.3 Rotate a Client Secret

1. App Registration → **Certificates & secrets** → **Client secrets** → **New client secret**
2. Give it a descriptive name (e.g., `prod-2026-05`) and set expiry (max 24 months, prefer 12)
3. **Copy the Value immediately** — it is never shown again
4. Store the new secret in Key Vault or update the environment variable
5. Verify the application is working with the new secret
6. Delete the old secret from the App Registration

### 8.4 Add a Graph API Permission

1. App Registration (ITF DWP M365 Agentic System) → **API permissions** → **Add a permission**
2. **Microsoft Graph** → **Application permissions** → search and select → **Add permissions**
3. Click **Grant admin consent for [tenant]** (requires admin role)

### 8.5 Add a New Authorized Caller to the Azure Function

1. App Registration (ITF DWP Azure Function) → **Expose an API** → **Add a client application**
2. Enter the client ID of the calling app and select the scope(s) to authorize → **Add application**

### 8.6 Verify a Managed Identity Assignment

1. Azure Portal → **App Services** → select the bot app service
2. **Settings** → **Identity** → **User assigned** tab
3. Confirm the Managed Identity for `ITF DWP Agentic System` is listed
4. If missing: **Add** → select the Managed Identity resource → **Add**

---

## 9. Architecture Summary — How All Three Connect

```
┌─────────────────────────────────────────────────────────┐
│                   Microsoft Teams                       │
└───────────────────────┬─────────────────────────────────┘
                        │ message
                        ▼
┌─────────────────────────────────────────────────────────┐
│               Azure Bot Service                         │
│  uses: ITF DWP Agentic System (App ID)                  │
│        + Managed Identity (no secret needed)            │
└───────────────────────┬─────────────────────────────────┘
                        │ POST /api/messages
                        ▼
┌─────────────────────────────────────────────────────────┐
│        App Service (Python bot application)             │
│                                                         │
│  MicrosoftAppId    = ITF DWP Agentic System client ID   │
│  MicrosoftAppType  = UserAssignedMSI                    │
│                                                         │
│  Graph API calls   = ITF DWP M365 Agentic System        │
│                       (client ID + secret/cert)         │
└───────────────────────┬─────────────────────────────────┘
                        │ calls HTTP endpoint
                        ▼
┌─────────────────────────────────────────────────────────┐
│              Azure Function App                         │
│  Protected by Easy Auth:                                │
│    App Registration = ITF DWP Azure Function            │
│    validates bearer token before code runs              │
└─────────────────────────────────────────────────────────┘
```

---

## 10. Quick Reference: Important IDs

| App Registration | Application (client) ID | Used as / in |
|---|---|---|
| ITF DWP Agentic System | *(find in portal → Overview)* | `MicrosoftAppId` in bot config; Azure Bot resource |
| ITF DWP M365 Agentic System | *(find in portal → Overview)* | Graph API auth client ID; code env variable |
| ITF DWP Azure Function | *(find in portal → Overview)* | Easy Auth audience; `MICROSOFT_PROVIDER_AUTHENTICATION_SECRET` in Function App |

**Tenant ID** (same for all three, it's our M365 tenant):  
Azure Portal → Microsoft Entra ID → Overview → **Tenant ID**

---

## 11. Links & References

| Resource | URL |
|---|---|
| Microsoft Entra admin center | https://entra.microsoft.com |
| Azure Portal | https://portal.azure.com |
| MS Learn: Register a Bot with Azure | https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration?view=azure-bot-service-4.0 |
| MS Learn: Add and manage app credentials | https://learn.microsoft.com/en-us/entra/identity-platform/how-to-add-credentials |
| MS Learn: Configure Entra auth for App Service / Functions | https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad |
| MS Learn: Microsoft Graph permissions reference | https://learn.microsoft.com/en-us/graph/permissions-reference |
| MS Learn: What are Managed Identities? | https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview |
