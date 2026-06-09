# MLOps — Machine Learning Operations

> Part of the operational-workflows series: **[CI/CD] · [GitOps] · [MLOps] · [LLMOps]**.
> Original trio sourced from an infographic by Vishakha Sadhwani (@vsadhwani).

---

## What It Is

MLOps is the lifecycle for **building, deploying, and maintaining ML models in production**.
It adds two concerns that classic software doesn't have:

1. **Data** is a first-class, versioned dependency (the model is only as good as its data).
2. **Models decay** — the world shifts under a deployed model, so performance degrades over
   time even with zero code changes.

**Mental model:** data → features → train → evaluate → serve → monitor → **retrain (loop)**.
Unlike CI/CD (linear) it is inherently **cyclical** — note step 10 loops back to step 3.

---

## The 10 Steps

| # | Step | What happens |
|---|------|--------------|
| 1 | **Data Ingestion** | Collect/load raw data from sources. |
| 2 | **Feature Engineering** | Transform raw data into model-ready features → **Feature Store** + **Data Validation**. |
| 3 | **Train Model** | Fit the model → **HPO Tuning** (hyperparameter optimization) + **Experiment Tracking**. |
| 4 | **Evaluate Model** | Assess quality → **Accuracy/F1** (predictive metrics) + **Fairness Checks** (bias). |
| 5 | **Register Model** | Store the validated model in a registry with versioning. |
| 6 | **Package & Serve** | Wrap for serving → **vLLM/TRT** (inference engines) + **REST/gRPC** (API protocols). |
| 7 | **A/B Testing** | Compare model versions on live traffic. |
| 8 | **Monitor Inference** | Watch the deployed model → **Data Drift** + **Latency/SLA**. |
| 9 | **Trigger Retraining** | When drift/degradation is detected, kick off retraining. |
| 10 | **Loop back to Step 3** | Return to training — MLOps is a continuous loop. |

---

## Flow

```
Data Ingestion
   │
Feature Engineering ──► Feature Store
   │           └─────────► Data Validation
   │
Train Model ──────────► HPO Tuning
   │         └───────────► Experiment Tracking
   │
Evaluate Model ───────► Accuracy / F1
   │          └──────────► Fairness Checks
   │
Register Model
   │
Package & Serve ──────► vLLM / TRT
   │          └──────────► REST / gRPC
   │
A/B Testing
   │
Monitor Inference ────► Data Drift
   │           └─────────► Latency / SLA
   │
Trigger Retraining
   │
Loop back to Step 3 ──┐
   └──────────────────┘  (continuous)
```

---

## Deeper Dive

### Why MLOps ≠ CI/CD for code

A model has **three** things to version, not one:

```
Reproducible model = Code  ×  Data  ×  Hyperparameters/Config
```

Change any of the three and you get a different model. So MLOps needs **data versioning**
(DVC, lakeFS, Delta Lake) and **experiment tracking** on top of normal Git. This is why the
diagram dedicates steps to feature stores, validation, and experiment tracking.

### Feature Store vs Data Validation (step 2)

- **Feature Store** (Feast, Tecton, Databricks FS, Vertex FS) — a central, versioned repository
  of computed features. Solves two big problems: **reuse** (teams share features) and
  **train/serve skew** (the exact same feature logic is used at training *and* inference time).
- **Data Validation** (Great Expectations, TFDV, Pandera) — schema/quality/distribution checks.
  Catches the silent killers: null spikes, schema changes, out-of-range values, label leakage.

### HPO Tuning vs Experiment Tracking (step 3)

- **HPO (Hyperparameter Optimization)** — search the config space for the best model: grid,
  random, Bayesian (Optuna), Hyperband. Often the biggest accuracy lever after good data.
- **Experiment Tracking** (MLflow, Weights & Biases, Neptune) — log every run's params, metrics,
  artifacts, and lineage so results are **reproducible and comparable**. Without it, "which run
  produced this model?" is unanswerable.

### Evaluate: Accuracy/F1 vs Fairness (step 4)

- **Accuracy/F1** — predictive performance. F1 (harmonic mean of precision & recall) matters
  on **imbalanced** data where raw accuracy lies (99% accuracy is useless if 99% of labels are
  one class). Also: AUC-ROC, precision@k, RMSE/MAE for regression.
