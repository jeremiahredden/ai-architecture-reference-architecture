# Reference System: Analytics Warehouse Copilot

> **Status.** Reference architecture, current as of 2026-Q2 review. Built from real systems and anonymized; the client (Meridian Health) is a fictional regulated healthcare SaaS used across the sibling reference architecture repos. **Audience.** Architects designing a natural-language-to-SQL system over an enterprise data warehouse. Tech leads weighing the constrained-generation pattern vs an agent loop for SQL generation. Compliance teams whose data-warehouse access is governed by re-identification risk and the AI Assist must respect it. **Cross-links.** The sibling [Care Coordinator reference](./meridian-care-coordinator.md) and [Patient API AI Assist reference](./patient-api-ai-assist.md) for comparison; pattern selection rationale in [reference-patterns/structured-output-patterns.md](../reference-patterns/structured-output-patterns.md); engineering practice in the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture).

---

## 1. Executive summary

The Analytics Warehouse Copilot is a natural-language-to-SQL assistant for Meridian Health's de-identified analytics warehouse. It helps internal Meridian analysts and analysts at hospital customer organisations write SQL queries against the de-identified analytics dataset — answering questions like "what was the average 30-day readmission rate for CHF patients by service line in Q1?" with both the SQL query and the result set, while strictly enforcing the data-access controls that prevent re-identification.

Meridian's analytics warehouse contains de-identified data per HIPAA Safe Harbor; it's the basis for internal analytics, customer-facing analytics dashboards, and ad-hoc analyst queries. The warehouse is large (~50TB), structured across ~120 fact and dimension tables, with a semantic layer (dbt-modelled) on top. SQL is the lingua franca; users are technical analysts who can read SQL but appreciate the speed of natural-language generation.

The architectural commitment differs from the [Care Coordinator](./meridian-care-coordinator.md) (clinical-agent, multi-step, real-time) and the [Patient API AI Assist](./patient-api-ai-assist.md) (patient-facing, high-volume, single-call):

