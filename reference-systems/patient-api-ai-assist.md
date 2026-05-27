# Reference System: Patient API AI Assist

> **Status.** Reference architecture, current as of 2026-Q2 review. Built from real systems and anonymized; the client (Meridian Health) is a fictional regulated healthcare SaaS used across the sibling reference architecture repos. **Audience.** Architects sizing a higher-volume, lower-risk AI assist on top of an existing customer-facing API. Engineering leaders weighing this against the Care Coordinator's heavier architecture. Reviewers comparing this reference against their own patient- or customer-facing AI feature. **Cross-links.** The sibling [Care Coordinator reference](./meridian-care-coordinator.md) for the higher-risk clinical-agent counterpart; pattern selection rationale in [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) and [reference-patterns/structured-output-patterns.md](../reference-patterns/structured-output-patterns.md); engineering practice in the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture).

---

## 1. Executive summary

The Patient API AI Assist is a patient-facing AI feature embedded in Meridian Health's patient-facing API (the API the hospital customers use to power their patient portals, mobile apps, and self-service kiosks). It assists patients in completing common tasks: understanding their statement of benefits, finding a provider, scheduling routine appointments, understanding common lab results in plain language, and filling out forms with appropriate assistance.

Meridian operates as an HIPAA business associate serving ~240 hospital customers; the Patient API is the customer-facing surface of the platform, and the AI Assist is consumed by ~9M end-users across customer portals and apps. Unlike the [Care Coordinator](./meridian-care-coordinator.md) — which is a clinical-staff-facing agent with deep clinical capabilities — the AI Assist is a patient-facing feature with deliberately narrow scope: no clinical decision support, no order entry, no diagnostic interpretation. It explains, navigates, and assists; it does not advise.

The architecture commits to:

- **Single LLM call with structured output** as the dominant pattern; no agent loop. Each request resolves in one model call producing a structured response.
- **RAG over patient-education content + procedural documentation**, with hybrid retrieval and per-tenant scope enforcement.
- **Sub-2-second p99 latency** for chat-shaped interactions; streaming responses for perceived-latency.
- **Tier-routed model strategy** with Haiku-class for the bulk of interactions and Sonnet-class for harder cases identified by a fast classifier.
- **Lighter guardrails than the Care Coordinator** — the patient-facing context narrows the risk surface — but with strict refusal patterns for clinical advice and a mandatory escalation path to a human (hospital-staffed) for cases beyond the assistant's scope.
- **Per-interaction cost target** of approximately $0.012 at production scale — ~15× cheaper than the Care Coordinator, appropriate to the volume.

The system is sized to handle approximately 250,000 patient interactions/day across all 240 hospitals at GA. Multi-tenancy is shared-model + isolated-data: every hospital's patients see content scoped to that hospital's customisations (their formulary, their providers, their facility-specific procedures), with hard isolation enforced at the retrieval and gateway layers.

This document is structured as nine sections: (1) executive summary, (2) workload and non-functional requirements, (3) architectural decision records, (4) reference architecture diagram, (5) data-flow walkthroughs, (6) eval surface, (7) cost model, (8) findings (`ARCH-PAA-001` through `ARCH-PAA-018`), and (9) sprint sequencing.

---

## 2. Workload and non-functional requirements

### 2.1 What the AI Assist does

The Patient API AI Assist is invoked from four entry points across customer portals:

1. **Self-service chat.** Patients ask questions in a chat interface — "what does this charge on my statement mean?", "where do I find my lab results from last week?", "is Dr. Smith in my network?", "how do I prepare for tomorrow's procedure?" — and receive answers with optional follow-up actions (link to relevant portal page, schedule appointment via form-fill, generate a question list for their clinician). The bulk of interactions (~70% of volume) come through this path.
2. **Form-fill assistance.** When patients are completing forms (registration, intake, financial assistance, advance directives), the AI Assist helps them understand fields, generate appropriate text, and verify their entries. Common pattern: "I don't know how to answer this question about my insurance" → AI explains the question and helps draft an answer.
3. **Lab/result plain-language explainer.** When patients view lab results or imaging reports in the portal, the AI Assist provides a plain-language overview alongside the clinical document. Strict scope: explains what was tested and the general meaning of in-range / out-of-range; does NOT interpret the patient's specific results clinically.
4. **Procedure preparation and follow-up.** Procedure-specific guidance: "your colonoscopy is in 3 days; here's what to do" / "your surgery is tomorrow; here are the questions to ask your provider." Templated and personalised, generated from the patient's procedure data combined with the hospital's standard procedure prep content.

