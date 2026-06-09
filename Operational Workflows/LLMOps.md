# LLMOps — Large Language Model Operations (a.k.a. GenAIOps)

> Part of the operational-workflows series: **[CI/CD] · [GitOps] · [MLOps] · [LLMOps]**.
> The natural sequel to [[MLOps]] for generative-AI applications.

---

## What It Is

LLMOps is the lifecycle for **building, deploying, and operating applications on top of
large language models** — chatbots, RAG assistants, copilots, and agents. The defining
difference from [[MLOps]] is the starting point:

> **MLOps builds the model. LLMOps wraps, grounds, evaluates, and guards a model someone
> else already built.**

You rarely train weights. Your levers are **composition** (prompts, retrieved context,
tools), **evaluation** (because "correct" is fuzzy and subjective), and **governance +
security** (because the model is a non-deterministic, externally-controlled component that
can be manipulated and can leak data).

**Mental model:** scope → govern → curate data → pick a model → compose (prompt + RAG +
tools) → evaluate → guard → serve → monitor → **iterate on prompts/retrieval (loop)**.
The cheap, fast iteration lever is the **prompt and the retrieval corpus**, *not* retraining.

### LLMOps vs MLOps at a glance

| | [[MLOps]] | LLMOps |
|--|-----------|--------|
| You start from | Raw data + a blank model | A **pretrained foundation model** you don't own |
| Primary unit of change | A trained model (weights) | A **prompt / chain / agent + retrieval corpus** |
| Main lever | Train / tune weights | **Compose**; fine-tune is the exception |
| "Correct" is | Measurable (accuracy/F1 vs labels) | **Fuzzy & subjective** → eval is the hard problem |
| Cost shape | Mostly training (upfront) | Mostly **inference / tokens** (ongoing, per-call) |
| New failure modes | Drift, bias | **Hallucination, prompt injection, data exfiltration, jailbreaks** |
| Latest data via | Retraining | **Retrieval (RAG)** — no retraining needed |
| Iteration loop returns to | Train (step 3) | Prompt/Retrieval (step 5) |

---

## The 10 Steps

| # | Step | What happens |
|---|------|--------------|
| 1 | **Define Use Case** | Scope + **success criteria**. What does "good" look like, who's the user, what's out of scope. |
| 2 | **Govern & Approve** | **The "what's allowed" gate** → *Risk Classification* (EU AI Act tier, NIST AI RMF, ISO 42001) + *Use Approval & Model/Data Allow-list*. Defines capability, data, and human-oversight boundaries **before** any build. |
| 3 | **Curate Data** | Assemble the two datasets you *do* own → *Knowledge Corpus* (what RAG retrieves over) + *Eval Set* (golden questions). Data rights, PII, licensing checked here. |
| 4 | **Select Model** | *Closed API vs Open Weights* + *Managed vs Self-hosted*. Trade off capability, context window, $/token, latency, fine-tunability, and **data residency/privacy**. |
| 5 | **Prompt Engineering** | System prompts, few-shot examples, output schemas / structured outputs. The primary, cheapest iteration lever. |
| 6 | **Orchestration & RAG** | *Retrieval* (chunk → embed → vector store → rerank → augment) + *Tool calling & Agent chains* (with **scoped permissions**). |
| 7 | **Evaluate** | *LLM-as-Judge* + *Human Review* against golden sets, offline **and** online. There is no single accuracy number — you build a rubric. |
| 8 | **Guardrails & Security** | *Input filter* (prompt-injection, PII) + *Output validation* (grounding/citation, toxicity, leakage). Maps to the **OWASP LLM Top 10**. |
| 9 | **Deploy & Serve** | *Model Gateway* (routing, **semantic cache**, rate-limit, fallback routing) + *AuthN/Z & Policy Enforcement*. The runtime governance enforcement point. |
| 10 | **Monitor, Audit & Iterate** | *Tracing + Token Cost/Latency* + *Compliance Audit Logs + Red-teaming*. Capture feedback → **loop back to Step 5** (prompt/retrieval). |

---

## Flow