- **Hybrid agent step inside a workflow** (per the engineering sibling's [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md)): outer workflow handles intake, schema retrieval, execution, and result presentation; inner agent step handles SQL generation with potential repair loops.
- **Constrained code generation** with strict schema enforcement; the SQL must reference only the analyst's authorised tables/columns.
- **Execution sandbox** with row-count and runtime limits; the generated SQL runs in a separate context that can't affect production systems.
- **HITL for re-identification-risk queries**: queries that combine fields in a way that could potentially re-identify a patient trigger human review before execution.
- **Different cost profile** (~$0.14 per query, but lower volume); different latency profile (10-30 seconds acceptable for analysts who would otherwise spend 5-15 minutes).
- **Different multi-tenancy**: each analyst's authorised scope (their tenant's data; their team's data subsets) enforced at SQL generation and execution layers.

The system is sized to handle approximately 1,200 query attempts/day at GA (across internal Meridian analysts + hospital customer analysts who opt into the copilot capability). The volume is meaningfully lower than the AI Assist; the per-query value is meaningfully higher (saves analyst time; enables non-SQL-fluent analysts to explore).

This document is structured as nine sections: (1) executive summary, (2) workload and non-functional requirements, (3) architectural decision records, (4) reference architecture diagram, (5) data-flow walkthroughs, (6) eval surface, (7) cost model, (8) findings (`ARCH-AWC-001` through `ARCH-AWC-018`), and (9) sprint sequencing.

---

## 2. Workload and non-functional requirements

### 2.1 What the Copilot does

The Analytics Warehouse Copilot is invoked from four entry points:

1. **In-warehouse query editor.** Analysts working in the warehouse's SQL editor have a "ask the copilot" button. They describe what they want in natural language; the copilot generates SQL, shows it, and executes on confirmation. Bulk of traffic (~70%).
2. **Dashboard authoring.** Analysts building dashboards use the copilot to generate metric queries. The output is added to the dashboard's underlying SQL; the analyst can edit before saving.
3. **Ad-hoc exploration.** Analysts ask exploratory questions to scope a deeper analysis. The copilot's answer informs whether the question is worth a full analysis.
4. **Schema understanding.** Analysts ask "where would I find data about X?" — the copilot uses the schema-introspection tool to answer.

### 2.2 The five questions

| Question | Answer |
| --- | --- |
| What is the workload's primary shape? | Constrained code generation with optional repair loop. |
| What is the latency tolerance? | 10-30 seconds acceptable for analyst workflows. |
| What is the cost tolerance? | ~$0.14 per query at projected 1,200/day. |
| What are the regulatory constraints? | HIPAA Safe Harbor for de-identification + scope enforcement to authorised data subsets. |
| Who is on call when it breaks? | Analytics platform team + AI-platform on-call. |

### 2.3 The non-functional requirements

- **Latency:** p50 < 12s; p99 < 30s; acceptable because the alternative is "write SQL by hand."
- **Throughput:** 1,200 queries/day average; peaks during quarter-end.
- **Availability:** 99% (analysts can fall back to manual SQL).
- **Multi-tenancy:** every query scoped to the analyst's authorised data.
- **Cost:** ~$0.14/query average; ~$0.52 worst case.
- **Quality:** SQL correctness > 90%; SQL runs without error > 95%; SQL produces the intended answer > 85%.
- **Compliance:** HIPAA Safe Harbor maintained; re-identification risk queries reviewed.

### 2.4 What's explicitly out of scope

- **Direct query to identified data.** The warehouse is de-identified; the copilot doesn't access the underlying identified-data systems.
- **Writes to the warehouse.** SELECT only; no INSERT / UPDATE / DELETE.
- **Cross-tenant data exposure.** Internal Meridian analysts have broader scope; customer analysts see only their hospital's data.
- **Decision support.** This is a query-generation tool; the analyst interprets results.

### 2.5 The re-identification risk

The warehouse is de-identified under HIPAA Safe Harbor. Some query patterns increase re-identification risk:

- Small-cell counts (a query returning a single patient is potentially identifying).
- Combinations of demographic fields (rare combinations are identifying).
- Joins across temporal sequences with rare events.

The architecture must detect and gate these query patterns; the HITL is the safety net.

---

## 3. Architectural decision records

### 3.1 ADR-1: Hybrid (workflow + agent step) architecture

**Decision.** Outer workflow (intake → schema retrieval → SQL generation → validation → execution → presentation); SQL generation is an inner agent step that can iterate on the SQL.

**Why.**
- Per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md): the workflow shape captures the outer flow's deterministic structure; the agent step inside captures the genuinely-flexible SQL generation.
- The outer workflow's failure modes are bounded (each step is a clear failure surface); the inner agent loops only on SQL generation (bounded retries).

**Considered and rejected.**
- Pure single-call: SQL generation often needs revision based on schema verification; single-call would fail too often.
- Pure agent loop: would over-engineer the deterministic intake / execution / presentation.

### 3.2 ADR-2: Constrained code generation for SQL

**Decision.** SQL generation uses constrained decoding against a schema-aware grammar; output is restricted to syntactically valid SQL referencing only authorised tables/columns.

**Why.**
- Per [structured-output-patterns.md](../reference-patterns/structured-output-patterns.md): constrained decoding prevents many classes of failure.
- The grammar can enforce: only SELECT statements; only authorised tables; only authorised columns; LIMIT clause required (prevents accidental full-table scans).
- Constrained decoding is more reliable than post-hoc validation.

**Considered and rejected.**
- Unconstrained generation + validation: validation catches some issues but lets others through (e.g., subtle column-name confusion); constrained is preferred.
- Templated query generation: too rigid for natural-language flexibility.

### 3.3 ADR-3: Schema-introspection as primary retrieval