### 2.2 The five questions

| Question | Answer |
| --- | --- |
| What is the workload's primary shape? | High-volume, low-risk question-answering with structured output. |
| What is the latency tolerance? | Sub-2-second p99 for chat; sub-5-second for form assistance. Streaming for perceived latency. |
| What is the cost tolerance? | $0.012 per interaction at projected 250k/day = $90k/month. |
| What are the regulatory constraints? | HIPAA (patient-facing; PHI in retrieval context); content-safety (no clinical advice). |
| Who is on call when it breaks? | The Patient API platform team + a small AI-platform on-call shared with the Care Coordinator. |

### 2.3 The non-functional requirements

- **Latency:** p50 chat response < 0.8s; p99 < 2s; time-to-first-token < 500ms.
- **Throughput:** 250k requests/day, with peaks of 25k/hour (8am and 5pm patient activity).
- **Availability:** 99.9% (the patient API itself is the dependency; AI Assist degrades gracefully when down).
- **Multi-tenancy:** every patient interaction scoped to their hospital's content.
- **Cost:** ~$0.012/interaction average; ~$0.04 worst case.
- **Quality:** 95%+ correctness on patient-eval golden set; refusal-appropriateness > 92%.
- **Compliance:** HIPAA Privacy Rule; no PHI in logs; consent-based personalisation.

### 2.4 What's explicitly out of scope

- **Clinical decision support.** The AI Assist does NOT advise on treatment decisions, medication adjustments, or diagnostic interpretations. Cases that approach this are refused with escalation to a human (the patient's clinical contact via the existing patient-portal flows).
- **Crisis support.** Mental-health crises, suicidality, immediate medical emergencies route to a refusal that surfaces the appropriate crisis line, without engaging the assistant.
- **Billing disputes.** The AI Assist explains charges; doesn't negotiate. Disputes route to the hospital's billing team.
- **Multi-turn complex coordination.** That's the Care Coordinator's job; the AI Assist handles single-turn patient questions, not multi-step orchestration.

---

## 3. Architectural decision records

The key decisions, with rationale.

### 3.1 ADR-1: Single-call architecture (not agent loop)

**Decision.** Each request resolves in one LLM call producing structured output; no agent loop.

**Why.**
- The workload is question-answering with bounded context; the next-step decision is "respond" not "what tool to call next."
- Single-call has bounded cost, bounded latency, and a simple debugging surface — all matter at this volume.
- Per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md) in the engineering sibling: the right shape is the simplest shape that handles production traffic; agent loop is over-engineering here.

**Considered and rejected.**
- Workflow with multiple steps: form-fill might warrant 2 steps (retrieve form schema → answer), but the bulk of traffic is single-step; the overhead isn't justified.
- Agent loop: would add unbounded cost and latency for negligible quality gain.

### 3.2 ADR-2: Hybrid retrieval (BM25 + vector) over patient-education + procedural content

**Decision.** OpenSearch with hybrid retrieval over a corpus of (a) patient-education content (general and per-hospital), (b) procedural documentation (hospital-specific), (c) provider directories, (d) facility information.

**Why.**
- Per [hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md): clinical and patient content benefit from both semantic and lexical matching.
- Per-tenant scope is enforced at the retrieval layer; patient sees their hospital's content.
- Procedure-specific terms, drug names, and provider names benefit from lexical; general "what does this mean" benefits from semantic.

**Considered and rejected.**
- Pure vector: would miss lexical specifics (provider names, drug names, hospital-specific terminology).
- Pure lexical: misses semantic question-answering value.

### 3.3 ADR-3: Tier-routed model strategy — Haiku-class default, Sonnet-class escalation

**Decision.** A fast classifier (Haiku-class) categorises the request; most requests handled by Haiku-class; harder cases routed to Sonnet-class.