```
Define Use Case
   │
Govern & Approve ───────► Risk Classification (EU AI Act / NIST AI RMF / ISO 42001)
   │           └──────────► Use Approval & Model/Data Allow-list
   │                                         ┌─ governance spine ─┐
Curate Data ────────────► Knowledge Corpus   │                    │
   │          └───────────► Eval Set (golden) │                    │
   │                                          │                    │
Select Model ───────────► Closed API / Open Weights                │
   │           └───────────► Managed / Self-hosted (vLLM)          │
   │                                          │                    │
Prompt Engineering ◄───────────────────┐     │                    │
   │                                    │     │                    │
Orchestration & RAG ────► Retrieval (vector store · rerank)        │
   │             └─────────► Tools & Agents (scoped permissions)   │
   │                                    │     │                    │
Evaluate ───────────────► LLM-as-Judge  │     │                    │
   │       └───────────────► Human Review│     │                    │
   │                                    │     │                    │
Guardrails & Security ──► Input filter (injection · PII)           │
   │             └─────────► Output validation (grounding · leak)  │
   │                                    │     │                    │
Deploy & Serve ─────────► Gateway (cache · fallback) ◄── enforce ──┘
   │           └───────────► AuthN/Z & Policy Enforcement
   │                                    │
Monitor, Audit & Iterate ► Tracing · Cost · Latency
   │              └──────────► Audit Logs · Red-teaming ◄── audit ──┐
   └──── feedback ↻ to Prompt Engineering (step 5) ────────────────┘
```

---

## Deeper Dive

### Why governance is step 2, not a footnote (what AI is *allowed* to do)

Unlike the other three workflows — which govern themselves through technical gates (tests,
policy admission, eval metrics) — GenAI's binding constraints are **external and regulatory**,
and they decide *whether you may ship at all*. So "what the AI is allowed to do" is decided
**up front** (step 2), **enforced at runtime** (step 9 gateway), and **audited continuously**
(step 10). That's a governance *spine*, anchored by one explicit step.

"What AI is allowed to do and not" decomposes into:

- **Use boundaries** — acceptable-use policy; prohibited use cases (EU AI Act *unacceptable-risk*
  tier: social scoring, manipulative systems, etc.).
- **Capability boundaries** — which tools/actions an agent may invoke, and the blast radius
  (OWASP **LLM06: Excessive Agency** — the #1 emerging agentic risk).
- **Data boundaries** — what data may be sent to which model/provider (residency, PII, IP),
  retention, training-opt-out. This is *why* model selection (step 4) is a governance decision.
- **Human oversight** — where a human must stay in the loop; contestability; right to explanation.
- **Accountability** — model/system cards, audit logs, lineage, a named owner of record.

**Frameworks to anchor to:**

| Framework | What it gives you |
|-----------|-------------------|
| **EU AI Act** | Risk tiers (unacceptable / high / limited / minimal); GPAI obligations live since Aug 2025, high-risk phasing through 2026–27. |
| **NIST AI RMF + GenAI Profile** (AI 600-1) | Operating model: **Govern · Map · Measure · Manage**. |
| **ISO/IEC 42001** | Certifiable AI management system (the "ISO 27001 for AI"). |

**Players:** Credo AI, Holistic AI, IBM watsonx.governance, Fiddler, Arthur, CalypsoAI;
**Microsoft Purview** for data-governance-of-AI.

### AI security is its own discipline — the OWASP LLM Top 10

GenAI adds threat classes that classic appsec doesn't cover. The key ones:

- **Prompt injection (LLM01)** — direct, and **indirect**: malicious instructions hidden in
  *retrieved* documents. RAG turns your knowledge corpus into an attack surface.
- **Sensitive-information disclosure (LLM02)** & **system-prompt leakage (LLM07)**.
- **Supply chain (LLM03)** & **data/model poisoning (LLM04)** — untrusted models/datasets.
- **Improper output handling (LLM05)** — model output flowing unsanitized into SQL/shell/UI.
- **Excessive agency (LLM06)** — an agent with too many permissions doing damage.
- **Vector & embedding weaknesses (LLM08)** — the new RAG-specific class.
- **Misinformation (LLM09)** & **unbounded consumption (LLM10)** — cost/DoS via token amplification.

The principle: **guardrails sit on *both* sides** — input (injection/PII detection) *and*
output (grounding/citation, toxicity, leakage) — reinforced at the gateway (authn/z, rate
limits), in audit logs, and via **red-teaming** before and after launch.

- **Threat models:** OWASP LLM Top 10 (2025), **MITRE ATLAS**.
- **Guardrails:** NVIDIA **NeMo Guardrails**, **Guardrails AI**, Meta **Llama Guard / Prompt
  Guard**, Azure **AI Content Safety + Prompt Shields**, **AWS Bedrock Guardrails**, Google
  **Vertex Model Armor**, **Lakera Guard**.
- **AI red-team / model security:** Microsoft **PyRIT**, NVIDIA **garak**, **HiddenLayer**,
  **Protect AI** (now Palo Alto Networks), **Robust Intelligence** (now Cisco).

### Select Model — the decision with no MLOps analog (step 4)

You're choosing, not building. The axes:

- **Closed API vs Open Weights** — closed (GPT, Claude, Gemini) = best capability, zero ops,
  but data leaves your boundary and you're price/rate-limited. Open weights (Llama, Mistral,
  Qwen, DeepSeek, Gemma) = data stays in, full control, fine-tunable, but you run the GPUs.