- **Fairness Checks** — does the model behave equitably across groups? Metrics like demographic
  parity, equalized odds. Increasingly a compliance requirement (EU AI Act, etc.), not optional.

### Register Model (step 5) — the model registry

A versioned store of trained models with **stage transitions** (Staging → Production → Archived),
lineage back to data/code/experiment, and approval workflows. Tools: MLflow Model Registry,
SageMaker/Vertex registries. This is the model equivalent of CI/CD's artifact registry.

### Package & Serve (step 6) — two axes

This step is two *different* decisions, which is why it branches:

- **Inference engine — vLLM / TRT:** how the model executes.
  - **vLLM** — high-throughput LLM serving via PagedAttention + continuous batching; the
    de-facto open-source LLM server.
  - **TensorRT (TRT) / TensorRT-LLM** — NVIDIA's compiler that fuses/quantizes models for
    max GPU throughput and low latency.
  - (Classic ML uses ONNX Runtime, Triton, TorchServe instead.)
- **API protocol — REST vs gRPC:** how clients call it.
  - **REST/JSON** — universal, human-readable, easy to debug.
  - **gRPC/protobuf** — binary, lower latency, streaming; better for high-QPS internal traffic.

### Serving patterns

- **Online / real-time** — low-latency request/response (the REST/gRPC path above).
- **Batch** — score large datasets on a schedule.
- **Streaming** — score events off a queue (Kafka/Pub-Sub).

### A/B Testing & rollout (step 7)

ML-specific rollout strategies that go beyond CI/CD's blue-green/canary:

- **Shadow / dark launch** — new model gets real traffic but its predictions aren't served;
  you compare offline. Zero user risk.
- **A/B test** — split live traffic between models, measure a *business* metric (CTR, revenue),
  not just offline accuracy. Offline-good ≠ online-good is a constant MLOps surprise.
- **Champion/Challenger** — challenger must beat the incumbent before promotion.

### Monitor Inference (step 8) — the part that makes MLOps a loop

Code monitoring watches latency/errors. ML monitoring must *also* watch whether the model is
still **correct** as the world changes:

- **Data Drift** — input distribution shifts vs training (e.g. new user behavior). Detect with
  PSI, KL-divergence, KS-test. (Related: **concept drift** = the input→output relationship itself
  changes; **prediction drift** = output distribution shifts.)
- **Latency/SLA** — serving p50/p95/p99 latency, throughput, error rate, GPU utilization/cost.
- The catch: **ground-truth labels often arrive late or never**, so drift on *inputs* is your
  early-warning signal before accuracy can even be measured. Tools: Evidently, Arize, WhyLabs, Fiddler.

### Trigger Retraining & the loop (steps 9–10)

When drift or degradation crosses a threshold (or on a schedule, or when enough new labeled
data arrives), the pipeline **retrains** and re-enters at step 3 — new model is trained,
evaluated, registered, and rolled out, **gated by the same evaluation checks** so a worse model
never reaches prod. This closed feedback loop is the single biggest structural difference from
CI/CD and GitOps. The discipline of automating this loop is sometimes called **CT (Continuous
Training)** — the third "C" unique to ML.

### Maturity levels (Google's framing)

- **Level 0** — manual, notebook-driven, manual deploys.
- **Level 1** — automated *training pipeline*; retrain on new data, but pipeline changes are manual.
- **Level 2** — full CI/CD/**CT** automation of the pipeline itself; new ideas reach prod rapidly and safely.

---

## Relationship to CI/CD, GitOps & LLMOps

- **vs [CI-CD]:** MLOps reuses CI/CD's build/test/promote ideas but adds data versioning,
  experiment tracking, model evaluation/registry, and live monitoring — and closes the loop
  with retraining (Continuous Training).
- **vs [GitOps]:** GitOps can serve as the **deployment substrate** for MLOps — model-serving
  manifests and infra declared in Git, reconciled continuously, while MLOps owns the
  data→train→evaluate→monitor→retrain lifecycle on top.
- **vs [[LLMOps]]:** LLMOps **sits on top of** MLOps — the foundation model an LLM app serves
  was produced by an MLOps pipeline. Where MLOps *trains* models, LLMOps *composes, grounds,
  evaluates, and guards* a pretrained one, adding an AI-security + governance spine.
