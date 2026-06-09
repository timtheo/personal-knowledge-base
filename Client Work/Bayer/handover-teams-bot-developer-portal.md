# Handover: Microsoft Teams Bot — Developer Portal Guide

**Author:** Tim Kroll  
**Date:** 2026-05-07  
**Audience:** Colleagues taking over Teams Bot maintenance and configuration

---

## 1. What Is This About?

We have deployed an **Azure Bot** (hosted on Azure) that is integrated into **Microsoft Teams** as a Teams App. The Teams side of this integration — i.e. what users in Teams see, the app name, icons, scopes, and the binding between our Azure Bot and Teams — is managed through the **Microsoft 365 Developer Portal for Teams**.

This document explains:
- What the Developer Portal is and how to access it
- How our Teams App is structured
- How to navigate and modify the app in the portal
- How the Azure Bot and the Teams App relate to each other

---

## 2. Key Concepts

### 2.1 Azure Bot Service (Azure Side)

The bot **logic** runs as a web application on Azure (App Service / Azure AI Foundry). The **Azure Bot resource** in the Azure Portal acts as a relay: it registers the bot's messaging endpoint, handles authentication, and connects to channels (Teams, Web Chat, etc.).

| Azure resource | Role |
|---|---|
| App Service / Container App | Hosts the Python bot application |
| Azure Bot | Registers the bot endpoint; manages channel connections |
| Microsoft Entra ID App Registration | Identity of the bot (App ID + Secret) |

### 2.2 Teams App (Developer Portal Side)

A **Teams App** is a package that tells Microsoft Teams how to display and launch our bot. It contains:

- `manifest.json` — JSON file describing the app (name, description, icons, capabilities)
- Two icons (color 192×192 px, outline 32×32 px in PNG format)

The Teams App is **not** the bot itself — it is the wrapper that connects Teams to the bot. The connection is made by referencing the bot's **App ID** (from Entra ID or Bot management) inside the manifest.

---

## 3. The Microsoft 365 Developer Portal

### 3.1 What Is It?

The **Developer Portal for Teams** is the web-based management UI for Teams apps. It lets you create, configure, validate, and publish Teams apps — without needing to manually edit JSON files.

**URL:** https://dev.teams.microsoft.com

Sign in with your Microsoft 365 / organizational account (the same tenant where the bot is deployed).

### 3.2 Navigation Overview

The left-hand sidebar has three main areas:

```
├── Apps           ← list of all your Teams apps
├── Tools          ← utility tools (Bot management, Adaptive Cards, etc.)
└── Home           ← portal dashboard / announcements
```

---

## 4. Finding Our App

1. Open https://dev.teams.microsoft.com and sign in.
2. Select **Apps** in the left pane.
3. Find the app by name. Click it to open the app management view.

> **Note:** If you cannot see the app, you may not be listed as an owner. Ask an existing owner to add you (see Section 7.1).

---

## 5. App Management — Section by Section

Once you open the app, you will see tabs on the left sidebar grouped into **Configure**, **Overview**, **Advanced**, **Develop**, and **Publish**.

### 5.1 Overview

| Sub-section | What you'll find |
|---|---|
| **Dashboard** | App ID, version, manifest version, active user count, validation status |
| **Analytics** | Usage and engagement metrics |

The **App ID** shown here is the Teams App ID (a GUID). This is **different** from the Bot's App ID (Entra ID).

### 5.2 Configure

This is where you make most changes to the app definition.

#### Basic Information
- **App names** (short name shown in Teams, full name)
- **Descriptions** (short ≤ 80 chars, long ≤ 4000 chars)
- **Version** (increment when publishing updates, e.g. `1.0.0` → `1.0.1`)
- **Developer information** and **App URLs** (Privacy policy, Terms of use)
- **Application (client) ID** — the Entra ID App Registration ID for the bot

#### Branding
Upload PNG icons:
- **Color icon:** 192×192 px (shown in app store, chat)
- **Outline icon:** 32×32 px with transparent background (shown in sidebar)

#### App Features
This section lists the capabilities the app exposes. Our app has the **Bot** capability configured here.

To view or edit the Bot feature:
1. Click **App features** → **Bot**
2. You will see:
   - **Bot ID** — the ID of the registered bot (from Bot management or Entra ID)
   - **Scopes** — where users can interact with the bot:
     - `Personal` — 1:1 chat with the bot
     - `Team` — bot is added to a Teams channel
     - `Group chat` — bot added to a group chat
   - **Bot commands** (optional) — suggested commands shown to users

> **Important:** The Bot ID here must match the App Registration (client) ID used by the running bot application. Changing it breaks the connection.

#### App Package Editor
A file-level editor for the app package contents. You can directly edit `manifest.json`, swap icons, or add declarative agent files. Use this only if you need fine-grained control — basic changes are better done through the UI sections above.

#### Permissions
Defines what the app is allowed to access:
- Device permissions (camera, microphone, location)
- Team permissions (read team info, etc.)
- Chat/Meeting permissions
- User permissions

Only add permissions that the bot actually needs.

#### Single Sign-On (SSO)
If the bot uses SSO, the **Application ID URI** from Entra ID must be set here.

> **Note:** SSO only works if the bot is registered via **Microsoft Entra ID** (App Registration), not if it was created purely through the Developer Portal's Bot management tool.

#### Domains
List every external domain the bot loads content from (e.g., `*.yourdomain.com`). Teams blocks requests to unlisted domains.

---

### 5.3 Advanced

| Sub-section | Purpose |
|---|---|
| **Owners** | Manage who can edit this app registration (Administrator / Operative roles) |
| **App content** | Loading indicator, full-screen mode, supported channel types |
| **Environments** | Define environment variables for dev/staging/prod packages |
| **Admin settings** | Block app by default; customize display name/description per tenant |

