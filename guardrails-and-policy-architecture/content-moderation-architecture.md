# Content Moderation Architecture

> **Audience.** Architects designing the content moderation layer for an AI system. Tech leads choosing between OpenAI Moderation, Anthropic's built-in safety, AWS Bedrock Guardrails, Azure Content Safety, Lakera, custom classifiers, or LLM-as-moderator. Anyone whose moderation strategy is "the model won't produce harmful content because it's trained well." **Scope.** The *architectural* decisions — where moderation sits (pre-model / post-model / inline / HITL), vendor vs custom, performance and cost implications, multi-stage architectures. Not the engineering of moderation pipelines (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)). Not the threat-model framing (sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Content moderation in AI systems is harder than its conventional content-moderation cousin. Conventional moderation deals with user-generated content (forum posts, comments, uploaded images); patterns are mature. AI moderation deals with model-generated content plus user inputs plus retrieved content plus multi-turn conversations — a wider surface area with new failure modes.

The architectural decisions:

- **Where moderation sits.** Pre-model filters inputs before they reach the model. Post-model filters outputs before they reach users. Inline moderation uses constrained decoding so the model can't generate prohibited content in the first place. HITL routes high-risk classes to human review.
- **Vendor vs custom.** Vendor moderators (OpenAI Moderation, Anthropic Constitutional, Azure Content Safety, AWS Bedrock Guardrails, Lakera, Perspective API) are mature for general categories; custom classifiers fit domain-specific needs (medical safety beyond generic categories; financial compliance; brand-specific).
- **LLM-as-moderator.** Using an LLM to judge content. Flexible; expensive; quality varies.
- **Multi-stage architectures.** Combinations of cheap-first / expensive-second moderation.

The wrong choice in either direction is costly. Single-layer moderation leaks; over-engineered moderation costs and slows the system. The right architecture matches the workload's actual risks.

This document is opinionated about four things:

1. **Moderation is multi-stage, not single-call.** The default architecture has a cheap input filter, the model's built-in safety, and a cheap output filter; expensive LLM-as-moderator only where needed.
2. **Vendor moderation is the default; custom only when justified.** Vendor models are good enough for most general categories. Custom is justified by domain-specific needs the vendor can't address.
3. **Inline (constrained decoding) is powerful but underused.** Structured-output schemas + content constraints prevent generation of prohibited content; better than detecting after.
4. **HITL is for high-risk classes; not the primary moderation.** Human review is expensive and slow; reserve for cases where automated moderation is insufficient and the consequence justifies the cost.

Structure: (2) the four moderation positions; (3) vendor moderation options; (4) custom moderation and LLM-as-moderator; (5) multi-stage architectures; (6) performance and cost implications; (7) HITL integration; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four moderation positions

Where moderation can sit in the architecture.

### 2.1 Pre-model (input filtering)

Moderation applied to the input before it reaches the model.

```
User input → input moderation → 
  if rejected: refuse with explanation
  if accepted: → model → output
```

**What it catches.**

- Inputs requesting prohibited content.
- Inputs with prompt-injection payloads.
- Inputs containing PII / PHI the system shouldn't process.
- Off-topic / abuse inputs.

**Pros.**
- Cheap (filter runs even if input is rejected; no model call).
- Fast (input is small; moderation latency low).
- Predictable (the same input always gets the same decision).

**Cons.**
- Doesn't catch what the model generates (which may be different from what the input requested).
- Some prohibited content can be evoked without explicit input (model hallucination, retrieval contamination).

**When right.** First line of defense; combined with output moderation.

### 2.2 Post-model (output filtering)

Moderation applied to the model's output before it reaches the user.

```
User input → model → output → output moderation →
  if rejected: refuse / regenerate / redact
  if accepted: → return to user
```

**What it catches.**

- Harmful content the model generated despite intent.
- PII / PHI in the output that shouldn't be there.
- Hallucinated content (some moderators check grounding).
- Off-policy responses.

**Pros.**
- Catches what input moderation misses.
- Catches model failures.
- Can include sophisticated checks (claim grounding, citation requirements).

**Cons.**
- Adds latency to the response (user waits).
- Adds cost (one more call per response).
- If moderation rejects, the entire response is wasted (LLM tokens already spent).