**Decision.** Schema, table descriptions, column descriptions, and example queries are the retrieval surface. The semantic layer (dbt models) provides higher-level abstractions.

**Why.**
- Schema is the data the model must reason about; retrieval surfaces it directly.
- Per [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md): schema is a versioned data contract.
- Example queries from query history (filtered for quality) provide pattern-grounding.

### 3.4 ADR-4: Execution sandbox with strict limits

**Decision.** Generated SQL executes in a sandbox role with strict row-count, runtime, and resource limits.

**Why.**
- Bounded resource consumption (can't accidentally crash the warehouse with a billion-row query).
- Bounded blast radius (can't access tables outside the analyst's scope; sandbox role's permissions are the floor).
- Failure mode is clean (sandbox limit hit → query terminates → user notified).

### 3.5 ADR-5: HITL for re-identification-risk queries

**Decision.** Per [refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md), queries flagged as re-identification risk route to a compliance reviewer before execution.

**Why.**
- The warehouse's de-identification is the legal posture; queries that compromise it require oversight.
- Automated detection (cell-count rules, demographic-combo rules) is imperfect; HITL catches what automation misses.
- HITL adds latency but only for the flagged subset (~3-5% of queries).

### 3.6 ADR-6: Tier-routed model strategy

**Decision.** Smaller model (Haiku-class) for the intake/classification; larger model (Sonnet or Opus) for SQL generation; Haiku for result interpretation.

**Why.**
- SQL generation is the hard step; warrants the larger model.
- Intake and result interpretation are easier; smaller model suffices.
- Per [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md).

### 3.7 ADR-7: Multi-tenant scope at SQL generation AND execution

**Decision.** Per-analyst scope is enforced at SQL generation (the model is restricted to authorised tables/columns) AND at execution (the sandbox role only has access to authorised data).

**Why.**
- Defense in depth: generation-layer scope prevents the model from suggesting cross-scope queries; execution-layer scope prevents anything the model gets wrong.
- Per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) and [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/).

### 3.8 ADR-8: Cost cap per query and per session

**Decision.** Per-query budget ($0.50); per-session budget ($5); per-analyst daily budget ($50).

**Why.**
- A single misbehaving query (the agent step loops) shouldn't consume the day's budget.
- Per [agent-engineering/agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md).

### 3.9 ADR-9: SQL transparency to the analyst

**Decision.** The generated SQL is always shown to the analyst before execution (unless the analyst opts into auto-execute for low-risk queries). The analyst can edit before running.

**Why.**
- Analysts are technical; they want to see the SQL.
- Reviewing the SQL is a quality check.
- Transparency builds trust.

### 3.10 ADR-10: Result interpretation as separate step

**Decision.** After execution, a separate (Haiku-class) call interprets the result set in narrative form. The analyst sees both the table and the narrative.

**Why.**
- Narrative interpretation helps non-SQL-fluent stakeholders who consume the analyst's output.
- Separating from SQL generation keeps each step focused.

---

## 4. Reference architecture diagram

```
                          Analyst Query Editor / Dashboard Builder
                                          │
                                          ▼
                                 Copilot Service
                                          │
                       ┌──────────────────┼──────────────────┐
                       ▼                  ▼                  ▼
              Intake (Haiku)    Schema Retrieval     Sandbox Service
              (classify, route)  (semantic layer +   (executes SQL
                                  schema + examples)  with limits)
                                          │                  ▲
                                          ▼                  │
                                    AI Gateway               │
                                  (meridian-llm-             │
                                    gateway)                 │
                                          │                  │
                                          ▼                  │
                                  SQL Generation             │
                                  Agent Step                 │
                                  (Sonnet/Opus, with         │
                                   schema-aware              │
                                   constrained decoding)     │
                                          │                  │
                                          ▼                  │
                                  SQL Validator              │
                                  (syntax, scope,            │
                                   re-id check)              │
                                          │                  │
                       ┌──────────────────┼──────────────────┘
                       ▼ (re-id flagged)  ▼
                  HITL Queue          Execute SQL
                  (compliance         in Sandbox
                   reviewer)               │
                       │                   │
                       └───────────────────┘
                                          ▼
                                  Result + Narrative
                                  Interpretation (Haiku)
                                          │
                                          ▼
                                  Display to Analyst
```

Key boundaries:

- **Copilot Service** orchestrates the workflow.
- **Schema Retrieval** is read-only; per analyst's authorised scope.
- **SQL Generation Agent Step** is the inner agent loop; bounded.
- **SQL Validator** enforces scope, syntax, re-identification rules.
- **Sandbox Service** runs the SQL in an isolated role; resource limits.
- **HITL Queue** for flagged queries.

---

## 5. Data-flow walkthroughs

### 5.1 Simple query: "What was Q1 readmission rate by service line?"

1. Analyst types: "what was Q1 readmission rate by service line?"
2. Intake call (Haiku): classifies as "metric_query, complexity=low."
3. Schema Retrieval: returns relevant tables (encounters, service_lines, readmissions), columns, and 3 similar past queries.
4. AI Gateway: passes; analyst authorized.
5. SQL Generation: Sonnet generates SQL using the schema; constrained decoding ensures only authorised tables/columns; outputs:
   ```sql
   SELECT s.service_line_name, 
          COUNT(DISTINCT CASE WHEN r.is_readmit_30d THEN r.patient_id END) * 1.0 / 
          COUNT(DISTINCT r.patient_id) AS readmission_rate
   FROM analytics.readmissions r
   JOIN analytics.encounters e ON r.encounter_id = e.encounter_id
   JOIN analytics.service_lines s ON e.service_line_id = s.id
   WHERE e.discharge_date >= '2026-01-01' AND e.discharge_date < '2026-04-01'
   GROUP BY s.service_line_name
   ORDER BY readmission_rate DESC
   LIMIT 100;
   ```
6. SQL Validator: passes (syntax OK; scope OK; no re-id risk; LIMIT present).
7. SQL shown to analyst with a "run" button.
8. Analyst clicks "run."
9. Sandbox executes; result returned (8 service lines, rates 0.04 to 0.18).
10. Narrative Interpretation (Haiku): "In Q1 2026, the highest 30-day readmission rate was in Cardiology (18%), followed by Pulmonology (15%). Lowest were Orthopedics and Dermatology (both 4%)."
11. Display: SQL + result table + narrative. Total time ~14 seconds; total cost ~$0.11.

### 5.2 Complex query with repair loop: "Compare CHF outcomes across payers"

1. Analyst types complex multi-dimension question.
2. Schema Retrieval surfaces relevant tables.
3. SQL Generation: first attempt has a column-name error (the schema description was ambiguous).
4. SQL Validator: rejects with "column 'payer_class' not found in payers table; did you mean 'payer_type'?"
5. SQL Generation agent step: re-attempts with corrected column.
6. SQL Validator: passes.
7. Continues as in 5.1. Total time ~22 seconds (repair loop adds ~8s); total cost ~$0.18.

### 5.3 Re-identification-risk query: HITL

1. Analyst types: "show me individual patient outcomes for the 3 patients who had rare procedure X in Q1."
2. SQL Generation produces SQL.
3. SQL Validator: small-cell count detected (only 3 patients matching); re-id risk flagged.
4. Workflow routes to compliance HITL queue rather than executing.
5. User notified: "this query may have re-identification implications; it's been forwarded to compliance for review. Estimated response: 4 hours."
6. Compliance reviewer evaluates: in this case, denies (cell count too small; redirect analyst to aggregate-only alternative).
7. Analyst receives: "query denied; here are alternative aggregate queries that wouldn't have re-id implications."

---

## 6. Eval surface

### 6.1 Eval set composition (target at GA)

- **Analyst eval set:** ~500 (question, expected SQL / expected result) pairs.
  - Simple metric queries: ~150.
  - Multi-table joins: ~150.
  - Edge cases (complex aggregations, time-series, comparison): ~100.
  - Re-identification cases: ~50.
  - Out-of-scope cases (refusal expected): ~50.

### 6.2 Eval metrics

- SQL correctness (exact match or semantically equivalent): target 90%+.
- SQL runs without error: target 95%+.
- Result correctness (does the SQL produce the right answer?): target 85%+.
- Re-id flagging accuracy: precision and recall.
- Latency (p99 < 30s).
- Cost per query.

### 6.3 Eval cadence

- Per-commit: Tier 1 (~150 cases) on every promotion.
- Daily: continuous on production samples (anonymised).
- Quarterly: full eval set.

### 6.4 Eval gate

- SQL correctness within 2% of baseline.
- No regression on re-id flagging.
- Cost within 15% of baseline.
- Latency P99 within 10% of baseline.

---

## 7. Cost model

### 7.1 Per-query cost breakdown

| Component | Cost |
| --- | --- |
| Intake (Haiku) | $0.002 |
| Schema retrieval (embedding + lookup) | $0.001 |
| SQL Generation (Sonnet, avg with 1.2 attempts) | $0.10 |
| Validator (deterministic; no LLM cost) | $0 |
| Result interpretation (Haiku) | $0.005 |
| HITL (3% of queries, ~$3 each via reviewer time) | $0.03 (amortised across all queries) |
| **Weighted average** | **~$0.14** |

### 7.2 Daily/monthly cost projection

At 1,200 queries/day:

- Daily: $168/day in LLM cost.
- Monthly: ~$5,000/month in LLM cost.
- Plus ~$2,500/month for the compliance reviewer's time on HITL.

### 7.3 Per-analyst budget

Daily $50/analyst; few analysts approach the cap.

---

## 8. Findings (sprint-assignable)

### ARCH-AWC-001 — Severity: Critical
**Finding.** Re-identification risk detection must be reliable; false negatives are HIPAA exposures.
**Recommendation.** Multi-layer detection (cell-count rule + demographic-combo rule + LLM judge); HITL for any flagged query; eval set with extensive re-id cases.
**Owner.** ai-platform-eng + compliance, sprint N+1.

### ARCH-AWC-002 — Severity: Critical
**Finding.** Sandbox role's permissions are the security boundary; must be tightly scoped per analyst's authorisation.
**Recommendation.** Per [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) and [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-AWC-003 — Severity: Critical
**Finding.** Generated SQL must be validated before execution; never trust the model directly.
**Recommendation.** SQL Validator (syntax + scope + re-id) before sandbox execution.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-AWC-004 — Severity: High
**Finding.** Per-query and per-session cost budgets must be enforced; runaway repair loops can blow the budget.
**Recommendation.** Per [agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md); budget hierarchy.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-AWC-005 — Severity: High
**Finding.** Schema retrieval quality is critical; bad schema descriptions lead to wrong SQL.
**Recommendation.** Schema documentation as data-contract; updated alongside warehouse changes.
**Owner.** ai-platform-eng + data-warehouse team, sprint N+2.

### ARCH-AWC-006 — Severity: High
**Finding.** Sandbox execution limits must be tunable per query type; complex but legitimate queries may need higher limits than default.
**Recommendation.** Per-query-type limit profiles; analyst can request elevated limits with approval.
**Owner.** ai-platform-eng + data-warehouse team, sprint N+2.

### ARCH-AWC-007 — Severity: High
**Finding.** Cross-tenant query attempts (analyst at hospital A querying hospital B data) must be impossible at SQL generation; the schema retrieval scope is the enforcement.
**Recommendation.** Per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md); CI test.
**Owner.** ai-platform-eng + security, sprint N+2.

### ARCH-AWC-008 — Severity: Medium
**Finding.** SQL Generation repair loop must be bounded; max 2 attempts then fail with explanation.
**Recommendation.** Per [agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md); turn budget.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-AWC-009 — Severity: Medium
**Finding.** Example queries used for few-shot must be curated; some historical queries are anti-patterns.
**Recommendation.** Per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md); curate the example pool.
**Owner.** ai-platform-eng + analytics-team, sprint N+3.

