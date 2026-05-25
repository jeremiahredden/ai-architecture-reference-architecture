# Model Routing and Tiering

> **Audience.** Architects designing the model strategy for a multi-tier AI workload. Tech leads who have seen "all of our traffic goes to the most expensive model by default" and want a structured alternative. **Scope.** The *architectural* decision: which models in a portfolio, what tiering structure, what routing pattern at the platform level. Not the engineering implementation (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `cost-and-finops/tier-routing-for-cost.md` owns that). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Model strategy in 2026 is not "pick a model"; it is "pick a portfolio of models and define how requests route through it." The portfolio decision shapes the cost line (an Opus-everywhere strategy costs ~5x an Opus-and-Sonnet-and-Haiku strategy), the capability ceiling (the most capable model in the portfolio sets the ceiling), the operational complexity (more tiers = more eval coverage), and the regulatory posture (each model needs BAA / FedRAMP / data-residency clearance).

The architectural questions: How many tiers? Which models in each? What router architecture sits in front? Where does the router live in the architecture? When does the team graduate from one-tier to multi-tier? Most teams default to one-tier (all-on-the-best-model) because it is operationally simplest; they discover the cost implications at scale.

This document is the architectural framework for the decision. The engineering implementation lives in the sibling repo's `cost-and-finops/tier-routing-for-cost.md`; this document is upstream of that — *what* tiers, *what* routing, *where* the router sits in the architecture.

This document is opinionated about three things:

1. **A portfolio of 2-4 model tiers is the architectural default.** One tier wastes cost; five tiers wastes operational complexity. Two to four tiers (typically: a frontier reasoning tier, a mid-tier for drafting/formatting, a cheap tier for classification/routing, optionally a specialty tier) is the sweet spot.
2. **The router is a platform-level component.** Per-feature routing produces inconsistency. The router lives in the AI gateway (per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md)) or alongside it, serving every feature uniformly.
3. **Tiering is a portfolio decision, not just a per-call cost lever.** The portfolio determines what the platform can do, what it costs, and what its operational shape is. Architecturally significant; revisit deliberately.

Structure: (2) the portfolio decision; (3) the tiering structure; (4) the router architecture; (5) the where-the-router-lives decision; (6) regulatory / BAA considerations; (7) migration patterns; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The portfolio decision

### 2.1 The five questions

Five questions. Answer in writing.

1. **What is the workload's capability ceiling?** The most capable model in the portfolio sets the upper bound for what the platform can do. A workload that requires top-tier reasoning has to include a frontier model in the portfolio.
2. **What is the per-interaction cost target?** A $0.20-per-interaction budget rules out an Opus-everywhere portfolio. A $0.02-per-interaction budget rules out frontier models for most calls.
3. **What is the regulatory posture?** Some models have BAA / FedRAMP / regional coverage; others do not. The portfolio is constrained to the regulatory-compliant subset.
4. **What is the team's operational capacity?** More tiers means more eval coverage, more migration projects, more router calibration. A 5-tier portfolio with a 4-person team is over-stretched.
5. **What is the volume?** At high volume, the cost savings of tiering justify the operational overhead; at low volume, simpler is often right.

### 2.2 The size of the portfolio

| Number of tiers | When right |
|---|---|
| 1 (single-tier) | Very small workload; single capability profile; cost not a meaningful concern |
| 2 (two-tier) | Common starting portfolio. Frontier for the hard work + mid-tier for the bulk |
| 3-4 (multi-tier) | Production-scale workloads. Frontier + mid + cheap (+ optionally specialty) |
| 5+ | Usually over-engineered; consolidate |

The Meridian Care Coordinator uses 3 tiers (Opus + Sonnet + Haiku). The team considered 4 (adding a fine-tuned classification tier) and rejected on operational cost.

### 2.3 The portfolio shape

A typical 3-tier portfolio:

| Tier | Role | Example models | Per-interaction cost share |
|---|---|---|---|
| Frontier | Hard reasoning; primary user-facing answer | Claude Opus, GPT-5 | 50-70% |
| Mid-tier | Drafting, formatting, structured output | Claude Sonnet, GPT-4o | 15-25% |
| Cheap-tier | Classification, routing, simple lookups | Claude Haiku, GPT-4o-mini | 5-10% |
| (Specialty) | Embeddings, reranking, etc. | OpenAI text-embedding, Cohere Rerank | 5-15% |

The cost shares are illustrative; actual depends on traffic distribution.

### 2.4 The portfolio versioning