**Why.**
- Per [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md): tier routing typically reduces cost 60-80% without quality loss on the bulk of traffic.
- The AI Assist's bulk traffic is simple ("where do I find X", "what does Y mean"); Haiku handles these well.
- ~15% of traffic (form assistance, lab explanations, complex personalisation) benefits from Sonnet; routing decision based on classifier output.

**Considered and rejected.**
- Sonnet-only: cost would be ~3× higher; quality gain on simple traffic negligible.
- Haiku-only: quality on complex traffic would be insufficient.

### 3.4 ADR-4: Structured output with constrained decoding

**Decision.** Every response is structured (JSON schema with response text, optional follow-up actions, optional citations, confidence level). Constrained decoding enforces schema.

**Why.**
- Per [structured-output-patterns.md](../reference-patterns/structured-output-patterns.md) and the engineering sibling's [structured-output-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/structured-output-engineering.md): structured output enables reliable downstream processing.
- The portal UI consumes structured fields (the response text, the follow-up action options, the disclaimers); freeform output would force the UI to parse.
- Schema enforces the disclaimer field for any health-related content; not optional.

### 3.5 ADR-5: Streaming responses

**Decision.** Responses stream from the model to the patient UI; time-to-first-token is the perceived latency.

**Why.**
- Per [integration-architecture/sync-vs-async-vs-streaming.md](../integration-architecture/sync-vs-async-vs-streaming.md): streaming is the right pattern for chat where perceived latency matters and UX can render incrementally.
- Without streaming, p99 latency feels worse despite same total time.

### 3.6 ADR-6: Per-tenant content scoping at retrieval

**Decision.** Every retrieval query carries `tenant_id` (the patient's hospital); the retrieval client refuses queries without it.

**Why.**
- Per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md): tenant isolation is enforced at the data layer, not the prompt layer.
- Cross-hospital content leakage would be a HIPAA violation and a customer-trust failure.

### 3.7 ADR-7: Strict refusal patterns for clinical advice

**Decision.** Per [refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md), specific refusal patterns for medical advice requests, with clear escalation paths.

**Why.**
- The AI Assist is patient-facing; clinical advice is out of scope and would be a safety risk.
- Refusal patterns include clear next-step (contact your clinician via the patient portal).
- Refusal templates versioned and eval'd.

### 3.8 ADR-8: Lighter guardrails than Care Coordinator

**Decision.** Input/output moderation present but lighter than Care Coordinator; no agent-loop guardrails (no agent); HITL only for escalations.

**Why.**
- Single-call architecture has fewer attack surfaces than agent loop.
- Patient-facing context means moderation is the primary guardrail; agent-step / tool-call guardrails not applicable.
- HITL volume bounded by escalation criteria; reviewer load manageable.

### 3.9 ADR-9: AI gateway shared with other Meridian AI features

**Decision.** Use the shared `meridian-llm-gateway` (per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md)); no AI Assist–specific gateway.

**Why.**
- Per [guardrail-placement-decision-framework.md](../guardrails-and-policy-architecture/guardrail-placement-decision-framework.md): the gateway is the choke point for input/output moderation, cost attribution, rate limiting.
- Reuse pays off; the AI Assist team doesn't reinvent the gateway.

### 3.10 ADR-10: Eval gate with patient-eval golden set

**Decision.** Per [eval-engineering/eval-gate-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/eval-gate-architecture.md), promotion gated on eval; patient-eval golden set ~800 cases at GA.

**Why.**
- Patient-facing quality is critical; regressions reach end users directly.
- Eval includes refusal-appropriateness (correctly refuses clinical advice; correctly doesn't over-refuse legitimate questions).

---

## 4. Reference architecture diagram

```
                                Patient Portal / App / Kiosk
                                          │
                                          ▼
                                Patient API Gateway
                                (auth, rate limit, audit)
                                          │
                                          ▼
                                AI Assist Service
                                          │
                       ┌──────────────────┼──────────────────┐
                       ▼                  ▼                  ▼
              Classifier          AI Gateway      Retrieval Client
              (Haiku, ~50ms)    (meridian-llm-    (patient-content
                                  gateway)          OpenSearch)
                       │                  │                  │
                       │     Routing      │                  │
                       └────────────────► │                  │
                                          │                  │
                                          ▼                  │
                                Model call (Haiku or         │
                                Sonnet, structured           │
                                output, streaming)           │
                                          │                  │
                                          ▼                  │
                                Response with                │
                                citations, disclaimers       │
                                          │                  │
                                          ▼                  │
                                Output moderation            │
                                (PHI leak, content)          │
                                          │                  │
                                          ▼                  │
                                Streaming response  ◄────────┘
                                back to patient
```

Key boundaries:

- **AI Assist Service** is the patient-facing entry; it composes classifier + retrieval + LLM gateway.
- **AI Gateway** is the choke point per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md); handles auth, cost attribution, input/output moderation.
- **Retrieval Client** enforces per-tenant scope per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
- **Classifier** is a fast small-model call that routes (~50ms; Haiku); decides which tier handles the main request.

