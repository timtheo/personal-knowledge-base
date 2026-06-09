# Azure Load Balancing Services — Overview & Decision Guide

> Based on Microsoft Architecture Center documentation (last reviewed: April 2026).

---

## Quick Reference

| Service | OSI Layer | Scope | Traffic Type | Core Purpose |
|---|---|---|---|---|
| **Azure Front Door** | L7 | 🌍 Global | HTTP(S) | Global web delivery, CDN, edge WAF |
| **Application Gateway** | L7 | 🏠 Regional | HTTP(S), TCP, TLS | Regional reverse proxy, WAF, path routing |
| **Load Balancer** | L4 | 🏠 Regional (+ cross-region) | TCP / UDP (any) | High-throughput VM/VMSS load balancing |
| **Traffic Manager** | DNS | 🌍 Global | Any (DNS-based) | Multi-region DNS failover & routing |
| **API Management** | L7 | 🏠/🌍 Regional or Global | HTTP(S) APIs only | API gateway, lifecycle, and backend pooling |

**Rule of thumb:**
- Need global distribution for web traffic → **Front Door**
- Need regional Layer-7 features (WAF, URL routing) → **Application Gateway**
- Need to balance VMs/containers at Layer 4 → **Load Balancer**
- Need DNS-based global routing (non-HTTP, or simple failover) → **Traffic Manager**
- Need to manage, secure, and version APIs → **API Management**

---

## Decision Flowchart

```
Is it a web (HTTP/HTTPS) application?
│
├─ NO  → Is it public-facing across regions?
│        ├─ YES → Traffic Manager  (DNS-based global routing)
│        └─ NO  → Load Balancer    (Layer 4, within a VNet)
│
└─ YES → Is it internet-facing?
         │
         ├─ NO  → Load Balancer (internal) or Application Gateway (internal)
         │
         └─ YES → Does it need to span multiple regions or require CDN/acceleration?
                  │
                  ├─ NO  → Application Gateway  (regional L7, WAF, path routing)
                  │
                  └─ YES → Azure Front Door  (global L7, CDN, WAF at the edge)
                           │
                           ├─ + Regional L7 processing (path routing in VNet)?
                           │   → Front Door + Application Gateway
                           │
                           ├─ + API-only backend?
                           │   → Front Door + API Management
                           │
                           └─ + IaaS VMs behind it?
                               → Front Door + Load Balancer
```

---

## 1. Azure Front Door

### What it is
Azure Front Door is a **global, cloud-native application delivery network (ADN)** that operates at Layer 7. It uses Microsoft's global edge network (118+ edge locations across 100 metro areas) to accelerate, secure, and deliver web applications and APIs worldwide.

It is the go-to service when your app is public-facing, needs to be globally fast, and needs edge-level security.

### Key Features
- **Global anycast routing** — routes users to the nearest healthy PoP, reducing latency by up to 3×
- **Split TCP** — terminates the TLS connection at the edge PoP, reducing handshake latency
- **CDN + caching** — static content cached at the edge, dynamic content accelerated via optimized WAN
- **SSL/TLS offload** — free auto-renewing managed certificates, TLS terminates at the edge
- **Integrated WAF** (Premium SKU) — OWASP rule sets, bot protection, custom rules, rate limiting
- **Health probes** — active monitoring of origin health, automatic failover in seconds
- **Private Link origins** (Premium SKU) — connect to backends without exposing them to the internet
- **Rules engine** — URL rewrites, header manipulation, redirects, conditional routing at the edge
- **DDoS protection** — Layer 3/4 built-in, Layer 7 via WAF

### SKUs
| | Standard | Premium |
|---|---|---|
| CDN + routing | ✅ | ✅ |
| WAF | ❌ | ✅ |
| Private Link origins | ❌ | ✅ |
| Bot protection | ❌ | ✅ |
| SLA | 99.99% | 99.99% |