Each tier is pinned to a specific model version per [model-registry.md](../../ai-engineering-reference-architecture/model-lifecycle/model-registry.md). The portfolio is a structured set of pinned tiers; changes are deliberate.

---

## 3. The tiering structure

### 3.1 The capability hierarchy

The tiers form a capability hierarchy. The frontier tier handles requests the mid-tier cannot handle well; the mid-tier handles requests the cheap tier cannot handle well. Routing pushes work down the hierarchy as far as quality allows.

The hierarchy is workload-specific. For Meridian:
- Clinical reasoning: Opus only.
- General drafting / formatting: Sonnet handles most; escalates to Opus rarely.
- Classification / routing: Haiku handles most; rarely escalates.

### 3.2 The specialty tiers

Some workloads benefit from specialty tiers — fine-tuned models for specific domains, smaller models specialized for a single task (embeddings, reranking, intent classification).

Specialty tiers add to the portfolio with the trade-off: they reduce cost / improve quality on their specialty, but increase operational complexity (per-tier eval, per-tier lifecycle).

For Meridian: embedding (OpenAI text-embedding-3-large) and reranking (Cohere Rerank-3.5) are specialty tiers. A fine-tuned clinical-classification model was considered and rejected.

### 3.3 The cross-provider portfolio

A portfolio can span providers (Claude + OpenAI + Cohere) or stay within one provider. Cross-provider:

- **Pros.** Resilience to provider outages; per-task best-of-breed; pricing competition.
- **Cons.** Multiple vendor relationships; multiple BAA / regulatory checks; harder to migrate uniformly.

Single-provider:
- **Pros.** Simpler operations; one BAA; one set of API patterns.
- **Cons.** Single point of failure; locked to one provider's roadmap.

Meridian uses cross-provider for embeddings (OpenAI), reranking (Cohere), and main inference (Anthropic). The cross-provider posture was contractually justified — each provider's BAA covers their respective data.

### 3.4 The migration considerations

The portfolio evolves. Models deprecate; new models land; pricing shifts. The portfolio strategy includes migration plans:

- When a tier's primary model is deprecated: identify the replacement model; eval-validate; coordinate rollout.
- When a new model lands that could replace an existing tier: shadow-test against the existing; decide whether to swap.
- When provider pricing changes meaningfully: re-evaluate the cost-share math; rebalance tiers if needed.

---

## 4. The router architecture

The router is the architectural component that decides which tier handles each request.

### 4.1 Router architectures

- **Rule-based router.** Code-defined rules. Simple workloads.
- **Classifier-based router.** Trained classifier. More flexible.
- **LLM-as-router.** Cheap-tier LLM call. Most flexible; handles new cases.
- **Hybrid.** Rules for fast paths; LLM for the rest.

The architecture sibling discussion in [reference-patterns/agent-topologies.md](../reference-patterns/agent-topologies.md) introduces the supervisor / worker pattern; the router is the supervisor's dispatcher in that topology. For non-agent features, the router is a standalone component.

### 4.2 Router placement in the architecture

```
User request
   │
   ▼
AI gateway (per [ai-gateway-pattern.md])
   │
   ▼
[ Router decides which tier ]
   │
   ▼
LLM call (selected tier)
   │
   ▼
Response
```

The router lives inside or immediately downstream of the gateway. The gateway provides the call context the router consumes (tenant, user, feature, session); the router emits the routing decision; the gateway dispatches the call to the selected tier.

### 4.3 The router-as-platform-component decision

Per-feature routing (each feature has its own router) vs platform routing (one router for the platform):

- **Platform routing.** Consistent decisions across features; centralized eval; one router to maintain.
- **Per-feature routing.** More flexibility per feature; risk of inconsistency.

For most teams, platform routing is right. Per-feature exception when a specific feature has a distinctive workload structure.

For Meridian: platform routing via the classifier worker pattern. Different features can configure their routing rules (which classes route where) but the routing infrastructure is shared.

### 4.4 The router's failure mode

The router can fail. Routing decisions can be wrong. The architectural choice: what does the system do when routing fails or is wrong?

- **Conservative escalation.** Wrong routing toward a cheap tier escalates to the capable tier when the cheap tier produces low-confidence output.
- **Eval-validated routing.** Routing is calibrated against the eval suite; quality regressions from misrouting are caught in eval.
- **Manual override.** Some workflows allow the caller to override the router (force tier selection).

The pattern is layered: prevention via calibration; detection via eval and SLI; recovery via escalation.

---

