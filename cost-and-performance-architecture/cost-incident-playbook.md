# Cost Incident Playbook

> **Audience.** Architects whose cost-incident response is currently an on-call engineer's improvisation. Tech leads whose post-incident reviews keep producing "we should add another rate limit" recommendations. Anyone whose cost incidents follow the same pattern: detect, scramble, mitigate, eventually understand. **Scope.** The *architectural* playbook complementing the engineering-side runbook: architectural patterns that prevent or contain incidents; pre-incident architectural readiness; the architectural review of incident patterns; strategic decisions vs. tactical runbook. Not the engineering-side incident response (see sibling [cost-and-finops/cost-incident-runbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md)). Not the dashboard / alert design (see sibling [cost-and-finops/cost-dashboards-and-alerts.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-dashboards-and-alerts.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The engineering-side runbook covers on-call response. This document covers the architectural readiness that prevents most incidents, contains the rest, and informs the post-incident architecture review.

The engineering runbook fires when an incident has already happened. Architecture's role is upstream:

- What patterns prevent incidents?
- What architectural primitives enable fast response?
- How does the architecture evolve based on incidents?

Without architectural readiness:

- Every incident is improvisation.
- Same incident class recurs.
- Mitigation is tactical (rate-limit harder); cause is systemic.

With architectural readiness:

- Incident classes are anticipated.
- Architecture provides containment.
- Recurring patterns trigger architectural fixes.

This document is opinionated about four things:

1. **Cost incidents are predictable.** Specific patterns recur; architecture can anticipate.
2. **The architecture's job is containment, not response.** Architecture limits blast radius; response is tactical.
3. **Post-incident review must lead to architectural change.** Otherwise the same incident recurs.
4. **The cost-budget-as-circuit-breaker is the foundational primitive.** Without it, no other architectural protection holds.

Structure: (2) the incident classes architecturally; (3) pre-incident architectural primitives; (4) architectural containment patterns; (5) post-incident architectural review; (6) the architectural-changes-from-incidents log; (7) the prevention-vs-reaction balance; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The incident classes architecturally

What architecture must defend against.

### 2.1 The seven classes (architectural framing)

Per [cost-incident-runbook.md §2](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md):

1. **Runaway cost (general).** A workload misbehaves; cost balloons.
2. **Prompt bloat.** A prompt change increases per-call cost.
3. **Retrieval bloat.** Retrieval returns more / larger chunks.
4. **Agent loop.** Agent recurses or loops.
5. **Abuse / external.** Tenant misbehavior or abuse.
6. **Pricing change.** Provider raises rates.
7. **Retry storm.** Provider degradation cascades.

Each has architectural defenses.

### 2.2 The runaway-cost defense

Architectural defense:

- Per-feature cost budget with circuit-breaker.
- Per-tenant cost budget with circuit-breaker.
- Aggregate platform cost budget.

When activated: bleeding stops without human intervention.

Cross-link to [token-economics.md §6](./token-economics.md) and [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md).

### 2.3 The prompt-bloat defense

Architectural defense:

- Pre-deploy cost-impact estimate (cross-link to [system-prompt-architecture.md §6.5](../context-and-prompt-architecture/system-prompt-architecture.md)).
- Per-prompt-version cost-per-call alert.
- Auto-revert on cost spike.

### 2.4 The retrieval-bloat defense

Architectural defense:

- Retrieval-token budget per workload (cross-link to [context-and-prompt-architecture/context-window-budgeting.md §3.2](../context-and-prompt-architecture/context-window-budgeting.md)).
- Alert on retrieval allocation growth.

### 2.5 The agent-loop defense

Architectural defense:

- Per-agent-task LLM-call cap (cross-link to [ai-engineering-reference-architecture / agent-engineering / agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md)).
- Loop detection (cross-link to [ai-engineering-reference-architecture / reliability-engineering / timeout-strategy.md §7.6](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/timeout-strategy.md)).
- Per-task timeout.

### 2.6 The abuse defense

Architectural defense:

- Per-tenant cost cap (cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md §5](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)).
- Per-tenant rate-limit.
- Abuse-pattern detection.

### 2.7 The pricing-change defense

Architectural defense:

- Subscribe to provider release notes.
- Automated rate-table updates.
- Pricing-change impact assessment as part of provider relationship.

