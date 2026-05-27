# Multi-Model Orchestration

> **Audience.** Architects designing systems that use more than one model. Tech leads weighing "one big model" vs "router + tiered models." Anyone whose AI feature's cost is dominated by frontier-model usage when half the traffic would be served by a cheap model. **Scope.** The *architectural* patterns: router types (rules, LLM-as-router, classifier-as-router); tier-routing (cheap-first vs capability-first); ensemble patterns; model-portfolio architecture for resilience. Not the engineering of model wrappers (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)). Not the model-selection decision itself (see [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Few production AI systems run on a single model. Most use 2-5 models in some combination:

- A cheap small model for orchestration / classification / simple cases.
- A medium model for typical workloads.
- A frontier model for hard cases / synthesis / high-stakes.
- A specialised model for specific tasks (code embedding, multilingual, medical-domain).
- A judge model for eval / moderation.

The architectural question isn't "which model" — it's "how do these models combine into a coherent system." The router pattern (something decides which model handles a given request) is the central design choice. The ensemble pattern (multiple models contributing to one decision) is the next layer. The portfolio pattern (multiple models maintained for resilience and capability coverage) is the strategic layer.

The wrong orchestration produces predictable failures:

- **Over-frontier.** Everything goes to the most expensive model; cost is 3-5× what tier routing would deliver.
- **Under-frontier.** Hard cases routed to weak models; quality drops.
- **Brittle router.** The router itself fails; system has no fallback; service degrades.
- **Ensemble overhead.** N models called when 1 would do; latency and cost compound.
- **Single-provider risk.** One vendor; one model family; vendor outage takes the system down.

This document covers the patterns that produce good orchestration: which router type for which workload; tier routing done well; when ensembling is justified; how the model portfolio earns its operational cost.

This document is opinionated about four things:

1. **The router is an architectural component, not a quick conditional.** Routing decisions are explicit, observable, eval'd, and changeable.
2. **Tier routing is the default for cost-sensitive workloads.** Most workloads have a mix of easy and hard requests; routing them to appropriately-sized models is the primary cost lever.
3. **Ensembles are rare; reserve for high-stakes.** Cost compounds; latency suffers; quality gain often modest.
4. **The portfolio is part of architecture, not procurement.** Multiple providers / model families is resilience; single-vendor lock-in is a real risk in 2026.

Structure: (2) the router types; (3) tier routing patterns; (4) ensemble patterns; (5) the model portfolio for resilience; (6) router engineering and observability; (7) the cost / quality / latency tradeoffs; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The router types

What decides which model handles a request.

### 2.1 Rules-based router

A deterministic conditional decides:

```python
def select_model(request):
    if request.intent == "simple_lookup":
        return "haiku"
    if request.complexity > 0.7:
        return "opus"
    if request.feature == "code_assist":
        return "voyage-code"
    return "sonnet"
```

**Pros.**
- Predictable; testable.
- Cheap (no LLM call to route).
- Fast (microseconds).
- Easy to reason about.

**Cons.**
- Brittle on novel inputs.
- Requires explicit features (intent, complexity) which themselves need engineering.
- Edge cases proliferate over time.

**When right.**
- Clear request categories that map to clear model choices.
- Cost-sensitive workloads where router cost matters.
- Teams with strong product / domain understanding to enumerate the categories.

### 2.2 Classifier-based router

A small ML classifier (often a fine-tuned smaller LLM or a non-LLM classifier) categorises requests; routing is per-category:

```
Request → classifier → category → category-to-model mapping → model call
```

**Pros.**
- Handles novel inputs better than rules (generalises from training).
- Fast inference (small classifier).
- Trainable on production data.

**Cons.**
- Training data requirement.
- Classifier maintenance (drift over time).
- Classification errors propagate to routing errors.

**When right.**
- Request categories are diverse enough that rules don't generalise.
- Production data available to train.
- Team has ML engineering capacity.

### 2.3 LLM-as-router

A small LLM (often Haiku-class) judges the request and decides:

```
Request → LLM-as-router prompt → JSON {model: "...", reason: "..."} → 
  model call
```

