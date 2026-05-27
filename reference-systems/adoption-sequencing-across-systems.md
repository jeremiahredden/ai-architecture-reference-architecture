# Reference: Adoption Sequencing Across Systems

> **Status.** Reference architecture, current as of 2026-Q2 review. Built from real systems and anonymized; the client (Meridian Health) is a fictional regulated healthcare SaaS used across the sibling reference architecture repos. **Audience.** Engineering leadership and architects deciding the order in which AI systems land. Anyone whose AI roadmap has three systems in parallel and capacity for one at a time. **Cross-links.** The three reference systems: [Care Coordinator](./meridian-care-coordinator.md), [Patient API AI Assist](./patient-api-ai-assist.md), [Analytics Warehouse Copilot](./analytics-warehouse-copilot.md). Platform-investment patterns in [reference-patterns/](../reference-patterns/) and the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture).

---

## 1. Why this document exists

The three Meridian reference systems represent meaningfully different AI architectures. A team approaching them in parallel discovers, painfully, that:

- The platform investment (AI gateway, eval platform, model catalogue, observability, cost attribution, retrieval client, agent runner) is substantial.
- Each system shares ~70% of the platform with the others; building three systems in parallel triples the platform work.
- Each system's risk profile is different; the team's operational maturity needs to grow with the higher-risk systems, not be tested by them first.
- Capacity (engineering, AI platform, reviewers, on-call) is finite.

The right sequencing isn't "three in parallel"; it's a deliberate order that builds platform maturity and operational confidence before tackling the higher-risk systems. The order this document recommends — analytics copilot first, patient API AI Assist second, Care Coordinator third — reflects the risk profile, the platform-investment graph, and the team's learning curve.

A team that lands them in parallel will:

- Ship a buggy Care Coordinator (it's the highest-risk, and platform immaturity hits hardest).
- Build duplicate platform components (each team builds their own gateway / eval / observability before the platform catches up).
- Experience compounding incident risk (each system introduces failures; the team's incident-response capacity is overwhelmed).

A team that lands them sequentially will:

- Build the platform once; subsequent systems inherit it.
- Test the team's operational discipline on the lower-risk system; learn before the high-risk one.
- Stabilise each system before the next launches.

This document is opinionated about four things:

1. **The order is risk-ascending.** Lowest-risk first; highest-risk last. The team's first AI system shouldn't be the one where mistakes hurt patients.
2. **The platform is the shared substrate.** Investments in the gateway, eval, observability, and cost attribution pay back across all systems. Building them deliberately for the first system pays off for the second and third.
3. **The operational discipline is built incrementally.** On-call practice, incident response, eval cadence, and cost discipline all need development. Build them on the lower-risk system.
4. **The sequencing is not negotiable, even if pressure pushes for parallel.** "Just give it to a different team" doesn't solve the platform overlap; it makes it worse.

Structure: (2) the three systems by risk profile; (3) the platform-investment graph; (4) the operational-discipline curve; (5) the recommended sequence; (6) the "parallel" anti-pattern; (7) capacity sizing across the sequence; (8) findings (`ARCH-ADOPT-001` through `ARCH-ADOPT-018`); (9) sprint sequencing across the full multi-year program.

---

## 2. The three systems by risk profile

The three reference systems compared.

### 2.1 Comparison matrix

| Dimension | Care Coordinator | Patient API AI Assist | Analytics Copilot |
| --- | --- | --- | --- |
| Audience | Clinical staff | Patients | Analysts |
| Volume | 3k/day | 250k/day | 1.2k/day |
| Per-interaction cost | $0.18 | $0.012 | $0.14 |
| Latency budget | 4-12s | 0.8-2s | 12-30s |
| Architecture shape | Hybrid agent | Single call | Hybrid agent |
| Real-world side effects | Yes (orders, messages, scheduling) | No (read + form-fill suggestions only) | No (read-only SQL) |
| Risk if AI errs | High (clinical) | Medium (patient-facing) | Medium (incorrect analysis) |
| Re-identification risk | Low (clinical staff has access) | Low (patient sees own data) | High (de-id boundary) |
| Compliance burden | HIPAA + clinical workflow regs | HIPAA + patient-safety | HIPAA Safe Harbor enforcement |
| HITL requirement | Heavy (orders, care plan changes) | Light (escalation only) | Medium (re-id queries) |
| Multi-tenancy | Per-hospital | Per-hospital | Per-tenant + per-analyst scope |
| Platform components used | Most | Many | Some |

### 2.2 The risk ranking

Ranking by "if the AI errs, what's the consequence":

1. **Lowest:** Analytics Copilot. Bad SQL → analyst notices → fixes. Bad result → analyst interprets in context. Re-id risk gated by HITL. The blast radius is contained.

2. **Medium:** Patient API AI Assist. Bad information → patient confusion or call to provider. Refusal patterns + scope limit catch the worst cases. Patient-safety risk bounded by strict scope.

3. **Highest:** Care Coordinator. Bad clinical reasoning → wrong order proposed → if HITL fails or doesn't catch it, patient harm. The HITL is the safety net; HITL maturity matters.

The risk ranking informs the sequencing. The team's first system shouldn't be the riskiest.

### 2.3 The operational-burden ranking

Ranking by ongoing operational cost (excluding LLM cost):

1. **Lowest:** Analytics Copilot. Low volume; analysts tolerate higher latency; HITL only for re-id (manageable volume).

2. **Medium:** Care Coordinator. Moderate volume; HITL for orders (significant reviewer capacity); on-call burden non-trivial.

3. **Highest:** Patient API AI Assist. High volume (250k/day); 24/7 patient-facing (cannot have multi-hour outage); per-tenant cost monitoring across 240 hospitals.

The operational-burden ranking does not align with the risk ranking. The Patient API has high operational burden but moderate risk; the Care Coordinator has moderate burden but high risk.

The sequence accounts for both.

### 2.4 The platform-overlap ranking

Ranking by "how much of the platform does this system need":

1. **Smallest overlap:** Analytics Copilot. Specific components (sandbox, semantic-layer retrieval) it doesn't share. ~50% platform overlap with the others.

2. **Medium overlap:** Patient API AI Assist. Uses the standard gateway, eval, retrieval client. ~75% overlap.

3. **Largest overlap:** Care Coordinator. Uses everything the others use + agent runner + tool registry + complex HITL queues. ~95% of the platform.

Building the Care Coordinator first would require building most of the platform. Building Analytics Copilot first builds a smaller platform that's still useful for the next two systems.

---

## 3. The platform-investment graph

What the platform contains and which systems use what.

### 3.1 The platform components

| Component | Care Coordinator | Patient API | Analytics Copilot |
| --- | --- | --- | --- |
| AI gateway (auth, attribution, moderation, rate limit) | ✓ | ✓ | ✓ |
| Eval platform (gates, online eval) | ✓ | ✓ | ✓ |
| Observability (traces, dashboards) | ✓ | ✓ | ✓ |
| Cost attribution + dashboards | ✓ | ✓ | ✓ |
| Model catalogue + version pinning | ✓ | ✓ | ✓ |
| Retrieval client (with scope enforcement) | ✓ | ✓ | ✓ (schema retrieval) |
| Embedding pipeline | ✓ | ✓ | ✓ (less critical; schema mostly) |
| Hybrid (lexical + vector) infrastructure | ✓ | ✓ | (less; mostly lexical/structural) |
| Agent runner | ✓ | (no agent loop) | ✓ (inner agent step) |
| Tool registry + authorization | ✓ | (no tools) | (limited; sandbox only) |
| HITL queue infrastructure | ✓ (large queue) | ✓ (small queue) | ✓ (small queue, compliance) |
| Refusal / escalation templates | ✓ | ✓ | ✓ |
| Per-tenant policy infrastructure | ✓ | ✓ | ✓ |
| Sandbox execution service | (not applicable) | (not applicable) | ✓ (unique to copilot) |
| Knowledge graph (drug interactions) | ✓ | (not applicable) | (not applicable) |

### 3.2 The platform that the first system must build

For the first system landed, the platform components it needs become the first investment. The team's incentive is to land the smallest-platform system first, so the platform investment is sized to what's needed (and grows for subsequent systems).

**If Analytics Copilot is first:** team builds AI gateway, eval platform, observability, cost attribution, model catalogue, retrieval client (basic), agent runner (for the inner step), small HITL queue, sandbox service. Components specific to Analytics: schema retrieval, sandbox.

**If Patient API is first:** team builds AI gateway, eval platform, observability, cost attribution, model catalogue, retrieval client (full hybrid), embedding pipeline, refusal/escalation templates, small HITL queue, per-tenant policy. Components specific to Patient API: per-hospital content pipeline.

**If Care Coordinator is first:** team builds AI gateway, eval platform, observability, cost attribution, model catalogue, retrieval client (full hybrid), embedding pipeline, agent runner, tool registry, large HITL queue, knowledge graph, complex per-tenant policy. The entire platform, more or less.

### 3.3 The accumulation pattern

The platform grows as systems land. Subsequent systems benefit:

- **System 1 lands.** Builds its required platform components.
- **System 2 lands.** Reuses what System 1 built; adds System-2-specific components.
- **System 3 lands.** Reuses everything; adds the last specific components.

If the order is Analytics → Patient API → Care Coordinator:

- After Analytics: gateway, eval, observability, cost, retrieval (basic), agent runner (basic), HITL (basic), sandbox.
- After Patient API: + hybrid retrieval, embedding pipeline (full), refusal/escalation templates, per-tenant content pipeline.
- After Care Coordinator: + tool registry, large HITL infrastructure, knowledge graph, complex policy.

By the time Care Coordinator launches, 80%+ of its platform requirements are already in place.

### 3.4 The "duplicate platform" cost

Building systems in parallel typically produces duplicate platform components. Three teams each build their own:

- Gateway (3 gateways with diverging behaviour).
- Eval pipeline (3 evals, hard to compare).
- Observability (3 trace formats).
- Cost attribution (3 dashboards).

The cost is engineering time × 3 + ongoing maintenance × 3 + the cost of consolidation later (the "we have to merge these" project that takes 2 quarters).

Sequential development with a single platform team avoids this entirely.

### 3.5 The platform-team sizing

For a single system: 2-3 platform engineers. For a sequenced 3-system program: 3-5 platform engineers (with the load growing as systems land).

The platform team supports all feature teams; sized accordingly.

### 3.6 The "platform vs feature" investment ratio

For early systems: platform investment is large (~60% of effort).

For later systems: platform investment is small (~20% of effort) because most platform exists.

A team that ships the first system at 60/40 platform-to-feature ratio is making the right investment; the same ratio for the third system would be wrong (the platform is already built).

---

## 4. The operational-discipline curve

The team's operational maturity needs to grow with the system risk.

### 4.1 The disciplines required

For any production AI system:

- **Cost discipline.** Budgets, attribution, alerts, runbook (sub-15-min MTTM).
- **Eval discipline.** Eval gate, continuous eval, production-trace promotion.
- **Observability discipline.** Trace coverage, dashboards, loop-aware alerts, runbook integration.
- **Incident response discipline.** Runbook, on-call rotation, post-incident reviews.
- **Privacy discipline.** PHI redaction, retention, deletion, audit.
- **Quality discipline.** Per-feature SLO; degradation responses; refusal patterns.

### 4.2 The discipline maturity curve

A team starts at low maturity on each. Each production system raises the bar:

| Discipline | After System 1 (Analytics) | After System 2 (Patient API) | After System 3 (Care Coord) |
| --- | --- | --- | --- |
| Cost | Basic attribution + per-feature cap | Per-tenant caps + multiple alerts | Sub-15-min MTTM proven |
| Eval | Gate present | Continuous eval added | Per-trajectory eval for agent |
| Observability | Traces + basic dashboards | Per-tenant dashboards + alerts | Trajectory observability + cross-system view |
| Incident response | Runbook exists; first incidents handled | Multiple incident types handled; runbook mature | Sub-30-min MTTM on critical incidents |
| Privacy | PHI redaction in traces; deletion supported | Per-tenant deletion + accounting-of-disclosures | Cross-system PHI flow audited |
| Quality | First SLOs defined; eval gate operational | Refusal patterns proven | Multi-layer guardrails proven |

Each system both *uses* the maturity built so far AND *develops it further*. The third system isn't asking for foundational maturity; it's asking for advanced maturity that earlier systems built.

### 4.3 The "first system as platform proving ground"

The first system is partly chosen for its value as a learning ground. Lower-risk + meaningful-value + uses much of the platform = good first system.

Analytics Copilot fits: meaningful value (analyst productivity); lower risk (analyst-mediated; read-only); uses much of the platform (gateway, eval, observability); has the unique sandbox component the team learns from.

### 4.4 The "first cost incident is the educator"

Every team learns cost discipline through their first cost incident. The first incident on a low-risk, lower-volume system is much cheaper than the first incident on a high-volume patient-facing system.

If the first system is Patient API, the first cost incident could be $50k-100k+ (high volume × runaway cost). If the first system is Analytics, the first cost incident is more likely $1k-5k (bounded volume).

The cost of the lesson is part of the sequencing argument.

### 4.5 The "first incident response trains on-call"

The on-call rotation handles incidents on the first system. The team learns:

- What patterns of incident exist for AI features.
- How to triage.
- How to mitigate.
- How to communicate.

Each subsequent system inherits the trained on-call practice.

### 4.6 The "rubber-stamp risk on HITL"

HITL discipline (per [content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md) and [integration-architecture/human-in-the-loop-boundaries.md](../integration-architecture/human-in-the-loop-boundaries.md)) is a maturity. A team that hasn't built HITL discipline shouldn't ship Care Coordinator (where HITL is the safety net for clinical-side-effect actions).

The first system with HITL (Analytics, with compliance HITL for re-id queries) builds the discipline at lower stakes; the second (Patient API, with light escalation HITL) extends it; the third (Care Coordinator, with high-stakes order-approval HITL) requires it to be mature.

---

## 5. The recommended sequence

### 5.1 The sequence

1. **Analytics Warehouse Copilot** (Quarters 1-2).
2. **Patient API AI Assist** (Quarters 3-5).
3. **Care Coordinator** (Quarters 4-7, with overlap to Patient API).

### 5.2 Why this order

**Analytics first.**

- Lowest risk (analyst-mediated; read-only; bounded blast radius).
- Bounded volume (1.2k/day; cost incidents bounded).
- HITL discipline development at lower stakes (compliance reviewer for re-id; manageable volume).
- Builds ~50% of the platform that subsequent systems will use.
- Demonstrates value to leadership and customers (internal analytics first; customer-analyst preview second).

**Patient API second.**

- Reuses the platform Analytics built.
- Adds high-volume operational discipline (caches, cost monitoring at scale, 24/7 readiness).
- Lower per-interaction risk than Care Coordinator.
- Builds out hybrid retrieval and refusal/escalation patterns.
- Builds 24/7 on-call readiness before the highest-stakes system.

**Care Coordinator third.**

- Inherits substantial platform.
- Inherits operational discipline.
- Adds agent-specific platform (tool registry, agent runner enhancements).
- Adds high-stakes HITL infrastructure.
- The team is ready for the risk by this point.

### 5.3 The overlap window

Care Coordinator can begin in Q4 (overlapping Patient API's GA) because:

- Platform-wise: gateway, eval, observability, cost, refusal/escalation are all in place.
- Operationally: on-call is mature; cost discipline is proven; incident response is practised.
- Care Coordinator's unique components (tool registry, agent runner enhancements) are the focused investment.

### 5.4 The "what if priorities push differently" pressure

Leadership may want Care Coordinator first (the most clinically valuable; the most strategic). The pushback:

- Lower-risk first builds the platform that Care Coordinator needs.
- Care-Coordinator-first has 40-60% higher risk of a launch-window incident.
- The platform investment is roughly the same total cost; the sequencing affects when each system actually ships, not the total investment.

The case is "we ship all three faster by sequencing than by parallel; we ship them safer; we ship them at lower platform cost."

### 5.5 The "what about market windows" pressure

Sometimes external pressure (competitor launches; customer demand) forces a system to land earlier.

If Care Coordinator must launch first:

- Plan for higher incident risk; staff incident-response capacity accordingly.
- Plan for platform cost approximately 2x what sequenced would have been.
- Accept a longer ramp to operational confidence.

It's possible; it's just expensive.

### 5.6 The "different teams in parallel" alternative

What if each system has a different team building it in parallel? The platform overlap problem remains:

- Three teams build three gateways (or worse, all use one gateway built by a fourth team that's overwhelmed).
- Each team's eval pipeline diverges.
- Cost dashboards are duplicated.

A platform team supporting all three feature teams is the standard answer; the sequencing then determines the platform team's roadmap.

---

## 6. The "parallel" anti-pattern

Why parallel development of the three systems usually fails.

### 6.1 The platform-team overload

Three teams asking for the same platform features at the same time:

- Gateway: each team wants their feature.
- Eval: each team wants their evaluations.
- Observability: each team wants their dashboards.

The platform team is either:

- Overwhelmed (slow delivery; teams build workarounds; technical debt).
- Triaging (one team gets first-class platform; others get hacks).
- Duplicated (each feature team builds their own version of platform components).

None of these are good.

### 6.2 The duplicate-component proliferation

Each feature team builds its own:

- Gateway wrapper.
- Eval pipeline.
- Observability integration.
- Cost dashboard.

Six months later, the team has three of each, all slightly different. Consolidation is a 6-12 month project. The lesson: don't.

### 6.3 The "everything is in progress, nothing is in production" failure

Parallel development across three systems means three systems in beta state. None is at production maturity:

- Each lacks the runbook that production demands.
- Each lacks the eval coverage.
- Each lacks the observability.

Operationally, you have three half-products instead of one finished one.

### 6.4 The "incident response across three systems simultaneously"

When the first major incident happens, the on-call engineer may face:

- An incident on System A.
- An incident on System B (correlated to a shared platform issue).
- An incident on System C (also correlated).

The on-call's ability to handle three simultaneous incidents is a function of practice. A team that's been operating one system for six months can handle three incidents. A team that just launched three systems can't.

### 6.5 The "platform debt at launch"

Parallel development tends to ship platform debt:

- Tactical hacks because deliberate platform wasn't ready.
- Bypass patterns that defeat the gateway's purpose.
- Per-team logging that doesn't aggregate.

Each tactical decision compounds. By the time the team realises, the cost of fixing is high.

### 6.6 The honest assessment

Sequencing isn't a preference; it's a constraint. Parallel works only when:

- The team has 3× normal platform-team capacity (rare).
- The three systems are truly independent in platform requirements (this isn't the case here).
- The risk profile of parallel failures is acceptable (rare).

For Meridian's three systems, none of these hold; sequencing is the right architecture.

---

## 7. Capacity sizing across the sequence

The capacity needed at each phase.

### 7.1 Phase 1 (Quarters 1-2): Analytics first

- AI platform team: 3 engineers (2 building gateway + eval; 1 supporting the analytics team).
- Analytics feature team: 4-5 engineers.
- Compliance reviewer (for re-id HITL): 0.5 FTE.
- Total: ~8 FTE.

### 7.2 Phase 2 (Quarters 3-5): Patient API as primary; Analytics in maintenance

- AI platform team: 4 engineers (gateway + eval + observability + retrieval enhancements).
- Patient API feature team: 5-6 engineers.
- Analytics feature team: 2 engineers (maintenance + iterations).
- Care managers / patient-facing reviewers (escalation HITL): 1 FTE.
- Compliance reviewer: 0.5 FTE.
- Total: ~13 FTE.

### 7.3 Phase 3 (Quarters 4-7): Care Coordinator launching; Patient API in scale-up; Analytics in maintenance

- AI platform team: 5 engineers (agent runner, tool registry, advanced observability).
- Care Coordinator feature team: 6-8 engineers.
- Patient API feature team: 3-4 engineers (scale-up + iterations).
- Analytics feature team: 2 engineers.
- Care managers (Care Coordinator HITL — orders, care plans): 18 FTE at GA.
- Patient-facing reviewers: 1-2 FTE.
- Compliance reviewer: 0.5-1 FTE.
- Total: ~38-44 FTE.

### 7.4 The capacity-growth shape

The team grows as the program advances:

- Phase 1: 8 FTE.
- Phase 2: 13 FTE.
- Phase 3: 38-44 FTE.

The reviewer team's growth (especially care managers) is the largest. Plan for hiring lead time; reviewers must be trained before they can handle production load.

### 7.5 The "we can't hire that fast" reality

If the team can't grow as the sequence requires:

- Phase 3 (Care Coordinator launch) must be deferred until reviewer capacity is in place.
- Alternative: launch Care Coordinator with reduced scope (single hospital pilot for 6 months while reviewer team grows).

The hiring lead time is part of the sequence planning.

### 7.6 The cross-system on-call

On-call covers all three systems simultaneously (in Phase 3):

- Single on-call rotation across all AI features.
- Runbooks per system.
- Cross-system observability dashboards.
- Per-system mitigation procedures.

A common on-call practice is cheaper than per-system rotations and trains the team across systems.

---

## 8. Findings (sprint-assignable)

### ARCH-ADOPT-001 — Severity: Critical
**Finding.** Three-system parallel development underway without platform-team capacity to support; risk of platform debt and duplicate components.
**Recommendation.** Re-sequence per section 5; defer two systems while first is built deliberately.
**Owner.** engineering-leadership + architecture, sprint N+1.

### ARCH-ADOPT-002 — Severity: Critical
**Finding.** Care Coordinator scheduled first; risk profile too high for team's current operational maturity.
**Recommendation.** Re-sequence to Analytics or Patient API first; Care Coordinator after platform and operational maturity in place.
**Owner.** engineering-leadership, sprint N+1.

### ARCH-ADOPT-003 — Severity: Critical
**Finding.** No platform team exists; each feature team building its own gateway / eval / observability.
**Recommendation.** Establish platform team; assign ownership of shared components per section 3.5.
**Owner.** engineering-leadership, sprint N+1.

### ARCH-ADOPT-004 — Severity: High
**Finding.** Reviewer-capacity sizing not done for Care Coordinator's HITL requirements; can't ship without 18 care managers in place.
**Recommendation.** Hiring plan per section 7.4; lead time accounted in launch date.
**Owner.** HR + product + engineering-leadership, sprint N+2.

### ARCH-ADOPT-005 — Severity: High
**Finding.** Cross-system observability not planned; cross-system incidents will be hard to diagnose.
**Recommendation.** Unified observability stack per [observability-and-telemetry/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/observability-and-telemetry) in the engineering sibling.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-ADOPT-006 — Severity: High
**Finding.** Each system's platform-component requirements not documented; risk of duplicate-build.
**Recommendation.** Per-system platform-dependency map per section 3.1; shared roadmap.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-ADOPT-007 — Severity: High
**Finding.** First-system choice not made based on platform / risk analysis.
**Recommendation.** Analytics Copilot first per section 5.2; document rationale.
**Owner.** architecture + engineering-leadership, sprint N+2.

### ARCH-ADOPT-008 — Severity: Medium
**Finding.** Platform-component reuse not tracked; risk that subsequent systems re-implement rather than reuse.
**Recommendation.** Platform-component catalog; per-system reuse declaration in design review.
**Owner.** architecture + ai-platform-eng, sprint N+3.

### ARCH-ADOPT-009 — Severity: Medium
**Finding.** On-call rotation not unified across AI features; per-system rotations being staffed.
**Recommendation.** Single AI on-call rotation per section 7.6.
**Owner.** ops + ai-platform-eng, sprint N+3.

### ARCH-ADOPT-010 — Severity: Medium
**Finding.** Eval gates not standardised across systems; each system using different framework.
**Recommendation.** Single eval framework per [eval-engineering/eval-gate-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/eval-gate-architecture.md).
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-ADOPT-011 — Severity: Medium
**Finding.** Cost dashboard per system rather than per-tenant aggregated; multi-system tenant view absent.
**Recommendation.** Cross-system per-tenant view per [cost-and-finops/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/cost-and-finops) in engineering sibling.
**Owner.** ai-platform-eng + finance, sprint N+3.

### ARCH-ADOPT-012 — Severity: Medium
**Finding.** First system's lessons not captured for benefit of subsequent systems.
**Recommendation.** Post-launch retrospective per system; outputs inform subsequent.
**Owner.** engineering-leadership + ai-platform-eng, sprint N+3.

### ARCH-ADOPT-013 — Severity: Medium
**Finding.** Capacity-growth plan not aligned with hiring lead time; rapid growth may not be feasible.
**Recommendation.** Phased hiring per section 7.4; revise launch dates accordingly.
**Owner.** HR + engineering-leadership, sprint N+4.

### ARCH-ADOPT-014 — Severity: Medium
**Finding.** Cross-system platform changes (e.g., gateway upgrade) coordination informal.
**Recommendation.** Quarterly platform-roadmap review with all consumer teams.
**Owner.** ai-platform-eng + architecture, sprint N+4.

### ARCH-ADOPT-015 — Severity: Medium
**Finding.** Per-system runbooks duplicate platform-level guidance.
**Recommendation.** Layered runbook structure: platform runbooks + per-system overlays.
**Owner.** ops + ai-platform-eng, sprint N+4.

### ARCH-ADOPT-016 — Severity: Low
**Finding.** Annual portfolio-level review of AI systems not scheduled; strategic alignment opportunity missed.
**Recommendation.** Annual review per section 1; engineering-leadership cadence.
**Owner.** engineering-leadership, sprint N+5.

### ARCH-ADOPT-017 — Severity: Low
**Finding.** Customer-facing communication about the multi-system program not coordinated.
**Recommendation.** Single program narrative; customer briefings aligned.
**Owner.** product + customer-success, sprint N+5.

### ARCH-ADOPT-018 — Severity: Low
**Finding.** Cost-modeling across all three systems not consolidated; total program cost surprises possible.
**Recommendation.** Multi-year cost model per section 7; finance partnership.
**Owner.** finance + engineering-leadership, sprint N+5.

---

## 9. Sprint sequencing across the multi-year program

### 9.1 Year 1 (Quarters 1-4)

**Q1 — Platform foundation + Analytics Copilot launch.**
- Platform team builds gateway, eval, observability, cost attribution, model catalogue.
- Analytics Copilot feature team builds the copilot's specific components.
- End of Q1: Analytics Copilot pilot with internal Meridian analysts.

**Q2 — Analytics maturity + Patient API foundation.**
- Analytics Copilot iterates; first customer-analyst pilots.
- Patient API team starts; reuses platform; adds hybrid retrieval, refusal templates.
- End of Q2: Patient API in dev pilot.

**Q3 — Patient API pilot + Care Coordinator design.**
- Patient API in customer pilot at 10 hospitals.
- Analytics Copilot GA (all opted-in customers).
- Care Coordinator design begins; agent runner specifications.
- End of Q3: 1 system at GA; 1 in pilot; 1 in design.

**Q4 — Patient API expansion + Care Coordinator foundation.**
- Patient API expanding to 50 hospitals.
- Care Coordinator team starts; reuses platform; adds tool registry, agent runner, complex HITL.
- End of Q4: 1 system at GA; 1 in expansion; 1 in early development.

### 9.2 Year 2 (Quarters 5-8)

**Q5 — Patient API GA + Care Coordinator pilot.**
- Patient API GA across all 240 hospitals.
- Care Coordinator pilot at 3 hospitals (the GA-target hospitals).
- End of Q5: 2 systems at GA; 1 in pilot.

**Q6-Q7 — Care Coordinator iteration and expansion.**
- Care Coordinator iterates; expands to 30 hospitals.
- Cross-system observability matures.
- End of Q7: 2 systems at GA; 1 in expanded pilot.

**Q8 — Care Coordinator GA.**
- Care Coordinator GA across all 240 hospitals.
- All 3 systems in production.

### 9.3 Year 3 onward

- Platform iteration as needed.
- Per-system iteration based on production data.
- New AI features added (they inherit the mature platform).

### 9.4 The "what could go wrong" reality

The plan assumes:

- Platform team is adequately resourced.
- Reviewers can be hired on schedule.
- Each system's pilot reveals fixable issues, not architectural failures.
- No major incident causes a halt.

Any of these can derail. The plan needs quarterly review with willingness to slip launches rather than skip discipline.

### 9.5 The "if we're behind" decision tree

If at any quarter the plan is behind:

- **Behind on platform?** Don't launch next system without it.
- **Behind on reviewers?** Defer the launch that depends on them.
- **Behind on eval?** Don't launch without eval gate.
- **Behind on observability?** Limit blast radius (pilot only).

The discipline preserves quality at the cost of timing. The alternative — launching with discipline gaps — produces incidents that cost more than the delay would have.