### 2.8 The retry-storm defense

Architectural defense:

- Provider-aware backoff (cross-link to [ai-engineering-reference-architecture / reliability-engineering / retry-strategy.md §5](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/retry-strategy.md)).
- Cached retry pattern (cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md §2.4](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md)).
- Fleet-wide pause on 429.

### 2.9 The defense matrix

```
Class               Primary architectural defense
─────────────────────────────────────────────────
Runaway cost        Cost-budget circuit-breaker
Prompt bloat        Pre-deploy cost estimate + alert
Retrieval bloat     Retrieval budget + alert
Agent loop          Per-task LLM-call cap
Abuse               Per-tenant cap + rate-limit
Pricing change      Provider subscription + alert
Retry storm         Provider-aware backoff + cache
```

Architecture has a defense per class.

---

## 3. Pre-incident architectural primitives

The infrastructure that makes incident response work.

### 3.1 The cost-budget-as-circuit-breaker

Primary primitive:

- Per-feature daily / monthly budget.
- Per-tenant daily / monthly budget.
- Aggregate platform budget.
- Auto-mitigate at threshold.

Without this: cost is unbounded; incidents have no containment.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md).

### 3.2 The cost-attribution telemetry

Per-call cost attributed to:

- Tenant.
- Feature.
- User.
- Model.
- Prompt version.
- Endpoint.

Cross-link to [token-economics.md §5](./token-economics.md) and [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md).

### 3.3 The kill-switch per feature

For incidents requiring complete halt:

- Feature can be disabled.
- Flag-driven.
- Tested in pre-production.

Cross-link to [ai-engineering-reference-architecture / reliability-engineering / degraded-mode-design.md §2.8](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md).

### 3.4 The per-tenant pause

For tenant-specific incidents:

- Tenant pause via single API call.
- Other tenants unaffected.
- Reversible.

### 3.5 The route-down-tier

For mitigation via cheaper model:

- Routing layer supports forced-cheaper-model mode.
- Activated per-feature.
- Quality impact understood.

Cross-link to [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md).

### 3.6 The retry-pause

For retry-storm incidents:

- Provider-wide retry pause.
- Fleet-wide signal.

Cross-link to [ai-engineering-reference-architecture / reliability-engineering / retry-strategy.md §5.2](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/retry-strategy.md).

### 3.7 The audit-log infrastructure

Per cost-bearing event:

- Logged with attribution.
- Queryable for investigation.
- Retained per compliance.

### 3.8 The "we don't have circuit-breaker; first incident is exposed" gap

Without these primitives:

- Incident detection works.
- Mitigation is improvised.
- Containment fails.

Build primitives before they're needed.

### 3.9 The primitives-implementation order

Roughly:

1. Cost attribution (visibility).
2. Per-feature budget + circuit-breaker.
3. Per-tenant budget + circuit-breaker.
4. Kill-switch per feature.
5. Per-tenant pause.
6. Retry-pause.
7. Route-down-tier.

Each builds on previous; deploy in order.

### 3.10 The primitive maintenance

Periodically:

- Test each primitive (synthetic trigger).
- Verify works as designed.
- Document operator usage.

Untested primitives may not work when needed.

---

## 4. Architectural containment patterns

How architecture limits blast radius.

### 4.1 The blast-radius dimensions

When an incident occurs, blast radius dimensions:

- **Time.** How long does the incident last?
- **Cost.** How much extra spend?
- **Affected users.** Single user, many, all?
- **Affected workloads.** Single feature, multiple, platform?
- **Customer visibility.** Internal-only? Customer-affected?

Architecture limits each.

### 4.2 The time-bounded blast

Circuit-breakers fire at threshold:

- Budget consumed at 80% → warning.
- Budget consumed at 100% → mitigation activated.
- Time-to-detection: minutes.
- Time-to-mitigation: seconds.

vs without: hours to days.

### 4.3 The cost-bounded blast

Hard budget caps:

- Daily budget = max possible cost in a day.
- Tenant runaway capped at tenant's budget.

vs without: unbounded.

### 4.4 The user-bounded blast

Per-tenant isolation:

- One tenant's incident affects only their service.
- Other tenants continue normally.

Cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 4.5 The workload-bounded blast

Per-feature isolation:

- One feature's cost issue stays in that feature.
- Other features keep working.

### 4.6 The customer-visibility containment

For some incidents:

- Customer-visible (mitigation paused workload).
- Architecture provides degraded mode (cross-link to [ai-engineering-reference-architecture / reliability-engineering / degraded-mode-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md)).

Degraded > outage.

### 4.7 The "incident escaped its workload" failure

Without isolation:

- One tenant's runaway saturates provider account.
- All tenants experience degraded service.

Per-tenant rate-limit infrastructure prevents.

### 4.8 The cross-tenant contamination case

Particularly bad:

- Tenant A's bad behavior degrades tenant B.
- Tenant B has no idea why.

Architecture must isolate at all dimensions:

- Cost.
- Rate.
- Capacity.

Cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 4.9 The cascading-impact prevention

For platform-wide incidents (provider issues):

- Per-workload degraded mode prevents user-visible cascade.
- Critical workloads protected (premium tier capacity).
- Less-critical workloads queued or shed.

### 4.10 The "blast-radius drill" pattern

Pre-production drill:

- Simulate each incident class.
- Verify containment works.
- Update architecture based on learnings.

Quarterly.

---

## 5. Post-incident architectural review

What happens after an incident.

### 5.1 The architectural-review-trigger

For every P0 or P1 cost incident:

- Engineering response review (operational).
- Architectural review (this section).

Two views; complementary.

### 5.2 The "is the architecture sufficient" question

Per incident:

- Did existing primitives engage?
- Did they contain blast radius?
- What more would have helped?

Diagnostic of architecture quality.

### 5.3 The architectural-changes-from-incidents

For each incident's architectural review:

- What change would prevent recurrence?
- What change would faster contain?
- Are these changes worth the implementation cost?

Decide.

### 5.4 The patterns-across-incidents

Multiple incidents over time:

- What pattern emerges?
- Architectural change for the pattern, not the individual incident.

### 5.5 The "the architecture worked; cost was X" outcome

Sometimes architecture worked:

- Circuit-breaker fired.
- Containment held.
- Cost was bounded.

Document; not every incident requires architectural change.

### 5.6 The "the architecture failed; we need Y" outcome

Sometimes architecture failed:

- Circuit-breaker missed.
- Containment didn't hold.

Architectural change.

### 5.7 The change-implementation lifecycle

After architectural review:

- Define the change.
- Assign owner.
- Implement.
- Test.
- Deploy.
- Verify (next similar incident catches it).

### 5.8 The "we identified the change; nobody owns it" failure

Without owner, architectural changes don't ship.

**Corrective.** Architecture review produces ticketed work; owner identified.

### 5.9 The cross-incident learning

If the same architectural gap shows in multiple incidents:

- Higher priority.
- Architectural change essential.

Pattern recognition.

### 5.10 The cost of architectural changes

Some architectural changes are expensive:

- New infrastructure.
- New eval requirements.
- Integration work.

ROI calculation:

- Expected incidents prevented × cost-per-incident vs implementation cost.

Justify implementation.

---

## 6. The architectural-changes-from-incidents log

Track architectural evolution.

### 6.1 The log structure

Per architectural change:

```yaml
change: Add per-tenant cost circuit-breaker
date_added: 2026-02-15
triggered_by_incident: Q1-2026-002 (free-tier runaway agent)
description: Per-tenant daily / monthly cost cap with circuit-breaker.
expected_benefit: Cap individual tenant cost at budget.
implementation_cost: 2 weeks engineering.
status: implemented
verification: Q3-2026-003 (similar incident; capped at $4.30)
```

Structured; auditable.

### 6.2 The change-led patterns

Over time, the log reveals:

- Recurring incident classes.
- Architectural primitives that emerged.
- Pattern of investment.

Useful for leadership.

### 6.3 The shared learning

Cross-team:

- Other AI features adopt architectural primitives that worked.
- Patterns become organization-wide.

### 6.4 The retrospective on architectural decisions

Annually:

- Look at all architectural changes from incidents.
- Are they working?
- Are some over-engineered?
- Are some still gaps?

Refine.

### 6.5 The "we keep adding rate limits" smell

If the log shows mostly "add another rate limit":

- Underlying pattern may need different architecture.
- Per-tenant rate limit, then per-feature, then per-user, then ...

