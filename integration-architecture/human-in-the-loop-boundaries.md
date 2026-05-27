# Human-in-the-Loop Boundaries

> **Audience.** Architects designing where the human sits relative to the AI in a workflow. Tech leads whose HITL was added as an afterthought and has become a friction layer. Product teams whose AI features hit the "humans can't keep up" wall. **Scope.** The *architectural* decision: which HITL pattern fits which feature, the UX and latency implications, the operational shape, and the failure modes (notably the rubber-stamp problem). Not the implementation of the review UI (engineering details). Not the moderation HITL (see [guardrails-and-policy-architecture/content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md)). Not the refusal/escalation flow (see [guardrails-and-policy-architecture/refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Human-in-the-loop is positioned as the safety net for AI systems: when the AI is uncertain, when the stakes are high, when the action is irreversible, a human is in the workflow. The framing is correct; the architectural execution is often wrong.

The common failure modes:

- **HITL added as an afterthought.** The AI workflow is designed; HITL is bolted on; the UX is rough; the human's role is unclear; throughput is wrong-sized.
- **HITL as friction.** Every AI output goes through a human; users wait; humans burn out; the AI's value is muted.
- **HITL as rubber stamp.** Human approval rate is 99%; the human isn't adding value; the safety net isn't actually catching things.
- **HITL with no humans.** The architecture assumes humans exist for review; the team didn't budget the reviewer FTE; queues back up; SLA violations cascade.

The architectural decision is which HITL pattern fits the workflow. Four primary patterns: approver before action; reviewer after action; escalation on uncertainty; always-on copilot. Each has different UX, latency, throughput, and reviewer-design implications. Choosing the wrong pattern produces predictable failure modes; choosing the right one earns the value HITL is supposed to deliver.

This document is opinionated about four things:

1. **HITL is an architectural decision, not a control to add later.** The pattern shapes the workflow; designing the workflow first and adding HITL later produces friction.
2. **The four patterns have distinct fits.** Match the pattern to the workflow's risk profile, latency tolerance, and reviewer capacity.
3. **Reviewer capacity is part of the architecture.** A pattern that requires more reviewers than you have is a broken pattern; right-size before deploying.
4. **The rubber-stamp failure has architectural correctives.** Reduce volume routed to humans (auto-approve high-confidence); design the reviewer's interface for thoughtful work; measure rejection rate.

Structure: (2) the four HITL patterns; (3) the decision criteria; (4) reviewer capacity sizing; (5) the rubber-stamp failure and correctives; (6) UX implications per pattern; (7) operational considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four HITL patterns

The architectural shapes.

### 2.1 Pattern A — Approver before action

The AI proposes; the human approves before the action commits.

```
User request → AI generates proposal → 
  human reviews → approve / modify / reject →
  if approved: → action executes →
  user receives outcome
```

**Examples.**

- Clinical agent suggests medication order → clinician approves → order is placed.
- Marketing agent drafts email → manager approves → email is sent.
- Finance agent proposes refund → approver signs off → refund processed.

**Properties.**

- Action is gated; doesn't happen without human approval.
- High-stakes / irreversible actions are safe.
- User-facing latency: must wait for human approval (minutes to days).
- Reviewer load: 1 per proposal.

### 2.2 Pattern B — Reviewer after action

The AI takes the action; the human reviews after for quality / correctness.

```
User request → AI generates response / takes action → 
  user receives outcome →
  human review (async; may approve, edit, escalate)
```

**Examples.**

- Customer-support AI sends response → user gets immediate reply → reviewer audits a sample after.
- Code-generation copilot suggests code → developer accepts → senior dev reviews the diff later.
- Content-generation drafts article → published with disclaimer → editor reviews for follow-up.

**Properties.**

- User gets immediate response.
- Quality is checked; some bad responses ship before being caught.
- Lower review burden (often a sample, not 100%).
- Less appropriate for irreversible / high-stakes actions.

### 2.3 Pattern C — Escalation on uncertainty

The AI handles the simple cases; routes uncertain cases to humans.

```
User request → AI evaluates →
  confident: → AI handles, user receives outcome
  uncertain: → escalate to human → human handles → user receives outcome
```

**Examples.**

- IT-support AI handles password resets directly; routes complex tickets to human.
- Customer-service AI handles common questions; escalates account-specific issues.
- Medical triage AI handles routine intake; escalates concerning symptoms.

**Properties.**

- Most cases handled automatically (fast, cheap).
- Hard cases get human attention.
- Reviewer load is bounded by the uncertainty rate.
- Confidence calibration is critical; too sensitive = high volume; too lenient = bad cases handled poorly.

### 2.4 Pattern D — Always-on copilot

The human is the operator; the AI is the assistant.

```
Human is doing work → AI suggests / autocompletes / drafts →
  human reviews / accepts / modifies each suggestion
```

**Examples.**

- GitHub Copilot in IDE.
- Email composition assistance.
- Document-drafting copilots.

**Properties.**

- Human is always in control; AI doesn't act independently.
- AI accelerates human work; doesn't replace it.
- High volume (many suggestions per hour); each lightweight to accept/reject.
- No escalation needed — human is already there.

### 2.5 The pattern comparison

| Property | A: Approver | B: Reviewer | C: Escalation | D: Copilot |
| --- | --- | --- | --- | --- |
| AI acts independently | No (gated) | Yes | Sometimes | No |
| User-facing latency | High (await approval) | Low | Mixed | Low (per suggestion) |
| Reviewer load | 1 per item | Sample of items | Only uncertain | Continuous |
| Best for | Irreversible / high-stakes | Quality control | Bounded uncertainty | Augmenting human work |
| Rubber-stamp risk | High | Medium | Low | Low |

The matrix helps narrow the right pattern per use case.

### 2.6 The hybrid patterns

Real systems often combine patterns:

- Approver pattern for high-stakes actions; reviewer pattern for everything else.
- Escalation pattern as default; approver pattern when escalation reveals high-stakes content.
- Copilot pattern in the UI; reviewer pattern on the published output.

Composition is fine when intentional; concerning when ad-hoc.

### 2.7 The "always Pattern A" mistake

Some teams default to Pattern A (approver) for safety: "the human always approves." The result: massive reviewer load; user-facing latency; throughput limited by reviewer capacity.

The corrective: per-use-case pattern selection. Pattern A only for cases that warrant the cost; other patterns elsewhere.

### 2.8 The "no HITL anywhere" mistake

Opposite extreme: the team ships without HITL, citing AI's competence. The first high-stakes mistake reveals the gap.

The corrective: per-use-case analysis identifies where HITL is needed; pattern chosen accordingly. Most production AI systems use HITL somewhere.

---

## 3. The decision criteria

How to choose the pattern.

### 3.1 Criterion 1 — Action reversibility

If the AI's action is reversible (a message can be unsent — well, sort of; a record can be deleted; a state can be reverted), Pattern B (reviewer after) is more acceptable. Mistakes are recoverable.

If the action is irreversible (a charge made, an email actually sent, a record permanently created), Pattern A (approver before) is the safer pattern. The cost of a mistake is high.

### 3.2 Criterion 2 — Action stakes

Low stakes (a wrong suggestion is mildly annoying): Pattern C (escalation) or B (reviewer) or D (copilot).

High stakes (a wrong action causes harm, legal liability, financial loss): Pattern A (approver) gates the action.

### 3.3 Criterion 3 — User-facing latency tolerance

If the user expects immediate response: Pattern A is wrong (the wait is unacceptable). Pattern B / C / D fit.

If the user accepts async response (will hear back later): Pattern A is feasible.

### 3.4 Criterion 4 — Reviewer capacity available

Reviewer capacity is a hard constraint. Pattern A on a high-volume feature requires many reviewers; if you don't have them, Pattern A isn't feasible regardless of other criteria.

The capacity sizing (per section 4) precedes pattern commitment.

### 3.5 Criterion 5 — Quality requirement

If the team needs every output to be high-quality (regulated industry, brand-sensitive), Pattern A or B with high coverage.

If lower quality is acceptable on the long tail, Pattern C is fine.

### 3.6 Criterion 6 — User's expectation of human involvement

In some domains (medical, legal), users expect a human to be involved. Pattern A or B is the user-facing promise even if technically not always necessary.

In other domains (consumer chatbots), users expect AI; HITL is invisible to them.

### 3.7 The decision tree

```
Is the action irreversible AND high-stakes?
  Yes → Pattern A (approver)
  No → continue

Is user-facing latency tolerance > 5 minutes?
  Yes → Pattern A possible; or B with quality review
  No → continue

Can the AI reliably classify "I'm uncertain"?
  Yes → Pattern C (escalation)
  No → continue

Is the human the primary operator (AI assists)?
  Yes → Pattern D (copilot)
  No → continue

Default: Pattern B (reviewer after) with sampling
```

The tree is a starting point; the team adapts.

### 3.8 The "right pattern for the right call" framing

A multi-call workflow may use different patterns per call:

- AI agent makes 5 routine read calls (no HITL needed).
- AI agent proposes a state change (Pattern A approval).
- AI agent generates a notification (Pattern B reviewer-after).

Per-call pattern selection is the right granularity.

---

## 4. Reviewer capacity sizing

The architectural constraint that often dominates.

### 4.1 The capacity math

Reviewer capacity:

```
items_per_reviewer_per_hour × reviewer_hours_per_day × number_of_reviewers
= items_capacity_per_day
```

Typical numbers:

- Simple items (text review): 60-120 per reviewer-hour.
- Complex items (clinical, legal review): 10-30 per reviewer-hour.
- Compound items (multi-faceted): 5-15 per reviewer-hour.

A reviewer working 6 hours per day (the rest in breaks, meetings, queue smoothing): 60-720 items per day depending on complexity.

### 4.2 The demand math

```
features_with_HITL × requests_per_feature_per_day × fraction_routed_to_HITL
= items_demand_per_day
```

Example: 3 features × 30k requests/day × 5% routed to HITL = 4,500 items/day.

### 4.3 The reviewer count

Reviewer count needed:

```
demand / capacity = items_per_day / items_per_reviewer_per_day
```

For 4,500 items/day at 100/reviewer-day (complex items): 45 reviewers.

For 4,500 items/day at 500/reviewer-day (simple items): 9 reviewers.

Reviewer cost: typically $50-200/hour fully loaded. At 9 reviewers × $80/hour × 8 hours × 250 days/year: $1.4M/year.

### 4.4 The capacity gap

If the team's projected demand exceeds available reviewer capacity:

- Reduce HITL volume (raise the threshold; auto-approve more).
- Use a different pattern (Pattern C with bounded escalation rate; Pattern D where human is operator).
- Scale up reviewers (cost; hiring time).
- Accept SLA violations (degraded UX).

Each has costs; the team picks. What's not acceptable is shipping with a capacity gap nobody acknowledged.

### 4.5 The growth math

If demand grows, capacity must grow proportionally. Pattern A on a 10x-growing feature means 10x reviewers; often not feasible.

Per-feature sustainability:

- Pattern A: sustainable only for bounded-growth use cases or where reviewer capacity scales with revenue.
- Pattern B with sampling: more sustainable (sample size grows slowly).
- Pattern C: sustainable if uncertainty rate stays bounded.
- Pattern D: scales with user count (each user reviews their own AI suggestions).

### 4.6 The peak-vs-average

Demand isn't uniform:

- Business-hours peak.
- End-of-day surges.
- Marketing-driven spikes.

Capacity sized for peak (with queue buffer) is expensive; sized for average (with SLA-violation acceptance during peaks) is cheaper but produces user-visible degradation.

Many teams use:

- Average capacity for base.
- Surge capacity for predictable peaks (extra reviewers on schedule).
- Queue + degraded SLA for unpredictable spikes.

### 4.7 The reviewer specialisation

Some reviewers handle multiple categories; some are specialised:

- Clinical content: reviewed by clinically-trained reviewers.
- Financial content: reviewed by finance reviewers.
- General content: reviewed by general-trained reviewers.

Specialisation increases quality but reduces flexibility; the team balances per workload.

### 4.8 The reviewer talent

Hiring reviewers:

- Quality matters (a reviewer who misses bad items defeats HITL).
- Training matters (reviewers need to understand the policies they're enforcing).
- Retention matters (turnover destroys consistency).

Reviewer talent investment is part of the operational cost; budget for it.

---

## 5. The rubber-stamp failure and correctives

The most common HITL anti-pattern.

### 5.1 What rubber-stamping looks like

Reviewer approval rate is 95%+. Rejection is rare. The reviewer's role is reduced to clicking "approve." The safety net isn't catching things; users get a false sense of human oversight.

### 5.2 Why rubber-stamping happens

- **High volume.** Reviewer is overwhelmed; can't review each item carefully.
- **Bad UX.** Approve is one click; reject takes longer (typing reason); friction asymmetry encourages approve.
- **AI quality is high.** Most items are genuinely correct; reviewer learns "default to approve."
- **No feedback to reviewer.** Reviewer doesn't see the consequence of approve vs reject; doesn't learn what to look for.
- **Audit gap.** No random-sample audit; rubber-stamping isn't detected.
- **Wrong incentive.** Reviewer's metrics are throughput (items per hour); quality (catch rate) isn't measured.

### 5.3 Corrective 1 — Reduce volume

Auto-approve high-confidence cases:

- AI's confidence score determines routing.
- High confidence → auto-approve.
- Uncertain → human review.
- The human reviews fewer items; can be more thoughtful.

The corrective is the right answer for many cases; reduces reviewer load and improves attention quality.

### 5.4 Corrective 2 — Symmetric UX

Approve and reject should have similar friction:

- One-click approve.
- One-click reject (reason optional; quick categories for common reasons).
- The reviewer's choice isn't biased by UX friction.

### 5.5 Corrective 3 — Random-sample audit

Even with auto-approval, random sampling routes some auto-approved items to human review. The human verifies the auto-approval was correct.

- Audit rate: 1-5% of auto-approved items.
- Auditor reviews; agreement rate is the auto-approval's accuracy.
- Disagreements are cases where auto-approval was wrong; feedback to improve the confidence model.

The audit catches drift; without it, the auto-approval's quality is unmeasured.

### 5.6 Corrective 4 — Reviewer feedback

The reviewer sees:

- The outcome of their decisions (was the approved item later found to be wrong?).
- Per-category catch rate (am I rejecting at the expected rate?).
- Calibration data (where am I diverging from other reviewers?).

The feedback loop helps the reviewer improve and stay engaged.

### 5.7 Corrective 5 — Quality metrics

Reviewer success measured by:

- Catch rate (rejecting items that shouldn't have been approved).
- Inter-reviewer agreement.
- Auditor agreement.

Not just throughput. The metrics align incentives with the safety value.

### 5.8 Corrective 6 — Time budget per item

Reviewer's expected time per item is calibrated:

- Simple item: 1-2 minutes.
- Complex item: 5-15 minutes.
- The metric is "thoughtful review per item time," not "items per hour."

If demand exceeds the time-budget × reviewer count, capacity is added (or volume reduced); not the time-per-item.

### 5.9 The "rubber stamp is acceptable" framing

For some teams, rubber-stamping is acceptable: the human is there for compliance, not substantive review. The audit acknowledges this; the architecture doesn't pretend the human is adding value.

This framing is honest but limits the HITL's value; the team should consider whether the HITL is worth the cost if it's only compliance theater.

### 5.10 The corrective combo

In practice: reduce volume (auto-approve high-confidence) + audit (catch what auto-approval misses) + reviewer feedback (improve quality) + quality metrics (right incentive).

Together they make HITL the safety net it's supposed to be.

---

## 6. UX implications per pattern

The user-facing experience varies significantly per pattern.

### 6.1 Pattern A UX

The user knows their request is awaiting approval:

- Submission confirmation ("your request has been submitted").
- Status updates ("in review by [role]; estimated [timeframe]").
- Resolution notification ("approved" / "rejected" / "modified").

The waiting can be frustrating; the UX must manage expectations.

### 6.2 Pattern B UX

The user gets immediate response:

- AI response delivered.
- Disclaimer optional ("this was generated by AI; we'll review it shortly").
- Notification if reviewer changes the response (rare; high-friction).

User-facing experience is smooth; the human's role is invisible (most of the time).

### 6.3 Pattern C UX

The user experiences a mix:

- Most queries answered immediately (AI-handled).
- Some queries trigger "this needs a human's attention" message.
- The hand-off is clear; the user knows what happened.

The mix must be coherent; the user shouldn't be surprised when escalation happens (or when it doesn't).

### 6.4 Pattern D UX

The AI is in the user's working environment:

- Suggestions appear at the right moment (in the editor; in the email composition).
- Accept / reject is low-friction (single keystroke).
- Suggestions don't interrupt the user's flow.

The UX is the product; quality of the integration matters more than the AI's raw capability.

### 6.5 The cross-pattern transitions

A single workflow may transition patterns:

- Default Pattern C (escalation); when escalated, becomes Pattern A (approver before action).
- Default Pattern D (copilot); on publish, becomes Pattern B (reviewer after).

The transitions are intentional; the user shouldn't experience whiplash.

### 6.6 The "where's the human" signal

Users should know when a human is involved:

- Some users prefer human handling (request escalation explicitly).
- Some users want speed (prefer AI-only).
- Mixed feature: the user knows which they're getting.

Transparency about the human's involvement builds trust.

### 6.7 The mobile considerations

HITL on mobile:

- Pattern A: notifications work well for approval requests.
- Pattern B: same as desktop; lower-friction UX.
- Pattern C: handoff to human may complicate mobile UX (push notifications, in-app updates).
- Pattern D: less common on mobile; some copilots work (text suggestions).

Mobile UX is its own design concern; HITL patterns adapt accordingly.

---

## 7. Operational considerations

The infrastructure layer.

### 7.1 The queue

Items routed to humans live in a queue:

- Per-pattern queues (approval queue; review queue; escalation queue; sample audit queue).
- Per-category queues (clinical; financial; general).
- Per-priority queues (urgent; standard).

The queue is the operational surface; observability and management matter.

### 7.2 The queue management

- Queue depth (how many items waiting).
- Age of oldest item.
- Average wait time.
- SLA compliance.

Alerts on queue depth and SLA violations.

### 7.3 The reviewer assignment

How items get assigned:

- **Pull-based.** Reviewer pulls next item from queue.
- **Push-based.** System assigns items to reviewers.
- **Specialised routing.** Item type routes to specialised reviewer.

Each has tradeoffs; pull-based scales better; push-based controls workload more precisely.

### 7.4 The reviewer interface

The reviewer's tool / app:

- Per-item context (per [refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md) section 4.3).
- Decision options (per pattern).
- Time-tracking (for capacity analysis).
- Quality metrics visible to the reviewer.
- Escalation option (re-route to senior or different specialist).

The interface is the reviewer's primary tool; UX investment pays off in efficiency and quality.

### 7.5 The audit infrastructure

Per-decision audit log:

- What was the item.
- What was the reviewer's decision.
- When.
- How long did it take.
- Quality outcomes (if measurable later).

The audit supports compliance, reviewer-quality analysis, and pattern-rate analysis.

### 7.6 The training and onboarding

New reviewers:

- Initial training on the policies and patterns.
- Shadow mode (decisions don't take effect; calibration against experienced reviewers).
- Graduated to live decisions.
- Continuous education on new patterns / categories.

The investment is significant; quality reviewers don't happen by accident.

### 7.7 The compliance integration

For regulated domains:

- Reviewer credentials may be required (clinical reviewer must be licensed).
- Audit logs must meet regulatory requirements.
- Specific items may require specific reviewer types.

The architecture enforces these constraints; the team can't accidentally route to unqualified reviewers.

### 7.8 The cost

HITL is operationally expensive:

- Reviewer salaries (often the largest cost).
- Interface and infrastructure.
- Training and quality programs.
- Compliance and audit.

For a 9-reviewer team at $80/hr × 8hr × 250 days: $1.4M/year base. Plus infrastructure and overhead.

Budget HITL operationally; it's not a free safety feature.

### 7.9 The vendor options

Some HITL functions can be vendored:

- Vendor moderation queues (Scale AI, Appen, Cohen Tech).
- Vendor expert reviewers (TaskUs for general; specialised firms for clinical, legal).

In-house HITL is more controllable; vendor HITL scales faster. Trade-offs.

### 7.10 The privacy considerations

Reviewers see user data:

- PII / PHI access controls.
- Audit of reviewer access.
- Reviewer training on confidentiality.
- Restrictions on data egress.

For regulated data, the reviewer infrastructure must meet regulatory requirements (HIPAA workforce training, etc.).

---

## 8. Worked Meridian example

Meridian's HITL architecture.

### 8.1 The HITL surface

Across Meridian's AI features:

| Feature | HITL pattern | Volume | Reviewer type |
| --- | --- | --- | --- |
| Care-coordinator (clinical Q&A) | Pattern C escalation | ~3% of interactions | Care manager |
| Care-coordinator (care plan changes) | Pattern A approver | All care plan changes | Patient's physician |
| Patient-summary | Pattern B reviewer (sample) | 5% audit | Clinical content reviewer |
| Analytics-copilot | Pattern D copilot (in analyst's tool) | All analyst-initiated | Analyst (the user) |
| Patient-API copilot | Pattern D copilot (in developer's IDE) | All suggestions | Developer (the user) |

Per-feature pattern; not uniform.

### 8.2 The reviewer capacity

- Care managers: 18 reviewers (clinical role; handle care-coordinator escalations + care plan reviews).
- Clinical content reviewers: 3 (sample audit of patient-summary).
- No dedicated reviewers for copilot patterns (Patterns D are self-service).

Total dedicated reviewer FTE: 21. Annual cost ~$3M loaded.

### 8.3 The reviewer demand

Per day:

- Care-coordinator escalations: ~1,500 items.
- Care plan approvals: ~500 items.
- Patient-summary samples: ~300 items.

Total: ~2,300 items/day for 21 reviewers = ~110 items per reviewer per day. Within capacity (calibrated at 100-150 per reviewer-day for clinical items).

### 8.4 Pattern A — Care plan approver

Care plan changes require physician approval:

- AI agent proposes the care plan change.
- Physician notified (in their EHR workflow); reviews; approves / modifies / rejects.
- User (clinician submitting the request) sees the proposal is pending; gets notified when approved.

SLA: 24 hours; typical resolution 6-8 hours.

The pattern is appropriate because care plan changes are clinically significant and physician approval is a regulatory expectation.

### 8.5 Pattern C — Care-coordinator escalation

Most care-coordinator interactions are handled by the AI directly. Escalations trigger on:

- AI confidence < 0.85 on clinical content.
- AI confidence < 0.75 on policy-sensitive content.
- User explicit request for human.

The 3% escalation rate is bounded; care manager queue stays within SLA (2-hour target; typical 30-90 minutes).

### 8.6 Pattern B — Patient-summary sample review

5% of patient summaries reviewed:

- Random sample selected.
- Reviewed by clinical content reviewer.
- Catch rate ~12% (some summaries have minor errors caught and corrected).
- Errors fed back to improve the summary prompt and eval.

Pattern B is appropriate because summaries are pre-published artefacts; correction in review prevents user-visible problems.

### 8.7 Pattern D — Analyst and developer copilots

Analytics-copilot and patient-API copilot work in the user's primary tool:

- Suggestions appear inline.
- Accept (single keystroke) / reject (single keystroke).
- No queue, no reviewer external to the user.

The pattern scales naturally with user count; no separate reviewer FTE needed.

### 8.8 The rubber-stamp prevention

For Pattern A (care plan approver):

- Physician approval rate ~78% (rejection rate 22%); not rubber-stamping.
- Random-sample audit by senior physician quarterly.
- Time-per-review averages 4 minutes; designed for thoughtful work.

For Pattern C (escalation):

- Care manager rejection / re-escalation rate ~15%.
- Quarterly calibration session across care managers.

Neither pattern shows rubber-stamping symptoms.

### 8.9 The Q3-25 incident

A surge in care-coordinator escalations (related to a marketing push that brought new users with unusual questions) overwhelmed care manager capacity. Queue grew to 6-hour backlog.

Mitigations:

- Auto-approval logic improved for high-confidence cases (reduced escalation rate by 30%).
- Care manager capacity scaled (3 additional reviewers for 2 months).
- Queue depth alerting added.

Post-incident: escalation routing improved; queue stayed within SLA.

### 8.10 The Q4-25 rubber-stamp scare

Patient-summary sample reviewers had approval rate of 96%; the team worried about rubber-stamping. Investigation:

- Manual senior review of a random sample of "approved" summaries showed 92% truly correct; 4% had subtle errors the reviewers missed.
- Reviewer training refreshed; specific error categories highlighted.
- Time budget per item increased from 3 to 5 minutes.
- Random audit by senior reviewer institutionalised.

Six months later: review rejection rate climbed to 12%; quality of approved summaries materially better.

### 8.11 What worked

- **Per-feature pattern selection.** No "one size" pattern; each feature got the right shape.
- **Reviewer capacity sized to demand.** No SLA violations from capacity mismatch.
- **Quality metrics not just throughput.** Reviewers focused on catch rate, not item rate.
- **Periodic audit.** Caught the Q4-25 rubber-stamp drift before it became an incident.

### 8.12 What didn't work initially

- **Pattern A for everything.** Early thinking was "everything goes through approval"; would have required 80+ care managers; abandoned in favour of per-feature patterns.
- **Reviewer interface poor.** First version was a basic web app; reviewers worked slowly. Investment in the interface improved per-reviewer throughput 40%.
- **No queue alerting.** Initial backlog issues weren't detected until users complained; alerting deployed.

---

## 9. Anti-patterns

### 9.1 "HITL added as afterthought"

The AI workflow is designed; HITL is bolted on; UX rough; throughput wrong-sized.

**Corrective.** HITL pattern in the architectural design per section 1.

### 9.2 "Pattern A for everything"

All AI actions require approval; reviewer load excessive; user-facing latency intolerable.

**Corrective.** Per-feature pattern selection per section 3.

### 9.3 "Rubber stamp"

Reviewer approval rate > 95%; safety value diluted.

**Corrective.** Volume reduction + UX + audit + quality metrics per section 5.

### 9.4 "Sync HITL blocking response"

User waits indefinitely for human approval (or even minutes); UX intolerable.

**Corrective.** Async HITL per [refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md) section 4.5.

### 9.5 "Capacity gap"

Pattern committed without reviewer capacity; queues back up; SLA violated.

**Corrective.** Sizing per section 4; reduce volume or pattern if capacity insufficient.

### 9.6 "No queue observability"

Queue depth, age, SLA not monitored; problems discovered by user complaints.

**Corrective.** Per section 7.2.

### 9.7 "Reviewer interface as afterthought"

Bad UX increases per-item time; quality suffers; reviewers burn out.

**Corrective.** Investment in reviewer interface per section 7.4.

### 9.8 "Reviewer metrics on throughput only"

Reviewers incentivised to approve quickly; catch rate ignored; rubber-stamp tendency.

**Corrective.** Quality metrics per section 5.7.

---

## 10. Findings (sprint-assignable)

### ARCH-HITL-001 — Severity: Critical
**Finding.** HITL pattern not deliberately chosen per feature; one pattern used everywhere.
**Recommendation.** Per-feature pattern selection per section 3.
**Owner.** architecture + product, sprint N+1.

### ARCH-HITL-002 — Severity: Critical
**Finding.** Reviewer capacity insufficient for current demand; queues back up; SLAs violated.
**Recommendation.** Sizing per section 4; reduce volume or scale capacity.
**Owner.** product + operations, sprint N+1.

### ARCH-HITL-003 — Severity: Critical
**Finding.** Rubber-stamp pattern detected (approval > 95%); HITL not providing safety value.
**Recommendation.** Correctives per section 5; reduce volume; UX symmetry; quality metrics.
**Owner.** product + ai-platform-eng, sprint N+1.

### ARCH-HITL-004 — Severity: High
**Finding.** Sync HITL blocking user response; UX intolerable.
**Recommendation.** Async HITL per section 6.1 / 6.2; status updates.
**Owner.** product + ai-platform-eng, sprint N+2.

### ARCH-HITL-005 — Severity: High
**Finding.** Queue observability absent; problems detected only by user complaints.
**Recommendation.** Queue depth + age + SLA monitoring per section 7.2.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-HITL-006 — Severity: High
**Finding.** Reviewer metrics on throughput only; quality and catch rate not measured.
**Recommendation.** Quality metrics per section 5.7.
**Owner.** product + ai-platform-eng, sprint N+2.

### ARCH-HITL-007 — Severity: High
**Finding.** No random-sample audit; rubber-stamping risk undetected.
**Recommendation.** Audit per section 5.5; quarterly cadence.
**Owner.** product + ai-platform-eng, sprint N+2.

### ARCH-HITL-008 — Severity: Medium
**Finding.** Reviewer interface poor; per-item time high; quality suffers.
**Recommendation.** Interface investment per section 7.4.
**Owner.** product, sprint N+3.

### ARCH-HITL-009 — Severity: Medium
**Finding.** Pattern selected without considering reviewer capacity; theoretical only.
**Recommendation.** Capacity-first analysis per section 4.4.
**Owner.** architecture + product, sprint N+3.

### ARCH-HITL-010 — Severity: Medium
**Finding.** Reviewer training and onboarding informal; quality varies.
**Recommendation.** Per section 7.6.
**Owner.** product + operations, sprint N+3.

### ARCH-HITL-011 — Severity: Medium
**Finding.** Specialisation mismatch; reviewers handling content outside their expertise.
**Recommendation.** Per section 4.7; specialised queues.
**Owner.** product, sprint N+3.

### ARCH-HITL-012 — Severity: Medium
**Finding.** Reviewer access to user data not audited; privacy compliance risk.
**Recommendation.** Per section 7.10.
**Owner.** privacy + ai-platform-eng, sprint N+3.

### ARCH-HITL-013 — Severity: Medium
**Finding.** Auto-approval logic absent; all items reviewed manually; capacity wasted.
**Recommendation.** Per section 5.3; high-confidence cases auto-approved.
**Owner.** ai-platform-eng + product, sprint N+4.

### ARCH-HITL-014 — Severity: Medium
**Finding.** Peak-demand handling ad-hoc; backlogs during predictable surges.
**Recommendation.** Per section 4.6.
**Owner.** operations + product, sprint N+4.

### ARCH-HITL-015 — Severity: Low
**Finding.** Reviewer compensation / incentives misaligned with quality.
**Recommendation.** Per section 5.7; review compensation structure.
**Owner.** HR + leadership, sprint N+4.

### ARCH-HITL-016 — Severity: Low
**Finding.** Reviewer career path absent; turnover high.
**Recommendation.** Per section 4.8; reviewer career investment.
**Owner.** HR + operations, sprint N+5.

### ARCH-HITL-017 — Severity: Low
**Finding.** Vendor reviewer options not evaluated; in-house may be expensive.
**Recommendation.** Per section 7.9; cost-benefit analysis.
**Owner.** operations + finance, sprint N+5.

### ARCH-HITL-018 — Severity: Low
**Finding.** Reviewer-decision audit log not queryable; compliance review slow.
**Recommendation.** Per section 7.5; structured logs; queryable.
**Owner.** ai-platform-eng + compliance, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team designing HITL:

- [ ] **Sprint 0 — per-feature analysis.** Which HITL pattern fits each AI feature?
- [ ] **Sprint 0 — capacity sizing.** Reviewer count needed for projected demand.
- [ ] **Sprint 0 — capacity decision.** Hire reviewers? Reduce volume? Different pattern?
- [ ] **Sprint 1 — reviewer interface.** Per section 7.4; quality investment.
- [ ] **Sprint 1 — queue infrastructure.** Per section 7.1 / 7.2.
- [ ] **Sprint 1 — async UX.** Per section 6; status updates; expectations.
- [ ] **Sprint 2 — quality metrics.** Per section 5.7; not just throughput.
- [ ] **Sprint 2 — random-sample audit.** Per section 5.5.
- [ ] **Sprint 3 — reviewer training.** Per section 7.6.
- [ ] **Sprint 3 — observability.** Queue depth, age, SLA, decision rates.
- [ ] **Sprint 4 — auto-approval logic.** For Pattern A or B; reduces volume.
- [ ] **Sprint 4 — privacy and compliance.** Per section 7.10 / 7.7.
- [ ] **Ongoing — quarterly review.** Capacity vs demand; rubber-stamp check; quality.

For a team retrofitting:

- [ ] **Sprint 0 — audit.** What HITL exists; which patterns; what's the quality.
- [ ] **Sprint 1 — fix rubber-stamping.** If detected, apply correctives.
- [ ] **Sprint 2 — capacity review.** Right-size for current demand.
- [ ] **Sprint 3 — interface improvements.** Per section 7.4.

A team that completes the sequence has HITL that adds genuine value at appropriate cost. A team that doesn't has HITL that's either friction or theater.

---

## 12. References

- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — async patterns for HITL.
- [tool-call-architecture.md](./tool-call-architecture.md) — tool authorization can require HITL.
- [event-driven-ai-integration.md](./event-driven-ai-integration.md) — events drive HITL queues.
- [backpressure-and-queueing.md](./backpressure-and-queueing.md) — queue management for HITL.
- [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md) — async HITL responses.
- [integration-failure-patterns.md](./integration-failure-patterns.md) — HITL fallback if capacity insufficient.
- [guardrails-and-policy-architecture/content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md) — HITL in moderation.
- [guardrails-and-policy-architecture/refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md) — refusal/escalation interacts with HITL routing.
- [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) — HITL on tool authorization.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — HITL routing as policy.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) — HITL-as-rubber-stamp anti-pattern.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — engineering pattern for HITL integration.
- Sibling repo: ai-security-reference-architecture — HITL as security control.
- Conversational AI and HITL literature — e.g., labelling industry references (Scale, Appen).
- "Human-in-the-Loop Machine Learning" (Munro, 2021) — broader HITL literature.
