# Tier Routing for Cost

> **Audience.** Architects designing the routing layer that decides which model serves which request. Tech leads whose AI spend is 60% on workloads that could run on cheaper models. Anyone who has heard "we should route smarter" and is wondering what that actually means architecturally. **Scope.** The *architectural* design of cheap-first routing with escalation: router architecture options; routing-decision criteria; escalation triggers; per-workload routing policy; routing-as-cost-lever; integration with build-vs-buy and capability-vs-cost-vs-latency. Not the engineering of router implementation (see sibling [cost-and-finops/tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md)). Not the model selection per workload (see [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Most production AI workloads can be served by multiple model tiers:

- A simple classification question doesn't need a frontier model.
- A complex clinical reasoning task might need the most capable available.
- A long-form draft generation might fit in between.

Without tier routing: every request goes to the same model — usually the most capable one for safety. Cost per call is high; volume × per-call cost = wasted spend.

With tier routing: each request goes to the model that fits its needs. Cost can drop 40-70% with minimal quality impact.

The engineering-side document (sibling) covers the routing mechanics — how to implement the router, how to track costs, how to validate quality. This document covers the architectural decisions: which routing strategy fits, how to design the router, where to place it in the stack.

This document is opinionated about four things:

1. **Default to cheap; escalate on signal.** Cheap models handle most cases; escalation reserved for cases that benefit from premium capability.
2. **The signal for escalation matters more than the router's logic.** Low confidence, structured-output failure, complex-query class — these signals determine routing quality.
3. **Routing is per-workload, not platform-wide.** Different workloads have different escalation patterns.
4. **Validate routing with eval; don't trust it.** Routing decisions can degrade quality silently; eval-gated.

Structure: (2) the router architecture options; (3) routing decision criteria; (4) escalation triggers; (5) per-workload routing policy; (6) routing-as-cost-lever; (7) integration with broader model strategy; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The router architecture options

How the routing layer can be designed.

### 2.1 Rule-based router

```python
def route(request):
    if request.input_tokens > 8000:
        return "sonnet"  # long context
    if request.workload == "clinical_decision_support":
        return "sonnet"  # quality-critical
    if request.workload in ["faq", "simple_classification"]:
        return "haiku"
    return "sonnet"  # default
```

Explicit; predictable.

**Pros.** Simple; testable; debuggable.

**Cons.** Inflexible; doesn't adapt to nuance.

**When right.** Most workloads at most companies. Start here.

### 2.2 Classifier-based router

A small model (or learned classifier) predicts complexity:

```python
def route(request):
    complexity = complexity_classifier.predict(request.input)
    if complexity == "simple":
        return "haiku"
    elif complexity == "moderate":
        return "sonnet"
    else:
        return "opus"
```

ML-driven routing.

**Pros.** Handles cases rules miss; adapts.

**Cons.** Classifier needs training data; can be wrong; latency overhead (classification time).

**When right.** Mature platforms with clear complexity gradient.

### 2.3 LLM-as-router

Use an LLM to decide which LLM:

```python
routing_response = llm.call(
    prompt=f"Decide: which tier for this query? Query: {request.input}. Options: cheap, medium, expensive."
)
return routing_response.tier
```

LLM-based.

**Pros.** Sophisticated; flexible.

**Cons.** Adds LLM call latency; cost (the routing call); ironic.

**When right.** Rarely worth it; consider only for highly variable workloads.

### 2.4 Hybrid (rule + escalation)

Most production systems:

```python
def route(request):
    # Rules for clear cases.
    primary_model = rule_based_route(request)
    
    # Try primary.
    response = call(primary_model, request)
    
    # Escalation triggers.
    if response.confidence < threshold or response.refused or response.malformed:
        # Escalate to bigger model.
        return call(escalation_model, request)
    
    return response
```

Cheap-first; escalate on signal.

**Pros.** Best balance.

**Cons.** Implementation complexity; per-workload tuning.

**When right.** Most production workloads.

### 2.5 The architecture comparison

```
Architecture           Setup time    Latency    Cost savings   Flexibility
─────────────────────────────────────────────────────────────────────────────
Rule-based             Days           +0ms       Modest          Low
Classifier-based       Weeks          +50-200ms  Medium          Medium
LLM-as-router          Hours          +500-2000ms High overhead   High
Hybrid (rule+escalate) Days-weeks     +0-3000ms  Highest         Medium
```

Hybrid is the typical right answer.

### 2.6 The router placement

Router sits between application and LLM:

```
Application
    ↓
Router
    ↓
Model A | Model B | Model C
```

Logically: cross-cutting concern.

Physically: function in application code, or separate service.

### 2.7 The "router-as-platform-service" pattern

For platforms with many features:

- Routing logic centralized.
- Per-feature configuration.
- Single point of evolution.

Cross-link to [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md).

### 2.8 The "router-as-application-code" pattern

For simpler platforms:

- Each feature's code handles its routing.
- Per-feature rules.

Less centralized; more flexible per feature.

### 2.9 The choice

Most platforms: hybrid + platform-service.

---

## 3. Routing decision criteria

What goes into the decision.

### 3.1 Input characteristics

- Length (input tokens).
- Complexity (vocabulary, structure).
- Domain (technical, casual, etc.).

These can be measured pre-call.

### 3.2 Workload characteristics

- Real-time vs batch.
- Cost-sensitive vs quality-sensitive.
- Latency-critical vs tolerant.

These are per-workload constants.

### 3.3 Tenant characteristics

- Premium vs standard tier.
- Tenant-specific model preference.

Cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md §4](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 3.4 User characteristics

- User's role (admin vs end-user).
- User's history (sometimes).

Per-user.

### 3.5 Time characteristics

- Peak vs off-peak (cost may shift).
- Time of day (provider rate-limit pressure).

Operational adjustment.

### 3.6 The pre-call estimation

Before routing, estimate cost / latency / capability needed:

```python
def estimate(request):
    return {
        "estimated_tokens": count_tokens(request.input),
        "estimated_complexity": estimate_complexity(request),
        "workload_priority": request.workload_priority,
    }
```

Inputs to the router.

### 3.7 The post-call signal

After the call, signal informs future routing:

- Was the response good?
- Was confidence sufficient?
- Was the user satisfied?

Feedback loop.

### 3.8 The "we have no escalation signal" gap

Without escalation signal:

- Router can't escalate.
- Cheap-first is risky.

Build signals first.

### 3.9 The composite signal

Most routers use composite:

- Input characteristics + workload + tenant + time.
- Weighted; rule-based.

Per-workload tuned.

---

## 4. Escalation triggers

What signals "escalate to bigger model."

### 4.1 The confidence signal

If the model exposes confidence (or you compute):

- Low confidence → escalate.

Typical: model's logprobs or self-reported uncertainty.

### 4.2 The structured-output failure

For structured outputs:

- Failed schema validation → escalate.

Cross-link to [reference-patterns/structured-output-patterns.md](../reference-patterns/structured-output-patterns.md).

### 4.3 The refusal signal

If the model refused:

- May indicate complexity.
- Escalate to see if larger model can handle.

Sometimes; not always (refusal may be appropriate).

### 4.4 The user-feedback signal

If user thumbs-down:

- Future similar requests escalate.

Feedback-driven.

### 4.5 The complexity-detected post-call

After response:

- If response is brief / incomplete on a complex-seeming query.
- Possibly escalate.

Self-evaluation.

### 4.6 The "explicit complexity flag in request" pattern

For some workloads:

- User / caller indicates "this is complex."
- Route accordingly.

When the caller knows.

### 4.7 The escalation cost

Each escalation: original call cost + escalation call cost.

If escalation rate is high (>30%): cheap-first isn't saving much.

Tune triggers.

### 4.8 The "never escalate" default

For some workloads:

- Always cheap.
- Accept the quality.

For non-critical workloads (internal copilot, draft generation).

### 4.9 The "always premium" override

For some workloads:

- Always escalate (i.e., always premium).
- For safety-critical or latency-critical.

Per-workload.

### 4.10 The escalation logging

Track:

- Per-request: cheap chosen or escalated.
- Per-workload: escalation rate.
- Per-cause: which trigger.

Inform tuning.

---

## 5. Per-workload routing policy

How different workloads route.

### 5.1 The workload matrix

```
Workload                    Primary tier    Escalation    Trigger
──────────────────────────────────────────────────────────────────────
Patient API chat (FAQ)      Haiku           Sonnet         Low confidence
Patient API chat (clinical) Sonnet          Opus           Quality flag
Care Coordinator agent      Sonnet (steps)   -              n/a
Clinical decision support   Sonnet          Opus           Always (safety)
Document classification     Haiku           Sonnet         Confidence < 0.85
Embedding                   BGE              -              n/a
Internal copilot            Haiku            -              n/a (low stakes)
Analytics SQL copilot       Sonnet          Opus           Complex schema
```

Per-workload policy.

### 5.2 The policy review

Quarterly:

- Per workload: is the policy still right?
- Has volume changed?
- Has quality drift?

Tune.

### 5.3 The per-tenant override

Premium tenants:

- May get higher tier as default.
- Or may opt for the standard tier (cost-conscious customers).

Per-tenant configuration.

### 5.4 The locale considerations

Some locales need different routing:

- Languages where one model is stronger.
- Multilingual specifics.

Per-locale routing.

### 5.5 The "we have one workload with many query types" complexity

Within a workload, types differ:

- Greeting → cheap.
- Clinical question → sonnet.

Sub-workload routing per request.

### 5.6 The fallback chain

For each workload, a fallback chain:

```
Patient API chat:
  Primary: Sonnet US
  Fallback: Haiku US (provider issue on Sonnet)
  Fallback: Cached response
  Fallback: Templated response
```

Cross-link to [reliability-engineering/fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md).

### 5.7 The "we deviate from policy" exception

Some requests bypass routing:

- Internal debug.
- Specific user with specific need.

Audited.

### 5.8 The cost-conscious tenant overlay

Some tenants pay for cheaper tier:

- Their policy: only Haiku.
- Acceptable quality for their use case.

Cost differentiation.

### 5.9 The metric per workload

Track:

- Primary-tier usage %.
- Escalation rate.
- Per-tier cost.

Per workload; reviewed.

---

## 6. Routing-as-cost-lever

The savings model.

### 6.1 The savings math

For a workload:

```
With routing:
  60% Haiku at $0.004/call
  35% Sonnet at $0.020/call  
  5% Opus at $0.080/call (escalated)
  Avg: $0.012/call

Without routing (all Sonnet):
  $0.020/call

Savings: 40%.
```

40-70% typical for tier-routable workloads.

### 6.2 The break-even analysis

Routing complexity vs savings:

- Implementation: 2-4 weeks engineering.
- Operational: ongoing tuning.
- Savings: per-month cost reduction.

Payback usually 1-3 months for moderate-volume workloads.

### 6.3 The volume-driven justification

Routing benefits scale with volume:

- Small volume ($1k/month): savings marginal.
- Medium volume ($10k/month): substantial savings.
- High volume ($100k/month): massive savings.

Justify per workload's volume.

### 6.4 The quality-vs-cost trade-off

More aggressive routing (more cheap):

- Higher savings.
- Higher escalation rate.
- Possible quality degradation.

Per-workload tuning.

### 6.5 The "we cut cost 50%" hype

Per-workload savings vary:

- Some workloads: 50%+ achievable.
- Some workloads: 10-15% (mostly already on cheap; escalation rare).

Per-workload measurement.

### 6.6 The cost-attribution per tier

Per workload:

- Cost broken down by tier (Haiku $X, Sonnet $Y, Opus $Z).
- Each tier's contribution.

Inform whether routing is achieving its goal.

### 6.7 The "we should route more aggressively" iteration

If primary-tier usage > expected:

- Many workloads getting escalated.
- Investigate signals.

If escalation rate < expected:

- Maybe cheap-tier is delivering.

Tune iteratively.

### 6.8 The "tiers we should add" expansion

Multi-tier:

- Haiku + Sonnet (2 tiers).
- + Opus (3 tiers).
- + Self-hosted Llama (4 tiers).

Each tier adds complexity; rare to need more than 3.

---

## 7. Integration with broader model strategy

How tier routing relates to other model decisions.

### 7.1 The capability-vs-cost-vs-latency input

Routing decisions are informed by per-workload capability/cost/latency analysis:

Cross-link to [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md §6](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md).

Per workload: which tiers are eligible; per-tier capability.

### 7.2 The model-routing-and-tiering pattern

Cross-link to [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md).

Architectural router design.

### 7.3 The build-vs-buy implications

Self-hosted models can be tiers:

- Tier 0: self-hosted Llama 8B (cheapest).
- Tier 1: hosted Haiku.
- Tier 2: hosted Sonnet.

Cross-link to [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md).

### 7.4 The multi-tenancy interaction

Per-tenant tier preferences:

- Free: only cheap.
- Standard: cheap with escalation.
- Premium: premium default.

Cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md §4](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 7.5 The catalogue integration

Each tier has a catalogue entry (cross-link to [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md)):

- Tier 1 (Haiku) catalogue entry.
- Tier 2 (Sonnet) catalogue entry.
- Tier 3 (Opus) catalogue entry.

Router consults catalogue.

### 7.6 The migration consideration

When migrating models (cross-link to [model-strategy/model-migration-playbook.md](../model-strategy/model-migration-playbook.md)):

- Update tier in catalogue.
- Router automatically routes to new version.

### 7.7 The capacity considerations

Cross-link to [cost-and-performance-architecture/throughput-and-concurrency.md](./throughput-and-concurrency.md).

Each tier has its own capacity / rate limits.

Router respects per-tier headroom.

### 7.8 The cost-incident integration

When cost budget burns:

- Route to cheaper tier (route-down-tier mitigation).
- Cross-link to [cost-incident-playbook.md](./cost-incident-playbook.md).

Routing as cost-control.

---

## 8. Worked Meridian example

Meridian's tier-routing architecture.

### 8.1 The routing layer

Architecture: hybrid (rule + escalation), platform-service.

```
Application code
    ↓
Router service (Python; integrated with feature configs)
    ↓
Model providers (Anthropic, Cohere, self-hosted)
```

Centralized; consistent across features.

### 8.2 The router rules

Per workload, configured:

```yaml
care-coordinator:
  primary_tier: sonnet
  escalation_tier: opus
  escalation_triggers:
    - quality_judge_low_confidence: < 0.7
    - explicit_complex_flag

patient-api-chat:
  primary_tier: sonnet
  escalation_tier: opus
  escalation_triggers:
    - quality_judge_low_confidence: < 0.75

internal-copilot:
  primary_tier: haiku
  escalation_tier: none

document-classification:
  primary_tier: llama-self-hosted
  escalation_tier: sonnet
  escalation_triggers:
    - schema_validation_failure
    - confidence < 0.85

analytics-sql-copilot:
  primary_tier: sonnet
  escalation_tier: opus
  escalation_triggers:
    - schema_validation_failure
    - complex_query_signal
```

Per workload.

### 8.3 The savings achieved

```
Workload                 Pre-routing cost   Post-routing cost   Savings
──────────────────────────────────────────────────────────────────────
Care Coordinator         $30k/month         $30k/month           0% (primarily Sonnet)
Patient API chat (US)    $14k/month         $14k/month           0% (all Sonnet for quality)
Document classification  $25k/month         $5k/month            80% (self-host primary)
Analytics SQL copilot    $8k/month          $5k/month            38% (most queries are simple)
Internal copilot         $1k/month          $200/month           80% (Haiku primary)
─────────────────────────────────────────────────────────────────────
Total                    $78k/month         $54k/month           ~30%
```

Significant savings; per-workload varied.

### 8.4 The escalation rates

```
Care Coordinator: 8% escalation to Opus (clinical complexity)
Patient API chat: 12% escalation
Document classification: 5% escalation
Analytics SQL: 22% escalation (complex schemas)
```

Tuned per workload.

### 8.5 The Q1 2026 routing review

Annual review of per-workload routing:

- Patient API chat: Sonnet was primary; consider Haiku for FAQ subset?
- Eval showed Haiku acceptable for FAQ (~20% of traffic).
- New routing: Haiku for FAQ; Sonnet for non-FAQ.
- Additional 15% savings.

Iterative tuning.

### 8.6 The Q2 2026 escalation-signal investigation

Care Coordinator escalation rate climbed from 8% to 14%.

Investigation:

- Was the model getting worse? No (eval was stable).
- Was the workload getting more complex? Yes (newer clinical workflows).
- Decision: accept escalation increase; tune trigger to be less sensitive.

Quarterly tuning.

### 8.7 The integration with cost-incident response

During the Q1 2026 cost-bloat incident:

- Cost-budget circuit-breaker fired.
- Router temporarily forced to cheaper-tier for affected feature.
- Cost contained.

Routing infrastructure as part of cost-incident response.

### 8.8 The cost of the routing layer

- Engineering: 4 weeks initial; 1 week per workload tuning.
- Ongoing: ~5% of platform team for tuning.

### 8.9 The benefits realized

- $24k/month savings ongoing (after routing optimization).
- Quality maintained (per eval).
- Cost-incident response improved (routing as mitigation).

### 8.10 The lessons

- Routing isn't worth it for every workload; per-workload analysis.
- Escalation signal is the load-bearing decision; build signals first.
- Quarterly tuning prevents drift.
- Integration with broader model strategy (catalogue, capacity, cost) is essential.

---

## 9. Anti-patterns

### 9.1 The "always sonnet" default

**Pattern.** Single model for everything; tier savings ignored.

**Corrective.** Per-workload tier per §5.

### 9.2 The no-escalation router

**Pattern.** Cheap tier always; quality degrades.

**Corrective.** Escalation triggers per §4.

### 9.3 The over-aggressive escalation

**Pattern.** Most requests escalate; cheap-tier rarely chosen; no savings.

**Corrective.** Tune triggers per §4.7.

### 9.4 The router-without-eval validation

**Pattern.** Routing chosen; never eval'd; quality silently degrades.

**Corrective.** Eval-driven validation per §3.7.

### 9.5 The LLM-as-router for everything

**Pattern.** Use LLM for routing; latency + cost overhead exceeds savings.

**Corrective.** Hybrid pattern per §2.4.

### 9.6 The per-feature router duplication

**Pattern.** Each feature has its own router code; inconsistent; hard to maintain.

**Corrective.** Platform-service routing per §2.7.

### 9.7 The "we never tune the router" stagnation

**Pattern.** Initial config; never updated. Drift; sub-optimal.

**Corrective.** Quarterly review per §5.2.

### 9.8 The "we don't measure escalation rate" blindness

**Pattern.** No metric; routing effectiveness unknown.

**Corrective.** Escalation logging per §4.10.

### 9.9 The "premium tenants get premium" wastefulness

**Pattern.** Premium tenants get Opus for everything; their cheap queries are over-served.

**Corrective.** Per-request tier; tenant preference influences but doesn't override.

### 9.10 The fallback-chain absence

**Pattern.** Primary tier fails; no fallback; user sees outage.

**Corrective.** Fallback chain per §5.6.

---

## 10. Findings (sprint-assignable)

**ARCH-TRC-001 (P0). Tier routing not deployed for cost-sensitive workloads.** 40-70% savings on table. Per §2 and §5. Owner: AI platform + FinOps.

**ARCH-TRC-002 (P0). Single model for all workloads.** Wasted spend. Per §5. Owner: AI platform.

**ARCH-TRC-003 (P0). No escalation signal.** Cheap-first impossible. Per §4. Owner: AI platform.

**ARCH-TRC-004 (P1). Routing not eval-validated.** Quality may degrade silently. Per §3.7. Owner: AI platform + eval.

**ARCH-TRC-005 (P1). Per-feature routing code duplicated.** Maintenance burden. Per §2.7. Owner: AI platform.

**ARCH-TRC-006 (P1). Quarterly tuning absent.** Drift; sub-optimal. Per §5.2. Owner: AI platform.

**ARCH-TRC-007 (P1). Per-tier cost not tracked per workload.** Hard to see savings. Per §6.6. Owner: observability + FinOps.

**ARCH-TRC-008 (P1). Escalation rate per workload not monitored.** Trigger tuning blind. Per §4.10. Owner: AI platform.

**ARCH-TRC-009 (P2). Fallback chain absent per workload.** Provider issues = outage. Per §5.6. Owner: AI platform.

**ARCH-TRC-010 (P2). Per-tenant tier preference not implemented.** Premium tier capability not differentiated. Per §5.3. Owner: AI platform + product.

**ARCH-TRC-011 (P2). Self-hosted tier not in router.** Cost-savings opportunity. Per §7.3. Owner: AI platform.

**ARCH-TRC-012 (P2). Router-as-platform-service not implemented.** Per-feature duplication. Per §2.7. Owner: AI platform.

**ARCH-TRC-013 (P2). LLM-as-router used unnecessarily.** Latency + cost overhead. Per §2.3. Owner: AI platform.

**ARCH-TRC-014 (P2). Catalog-integration absent.** Router has its own model list. Per §7.5. Owner: AI platform.

**ARCH-TRC-015 (P3). Sub-workload routing not used.** Sub-types within a workload all on same tier. Per §5.5. Owner: AI platform.

**ARCH-TRC-016 (P3). Time-aware routing absent.** Off-peak cost shifts not exploited. Per §3.5. Owner: AI platform.

**ARCH-TRC-017 (P3). Cost-incident-routing integration absent.** Routing-as-mitigation unavailable. Per §7.8. Owner: AI platform.

**ARCH-TRC-018 (P3). Annual model-portfolio review absent.** Routing decisions stale. Owner: AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Per-workload tier-routing decision (§5).**
- [ ] **Build escalation signal per workload (§4).**
- [ ] **Implement hybrid router (§2.4).**
- [ ] **Platform-service routing (§2.7).**
- [ ] **Per-tier cost / metric tracking (§6.6, §4.10).**
- [ ] **Quarterly routing review (§5.2).**
- [ ] **Fallback chain per workload (§5.6).**
- [ ] **Per-tenant tier preference (§5.3).**
- [ ] **Catalog integration (§7.5).**
- [ ] **Cost-incident-routing integration (§7.8).**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — cost framework.
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — latency.
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — capacity.
- [caching-tiers.md](./caching-tiers.md) — caching as cost lever.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) — self-host as tier.
- [cost-incident-playbook.md](./cost-incident-playbook.md) — routing-as-mitigation.

**Elsewhere in this repo.**
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — routing architecture.
- [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md) — per-workload analysis.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — self-hosted tiers.
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant tier.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — engineering mechanics (companion).
- [ai-engineering-reference-architecture / reliability-engineering / fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) — fallback patterns.

**External.**
- Provider tier comparisons (Anthropic Haiku/Sonnet/Opus, OpenAI GPT-4o/o1, Google Gemini variants).
- A/B testing literature for routing validation.
