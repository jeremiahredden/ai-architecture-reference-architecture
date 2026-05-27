# Few-Shot vs Fine-Tune vs System Prompt

> **Audience.** Architects whose workload needs to specialize a model and they're choosing between adding few-shot examples, fine-tuning, or expanding the system prompt. Tech leads whose few-shot examples have grown to 30 and they're wondering if that's reasonable. Anyone whose vendor pitched fine-tuning and they're not sure if it's right. **Scope.** The *architectural* framework for when each approach fits: few-shot examples (flexible, expensive per call), fine-tune (cheap per call, slow to update), system prompt (cheap, hits ceiling); the decision tree; migration paths between them. Not the engineering of fine-tuning (see sibling [model-lifecycle/fine-tuning-operations.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/fine-tuning-operations.md)). Not per-tenant fine-tuning (see [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

When a workload needs the model to do something specific — produce a particular output format, follow domain conventions, demonstrate specialized reasoning — three approaches:

- **System prompt.** Tell the model what to do in instructions.
- **Few-shot examples.** Show the model examples of the desired behavior.
- **Fine-tune.** Train the model on the specific behavior.

Each has properties:

- **System prompt:** cheapest, fastest to update, lowest capability ceiling.
- **Few-shot:** flexible, expensive per call (examples consume tokens every call), moderate ceiling.
- **Fine-tune:** cheap per call (no examples needed), slow to update, highest ceiling but heaviest operational cost.

Most teams pick one and move on. The architectural question is: which is right for this workload?

The answer is often a combination. The wrong answer is "we picked one because the vendor's pitch was compelling."

This document covers the decision framework and migration paths.

This document is opinionated about four things:

1. **System prompt is the default; escalate as needed.** Most workloads can start with system prompt; few-shot and fine-tune for specific needs.
2. **Few-shot is the most underused.** Many workloads that "need fine-tuning" are actually solvable with well-chosen few-shot examples.
3. **Fine-tune is the most overused.** Vendor pitches favor it; operational reality is heavy. Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).
4. **The choice is per-workload, not per-platform.** Different workloads may use different approaches.

Structure: (2) the three approaches characterized; (3) the decision framework; (4) when system-prompt suffices; (5) when few-shot is right; (6) when fine-tune is right; (7) migration paths; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three approaches characterized

What each is.

### 2.1 System prompt

Instructions in the system prompt:

```
You are a clinical AI assistant. Respond in this format:
- Summary (2 sentences)
- Key findings (bulleted)
- Recommended actions (numbered)
- Citations [source: ...]

For clinical decisions, defer to the clinician's judgment.
```

The model follows the instructions.

**Properties.**
- Cost: pays per-call for prompt tokens (typically 500-2000 tokens).
- Update: change the prompt; deploy.
- Ceiling: model's instruction-following capability.
- Capability: handles structured behavior; clear refusal patterns; format adherence.

### 2.2 Few-shot

Examples in the prompt:

```
System prompt with instructions.

Examples:
Example 1:
Input: "Patient with T2DM, A1c 8.5, on metformin 1000mg BID."
Output: {
  summary: "T2DM uncontrolled despite max metformin.",
  key_findings: ["A1c above target", "Metformin max dose"],
  ...
}

Example 2: ...
Example 3: ...

Now respond to this:
Input: <current input>
```

The model patterns its output on examples.