---

## 5. Data-flow walkthroughs

Three representative walkthroughs.

### 5.1 "What does this charge mean?" (simple chat)

1. Patient asks via portal chat: "what is the $185 charge from 'Meridian Lab Services' on my statement?"
2. Patient portal sends request to AI Assist Service with patient_id, tenant_id, conversation_id.
3. AI Assist Service calls the classifier: returns "billing_question, complexity=low."
4. Retrieval Client queries (tenant_id, intent=billing): retrieves the hospital's billing-FAQ content and statement-format documentation.
5. AI Gateway: input moderation passes; patient + tenant authorized.
6. Model call (Haiku) with structured-output schema; streaming response starts at ~400ms.
7. Output moderation: passes; no PHI leak; no disallowed content.
8. Streaming response to patient: "The $185 charge appears to be from a lab service. Common lab charges include... If you have questions about a specific charge, contact billing at [hospital phone]."
9. Total time: ~1.2 seconds; total cost: ~$0.008.

### 5.2 "I don't understand my lab result" (lab explainer, escalation)

1. Patient views their lab result in the portal; clicks "explain in plain language" link.
2. Portal sends the result data + patient context to AI Assist Service.
3. Classifier: returns "lab_explainer, complexity=medium."
4. Retrieval Client queries (tenant_id, lab_type=glucose, intent=patient_education): retrieves general glucose explainer content.
5. AI Gateway: passes.
6. Model call (Sonnet, given complexity=medium) with strict prompt:
   - Explain what glucose testing is.
   - State whether the patient's specific value is in the normal range (compared to reference range in the data).
   - **Do NOT interpret clinical significance.**
   - Always include disclaimer: "discuss your specific results with your provider."
   - Structured output with text + disclaimer + suggested_follow_up.
7. Output moderation: passes; disclaimer is present per inline schema constraint.
8. Streaming response with disclaimer + suggested follow-up ("schedule a discussion with Dr. Smith").
9. Total time: ~2.1 seconds; total cost: ~$0.04.

### 5.3 "Should I take my insulin if I'm not feeling well?" (refusal + escalation)

1. Patient asks via chat: "I'm feeling really nauseous; should I still take my insulin?"
2. Classifier: returns "clinical_advice_request" (flagged).
3. Refusal pattern triggered: "This is a question your clinician should answer. I'm not able to help with medication decisions when you're feeling unwell.

   **For urgent concerns, call your provider's office:** [phone].

   **If this is an emergency, call 911 or go to the nearest emergency room.**

   I can help you message your care team through the portal — would you like me to start that?"
4. Total time: ~200ms (no model call beyond classifier; refusal template).
5. Logged: refusal event with category "clinical_advice"; product feedback loop receives it.

---

## 6. Eval surface

### 6.1 Eval set composition (target at GA)

- **Patient-eval golden set:** ~800 cases.
  - Simple questions (procedural, billing, navigation): ~400.
  - Form assistance cases: ~150.
  - Lab/result explanation: ~100.
  - Refusal cases (clinical advice, crisis, out-of-scope): ~100.
  - Edge cases (ambiguous, multi-intent): ~50.

### 6.2 Eval cadence

- **Per-commit:** Tier 1 critical cases (~200) run on every promotion candidate.
- **Daily continuous:** LLM-judge on a sample of production interactions.
- **Quarterly comprehensive:** All ~800 cases.

### 6.3 Eval metrics