**Pros.**
- Flexible (handles any input shape).
- Quick to deploy (just a prompt).
- Reasons readable.

**Cons.**
- Adds latency (~200-500ms for the router call).
- Adds cost (per-request router cost).
- Stochasticity in routing decisions.

**When right.**
- Request shapes too varied for rules or stable classifier.
- Latency budget can absorb the routing call.
- Team prefers prompt-iteration speed over training cycles.

### 2.4 Hybrid router

A small router does the bulk; an LLM handles the uncertain edge:

```
Request → rules + classifier (fast) →
  if clear: route directly
  if uncertain: LLM-as-router (slow but accurate) → route
```

**Pros.**
- Best of both: fast for clear cases, accurate for uncertain.
- Bounded LLM-router cost (only on uncertain).

**Cons.**
- Operational complexity (two router layers).

**When right.**
- Bulk of traffic is clear; minority is genuinely ambiguous.
- Team values cost efficiency and quality on edge cases.

### 2.5 The router-type decision

| Criterion | Rules | Classifier | LLM | Hybrid |
| --- | --- | --- | --- | --- |
| Predictability | High | Medium | Lower | Medium |
| Latency | Microseconds | Milliseconds | Hundreds of ms | Mixed |
| Cost | Free | Low | Per-request | Mostly low |
| Handles novel | Poor | Good | Excellent | Good |
| Operational complexity | Low | Medium | Low | Medium |
| Maintenance | Manual rules updates | Training data + retraining | Prompt updates | Both |

Default to rules; graduate to classifier when rules become unwieldy; LLM-as-router when even classifier struggles.

### 2.6 The "what does the router decide" scope

Routers decide:

- Which model (tier choice).
- Which retrieval source.
- Which tool subset.
- Which prompt variant.
- Whether to refuse / escalate.

A single router can decide multiple; or each decision is its own router.

### 2.7 The router as architectural choke point

The router is where orchestration decisions live. It's worth treating it as an architectural object:

- Versioned.
- Tested.
- Eval'd.
- Observable.
- Owned.

Routers buried in calling code drift; routers as platform components don't.

---

## 3. Tier routing patterns

The most common orchestration: route to model tier based on request complexity.

### 3.1 Cheap-first / escalate-on-signal

Default to the cheap tier; escalate if quality signal is bad:

```
Request → Haiku (cheap) → 
  if confidence < 0.7: → Sonnet (medium) → 
    if confidence < 0.7: → Opus (frontier)
```

**Pros.**
- Bulk of traffic handled cheaply.
- Cost minimised.
- Quality preserved on hard cases.

**Cons.**
- Latency for escalated cases (multiple sequential calls).
- Confidence signal must be reliable.
- Sequential escalation has higher worst-case cost than single-call.

**When right.** Workloads where most requests are easy; few need frontier capabilities.

### 3.2 Classify-then-route

Classify request complexity; route to appropriately-sized model:

```
Request → classifier (fast) → complexity_tag →
  simple: → Haiku
  medium: → Sonnet
  complex: → Opus
```

**Pros.**
- Parallel cost (just one model call after classification).
- Bounded latency.
- Predictable routing.

**Cons.**
- Classification must be accurate.
- Mid-complexity cases may go wrong tier.

**When right.** Workloads with stable request distribution; classification trainable.

### 3.3 Per-feature tiering

Different features get different default tiers:

```
Care Coordinator (high-stakes) → Sonnet/Opus.
Patient API (high-volume) → Haiku.
Analytics Copilot (specialised) → Sonnet for SQL gen, Haiku for everything else.
```

Each feature's tier choice is a product decision; less per-request routing.

### 3.4 Capability-first / specialize-on-need

Default to a capable tier; specialise (cheaper, faster) when the request fits a known specialised path:

```
Request → Sonnet by default →
  if request matches "translation" pattern: → specialised translation model →
  if request matches "code" pattern: → specialised code model
```

**Pros.**
- Quality assured by default.
- Specialised paths give cost / quality lift for specific cases.

**Cons.**
- Default cost is higher than cheap-first.
- Pattern matching for specialisation must be accurate.