**Properties.**
- Cost: pays per-call for examples (typically 3-15 examples × 200-500 tokens = 600-7500 tokens of examples).
- Update: change the examples; deploy.
- Ceiling: higher than system prompt (examples convey nuance instructions can't).
- Capability: pattern-matching workloads; format-by-example; nuanced reasoning.

### 2.3 Fine-tune

Train the model on examples:

- Train: prepare 1000s of (input, output) pairs; train the model.
- Result: a model that's specialized for the workload.

**Properties.**
- Cost: cheap per call (no examples needed in prompt).
- Update: re-train (days to weeks); operational lifecycle.
- Ceiling: highest if data is good; can match per-workload performance of a much larger model.
- Capability: specialized output; high reliability on the trained pattern.

### 2.4 The comparison table

```
Dimension                System Prompt   Few-Shot           Fine-Tune
─────────────────────────────────────────────────────────────────────
Setup time              Hours           Days (collect ex.)  Weeks-months
Per-call cost           Low             Medium-High         Lowest
Update speed            Hours           Hours               Days-weeks
Update cost             Free            Free                Per-update cost
Capability ceiling      Lowest          Medium              Highest (for trained task)
Operational burden      Lowest          Low                 Highest (lifecycle)
Variance across workloads Adapts well   Adapts (within ex.) Specialized
```

Each appropriate for different workloads.

### 2.5 The "combine all three" option

Often the best:

- System prompt: defines structure / identity / refusal.
- Few-shot examples: clarify output style.
- Optional fine-tune: for very specialized tasks if needed.

Layered.

### 2.6 The cost-per-call comparison

For 5000 calls/day:

```
System-prompt only (1500 tokens):
  Per call: ~$0.005 (Sonnet)
  Per day: $25
  
With 10 few-shot examples (1500 + 5000 = 6500 tokens):
  Per call: ~$0.020 (4x)
  Per day: $100
  
Fine-tune (cheaper model; smaller prompt; 800 tokens):
  Per call: ~$0.0008 (self-hosted Llama 8B)
  Per day: $4
  Plus fine-tune infrastructure: ~$3k/month
```

Fine-tune wins at high volume; system prompt at low.

### 2.7 The "what is each really for" summary

- **System prompt:** behavior and structure.
- **Few-shot:** pattern recognition and style.
- **Fine-tune:** specialized capability at scale.

---

## 3. The decision framework

How to choose.

### 3.1 The decision tree

```
Question 1: Does the workload need specialized behavior?
  Yes → continue.
  No → use generic model with system prompt only.

Question 2: Can the system prompt clearly describe the behavior?
  Yes → system prompt; possibly with few-shot examples for style.
  No → continue.

Question 3: Can few-shot examples teach the behavior?
  Yes → few-shot (with system prompt).
  No → continue.

Question 4: Is there enough data to fine-tune AND is volume high enough to justify the operational cost?
  Yes → fine-tune.
  No → revisit; may need stronger model or different approach.
```

### 3.2 The "can the system prompt describe it" test

Try writing the behavior as instructions:

- "Respond in JSON format with these fields."
- "Refuse if the patient is critically ill; instead escalate to physician."
- "Use formal medical terminology."

If clear instructions exist and the model follows them: system prompt suffices.

If the behavior is hard to describe in words: few-shot or fine-tune.

### 3.3 The "few-shot teaches better than instructions" test

For some workloads:

- Words: "Use the company's specific format."
- Few-shot: actual examples of the company's format.

Examples convey nuance words can't. Few-shot can teach what instructions can't.

### 3.4 The "fine-tune needed" indicators

Fine-tune is justified when:

- High volume (per-call cost matters more than fine-tune cost).
- Per-call latency very tight (smaller fine-tuned model faster).
- Specific structured output where reliability matters (fine-tune more reliable).
- Domain-specific reasoning that doesn't generalize from few-shot.

### 3.5 The "we tried few-shot and it works" outcome

For most "we want specialized behavior" cases:

- Try few-shot first.
- If it works at quality bar: done.
- If not: consider fine-tune.

Few-shot often suffices.

### 3.6 The "data quality matters for fine-tune" gate

Fine-tune requires high-quality data:

- 100-1000 (input, output) pairs.
- Curated.
- Validated.

Without this, fine-tune produces bad output. Few-shot doesn't have this constraint.

### 3.7 The "volume / cost crossover" calculation

When fine-tune is worth it:

```
hosted_cost = volume × per_call_cost_hosted
fine_tune_cost = volume × per_call_cost_self_hosted + operational_overhead

crossover when hosted_cost = fine_tune_cost
```

For low-cost workloads ($0.001 per call): fine-tune crossover at very high volume.

For high-cost workloads ($0.10 per call): crossover at moderate volume.

Compute per-workload.

### 3.8 The "this workload is changing" test

If the workload's requirements change frequently:

- System prompt: change instantly.
- Few-shot: change examples; re-deploy.
- Fine-tune: re-train; weeks.

Frequent change → prefer system prompt or few-shot.

### 3.9 The "the model isn't capable enough" outcome

If even fine-tune doesn't reach quality bar:

- Try larger model.
- Try architectural change (different agent flow; different RAG).

Fine-tune isn't always the answer.

---

## 4. When system-prompt suffices

The default case.

### 4.1 The "the model already does it" signal

Test: ask the model directly via system prompt:

```
You are X. Do Y.
```

If response quality is acceptable: done. No few-shot or fine-tune needed.

### 4.2 The behavior types that suit system prompt

- Identity / role definition.
- Refusal policy.
- Format instructions ("respond in JSON with these fields").
- Tool-use guidance.
- High-level behavior ("be concise" / "be detailed").

These are described well in words.

### 4.3 The instruction-following limit

System prompt fails when:

- Behavior is too nuanced for words.
- Behavior depends on edge cases not described.
- Quality requires example demonstration.

For these: escalate.

### 4.4 The "we keep adding instructions" smell

When the system prompt grows >2000 tokens with elaborate edge cases:

- Diminishing returns.
- Consider few-shot examples instead.
- Or refactor the system prompt.

Cross-link to [system-prompt-architecture.md](./system-prompt-architecture.md).

### 4.5 The cost benefit

System prompt is cheap:

- ~1500 tokens fixed.
- Per-call cost: ~$0.005 (Sonnet).
- Volume-scaled: still cheap.

If quality is acceptable: stop here.

### 4.6 The agility benefit

System prompt is fast to change:

- Edit; eval; deploy.
- 1-2 hours.

For iterating on requirements: fast.

### 4.7 The "instruction following has gotten really good" trend

In 2026, frontier models are excellent at instruction following:

- Many workloads that needed few-shot in 2024 are fine with system prompt in 2026.
- Re-evaluate annually.

Models keep improving; prompt-only suffices for more cases.

---

## 5. When few-shot is right

Examples teach what instructions can't.

### 5.1 The pattern-matching workloads

Few-shot excels at:

- "Respond in the same style as these examples."
- "Classify like these examples."
- "Extract entities like these examples."

The pattern is conveyed by the examples.

### 5.2 The format-by-example

Some formats are hard to describe in words:

```
Examples of output format:

Patient_ID | Status | Recommendation
P12345    | Stable | Continue current plan
P12346    | Critical | Escalate; ICU consult
P12347    | Stable | Routine follow-up
```

Few-shot conveys the format without needing format instructions.

### 5.3 The nuanced-reasoning examples

For workloads where reasoning style matters:

- Step-by-step examples.
- Demonstrate consideration of edge cases.
- Show appropriate level of detail.

Model patterns its reasoning on examples.

### 5.4 The example selection strategy

For good few-shot:

- Cover the workload's distribution.
- Include edge cases.
- Diverse but representative.
- 3-15 examples typical.

Cross-link to [ai-engineering-reference-architecture / prompt-engineering / few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md).

### 5.5 The example-storage

Few-shot examples are part of the prompt:

- Stored as prompt components.
- Versioned.
- Cross-link to [system-prompt-architecture.md §4](./system-prompt-architecture.md).

### 5.6 The "we have 30 examples" warning

If few-shot grows large:

- 30+ examples is a lot.
- Cost per call rises.
- Diminishing returns past ~10-15.

Consider:

- Subset based on query (dynamic few-shot).
- Fine-tune instead.

### 5.7 The dynamic-few-shot pattern

Instead of all examples in prompt:

- Retrieve most-relevant examples (semantic similarity).
- Include top-K in prompt.

```python
def get_relevant_examples(query, examples_kb):
    similar = semantic_search(query, examples_kb, top_k=5)
    return similar
```

Like RAG but for examples.

### 5.8 The example-eval-and-curation

Examples themselves need eval:

- Verify each example is correct.
- Verify together they convey the pattern.
- Re-curate as workloads evolve.

### 5.9 The few-shot vs fine-tune crossover

Few-shot wins when:

- Workload requirements change.
- Examples can fit in prompt (< 15 needed).
- Volume modest.

Fine-tune wins when:

- Workload is stable.
- Many examples needed (> 50; few-shot can't fit).
- Volume high.

---

## 6. When fine-tune is right

The cases where fine-tune justifies.

### 6.1 The cost-driven case

Very high volume with established workload:

- 1M+ calls/day.
- Per-call cost matters more than fine-tune cost.
- Workload stable enough to justify training.

Fine-tune smaller model (Llama 3 70B); operate self-hosted.

Cross-link to [model-strategy/build-vs-buy-decision.md §4](../model-strategy/build-vs-buy-decision.md).

### 6.2 The latency-driven case

Very tight latency requirements:

- Voice / real-time interfaces (P99 < 300ms).
- Frontier models too slow.
- Fine-tuned smaller model fast enough.

### 6.3 The capability-driven case

For workloads where:

- Quality with frontier model + few-shot still below bar.
- Fine-tune produces meaningfully better results.

Run eval; verify fine-tune improvement.

### 6.4 The structured-output case

For structured output where reliability matters:

- Specific JSON schema with low malformatting tolerance.
- Specific medical coding workflow.

Fine-tune produces near-100% reliability (vs few-shot at 95-98%).

### 6.5 The data prerequisite

Fine-tune requires:

- 1000+ high-quality (input, output) pairs.
- Curated; reviewed.
- Representative.

Without good data: fine-tune fails. Few-shot doesn't have this constraint.

### 6.6 The operational cost

Fine-tune has lifecycle:

- Training infrastructure.
- Eval per model version.
- Deployment.
- Drift monitoring.

Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md §5](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

Operational cost: ~0.5-1 FTE for actively-maintained fine-tunes.

### 6.7 The "vendor pitched fine-tune" caution

Vendors profit from fine-tune services:

- Custom training fees.
- Inference cost premium.

Diagnose whether the workload actually needs it; resist the pitch unless justified.

### 6.8 The "fine-tune and ship the smaller model" pattern

Fine-tune a smaller model:

- Llama 3 8B / Qwen 2 7B / Mistral 7B.
- Easier to host; cheaper per call.
- After fine-tune: comparable capability to frontier on specific task.

### 6.9 The "fine-tune adapters" alternative

LoRAs (Low-Rank Adaptation):

- Smaller adapters; cheaper to train.
- Multiple adapters on shared base model.
- Less operational overhead.

Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md §6](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

For some cases: LoRA is the middle ground.

---

## 7. Migration paths

When you need to move between approaches.

### 7.1 System-prompt → few-shot

When system prompt isn't enough:

- Add few-shot examples to the prompt.
- Keep system prompt as the structural backbone.
- Re-eval; verify quality improvement.

Migration: hours; low risk.

### 7.2 Few-shot → fine-tune

When few-shot has grown unmanageable:

- Collect data from few-shot history (the examples + actual inputs/outputs).
- Curate; validate.
- Fine-tune.
- Compare quality + cost vs few-shot.
- If wins: migrate.

Migration: weeks; moderate risk.

### 7.3 Fine-tune → larger model

When fine-tune isn't enough:

- Try larger frontier model with few-shot.
- May exceed fine-tune quality.
- Higher per-call cost but lower operational overhead.

Migration: weeks; moderate risk.

### 7.4 Few-shot → system-prompt (downgrade)

If few-shot is overkill:

- Try without examples.
- Eval-driven verification.
- Maintain quality.

Migration: easy.

### 7.5 Fine-tune → few-shot (downgrade)

If fine-tune isn't justifying its overhead:

- Switch to frontier model + few-shot.
- Per-call cost higher; lifecycle cost lower.
- Net may favor downgrade.

Migration: easy in terms of code; new evals needed.

### 7.6 The cross-direction migration

Some workloads migrate over their lifetime:

- System prompt → few-shot (as requirements solidify).
- Few-shot → fine-tune (as volume grows).
- Fine-tune → system prompt (if model improves enough).

Re-evaluate periodically.

### 7.7 The hybrid coexistence

For some workloads, multiple approaches simultaneously:

- Fine-tune base model with general knowledge.
- System prompt with feature-specific structure.
- Few-shot for nuanced cases.

Composable.

---

## 8. Worked Meridian example

Meridian's per-workload choices.

### 8.1 The Care Coordinator workload

Approach: **System prompt + few-shot**.

System prompt: identity / behavior / refusal / format / tools.

Few-shot: 8 examples covering common clinical scenarios (new patient, follow-up, escalation, referral, etc.).

Why:

- Workload diverse (many clinical scenarios).
- Examples convey nuance.
- Volume moderate (5k tasks/day); few-shot cost acceptable.

Cost: ~$25 per task ($30k/month) — manageable.

### 8.2 The Patient API chat workload

Approach: **System prompt only**.

System prompt: identity / behavior / format.

No few-shot.

Why:

- Workload conversational; few-shot doesn't add much.
- Volume high (12k turns/day); few-shot cost would balloon.
- Quality acceptable with prompt-only.

Cost: ~$10 per conversation ($3k/month).

### 8.3 The document classification workload

Approach: **Fine-tune (Llama 3 70B self-hosted)**.

Why:

- Very high volume (40k docs/day).
- Workload stable (classification taxonomy).
- Cost critical (per-call $0.0001 vs $0.025 hosted).
- Quality matched (eval verifies).

Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md §8](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

Cost: $5k/month infrastructure + $4k/month ops vs $25k/month hosted alternative.

### 8.4 The billing-code workflow (specialty practice tenant)

Approach: **Fine-tune (Llama 3 8B with LoRA)**.

Why:

- Specific workflow with reliability requirement.
- Specialty practice had high volume.
- Eval showed fine-tune improves structured output 99.4% (vs 85% few-shot).

Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md §8.4](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

### 8.5 The clinical decision support workload

Approach: **System prompt + few-shot + extensive retrieval**.

Why:

- Safety-critical; needs comprehensive reasoning.
- Few-shot demonstrates good reasoning.
- Retrieval provides clinical evidence base.

Not fine-tuned: workload evolves; quality with frontier + few-shot is acceptable; fine-tune lifecycle would risk regression.

### 8.6 The "we tried few-shot for billing-code" first

For the billing-code workflow:

- Started with 15 few-shot examples; quality 85%.
- Added more examples; diminishing returns.
- Migrated to fine-tune; quality 99.4%.

The migration was justified by the eval result.

### 8.7 The "we tried fine-tune for Patient chat" exploration

A team explored fine-tuning Patient API chat:

- Data collected from production conversations.
- Fine-tuned model.
- Eval showed marginal improvement (96% → 97%).
- Operational cost: ~$30k/year (lifecycle).

Decision: stay on system prompt + frontier model. Marginal capability gain didn't justify operational overhead.

### 8.8 The annual review

Each Q1:

- Review per-workload approach.
- Has volume changed (affects fine-tune crossover)?
- Has model improved (system prompt may suffice for more)?
- Has workload solidified (fine-tune more viable)?

Adjustments per data.

### 8.9 The portfolio summary

```
Workload                Approach                    Rationale
─────────────────────────────────────────────────────────────
Care Coordinator        System prompt + few-shot    Diverse; nuance via examples
Patient API chat        System prompt               Volume; simplicity
Document classification Fine-tune (Llama 70B)       Volume; cost
Billing-code (specialty) Fine-tune (Llama 8B LoRA)  Reliability requirement
Clinical decision       System prompt + few-shot    Safety; evolving
Analytics SQL copilot   System prompt + few-shot    Per-customer queries vary
```

Mixed; per-workload.

### 8.10 The lessons

- System prompt + few-shot covers most cases.
- Fine-tune for volume / cost / reliability-driven cases.
- Per-workload analysis; annual review.
- Vendor pitch shouldn't be the deciding factor.

---

## 9. Anti-patterns

### 9.1 The "vendor pitched fine-tune; we should do it" decision

**Pattern.** Decision driven by vendor; not workload analysis.

**Corrective.** Per-workload analysis per §3.

### 9.2 The "few-shot has 50 examples" excess

**Pattern.** Few-shot grew incrementally; now overwhelming.

**Corrective.** Subset; or migrate to fine-tune per §5.6.

### 9.3 The fine-tune without data

**Pattern.** Fine-tune attempted with insufficient data; quality fails.

**Corrective.** Data prerequisite per §6.5.

### 9.4 The system-prompt-only when more is needed

**Pattern.** System prompt growing to 4000 tokens to handle edge cases.

**Corrective.** Few-shot examples per §4.4.

### 9.5 The fine-tune for changing workload

**Pattern.** Workload evolves quickly; fine-tune always stale.

**Corrective.** System prompt or few-shot for changing workloads §3.8.

### 9.6 The "we'll fine-tune eventually" deferral

**Pattern.** Volume justifies fine-tune; team defers; cost balloons.

**Corrective.** Threshold-based decision per §6.1.

### 9.7 The dogmatic "always system prompt"

**Pattern.** Some workloads need few-shot or fine-tune; team refuses.

**Corrective.** Per-workload analysis.

### 9.8 The non-evaluated few-shot examples

**Pattern.** Examples added without eval; quality not validated.

**Corrective.** Example curation per §5.8.

### 9.9 The fine-tune-now-and-pray operations

**Pattern.** Fine-tune done; no monitoring; quality drift undetected.

**Corrective.** Lifecycle discipline per [per-tenant-fine-tuning.md §5](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

### 9.10 The "we built infrastructure for fine-tune; let's use it" pattern

**Pattern.** Fine-tune infrastructure exists; team applies it to workloads that don't need it.

**Corrective.** Workload-justification per §3.

---

## 10. Findings (sprint-assignable)

**ARCH-FSP-001 (P0). Per-workload approach not documented.** Ad-hoc decisions. Document per workload per §3. Owner: AI platform.

**ARCH-FSP-002 (P0). Fine-tune used without data prerequisite.** Quality fails. Per §6.5. Owner: AI platform.

**ARCH-FSP-003 (P0). Few-shot grown to >20 examples without review.** Excess cost. Per §5.6. Owner: AI platform.

**ARCH-FSP-004 (P1). System-prompt > 3000 tokens.** Should consider few-shot. Per §4.4 and [system-prompt-architecture.md](./system-prompt-architecture.md). Owner: AI platform.

**ARCH-FSP-005 (P1). Annual approach review absent.** Drift; better options not adopted. Per §8.8. Owner: AI platform.

**ARCH-FSP-006 (P1). Vendor-pitch driving fine-tune adoption.** Not workload-justified. Per §6.7. Owner: AI platform + procurement.

**ARCH-FSP-007 (P1). Few-shot examples not curated / evaluated.** Quality unverified. Per §5.8. Owner: AI platform + eval.

**ARCH-FSP-008 (P1). Cost crossover analysis not done.** Fine-tune decision lacks economic basis. Per §3.7. Owner: AI platform + FinOps.

**ARCH-FSP-009 (P2). Dynamic few-shot not used.** Static all-examples-in-prompt is wasteful. Per §5.7. Owner: AI platform.

**ARCH-FSP-010 (P2). LoRA / adapter alternative not considered.** Lighter than full fine-tune. Per §6.9. Owner: AI platform.

**ARCH-FSP-011 (P2). Migration paths not documented.** Future migrations improvised. Per §7. Owner: AI platform.

**ARCH-FSP-012 (P2). Fine-tune-vs-larger-model comparison absent.** Stronger model sometimes simpler. Per §6.3 and [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md). Owner: AI platform.

**ARCH-FSP-013 (P2). Hybrid (system prompt + few-shot) not used where applicable.** Either alone may be suboptimal. Per §2.5 and §7.7. Owner: AI platform.

**ARCH-FSP-014 (P2). Fine-tune lifecycle cost not factored.** Per-call savings overstated. Per §6.6. Owner: AI platform + FinOps.

**ARCH-FSP-015 (P3). Cross-workload approach learnings not shared.** Same lessons rediscovered. Owner: AI platform + engineering management.

**ARCH-FSP-016 (P3). Annual model-improvement review absent.** Frontier improvements may make system prompt suffice. Per §4.7. Owner: AI platform.

**ARCH-FSP-017 (P3). Workload-stability assessment absent for fine-tune decisions.** Changing workloads should not fine-tune. Per §3.8. Owner: product + AI platform.

**ARCH-FSP-018 (P3). Per-workload migration plan absent.** No documented "when to consider fine-tune" trigger. Owner: AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Per-workload approach decision (§3).**
- [ ] **Document approach + rationale per workload.**
- [ ] **Few-shot curation / eval per §5.8.**
- [ ] **Fine-tune data prerequisite per §6.5.**
- [ ] **Cost crossover analysis per §3.7.**
- [ ] **Annual review of approach per workload (§8.8).**
- [ ] **Migration paths documented (§7).**
- [ ] **Hybrid approach when applicable (§2.5).**
- [ ] **Dynamic few-shot for large example sets (§5.7).**
- [ ] **LoRA / adapter consideration (§6.9).**

---

## 12. References

**In this folder.**
- [system-prompt-architecture.md](./system-prompt-architecture.md) — system prompt design.
- [context-window-budgeting.md](./context-window-budgeting.md) — context cost.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — assembly of approaches.
- [long-context-vs-rag.md](./long-context-vs-rag.md) — context content decision.
- [prompt-as-api-discipline.md](./prompt-as-api-discipline.md) — versioning of prompts.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — per-tenant fine-tune.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — self-host vs hosted.
- [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md) — comparison.

**Sibling repos.**
- [ai-engineering-reference-architecture / prompt-engineering / few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) — few-shot engineering.
- [ai-engineering-reference-architecture / prompt-engineering / structured-output-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/structured-output-engineering.md) — structured output.
- [ai-engineering-reference-architecture / model-lifecycle / fine-tuning-operations.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/fine-tuning-operations.md) — fine-tune operations.

**External.**
- LoRA paper (Hu et al., 2021).
- Few-shot prompting research (Brown et al., 2020).
- Provider fine-tuning documentation.