- **Managed vs Self-hosted** — provider API / Bedrock / Vertex / Azure OpenAI vs self-serve
  on **vLLM / TGI / TensorRT-LLM**.
- **Criteria:** capability on *your* eval set, context window, **$/token**, latency, licensing,
  fine-tunability, and **data residency** — the last is a hard governance gate, not a preference.

### Orchestration & RAG (step 6)

**RAG anatomy:** `retrieve → (rerank) → augment → generate`. Retrieval beats retraining for
freshness — you update the corpus, not the weights. Chunking strategy, embedding model, and a
reranker matter more than people expect; most "the LLM is wrong" bugs are actually retrieval bugs.

**Agents & tool-calling:** single-shot vs multi-step agents trade reliability and cost for
capability. Give tools **least-privilege, scoped permissions** (ties back to *Excessive Agency*).
**MCP (Model Context Protocol)** is the emerging standard for permissioned, portable tool access.

- **Orchestration:** LangChain/**LangGraph**, **LlamaIndex**, Microsoft **Semantic Kernel /
  AutoGen**, **CrewAI**, **DSPy**.
- **Vector / retrieval:** **Pinecone**, **Weaviate**, **Qdrant**, **Milvus**, **pgvector**,
  **Azure AI Search**.

### Evaluation — the hard problem (step 7)

There is no ground-truth label for "write a good summary." So evaluation is constructed:

- **LLM-as-judge** — a strong model scores outputs against a rubric (reference-free or
  reference-based). Cheap and scalable, but the judge itself must be validated.
- **Golden sets** — curated input→expected-behavior pairs you regression-test against.
- **Online eval** — real traffic signals (thumbs, task completion), because offline-good ≠
  online-good. Watch for **eval drift** as the corpus and models change.

**Players:** **LangSmith**, **Arize Phoenix**, **Ragas**, **DeepEval**, **TruLens**,
**Braintrust**, **Patronus AI**, **Galileo**, W&B **Weave**, OpenAI **Evals**.

### Deploy & Serve — the policy enforcement point (step 9)

The **model gateway** is where governance becomes runtime-real: authn/z, per-team rate limits
and budgets, model routing (cheap model first), **semantic caching** (answer-reuse to cut cost
and latency), and **fallback routing** (provider down → secondary). Stream responses for UX.

- **Gateways:** **Portkey**, **LiteLLM**, **Kong AI Gateway**, **Cloudflare AI Gateway**,
  Azure **API Management** AI gateway.

### Cost, latency & the fine-tune-vs-RAG-vs-prompt decision

- **Token economics** dominate run cost — measure **$/request** and **$/resolved-task**, not
  just $/token. Caching and routing are the biggest levers.
- **Decision tree:** exhaust **prompt engineering**, then **RAG**, before touching **weights**.
  Fine-tuning is for *behavior/format/style* and latency, **not** for injecting fresh knowledge
  (that's RAG's job). Most teams never need it.

### Maturity signal

- **Level 0** — prompts in a notebook, no eval, no guardrails, no cost visibility.
- **Level 1** — versioned prompts, an eval set, basic guardrails, a gateway with budgets.
- **Level 2** — governed (risk-classified, audited), continuously evaluated and red-teamed,
  automated prompt/retrieval iteration loop.

---

## Relationship to the Other Workflows

- **vs [[MLOps]]:** LLMOps **sits on top of** MLOps — the foundation model it serves was
  produced by an MLOps pipeline. LLMOps adds prompt/RAG/agent composition, subjective
  evaluation, and an AI-security + governance spine that classic MLOps doesn't need.
- **vs [CI/CD]:** the app *around* the LLM (prompts-as-code, eval suites, guardrail configs)
  still ships through a normal CI/CD pipeline — evals become a CI gate.
- **vs [GitOps]:** serving infra, gateway config, and model/prompt versions can be declared in
  Git and deployed via a GitOps operator, inheriting drift detection and audit-by-Git-history.
