# Refusal and Escalation Design

> **Audience.** Architects designing how AI features respond when they shouldn't handle the request themselves — what they refuse, what they escalate, what they degrade. Tech leads whose AI feature's refusals frustrate users instead of helping them. Product teams whose "the AI just said no" complaint backlog is growing. **Scope.** The *architectural* design — when to refuse vs escalate vs degrade; refusal pattern catalogue; escalation routing; HITL handoff; user-facing explanation; eval of refusal appropriateness. Not the engineering of escalation flows (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)). Not the moderation classification (see [content-moderation-architecture.md](./content-moderation-architecture.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Refusal is a feature, not friction. An AI system that refuses well — clearly, with a helpful path forward, with an appropriate escalation — earns user trust. An AI system that refuses badly — opaquely, with no next step, with no escalation — erodes trust and trains users to work around it.

Most AI features ship with refusal as an afterthought. The model is prompted to "refuse if you don't know"; the moderation layer rejects with a generic message; the agent escalates with "I cannot complete this." The user is left wondering what they should do; the team is left wondering whether the refusal was appropriate; nobody learns from the refusals because they aren't observable.

The architectural decisions are: what triggers refusal vs escalation vs degradation; what the refusal looks like to the user; where the escalation goes; what the human reviewer sees; how appropriateness is evaluated. Done well, these decisions turn refusal into a productive signal — a feature users navigate and a source of product feedback. Done badly, they turn refusal into a frustration generator and a quality problem.

This document is opinionated about four things:

1. **Refusal, escalation, and degradation are different responses to different conditions.** Each has its own architectural pattern; conflating them produces bad outcomes for all three.
2. **Refusals must include a path forward.** "I cannot help" without next steps is rude; "I cannot help, but here's what you can do" is helpful.
3. **Escalations must reach a human who can actually help.** Routing escalations to "the team" without specific responsibility is a black hole.
4. **Refusals are observable and eval'd.** Per-refusal logging, refusal-appropriateness eval, product feedback loop. The refusal isn't the end of the engagement; it's the start of the next iteration.

Structure: (2) the three response types — refuse / escalate / degrade; (3) refusal pattern catalogue; (4) escalation routing and HITL handoff; (5) user-facing explanation design; (6) the refusal observability loop; (7) eval of refusal appropriateness; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three response types

Three distinct architectural patterns; each fits different conditions.

### 2.1 Refuse

The system tells the user it cannot help with this request and provides guidance.

**When right.**

- The request is out of scope (system can't help with this kind of thing).
- The request violates policy (system won't help with this kind of thing).
- The request requires capabilities or data the system doesn't have.
- The request would produce a low-quality / unsafe response.

**Characteristics.**

- Clear about why (without revealing sensitive policy details).
- Includes a path forward (alternative tool, retry with different framing, contact human).
- Logged and observable.
- Not the same as "error" (an error is when something broke; a refusal is when the system worked correctly to say no).

### 2.2 Escalate

The system hands the interaction to a human or another system that can help.

**When right.**

- The request is within scope but requires human judgment.
- The request is too high-stakes for the system to handle alone.
- The system is uncertain and the consequence of being wrong is significant.
- The user requested escalation.
- The request requires actions the system cannot take.

**Characteristics.**

- The escalation goes to a specific, accountable destination (not "the team").
- The user is informed about the escalation (what's happening, expected timeline).
- The destination receives context (what the user asked, what's been done, why escalation).
- The user remains in control (can cancel, can follow up).

### 2.3 Degrade

The system continues with reduced capabilities or a fallback path.

**When right.**

- A dependency is down (model, tool, data source).
- The request can be partially satisfied (some parts work; some don't).
- The system can provide a less-good answer rather than no answer.
- The cost / time / capacity budget is constrained.

**Characteristics.**

- The user gets a response, possibly less complete than usual.
- The user is informed about the degradation (if material).
- Logged and observable for diagnosis and follow-up.
- A path to retry with full capability when conditions allow.

### 2.4 The "default to refusal" failure

Some teams refuse when escalation or degradation would be better. The user gets a "I cannot help" when actually a human could have helped (escalation) or a partial response would have been useful (degradation).

The corrective: choose deliberately per condition. The matrix below helps.

### 2.5 The matrix

| Condition | Right response | Wrong responses |
| --- | --- | --- |
| Out of scope | Refuse + suggest alternative | Hallucinated partial answer |
| Policy violation | Refuse + cite policy | Hallucinated reason |
| Within scope, but uncertain | Escalate to expert | Refuse without explanation |
| Within scope, high stakes | Escalate to approver | Auto-execute |
| Dependency down | Degrade (use fallback) | Refuse with "service unavailable" |
| Budget exceeded | Degrade (simpler response) | Cut off mid-response |
| User asks for human | Escalate | Insist on AI handling |

The matrix is per-architecture; teams adapt to their specific failure modes.

### 2.6 The composability

A single interaction may use multiple patterns:

- The system attempts a response → degrades when a tool is slow → escalates when the degraded response is insufficient → refuses if escalation is unavailable.

The composition is intentional; the user sees a single coherent experience even though the system used multiple patterns.

### 2.7 The "always escalate, just in case" anti-pattern

Some teams escalate everything as a CYA pattern. Human reviewers are overwhelmed; queues back up; the value of escalation is diluted (rubber-stamp problem).

The corrective: refuse what the system shouldn't handle; escalate what humans should decide; degrade what tolerates reduced capability. Each pattern for its specific case.

---

## 3. Refusal pattern catalogue

The shapes refusals take.

### 3.1 The "out of scope" refusal

```
"I can help with care coordination questions about patients in your care. 
This question seems to be about [topic], which is outside my scope. 

You can try:
- Rephrasing your question to focus on a specific patient
- Using [other_feature] for [topic] questions
- Asking your care manager directly"
```

**Pattern.** Acknowledge the question; explain scope; offer alternatives.

### 3.2 The "policy violation" refusal

```
"I'm not able to help with this request. 

Our policy requires that [high-level reason, e.g., 'medical advice be provided 
by a licensed clinician']. 

For this kind of question, please contact [appropriate destination, e.g., 
'your physician's office' or 'our patient support line at X-XXX-XXXX']."
```

**Pattern.** Explain at the right level (not too specific to leak policy details; not too vague to be useless); provide the appropriate destination.

### 3.3 The "capability gap" refusal

```
"I'd like to help with this, but I don't have access to [the data / capability]. 

[If true:] In the future, this kind of question may be supported.

For now, please [alternative path]."
```

**Pattern.** Honest about the limitation; suggest path forward.

### 3.4 The "uncertain" refusal (often paired with escalation)

```
"I'm not confident in the right answer here. 

Rather than risk giving you incorrect information, I've forwarded your question 
to a care manager. They'll get back to you within [timeframe].

If this is urgent, please contact [urgent destination]."
```

**Pattern.** Honest about uncertainty; escalation as the path forward; urgency option.

### 3.5 The "tried and failed" refusal

```
"I tried [actions taken] but wasn't able to complete this. 

This may be because [likely reason if known]. 

You can try [alternative], or I can [escalation option]."
```

**Pattern.** Show effort; give honest assessment; offer next step.

### 3.6 The bad refusal (anti-pattern)

```
"I cannot help with this request."
```

No reason, no path forward, no acknowledgment. Trains users to dislike the system; doesn't help anyone.

Replace with one of the patterns above.

### 3.7 The hallucinated refusal

```
"I cannot help with this because [made-up reason that doesn't match actual policy]."
```

The model invents a refusal reason. Misleading; can produce compliance violations.

Mitigation: refusal text is from a template (policy-driven) rather than free-form generated; the model decides *whether* to refuse, not *what to say* in the refusal.

### 3.8 The "passive-aggressive" refusal

```
"I cannot help with that, but here's some information you didn't ask for: ..."
```

The refusal that ignores what the user asked and answers a different question. Frustrating.

Replace: explicit refusal + clear alternative.

### 3.9 The "soft refusal" (declining without refusing)

For some cases, the right pattern is:

```
"That's an interesting question. Let me share what I know: [partial answer with caveats]. 

This isn't a substitute for [authoritative source]; for definitive guidance, 
please consult [destination]."
```

The system provides what it can while clearly indicating it's not the final word. Useful when complete refusal is unhelpful but the system shouldn't be authoritative.

### 3.10 The pattern selection

| Situation | Recommended pattern |
| --- | --- |
| Out of scope | Section 3.1 |
| Policy violation | Section 3.2 |
| Missing capability | Section 3.3 |
| Uncertain on high-stakes | Section 3.4 |
| Tried and failed | Section 3.5 |
| Below threshold but informative | Section 3.9 |

Each pattern templates so the team can ensure consistency.

---

## 4. Escalation routing and HITL handoff

Where escalations go and how the handoff works.

### 4.1 The escalation destinations

Per organisation:

- **Specific human reviewer / approver.** Named role with explicit responsibility.
- **Queue of reviewers.** Pool of qualified reviewers; load-balanced.
- **Specific team.** With defined SLA for response.
- **External party.** (Physician's office; legal team; etc.)
- **Another AI system.** (More capable model; specialised feature.)
- **The user themselves.** "Please follow up via this form" — the user is the escalation path.

### 4.2 The destination selection

Per refusal / escalation type:

- Medical-content uncertainty: escalate to care manager.
- Care-plan changes: escalate to attending physician.
- Account-data questions: escalate to customer support.
- Policy interpretation: escalate to compliance team.

The mapping is policy-driven (per [policy-as-code-for-ai.md](./policy-as-code-for-ai.md)); the architecture routes accordingly.

### 4.3 The HITL handoff context

When escalating to a human, the human needs context:

- What the user asked (verbatim).
- What the AI tried (trajectory summary).
- Why escalating (specific reason).
- What the user expects (timeline, action).
- Relevant data (patient context, account context).
- Any prior interactions on this topic.

The handoff is structured; the human gets a complete picture without having to investigate.

### 4.4 The reviewer interface

Per [content-moderation-architecture.md](./content-moderation-architecture.md) section 7.6. The reviewer's interface:

- Per-item context.
- Decision options (approve / modify / refuse / re-escalate).
- Time-budget per item (designed for thoughtful work).
- History of similar items (for consistency).

### 4.5 The user-side experience

When escalation is triggered:

- User is informed (clear message; not just silence).
- Expected timeline communicated (if known).
- Identifier provided for follow-up.
- Alternative path if urgent (e.g., emergency contact).

Async escalation works only if the user knows what's happening; sync escalation (user waits) is rarely the right experience.

### 4.6 The escalation acknowledgement

When the human reviewer takes the item:

- User notified ("your question is being reviewed by [role]").
- Status updated.
- Estimated completion time updated.

### 4.7 The escalation resolution

When the reviewer's decision is made:

- User receives the resolution (approval + response, or refusal + reason, or modification + explanation).
- If actionable, the system executes (e.g., the care plan change is committed).
- Logged for audit.

### 4.8 The "escalation black hole" anti-pattern

Escalation goes to a queue nobody is responsible for; items sit indefinitely; user never hears back.

Mitigation:

- Every escalation queue has an owner.
- SLA on response time.
- Alerting on queue depth and oldest item.
- Per-item ageing visible to the team.

### 4.9 The "re-escalation" pattern

Sometimes a reviewer needs to escalate further:

- Care manager → physician (clinical complexity beyond care manager).
- Customer support → manager (refund authority needed).
- Compliance team → legal (legal question).

The system supports multi-hop escalation; each hop adds context; each hop has its own SLA.

### 4.10 The reciprocal escalation

The user can also escalate the AI:

- "I'd like to talk to a human" → immediately routes to a human reviewer.
- "This isn't what I needed" → flag for review; potentially re-escalate.

User-initiated escalation is a respected path; not something to fight.

---

## 5. User-facing explanation design

What the user actually sees.

### 5.1 The explanation requirements

A good refusal / escalation message:

- **Acknowledges what the user asked.** Demonstrates the system understood.
- **Explains the response.** Why this response (refuse / escalate / degrade).
- **Provides a path forward.** What the user can do next.
- **Sets expectations.** If escalated, when they'll hear back.
- **Doesn't reveal sensitive details.** Policy specifics, internal reasons that could be abused.
- **Uses appropriate tone.** Not robotic; not over-apologetic; not condescending.

### 5.2 The "I cannot do that, Dave" tone

A robotic refusal feels alienating. Better:

- "I'm not able to help with this directly, but here's what I can do..."
- "This needs a human's review; I've sent it to..."
- "I tried but couldn't complete it. Let me suggest..."

The tone is conversational and helpful; the message still clearly indicates the AI can't do what was asked directly.

### 5.3 The "fake humanity" anti-pattern

Over-apologetic refusals feel disingenuous:

- "I'm so sorry, I really wish I could help with this, but unfortunately..."

This wastes the user's time and feels performative. Direct, brief, helpful is better.

### 5.4 The "wall of text" anti-pattern

Long explanations of why and detailed alternatives can bury the action. Better: short clear refusal + the most important next step + offer for more detail.

```
"This kind of question needs a clinician's review. I've sent it to your care 
manager — they'll respond within 2 hours.

Need more help right now? Call the patient line at X-XXX-XXXX."
```

Concise. Action-oriented. Includes urgency option.

### 5.5 The language and accessibility

- Plain language (not jargon).
- Localised (per user's language preference).
- Accessible (screen-reader-friendly; appropriate reading level).

These aren't optional; they're part of inclusive design.

### 5.6 The status updates

For long-running escalations:

- "Your question has been reviewed and approved. Here's the response..."
- "Your question is still in review; we'll update you in 30 minutes."
- "Your question requires additional clarification. Please provide..."

Periodic status reduces user anxiety and the "did anyone see this" follow-ups.

### 5.7 The mobile / multi-channel considerations

Refusal / escalation may happen across channels:

- User asks via web; gets a refusal; the escalation result comes via email.
- User asks via SMS; gets a status update via SMS; final response via voice call.

The architecture supports cross-channel flow; the user shouldn't have to track which channel to check.

### 5.8 The explanation template library

Templates per pattern × user type × language. Maintained centrally; updated as the team learns what works.

Versioned; A/B tested for clarity and user satisfaction.

---

## 6. The refusal observability loop

Refusals are signals; observe them.

### 6.1 What to observe

Per refusal / escalation:

- Type (out of scope, policy, capability, uncertain, etc.).
- Trigger (input characteristics, model output classification, tool failure, etc.).
- Destination (refused, escalated to whom).
- User-facing message used.
- User reaction (continued? gave up? rated negatively?).
- Resolution (if escalated, what was the human's decision?).

### 6.2 The aggregations

- Refusal rate per feature, per user type, per topic.
- Top refusal reasons.
- Escalation rate per destination.
- Average resolution time per destination.
- User satisfaction by refusal type.

### 6.3 The "high refusal rate" signal

If a feature has a high refusal rate, possible interpretations:

- The feature is being used out of scope (UX guidance issue).
- The system is over-refusing (policy too strict; eval revealing this).
- Users are testing the boundaries (intentional probes).
- The model has degraded (regression).

Each interpretation has a different response; the observability supports diagnosis.

### 6.4 The "high escalation rate" signal

If escalation is high, possible interpretations:

- The system is genuinely uncertain (needs better data or model).
- The uncertainty threshold is too sensitive.
- The destination is overloaded.
- The escalation criteria are too broad.

### 6.5 The continuous eval of refusal appropriateness

Were the refusals correct? Per section 7.

### 6.6 The product feedback loop

Refusal patterns inform product decisions:

- "Many users ask about X; system refuses; we should add support for X."
- "Users escalate Y frequently; we should expand the human-handling capacity for Y."
- "Refusal-rate spike correlates with a recent prompt change; investigate."

The feedback loop is regular (weekly or monthly review); refusal patterns inform roadmap.

### 6.7 The dashboards

Per-feature dashboard:

- Refusal / escalation / degradation rates over time.
- Top reasons.
- Destination breakdown for escalations.
- User satisfaction proxy (continued vs gave up).

The dashboard is reviewed alongside other quality metrics.

### 6.8 The privacy of refusal data

Refusal data may contain user inputs; treat with same care as other AI logs:

- PII / PHI redaction at capture.
- Retention per policy.
- Access controls.

---

## 7. Eval of refusal appropriateness

The quality measurement for refusals.

### 7.1 The eval question

For each refusal: was it appropriate?

- Should the system have refused? (Was the refusal correct?)
- Was the refusal pattern right? (Was it informative? Did it have a path forward?)
- Did the escalation go to the right destination?
- Did the user get what they needed (eventually)?

### 7.2 The methods

**LLM-judge.** Given a refused interaction and the policy, judge whether the refusal was appropriate.

**Human-judge.** Sample of refusals reviewed by domain experts.

**Outcome eval.** Did the user accomplish their goal (via escalation, alternative path, etc.)?

**User satisfaction.** Direct feedback (rating; follow-up survey).

The combination produces a clearer picture than any one alone.

### 7.3 The false-positive rate

Refusals that should not have happened. The user wanted help the system could have provided; the system refused inappropriately.

Detection:

- LLM-judge or human-judge sample.
- User satisfaction signals (low rating after refusal).
- Follow-up where user got the same question answered elsewhere.

False positives erode user trust. The team's eval should bound them.

### 7.4 The false-negative rate

Cases the system should have refused but didn't.

Detection harder (the system did respond; it may not be obvious the response was wrong).

- Periodic adversarial testing.
- Post-hoc review of high-stakes interactions.
- User feedback indicating harm or inappropriate response.

False negatives are often more dangerous than false positives.

### 7.5 The eval set

A refusal-eval set includes:

- Examples that should refuse (out-of-scope, policy violation, etc.).
- Examples that should NOT refuse (in-scope, within policy).
- Edge cases (ambiguous; what should the system do?).

The set is curated and expanded over time as production reveals patterns.

### 7.6 The promotion gate

Prompt / model / policy changes that affect refusal behaviour go through:

- Refusal-eval on the changes.
- Comparison to current production.
- Promotion if no regression.

The gate prevents inadvertent changes to refusal behaviour.

### 7.7 The user feedback integration

When users provide feedback on refused interactions:

- Feedback captured (rating, comment, "this was wrong" flag).
- Reviewed regularly.
- Fed into the eval set as new cases.

The loop closes: production refusals → user feedback → eval set → improved refusal behaviour.

---

## 8. Worked Meridian example

Meridian's refusal and escalation architecture.

### 8.1 The architectural map

```
Care-coordinator interaction →
  if input out of scope: → refuse with alternative
  if input policy violation (medical advice from non-clinician): → refuse + escalation hint
  if uncertain (confidence < 0.85 on clinical content): → escalate to care manager
  if high-stakes action (care plan change): → escalate to physician
  if dependency down (retrieval failed): → degrade (acknowledge + offer retry)
  if cost / time budget breached: → degrade (graceful end)
  if user requests human: → escalate to care manager
  if all paths fail: → refuse with care line phone number
```

### 8.2 The refusal patterns in use

| Pattern | Frequency | Example trigger |
| --- | --- | --- |
| Out of scope | ~2% of interactions | Question about unrelated medical topic |
| Policy violation | ~0.5% | Diagnostic question from non-clinician |
| Capability gap | ~1% | Question about feature not yet built |
| Uncertain (with escalation) | ~3% | Complex clinical question; agent unsure |
| Tried and failed | ~0.5% | Tool sequence failed |
| Soft refusal (partial answer + disclaimer) | ~2% | Question on the edge of medical advice |

Total refusal-related response rate: ~9%. Escalation rate: ~3% (within refusals).

### 8.3 The escalation routing

| Refusal type | Destination | SLA |
| --- | --- | --- |
| Clinical uncertainty | Care manager queue | 2 hours |
| Care plan changes | Patient's physician | 24 hours |
| Account questions | Customer support | 4 hours |
| Billing | Billing team | 24 hours |
| User-requested human | Care manager queue | 2 hours |

Each queue has owners; SLA monitoring; alerting on queue depth.

### 8.4 The user-facing messages

Templates per pattern. Example (clinical uncertainty escalation):

```
"This is a question I'd like a care manager to review with you.

I've forwarded your question to your care team — they'll respond within 
2 hours during business hours.

If this is urgent, please call [phone] or use the emergency contact form."
```

Templates versioned; A/B tested for clarity.

### 8.5 The observability

Per-feature dashboard:

- Refusal/escalation/degradation rates over time.
- Top reasons.
- Per-destination escalation counts.
- Average resolution time per destination.
- User satisfaction (1-5 rating after each interaction).

Weekly review by product + AI platform.

### 8.6 The refusal eval

- Eval set: ~200 cases (should-refuse + should-not-refuse + edge cases).
- LLM-judge runs daily on a sample of production refusals.
- Quarterly human-judge review of LLM-judge accuracy.
- Refusal-affecting changes go through the eval gate.

Current accuracy on the eval set: 92% (target 90%+).

### 8.7 The incidents

**Q3-25.** False-positive spike: a prompt change made the medical-content classifier overly sensitive; refusal rate jumped from 9% to 16% for one week before detection.

Mitigations: rollback the prompt change; refusal-rate alerting added (alerts when rate exceeds rolling 7-day +30%); refusal-eval gate caught the change retrospectively (it had passed because the eval set didn't have enough edge cases).

**Q1-26.** Escalation backlog: care manager queue grew to 6-hour wait; user-facing SLA violation.

Mitigations: queue depth alerting; capacity increase for care manager team; auto-approval logic for low-risk cases to reduce queue load.

**Q1-26.** False-negative discovered: a user asked an out-of-scope clinical question; the system answered with hallucinated content; the user noticed and complained.

Mitigations: out-of-scope classifier improved; eval cases added; medical-content scope guardrails strengthened.

### 8.8 What worked

- **Templated user-facing messages.** Consistent; clear; helpful.
- **Per-destination escalation routing.** No black holes; SLAs met.
- **Soft refusal pattern.** Captures cases where complete refusal would be unhelpful but the system shouldn't be authoritative.
- **Refusal eval and product feedback loop.** Refusals inform product roadmap.

### 8.9 What didn't work initially

- **Generic refusal messages.** "I cannot help with this" produced high user dissatisfaction; templates with paths forward changed this.
- **No escalation owner.** Initially escalations went to "the team"; nothing happened. Per-destination ownership and SLAs deployed.
- **Refusal data not analysed.** First year of operation, refusal data accumulated unanalysed; weekly review changed this; the data became a product input.

---

## 9. Anti-patterns

### 9.1 "Bare 'I cannot help' refusal"

No reason, no alternative, no path forward. User frustration; system distrust.

**Corrective.** Pattern-driven refusals per section 3.

### 9.2 "Escalation to a black hole"

Escalations go to a queue nobody owns; items linger.

**Corrective.** Owners + SLAs per section 4.8.

### 9.3 "Sync escalation blocking response"

User waits for human approval; UX intolerable.

**Corrective.** Async escalation per section 4.5; user gets immediate acknowledgment.

### 9.4 "Default to refusal"

The system refuses when escalation or degradation would help more.

**Corrective.** Per-condition matrix per section 2.5.

### 9.5 "Hallucinated refusal reasons"

The model invents the refusal text; misleading users.

**Corrective.** Template-driven messages per section 3.7.

### 9.6 "Refusals not observable"

Refusal data not captured or analysed; nobody knows the patterns.

**Corrective.** Per-refusal logging and dashboards per section 6.

### 9.7 "No refusal eval"

Whether refusals are appropriate is unknown; false positives and negatives accumulate.

**Corrective.** Refusal eval per section 7.

### 9.8 "Always escalate as CYA"

Every uncertain case escalates; queues overwhelmed; value of escalation diluted.

**Corrective.** Tiered handling per section 2.7; escalate only what humans should decide.

---

## 10. Findings (sprint-assignable)

### ARCH-REFUSE-001 — Severity: Critical
**Finding.** Refusals use bare "cannot help" messages; no path forward.
**Recommendation.** Template-driven refusals per section 3.
**Owner.** product + ai-platform-eng, sprint N+1.

### ARCH-REFUSE-002 — Severity: Critical
**Finding.** Escalation queues lack owners and SLAs; items linger.
**Recommendation.** Per-destination ownership and SLAs per section 4.8.
**Owner.** product + ai-platform-eng, sprint N+1.

### ARCH-REFUSE-003 — Severity: Critical
**Finding.** Sync escalation blocks user response; UX intolerable.
**Recommendation.** Async escalation per section 4.5.
**Owner.** product + ai-platform-eng, sprint N+1.

### ARCH-REFUSE-004 — Severity: High
**Finding.** Refusals not observable; patterns not analysed; product feedback not closed.
**Recommendation.** Observability + weekly review per section 6.
**Owner.** ai-platform-eng + product, sprint N+2.

### ARCH-REFUSE-005 — Severity: High
**Finding.** Refusal appropriateness not eval'd; false positives / negatives accumulate.
**Recommendation.** Eval per section 7; promotion gate.
**Owner.** ai-platform-eng + product, sprint N+2.

### ARCH-REFUSE-006 — Severity: High
**Finding.** Refusal messages hallucinated by model; reasons inconsistent or misleading.
**Recommendation.** Template-driven messages per section 3.7.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-REFUSE-007 — Severity: High
**Finding.** Default to refusal where escalation or degradation would be better.
**Recommendation.** Per-condition response matrix per section 2.5.
**Owner.** architecture + product, sprint N+2.

### ARCH-REFUSE-008 — Severity: Medium
**Finding.** Escalation context lacks structure; reviewers spend time gathering context.
**Recommendation.** Structured handoff per section 4.3.
**Owner.** ai-platform-eng + product, sprint N+3.

### ARCH-REFUSE-009 — Severity: Medium
**Finding.** User satisfaction on refusals not captured; UX impact unknown.
**Recommendation.** Per-interaction feedback capture per section 6.1.
**Owner.** product, sprint N+3.

### ARCH-REFUSE-010 — Severity: Medium
**Finding.** Refusal-related changes not gated on eval; regressions reach production.
**Recommendation.** Refusal eval in CI gate per section 7.6.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-REFUSE-011 — Severity: Medium
**Finding.** Always-escalate-as-CYA pattern; reviewer queues overwhelmed.
**Recommendation.** Tiered handling per section 2.7.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-REFUSE-012 — Severity: Medium
**Finding.** Multi-channel escalation experience inconsistent; users lose track.
**Recommendation.** Cross-channel coordination per section 5.7.
**Owner.** product, sprint N+3.

### ARCH-REFUSE-013 — Severity: Medium
**Finding.** Refusal templates not versioned; inconsistencies across features.
**Recommendation.** Template library per section 5.8.
**Owner.** product + ai-platform-eng, sprint N+4.

### ARCH-REFUSE-014 — Severity: Medium
**Finding.** Refusal rate alerting absent; spikes go undetected.
**Recommendation.** Alerting per section 6.3.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-REFUSE-015 — Severity: Low
**Finding.** Reviewer UX poor; long times per item; backlog grows.
**Recommendation.** Reviewer UX investment per section 4.4.
**Owner.** product, sprint N+4.

### ARCH-REFUSE-016 — Severity: Low
**Finding.** Re-escalation flow undefined; complex cases stall at first reviewer.
**Recommendation.** Per section 4.9; multi-hop escalation defined.
**Owner.** product + ai-platform-eng, sprint N+5.

### ARCH-REFUSE-017 — Severity: Low
**Finding.** User-initiated escalation friction; "I want a human" not a respected path.
**Recommendation.** Per section 4.10; reciprocal escalation supported.
**Owner.** product + ai-platform-eng, sprint N+5.

### ARCH-REFUSE-018 — Severity: Low
**Finding.** Refusal data retention not aligned with privacy policy.
**Recommendation.** Per section 6.8; standard retention discipline.
**Owner.** ai-platform-eng + privacy, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building refusal architecture:

- [ ] **Sprint 0 — response-type framework.** Refuse / escalate / degrade matrix per section 2.5.
- [ ] **Sprint 0 — escalation destinations.** Per type; with owners and SLAs.
- [ ] **Sprint 1 — refusal templates.** Per pattern per section 3.
- [ ] **Sprint 1 — escalation structured handoff.** Context for reviewers.
- [ ] **Sprint 2 — observability.** Per-refusal logging; dashboards.
- [ ] **Sprint 2 — alerting.** Refusal-rate and queue-depth alerts.
- [ ] **Sprint 3 — refusal eval.** Set + judge; CI gate.
- [ ] **Sprint 3 — user feedback capture.** Rating; comment; this-was-wrong flag.
- [ ] **Sprint 4 — product feedback loop.** Weekly / monthly review.
- [ ] **Sprint 4 — multi-channel coordination.** If applicable.
- [ ] **Ongoing — quarterly review.** Patterns, templates, SLAs, eval quality.

For a team retrofitting:

- [ ] **Sprint 0 — audit refusal patterns.** What's currently being refused; how is it presented.
- [ ] **Sprint 1 — fix worst gaps.** Often template messages and escalation routing.
- [ ] **Sprint 2 — observability.** Get visibility.
- [ ] **Sprint 3 — eval.** Verify appropriateness.

A team that completes the sequence has refusals that earn user trust and produce product insight. A team that doesn't has refusals that erode trust and remain a blind spot.

---

## 12. References

- [guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md) — refusals are triggered by guardrail decisions.
- [policy-as-code-for-ai.md](./policy-as-code-for-ai.md) — refusal policies as code.
- [content-moderation-architecture.md](./content-moderation-architecture.md) — moderation rejection produces refusal.
- [ai-gateway-pattern.md](./ai-gateway-pattern.md) — gateway is often where refusal is emitted.
- [tool-call-authorization.md](./tool-call-authorization.md) — tool-authorization denial produces refusal.
- [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md) — scope violation produces refusal.
- [integration-architecture/human-in-the-loop-boundaries.md](../integration-architecture/human-in-the-loop-boundaries.md) — HITL integration depth.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant refusal patterns possible.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) — HITL-as-rubber-stamp anti-pattern.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — degradation patterns.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-evals.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-evals.md) — eval approach.
- Sibling repo: ai-security-reference-architecture — refusal as safety mechanism.
- "Don't Make Me Think" (Krug) — broader UX framing that informs refusal message design.
- Conversational UX research (various) — patterns for AI-system messaging.
