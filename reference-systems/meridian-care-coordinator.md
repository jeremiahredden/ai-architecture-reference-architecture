# Reference System: Meridian Care Coordinator

> **Status.** Reference architecture, current as of 2026-Q2 review. Built from real systems and anonymized; the client (Meridian Health) is a fictional regulated healthcare SaaS used across the sibling reference architecture repos. **Audience.** Architects and staff engineers designing a regulated-domain agent system; engineering leaders sizing an agent platform investment; reviewers comparing this reference against their own design. **Cross-links.** Pattern selection rationale in [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md); engineering practice in the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture); threat model and attack surface in the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)'s `agent-security/` folder.

---

## 1. Executive summary

The Meridian Care Coordinator is a clinical-domain agent system embedded in Meridian Health's clinical workflow platform. It assists clinical staff (RNs, MAs, care coordinators, and — with stricter approval boundaries — physicians) in coordinating patient care across the platform: looking up clinical guidelines, checking drug interactions, drafting structured messages to the patient, drafting clinical orders for clinician approval, scheduling follow-up appointments, and routing internal hand-offs.

Meridian operates as an HIPAA business associate serving ~240 hospital customers, with ~350 engineering staff, AWS-primary infrastructure, an Okta → IAM Identity Center identity model, and a separate FedRAMP-track GovCloud Organization for evaluation work. The Care Coordinator launched as a pilot on three customer hospitals in 2025-Q4 and is GA-targeted for all 240 hospitals through 2026.

The architecture commits to:

- A **supervisor / worker agent topology** with model-tier routing — Opus-class for clinical reasoning and the supervisor role, Sonnet-class for the orchestration / planning worker, Haiku-class for classification and query-rewriting subtasks.
- **Hybrid retrieval (BM25 + vector) with a Cohere reranker** over a per-tenant-namespaced pgvector store, plus a **targeted FDA drug-interaction graph** augmenting interaction-class queries only.
- **Streaming responses** for clinician chat interactions, **async with callback** for bulk coordination tasks, and **mandatory human-in-the-loop** for any operation that takes a real-world side effect (orders, patient messages, scheduling changes).
- **Per-hospital tenant isolation** across every layer — vector namespace, conversation store, retrieval client, audit log, cost accounting — enforced inside platform-level abstractions rather than relied on from calling code.
- **Designed-in guardrails** at the AI-gateway layer (PHI filter, per-tenant routing, prompt-injection screening), at the tool registry (per-role authorization for any side-effect tool), and at the retrieval client (mandatory tenant filter).
- **Per-interaction cost target** of approximately $0.18 at production scale, with caching and tier-routing levers identified to flex 30–40% in either direction.

The system is sized to handle approximately 3,000 coordinator interactions/day across all 240 hospitals at GA, with per-interaction latency budgets of 6 seconds for chat-shaped questions and 12 seconds for multi-step coordination tasks. Multi-tenancy posture is shared-model + isolated-data; one customer hospital is on a dedicated-infrastructure variant under a separate contract and is the canary for the dedicated pattern.

This document is structured as nine sections: (1) executive summary, (2) workload and non-functional requirements, (3) the architectural decision records that drive the system, (4) reference architecture diagram, (5) three data-flow walkthroughs, (6) the eval surface, (7) the cost model, (8) findings (`ARCH-CARE-001` through `ARCH-CARE-020`), and (9) sprint sequencing from pilot to GA.

---

## 2. Workload and non-functional requirements

### 2.1 What the Care Coordinator does

The Care Coordinator is invoked from four entry points in the Meridian clinical platform:

1. **Clinician chat panel.** An in-app chat panel where clinical staff ask coordination questions — "what is the post-discharge follow-up protocol for a CHF patient on the new pathway?" — and receive cited answers with optional follow-up actions (draft a patient message, draft a follow-up appointment, draft an order). The bulk of interactions (~80% of volume) come through this path. Latency budget: streaming response with time-to-first-token under 1.5 seconds, time-to-useful-content under 4 seconds.
2. **Inline assistance in the order-entry workflow.** When a clinician is entering an order, the Care Coordinator surfaces relevant drug interactions, dosing references, and prior-authorization considerations from the same retrieval and reasoning stack. Latency budget: streaming, time-to-first-token under 1 second.
3. **Async coordination tasks.** Multi-step tasks initiated by the care-coordinator role: "for every CHF patient discharged in the last 7 days who has not scheduled their 14-day follow-up, draft an outreach message and a suggested appointment slot." These run as background jobs with per-task callback notification. Latency budget: 30–300 seconds per patient.
4. **Triage and escalation.** When the Care Coordinator's confidence is below the escalation threshold, the interaction routes to a human escalation queue (clinical pharmacist, senior coordinator, or attending depending on the topic). Latency budget: the routing decision is instant; the human response is governed by the hospital's own SLA.

### 2.2 The five questions, answered

Per the framework in the architecture sibling's [rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md):