## 5. Where the router lives

The router is part of the architecture; where it lives shapes its operational properties.

### 5.1 Inside the AI gateway

The gateway has the routing logic embedded. Every call through the gateway is routed.

- **Pros.** Single chokepoint; routing applies uniformly; cost circuit-breakers integrate naturally.
- **Cons.** Gateway becomes more complex; routing changes are gateway changes.

This is the default for most platforms. The Care Coordinator uses this pattern (the classifier worker is one of the supervisor's tool calls, executed inside the agent loop which itself runs through the gateway).

### 5.2 Alongside the AI gateway (sidecar)

The router is a separate component the gateway calls. The gateway delegates routing.

- **Pros.** Router has its own deployment cadence; routing changes ship independently.
- **Cons.** Additional component; extra hop.

Useful when routing logic is complex enough to warrant independent management.

### 5.3 In the application layer

The application code makes routing decisions before calling the gateway. The gateway is a downstream pass-through.

- **Pros.** Application can incorporate context the gateway does not see.
- **Cons.** Inconsistent across applications; loses the chokepoint discipline.

Mostly an anti-pattern; useful only when the application's routing logic is genuinely workload-specific in a way no shared router could express.

### 5.4 The placement decision matrix

| Workload | Recommended placement |
|---|---|
| Single feature, simple routing | Inside the gateway |
| Multiple features, shared classes | Inside the gateway |
| Specialized routing per feature | Sidecar or per-feature router |
| Custom routing per application | Application layer (with care) |

---

## 6. Regulatory / BAA considerations

The portfolio is constrained by regulatory coverage.

### 6.1 The constraint matrix

For regulated workloads, each model in the portfolio must clear the regulatory bar:

- HIPAA (Meridian): each model needs BAA coverage.
- FedRAMP: each model needs FedRAMP authorization.
- EU GDPR: each model needs EU data residency (if EU customers are served).

Models without coverage cannot be in the portfolio for the regulated workload.

### 6.2 The portfolio splitting

For platforms serving both regulated and non-regulated workloads, the portfolio may split:

- Regulated workload portfolio: BAA-covered models only.
- Non-regulated workload portfolio: full model set.

The gateway routes per workload to the appropriate portfolio.

### 6.3 The BAA-covered tier sweet spot

In 2026, BAA coverage for major LLM providers is increasingly available but not uniform. The portfolio decision is constrained:

- Anthropic: Claude models on Anthropic API (BAA available), Bedrock (BAA via AWS), Vertex (BAA via Google).
- OpenAI: GPT models on OpenAI API (BAA via Azure OpenAI primarily).
- Google: Gemini on Vertex (BAA via Google).
- Cohere: BAA via specific enterprise contracts.

The portfolio strategy respects the coverage; substitute models that lack coverage.

### 6.4 The premium-tier dedicated infrastructure

For premium customers on dedicated infrastructure (per [isolation-models.md](../multi-tenancy-and-isolation/isolation-models.md)), the portfolio may include dedicated-deployment versions of models (e.g., Azure OpenAI dedicated deployment) for the contracted isolation guarantees.

---

## 7. Migration patterns

The portfolio evolves; migrations between portfolios happen.

### 7.1 Tier-version migration

When a tier's primary model is deprecated, the team migrates to the replacement:

1. Identify the replacement model (within the same provider's catalog).
2. Eval the replacement against the workload (per [eval-engineering-playbook.md](../../ai-engineering-reference-architecture/eval-engineering/eval-engineering-playbook.md)).
3. Update the model registry per [model-registry.md](../../ai-engineering-reference-architecture/model-lifecycle/model-registry.md).
4. Update the release manifest per [model-version-pinning.md](../../ai-engineering-reference-architecture/cicd-and-eval-gates/model-version-pinning.md).
5. Canary, then full rollout.

Typical timeline: 2-6 weeks per tier migration, depending on eval complexity.

### 7.2 Cross-provider migration

When the team switches a tier from one provider to another:

1. Eval the new provider's model on the workload.
2. Verify regulatory coverage.
3. Plan prompt-portability work (prompts may behave differently across providers).
4. Stage the new provider in canary.
5. Roll out; monitor.

Typical timeline: 4-12 weeks; the prompt-portability work is often the long pole.

### 7.3 Portfolio expansion

Adding a new tier (e.g., adding a specialty fine-tuned model):

1. Confirm the new tier addresses a workload gap (specific class where existing tiers underperform).
2. Eval-validate.
3. Update the registry, router, and observability for the new tier.
4. Roll out.

The expansion adds operational overhead; do not expand without a clear workload need.

### 7.4 Portfolio contraction

Removing a tier (consolidation, decommissioning):

1. Identify the workload currently served by the tier.
2. Re-route to other tiers; eval-validate the new routing.
3. Deprecate the tier per the model lifecycle.
4. Retire after migration completes.

Useful when a tier is no longer earning its operational cost.

---

## 8. Worked Meridian Health example

### 8.1 The portfolio

The Care Coordinator's model portfolio:

| Tier | Model | Version | Role | BAA coverage |
|---|---|---|---|---|
| Frontier | Claude Opus 4.7 | 2026-04-12 | Supervisor + clinical-knowledge worker | Anthropic API BAA |
| Mid-tier | Claude Sonnet 4.6 | 2025-08-15 | Drafting worker + summarization | Anthropic API BAA |
| Cheap | Claude Haiku 4.5 | 2025-10-01 | Classifier + query rewriter | Anthropic API BAA |
| Specialty (embedding) | OpenAI text-embedding-3-large | 2024-01-25 | Vector embeddings | Azure OpenAI BAA |
| Specialty (rerank) | Cohere Rerank 3.5 | 2025-09-01 | Reranker | Cohere enterprise BAA |

Three Anthropic tiers + two specialty cross-provider tiers. 5 models in the portfolio; manageable operational complexity.

### 8.2 The routing architecture

The router is the classifier worker (Haiku-tier), implemented inside the agent gateway. The architecture:

1. User request arrives at the chat panel.
2. The supervisor agent (Opus) receives the request.
3. Supervisor dispatches to the classifier (Haiku) for question classification.
4. Based on classification, supervisor dispatches to the appropriate worker tier.
5. The worker's response flows back to the supervisor for consolidation.
6. Final response streams to the user.

The classifier serves as the router; tier selection is its output. The supervisor is the dispatcher.

### 8.3 The router calibration

The classifier prompt is versioned and eval-validated. Quarterly re-calibration includes:

- Per-class classification accuracy measurement.
- Per-tier quality SLI measurement (is the cheap-tier worker producing acceptable quality on its routed-down cases?).
- Cost-savings measurement (relative to all-on-Opus baseline).

Recent measurements:
- Classification accuracy: 94%.
- Per-tier quality: clinical = 95.2% (vs 95.4% all-Opus); drafting = 91.8% (vs 93.1% all-Opus); classification accuracy ratifies the routing.
- Cost savings: 57% relative to all-on-Opus baseline.

### 8.4 The portfolio's BAA story

All five models in the portfolio have BAA coverage for the workload's PHI handling. The audit trail surfaces this on every call (the model registry's `regulatory_coverage.baa_covered` field is propagated to traces and audit logs).

The dedicated-infrastructure premium customer has a separate portfolio of the same models on a dedicated Anthropic account; the routing layer routes premium-tenant traffic to that portfolio.

### 8.5 The 2026-04-29 portfolio incident

The cost incident referenced in [cost-budget-circuit-breaker.md](../../ai-engineering-reference-architecture/cost-and-finops/cost-budget-circuit-breaker.md) was traced to a portfolio drift: the Opus reference was an alias that resolved to a new model version with different pricing.

The remediation:
- All portfolio entries pinned to full version strings.
- Registry refuses alias registrations.
- Quarterly portfolio review confirms pinning discipline.

### 8.6 The portfolio's operational discipline

- Quarterly portfolio review (the 5 models reviewed for currency, pricing, deprecation timeline).
- Each tier has eval coverage on the workloads it serves.
- The classifier prompt (the router) is itself versioned and eval-gated.
- New model evaluation is a structured process (registry registration → canary → eval validation → portfolio admission).

---

## 9. Anti-patterns

### 9.1 "All-Opus everywhere"

The team uses the most capable model for everything because "it's the safest." Cost is 5x what tiering would produce; the savings are left on the table.

**Corrective.** Two-tier minimum for production workloads; tier the easy cases down to a cheaper model.

### 9.2 "Single-provider portfolio with no contingency"

The portfolio is entirely on one provider. The provider has an outage; the entire platform is down.

**Corrective.** Cross-provider for resilience-critical workloads; or document the single-provider risk and operational acceptance.

### 9.3 "Routing decisions in application code"

Each application implements its own routing. The decisions diverge over time; cost-savings vary by feature inconsistently.

**Corrective.** Platform-level routing per section 5.

### 9.4 "Tier proliferation"

The portfolio has 6+ tiers. Each tier has its own eval coverage; the operational load is unsustainable.

**Corrective.** Consolidate to 2-4 tiers. Specialty tiers added only with clear workload justification.

### 9.5 "Aliases in the portfolio"

The portfolio uses model aliases (claude-opus-latest). Provider-side version changes shift the portfolio without notice.

**Corrective.** Full version pins per [model-version-pinning.md](../../ai-engineering-reference-architecture/cicd-and-eval-gates/model-version-pinning.md).

### 9.6 "Regulatory bar checked retroactively"

The team builds the portfolio, then checks BAA / regulatory coverage. Some models in the portfolio lack coverage for the workload; production is non-compliant until migration.

**Corrective.** Regulatory check is question 3 in the portfolio decision; gates model admission.

### 9.7 "Specialty tiers added without measurement"

The team adds a fine-tuned classification tier "because the cost looks lower." Operational load (fine-tune lifecycle, eval, drift detection) consumes more than the savings; the specialty tier is a net loss.

**Corrective.** Specialty tiers added only when measurement shows they earn the operational cost.

### 9.8 "Portfolio frozen"

The portfolio was decided once and never revisited. New models land that could improve the cost-quality joint; the team does not consider them.

**Corrective.** Quarterly portfolio review per section 8.6.

---

## 10. Findings (sprint-assignable)

### ARCH-ROUTE-001 — Severity: High
**Finding.** Single-tier portfolio (all traffic on the most capable model); cost is much higher than tiered would produce.
**Recommendation.** Adopt a 2-3 tier portfolio per section 2; eval-validate; deploy.
**Owner.** ai-platform-eng + finops, sprint N+1.

### ARCH-ROUTE-002 — Severity: High
**Finding.** Portfolio includes model aliases; provider-side version shifts have caused incidents.
**Recommendation.** Pin to full versions per [model-version-pinning.md](../../ai-engineering-reference-architecture/cicd-and-eval-gates/model-version-pinning.md).
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-ROUTE-003 — Severity: High
**Finding.** Regulatory / BAA coverage was checked after portfolio selection; some models are non-compliant.
**Recommendation.** Confirm coverage as part of portfolio decision per section 6; remove non-compliant models.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+1.

### ARCH-ROUTE-004 — Severity: High
**Finding.** Router lives in application code; routing decisions diverge across features.
**Recommendation.** Platform-level router per section 5.1; centralize.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-ROUTE-005 — Severity: High
**Finding.** Portfolio is single-provider; provider outage takes down the platform.
**Recommendation.** Cross-provider for resilience-critical workloads per section 3.3; document the operational shape.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ARCH-ROUTE-006 — Severity: High
**Finding.** Portfolio includes tiers that are no longer earning their operational cost.
**Recommendation.** Portfolio contraction per section 7.4; consolidate.
**Owner.** ai-platform-eng + finops, sprint N+3.

### ARCH-ROUTE-007 — Severity: Medium
**Finding.** Tier-version migrations are unplanned; deprecation surprises cause production scrambles.
**Recommendation.** Migration plans per section 7.1; subscribe to provider deprecation announcements.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-ROUTE-008 — Severity: Medium
**Finding.** Specialty tiers were added without measurement; some are operational drag without compensating benefit.
**Recommendation.** Measure each specialty tier's cost-quality contribution; retire those that do not earn.
**Owner.** ai-platform-eng + finops, sprint N+3.

### ARCH-ROUTE-009 — Severity: Medium
**Finding.** Router decisions are not consistently logged; routing investigations require correlation.
**Recommendation.** Router decisions in trace per [retrieval-instrumentation.md](../../ai-engineering-reference-architecture/observability-and-telemetry/retrieval-instrumentation.md)-style attributes.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ARCH-ROUTE-010 — Severity: Medium
**Finding.** Per-feature routing is allowed without coordination; features make conflicting decisions about the same routing concern.
**Recommendation.** Coordinate per-feature routing through the platform; document deviations.
**Owner.** ai-platform-eng team lead, sprint N+3.

### ARCH-ROUTE-011 — Severity: Medium
**Finding.** Routing escalation pattern is not designed; cheap-tier failures fall through to the user.
**Recommendation.** Escalation per [tier-routing-for-cost.md](../../ai-engineering-reference-architecture/cost-and-finops/tier-routing-for-cost.md) section 5; engineered, not emergent.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-ROUTE-012 — Severity: Medium
**Finding.** Portfolio review is not scheduled; new models land without team awareness.
**Recommendation.** Quarterly portfolio review per section 8.6.
**Owner.** ai-platform-eng team lead, sprint N+3.

### ARCH-ROUTE-013 — Severity: Medium
**Finding.** Multi-region / data-residency requirements are not aligned with portfolio model availability.
**Recommendation.** Confirm per-region model availability for each portfolio entry; document constraints.
**Owner.** ai-platform-eng + compliance, sprint N+4.

### ARCH-ROUTE-014 — Severity: Medium
**Finding.** Premium-tier customer has the same portfolio as standard tier; the contractual differentiation is not architecturally enforced.
**Recommendation.** Per-tier portfolios (or portfolio variants); premium tenants route to the dedicated portfolio.
**Owner.** ai-platform-eng + customer-success, sprint N+4.

### ARCH-ROUTE-015 — Severity: Medium
**Finding.** Cross-provider portfolio's per-provider prompt portability is not validated; prompts may behave differently.
**Recommendation.** Eval coverage per provider; document portability findings.
**Owner.** ai-platform-eng + prompt-engineering, sprint N+4.

### ARCH-ROUTE-016 — Severity: Low
**Finding.** Portfolio capability ceiling is not documented; new feature scoping unclear on what the platform can do.
**Recommendation.** Document the capability ceiling alongside the portfolio; surface to feature teams.
**Owner.** ai-platform-eng + product, sprint N+5.

### ARCH-ROUTE-017 — Severity: Low
**Finding.** New-model evaluation has no defined process; ad-hoc trials accumulate without disposition.
**Recommendation.** Define new-model-eval process per section 7.3; structured eval → canary → admission or rejection.
**Owner.** ai-platform-eng team lead, sprint N+5.

### ARCH-ROUTE-018 — Severity: Low
**Finding.** Portfolio cost contribution per tier is not separately tracked; per-tier optimization opportunities are hidden.
**Recommendation.** Per-tier cost attribution in dashboards.
**Owner.** ai-platform-eng + finops, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team without a deliberate portfolio:

- [ ] **Sprint 0 — inventory.** Catalog every model in production use. Document costs and capabilities.
- [ ] **Sprint 0 — decide.** Answer the five questions per section 2. Choose the portfolio size and structure.
- [ ] **Sprint 1 — registry alignment.** Each portfolio entry registered per [model-registry.md](../../ai-engineering-reference-architecture/model-lifecycle/model-registry.md). BAA coverage verified.
- [ ] **Sprint 1 — version pinning.** All entries pinned to full version strings.
- [ ] **Sprint 2 — router architecture.** Platform-level router; placement decided per section 5.
- [ ] **Sprint 2 — router calibration.** Classification accuracy measured; routing eval-validated.
- [ ] **Sprint 3 — observability.** Router decisions on traces; per-tier dashboards.
- [ ] **Sprint 3 — escalation.** Per [tier-routing-for-cost.md](../../ai-engineering-reference-architecture/cost-and-finops/tier-routing-for-cost.md) section 5; engineered escalation paths.
- [ ] **Sprint 4 — migration planning.** Documented migration plans per tier; provider-deprecation subscriptions.
- [ ] **Sprint 5 — ongoing.** Quarterly portfolio review; recalibration cadence; new-model evaluation process.

A team that completes this sequence has the portfolio discipline that produces the right cost-quality joint for its workload. A team that defaults to "use the best model" pays the all-frontier-tier baseline indefinitely.

---

## 12. References

- This repo: [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](./) (coming) — the build-vs-buy-vs-train decision.
- This repo: [model-strategy/build-vs-buy-decision.md](./) (coming) — hosted vs self-hosted decision.
- This repo: [model-strategy/model-catalogue-and-registry.md](./) (coming) — the architecture-level catalog framework.
- This repo: [guardrails-and-policy-architecture/ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md) — the gateway where the router lives.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked architecture using these patterns.
- This repo: [reference-patterns/agent-topologies.md](../reference-patterns/agent-topologies.md) — supervisor / worker topology context.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — the engineering implementation this architectural decision flows into.
- Sibling repo: [ai-engineering-reference-architecture/model-lifecycle/model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md) — the registry that catalogs portfolio entries.
- Sibling repo: [ai-engineering-reference-architecture/cicd-and-eval-gates/model-version-pinning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cicd-and-eval-gates/model-version-pinning.md) — the release-side pinning discipline.
- Sibling repo: [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — security considerations across the portfolio.