### When to use
- ✅ Internet-facing web app or API that needs to serve users **across multiple regions or globally**
- ✅ You need **CDN caching** for static content and **acceleration** for dynamic content
- ✅ You need a **WAF at the edge** (before traffic hits your origin) — Premium SKU
- ✅ You want **TLS termination at the edge** without managing certificates per origin
- ✅ You want **fast failover** (seconds, not minutes like Traffic Manager) across regions
- ✅ You want to consolidate global routing, CDN, and security in one service
- ✅ Your backends are Azure PaaS (App Service, Container Apps, Storage) or behind Private Link

### When NOT to use
- ❌ Your application is **not HTTP/HTTPS** (use Traffic Manager or Load Balancer)
- ❌ You need to route traffic **within a VNet** to VMs (use Application Gateway or Load Balancer)
- ❌ You only have a **single-region** deployment without global users (Application Gateway is simpler and cheaper)
- ❌ You need **WebSocket or gRPC** with stateful sessions over long-lived connections (Front Door adds latency via split TCP)

### Architecture
```
Users (worldwide)
       │
       ▼
┌─────────────────────────────────────────────┐
│           Azure Front Door (Global)          │
│  PoP: West EU  │  PoP: East US  │  PoP: SEA │
│  - Anycast routing (nearest PoP)             │
│  - CDN cache                                 │
│  - WAF (Premium)                             │
│  - TLS offload                               │
│  - Health probes                             │
└──────────────────────┬──────────────────────┘
                       │  Origin routing
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    Origin (EU)   Origin (US)  Origin (Asia)
  (App Service,  (App Service, (Container App,
   Storage, etc.) AKS, etc.)    etc.)
```

---

## 2. Azure Application Gateway

### What it is
Azure Application Gateway is a **regional Layer-7 reverse proxy and load balancer**. It terminates client connections, inspects HTTP(S) requests, and makes intelligent routing decisions based on URL paths, host headers, and other L7 attributes before forwarding traffic to backend pools within a VNet.

Think of it as an enterprise-grade **nginx/HAProxy** as a managed Azure service.

### Key Features
- **URL path-based routing** — `/images/*` → Image servers, `/api/*` → API servers
- **Multi-site hosting** — Multiple domains behind a single gateway via host header routing
- **TLS/SSL offload and end-to-end TLS** — Centralized certificate management
- **WAF (v2)** — OWASP CRS, bot protection, custom rules, per-URI exclusions
- **Autoscaling** — v2 SKU scales automatically based on traffic load
- **Zone redundancy** — v2 SKU deploys across availability zones
- **Session affinity** — Cookie-based sticky sessions to the same backend instance
- **Connection draining** — Graceful removal of backend instances during deployments
- **Private-only deployment** — Can be fully internal within a VNet
- **Layer 4 support (TCP/TLS proxy)** — v2 also supports TCP and TLS passthrough

### SKUs
| | Standard v2 | WAF v2 |
|---|---|---|
| L7 routing | ✅ | ✅ |
| Autoscaling + zones | ✅ | ✅ |
| WAF (OWASP) | ❌ | ✅ |
| Bot protection | ❌ | ✅ |
| SLA | 99.95% | 99.95% |

> Basic SKU is available for dev/test at lower cost without autoscaling or zones.

### When to use
- ✅ You need **regional** Layer-7 load balancing and routing **within Azure** (public or private)
- ✅ You need a **WAF in front of a regional app** (e.g., a single-region App Service or AKS cluster)
- ✅ You need **URL path-based routing** to different backend pools (e.g., microservices behind one IP)
- ✅ You need **session affinity** (sticky sessions) for stateful applications
- ✅ You need **connection draining** for zero-downtime deployments
- ✅ You are routing traffic to **IaaS VMs or AKS** inside a VNet
- ✅ You want a **private-only gateway** with no public internet exposure
- ✅ You are deploying behind **Front Door** for deeper regional inspection (Front Door → App Gateway)

