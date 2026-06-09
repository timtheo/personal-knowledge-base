# Network Architecture — Front Door + Container App Environment

How to wire Azure Front Door as the public entry point in front of a Container App Environment that talks to several PE-fronted Azure services (Key Vault, Cosmos, ACR, AI Foundry, etc.).

## The fork in the road

Two architectures. Everything else flows from this choice.

### Pattern A — Front Door (Standard) → public CAE → private PEs

```
Internet → Front Door (Standard, WAF) → CAE *external* LB (public IP) → app
                                                                          │
                                                                          ↓
                                                            private VNet ─┘
                                                                          │
                                                       PE → Key Vault
                                                       PE → Cosmos
                                                       PE → ACR
                                                       PE → AI Foundry
```

- CAE in external mode — has a public IP for ingress.
- Outbound from the app stays private because each backing service has a PE.
- Front Door has a public origin (the CAE's public hostname).
- Defense in depth at the CAE ingress: restrict to Front Door's source IPs and validate the `X-Azure-FDID` header in the app.

**Cost:** cheap. Front Door Standard ~$35/mo base, CAE the same as today.

### Pattern B — Front Door (Premium) → Private Link → internal CAE → private PEs

```
Internet → Front Door Premium (Private Link to your VNet)
                                       │
                                       ↓
                            CAE *internal* LB (private IP) → app
                                                              │
                                                              ↓
                                                  PE → KV / Cosmos / ACR / Foundry
```

- CAE in internal mode — no public IP anywhere on the app side.
- Front Door Premium opens a private link connection into the VNet.
- Strongest posture: the only thing on the public internet is Front Door's edge POPs.

**Cost:** ~$330/mo base for Front Door Premium plus per-policy WAF cost.

## Recommendation — Pattern A with hardening

For a dev / single-app project at this stage of the lifecycle, the security delta between A and B is roughly **5-10% of risk reduction for a ~10× cost increase** on the Front Door bill. The bigger threats (compromised secret in code, weak API auth, mis-scoped role assignment) are not what these patterns defend against.

Pattern A also matches what's already wired up (Standard FD, ACR Standard SKU, no Front Door Private Link service). Switching to Pattern B is a meaningful project — new SKU, new Private Link resource, internal CAE mode (which is sticky), custom DNS work.

**Reserve Pattern B for when:**
- Regulated data (HIPAA, PCI, GDPR with strict residency)
- Threat model includes attackers finding the CAE public IP through DNS enumeration or scanning and bypassing Front Door
- Already running Front Door Premium for other reasons (bot protection, OWASP managed rules, IP intelligence)

## Hardening Pattern A (do all four)

Together these are worth more than upgrading to Pattern B.

### 1. Restrict CAE ingress to Front Door's source IPs

Use the `AzureFrontDoor.Backend` **service tag** as an IP restriction on every container app ingress. That tag is the set of IPs Microsoft's Front Door POPs use when calling origins.

```hcl
ip_security_restrictions = [
  {
    name             = "allow-front-door"
    action           = "Allow"
    ip_address_range = "AzureFrontDoor.Backend"  # service tag
    description      = "Only Microsoft Front Door POPs"
  },
]
```

Without this, anyone who guesses the `<app>.<default_domain>` URL bypasses your WAF and hits the app directly.

### 2. Validate `X-Azure-FDID` in app code

The IP restriction stops random attackers but not "someone with *their own* Front Door pointed at your origin." Front Door adds an `X-Azure-FDID: <your-front-door-instance-id>` header on every request. App code should reject requests without the matching value.

Application-layer concern, not Terraform — note it in the runbook.

### 3. Attach a WAF policy to Front Door

Front Door Standard supports WAF policies (~$5/mo base + per-million-request fees) including the Microsoft Default Rule Set (OWASP Top 10 baseline). Don't run Front Door without one.

### 4. Stream Front Door access logs to Log Analytics

Same `diagnostic_setting` pattern used elsewhere in the project. Detect probing, alert on 403 spikes, audit traffic patterns.

## IP plan — current subnets are fine

| Subnet | Current | Capacity | Notes |
|---|---|---|---|
| `snet-private-endpoints` | `10.0.1.0/24` | ~251 usable | 1 IP per PE. Project plans 8-12 PEs. Massive overkill but harmless — a `/27` (29 IPs) would suffice. |
| `snet-container-apps` | `10.0.2.0/23` | ~507 usable | CAE infra reserves ~12 IPs. With Workload Profiles, ~1 IP per replica + headroom. Good for ~50 active replicas across all apps. |
| `AzureBastionSubnet` | `10.0.4.0/26` | 58 usable | Azure requires exactly `/26` here. Correct. |

**Don't resize existing subnets.** Subnet changes can force VNet recreation. The PE subnet being /24 is fine — better slack than "no more IPs" mid-incident.

### When to outgrow

- **>50 concurrent container app replicas**: bump container-apps subnet to `/22` (needs a different address space — `10.0.2.0/23` overlaps with where a /22 would land).
- **Add a VM jumpbox, Application Gateway, or APIM**: add new subnets. The VNet has plenty of room (`10.0.5.0/24` onward is unused).

### Future subnets worth planning for

| Subnet | Purpose | Suggested size |
|---|---|---|
| `snet-app-gateway` | If Front Door → App Gateway for L7 inside the VNet | `/24` |
| `snet-jumpbox` | Small Linux VM for `kubectl`-style poking inside the VNet | `/29` |
| `snet-functions` | function-app-linux with VNet integration | `/26` |
| `snet-postgres` | PostgreSQL Flexible Server delegated subnet | `/26` |

All fit in unused space (`10.0.5.0` onward).

## Decision summary

**Today — stay on Pattern A.** Stop here:

1. Keep CAE external. Don't enable `internal_load_balancer_enabled` yet.
2. Add the `AzureFrontDoor.Backend` IP restriction on every container app ingress.
3. Add `X-Azure-FDID` validation to app code.
4. Attach a WAF policy to the Front Door profile.
5. Wire Front Door diagnostics into the monitoring workspace.
6. Leave subnets as-is.

**When stronger isolation is needed later — migrate to Pattern B.** Treat it as a discrete project:
- Upgrade Front Door SKU to Premium.
- Add `azurerm_cdn_frontdoor_origin` with private link service.
- Flip CAE to internal mode (destroy + recreate of the CAE and every app inside — schedule a maintenance window).