At some point: a different architectural primitive (cost circuit-breaker) is the answer.

### 6.6 The maturity-curve

Early platforms: ad-hoc rate limits per incident.

Maturing: cost-budget-as-SLO.

Mature: cost-budget-as-circuit-breaker; integrated with other architectural primitives.

The progression is normal.

### 6.7 The "we built infrastructure for one incident" outcome

Sometimes architecture is built for one specific incident; never fires again.

Was it worth it?

- If preventing future incidents (of same class): yes.
- If specific incident was one-off: maybe not.

Hard to know in advance.

---

## 7. The prevention-vs-reaction balance

How much investment in each.

### 7.1 The prevention-vs-reaction trade-off

Prevention:

- Architectural primitives.
- Pre-deploy checks.
- Synthetic monitoring.

Reaction:

- Incident response.
- Alerts.
- Runbooks.

Each has cost; balance per organization.

### 7.2 The "fully preventing all incidents is impossible" reality

Some incidents unavoidable:

- Provider issues.
- Novel attack vectors.
- Capacity-pressure issues.

Reaction capability needed.

### 7.3 The 80/20 prevention

Most cost incidents fall into 7 known classes (§2). Architectural defenses against these:

- ~80% of incidents prevented or contained.
- Remaining 20% novel / hard to predict.

Invest in prevention for the known.

### 7.4 The over-engineering risk

Too much prevention:

- Architecture complexity.
- Operational burden.
- Slower feature development.

Match prevention to risk.

### 7.5 The under-engineering risk

Too little prevention:

- Repeated incidents.
- Engineering time on response.
- Customer impact.

Match prevention to risk.

### 7.6 The right balance per organization

```
Small platform (low risk):
  Prevention: minimal (cost-attribution + alerts).
  Reaction: ad-hoc.

Growing platform (medium risk):
  Prevention: per-feature budgets + circuit-breaker.
  Reaction: documented runbook.

Mature platform (high risk):
  Prevention: per-feature + per-tenant + per-user; multi-tier.
  Reaction: full incident-response practice.
```

Maturity-curve.

### 7.7 The investment-trigger

When to add architecture:

- After an incident in that class (if not prevented).
- Annually, based on volume / risk growth.

### 7.8 The "we never need it; it's still useful" architecture

Some architecture rarely fires:

- Circuit-breakers in mature systems.
- Per-tenant pause.

Still useful: when they do fire, they save significant cost.

---

## 8. Worked Meridian example

Meridian's architectural cost-incident posture.

### 8.1 The architectural primitives deployed

```
Layer                                    Status
─────────────────────────────────────────────────────
Cost attribution                         ✓ deployed
Per-feature cost budget                  ✓ deployed; 7 features
Per-feature cost circuit-breaker         ✓ deployed
Per-tenant cost budget                   ✓ deployed; 12 tenants
Per-tenant cost circuit-breaker          ✓ deployed
Aggregate platform budget                ✓ deployed
Feature kill-switch                      ✓ deployed; tested
Per-tenant pause                         ✓ deployed; tested
Route-down-tier (haiku fallback)         ✓ deployed
Retry-pause (provider 429)               ✓ deployed
Pre-deploy cost-impact estimate          ✓ deployed (Q2 2026)
Provider release-note subscription       ✓ deployed
```

Comprehensive.

### 8.2 The Q1 2026 prompt-bloat incident as architectural-review

(Cross-link to [cost-incident-runbook.md §9](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md).)

Initial response: tactical (revert deploy in 30 min).

Architectural review:

- Was there a pre-deploy cost estimate? No.
- Would it have prevented? Yes.
- Cost to implement? ~1 week engineering.
- ROI? Avoided $4-5k in this incident; expected to prevent ~quarterly recurrence.

Architectural change implemented Q2 2026: pre-deploy cost-impact estimate.

Subsequent prompt changes: ran through estimate; one was flagged; engineer reviewed.

### 8.3 The Q1 2026 free-tier runaway incident as architectural-review

Initial response: tactical (per-tenant cap fired; tenant capped at $4.30).

Architectural review:

- Did the architecture work? Yes; circuit-breaker fired.
- Could it have caught earlier? Slightly; warning at 80% fired but tenant didn't notice.
- Change: customer-visible warning email at 80%.