### When NOT to use
- ❌ You need **global routing** across regions (use Front Door; App Gateway is regional only)
- ❌ Your traffic is **non-HTTP** TCP/UDP at Layer 4 (use Load Balancer)
- ❌ You need **DNS-based failover** across regions (use Traffic Manager)
- ❌ You only need simple round-robin load balancing across VMs without L7 features (Load Balancer is cheaper and faster)

### Architecture
```
Internet / Front Door
         │
         ▼
┌────────────────────────────────────┐
│     Application Gateway (Regional) │
│  - TLS offload                     │
│  - WAF (v2)                        │
│  - URL path routing                │
│  - Session affinity                │
│  - Health probes                   │
└───────────┬──────────┬─────────────┘
            │          │
     /api/* │          │ /images/*
            ▼          ▼
     ┌────────────┐  ┌────────────┐
     │ API Backend│  │ Image Pool │
     │ (VM / AKS) │  │ (VM / AKS) │
     └────────────┘  └────────────┘
```

---

## 3. Azure Load Balancer

### What it is
Azure Load Balancer is a **regional Layer-4 (TCP/UDP) pass-through load balancer**. It distributes inbound flows to backend VMs or VMSS based on a hash of IP/port tuples. It is ultra-low latency, high throughput, and handles millions of connections per second — but has no knowledge of the HTTP payload.

It is the right tool for **raw network performance** and **VM-level load distribution** inside a VNet.

> ⚠️ Basic Load Balancer was **retired on September 30, 2025**. Use Standard Load Balancer only.

### Key Features
- **Layer 4 pass-through** — no connection termination, minimal latency overhead
- **TCP and UDP** — supports all protocols, including non-HTTP workloads (databases, game servers, SMTP)
- **Public and internal** — external (internet-facing) or internal (within VNet)
- **Zone redundancy** — Standard SKU is zone-redundant by default
- **High-availability ports** — load balance all ports simultaneously (useful for NVAs)
- **Health probes** — TCP, HTTP, HTTPS health checks on backend instances
- **Outbound SNAT** — managed outbound connectivity for VMs without public IPs
- **Cross-region load balancer** — globally distribute traffic across regional Standard Load Balancers
- **Gateway Load Balancer integration** — chain NVAs (firewalls, IDS) transparently in the path
- **Zero Trust** — Standard LB is closed to inbound by default; NSGs control access

### SKUs
| | Standard | Gateway |
|---|---|---|
| Zone redundancy | ✅ | ✅ |
| SLA | 99.99% | 99.99% |
| Use case | General L4 | NVA chaining |
| HA ports | ✅ | ✅ |

### When to use
- ✅ You need to load balance **TCP or UDP traffic** (databases, game servers, streaming, SMTP, custom protocols)
- ✅ You need to distribute traffic across **VMs or VMSS** inside a VNet at high throughput / ultra-low latency
- ✅ You need **internal load balancing** between tiers of an application (e.g., web tier → database cluster)
- ✅ You need **outbound SNAT** for VMs that don't have public IPs
- ✅ You need to chain **network virtual appliances (NVAs)** transparently (Gateway Load Balancer)
- ✅ You are building a **non-HTTP workload** (databases, IoT, media streaming, etc.)

### When NOT to use
- ❌ You need **HTTP-aware routing** (URL paths, host headers) — use Application Gateway or Front Door
- ❌ You need **WAF** protection — Load Balancer has no L7 inspection
- ❌ You need **global routing** across regions via a single endpoint — use Front Door or Traffic Manager
- ❌ Your backends are **PaaS services** (App Service, Functions) — those have built-in load balancing; use Front Door

