# CI/CD — Continuous Integration / Continuous Delivery

> Part of a three-way comparison: **[CI/CD] vs [GitOps] vs [MLOps]**.
> Source: infographic by Vishakha Sadhwani (@vsadhwani).

---

## What It Is

CI/CD is the traditional pipeline for getting **application code** from a developer's
machine into **production** safely and automatically.

- **CI (Continuous Integration):** merge code frequently, build it, and run automated
  tests on every change so integration problems surface early.
- **CD (Continuous Delivery/Deployment):** take the tested build and promote it through
  environments up to production — either with a manual approval (Delivery) or fully
  automated (Deployment).

**Mental model:** code → tested → built → promoted through environments → prod.
It is **push-based** and **linear** — it runs once per change and ends at deploy.

---

## The 10 Steps

| # | Step | What happens |
|---|------|--------------|
| 1 | **Write Code** | A developer authors application code locally. |
| 2 | **Commit & Push** | Changes are committed to version control and pushed to the remote repo. |
| 3 | **Run Tests** | Automated suite kicks off → **Unit Tests** (functions in isolation) + **Integration Tests** (components working together). |
| 4 | **Static Analysis** | Inspect code without running it → **SAST/Lint** (security + style/quality) + **Dependency Scan** (3rd-party CVEs). |
| 5 | **Build Artifact** | Compile/package into a deployable artifact (container image, JAR, binary). |
| 6 | **Push to Registry** | Publish the artifact → **Tag & Version** (semantic versioning) + **Sign Image** (supply-chain integrity, e.g. Cosign). |
| 7 | **Deploy to Staging** | Release to a pre-prod environment that mirrors production. |
| 8 | **Smoke Tests** | Quick "is it alive?" checks → then **E2E Tests** (full user flows) + **Perf Tests** (load/performance). |
| 9 | **Approval Gate** | Manual/automated checkpoint before prod (human sign-off, change management). |
| 10 | **Deploy to Prod** | Release to production. |

---

## Flow

```
Write Code
   │
Commit & Push
   │
Run Tests ───────► Unit Tests
   │       └──────► Integration Tests
   │
Static Analysis ─► SAST / Lint
   │        └─────► Dependency Scan
   │
Build Artifact
   │
Push to Registry ─► Tag & Version
   │         └─────► Sign Image
   │
Deploy to Staging
   │
Smoke Tests ─────► E2E Tests
   │       └──────► Perf Tests
   │
Approval Gate
   │
Deploy to Prod
```

---

## Deeper Dive

### CI vs CD vs CD (the three acronyms hiding in two letters)

| Term | Scope | Ends at |
|------|-------|---------|
| **Continuous Integration** | Build + test on every commit | A validated, tested build artifact |
| **Continuous Delivery** | CI + automated release **to a staging-ready state**, prod gated by a **manual approval** | A human clicks "deploy" |
| **Continuous Deployment** | CI + **fully automated** release all the way to prod, no human gate | Auto-prod on green pipeline |

Step 9 (**Approval Gate**) in this diagram is the dividing line: keep it manual = Delivery;
remove it = Deployment.

### Why the parallel branches matter

Steps 3, 4, 6, and 8 fan out into two checks each. The point is **shift-left**: catch
problems as early and as cheaply as possible.

- **Unit vs Integration tests** — unit tests are fast and isolated (mock everything);
  integration tests are slower but catch wiring/contract bugs. Run unit first to fail fast.
- **SAST/Lint vs Dependency Scan** — SAST finds bugs in *your* code; dependency scanning
  (SCA) finds known CVEs in *other people's* code you pulled in. Most real-world breaches
  come from the latter, so both are needed.
- **Tag & Version vs Sign Image** — versioning gives you traceability and rollback targets;
  signing (Cosign/Sigstore) lets the deploy target *verify* the image wasn't tampered with.
  This is the heart of **supply-chain security** (SLSA, in-toto).

### The artifact is immutable — promote, don't rebuild

A core CI/CD principle: **build once, promote the same artifact**. The exact image tested
in staging is the one deployed to prod. Rebuilding per-environment reintroduces "works on
staging, breaks in prod" drift. This is also what lets GitOps take over after step 6 (see below).

### Deployment strategies (what "Deploy to Prod" can mean)

| Strategy | How it works | Trade-off |
|----------|--------------|-----------|
| **Recreate** | Kill old, start new | Simple, but downtime |
| **Rolling** | Replace instances gradually | No downtime, slow rollback |
| **Blue/Green** | Two full envs, flip traffic | Instant rollback, 2× cost |
| **Canary** | Route a small % of traffic to new version, ramp up | Safest, needs good metrics |

### Where CI/CD ends and GitOps begins

Classic CI/CD is **push-based**: the pipeline reaches into the target environment and runs
`kubectl apply` / `helm upgrade` with prod credentials. That means your CI system holds
cluster admin rights — a large attack surface.

**GitOps** splits the pipeline at step 6: CI builds and pushes the signed image, then writes
the new image tag into a Git repo. An in-cluster operator pulls and applies it. See [GitOps].

### Tooling landscape

- **Pipeline orchestrators:** GitHub Actions, GitLab CI, Jenkins, CircleCI, Tekton, Azure DevOps
- **Build/artifact:** Docker/BuildKit, Bazel, Kaniko; registries: GHCR, ECR, Artifactory, ACR
- **Testing:** Jest/JUnit/pytest (unit), Playwright/Cypress (E2E), k6/Locust (perf)
- **Security:** Semgrep/CodeQL/SonarQube (SAST), Trivy/Grype/Snyk (deps), Cosign (signing)

---

## Relationship to GitOps & MLOps

- **vs [GitOps]:** CI/CD is push-based and linear (ends at deploy). GitOps is pull-based and
  a continuous reconcile loop. In Kubernetes, they pair: **CI builds → GitOps deploys**.
- **vs [MLOps]:** MLOps borrows CI/CD ideas but adds data, experiment tracking, and
  monitoring stages, and is **cyclical** (models decay → retrain). CI/CD ships code; MLOps
  ships models + data.