Implementation: Q2 2026.

### 8.4 The Q2 2026 retrieval-bloat near-miss

A retrieval pipeline change increased per-call retrieval tokens:

- Detected by retrieval-allocation alert.
- Architectural change: stricter retrieval budget alerts.

Caught early; minimal impact.

### 8.5 The architectural log

```
Q1-2026-001 (prompt bloat): 
  Change: pre-deploy cost-impact estimate
  Status: implemented Q2 2026
  Verification: caught next attempted prompt change

Q1-2026-002 (free-tier runaway):
  Change: customer warning at 80% budget
  Status: implemented Q2 2026
  Verification: subsequent runaways saw warning at 80%

Q2-2026-001 (retrieval bloat near-miss):
  Change: stricter retrieval-budget alert
  Status: implemented Q3 2026
  Verification: alert fires now at 20% drift (was 30%)
```

Auditable; pattern visible.

### 8.6 The "we have not needed kill-switch" success

The feature kill-switch primitive exists:

- Tested in pre-production quarterly.
- Not used in production in 12 months (no incident severe enough).

The architecture provides containment that mostly catches issues before kill-switch needed.

### 8.7 The customer-facing communication

Architectural decisions affecting customers:

- Per-tenant budget warnings: documented in customer portal.
- Per-tenant pause: documented; customer success notified.

Customers know what the architecture does to protect them (and them from themselves).

### 8.8 The cross-team learning

Other AI features at Meridian adopted patterns:

- Internal copilot: deployed per-feature budget.
- Analytics warehouse copilot: deployed per-feature budget + warning.

Architecture spreads.

### 8.9 The cost of the architectural investment

- Initial: ~3 months engineering across primitives.
- Ongoing: ~5% of platform team time for maintenance.
- Tests / drills: ~1 day per quarter per primitive.

Total: ~0.3-0.5 FTE equivalent.

### 8.10 The cost of avoided incidents

Estimated:

- Per-quarter cost incidents prevented: 1-2 (vs no architecture).
- Per-incident cost (if escaped): $5-50k.
- Annual avoided cost: $20-200k.

ROI strongly positive.

### 8.11 The lessons

- Architectural primitives front-load investment.
- Each primitive justifies its cost.
- Incident review feeds architecture evolution.
- Cross-team adoption multiplies value.

---

## 9. Anti-patterns

### 9.1 The "we'll add primitives after an incident" reactive posture

**Pattern.** No architectural primitives; first incident is unmitigated.

**Corrective.** Pre-incident readiness per §3.

### 9.2 The kill-switch-not-tested

**Pattern.** Primitive exists; never tested; first fire reveals broken.

**Corrective.** Quarterly drill per §3.10.

### 9.3 The "we keep adding rate limits" treadmill

**Pattern.** Each incident → more rate limits. Eventually unmanageable.

**Corrective.** Different primitive (cost budget) per §6.5.

### 9.4 The per-tenant primitive without per-feature

**Pattern.** Per-tenant cap exists; not per-feature; one feature exhausts platform.

**Corrective.** Both per §3.1.

### 9.5 The post-incident review without architectural changes

**Pattern.** Reviews held; conclusions logged; no architectural change.

**Corrective.** Owner-assigned changes per §5.7.

### 9.6 The "we have observability but no enforcement"

**Pattern.** Cost dashboards exist; no budget limits. See incidents; don't prevent.

**Corrective.** Enforcement per §3.1.

### 9.7 The cross-tenant contamination ignored

**Pattern.** One tenant's incident affects others; "we'll fix isolation later."

**Corrective.** Per-tenant isolation per §4.4 and cross-link.

### 9.8 The "architecture is engineering's problem" disengagement

**Pattern.** Architects don't engage with incidents. Architecture doesn't evolve.

**Corrective.** Architectural review per §5.

### 9.9 The "we built it for one incident" over-engineering

**Pattern.** Architecture built for specific past incident; over-engineered.

**Corrective.** Per §7.4; cost-justified.

### 9.10 The "we never have incidents; we don't need this" complacency

**Pattern.** Mature platforms have rare incidents; team relaxes investment.

**Corrective.** Maintain primitives; quarterly tests.

---

## 10. Findings (sprint-assignable)

**ARCH-CIP-001 (P0). Architectural primitives missing.** Incidents un-contained. Per §3. Owner: AI platform + engineering management.