#### Adding an Owner (important for team handover)
1. **Advanced** → **Owners** → **Add owners**
2. Enter the colleague's name, select from the dropdown
3. Assign role: **Operative** (can edit config) or **Administrator** (can also add/remove owners, delete app)
4. Click **Add**

---

### 5.4 Develop

Links to open the app in **Microsoft 365 Agents Toolkit** (VS Code extension). Only relevant if the team uses the toolkit for local development.

---

### 5.5 Publish

#### App Package
Downloads the current app package as a `.zip` file containing `manifest.json` and icons. Use this to manually sideload the app or submit it elsewhere.

#### App Validation
Runs automated tests against Microsoft's Teams Store review criteria. Recommended before every publish:
1. **Publish** → **App validation** → **Get Started** → **Start Validation**
2. Wait for completion. Fix any **Errors** (required); address **Warnings** (recommended).

#### Publish to Org
Makes the app available to all users in your Microsoft 365 tenant via the Teams app store (org-scoped). Requires Teams Admin approval.
1. **Publish** → **Publish to org** → **Publish your App**

#### Publish to Store
Submits to the public Microsoft Teams Store (if applicable — most enterprise bots skip this).

---

## 6. Tools Section

In the left sidebar, **Tools** provides standalone utilities:

### 6.1 Bot Management
Manages bot registrations independent of any specific app.

- **+ New Bot** — creates a new bot registration (single-tenant by default)
- Each bot has a **Bot ID** (GUID) and a configurable **Endpoint URL** (the HTTPS URL where the bot application receives messages)
- Click an existing bot to edit its endpoint URL or messaging channels

> **When to use:** If the bot's Azure App Service URL changes (e.g., after a redeployment to a new hostname), update the **Endpoint URL** here.

### 6.2 Other Tools (less relevant for our bot)
| Tool | Purpose |
|---|---|
| Scene studio | Design Together Mode scenes for meetings |
| Adaptive Cards editor | Preview/test Adaptive Card JSON |
| Identity platform management | Register apps with Entra ID |
| Agent Identity Blueprint | Create reusable agent configurations for Agent 365 |

---

## 7. Common Maintenance Tasks

### 7.1 Add a Colleague as Owner
1. Open the app → **Advanced** → **Owners** → **Add owners**
2. Enter name, select role, click **Add**

### 7.2 Update the Bot's Messaging Endpoint
If the Azure App Service URL changes:
1. **Tools** → **Bot management** → select the bot
2. Update the **Endpoint URL** field → save

Also update it in the **Azure Bot** resource in the Azure Portal (under **Configuration** → **Messaging endpoint**).

### 7.3 Update App Name, Description, or Icons
1. Open the app → **Configure** → **Basic information** or **Branding**
2. Make changes → save
3. Increment the **Version** number
4. Run **App validation** → fix issues
5. Re-publish (**Publish to org**)

### 7.4 Add or Restrict Bot Scopes
1. Open the app → **Configure** → **App features** → **Bot**
2. Check/uncheck **Personal**, **Team**, **Group chat**
3. Save → re-publish

### 7.5 Download the Current App Package
1. Open the app → **Publish** → **App package** → **Download app package**
2. Unzip to inspect `manifest.json`

### 7.6 Sideload/Test the App in Teams
1. Download the app package (`.zip`) as above
2. In Teams: **Apps** → **Manage your apps** → **Upload an app** → **Upload a custom app**
3. Select the `.zip` file — the bot is now available for testing

---

## 8. Architecture Summary

```
┌─────────────────────────────────┐
│     Microsoft Teams Client      │
│  (user sends a message to bot)  │
└────────────┬────────────────────┘
             │ Teams Channel
             ▼
┌─────────────────────────────────┐
│       Azure Bot Resource        │  ← configured in Azure Portal
│  (Bot ID / Entra App ID + Secret)│
│  Messaging endpoint → App URL   │
└────────────┬────────────────────┘
             │ HTTPS POST to /api/messages
             ▼
┌─────────────────────────────────┐
│   Python Bot App (App Service)  │  ← deployed via GitHub Actions + Terraform
│   (Azure AI Foundry / FastAPI)  │
└─────────────────────────────────┘

Teams App Package (managed in Developer Portal):
  manifest.json  →  references Bot ID  →  links Teams UI to Azure Bot
```

---

## 9. Important IDs to Know

| ID | Where to find it | What it's for |
|---|---|---|
| **Teams App ID** | Developer Portal → App → Overview → App ID | Identifies the Teams app package |
| **Bot ID / Entra App ID** | Azure Portal → App Registration → Application (client) ID | Identifies the bot in all auth flows |
| **Client Secret** | Azure Portal → App Registration → Certificates & Secrets | Bot authenticates to Azure using this (store in Key Vault!) |
| **Bot Endpoint URL** | Developer Portal → Tools → Bot management → Endpoint URL | Where Teams sends messages to our bot |

---

## 10. Links & References

| Resource | URL |
|---|---|
| Teams Developer Portal | https://dev.teams.microsoft.com |
| Microsoft Learn: Manage apps in Developer Portal | https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/build-and-test/manage-your-apps-in-developer-portal |
| Microsoft Learn: Connect bot to Teams channel | https://learn.microsoft.com/en-us/azure/bot-service/channel-connect-teams?view=azure-bot-service-4.0 |
| Microsoft Learn: Bots in Teams | https://learn.microsoft.com/en-us/microsoftteams/platform/bots/what-are-bots |
| Microsoft Learn: App manifest reference | https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema |
| Azure Portal | https://portal.azure.com |
