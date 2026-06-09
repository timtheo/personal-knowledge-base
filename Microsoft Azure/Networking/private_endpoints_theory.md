# Azure Private Endpoints — Theory & Architecture

---

## The Problem Private Endpoints Solve

By default, every Azure PaaS service (Key Vault, Storage, CosmosDB, Service Bus, etc.) has a **public endpoint** — a DNS name that resolves to a public IP address. Traffic from your VNet to these services crosses the public internet, even if both are in Azure.

```
Container App (10.0.2.5)
        │
        │  request to my-kv.vault.azure.net
        ▼
   Azure DNS resolves → 52.x.x.x  (PUBLIC IP)
        │
        ▼  ← travels over the internet
   Key Vault service
```

This means:
- The service is reachable from **anywhere on the internet** (only protected by auth)
- You cannot block access purely at the **network level**
- Compliance frameworks (PCI DSS, HIPAA, ISO 27001) often require data to never traverse public networks

Private Endpoints fix this by giving the PaaS service a **private IP address inside your own VNet**.

---

## What a Private Endpoint Actually Is

A private endpoint is a **network interface card (NIC) that Azure injects into your subnet**. That NIC gets a private IP from your subnet's address space — in our case from `snet-private-endpoints` (`10.0.1.0/24`).

```
Your VNet (10.0.0.0/16)
│
└── snet-private-endpoints (10.0.1.0/24)
         │
         ├── 10.0.1.4  ← private endpoint NIC for Key Vault
         ├── 10.0.1.5  ← private endpoint NIC for Storage
         └── 10.0.1.6  ← private endpoint NIC for CosmosDB
```

Each private endpoint consumes **exactly one private IP** from that subnet. A `/24` gives you ~251 usable addresses — room for a lot of services.

The underlying technology is called **Azure Private Link**. It creates a one-way tunnel from your NIC into Microsoft's backbone, terminating at the specific service instance. Traffic never leaves the Microsoft network.

---

## The 5 Components and How They Connect

Every private endpoint setup requires these five pieces working together:

```
┌─────────────────────────────────────────────────────────────────┐
│  1. PaaS Resource         e.g. azurerm_key_vault                │
│     - Has a public endpoint (to be disabled after setup)        │
│     - Exposes itself as a Private Link resource                  │
└──────────────────────────────┬──────────────────────────────────┘
                               │  Private Link connection
┌──────────────────────────────▼──────────────────────────────────┐
│  2. Private Endpoint         azurerm_private_endpoint            │
│     - A NIC injected into snet-private-endpoints                 │
│     - Gets a private IP (e.g. 10.0.1.4)                         │
│     - Holds the Private Service Connection to resource (1)       │
└──────────────────────────────┬──────────────────────────────────┘
                               │  DNS record added automatically
┌──────────────────────────────▼──────────────────────────────────┐
│  3. Private DNS Zone         azurerm_private_dns_zone            │
│     - One zone per service type (e.g. privatelink.vaultcore.azure.net) │
│     - Holds an A record: my-kv → 10.0.1.4                       │
└──────────────────────────────┬──────────────────────────────────┘
                               │  attached to VNet
┌──────────────────────────────▼──────────────────────────────────┐
│  4. VNet DNS Link            azurerm_private_dns_zone_virtual_network_link │
│     - Links the Private DNS Zone to your VNet                    │
│     - Makes the zone visible to all resources in the VNet        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  5. NSG on snet-private-endpoints (already in our VNet module)  │
│     - Controls which subnets can reach the private endpoint NICs │
│     - Currently: Allow VirtualNetwork inbound                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## DNS — The Critical and Confusing Part

This is the most important concept to get right. The DNS resolution is what makes your app automatically use the private IP instead of the public one — **without changing any application code**.

### Without private endpoint
```
App resolves: my-kv.vault.azure.net
         │
         ▼ Azure Public DNS
         52.x.x.x  ← public IP, traffic goes to internet
```

### With private endpoint (what actually happens — two CNAME hops)
```
App resolves: my-kv.vault.azure.net
         │
         ▼ Step 1: Azure Public DNS returns a CNAME
         my-kv.privatelink.vaultcore.azure.net   ← CNAME redirect
         │
         ▼ Step 2: Private DNS Zone (linked to VNet) resolves the privatelink name
         10.0.1.4   ← private IP of the private endpoint NIC
         │
         ▼ Traffic stays inside the VNet
         Key Vault service (through Microsoft backbone)
