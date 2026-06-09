# GitOps

> Part of the operational-workflows series: **[CI/CD] · [GitOps] · [MLOps] · [LLMOps]**.
> Original trio sourced from an infographic by Vishakha Sadhwani (@vsadhwani).

---

## What It Is

GitOps is a way of doing **continuous delivery for infrastructure & Kubernetes** where
**Git is the single source of truth** and an in-cluster **operator continuously reconciles**
the live cluster to match what's declared in Git.

The four GitOps principles (OpenGitOps / CNCF):

1. **Declarative** — the whole system is described declaratively (desired state, not steps).
2. **Versioned & immutable** — that desired state is stored in Git (versioned, auditable).
3. **Pulled automatically** — software agents pull the approved state from Git.
4. **Continuously reconciled** — agents continuously observe and correct drift.

**Mental model:** Git holds the desired state; a controller makes reality match it, forever.
It is **pull-based** and **a loop** — the opposite of CI/CD's push-once pipeline.

---

## The 10 Steps

| # | Step | What happens |
|---|------|--------------|
| 1 | **Git as Source of Truth** | The desired state of everything lives in Git. |
| 2 | **Declarative Manifests** | Describe *what* you want via templating → **Helm Charts** + **Kustomize**. |
| 3 | **Open Pull Request** | Changes to desired state are proposed via PR — enabling review and audit. |
| 4 | **Policy Validation** | Automated guardrails check manifests → **OPA Gatekeeper** + **Kyverno**. |
| 5 | **Merge to Main (GitHub)** | Approved changes merge — the merge *is* the deploy trigger. |
| 6 | **GitOps Operator Watches** | An in-cluster agent watches the repo → **ArgoCD** or **Flux**. |
| 7 | **Sync to Cluster** | The operator applies the Git state to the live cluster. |
| 8 | **Drift Detection** | Continuously compares live vs. desired → **Alert Drift** + **Auto Remediate** (self-heal). |
| 9 | **Secrets Management** | Secrets handled safely in Git → **Sealed Secrets** + **Vault ESO** (External Secrets Operator). |
| 10 | **Continuous Reconcile** | The loop never stops — the cluster is perpetually kept matching Git. |

---

## Flow

```
Git as Source of Truth
   │
Declarative Manifests ─► Helm Charts
   │            └────────► Kustomize
   │
Open Pull Request
   │
Policy Validation ─────► OPA Gatekeeper
   │           └─────────► Kyverno
   │
Merge to Main (GitHub)
   │
GitOps Operator Watches ─► ArgoCD
   │              └─────────► Flux
   │
Sync to Cluster
   │
Drift Detection ───────► Alert Drift
   │          └──────────► Auto Remediate
   │
Secrets Management ────► Sealed Secrets
   │           └─────────► Vault ESO
   │
Continuous Reconcile ◄──┐
   └────────────────────┘  (never stops)
```

---

## Deeper Dive

### Push vs Pull — the defining difference

| | Classic CI/CD (push) | GitOps (pull) |
|--|----------------------|---------------|
| Who applies changes? | CI server runs `kubectl apply` into the cluster | An agent *inside* the cluster pulls from Git |
| Credentials | CI holds cluster admin creds (broad blast radius) | Cluster creds never leave the cluster |
| Drift handling | None — manual `kubectl edit` goes unnoticed | Detected and (optionally) auto-reverted |
| Audit trail | Pipeline logs | Git history = full, signed audit log |
| Rollback | Re-run pipeline | `git revert` |

The security win is big: nothing outside the cluster needs write access to the cluster.

### ArgoCD vs Flux (step 6 — the two leading operators)

| | **ArgoCD** | **Flux** |
|--|-----------|----------|
| Origin | Intuit → CNCF graduated | Weaveworks → CNCF graduated |
| UI | Rich web UI + visual diff/sync | CLI-first, no built-in UI (use Weave GitOps / Capacitor) |
| Model | "Application" CRD, app-centric | Set of composable controllers (source, kustomize, helm, notification) |
| Multi-tenancy | Projects, RBAC, app-of-apps | Namespaced, Git-repo-scoped |
| Best when | You want a console + many teams/apps | You want lightweight, modular, GitOps-as-plumbing |

Both reconcile continuously; the choice is largely UX/operational preference.

### Helm vs Kustomize (step 2)

- **Helm** — package manager: templated charts with `values.yaml`, versioning, releases,
  rollbacks. Great for *distributing* reusable apps. Downside: Go templating gets gnarly.
- **Kustomize** — template-free overlays: a `base/` plus per-environment patches. Built into
  `kubectl`. Great for *your own* manifests across dev/staging/prod. Downside: no packaging/versioning.
- Common pattern: Helm for third-party charts, Kustomize overlays on top for env-specifics
  (ArgoCD and Flux both support Helm *and* Kustomize, including Helm-rendered-then-Kustomized).

### Policy as code (step 4) — OPA Gatekeeper vs Kyverno

These are **admission controllers** that reject non-compliant manifests before they hit the cluster.

- **OPA Gatekeeper** — policies written in **Rego** (OPA's language). Powerful, but Rego has
  a learning curve. Backed by the broader Open Policy Agent ecosystem.
- **Kyverno** — policies written as **Kubernetes YAML** (no new language). Easier to adopt,
  Kubernetes-native, can also *mutate* and *generate* resources.

Typical rules: "no `:latest` tags", "must set resource limits", "images must be signed",
"no privileged containers".

### Secrets in Git without leaking them (step 9)

You can't commit plaintext secrets to Git. Two patterns:

- **Sealed Secrets (Bitnami)** — encrypt a secret with a cluster-side public key; the
  *encrypted* `SealedSecret` is safe to commit. Only the in-cluster controller can decrypt it.
- **External Secrets Operator (ESO) + Vault** — Git holds only a *reference*; ESO fetches the
  real value at runtime from an external store (HashiCorp Vault, AWS/GCP/Azure secret managers)
  and materializes a Kubernetes `Secret`. The secret never touches Git at all.

### Drift detection & self-healing (step 8)

Because the operator knows the desired state, any manual `kubectl edit` or outside change is
**drift**. The operator can **alert** (notify, leave it) or **auto-remediate** (revert to Git).
This is what makes GitOps "self-healing" — the cluster can't permanently diverge from Git.

### Where GitOps sits relative to CI

GitOps is usually the **CD half**, not CI. The handoff:

```
[CI: build + test + push signed image]  ──►  writes new image tag into config repo (PR)
                                                      │
                          [GitOps operator pulls] ◄───┘  ──►  syncs to cluster, reconciles
```

CI never touches the cluster; it only updates Git. See [CI-CD].

---

## Relationship to CI/CD & MLOps

- **vs [CI-CD]:** GitOps is the pull-based, continuously-reconciling delivery layer. CI builds
  and pushes; GitOps deploys and keeps the cluster honest.
- **vs [MLOps]:** GitOps can *be the deployment substrate* for MLOps — model-serving manifests
  and infra managed declaratively in Git, with the same drift/reconcile guarantees.