**When right.** Standard pairing with input moderation; layered defense.

### 2.3 Inline (constrained decoding)

Moderation built into the model's generation; the model can't produce prohibited content because the generation is constrained.

```
User input → model with constrained decoding → 
  output (guaranteed within constraints) → return
```

**What it constrains.**

- Schema adherence (only valid JSON structures).
- Vocabulary restrictions (no use of prohibited terms).
- Format requirements (responses must include disclaimers).
- Citation requirements (claims must cite sources).

**Pros.**
- No "rejection then regeneration" cycle; correctness is guaranteed at generation.
- Lower latency than detect-and-regenerate.
- No wasted tokens.

**Cons.**
- Constraint complexity (some constraints are hard to encode).
- Not all model providers support full constrained decoding.
- May reduce model quality (the constraints limit expression).

**When right.** Structured output use cases; schema enforcement; format requirements.

### 2.4 HITL (human review)

Moderation routes specific outputs / actions to human review.

```
User input → model → output → 
  if requires review per policy: → human review → 
    if approved: → return
    if rejected: → refuse / modify
  if doesn't require review: → return
```

**What HITL catches.**

- Subjective judgments automated moderation can't make.
- High-stakes decisions (medical, financial, legal).
- Cases on the edge of policy where the right decision is unclear.

**Pros.**
- Human judgment for subjective / high-stakes cases.
- Accountability (decision has a named decider).
- Quality where automation falls short.

**Cons.**
- Slow (minutes to hours, not milliseconds).
- Expensive (human time).
- Limited capacity (reviewers are a constraint).
- Pattern can become rubber-stamping if poorly designed (per [agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) in the engineering sibling).

**When right.** High-stakes content where automated moderation is insufficient.

### 2.5 The positions together

| Position | Catches | Latency | Cost | Default fit |
| --- | --- | --- | --- | --- |
| Pre-model | Input-based threats | Low | Low | First line of defense |
| Post-model | Output failures | Medium | Medium | Second line of defense |
| Inline | Structure / format violations | Low | Embedded in model call | Use where applicable |
| HITL | Subjective / high-stakes | Hours | High | Reserve for justified cases |

Most production architectures use 2-4 of these in combination.

### 2.6 The "vendor model has safety built in" misconception

Frontier models (Anthropic Claude, OpenAI GPT, etc.) have built-in safety training. They refuse many harmful requests; their outputs are biased away from harmful content.

The built-in safety is real but not sufficient:

- It can be bypassed (prompt injection, sophisticated jailbreaks).
- It doesn't cover domain-specific requirements (medical safety, financial compliance, brand voice).
- It doesn't enforce business policy.
- It doesn't catch retrieval-contamination (the model could repeat harmful content from retrieved documents).

The architectural moderation supplements the model's built-in safety; doesn't replace it.

---

## 3. Vendor moderation options

The mature vendor landscape in 2026.

### 3.1 OpenAI Moderation

A free moderation API from OpenAI. Classifies text along multiple categories (harassment, hate, self-harm, sexual, violence, etc.).

**Pros.**
- Free.
- Mature; widely used.
- Fast.

**Cons.**
- General categories only; no domain-specific.
- US-context bias.
- Not tunable.

**When right.** General-purpose moderation; budget-conscious deployments.

### 3.2 Anthropic Constitutional AI / built-in safety

Claude models are trained with Constitutional AI; built-in safety is significant.

**Pros.**
- Bundled with the API call; no extra cost.
- Strong for general harmful content.
- Continuously improved.

**Cons.**
- Implicit; no policy-as-code visibility.
- Not configurable per use case.
- Domain-specific gaps.

**When right.** Always-on; supplemented by explicit moderation for specific needs.

### 3.3 Azure Content Safety

Microsoft's moderation service. Multiple modalities (text, image); category-based classification.

**Pros.**
- Mature.
- Multiple modalities.
- Custom category support (with training).
- Azure integration.

**Cons.**
- Costs (per-API-call pricing).
- Azure-centric.

**When right.** Azure-deployed; needs multi-modal moderation.

### 3.4 AWS Bedrock Guardrails

AWS's moderation layer for Bedrock-hosted models. Content filters, denied topics, sensitive info redaction.

**Pros.**
- Integrated with Bedrock workflow.
- AWS-native.
- Configurable.