**When right.** Quality is paramount; specialisation produces meaningful gains.

### 3.5 Within-feature tier routing for agent loops

Within an agent loop, per-turn tier routing:

- Supervisor turns (planning) → Haiku.
- Specialist tool calls → Sonnet (the tool's internal LLM call).
- Final response synthesis → Sonnet.

Per [agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md) in the engineering sibling.

### 3.6 The tier-routing accuracy

The router's accuracy matters:

- If classifier sends too many to expensive tier: cost rises; no quality benefit.
- If classifier sends too few: quality drops on edge cases.

Eval the router's accuracy quarterly; adjust thresholds.

### 3.7 Tier routing in agent step

For hybrid (workflow + agent step) architectures: the outer workflow may not route by tier, but the inner agent step does. The agent step's runner consults the tier-routing policy per turn.

### 3.8 The "always frontier" reset

Some teams' first version is "everything on the frontier model" (the prototype was built that way). Tier routing introduced later:

- Eval reveals which tiers handle which requests well.
- Routing rules / classifier deployed.
- Cost typically drops 60-80%.

The reset is one of the highest-ROI optimizations a team can make on a mature AI feature.

---

## 4. Ensemble patterns

Multiple models contribute to a single decision. Rare; reserve for high-stakes.

### 4.1 Vote-based ensemble

Multiple models answer; vote determines final:

```
Request → 3 models (parallel) → 3 answers → majority vote → final answer
```

**Pros.**
- Robust to single-model failure.
- Quality typically higher than any single model.

**Cons.**
- 3× cost.
- 3× provider risk (one slow model blocks the answer).
- Disagreement-resolution logic.

**When right.** High-stakes decisions where the marginal cost is acceptable.

### 4.2 Hierarchical ensemble

Different models for different sub-tasks; results combine:

```
Request → 
  Model A: classify
  Model B: extract entities
  Model C: synthesise response
  → combined output
```

Less an ensemble than a pipeline; each step uses the best model for the step.

### 4.3 Critic ensemble

One model generates; another critiques:

```
Request → generator model → draft → critic model → 
  if critique requires revision: → generator (with critique) → final
```

**Pros.**
- Higher quality than single model on complex generation tasks.
- Self-improving via critique cycle.

**Cons.**
- Adds latency.
- Bounded by critic's quality.
- Repair loops can spiral (see anti-pattern in agent docs).

**When right.** High-quality bar; latency tolerance.

### 4.4 LLM-as-judge ensemble (separate model)

A different model judges the primary's output:

```
Request → primary model → output → judge model (different family) → 
  if judge approves: return output
  if judge disapproves: regenerate / escalate
```

The cross-family judge catches issues the primary doesn't notice; common for safety.

### 4.5 The cost-quality trade-off

Ensembles cost N× the single-model price for ~10-20% quality gain typically. Reserve for:

- High-stakes decisions.
- Cases where eval shows ensemble materially better.

For most workloads, a single good model + good eval > ensemble.

### 4.6 The "everything is ensemble" anti-pattern

Some teams use ensembles for routine work because "redundancy is good." Cost explodes; latency suffers. Use sparingly.

### 4.7 The "constitutional AI" framing

Some safety patterns use a critic model as the constitutional layer. Anthropic's Claude has built-in constitutional training; explicit critic ensembles are typically applied for additional / domain-specific safety beyond what built-in provides.

---

## 5. The model portfolio for resilience

Multiple model families for vendor-risk mitigation and capability coverage.

### 5.1 The vendor-risk reality

In 2026, vendor risk is real:

- Provider outage (Anthropic, OpenAI have had hours-long outages).
- Pricing changes (margins compress; surprise increases).
- Model deprecation (older models retire; the team's eval is on the deprecated one).
- Terms changes (data usage; commercial terms).
- Acquisition / business changes (vendor strategy shifts).

A single-vendor architecture has all this risk concentrated.

### 5.2 The portfolio approach

Maintain multiple vendor relationships:

- Primary: the team's default model family (e.g., Anthropic).
- Secondary: a fallback ready to handle critical workloads if primary fails (e.g., OpenAI).
- Specialised: providers for specific capabilities (Voyage for code embeddings; Cohere for rerank).

The portfolio is engineering work: integrations, evals, ongoing maintenance per provider.

### 5.3 The failover pattern

```
Primary model call →
  if provider available: → use
  if provider failing: → fallback to secondary provider
```

**Pros.**
- Resilience to provider outage.
- Single-request fallover.

**Cons.**
- Secondary may have different behaviour; eval each.
- Fallover triggers must be reliable.
- Cost / quality may differ on secondary.

**When right.** Critical workloads where outage is unacceptable.

### 5.4 The portfolio eval

Each model in the portfolio is eval'd:

- Same eval set across providers.
- Quality scored per provider.
- Cost compared.
- Latency compared.

The eval informs:

- Which provider is primary.
- Which is secondary.
- When to promote secondary to primary (if it becomes better).

### 5.5 The capability portfolio

Beyond resilience, portfolio for capability coverage:

- Voyage for code embeddings (specialised).
- Anthropic for general chat (frontier).
- OpenAI for some structured outputs.
- Specialised medical embedding for clinical content.

Each provider does what they're best at; the architecture uses them accordingly.

### 5.6 The portfolio cost

Multiple vendor relationships have cost:

- Per-vendor minimums (sometimes).
- Per-vendor integration work.
- Per-vendor eval cost.

Budget for the portfolio; it's not free.

### 5.7 The "always single vendor" risk

Some teams commit to a single vendor for procurement simplicity. The risk:

- Vendor's outage becomes the team's outage.
- Vendor's pricing change becomes the team's cost shock.
- Vendor's deprecation becomes the team's forced-migration.

Single-vendor is acceptable for low-stakes workloads; risky for production systems.

### 5.8 The migration playbook

If the team needs to move from one provider to another (deprecation, cost, quality):

- Eval the alternative on the team's golden set.
- Per-feature rollout (start with low-stakes; graduate).
- Tier-routing layer absorbs the switch.

Per [model-migration-playbook.md](../model-strategy/model-migration-playbook.md) in model-strategy.

---

## 6. Router engineering and observability

The platform layer for orchestration.

### 6.1 Router as platform component

The router lives in the AI gateway (per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md)) or alongside it. Calling code doesn't implement routing; it asks the gateway.

### 6.2 Router observability

Per-request:

- Which model was selected.
- Why (rule that matched; classifier output; LLM-router reasoning).
- Selection latency.
- Result quality (if measurable later).

Aggregations:

- Per-model usage rates over time.
- Routing-error rate (cases where the eval shows wrong tier).
- Per-feature routing distribution.

### 6.3 Router eval

A routing eval set:

- Test cases with the "correct" routing answer.
- Per-router run; report accuracy.

Quarterly review; tune rules / retrain classifier / update LLM-router prompt.

### 6.4 The router's failure mode

If the router itself fails:

- **Fail closed.** Reject the request (safer; user gets clear error).
- **Fail open with default model.** Proceed with a default (the team's standard model); log the router failure.

The choice depends on the team's risk tolerance; both are valid.

### 6.5 The router's cost

Router cost itself:

- Rules: free.
- Classifier: ~$0.001 per call (small fixed cost).
- LLM-as-router: ~$0.002-0.005 per call.

The router's cost is part of the total; budget accordingly.

### 6.6 The router's evolution

Routers age. Production data reveals patterns the original router didn't cover. Quarterly review:

- New request patterns → new rules / new classifier training.
- Cost drift → tier-routing thresholds re-tuned.
- Quality drift → router accuracy re-eval'd.

### 6.7 The "single router for all features" decision

Some platforms have one router for all AI features; others have per-feature routers.

- **Single router.** Easier to maintain; consistent behaviour; harder to tune per feature.
- **Per-feature.** Each feature optimised; more maintenance burden.

Most production deployments: a platform router with per-feature overrides. The platform provides the framework; features customise.

---

## 7. The cost / quality / latency tradeoffs

Orchestration's central trade-off.

### 7.1 The cost dimension

- Always-frontier: highest cost.
- Tier routing: 30-70% cost reduction typical.
- Ensembles: 2-3× cost.
- Failover: ~same cost (only triggers on primary failure).

### 7.2 The quality dimension

- Always-frontier: best per-call quality (model's best); doesn't beat ensemble or specialist.
- Tier routing: bulk of traffic same quality (frontier on hard); slight drop on cheap-tier-handled.
- Ensembles: 10-20% quality gain on consensus.
- Specialist: higher quality on specialised tasks.

### 7.3 The latency dimension

- Single-tier: lowest latency (one call).
- Tier routing: low (classify + call; rarely escalate).
- Ensembles: higher (parallel reduces; sequential adds).
- Failover: low normally; high on failover.

### 7.4 The orchestration sweet spot

For most workloads: tier routing with classifier; failover for resilience. Captures most of the cost savings and most of the quality; modest latency overhead.

### 7.5 The "we want all three optimised" reality

Cost, quality, and latency can't all be optimised simultaneously beyond a point. Each axis pulls.

- Cost-optimised: cheaper tier on more traffic; quality bound; latency low.
- Quality-optimised: frontier on more traffic; cost up; latency may rise (multi-step ensembles).
- Latency-optimised: cheap single-call; quality bound; cost low.

Pick two; accept the third trades off.

### 7.6 The product-decision input

Orchestration choices are partly product:

- Latency-critical feature → latency-optimised orchestration.
- Cost-sensitive feature → cost-optimised.
- Quality-critical feature → quality-optimised.

The team's product priorities translate to the orchestration architecture.

---

## 8. Worked Meridian example

Meridian's multi-model orchestration.

### 8.1 The model portfolio

| Provider | Models used | Purpose |
| --- | --- | --- |
| Anthropic | Haiku, Sonnet, Opus | Primary text generation across all features |
| OpenAI | Embedding 3 (text-embedding-3-large) | Embeddings for retrieval |
| Voyage | voyage-code-3 | Code embeddings (developer copilot) |
| Cohere | rerank-3 | Reranking in retrieval pipelines |
| (Standby) OpenAI | GPT-5 | Failover for text generation |

Five providers across the portfolio. Each with eval coverage; each with operational support.

### 8.2 The Care Coordinator routing

Multi-layer routing:

- **Outer workflow routing (rules).** Classify question type (clinical / scheduling / lookup / escalation); each type has a default model tier.
- **Inner agent step (rules-based per turn).** Supervisor turns on Haiku; specialist tool calls on Sonnet; final synthesis on Sonnet.
- **Failover.** If Anthropic API is unavailable, fail-fast and route to OpenAI for the same role (with degraded quality acceptable for the short period).

### 8.3 The Patient API AI Assist routing

Classifier-based:

- A small Haiku classifier judges request complexity.
- ~85% routed to Haiku.
- ~15% routed to Sonnet (form assistance, lab explanations, complex questions).

### 8.4 The Analytics Copilot routing

Per-step tier:

- Intake (classify intent): Haiku.
- SQL generation: Sonnet (occasionally Opus for very complex queries).
- Result interpretation: Haiku.

### 8.5 The routing observability

Per-feature dashboard:

- Tier distribution over time.
- Routing-error rate (eval).
- Cost contribution per tier.
- Failover events.

### 8.6 The cost impact

Pre-tier-routing (everything on Sonnet): ~$0.30 per care-coordinator request average.

Post-tier-routing: ~$0.10 per care-coordinator request average. ~66% cost reduction; quality on golden set stable.

### 8.7 The failover events

In 18 months: 4 significant Anthropic outages where the team's failover kicked in.

- 3 outages: failover to OpenAI; quality on Sonnet-equivalent tasks ~5-8% lower; acceptable degradation for the duration.
- 1 outage: failover failed (OpenAI was rate-limited because everyone failed over); brief degraded service.

Lessons:

- Failover capacity must be reserved in advance.
- Multi-provider failover (not just single fallback) reduces single-failover-point.

### 8.8 The Q3-25 routing drift

Routing accuracy degraded; classifier was sending too many to Sonnet. Cost-per-request rose 15% over 3 weeks before noticed.

Mitigation: classifier retrained with current production data; thresholds re-tuned; quarterly review cadence established.

### 8.9 What worked

- **Tier routing as primary cost lever.** Routinely 60-70% savings vs single-tier.
- **Per-feature router customization.** Each feature's tier choices reflect its specific workload.
- **Portfolio for resilience.** Failover events handled cleanly.
- **Routing observability.** Drift detected and corrected.

### 8.10 What didn't work initially

- **Single-tier launches.** First version of each feature shipped on Sonnet; tier routing added later; saved significant cost retroactively.
- **No failover testing.** First failover event was the test; partial failure resulted. Now: quarterly failover drill.
- **Router as part of feature code.** Initially each feature had its router; consolidation to a platform router (in the gateway) reduced duplication and improved observability.

---

## 9. Anti-patterns

### 9.1 "Always frontier"

Everything on the most capable model; cost 3-5× what tier routing would deliver.

**Corrective.** Tier routing per section 3; classify or rules-based.

### 9.2 "Single-vendor everything"

One provider; one model family; vendor outage takes service down.

**Corrective.** Portfolio per section 5; failover at least.

### 9.3 "Router in calling code"

Each feature has its own routing logic; consistency drifts; observability scattered.

**Corrective.** Platform router per section 6; per-feature overrides.

### 9.4 "No routing eval"

Router accuracy unknown; cost / quality silently degrades.

**Corrective.** Routing eval per section 6.3.

### 9.5 "Ensemble for routine"

Routine work runs ensembles; cost compounds with no quality justification.

**Corrective.** Ensembles for high-stakes only per section 4.5.

### 9.6 "Untested failover"

Failover code exists; never tested; fails when needed.

**Corrective.** Quarterly failover drill per section 8.10.

### 9.7 "Router never updated"

Router was built once; production patterns evolved; router degraded.

**Corrective.** Quarterly review per section 6.6.

### 9.8 "Specialised model assumed without eval"

Team adopts a specialised model (e.g., code-specific embedding) without evaluating whether it actually outperforms general on their workload.

**Corrective.** Per-workload eval per section 5.4.

---

## 10. Findings (sprint-assignable)

### ARCH-MMO-001 — Severity: High
**Finding.** Always-frontier routing; cost 3-5× achievable.
**Recommendation.** Tier routing per section 3; classifier or rules.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-MMO-002 — Severity: High
**Finding.** Single-vendor architecture; no failover; provider outage means service outage.
**Recommendation.** Portfolio per section 5; failover capability.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-MMO-003 — Severity: High
**Finding.** Router in calling code; per-feature logic; observability scattered.
**Recommendation.** Platform router per section 6.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-MMO-004 — Severity: High
**Finding.** Routing decisions not eval'd; cost and quality drift silently.
**Recommendation.** Routing eval per section 6.3.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-MMO-005 — Severity: Medium
**Finding.** Failover capability untested; first real outage will be the test.
**Recommendation.** Quarterly failover drill per section 5.3 / 6.4.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-MMO-006 — Severity: Medium
**Finding.** Per-feature routing not customised; one-size-fits-all router.
**Recommendation.** Per-feature overrides per section 6.7.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-MMO-007 — Severity: Medium
**Finding.** Ensemble used for routine work; cost compounds.
**Recommendation.** Ensemble for high-stakes only per section 4.5.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-MMO-008 — Severity: Medium
**Finding.** Router cost not tracked; surprise during cost analysis.
**Recommendation.** Router cost per request in attribution per section 6.5.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MMO-009 — Severity: Medium
**Finding.** Specialised models adopted without per-workload eval.
**Recommendation.** Eval per section 5.4 / 8.3.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MMO-010 — Severity: Medium
**Finding.** Router behaviour not observable in trace; debugging routing issues hard.
**Recommendation.** Per-request routing metadata in spans per section 6.2.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-MMO-011 — Severity: Medium
**Finding.** Capability-portfolio decision (which provider for which capability) not deliberate; team uses defaults.
**Recommendation.** Per section 5.5; capability mapping documented.
**Owner.** architecture + ai-platform-eng, sprint N+4.

### ARCH-MMO-012 — Severity: Medium
**Finding.** Quarterly portfolio review not scheduled; model landscape changes outpace adoption.
**Recommendation.** Quarterly model-eval review per section 5.4.
**Owner.** architecture + ai-platform-eng, sprint N+4.

### ARCH-MMO-013 — Severity: Medium
**Finding.** Routing-error rate not measured; classifier or rules accuracy unknown.
**Recommendation.** Per section 6.2 / 6.3.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-MMO-014 — Severity: Medium
**Finding.** Failover capacity not reserved; failover would compete with normal traffic at the secondary.
**Recommendation.** Per section 8.7; capacity planning for failover scenarios.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-MMO-015 — Severity: Low
**Finding.** LLM-as-router used where rules would suffice; per-request cost unnecessary.
**Recommendation.** Per section 2.5; default to rules.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-MMO-016 — Severity: Low
**Finding.** Per-model latency not monitored; latency-tier routing decisions not data-driven.
**Recommendation.** Per-model latency dashboards.
**Owner.** ai-platform-eng + ops, sprint N+5.

### ARCH-MMO-017 — Severity: Low
**Finding.** Model deprecation not tracked; risk that a deprecated model is in use without team knowing.
**Recommendation.** Deprecation watch per section 5.4.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-MMO-018 — Severity: Low
**Finding.** Router is single point of failure; if router service is down, no routing happens.
**Recommendation.** Router resilience per section 6.4; in-process fallback to default.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building orchestration:

- [ ] **Sprint 0 — workload analysis.** What's the request distribution; what mix of tiers makes sense?
- [ ] **Sprint 0 — router type decision.** Rules / classifier / LLM / hybrid per section 2.5.
- [ ] **Sprint 1 — router implementation.** Per section 6; platform component.
- [ ] **Sprint 1 — observability.** Per-request routing metadata.
- [ ] **Sprint 2 — failover.** At minimum, fallback to secondary provider.
- [ ] **Sprint 2 — eval.** Routing accuracy + per-tier quality.
- [ ] **Sprint 3 — portfolio expansion.** Specialised providers where justified.
- [ ] **Sprint 3 — failover drill.** Test the failover.
- [ ] **Sprint 4 — per-feature customization.** Overrides where workload demands.
- [ ] **Sprint 4 — cost dashboard.** Per-tier cost contribution.
- [ ] **Ongoing — quarterly review.** Router accuracy; tier drift; provider landscape.

For a team retrofitting:

- [ ] **Sprint 0 — audit.** What's the current model usage; what's the cost; what's the resilience.
- [ ] **Sprint 1 — tier routing first.** Highest-impact change.
- [ ] **Sprint 2 — portfolio adoption.** Add at least one secondary provider.
- [ ] **Sprint 3 — observability and eval.** Make it measurable.

A team that completes the sequence has orchestration that's cost-controlled, resilient, and tunable. A team that doesn't has either over-cost or fragile architecture.

---

## 12. References

- [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) — retrieval-side decisions that interact with model orchestration.
- [agent-topologies.md](./agent-topologies.md) — agent topology choices affect orchestration (supervisor/worker = per-role tier routing).
- [structured-output-patterns.md](./structured-output-patterns.md) — structured output works with tier routing.
- [hybrid-retrieval-patterns.md](./hybrid-retrieval-patterns.md) — retrieval patterns alongside.
- [pattern-anti-patterns.md](./pattern-anti-patterns.md) — pattern-level anti-patterns.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — the model-strategy depth on routing.
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue of approved models that the portfolio draws from.
- [guardrails-and-policy-architecture/ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md) — gateway is router's home.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — routing decisions as policy.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md) — agent-side tier routing.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — cost-side tier routing depth.
- Sibling repo: [ai-engineering-reference-architecture/eval-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/eval-engineering) — eval that supports routing decisions.
- Anthropic, OpenAI, Google, Voyage, Cohere — major LLM and embedding providers in 2026.
- LiteLLM, Portkey, Helicone — gateway / routing libraries.