**ARCH-CIP-002 (P0). Cost-budget circuit-breaker not deployed.** Foundational missing. Per §3.1. Owner: AI platform + FinOps.

**ARCH-CIP-003 (P0). Per-tenant primitives absent.** Cross-tenant contamination risk. Per §4.7. Owner: AI platform.

**ARCH-CIP-004 (P1). Architectural review not held post-incident.** Architecture doesn't evolve. Per §5. Owner: engineering management.

**ARCH-CIP-005 (P1). Architectural-changes log not maintained.** Pattern not visible. Per §6. Owner: AI platform.

**ARCH-CIP-006 (P1). Primitives not tested quarterly.** Untested = unreliable. Per §3.10. Owner: SRE + AI platform.

**ARCH-CIP-007 (P1). Pre-deploy cost-impact estimate absent.** Prompt changes ship blind. Per §2.3. Owner: AI platform.

**ARCH-CIP-008 (P1). Kill-switch per feature absent.** No nuclear option. Per §3.3. Owner: AI platform.

**ARCH-CIP-009 (P2). Per-tenant pause not implemented.** Tenant incidents require coarse-grained mitigation. Per §3.4. Owner: AI platform.

**ARCH-CIP-010 (P2). Route-down-tier not implemented.** Cheaper-model fallback unavailable. Per §3.5. Owner: AI platform.

**ARCH-CIP-011 (P2). Provider release-notes not subscribed.** Pricing changes are surprises. Per §2.7. Owner: AI platform + FinOps.

**ARCH-CIP-012 (P2). Retry-pause not implemented.** Retry storms during provider issues. Per §3.6. Owner: AI platform.

**ARCH-CIP-013 (P2). Quarterly blast-radius drill absent.** Containment unverified. Per §4.10. Owner: SRE + AI platform.

**ARCH-CIP-014 (P2). Architectural changes lack owner.** Don't ship. Per §5.8. Owner: engineering management.

**ARCH-CIP-015 (P3). Cross-team learning from incidents absent.** Patterns rediscovered. Per §6.3. Owner: engineering management.

**ARCH-CIP-016 (P3). Annual architectural retrospective absent.** Over-engineered or under-engineered drift. Per §6.4. Owner: engineering management.

**ARCH-CIP-017 (P3). Customer-facing communication of architectural protections absent.** Customers unaware. Per §8.7. Owner: product.

**ARCH-CIP-018 (P3). Cost of avoided incidents not tracked.** ROI of primitives invisible. Per §8.10. Owner: FinOps + engineering management.

---

## 11. Adoption sequencing checklist

- [ ] **Build cost-budget circuit-breaker per §3.1.**
- [ ] **Deploy per-feature primitives (§3).**
- [ ] **Deploy per-tenant primitives (§3).**
- [ ] **Deploy aggregate platform budget (§3).**
- [ ] **Build kill-switch per feature (§3.3).**
- [ ] **Build per-tenant pause (§3.4).**
- [ ] **Build route-down-tier (§3.5).**
- [ ] **Pre-deploy cost-impact estimate (§2.3).**
- [ ] **Provider release-note subscription (§2.7).**
- [ ] **Architectural review per incident (§5).**
- [ ] **Maintain architectural-changes log (§6).**
- [ ] **Quarterly blast-radius drill (§4.10).**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — cost framework underlying.
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — latency dimension.
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — capacity dimension.
- [caching-tiers.md](./caching-tiers.md) — caching as cost lever.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) — self-host cost considerations.

**Elsewhere in this repo.**
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — route-down-tier.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant isolation.
- [context-and-prompt-architecture/system-prompt-architecture.md](../context-and-prompt-architecture/system-prompt-architecture.md) — pre-deploy cost estimate.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / cost-incident-runbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md) — engineering response (companion).
- [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) — primitive implementation.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — attribution.
- [ai-engineering-reference-architecture / cost-and-finops / cost-dashboards-and-alerts.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-dashboards-and-alerts.md) — alerts.
- [ai-engineering-reference-architecture / reliability-engineering / incident-response-for-ai.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/incident-response-for-ai.md) — broader incident framework.

**External.**
- Google SRE Book — incident response practices.
- Blameless post-mortem literature.