- Outcome quality (LLM-judge score, 0-5 scale; target 4.5+).
- Refusal appropriateness (rejects clinical advice; doesn't over-refuse).
- Latency (p99 < 2s).
- Cost per request.
- Citation accuracy (citations match the retrieved sources).

### 6.4 Eval gate thresholds

- Outcome aggregate within 1.5% of baseline.
- Critical-slice scores within 3% (refusal cases are critical).
- No new regressions on previously-passing cases.
- Latency p99 within 10% of baseline.
- Cost per request within 15% of baseline.

---

## 7. Cost model

### 7.1 Per-interaction cost breakdown

| Component | Cost |
| --- | --- |
| Classifier call (Haiku) | $0.001 |
| Retrieval | $0.0005 (embedding for query) |
| Main model call — Haiku (85% of traffic) | $0.008 |
| Main model call — Sonnet (15% of traffic) | $0.032 |
| Input/output moderation | $0.002 |
| **Weighted average** | **~$0.012** |

### 7.2 Daily/monthly cost projection

At 250k interactions/day:

- Daily cost: 250,000 × $0.012 = $3,000/day.
- Monthly cost: ~$90,000/month.

Plus shared platform cost (AI gateway, observability, etc.) allocated pro-rata.

### 7.3 Cost levers

- **Tier routing tuning.** If the classifier sends more traffic to Sonnet than warranted, cost rises; quarterly tuning.
- **Prompt caching.** Stable system prompt + per-hospital context cacheable at the provider level; expected to reduce Haiku cost by ~50%.
- **Embedding cache.** Repeated queries hit the embedding cache.

### 7.4 Per-tenant cost attribution

Per [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) in the engineering sibling. Each call tagged with tenant_id (hospital); enables per-hospital chargeback and noisy-neighbor detection.

---

## 8. Findings (sprint-assignable)

### ARCH-PAA-001 — Severity: Critical
**Finding.** Per-tenant retrieval scope must be enforced at the retrieval client; prompt-only scoping has been observed to fail.
**Recommendation.** Implement per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md); CI test for cross-tenant leakage.
**Owner.** ai-platform-eng + patient-api-team, sprint N+1.

### ARCH-PAA-002 — Severity: Critical
**Finding.** Refusal patterns for clinical advice must be eval-tested; over-refusal and under-refusal both unacceptable.
**Recommendation.** Per [refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md); ~100 refusal cases in eval.
**Owner.** ai-platform-eng + clinical-content, sprint N+1.

### ARCH-PAA-003 — Severity: Critical
**Finding.** Input moderation must block direct PHI in user input (e.g., patient typing their SSN); should redact or refuse appropriately.
**Recommendation.** PHI input filter per [content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md).
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-PAA-004 — Severity: High
**Finding.** Cost cap per tenant must be in place before GA; otherwise a single misbehaving hospital integration could exhaust budget.
**Recommendation.** Per-tenant cap per [cost-and-finops/per-tenant-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops) *(planned)*.
**Owner.** ai-platform-eng + finance, sprint N+2.

### ARCH-PAA-005 — Severity: High
**Finding.** Streaming response failure mode (mid-stream errors) must not leave incomplete answers visible.
**Recommendation.** Per [integration-architecture/integration-failure-patterns.md](../integration-architecture) *(coming)*; mid-stream error → final marker → clean failure UX.
**Owner.** patient-api-team, sprint N+2.

### ARCH-PAA-006 — Severity: High
**Finding.** Lab explainer must enforce "do not interpret" rule; current prompt-only enforcement has been seen to drift.
**Recommendation.** Schema-enforced via structured output (disclaimer field required; no_interpretation_provided flag).
**Owner.** ai-platform-eng + clinical-content, sprint N+2.

### ARCH-PAA-007 — Severity: High
**Finding.** Latency P99 currently 3.2s (target 2s); too long for chat UX.
**Recommendation.** Streaming + caching + classifier optimization; per section 7.3.
**Owner.** patient-api-team + ai-platform-eng, sprint N+2.