1. **Retrieval corpus.** ~50,000 chunks across (a) clinical guidelines from professional societies (AHA, ACC, IDSA, etc., licensed and ingested quarterly), (b) hospital-tenant-specific protocols and order sets (uploaded and maintained by each hospital's clinical informatics team), and (c) the Meridian-curated drug-interaction graph (~12,000 nodes, ~80,000 edges, sourced monthly from the FDA Structured Product Labels feed).
2. **Good.** Every clinical claim must cite a guideline or protocol source. Every drug-interaction claim must cite the FDA SPL source. Every order-entry assistance must be human-approved before any order is created. Every action that messages or schedules a patient must be human-approved before transmission. Quality is measured by a clinician-reviewed eval set (see §6).
3. **Latency budget.** Streaming chat: time-to-first-token < 1.5s, time-to-useful < 4s, total < 6s. Async coordination: 30–300s per patient. Inline order-entry: time-to-first-token < 1s.
4. **Cost budget.** Per chat interaction: target $0.15–$0.22; circuit-break at $0.50 per interaction, $1.50 per session, $50 per tenant per day. Async tasks have their own budgets governed by the task's expected fan-out.
5. **Corpus changes.** Clinical guidelines refresh quarterly (publishing-society release cadence). Hospital protocols refresh continuously (clinical informatics edits). Drug-interaction graph refreshes monthly (FDA SPL feed). All refreshes go through the corpus-as-product pipeline with versioning, eval cross-check, and a per-source freshness SLO.

### 2.3 Non-functional requirements (one-page table)

| Dimension | Requirement |
|---|---|
| Availability | 99.9% during clinical operating hours per hospital (effectively 24/7 due to multi-tenant coverage); 99.5% overall. |
| Latency (chat) | TTFT < 1.5s p95; TTU < 4s p95; total < 6s p95. |
| Latency (async) | < 300s per patient p95. |
| Quality | Clinical answer correctness ≥ 95%; interaction-answer correctness ≥ 98%; citation accuracy ≥ 99%; faithfulness (grounded in retrieved content) ≥ 98%. |
| Cost | $0.15–$0.22 per chat interaction at target scale; budget-as-circuit-breaker hard limits per session / tenant. |
| Tenant isolation | Per-hospital isolation across vector namespace, conversation store, audit log, cost accounting; enforced in platform abstractions. |
| Compliance | HIPAA business-associate scope. BAA-covered models only for any context that may contain PHI. Audit log meets HIPAA §164.312(b) requirements. SOC 2 Type II controls applied. HITRUST every two years. FedRAMP work parallel in GovCloud. |
| Human-in-the-loop | Mandatory for any side-effect operation (orders, patient messages, scheduling changes). |
| Observability | Per-interaction trace with prompt-version, model-version, retrieved doc IDs, tool calls, costs, latency components. 100% trace sampling for clinical interactions; aggregated metrics for cost dashboard. |
| Rollback | Pinned versions of code, prompt, model, dataset; rollback to last-good within 15 minutes. |
| Data residency | All PHI-touching data stays in US regions; GovCloud variant exists for the federal evaluation track. |

---

## 3. Architectural Decision Records

The architecture is the product of twelve numbered ADRs. Each names the decision, the alternatives considered, the trade-off, and the cross-link to the relevant pattern document. The ADRs are listed in the order they were resolved during the pilot design phase.

### ADR-CARE-001 — Agent topology: supervisor / worker

**Decision.** Supervisor / worker topology. A supervisor agent (Opus-class) receives the user request, plans the sub-tasks, and dispatches each to a specialized worker (clinical-knowledge worker on Opus-class, drafting / formatting worker on Sonnet-class, classification / routing worker on Haiku-class). The supervisor consolidates worker outputs into the final response.

**Alternatives considered.**
- *Single-agent ReAct loop.* Simpler, cheaper per-call, but quality on the clinical-reasoning subtasks measurably degraded — the same model handling both planning and clinical reasoning lost precision on the latter.
- *Pure workflow with embedded LLM steps.* Considered for the order-entry inline path (and is the architecture there). For the conversational chat path, the variety of question shapes made the static workflow brittle.
- *Multi-agent swarm.* Operational and observability complexity exceeded the benefit; the supervisor / worker pattern captures the parallelism without the coordination cost.

**Trade-off.** Higher per-interaction cost than single-agent (3 model calls per turn rather than 1), in exchange for better quality on the clinical-reasoning subtasks and clearer debuggability (each worker has its own trace span).

**Cross-link.** Engineering: `agent-engineering/agent-loop-design.md` (coming). Architecture: `reference-patterns/agent-topologies.md` (coming).

### ADR-CARE-002 — Model strategy: tier-routed

**Decision.** Three model tiers, deployed deliberately:
- **Opus-class** (current frontier reasoning model with BAA coverage) for the supervisor and the clinical-knowledge worker.
- **Sonnet-class** for the drafting / formatting worker (composing patient messages, formatting structured outputs, summarizing retrieved content).
- **Haiku-class** for classification (which retrieval surface to use, which question class, conversational context detection) and for query rewriting.

All three are hosted under a BAA-covered configuration. The system prompts are versioned per worker per environment.

**Alternatives considered.**
- *Opus-class everywhere.* Higher quality on the workers' subtasks, but the per-interaction cost would have nearly tripled and the latency would have exceeded the chat budget.
- *Open-weight self-hosted for the cheaper workers.* Considered for the Haiku-tier classifier; rejected on operational cost (the team did not have GPU capacity or the inference platform investment) and on the fact that hosted Haiku-class with BAA was cheap enough that the operational overhead would not pay back.

**Trade-off.** Multi-model dependency complexity (three model versions to pin, three providers' release cadences to track) in exchange for cost / latency optimization and per-task model-fit.

**Cross-link.** Architecture: `model-strategy/model-routing-and-tiering.md` (coming). Engineering: `model-lifecycle/model-registry.md` (coming).

### ADR-CARE-003 — Retrieval architecture: hybrid + reranker + query-rewrite + targeted graph

**Decision.** For the general clinical / protocol retrieval surface: hybrid retrieval (BM25 + vector, RRF merge) returning top-50, then Cohere Rerank-3.5 returning top-8 to format into the prompt. Query rewriting (Haiku-tier) runs for second-turn-and-later conversational queries. For interaction-shaped queries (detected by the Haiku-tier classifier), the FDA drug-interaction graph is queried in parallel with the hybrid retrieval, and graph results are formatted alongside chunk results.

**Alternatives considered, with measured rationale.**
- Naive vector-only: 74% recall on the clinical eval, rejected.
- Hybrid alone: 89% recall, 71% precision@5 — recall was good but the reranker pass measurably lifted precision.
- Hybrid + reranker: 89% recall, 88% precision@5 — the production baseline for non-interaction questions.
- Query rewriting added on top: lifted second-turn-and-later recall from 64% to 86% on the conversational subset.
- Graph as a *general* retrieval surface (corpus-wide knowledge graph): lift over hybrid+reranker was ~3 points on the clinical-eval, but the maintenance cost (continuous corpus drift, entity-extraction quality drift) was assessed at ~2 FTEs of ongoing work the team did not have. Targeted graph for interaction queries only: lift on interaction-question correctness from 81% to 96%, with maintenance scoped to the FDA SPL feed (monthly batch ingest, ~0.2 FTE of ongoing work).

**Trade-off.** Multiple retrieval surfaces increase operational complexity, but each is justified by measured quality lift on the eval set.

**Cross-link.** Architecture: [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md). Engineering: `rag-engineering/retrieval-engineering.md` (coming).

### ADR-CARE-004 — Data architecture: per-tenant pgvector + relational metadata + curated graph

**Decision.** pgvector inside the existing Aurora Postgres footprint as the vector store, with one Aurora cluster per AWS region and per-hospital tenant namespacing via a `tenant_id` column that is enforced as a mandatory filter inside the retrieval-client wrapper. Document and chunk metadata in the same Postgres database (joined via foreign keys to the vector index). The drug-interaction graph lives in a separate small Aurora cluster (~12K nodes, ~80K edges) — Postgres rather than a dedicated graph DB, because at this scale the recursive-CTE pattern is fast enough and the operational simplicity of "one database family" is valuable.

**Alternatives considered.**
- *Dedicated vector DB (Pinecone, Weaviate, Qdrant).* Higher throughput at scale, but the Care Coordinator's volume does not yet justify the operational addition, and the data-residency / BAA story is harder.
- *Dedicated graph DB (Neo4j).* For 12K nodes, the dedicated graph engine adds operational cost without proportional benefit. If the graph grows past ~500K nodes, this is the revisit threshold.

**Trade-off.** Single data-platform family (Aurora Postgres + extensions) for operational simplicity, at the cost of leaving some performance on the table that a dedicated stack could capture. Revisit criteria documented.

**Cross-link.** Architecture: `data-architecture-for-ai/vector-store-architecture.md` (coming) and `data-architecture-for-ai/knowledge-graph-augmentation.md` (coming).

### ADR-CARE-005 — Embedding strategy: text-embedding-3-large @ 1536 with pinned version

**Decision.** OpenAI `text-embedding-3-large` at 1536 dimensions, pinned to a specific version-string in the deployment manifest, with full re-embedding triggered on any version change. Embedding model treated as a first-class dependency in the same release artifact as code, prompt, and model versions.

**Alternatives considered.**
- *Cohere embed-v3* — comparable quality on the clinical eval, slightly lower cost, but the BAA story was less mature at the time of decision.
- *Self-hosted open-weight embeddings* — rejected on operational cost.

**Trade-off.** Vendor concentration risk on OpenAI for embeddings even while frontier reasoning is on Anthropic; mitigated by the embedding-migration playbook in `model-lifecycle/model-deprecation-playbook.md` (coming).

### ADR-CARE-006 — Integration architecture: streaming chat, async with callback for coordination, mandatory HITL for side effects

**Decision.**
- **Chat panel (entry point 1).** Server-Sent Events (SSE) streaming. The supervisor's final answer streams to the chat panel; intermediate worker calls do not stream (they are completed before the supervisor's final pass begins).
- **Inline order-entry (entry point 2).** Streaming with a tight latency budget.
- **Async coordination (entry point 3).** Initiated via API call that returns a task ID; the task runs on a worker pool (AWS Step Functions for orchestration, with Lambda functions for the per-patient agent loops); completion notification via webhook to the clinical platform.
- **Side-effect operations (orders, patient messages, scheduling changes).** Mandatory human-in-the-loop: the Care Coordinator drafts the side effect; the clinical staff approves (or edits + approves) in the existing platform UI; only on approval is the side effect committed.

**Alternatives considered.**
- *Synchronous-only HTTP for everything.* Rejected because the async coordination tasks exceed any reasonable HTTP timeout.
- *Auto-execute side effects with post-hoc review.* Rejected as incompatible with the clinical responsibility model and with regulatory posture.

**Cross-link.** Architecture: `integration-architecture/sync-vs-async-vs-streaming.md` (coming), `integration-architecture/human-in-the-loop-boundaries.md` (coming).

### ADR-CARE-007 — Guardrail placement: AI-gateway + tool-registry + retrieval-client wrapper

**Decision.** Three platform-level chokepoints:
- **AI gateway** in front of every model call. Handles per-tenant routing, authentication, cost accounting, request / response logging, PHI re-screening on outputs, prompt-injection screening on inputs derived from outside-trust sources (e.g., patient-supplied messages quoted into a Care Coordinator query), and per-tenant rate limiting.
- **Tool registry** mediates every tool call. Each tool has declared authorization requirements (role, scope, target-tenant); the registry evaluates them before allowing the call; the registry logs every call with the agent state at the time. Tools with side effects always include the HITL hop before execution.
- **Retrieval-client wrapper** mediates every retrieval. Per-tenant filter is injected mandatorily and verified after retrieval; query is logged; retrieval results are logged at the doc-ID level for traceability.

**Alternatives considered.** Inline calling-code controls — rejected as architecturally fragile (one new code path skips the controls).

**Cross-link.** Architecture: `guardrails-and-policy-architecture/ai-gateway-pattern.md`, `tool-call-authorization.md`, `retrieval-scope-enforcement.md` (all coming).

### ADR-CARE-008 — Multi-tenancy: per-hospital namespace across all layers; dedicated-infrastructure variant for premium

**Decision.** Per-hospital `tenant_id` is a first-class dimension across every layer: vector namespace, conversation store partition, audit log scope, cost-accounting attribution, retrieval filter, prompt-context injection, tool-call authorization scope. The default is shared-model + isolated-data. One hospital (the largest customer in the pilot) is on a dedicated-infrastructure variant: dedicated Aurora cluster, dedicated AI-gateway instance, dedicated cost attribution. This variant is the operational canary for the pattern.

**Alternatives considered.**
- *Shared-everything with metadata filtering only.* Rejected for the regulated workload — the failure mode of a missing filter is unacceptable.
- *Dedicated-infrastructure for all hospitals.* Cost-prohibitive at the 240-hospital scale.

**Trade-off.** Most customers get shared-model + isolated-data (the operational sweet spot); the dedicated-infrastructure variant exists for customers willing to contract for it.

**Cross-link.** Architecture: `multi-tenancy-and-isolation/isolation-models.md`, `per-tenant-vector-namespacing.md` (coming).

### ADR-CARE-009 — Memory: short-term context with running summary, long-term episodic via structured event log

**Decision.** Short-term memory (within a single chat session) is verbatim conversation history up to a token budget, with a Sonnet-tier running summary that replaces the older verbatim turns when the budget is approached. Long-term memory (across sessions for the same clinical staff member) is a structured event log — significant decisions, escalations, and unresolved follow-ups — surfaced via retrieval rather than included by default. The structured event log is not patient-identified by default (it tracks the clinician's pattern of work, not patient details).

**Alternatives considered.**
- *Pure verbatim with no summarization.* Hit the context budget too quickly on long sessions.
- *Full long-term memory included by default.* Too noisy; degraded quality.

### ADR-CARE-010 — Observability: 100% trace sampling for clinical interactions

**Decision.** Every interaction generates a full trace (per supervisor call, per worker call, per retrieval, per tool call) with attributes for prompt version, model version, model parameters, input / output tokens, cost, latency components, retrieved doc IDs, judge scores. Traces exported to the existing platform observability stack (Datadog) with PHI-redaction in the pipeline. 100% sampling is affordable at the projected volume (~3K interactions/day) and is required by the audit posture.

**Cross-link.** Engineering: `observability-and-telemetry/trace-and-span-design.md`, `llm-call-instrumentation.md` (coming).

### ADR-CARE-011 — Cost: per-interaction target with circuit-breaker

**Decision.** Per-chat-interaction cost target $0.15–$0.22 at the projected workload mix. Cost-as-circuit-breaker is configured at three levels: $0.50 per single interaction, $1.50 per session, $50 per tenant per day. Exceeded circuit breakers degrade the feature gracefully (the next interaction returns a "please try again or contact your clinical supervisor" response with no LLM call) and page on-call.

**Cross-link.** Engineering: `cost-and-finops/cost-budget-circuit-breaker.md` (coming).

### ADR-CARE-012 — Release artifact: pinned everything

**Decision.** Every Care Coordinator release pins: code version (git SHA), per-worker system prompt versions (semantic versioned), per-tier model versions (full provider model strings, not aliases), embedding model version, dataset versions (clinical guidelines version-string, drug-interaction graph version-string, eval-suite version-string). Rollback restores the entire pinned set.

**Cross-link.** Engineering: `cicd-and-eval-gates/release-artifacts-for-ai.md` (coming).

---

## 4. Reference architecture diagram

```
                                         clinician
                                             │
                                             ▼
              ┌──────────────────────────────────────────────────────────┐
              │              Clinical platform (existing)                 │
              │   ┌──────────────────┐  ┌───────────────────────────┐    │
              │   │ Chat panel (SSE) │  │ Order-entry inline assist │    │
              │   └────────┬─────────┘  └─────────────┬─────────────┘    │
              │            │                          │                  │
              └────────────┼──────────────────────────┼──────────────────┘
                           │                          │
                           ▼                          ▼
            ┌──────────────────────────────────────────────────────────┐
            │                       AI Gateway                          │
            │  - per-tenant routing       - cost accounting             │
            │  - auth (Okta → JWT)        - input PHI re-screen         │
            │  - prompt-injection screen  - output PHI re-screen        │
            │  - per-tenant rate limit    - request / response logging  │
            └─────────────────────────────┬────────────────────────────┘
                                          │
                                          ▼
                ┌───────────────────────────────────────────────┐
                │             Supervisor (Opus-class)           │
                │       plans sub-tasks; consolidates answer    │
                └─────┬────────────┬─────────────┬──────────────┘
                      │            │             │
            ┌─────────▼──┐  ┌──────▼──────┐  ┌──▼──────────────┐
            │ Classifier │  │  Clinical-  │  │ Drafting /      │
            │  (Haiku)   │  │  Knowledge  │  │ Formatting      │
            │            │  │   Worker    │  │ Worker          │
            │ • question │  │  (Opus)     │  │ (Sonnet)        │
            │   class    │  │             │  │                 │
            │ • retrieval│  │ • reasons   │  │ • patient msgs  │
            │   surface  │  │   over      │  │ • structured    │
            │ • conv     │  │   retrieved │  │   outputs       │
            │   context  │  │   chunks    │  │ • order drafts  │
            └─────┬──────┘  └──────┬──────┘  └─────────────────┘
                  │                │
        ┌─────────▼─────────┐      │
        │  Query rewriter   │      │
        │  (Haiku)          │      │
        │  for turn 2+      │      │
        └─────────┬─────────┘      │
                  │                │
                  ▼                │
        ┌────────────────────────────────────────────────────┐
        │           Retrieval-client wrapper                  │
        │  - mandatory tenant_id filter                       │
        │  - mandatory provenance metadata capture            │
        │  - per-call doc-ID logging                          │
        └─────┬──────────────┬──────────────┬────────────────┘
              │              │              │
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌─────────────────┐
       │  BM25      │ │  pgvector  │ │  FDA drug-      │
       │  (OS       │ │  (Aurora,  │ │  interaction    │
       │   Postgres │ │   per-     │ │  graph (Aurora) │
       │   FTS)     │ │   tenant   │ │  used only on   │
       │            │ │   ns)      │ │  interaction Q  │
       └────────────┘ └────────────┘ └─────────────────┘
              │              │              │
              └─────┬────────┴──────────────┘
                    │
                    ▼
            ┌────────────────────┐
            │  Cohere Rerank-3.5 │
            │  top-50 → top-8    │
            └─────────┬──────────┘
                      │
                      ▼
            back to Clinical-Knowledge Worker
                      │
                      ▼
              Supervisor consolidates
                      │
                      ▼
            ┌────────────────────────┐
            │   Tool registry         │
            │   - per-role authz      │
            │   - HITL on side effects│
            └─────────┬───────────────┘
                      │
                      ▼
        ┌─────────────┴─────────────┐
        │     Side-effect tools     │
        │  draft_patient_message    │
        │  draft_order              │
        │  draft_appointment        │
        │  ↓ (each requires HITL)   │
        │  Clinician approves       │
        │  ↓                        │
        │  Existing platform writes │
        └───────────────────────────┘

        ┌──────────────────────────────────────────────┐
        │  Observability: 100% trace sampling          │
        │  (prompt ver, model ver, tokens, cost,       │
        │   latency components, doc IDs, judge scores) │
        │  → Datadog (PHI-redacted in pipeline)        │
        └──────────────────────────────────────────────┘

        Async coordination tasks (entry point 3):
            API → Step Functions → per-patient Lambda
                 (same architecture as above, per patient)
            → webhook callback on completion
```

---

## 5. Data-flow walkthroughs

Three representative interactions, traced end-to-end. These are how the architecture is most clearly understood — abstract patterns become concrete when followed through one specific request at a time.

### 5.1 Walkthrough: a single-turn clinical lookup

**User input.** *RN on hospital tenant `mercy-cleveland` asks in the chat panel: "What's our protocol for monitoring INR on a patient just started on warfarin?"*

1. **Chat panel → AI Gateway.** SSE stream initiated. JWT carries the RN's identity and the hospital tenant. Gateway authenticates, attaches tenant context, opens a trace.
2. **Gateway PHI re-screen on input.** Lightweight regex + classifier check. No PHI tokens detected. (Had there been PHI tokens, the request would have proceeded but the trace flag would be set so any downstream logging knows to redact.)
3. **Gateway → Supervisor (Opus-class).** Supervisor receives the question with the per-tenant system prompt overlay and the platform's base system prompt. Supervisor's plan: classify the question; retrieve from the clinical-knowledge surface; compose an answer with citations.
4. **Supervisor dispatches to Classifier (Haiku-class).** Returns: question class = "clinical-protocol", retrieval surfaces = "clinical-guidelines + tenant-protocols", conversational context = "first turn".
5. **Query rewriter — skipped.** First turn, no rewrite needed.
6. **Supervisor dispatches to Clinical-Knowledge Worker (Opus-class).** Worker calls the retrieval-client wrapper with the user's question, retrieval surfaces from the classifier, and the tenant_id `mercy-cleveland`.
7. **Retrieval-client wrapper.** Constructs the BM25 query against the OpenSearch (Postgres FTS) index filtered by `tenant_id = 'mercy-cleveland' OR tenant_id = 'global-guidelines'`. Constructs the vector query against pgvector, same filter. Both run in parallel. Wrapper verifies post-retrieval that every returned chunk carries the expected tenant_id (defense-in-depth).
8. **Cohere Rerank-3.5.** 50 candidates → top-8.
9. **Clinical-Knowledge Worker (Opus-class) reasons over the 8 chunks.** Identifies the relevant clinical guideline ("Warfarin Initiation: Monitoring and Adjustment, AHA 2024") and the tenant-specific protocol ("Mercy Cleveland Anticoagulation Service Protocol v3.2"), composes the answer with citations to both, structured as: recommended INR monitoring frequency, target INR range for the indication, dose adjustment guidance, and a flag to consult anticoagulation service if the patient has specific risk factors.
10. **Supervisor consolidates.** Receives the worker's output, adds a brief introduction, decides whether to offer follow-up actions (in this case: "Want me to draft a 7-day follow-up reminder for this patient?"), and begins streaming.
11. **AI Gateway PHI re-screen on output.** No PHI in the response. Streaming proceeds.
12. **Trace captures.** Prompt versions (`supervisor@2.4.1`, `clinical-knowledge@1.8.0`, `classifier@1.2.0`), model versions (`claude-opus-4-7@2026-04-12`, `claude-haiku-4-5@2025-10-01`), retrieved doc IDs (chunked source references), tokens (in / out / cached), cost ($0.16 for this interaction), latency breakdown (gateway 80ms, classifier 240ms, retrieval 180ms, rerank 320ms, knowledge-worker 2.1s, supervisor 1.4s, total 4.3s end-to-end, TTFT 1.2s).
13. **RN sees streamed answer in chat panel** with citations and an offered follow-up. Clicks the follow-up → triggers walkthrough 5.2.

### 5.2 Walkthrough: a side-effect operation (drafting a patient message)

**User input.** *Continuing the previous chat: RN clicks "Want me to draft a 7-day follow-up reminder for this patient?" The chat panel resolves the active patient context from the EHR and sends "yes, draft it for patient ID `pt-44929`".*

1. **Chat panel → AI Gateway.** Same as before, with the patient context attached as a structured field (not in the prompt).
2. **Gateway PHI re-screen.** Patient context contains PHI; trace flag set; PHI-redaction enabled for the trace export.
3. **Supervisor → Classifier.** Returns: question class = "draft-patient-message", side-effect class = "patient-message-draft".
4. **Supervisor → Drafting / Formatting Worker (Sonnet-class).** Worker receives the patient context (name, indication, language preference) and the prior turn (warfarin monitoring), drafts a message in the patient's preferred language and reading-level configuration.
5. **Supervisor → Tool registry: `draft_patient_message`.** Tool registry checks: does the RN's role have authorization to draft a patient message for this patient on this tenant? Yes. Tool is invoked with the draft content. The tool creates a *draft* — visible in the clinical platform's outbound-message queue, flagged for HITL approval — and returns a draft-ID.
6. **Supervisor's response to the chat.** "I've drafted a 7-day follow-up reminder for [patient initials]. You can review and send it from the outbound queue, or I can show it here. (link to draft)"
7. **Clinician reviews the draft in the existing platform UI.** Edits if needed, approves, sends. *Only on this human action does any message actually leave the platform.*
8. **Trace captures.** Same as 5.1 plus the tool-call span (tool name, authorization decision, draft-ID returned), and the eventual human-action event (when the clinician approves) appended as a follow-up event on the same trace lineage.

This is the critical pattern: the agent *drafts*, the human *approves and sends*. The Care Coordinator never directly causes a side effect in the patient's care.

### 5.3 Walkthrough: an async coordination task

**User input.** *Care coordinator on hospital tenant `mercy-cleveland`, working through a daily worklist, initiates: "For every CHF patient discharged in the last 7 days who has not scheduled their 14-day follow-up, draft an outreach message and a suggested appointment slot."*

1. **Care coordinator UI → API endpoint.** Synchronous API call that:
   - Validates the request scope (the coordinator role can act on patients in this hospital tenant).
   - Estimates the fan-out (resolves the patient list from the EHR; in this case, 14 patients).
   - Estimates the cost (~14 × $0.30 = $4.20; checks against the tenant's daily async budget).
   - Returns a task ID.
2. **AWS Step Functions orchestrates.** One Lambda invocation per patient, fanned out with a concurrency cap of 4 (rate-limit-aware).
3. **Each per-patient Lambda runs the same supervisor / worker stack as the chat path.** Differences: streaming is disabled (output is captured to the task result), and the chat-panel-specific system prompt overlay is replaced with a coordination-task system prompt that biases toward concise, structured outputs.
4. **Each invocation produces two drafts** — a patient outreach message and a suggested appointment slot — and registers both as pending HITL items in the coordinator's worklist.
5. **Step Functions completes the task,** fires a webhook to the coordinator UI with the result summary (14 drafts created, 0 errors, total cost $4.18, links to each draft).
6. **The coordinator reviews and acts on each draft.** Same HITL pattern as the chat-panel side-effect path.

The Step Functions orchestration adds durability (retries, partial-success handling, observability) at the cost of latency overhead per patient (~500ms of orchestration). The trade-off was assessed as worth it for the regulated workload's audit posture.

---

## 6. Eval surface

The Care Coordinator's eval surface is built and maintained as a first-class engineering artifact, with separate suites for the different question classes and dedicated suites for the conversational and side-effect paths. The eval suite is what makes everything else operable; without it, model swaps, prompt edits, retrieval changes, and corpus refreshes would all be uncontrolled.

| Suite | Size | Refresh cadence | What it measures | Quality SLO |
|---|---|---|---|---|
| Clinical golden set | 200 cases | Quarterly + after every clinical incident | Retrieval recall, retrieval precision, answer correctness (LLM-as-judge calibrated against clinician review on 10% sample), citation accuracy, faithfulness | Answer-correct ≥ 95%; citation-accurate ≥ 99% |
| Drug-interaction subset | 60 cases | Quarterly + on FDA SPL feed schema changes | Interaction-question answer correctness, FDA-source citation accuracy | Answer-correct ≥ 98%; citation-accurate ≥ 99% |
| Conversational subset | 50 cases (multi-turn) | Quarterly | Second-turn-and-later retrieval recall, conversational coherence | Recall ≥ 85% on turn 2+; coherence ≥ 90% |
| Side-effect / HITL subset | 40 cases | Quarterly + on side-effect-class changes | Tool-call accuracy, authorization-enforcement, draft quality (clinician rating) | Tool accuracy ≥ 99%; authorization 100% (any failure is a Sev-1) |
| Refusal / escalation subset | 30 cases | Quarterly | When the Care Coordinator should refuse / escalate, does it? | Refuses when it should ≥ 95%; over-refuses ≤ 5% |
| Adversarial / prompt-injection subset | 25 cases | Quarterly + cross-link to security sibling | Injection-resistance, scope-enforcement | Injection-resistant ≥ 95% (cross-linked to security repo's red-team tests) |

The clinical golden set is built from de-identified, clinician-approved cases drawn from real interaction logs and from manually-curated edge cases. Every fixed clinical-quality bug becomes a permanent golden-set case to prevent silent regression. The judge model (Opus-class, separate from the production stack) is calibrated against clinician review quarterly; calibration measures inter-rater agreement between judge and clinician.

The eval suites run in three contexts: (1) per-PR fast suite (~30 representative cases, <8 minutes) gating merge; (2) nightly full suite (all cases) producing trend data; (3) release-candidate suite (full suite + adversarial + retrieval recall on a fresh production sample) gating deployment to staging and production.

Cross-link: engineering sibling [eval-engineering/eval-engineering-playbook.md](../../ai-engineering-reference-architecture/eval-engineering/eval-engineering-playbook.md) for the eval practice underneath this suite.

---

## 7. Cost model

The Care Coordinator's cost line is engineering's primary FinOps responsibility. The model below is calibrated to the production workload mix observed in the pilot phase and projected to GA volume.

### 7.1 Per-interaction cost breakdown (chat path, single-turn average)

| Component | Tokens / Calls | Unit cost | Per-interaction cost |
|---|---|---|---|
| Supervisor (Opus-class) | 3.2K input / 0.4K output | (frontier-tier) | ~$0.058 |
| Classifier (Haiku-class) | 0.4K in / 0.1K out | (cheap-tier) | ~$0.0008 |
| Query rewriter (Haiku-class, ~40% of interactions) | 0.6K in / 0.1K out | (cheap-tier) | ~$0.0006 amortized |
| Clinical-Knowledge Worker (Opus-class) | 5.5K in / 0.7K out | (frontier-tier) | ~$0.099 |
| Drafting Worker (Sonnet-class, ~25% of interactions) | 2.8K in / 0.4K out | (mid-tier) | ~$0.009 amortized |
| Hybrid retrieval (Postgres FTS + pgvector) | 1 query | (compute) | ~$0.0005 |
| Cohere Rerank-3.5 | 50 candidates | (vendor) | ~$0.002 |
| Embedding for query | 1 query | (vendor) | ~$0.0001 |
| Observability / trace export | 1 trace | (compute / Datadog) | ~$0.0015 |
| AI gateway + supporting infrastructure | amortized | (compute) | ~$0.003 |
| **Total per chat interaction (typical)** | | | **~$0.175** |

The single largest cost line is the Clinical-Knowledge Worker (Opus-class with a fairly large input due to retrieved chunks). The single largest *avoidable* cost is the supervisor's input tokens, which include the running summary of the conversation and the platform's base system prompt — prompt-prefix caching reclaims approximately 30% of supervisor input cost on second-turn-and-later interactions in the same session.

### 7.2 Levers

| Lever | Direction | Expected impact |
|---|---|---|
| Adopt prompt-prefix caching across all model calls | Cost down | ~15% per-interaction reduction (most of the gain on multi-turn sessions) |
| Route simple-class questions to Sonnet for the Knowledge Worker (with confidence-based escalation to Opus) | Cost down | ~10–15% per-interaction (with quality risk if calibration is wrong; gated by eval) |
| Move the Drafting Worker to Haiku for the simplest message types | Cost down | ~1% (small lever; mentioned for completeness) |
| Reduce retrieval top-K from 50 to 30 before rerank | Cost down marginally, latency down | ~3% cost, ~50ms latency (gated by eval — has not regressed quality in tests) |
| Switch reranker to Cohere Rerank-3.0 (older, cheaper) | Cost down | ~$0.0005/interaction; rejected on slight quality regression |
| Add response cache for FAQ-shaped questions | Cost down | ~5% on the ~15% of traffic that maps to recurring questions |
| Expand to a 4th tier (use a smaller open-weight model for classification only) | Cost down marginally | not pursued — operational cost outweighs the savings at current volume |

### 7.3 Circuit breakers

| Trigger | Action |
|---|---|
| Single interaction > $0.50 | Hard-stop; return graceful failure; page on-call |
| Single session > $1.50 | Session terminated; subsequent requests from same session return graceful failure |
| Tenant exceeds $50/day | Tenant rate-limited to read-only operations; auto-page on-call + customer success |
| Aggregate cost > 130% of 7-day trailing average for the same hour-of-week | Soft alert; investigate within the hour |

---

## 8. Findings (sprint-assignable)

These are the findings I would write into a Care Coordinator architecture review document. Each has an ID, severity, finding, recommendation, and a sprint owner template. Findings prefixed `ARCH-CARE-` are architecture-domain; cross-links indicate where engineering or security siblings own the implementation.

### ARCH-CARE-001 — Severity: Critical (GA-blocker)
**Finding.** Tenant filter in the retrieval client is enforced by application code rather than as a wrapper-mandatory constraint; a code path can bypass it.
**Recommendation.** Move tenant filter enforcement into the retrieval-client wrapper as a non-optional parameter; add the test that verifies every code path goes through the wrapper.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-CARE-002 — Severity: Critical (GA-blocker)
**Finding.** Side-effect tools (`draft_patient_message`, `draft_order`, `draft_appointment`) are exposed without a uniform HITL contract; the implementation enforces HITL by convention rather than by registry-level requirement.
**Recommendation.** Define side-effect tools as a tool-class with mandatory HITL contract in the tool registry; reject any tool registration in that class that does not declare an HITL approval path.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-CARE-003 — Severity: High
**Finding.** Per-interaction cost-budget circuit breaker is configured but not connected to an automated termination path; in a runaway-cost scenario the system would alert but continue spending.
**Recommendation.** Wire the circuit breaker to a hard-stop in the supervisor wrapper; return graceful-failure response; page on-call.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-CARE-004 — Severity: High
**Finding.** Model versions in deployment manifest reference `claude-opus-latest`-style aliases for the supervisor; aliases drift silently.
**Recommendation.** Pin to full model version strings; integrate with the model registry; fail deployment on unpinned references.
**Owner.** ai-platform-eng, sprint N+1.
**Cross-link.** Engineering: `model-lifecycle/model-version-pinning.md` (coming).

### ARCH-CARE-005 — Severity: High
**Finding.** Prompt versions are stored in code rather than a versioned prompt store; rollback requires a code rollback.
**Recommendation.** Migrate prompts to a versioned prompt store; pin prompt versions in the release artifact; enable prompt-only rollback.
**Owner.** ai-platform-eng, sprint N+2.
**Cross-link.** Engineering: `prompt-engineering/prompts-as-code-discipline.md`, `prompt-engineering/prompt-versioning.md` (coming).

### ARCH-CARE-006 — Severity: High
**Finding.** Drug-interaction graph refresh process is manual; a missed monthly refresh would silently serve stale interaction data.
**Recommendation.** Automate the FDA SPL ingest with an alert on missed schedule; eval cross-check on every refresh; rollback path on regression.
**Owner.** ai-platform-eng + clinical-informatics, sprint N+2.

### ARCH-CARE-007 — Severity: High
**Finding.** No structured separation between conversational long-term memory across staff identities; one staff member's prior session events can surface in another's retrieval if both work the same patient.
**Recommendation.** Scope long-term episodic memory by staff-identity, not just by tenant.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-CARE-008 — Severity: High
**Finding.** Audit log captures the LLM call but not the tool-call authorization decision; compliance reviewers cannot trace why a side effect was permitted.
**Recommendation.** Add authorization-decision events to the audit log; surface them in the trace; verify HIPAA §164.312(b) coverage.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-CARE-009 — Severity: Medium
**Finding.** Retrieval client logs retrieved doc IDs but not retrieval scores; retrieval-quality investigations have to re-run the retrieval to get scores.
**Recommendation.** Include retrieval scores (BM25 score, vector score, reranker score) in the retrieval span attributes.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-CARE-010 — Severity: Medium
**Finding.** Per-PR fast eval suite has 30 cases; coverage on side-effect / HITL paths is thin (3 cases).
**Recommendation.** Add 8–12 representative side-effect cases to the fast suite; verify wall-clock impact remains under the 8-minute target.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-CARE-011 — Severity: Medium
**Finding.** Cost dashboards show per-tenant cost but not per-feature-within-tenant cost; diagnosing which Care Coordinator path is driving a tenant's cost spike requires manual trace analysis.
**Recommendation.** Add per-feature dimension (chat / inline-order-entry / async-coordination) to cost attribution; surface in dashboards.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-CARE-012 — Severity: Medium
**Finding.** Embedding model is pinned in code but the rebuild-on-version-change procedure is documented in a wiki rather than codified.
**Recommendation.** Codify the rebuild procedure as a runnable pipeline; integrate with the model-lifecycle promotion gates.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-CARE-013 — Severity: Medium
**Finding.** Hospital-tenant protocol uploads do not go through an eval cross-check; a poorly-formatted protocol upload can degrade retrieval quality for that tenant silently.
**Recommendation.** Add a per-upload validation step (format check, chunk-size check, retrieval-recall smoke test on a tenant-specific 5-case sample); reject uploads that fail.
**Owner.** ai-platform-eng + clinical-informatics, sprint N+4.

### ARCH-CARE-014 — Severity: Medium
**Finding.** Long-context model fallback is not configured; if the Opus-class endpoint is unavailable, the system fails rather than degrading.
**Recommendation.** Implement model fallback (Opus → Sonnet with annotated reduced-confidence in the response) for the Clinical-Knowledge Worker; eval the fallback path against the clinical golden set.
**Owner.** ai-platform-eng, sprint N+3.
**Cross-link.** Engineering: `reliability-engineering/fallback-patterns.md` (coming).

### ARCH-CARE-015 — Severity: Medium
**Finding.** Adversarial / prompt-injection eval subset is 25 cases; coverage on retrieval-injection (poisoned source documents) is 4 cases.
**Recommendation.** Coordinate with the security sibling repo's red-team practice to expand the retrieval-injection coverage to 15+ cases.
**Owner.** ai-platform-eng + ai-security, sprint N+3.
**Cross-link.** Security: `llm-application-security/` and `agent-security/` in the sibling repo.

### ARCH-CARE-016 — Severity: Medium
**Finding.** Per-tenant cost-spike alert threshold (130% of trailing 7-day average) has not been calibrated against actual workload variance; risk of either alert-fatigue or missed spikes.
**Recommendation.** Calibrate per-tenant; consider per-hour-of-week baselining to handle hospital-specific traffic patterns.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-CARE-017 — Severity: Low
**Finding.** Drafting Worker (Sonnet) input includes the full retrieved-chunk set even when the drafting task does not need it; ~30% of Sonnet input tokens are unused.
**Recommendation.** Add a context-trim step before dispatching to the drafting worker; eval the impact on draft quality.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-CARE-018 — Severity: Low
**Finding.** Trace export includes raw chunk text by default; storage cost is meaningful at 100% sampling.
**Recommendation.** Switch trace export to doc-ID-only (with chunk text retrievable on demand from the source); ~60% reduction in trace storage volume.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-CARE-019 — Severity: Low
**Finding.** No automated check that the per-worker system prompt versions deployed to production match the versions referenced in the release artifact.
**Recommendation.** Add a post-deploy verification step.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-CARE-020 — Severity: Low
**Finding.** Dedicated-infrastructure variant for the premium-tier customer has not been load-tested at full projected volume.
**Recommendation.** Run a synthetic load test against the dedicated stack at 3x current premium-tier traffic; verify isolation and observability behave as designed.
**Owner.** ai-platform-eng, sprint N+5.

---

## 9. Sprint sequencing from pilot to GA

The Care Coordinator's path from pilot (3 hospitals) to GA (240 hospitals) is sequenced as a 12-sprint plan, with the first six sprints focused on closing the GA-blockers identified in the architecture review and the next six on the operational depth needed for the scale-up.

| Sprint | Focus | Deliverables (one-line) |
|---|---|---|
| N+1 | GA-blocker remediation | ARCH-CARE-001 (tenant filter), ARCH-CARE-002 (HITL contract), ARCH-CARE-003 (circuit breaker wiring), ARCH-CARE-004 (model pinning) |
| N+2 | Prompt & dataset versioning | ARCH-CARE-005 (prompt store), ARCH-CARE-006 (graph automation), ARCH-CARE-007 (memory scoping), ARCH-CARE-008 (audit-log enrichment), ARCH-CARE-010 (eval coverage) |
| N+3 | Observability & fallback | ARCH-CARE-009 (retrieval scores in spans), ARCH-CARE-011 (per-feature cost), ARCH-CARE-012 (rebuild pipeline), ARCH-CARE-014 (model fallback), ARCH-CARE-015 (adversarial coverage) |
| N+4 | Tenant onboarding scale-out | ARCH-CARE-013 (protocol upload validation), ARCH-CARE-016 (cost-spike calibration), ARCH-CARE-017 (drafting-worker context trim), ARCH-CARE-019 (post-deploy verification) |
| N+5 | Operational depth | ARCH-CARE-018 (trace storage), ARCH-CARE-020 (dedicated-stack load test), regional readiness, IR runbook integration |
| N+6 | Customer onboarding waves begin | Onboard customer waves of 20–30 hospitals per sprint; monitor cost-per-tenant baseline; refine playbook |
| N+7 to N+12 | Scale to GA | Continued onboarding, runbook refinement, eval-suite growth from production cases, model-deprecation drills |

The GA gate is: all critical and high findings closed, eval suites green at threshold for four consecutive weeks, cost-per-interaction within target range, no Sev-1 incidents in the trailing 60 days, customer-facing SLOs met on the pilot hospitals' production traffic.

---

## 10. Cross-references

- **Pattern selection rationale.** [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) for the retrieval-pattern decision; agent-topology and model-strategy documents (coming) for the rest.
- **Engineering practice.** Sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture): the eval, agent-engineering, observability, model-lifecycle, and reliability disciplines that the Care Coordinator runs on day-to-day.
- **Security and threat model.** Sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture): the Care Coordinator's threat model lives in `agent-security/`; the prompt-injection and retrieval-injection controls live in `llm-application-security/`.
- **Cloud platform.** Sibling [cloud-security-reference-architecture](https://github.com/jeremiahredden/cloud-security-reference-architecture): the multi-account topology, the workload-identity pattern (IRSA / GitHub Actions OIDC), the IR runbooks (`runbook-eks-pod-compromise.md` is the closest analogue for the inference layer), and the landing-zone baselines.
- **AppSec.** Sibling [appsec-reference-architecture](https://github.com/jeremiahredden/appsec-reference-architecture): the existing AppSec posture against the patient-API and the clinical-platform web app on top of which the Care Coordinator is integrated.

---

## 11. Open questions and future work

- **Multi-region serving.** Current architecture is single-region per AWS account; cross-region active/active for HA is on the roadmap and depends on the embedding pipeline's regional consistency story (currently weak).
- **Fine-tuning for clinical reasoning.** Evaluated and not pursued — the frontier base models are sufficient for the workload and the maintenance cost of a fine-tune is not justified at current scale. Revisit if model deprecation forces a base-model migration or if cost pressure rises.
- **MCP for tool integration.** Currently in-process tool calls; MCP servers being evaluated for the patient-message and order-draft tools to enable independent deployment. Decision deferred pending the broader MCP-adoption decision.
- **Caching beyond prompt-prefix.** Semantic cache for FAQ-class clinical questions is on the roadmap; estimated 5% cost reduction.
- **Federated tenants.** A handful of hospital customers operate as health-system networks with multiple sub-tenants; the current tenant model is flat. A two-level tenant model is on the roadmap for those customers.