### ARCH-AWC-010 — Severity: Medium
**Finding.** Result interpretation must not mis-state the SQL's findings; eval the interpretation step.
**Recommendation.** Per-step eval for interpretation accuracy.
**Owner.** ai-platform-eng + analytics-team, sprint N+3.

### ARCH-AWC-011 — Severity: Medium
**Finding.** Per-analyst observability (queries asked, success rate, latency) needed for adoption analysis.
**Recommendation.** Per-analyst dashboards.
**Owner.** ai-platform-eng + analytics-team, sprint N+3.

### ARCH-AWC-012 — Severity: Medium
**Finding.** Compliance HITL response time must meet analyst expectations; backlog risks analyst frustration.
**Recommendation.** Per [content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md); compliance reviewer capacity sized; SLA target.
**Owner.** compliance + ai-platform-eng, sprint N+3.

### ARCH-AWC-013 — Severity: Medium
**Finding.** Conversation context within an analyst's session must be available (follow-up queries reference prior results); current MVP is stateless.
**Recommendation.** Session memory per [memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md); scope-limited.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-AWC-014 — Severity: Medium
**Finding.** Auto-execute (skipping the analyst's review) should be opt-in per low-risk query types.
**Recommendation.** Auto-execute policy per query class; per-analyst opt-in.
**Owner.** product + ai-platform-eng, sprint N+4.

### ARCH-AWC-015 — Severity: Medium
**Finding.** Query history feedback (analyst rates output) must close the loop into eval set.
**Recommendation.** Rating UI; weekly promotion of rated queries to eval set.
**Owner.** ai-platform-eng + analytics-team, sprint N+4.

### ARCH-AWC-016 — Severity: Low
**Finding.** SQL formatting in display should follow team's style standard.
**Recommendation.** SQL formatter post-generation.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-AWC-017 — Severity: Low
**Finding.** SQL Generation eval set must include negative cases (queries that should refuse generation, e.g., DML attempts, system tables).
**Recommendation.** Negative cases per section 6.1.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-AWC-018 — Severity: Low
**Finding.** Per-team dashboards (analytics team aggregate; customer aggregate) for cost and quality.
**Recommendation.** Per-team rollup dashboards.
**Owner.** ai-platform-eng + finance, sprint N+5.

---

## 9. Sprint sequencing (pilot to GA)

**Sprint 0 — Foundations.** AI gateway integration; sandbox service; schema retrieval; minimal eval set.

**Sprint 1 — Single-call SQL generation.** No agent loop yet; one-shot SQL generation; basic validator.

**Sprint 2 — Agent step with repair.** Inner agent loop with bounded retries.

**Sprint 3 — Re-id detection.** Multi-layer detection; HITL queue.

**Sprint 4 — Per-tenant scoping.** Schema retrieval scoped per analyst; sandbox role per analyst.

**Sprint 5 — Result interpretation.** Narrative output.

**Sprint 6 — Internal pilot.** Internal Meridian analysts only.

**Sprint 7 — Hardening.** Cost caps; per-analyst observability; eval gate.

**Sprint 8 — Customer pilot.** 3 hospital customers opt in.

**Sprint 9 — GA preparation.** Capacity scaling; runbook; documentation.

**Sprint 10 — GA.** All opted-in customers.

The full sequence is approximately 10 sprints. Internal pilot value at Sprint 6; customer pilot at Sprint 8.