### Architecture
```
Internet / Application Gateway
              │
              ▼
┌─────────────────────────────────┐
│   Azure Load Balancer (L4)      │
│   (Public or Internal)          │
│   - TCP/UDP hash distribution   │
│   - Health probes               │
│   - SNAT outbound               │
└─────────┬────────────┬──────────┘
          │            │
          ▼            ▼
      ┌────────┐   ┌────────┐
      │  VM 1  │   │  VM 2  │
      └────────┘   └────────┘
      (any protocol: DB, SMTP, etc.)
```

---

## 4. Azure Traffic Manager

### What it is
Azure Traffic Manager is a **global DNS-based traffic router**. It does not handle actual traffic — it only responds to DNS queries with the IP address of the best available endpoint based on your routing policy. The client then connects directly to that endpoint.

Because it operates at DNS level, failover speed is limited by DNS TTL (minutes, not seconds). Use **Front Door** when you need faster failover for HTTP workloads.

### Key Features
- **6 routing methods** (see table below)
- **Endpoint monitoring** — HTTP/HTTPS/TCP health checks on endpoints
- **Automatic failover** — removes unhealthy endpoints from DNS responses
- **Any endpoint type** — Azure services, external URLs, on-premises endpoints
- **Nested profiles** — combine routing methods for complex topologies
- **Hybrid/multi-cloud** — works with non-Azure endpoints (on-prem, other clouds)
- **IPv6 support** — full dual-stack DNS resolution

### Routing Methods
| Method | Description | Use case |
|---|---|---|
| **Performance** | Routes to lowest-latency endpoint for the user | Globally distributed app, optimize user experience |
| **Priority** | Routes to primary; fallback to secondary | Active/passive DR failover |
| **Weighted** | Distributes traffic by weight percentage | Canary deployments, gradual traffic shifts |
| **Geographic** | Routes based on user's geographic location | Data residency, regional compliance |
| **Subnet** | Routes based on client IP subnet | Different backends for corporate vs. consumer users |
| **MultiValue** | Returns multiple healthy endpoints in one DNS response | Clients pick the best one themselves |

### When to use
- ✅ You need **global DNS-based failover** across regions (active/passive DR)
- ✅ Your traffic is **non-HTTP** and you still need global distribution (databases, SMTP, RTMP, etc.)
- ✅ You need **geographic routing** to enforce data residency (e.g., EU users → EU endpoint)
- ✅ You need a **simple, low-cost** global routing layer (Traffic Manager is much cheaper than Front Door)
- ✅ You have a **hybrid** environment and need to route to on-premises or multi-cloud endpoints
- ✅ You need to **combine routing methods** using nested profiles (e.g., priority between regions, performance within a region)

### When NOT to use
- ❌ You need **sub-second failover** for HTTP workloads — DNS TTL delays make this too slow; use **Front Door** instead
- ❌ You need **SSL termination, WAF, or caching** — Traffic Manager is DNS-only, it does not touch the traffic
- ❌ You need to route to **private/internal endpoints** (Traffic Manager only works with public DNS)
- ❌ You are putting Traffic Manager **behind Front Door** — this is explicitly an anti-pattern per Microsoft docs

### Architecture
```
User (any location)
        │
        │ DNS query: myapp.trafficmanager.net
        ▼
┌───────────────────────────────┐
│     Traffic Manager (Global)  │
│  - Health checks all endpoints│
│  - Returns IP of best endpoint│
│    (by routing method)        │
└───────────────────────────────┘
        │ DNS response: "use 52.x.x.x"
        │
        ▼ (direct connection — TM not in traffic path)
   Best endpoint
  (App GW EU / App GW US / On-prem)
```

---

## 5. Azure API Management

### What it is
Azure API Management (APIM) is a **full API lifecycle management platform**, not a traditional load balancer. Its primary purpose is to publish, secure, transform, monitor, and version HTTP(S) APIs. It includes an **optional backend pool load balancing** feature for distributing requests across multiple API backends.

> ⚠️ APIM is included here for completeness. **Do not use it solely for load balancing** — it adds significant cost and operational overhead. Only use its load balancing if you already need APIM for API management purposes.