**Cons.**
- AWS / Bedrock-specific.
- Cost.

**When right.** Bedrock-centric AI deployments.

### 3.5 Lakera Guard

Specialised for AI security; injection detection, content moderation.

**Pros.**
- AI-security focused; injection-detection is strong.
- Quick deployment.

**Cons.**
- Specialised; smaller scope than general moderation.
- Vendor risk (smaller company).

**When right.** AI security as the primary need; injection defense.

### 3.6 Perspective API (Jigsaw / Google)

Toxicity classification. Free with usage limits.

**Pros.**
- Mature; long history.
- Free.

**Cons.**
- Limited categories.
- Slower update cadence.

**When right.** Toxicity-specific moderation; budget-conscious.

### 3.7 The vendor choice

| Vendor | Best fit |
| --- | --- |
| Anthropic built-in | Always-on; bundled |
| OpenAI Moderation | General; free; supplements built-in |
| Azure Content Safety | Azure-deployed; multi-modal |
| AWS Bedrock Guardrails | Bedrock-deployed |
| Lakera | AI-security focus |
| Perspective | Toxicity-specific; budget |

For most teams: built-in (model's training) + OpenAI Moderation or equivalent (cheap explicit filter) + custom/LLM-as-moderator for domain needs.

### 3.8 The "any vendor will do" misconception

Vendor moderators differ. Recall and precision vary; the team should eval against their actual content distribution.

A vendor that performs well on general benchmarks may underperform on the team's specific content. Eval before committing.

### 3.9 The vendor's eval

When evaluating a vendor moderator:

- Test on the team's actual content samples (production samples + adversarial samples).
- Measure recall (does it catch the harmful content?) and precision (does it avoid false positives that hurt user experience?).
- Compare cost across vendors.
- Compare latency.
- Compare update cadence (how often does the vendor improve?).

The eval informs the choice; don't pick by reputation alone.

---

## 4. Custom moderation and LLM-as-moderator

When vendor moderation is insufficient.

### 4.1 Custom classifier

A bespoke classifier trained on the team's content.

**Pros.**
- Tailored to domain (medical, financial, brand-specific).
- Captures requirements vendor models don't.
- Owned; no vendor risk.

**Cons.**
- Engineering cost (training, deployment, maintenance).
- Requires labeled data.
- Quality depends on training investment.

**When right.** Specific domain needs vendor models don't cover.

### 4.2 LLM-as-moderator

Use an LLM (often a smaller / cheaper one) to judge content per a rubric.

**Pros.**
- Flexible (any rubric expressible in prompt).
- Fast to deploy.
- Quality high for well-specified rubrics.
- Adapts to new policies via prompt update.

**Cons.**
- Cost per evaluation.
- Latency.
- Stochasticity (LLMs vary).
- Same risks as any LLM call (hallucination of moderation decisions).

**When right.** Complex / subjective rubrics; rapid policy iteration.

### 4.3 The LLM-as-moderator pattern

```python
moderation_prompt = """
You are evaluating content for the [domain] AI assistant.
The content is: [content]

Check if the content violates any of these policies:
1. [Policy 1]
2. [Policy 2]
...

Return JSON: {is_violation: boolean, policy_violated: [list], reason: string, confidence: 0-1}
"""

decision = llm.call(moderation_prompt)
# Parse and act on decision
```

The prompt is the policy; the model's judgment is the verdict.

### 4.4 The LLM-as-moderator quality

LLM moderators have variance. For high-stakes decisions, two patterns mitigate:

- **Multi-judge consensus.** Multiple judges vote; majority wins.
- **Calibration against humans.** Periodic sample comparison; judge quality is monitored and corrected.

These add cost; reserve for high-stakes moderation.

### 4.5 The custom + vendor combination

Most production architectures combine:

- Vendor moderator for general categories (cheap, fast).
- Custom or LLM-as-moderator for domain-specific (more expensive, specialised).

The two layers cover different threat classes.

### 4.6 The "build the perfect moderator" trap

Some teams aim for a single moderator that catches everything. The result is typically:

- High cost (sophisticated moderator).
- High latency.
- Still doesn't catch everything.

The right pattern is layered, with specialised moderators for specialised concerns. Each layer is simple; together they cover.

### 4.7 The "we'll just prompt-engineer the model to be safe" anti-pattern

"Don't generate harmful content" in the system prompt is not moderation. It relies on the model's prompt-following; sophisticated jailbreaks defeat it.

Prompt instructions complement moderation; they don't replace it.

---

## 5. Multi-stage architectures

The standard production pattern.

### 5.1 The cheap-first / expensive-second pattern

```
Input → cheap moderation (OpenAI Moderation, Anthropic built-in) →
  if clearly fine: → model
  if clearly bad: → reject
  if uncertain: → expensive moderation (LLM-as-moderator) →
    final decision
```

Most content is handled by the cheap moderator; the expensive moderator only runs on uncertain cases. Cost is bounded; quality is high.

### 5.2 The category-routed pattern

Different categories of content go through different moderators:

```
Input → category classifier (cheap) →
  if "medical question": → medical-safety moderator →
  if "financial question": → financial-compliance moderator →
  if "general": → general moderator
```

Each moderator is specialised; total cost is controlled by category routing.

### 5.3 The input + output sandwich

Standard pattern: moderation on both ends.

```
Input → input moderation → model → output moderation → User
```

Each end catches different threats; together they cover more.

### 5.4 The input + output + tool moderation

For agent systems, add tool-level moderation:

```
Input → input moderation → 
  agent loop with tool moderation per call →
  output → output moderation → User
```

Per [tool-call-authorization.md](./tool-call-authorization.md), tool moderation is a separate concern from content moderation but they coordinate.

### 5.5 The "everything through LLM-as-moderator" anti-pattern

Some teams use LLM-as-moderator for all inputs and outputs. Cost explodes; latency suffers.

Mitigation: tiered moderation per section 5.1. LLM-as-moderator only for cases the cheap moderators flag as uncertain.

### 5.6 The "moderation fail-mode" coordination

When any moderation layer fails:

- The architecture decides: fail closed (block content) vs fail open (allow with warning) vs degraded (continue with the layers that work).

Default: fail closed for high-stakes; fail open for low-stakes with clear logging. Per [guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md) section 6.5.

### 5.7 The multi-stage eval

Each stage's quality is measured:

- Layer 1 (cheap): catches most cases; false-positive rate matters.
- Layer 2 (expensive): catches what layer 1 misses; false-negative rate matters.
- Overall: combined precision and recall.

Eval per layer; eval combined; tune.

---

## 6. Performance and cost implications

The operational layer.

### 6.1 Per-call cost

| Moderation type | Cost per call |
| --- | --- |
| Anthropic built-in | Included with model call |
| OpenAI Moderation | Free |
| Azure Content Safety | $0.001-0.01 per call (varies by category) |
| AWS Bedrock Guardrails | Varies; bundled with model in some pricing |
| Custom classifier | Infra cost (negligible per call); training cost |
| LLM-as-moderator (Haiku) | $0.001-0.005 per call (depending on length) |
| LLM-as-moderator (Sonnet) | $0.005-0.05 per call |

For a feature with high volume, moderation cost can add up.

### 6.2 Per-call latency

| Moderation type | Latency |
| --- | --- |
| Anthropic built-in | Embedded; no added latency |
| OpenAI Moderation | 50-200ms |
| Azure Content Safety | 100-300ms |
| Custom classifier (in-process) | 10-50ms |
| LLM-as-moderator (Haiku) | 200-500ms |
| LLM-as-moderator (Sonnet) | 500-2000ms |

For interactive features, the latency adds to response time.

### 6.3 The cost-latency-quality tradeoff

Per moderation requirement:

- Cheap and fast: vendor moderators (OpenAI, Anthropic built-in).
- Expensive and slow: LLM-as-moderator.
- Tradeoff: quality.

The decision: which moderation is needed for which content; reserve expensive moderators for cases that justify.

### 6.4 The total moderation budget

For a typical AI feature:

- Input moderation (per call): ~$0-0.005, 0-200ms.
- Built-in (per model call): no extra cost, no extra latency.
- Output moderation (per call): ~$0-0.005, 0-200ms.
- Total: ~$0-0.01, 0-400ms.

For high-stakes / domain-specific with LLM-as-moderator: add $0.005-0.05 and 500-2000ms.

For a 100k-request/day feature with moderate moderation: ~$1-10/day in moderation cost.

For high-stakes features with LLM-as-moderator: ~$50-500/day.

The cost is generally manageable; the latency may force architectural choices (caching moderation decisions, parallel moderation, etc.).

### 6.5 The caching of moderation decisions

Identical inputs can use cached moderation decisions:

- Input hash → cached decision.
- TTL appropriate to the threat (typically hours to days).
- Per-tenant scoping.

The cache reduces cost; the TTL bounds staleness risk.

### 6.6 The parallel moderation

For latency reduction, moderation calls can run in parallel:

- Input moderation runs in parallel with prompt-construction.
- Output moderation runs as soon as the first output tokens arrive (streaming-aware).

The architecture needs to support concurrent calls; the engineering investment is moderate.

### 6.7 The DR for moderation services

If a moderation vendor is down:

- Fail closed (block all content) or fail open (allow with warning) per the architecture's fail-mode decision.
- Fallback to alternative vendor if available.
- Alert.

For features where moderation can't fail, plan for fallback before the incident.

---

## 7. HITL integration

The human-in-the-loop position in moderation.

### 7.1 What HITL handles

- Subjective decisions (is this content "professional enough"?).
- High-stakes content where the consequence of automation failure is severe (medical, legal).
- Edge cases automated moderators flag as uncertain.
- New policy areas where automation hasn't been trained.

### 7.2 What HITL doesn't handle

- High-volume general content (capacity exceeded).
- Time-sensitive content (latency too high).
- Cases where automated moderation is clearly sufficient.

### 7.3 The HITL architecture

```
Content flagged for review → queue → 
  reviewer assigned → decision → 
  approved: → release / continue
  rejected: → refuse / modify / escalate
```

The queue, the reviewer assignment, the decision capture — these are infrastructure the team builds.

### 7.4 The reviewer capacity

A typical reviewer can process:

- Simple text content: 60-120 items per hour.
- Complex content (clinical, legal): 10-30 items per hour.

For a feature flagging 1% of 100k daily requests = 1,000 items per day. At 30 items/hour, ~33 reviewer-hours per day = ~4 FTE reviewers (with breaks, training, queue smoothing). Significant cost.

Mitigations:

- Tiered review: high-priority gets human; low-priority gets stricter automation.
- Pre-filter to reduce items reaching humans.
- Decision-templates to speed reviews.

### 7.5 The HITL latency

User-visible latency:

- Sync HITL: user waits for review. Unacceptable for most use cases.
- Async HITL: user gets immediate response (often "your request is being reviewed"); decision delivered later. Common for moderation; UX matters.
- Background HITL: review happens in background; doesn't affect user; used for analytics, training data improvement.

### 7.6 The HITL UX

The reviewer's interface matters:

- Per-item context (what was being reviewed; relevant supporting info).
- Decision options (approve / reject / modify / escalate / refer to senior reviewer).
- Decision rationale capture (for audit and training).
- Time-budget per review (designed for thoughtful review, not throughput).

### 7.7 The HITL anti-pattern (rubber stamp)

Per [agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) #8 in the engineering sibling. If reviewer approval rate is > 95% and rejection rate is < 5%, the HITL may be rubber-stamping; the value isn't being added.

Mitigations:

- Random-sample audit (does the reviewer catch what they should?).
- Reviewer training and feedback.
- Reduce review volume (pre-filter; auto-approve high-confidence cases).

### 7.8 The HITL integration with policy

HITL routing is itself a policy decision (per [policy-as-code-for-ai.md](./policy-as-code-for-ai.md)):

```rego
review_required {
    input.content_classification == "medical_advice"
    input.confidence < 0.85
}

review_required {
    input.user_role == "patient"
    input.action == "modify_appointment"
}
```

The policy decides; the application routes accordingly.

---

## 8. Worked Meridian example

Meridian's moderation architecture.

### 8.1 The stack

- **Pre-model:** Anthropic built-in (always); OpenAI Moderation API for explicit category checking; custom medical-safety classifier (LLM-as-moderator with claude-haiku-4-5).
- **Inline:** Structured output schemas; format constraints (citation required, disclaimer required where applicable).
- **Post-model:** OpenAI Moderation API; custom medical-safety classifier; PHI leak detection (Presidio + LLM judge).
- **HITL:** Care managers review flagged high-stakes content (medical advice, care plan changes).

### 8.2 The per-feature configuration

**Care-coordinator:**

- Input: OpenAI Moderation + medical-safety classifier.
- Inline: structured output (citation required for clinical claims; disclaimer for medical advice).
- Output: OpenAI Moderation + medical-safety + PHI leak.
- HITL: routes if content_classification = medical_advice AND confidence < 0.85.

**Patient-summary:**

- Input: minimal (generated internally, not user input).
- Inline: structured SOAP-note schema.
- Output: PHI leak check; format validation.
- HITL: no (workflow is automated reporting; reviewer in the downstream UI flow).

**Analytics-copilot:**

- Input: OpenAI Moderation; SQL-injection detection.
- Inline: SQL-generation schema (constrained to authorised tables/columns).
- Output: format validation; PII redaction (analytics queries shouldn't return PII).
- HITL: no.

**Patient-API copilot:**

- Input: OpenAI Moderation.
- Inline: code-format constraints.
- Output: OpenAI Moderation; check for accidental PHI in generated code examples.
- HITL: no.

### 8.3 The moderation cost

Per request, typical:

- Care-coordinator: ~$0.005 in moderation (Mod API free; medical-safety classifier ~$0.003; PHI ~$0.002).
- Patient-summary: ~$0.002 (PHI check primarily).
- Analytics-copilot: ~$0.002.
- Patient-API copilot: ~$0.001.

Total moderation cost ~3-5% of total request cost. Acceptable.

### 8.4 The moderation latency

Per request, typical:

- Pre-model: ~150ms (parallel input checks).
- Post-model: ~200ms (parallel output checks).
- HITL (when triggered): ~5-30 minutes (asynchronous; user is notified).

Adds ~350ms to interactive features; acceptable within the team's latency SLO.

### 8.5 The HITL volume

Care-coordinator flags ~3% of outputs for review:

- ~50k daily requests × 3% = ~1,500 reviews per day.
- Care managers handle this with ~3 FTE.

### 8.6 The medical-safety classifier

A specific concern: generic moderators don't catch medical-safety issues (inappropriate medication recommendations, misdiagnosis, unsafe advice). The team built a custom LLM-as-moderator with a medical-safety rubric:

- Trained against a curated set of safe/unsafe medical content.
- Calibrated against physician judgments quarterly.
- Catches ~88% of unsafe content the generic moderators miss.

### 8.7 The incident history

**Q2-25.** Output moderation false-negative: the system generated medical advice without the required disclaimer. The medical-safety classifier missed it; the inline schema didn't have a strict disclaimer requirement.

Mitigations: schema tightened to require disclaimer field; eval cases added; medical-safety classifier improved.

**Q4-25.** Input moderation false-positive: a clinical question containing the word "overdose" was flagged as self-harm content; the user got a refusal for a legitimate clinical question.

Mitigations: medical-safety classifier rebalanced; context-aware moderation (clinical role + medical-context vs general user vs medical-context).

**Q1-26.** HITL queue backlog: a marketing push surged care-coordinator traffic; HITL queue grew to 4-hour backlog; users waited for HITL-required responses.

Mitigations: auto-approval logic improved for high-confidence cases; queue capacity scaled; alerting on queue depth added.

### 8.8 What worked

- **Layered architecture.** Built-in + vendor + custom; catches different concerns.
- **Inline constraints.** Schema-enforced disclaimers and citations are bulletproof.
- **Medical-safety classifier.** Domain-specific where general moderators fell short.
- **Async HITL.** User experience acceptable; reviewers have time for thoughtful decisions.

### 8.9 What didn't work initially

- **Built-in only.** Initial deployment relied on Anthropic's built-in safety; insufficient for medical-specific content. Layered architecture added.
- **All output reviewed.** Initial post-moderation ran a full LLM judge on every output; cost was 5× sustainable. Tiered architecture (cheap first; expensive only for uncertain) reduced 80% of cost.
- **Sync HITL.** First HITL attempt blocked user response; UX intolerable. Async architecture (user gets response; HITL happens in background; if review changes outcome, user is updated) deployed.

---

## 9. Anti-patterns

### 9.1 "Built-in safety only"

The team relies on the model's training; no explicit moderation; gaps in domain-specific.

**Corrective.** Layered moderation per section 5; supplements built-in.

### 9.2 "LLM-as-moderator everywhere"

Every input and output uses LLM-as-moderator. Cost explodes; latency suffers.

**Corrective.** Tiered moderation per section 5.1; LLM-as-moderator for uncertain only.

### 9.3 "No output moderation"

Input is filtered; output isn't. Model failures and retrieval contamination reach users.

**Corrective.** Output moderation per section 2.2.

### 9.4 "HITL as primary moderation"

The team routes everything to HITL. Reviewer capacity exceeded; latency excessive; cost prohibitive.

**Corrective.** HITL for high-stakes only per section 7.

### 9.5 "Sync HITL blocking response"

User waits for HITL approval. UX intolerable.

**Corrective.** Async HITL per section 7.5.

### 9.6 "Single-vendor moderation without eval"

The team picks a vendor based on reputation; doesn't eval against their content; quality is unknown.

**Corrective.** Per-vendor eval per section 3.9.

### 9.7 "Prompt-engineering safety as moderation"

"Don't generate harmful content" in system prompt; no explicit moderation. Defeated by sophisticated jailbreaks.

**Corrective.** Per section 4.7; prompt engineering complements; doesn't replace.

### 9.8 "Moderation decisions not logged"

Decisions happen; no audit trail; can't investigate false positives / negatives.

**Corrective.** Per-decision logging per [policy-as-code-for-ai.md](./policy-as-code-for-ai.md) section 7.6.

---

## 10. Findings (sprint-assignable)

### ARCH-MOD-001 — Severity: Critical
**Finding.** No explicit moderation; reliance on built-in safety only; domain-specific gaps unaddressed.
**Recommendation.** Layered architecture per section 5; vendor moderator + domain-specific.
**Owner.** architecture + security, sprint N+1.

### ARCH-MOD-002 — Severity: Critical
**Finding.** Output moderation absent; model failures reach users.
**Recommendation.** Output moderation per section 2.2.
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-MOD-003 — Severity: Critical
**Finding.** Moderation decisions not logged; false-positive / false-negative analysis impossible.
**Recommendation.** Per-decision logging per section 9.8.
**Owner.** ai-platform-eng + ops, sprint N+1.

### ARCH-MOD-004 — Severity: High
**Finding.** Domain-specific content (medical, financial, legal) handled only by general moderators; specific risks unaddressed.
**Recommendation.** Custom or LLM-as-moderator for domain per section 4.
**Owner.** ai-platform-eng + domain experts, sprint N+2.

### ARCH-MOD-005 — Severity: High
**Finding.** HITL as primary moderation; reviewer capacity exceeded; backlogs grow.
**Recommendation.** HITL for high-stakes only per section 7.1.
**Owner.** product + ai-platform-eng, sprint N+2.

### ARCH-MOD-006 — Severity: High
**Finding.** Sync HITL blocking response; UX intolerable.
**Recommendation.** Async HITL per section 7.5.
**Owner.** product + ai-platform-eng, sprint N+2.

### ARCH-MOD-007 — Severity: High
**Finding.** Moderation cost > 10% of feature cost; not tiered.
**Recommendation.** Tiered moderation per section 5.1; cheap first.
**Owner.** ai-platform-eng + finance, sprint N+2.

### ARCH-MOD-008 — Severity: Medium
**Finding.** Vendor moderator chosen by reputation; not eval'd against team's content.
**Recommendation.** Per-vendor eval per section 3.9.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MOD-009 — Severity: Medium
**Finding.** Inline (constrained decoding) not used despite structured output use cases.
**Recommendation.** Schema enforcement per section 2.3.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MOD-010 — Severity: Medium
**Finding.** HITL reviewer approval rate > 95%; rubber-stamp risk.
**Recommendation.** Per section 7.7; audit, training, pre-filter.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-MOD-011 — Severity: Medium
**Finding.** Moderation latency adds > 30% to response time; not parallelised.
**Recommendation.** Parallel moderation per section 6.6.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MOD-012 — Severity: Medium
**Finding.** Moderation cache absent; identical inputs re-evaluated.
**Recommendation.** Cache per section 6.5.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-MOD-013 — Severity: Medium
**Finding.** Moderation eval not run periodically; quality drifts.
**Recommendation.** Per-stage eval per section 5.7.
**Owner.** ai-platform-eng + security, sprint N+4.

### ARCH-MOD-014 — Severity: Medium
**Finding.** Vendor fail-mode unclear; outage behaviour ad-hoc.
**Recommendation.** Fail-mode per section 6.7; documented.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-MOD-015 — Severity: Low
**Finding.** Multi-vendor failover not in place; single-vendor outage takes feature down.
**Recommendation.** Per section 6.7; alternative provider configured.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-MOD-016 — Severity: Low
**Finding.** Reviewer interface lacks per-item context; reviews take longer than necessary.
**Recommendation.** UX investment per section 7.6.
**Owner.** product, sprint N+5.

### ARCH-MOD-017 — Severity: Low
**Finding.** Moderation policies not documented; new engineers don't know what's checked.
**Recommendation.** Policy documentation per [policy-as-code-for-ai.md](./policy-as-code-for-ai.md).
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-MOD-018 — Severity: Low
**Finding.** Cross-feature moderation pattern not standardised; reinvention per feature.
**Recommendation.** Shared moderation library; central pattern.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building moderation:

- [ ] **Sprint 0 — threat model.** Per sibling security repo; what categories matter?
- [ ] **Sprint 0 — vendor eval.** Evaluate built-in + cheap vendor moderators against team content.
- [ ] **Sprint 1 — input moderation.** Cheap vendor at the gateway.
- [ ] **Sprint 1 — output moderation.** Cheap vendor at the gateway.
- [ ] **Sprint 1 — fail-mode definition.** Per stage.
- [ ] **Sprint 2 — inline constraints.** Schema, citation, disclaimer requirements.
- [ ] **Sprint 2 — observability.** Per-decision logging; per-stage metrics.
- [ ] **Sprint 3 — domain-specific.** Custom or LLM-as-moderator for domain needs.
- [ ] **Sprint 3 — tiered architecture.** Cheap first; expensive only for uncertain.
- [ ] **Sprint 4 — HITL.** For high-stakes only; async; with proper UX.
- [ ] **Sprint 4 — caching.** Per identical-input caching.
- [ ] **Sprint 5 — multi-vendor failover.** For resilience.
- [ ] **Ongoing — quarterly eval.** Per-vendor quality; per-stage performance; tune.

For a team with existing moderation:

- [ ] **Sprint 0 — audit.** What moderation exists; per stage; per feature.
- [ ] **Sprint 1 — fix worst gaps.** Often output moderation or domain-specific.
- [ ] **Sprint 2 — tiering.** If LLM-as-moderator on everything.
- [ ] **Sprint 3 — HITL review.** Rubber-stamp check; capacity check.

A team that completes the sequence has moderation that's effective, cost-controlled, and operationally sound. A team that doesn't has gaps that incidents reveal.

---

## 12. References

- [guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md) — moderation as a guardrail category; placement.
- [ai-gateway-pattern.md](./ai-gateway-pattern.md) — gateway as the home of input/output moderation.
- [tool-call-authorization.md](./tool-call-authorization.md) — tool moderation interacts with content moderation.
- [policy-as-code-for-ai.md](./policy-as-code-for-ai.md) — moderation policies as code.
- [refusal-and-escalation-design.md](./refusal-and-escalation-design.md) — what happens when moderation rejects.
- [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md) — retrieval contamination as a moderation concern.
- [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) — provenance enables claim-grounding moderation.
- [integration-architecture/](../integration-architecture/) — HITL boundary integration.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) — HITL-as-rubber-stamp anti-pattern.
- Sibling repo: [ai-engineering-reference-architecture/eval-engineering/llm-as-judge-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/llm-as-judge-patterns.md) — LLM-as-judge patterns apply to LLM-as-moderator.
- Sibling repo: ai-security-reference-architecture — threat model that informs moderation.
- OpenAI Moderation API documentation.
- Anthropic Constitutional AI documentation.
- Azure Content Safety, AWS Bedrock Guardrails — vendor offerings.
- Lakera Guard — AI-security focused moderation.
- Perspective API (Jigsaw / Google) — toxicity classification.
- Microsoft Presidio — PII detection (often paired with moderation).