### ARCH-PAA-008 — Severity: Medium
**Finding.** Tier-routing classifier accuracy must be measured monthly; degradation routes more traffic to Sonnet (cost) or misroutes complex cases to Haiku (quality).
**Recommendation.** Classifier eval per section 6; monthly report.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-PAA-009 — Severity: Medium
**Finding.** Per-tenant content customization (each hospital's procedure docs, FAQs) requires ongoing content ingestion; current process is manual.
**Recommendation.** Per [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md); content team owns; ingestion pipeline automated.
**Owner.** ai-platform-eng + content-team, sprint N+3.

### ARCH-PAA-010 — Severity: Medium
**Finding.** Crisis-detection (mental health crisis, immediate emergency) must route to a refusal with the appropriate crisis line for the patient's location.
**Recommendation.** Per-tenant crisis-resource directory; location-aware routing.
**Owner.** ai-platform-eng + clinical-content, sprint N+3.

### ARCH-PAA-011 — Severity: Medium
**Finding.** Multi-language support is required (patient population is diverse); current MVP is English-only.
**Recommendation.** Per-language eval set; per-language refusal templates; phased language rollout.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-PAA-012 — Severity: Medium
**Finding.** Audit log per interaction (patient, tenant, intent, response category) required for HIPAA accounting-of-disclosures.
**Recommendation.** Per [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md); audit pipeline.
**Owner.** ai-platform-eng + compliance, sprint N+3.

### ARCH-PAA-013 — Severity: Medium
**Finding.** Per-feature dashboard (per-hospital cost, latency, refusal rate, satisfaction) needed for hospital-customer success conversations.
**Recommendation.** Per [observability-and-telemetry/cost-dashboards.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/cost-dashboards.md) and equivalent quality dashboards.
**Owner.** ai-platform-eng + customer-success, sprint N+4.

### ARCH-PAA-014 — Severity: Medium
**Finding.** Conversation context (current patient session) must NOT be retained across sessions for privacy; current MVP has session memory.
**Recommendation.** Per [agent-engineering/memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md); session-scope only; explicit deletion at session end.
**Owner.** ai-platform-eng + privacy, sprint N+4.

### ARCH-PAA-015 — Severity: Medium
**Finding.** Form-fill assistance must avoid filling fields with hallucinated data; require user verification before submit.
**Recommendation.** Schema constraint: AI fills proposed values; user must confirm before submission.
**Owner.** patient-api-team + ai-platform-eng, sprint N+4.

### ARCH-PAA-016 — Severity: Low
**Finding.** Citations to source content are inconsistently formatted; UX would benefit from standard citation rendering.
**Recommendation.** Standard citation schema; UI renders consistently.
**Owner.** patient-api-team + ai-platform-eng, sprint N+5.

### ARCH-PAA-017 — Severity: Low
**Finding.** No A/B testing framework for the AI Assist; tuning is by trial.
**Recommendation.** Per [prompt-engineering/prompt-ab-testing.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-ab-testing.md); integrate with Statsig.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-PAA-018 — Severity: Low
**Finding.** Patient-feedback signal capture (thumbs-up/down on each response) not integrated into eval pipeline.
**Recommendation.** Feedback flows to refusal/satisfaction dashboard; informs eval set updates.
**Owner.** product + ai-platform-eng, sprint N+5.

---

## 9. Sprint sequencing (pilot to GA)

**Sprint 0 — Foundations.** AI gateway integration; retrieval client; classifier; minimal eval set.

**Sprint 1 — Single-call path.** Main model integration; structured output schema; streaming; basic moderation.

**Sprint 2 — Tier routing.** Classifier-driven routing; per-tier eval; cost monitoring.

**Sprint 3 — Refusal patterns.** Clinical-advice refusal; crisis detection; escalation templates.

**Sprint 4 — Per-tenant content.** Content ingestion pipeline; per-tenant scoping; first 3 pilot hospitals.

**Sprint 5 — Form assistance + lab explainer.** Specialised flows; targeted eval.

**Sprint 6 — Hardening.** Per-tenant cost caps; mid-stream failure handling; HIPAA audit log.

**Sprint 7 — Pilot expansion.** Roll to 10 hospitals; monitor; iterate.

**Sprint 8 — Multi-language.** Spanish + 3 additional priority languages.

**Sprint 9 — GA preparation.** Capacity scaling; per-hospital dashboards; runbook.

**Sprint 10 — GA.** Phased rollout across all 240 hospitals; ongoing tuning.

The full sequence is approximately 10 sprints (~5 months at typical cadence). The first pilot value is achieved at Sprint 4; the GA bar requires the full sequence.