### Key Features
- **API gateway** — single entry point for all APIs, routes to appropriate backend
- **Backend load balancing** — round-robin, weighted, priority-based across backend pool nodes
- **Policy engine** — JWT validation, rate limiting, quotas, IP filtering, request/response transformation, caching
- **Developer portal** — auto-generated, customizable API documentation and self-service subscriptions
- **Multi-region** — Premium tier supports deployment to multiple regions with active-active routing
- **Self-hosted gateway** — deploy the gateway container on-prem or in other clouds
- **Workspaces** — federated API management for multiple teams within one APIM instance
- **AI gateway capabilities** — load balance Azure OpenAI backends, token-rate limiting, semantic caching

### Tiers
| Tier | Use case | VNet integration | Multi-region |
|---|---|---|---|
| Consumption | Serverless, low traffic | ❌ | ❌ |
| Basic v2 | Dev/test | Outbound only | ❌ |
| Standard v2 | Production | Outbound only | ❌ |
| Premium v2 | Enterprise production | Full injection | ✅ |
| Developer | Non-production only | ✅ | ❌ |

### When to use
- ✅ You need a **full API gateway** with versioning, subscriptions, throttling, and a developer portal
- ✅ You are publishing **APIs to external partners or developers** and need a governed, documented interface
- ✅ You need **JWT/OAuth validation, rate limiting, or request transformation** before backends
- ✅ You have **multiple AI/OpenAI backends** to load balance and want token-rate limiting
- ✅ You already use APIM and want to **optionally load balance across its backend pool nodes**
- ✅ You need to **abstract and modernize legacy backends** without rewriting them

### When NOT to use
- ❌ You only need **load balancing** — the overhead of APIM is not justified
- ❌ Your traffic is **not HTTP/HTTPS** — APIM only handles HTTP APIs
- ❌ You need **high-throughput, low-latency** load balancing — APIM adds processing overhead per request
- ❌ You need **CDN, caching, or DDoS protection** at the edge — use Front Door for that

### Architecture
```
Client apps / Partners
         │
         ▼
┌──────────────────────────────────┐
│    API Management Gateway        │
│  - Auth (JWT, subscriptions)     │
│  - Rate limiting / quotas        │
│  - Request transformation        │
│  - Caching                       │
│  - Load balance backend pool     │
└──────────┬───────────┬───────────┘
           │           │
     Weight: 60%  Weight: 40%
           ▼           ▼
    ┌──────────┐  ┌──────────┐
    │ Backend 1│  │ Backend 2│
    │(Function)│  │(App Svc) │
    └──────────┘  └──────────┘
```

---

## Combined Architecture Patterns

### Pattern 1 — Simple Regional Web App
**Services:** Application Gateway (+ WAF)
**When:** Single-region, internet-facing, needs WAF and path routing

```
Internet → WAF v2 + Application Gateway → Backend VMs / AKS (VNet)
```

---

### Pattern 2 — Global Web App (PaaS backends)
**Services:** Azure Front Door
**When:** Multi-region PaaS backends (App Service, Container Apps), need CDN and edge WAF

```
Users (worldwide)
     │
     ▼
Azure Front Door (Global)
  - WAF at edge
  - CDN caching
  - Anycast routing
     │
     ├─── Origin: App Service (West EU)
     ├─── Origin: App Service (East US)
     └─── Origin: App Service (Asia)
```

---

### Pattern 3 — Global + Regional WAF + Private Backends
**Services:** Front Door → Application Gateway
**When:** Need global edge presence AND deep regional inspection (WAF, path routing) with VNet-private backends

```
Users (worldwide)
     │
     ▼
Azure Front Door (Global)  ← edge caching, DDoS, TLS offload
     │
     ▼
Application Gateway (per region)  ← WAF, URL routing, private VNet
     │
     ├── /api/*  → API Backend Pool (VMs/AKS)
     └── /web/*  → Web Backend Pool (VMs/AKS)
```