```

The CNAME to `*.privatelink.*` is added to the **public** Azure DNS automatically when you create the private endpoint. The VNet-linked Private DNS Zone then **overrides** the resolution of the `privatelink` name to the private IP.

**If you forget to create the Private DNS Zone or the VNet link**, step 2 fails and the `privatelink` name still resolves to the public IP — your private endpoint exists but is never used.

### Each service type has its own DNS zone

| Service | Subresource | Private DNS Zone |
|---|---|---|
| Key Vault | `vault` | `privatelink.vaultcore.azure.net` |
| Storage (blob) | `blob` | `privatelink.blob.core.windows.net` |
| Storage (file) | `file` | `privatelink.file.core.windows.net` |
| CosmosDB (SQL) | `Sql` | `privatelink.documents.azure.com` |
| Service Bus | `namespace` | `privatelink.servicebus.windows.net` |
| Container Registry | `registry` | `privatelink.azurecr.io` |
| Azure OpenAI | `account` | `privatelink.cognitiveservices.azure.com` |

One Private DNS Zone per service type can cover **all instances** of that service — the A records for each instance are just added into the same zone.

---

## Full Request Flow (Container App → Key Vault)

```
Container App (10.0.2.5, snet-container-apps)
  │
  │  1. GET https://my-kv.vault.azure.net/secrets/db-password
  │
  ▼
Azure DNS resolver (168.63.129.16 — Azure magic IP, always available)
  │
  │  2. my-kv.vault.azure.net
  │     → CNAME: my-kv.privatelink.vaultcore.azure.net
  │
  ▼
Private DNS Zone (privatelink.vaultcore.azure.net, linked to VNet)
  │
  │  3. my-kv.privatelink.vaultcore.azure.net → A: 10.0.1.4
  │
  ▼
NSG on snet-private-endpoints
  │
  │  4. Source: VirtualNetwork, Destination: VirtualNetwork → ALLOW
  │
  ▼
Private Endpoint NIC (10.0.1.4, snet-private-endpoints)
  │
  │  5. Microsoft backbone — never leaves Azure network
  │
  ▼
Key Vault service instance
  │
  │  6. Firewall check: is public access disabled? Yes.
  │     Is this coming via private endpoint? Yes → ALLOW
  │
  ▼
Secret returned through same path back to Container App
```

---

## Locking Down the Public Endpoint

Creating a private endpoint alone is not enough — the public endpoint still exists and is still reachable from the internet. You must also **disable public access** on the PaaS resource's own firewall:

```
Before:  internet → public IP → Key Vault ✅  (bad)
         VNet     → private IP → Key Vault ✅

After:   internet → public IP → Key Vault ❌  (blocked by service firewall)
         VNet     → private IP → Key Vault ✅
```

In Terraform this is expressed via `default_action = "Deny"` in the `network_acls` block on the resource. Every service has a slightly different way to express this.

**Important gotcha for CI/CD pipelines:** If you disable public access on Key Vault and your Terraform pipeline runs from GitHub Actions (public internet), Terraform can no longer manage Key Vault secrets. You have two options:
1. Run Terraform from a **self-hosted runner** inside the VNet
2. Keep a specific **IP allowlist** for your pipeline's outbound IP

---

## How This Maps to the Project Module Structure

```
azurerm_resource_group.networking
        │
        └── module.virtual_network
                │
                ├── vnet (10.0.0.0/16)
                │
                ├── snet-private-endpoints (10.0.1.0/24)   ← private endpoint NICs go here
                │       └── nsg-snet-private-endpoints       (allows VNet inbound)
                │
                ├── snet-container-apps (10.0.2.0/23)       ← Container Apps Environment
                │       └── nsg-snet-container-apps          (allows VNet inbound)
                │
                └── AzureBastionSubnet (10.0.4.0/26)        ← Bastion host goes here
                        └── nsg-AzureBastionSubnet           (mandatory Bastion rules)

For each PaaS service (Key Vault, Storage, etc.):
        ├── azurerm_key_vault                               ← the PaaS resource
        ├── azurerm_private_endpoint                        ← NIC in snet-private-endpoints
        ├── azurerm_private_dns_zone                        ← one per service type
        ├── azurerm_private_dns_zone_virtual_network_link   ← links zone to the VNet
        └── (A record auto-created when private endpoint is provisioned)
```

---

## Key Things to Remember Before Writing Code

1. **One Private DNS Zone per service type** — not per service instance. All Key Vaults share `privatelink.vaultcore.azure.net`.
2. **The VNet link is what makes the zone visible** to resources in your VNet. Without it, resolution falls back to the public IP.
3. **NSG policies on the private endpoints subnet must be enabled** — the VNet module handles this with `private_endpoint_network_policies = "NetworkSecurityGroupEnabled"`.
4. **Disable the public endpoint** on every PaaS resource after the private endpoint is working — otherwise the private endpoint only adds security theatre.
5. **Each private endpoint = one IP** consumed from `snet-private-endpoints`. A `/24` (251 usable IPs) is plenty for a real project.
6. **Pipeline access** — decide whether your Terraform runner is inside the VNet or needs an IP exception before disabling public access.
7. **Natural implementation order** — Private DNS Zones module first (shared across all services), then individual service modules that reference it.