> This is the recommended pattern for enterprise workloads needing both global scale and deep L7 inspection.

---

### Pattern 4 — Multi-Region with DNS Failover (non-HTTP or simple)
**Services:** Traffic Manager → Application Gateway (per region)
**When:** Non-HTTP workloads or simple active/passive failover, no need for CDN/acceleration

```
Users
  │ DNS query
  ▼
Traffic Manager (Global DNS)
  │ Returns IP of healthy region
  ▼ (client connects directly)
  ├── Application Gateway (West EU)  [Priority 1 - primary]
  └── Application Gateway (East US)  [Priority 2 - standby]
       └── Both route to regional VMs via Load Balancer
```

---

### Pattern 5 — API Platform with Global Reach
**Services:** Front Door → API Management
**When:** You need a governed API platform globally accessible with edge security

```
API Consumers (global)
     │
     ▼
Azure Front Door (Global)
  - TLS offload, DDoS, caching
     │
     ▼
API Management (Regional / Multi-region Premium)
  - Auth, rate limiting, transformation
  - Backend load balancing
     │
     ├── Backend: Azure Function
     ├── Backend: App Service
     └── Backend: Azure OpenAI (with token rate limiting)
```

---

### Pattern 6 — Full Enterprise Stack (IaaS)
**Services:** Traffic Manager → Application Gateway → Load Balancer
**When:** Multi-region IaaS with VMs, full-stack traffic control at every layer

```
Internet
    │ DNS
    ▼
Traffic Manager (Global DNS routing)
    │ Directs to nearest healthy region
    ▼
Application Gateway (Regional — L7)
    - WAF, TLS offload, URL routing
    │
    ▼
Azure Load Balancer (Internal — L4)
    - Distributes to VM backend pool
    │
    ├── VM 1
    ├── VM 2
    └── VM 3
         │ Internal LB
         ▼
    Database Cluster (Load Balancer)
    ├── DB Primary
    └── DB Replica
```

---

## Key Considerations Summary

### What to consider before choosing

- **Traffic type** — Is it HTTP(S)? Or raw TCP/UDP? If HTTP(S), you have richer options.
- **Global vs regional** — Do users come from worldwide? Or is the app regional only?
- **Failover speed** — DNS-based (Traffic Manager: minutes) vs. connection-based (Front Door: seconds)
- **WAF requirement** — Do you need L7 inspection and blocking? → Front Door (Premium) or App Gateway (WAF v2)
- **Backend type** — PaaS (App Service, Functions) → Front Door. IaaS VMs → App Gateway + Load Balancer.
- **Private backends** — Need backends unexposed to internet? → Front Door Private Link (Premium) or App Gateway VNet
- **Protocol support** — WebSockets, gRPC, TCP? → App Gateway or Load Balancer. HTTP/2? All support it.
- **Cost vs features** — Load Balancer < Traffic Manager < Front Door Standard < App Gateway < Front Door Premium < APIM Premium
- **API management** — If you need versioning, developer portal, subscriptions → API Management; if you just need routing → Front Door
- **Compliance/data residency** — Geographic routing for regulation? → Traffic Manager (geographic method) or Front Door (geo-filtering rules)

---

## Sources

- [Load Balancing Options — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview)
- [Azure Front Door Overview](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview)
- [Azure Application Gateway Overview](https://learn.microsoft.com/en-us/azure/application-gateway/overview)
- [Azure Load Balancer Overview](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
- [Azure Traffic Manager Overview](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
- [Azure API Management — Key Concepts](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts)
- [Using Load Balancing Services Together in Azure](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-load-balancing-azure)
- [Multi-region Load Balancing with Traffic Manager and Application Gateway](https://learn.microsoft.com/en-us/azure/high-availability/reference-architecture-traffic-manager-application-gateway)
