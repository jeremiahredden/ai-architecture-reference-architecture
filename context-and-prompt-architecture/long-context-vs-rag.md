# Long-Context vs RAG

> **Audience.** Architects deciding "should we stuff everything into a 1M-context model, or build RAG?" Tech leads whose first instinct is "the model has 1M context; just put it all in." Anyone whose RAG quality is mediocre and wondering if long-context would solve it. **Scope.** The *architectural* decision framework: when long-context (single large prompt) beats RAG; when RAG beats long-context; when RAG-into-long-context is the right hybrid. Calibrated for 2026 models (200k-1M context). Not RAG engineering itself (see sibling [rag-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/rag-engineering)). Not the context budgeting (see [context-window-budgeting.md](./context-window-budgeting.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The 1M-context model is real. Anthropic Claude Sonnet 4.6 with 1M context (and successors), Google Gemini 1.5-2.5 with 1-2M context, others on the way. The question naturally arises: "if we have 1M context, do we need RAG?"

The answer is almost always yes, you still need RAG. But the cases where long-context beats RAG are real and worth understanding.

The temptation: stuff everything (entire knowledge base, full conversation history, complete codebase) into one long prompt. The model can handle it. Why bother with retrieval?

The reality:

- **Cost.** Per-call cost scales with input tokens. A 500k-token call costs 50x a 10k-token call.
- **Latency.** TTFT scales with input length. A 500k-token prompt has slow TTFT.
- **Quality.** Models can be worse at long context ("lost in the middle"). Quality often drops.
- **Cache-ability.** Repeated questions can hit cached responses; long-context prompts are harder to cache.

RAG remains the right answer for most workloads in 2026. But:

- Long-context wins for some.
- Hybrid (RAG-into-long-context) wins for others.

This document covers the decision framework, the workload signals that point one way or the other, and the hybrid pattern.

This document is opinionated about four things:

1. **Default to RAG; long-context is the exception.** Cost and latency favor RAG; quality is comparable or better with good RAG.
2. **The workload's signal matters.** Some workloads have inherent properties (cross-document reasoning, every-word-matters) that tilt toward long-context.
3. **Hybrid is a real architecture.** RAG to retrieve relevant subset; then long-context for that subset.
4. **The cost-benefit gets re-evaluated as model pricing shifts.** When long-context becomes cheaper, the line moves.

Structure: (2) the decision framework; (3) when long-context wins; (4) when RAG wins; (5) the hybrid pattern; (6) cost analysis comparison; (7) quality analysis comparison; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The decision framework

The framework for choosing.

### 2.1 The three options

**Option A: RAG.** Retrieve relevant content; include in prompt; standard pattern.

**Option B: Long-context.** Include all relevant content; large prompt; no retrieval.

**Option C: Hybrid (RAG-into-long-context).** Retrieve broadly; let the long context hold what RAG returns.

Each appropriate for different workloads.

### 2.2 The signals

For each workload, ask:

- **Cross-document reasoning required?** Does the answer require synthesizing across many documents? (Long-context advantage.)
- **Workload value per call?** Is the per-call value high enough to justify higher per-call cost? (Long-context viable.)
- **Volume?** Is the workload high-volume? (RAG critical for cost.)
- **Latency requirement?** How fast must the response be? (RAG faster.)
- **Knowledge base size?** Does the knowledge base fit in the context? (If yes, long-context viable; if no, RAG.)
- **Knowledge base freshness?** Does the knowledge change frequently? (RAG handles freshness better.)
- **Reasoning depth?** Is the reasoning complex enough that the model benefits from the full picture? (Long-context advantage.)

### 2.3 The decision matrix

```
Signal                          Favors    Strength
─────────────────────────────────────────────────
KB fits in context             Long      ★★
KB grows / changes frequently   RAG       ★★★
High volume (>1k calls/day)     RAG       ★★★
Low latency requirement         RAG       ★★
Cross-document synthesis        Long      ★★
High per-call value (>$0.50)    Long      ★★
Cost sensitivity                RAG       ★★★
Knowledge ambiguity (need broad context) Long ★★
```

Sum signals; tilt toward the dominant choice.

### 2.4 The "is it long enough" filter

Even if signals favor long-context:

- KB > 200k tokens: not viable for Sonnet's 200k.
- KB > 1M tokens: not viable even for 1M-context.
- For very large KBs: RAG is the only option.

### 2.5 The "is RAG quality good enough" question

RAG quality depends on retrieval quality (cross-link to [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md)):

- If retrieval recall@k > 90% on the workload: RAG is solid.
- If retrieval misses 30% of the time: RAG is failing; consider hybrid or long-context.

Diagnose RAG before concluding it's insufficient.

### 2.6 The per-workload application

For each workload, run the framework:

```
Workload: care coordinator agent
  KB size: ~500MB (~50M tokens) → doesn't fit; RAG needed.
  Volume: ~5k tasks/day → high volume; cost-sensitive.
  Latency: P99 < 60s; some slack but not unlimited.
  Cross-document reasoning: yes (patient history + protocols).
  
Decision: RAG. Retrieval pulls patient context + relevant protocols.
```

Apply per workload.

### 2.7 The "re-evaluate as costs drop" reminder

When long-context becomes cheaper (provider price changes; model improvements), the decision can shift.

Annual review per workload.

---

## 3. When long-context wins

Specific scenarios where long-context is the right answer.

### 3.1 Cross-document synthesis on small knowledge base

If the knowledge base fits and the task requires synthesizing across many documents:

- Legal: analyze a 50-document contract package.
- Financial: review a quarter's worth of earnings calls.
- Medical: comprehensive medical history review.

For these, putting the documents in context (instead of retrieving snippets) gives the model the full picture.

### 3.2 Complex reasoning over interrelated content

When relationships between content items matter:

- Code-aware analysis (need cross-file reasoning).
- Research analysis (multiple papers' relationships).
- Multi-step business analysis.

RAG can miss cross-content relationships; long-context preserves them.

### 3.3 Single-shot deep work

Per-call value high; one-off:

- "Analyze this entire patient's clinical history."
- "Review this entire legal case."
- "Audit this codebase."

Cost less of a concern at $1-10 per call if value is hundreds.

### 3.4 Few-shot examples requiring many examples

When the task needs 30+ few-shot examples:

- Long context fits them all.
- RAG could retrieve them; but the examples are the point.

For some specialized domains, lots of examples beats prompting.

### 3.5 Code generation across a full codebase

For code agents:

- Full codebase as context.
- Model reasons about cross-file references.
- New code aware of all conventions.

Large context window models (Cursor's approach; some Claude / GPT plugins) leverage this.

### 3.6 Document Q&A on a single document

When the user has uploaded a document:

- The whole document in context.
- Model answers from the document.
- No retrieval needed.

For document-bounded Q&A, long-context is natural.

### 3.7 The signals summary

Long-context wins when:

- KB fits (< 200k or < 1M depending on model).
- Cross-document synthesis matters.
- Volume is low enough that per-call cost is acceptable.
- Latency tolerant.
- Per-call value high.

---

## 4. When RAG wins

Specific scenarios where RAG is the right answer.

### 4.1 Large knowledge base

If the KB is > 200k tokens:

- Long-context doesn't fit.
- RAG retrieves relevant subset.

For most enterprise KBs (100s of GB), RAG is the only viable option.

### 4.2 High-volume workloads

Per-call cost matters at volume:

- 100k calls/day × $0.50/call = $50k/day.
- 100k calls/day × $0.04/call (RAG with smaller context) = $4k/day.

90% cost reduction; substantial.

### 4.3 Low-latency workloads

TTFT scales with input:

- 10k context: TTFT ~1s.
- 100k context: TTFT ~3-5s.
- 500k context: TTFT ~10-15s.

Latency-sensitive workloads need small context.

### 4.4 Frequent KB updates

If the KB changes hourly or daily:

- RAG: re-index incrementally; fresh.
- Long-context: prompt has to be re-assembled; harder to keep current.

For news, current events, dynamic knowledge: RAG.

### 4.5 Cache-friendly workloads

Cache hits are easier with RAG:

- Query + retrieved-chunks = relatively stable input.
- Repeat queries cache.

Long-context calls are harder to cache (large input).

### 4.6 Per-tenant data

Multi-tenant: per-tenant retrieval (cross-link to [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md)):

- RAG retrieves from tenant's data only.
- Long-context would require per-tenant prompts assembled.

RAG handles multi-tenant cleanly.

### 4.7 The signals summary

RAG wins when:

- KB doesn't fit in context.
- Volume is high.
- Latency is tight.
- KB changes frequently.
- Cache-friendly.
- Multi-tenant.

---

## 5. The hybrid pattern (RAG-into-long-context)

The best-of-both approach.

### 5.1 The pattern

```
User query → 
  Retrieval (broad; 50-100 chunks) →
    Reranking (narrow; top 20-30 chunks) →
      Long context (these chunks; up to 50k-100k tokens) →
        Model
```

Retrieve broadly; let the long context hold what RAG returns. Not just top-10 chunks; top-30.

### 5.2 The benefit

- Retrieval recall improves (broader retrieval; more chunks).
- Cross-chunk synthesis enabled (chunks in context together).
- Cost: between RAG and long-context.

### 5.3 The when

For workloads where:

- KB doesn't fit (so not pure long-context).
- Cross-chunk synthesis matters (so not just top-5 RAG).
- Per-call cost can support 30-100 chunks.

Mid-volume, mid-stakes workloads.

### 5.4 The retrieval-then-prompt assembly

```python
def hybrid_retrieve(query, k_initial=100, k_final=30):
    # Broad retrieval
    initial = vector_search(query, k=k_initial)
    
    # Rerank
    reranked = rerank_model(query, initial)
    final = reranked[:k_final]
    
    # Assemble into long context
    context = "\n\n---\n\n".join([c.text for c in final])
    return context
```

Cross-link to [ai-engineering-reference-architecture / rag-engineering / reranking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/reranking-engineering.md).

### 5.5 The cost

Hybrid cost = retrieval cost + reranking cost + long-context cost.

For 30 chunks × 500 tokens = 15k tokens of context. At Sonnet rate: ~$0.05/call. Acceptable for many workloads.

### 5.6 The trade-off

vs pure RAG (5-10 chunks):

- Cost: 3-5x.
- Quality: usually better.

vs pure long-context (everything):

- Cost: 10-50x cheaper.
- Quality: nearly as good (relevant chunks; less noise).

Hybrid is often the sweet spot.

### 5.7 The "we don't need reranking" simplification

For lower-stakes workloads:

- Initial retrieval (k=30) without reranking.
- Less compute; some accuracy loss.

Per-workload.

### 5.8 The "retrieve a lot; let context sort it out" pattern

For high-context-capable models:

- Retrieve generously (50-100 chunks).
- Long context absorbs the noise.
- Model reasons through.

Works for some workloads; "lost in the middle" risk for others.

---

## 6. Cost analysis comparison

The numbers.

### 6.1 The per-call cost model

```
RAG (typical):
  Retrieved: 5 chunks × 500 tokens = 2500 input tokens
  Plus system + user: ~1500 tokens
  Total input: ~4000 tokens
  Output: ~500 tokens
  Cost (Sonnet): $0.012 + $0.0075 = $0.020

Hybrid (RAG-into-long-context):
  Retrieved: 30 chunks × 500 tokens = 15000 input tokens
  Plus system + user: ~1500 tokens
  Total input: ~16500 tokens
  Output: ~500 tokens
  Cost: $0.050 + $0.0075 = $0.058

Long-context (full KB if it fits):
  Full KB: 100,000 tokens
  Plus system + user: ~1500 tokens
  Total input: ~101,500 tokens
  Output: ~500 tokens
  Cost: $0.305 + $0.0075 = $0.313
```

Cost ratio: RAG : Hybrid : Long-context ≈ 1 : 3 : 15.

### 6.2 The per-day cost

For 1000 calls/day:

```
RAG: $20/day = $600/month
Hybrid: $58/day = $1740/month
Long-context: $313/day = $9390/month
```

At scale (10k calls/day):

```
RAG: $200/day = $6000/month
Hybrid: $580/day = $17,400/month
Long-context: $3130/day = $93,900/month
```

The cost scales linearly with volume; the ratio between approaches is the lever.

### 6.3 The break-even

For 100 calls/day, low volume:

```
RAG: $60/month
Hybrid: $174/month
Long-context: $940/month
```

Long-context is feasible at $940/month if the value justifies.

For 100,000 calls/day:

```
RAG: $60,000/month
Long-context: $940,000/month
```

Long-context is rarely feasible at high volume.

### 6.4 The "value per call" framing

If each call generates $X of value:

- $0.50 value: RAG only.
- $5 value: Hybrid feasible.
- $50 value: Long-context viable.

Per-call value defines the cost ceiling.

### 6.5 The cache effect

For repeat queries:

- RAG with cache: cache hit → ~free.
- Long-context with cache: harder to cache (large input).

RAG benefits more from caching.

### 6.6 The prompt-caching for long-context

Provider prompt-caching helps long-context:

- First call: full cost.
- Subsequent calls (within cache window): 10% of full cost for cached portion.

For stable long-context, prompt-caching reduces cost ~5x.

Doesn't make it as cheap as RAG; reduces the multiplier.

### 6.7 The volume-weighted cost projection

For a feature's expected mix:

- 70% standard queries: RAG ($0.02 each).
- 25% deep-context queries: Hybrid ($0.058 each).
- 5% comprehensive queries: Long-context ($0.31 each).

Weighted average: $0.045 per call.

vs all-RAG: $0.02 (cheaper but lower quality on the 30% deep cases).
vs all-Long: $0.31 (15x more expensive).

Per-workload mixing.

---

## 7. Quality analysis comparison

When quality differs.

### 7.1 The "RAG is good enough" baseline

For most workloads, well-built RAG achieves 90-95% quality (judge-pass-rate). The remaining 5-10% gap may not justify long-context's cost.

### 7.2 The "lost in the middle" effect

Long-context models can have degraded performance on content in the middle of long prompts:

- Information at the start: well-attended.
- Information in the middle: sometimes missed.
- Information at the end: well-attended.

For some workloads, RAG's selectivity (only relevant content) avoids this.

### 7.3 The cross-document synthesis advantage

Long-context preserves relationships:

- Document A says X.
- Document B says Y.
- Synthesis: "X and Y together imply Z."

RAG can miss this if A and B aren't both in top-K.

For workloads requiring synthesis: long-context (or hybrid) better.

### 7.4 The hallucination rate

Long-context provides full picture:

- Reduces "I don't have that information" gaps.
- Reduces fabrication of plausible-sounding wrong answers.

RAG can hallucinate when retrieval misses.

### 7.5 The "RAG missed something important" failure

RAG quality fails when:

- The relevant chunk isn't in top-K.
- The user's query doesn't match the retrieval embedding well.
- The knowledge is implicit (across multiple chunks).

For these, long-context or hybrid avoids.

### 7.6 The "long-context confused itself" failure

Long-context fails when:

- Too much irrelevant content distracts.
- Conflicting information across documents.
- Model can't synthesize across so many sources.

Hybrid (filtered to relevant chunks; not all) avoids both failures.

### 7.7 The eval per approach

Per workload, eval all three:

- RAG with top-5 chunks.
- Hybrid with top-30 chunks.
- Long-context (if KB fits).

Compare quality. The right choice is per-workload data-driven.

### 7.8 The "we keep adding retrieval chunks" pattern

When RAG quality plateaus:

- Trying top-10 chunks.
- Then top-20.
- Then top-30.

At some point, more chunks isn't helping; quality stalls. Diagnose: is retrieval missing relevant content, or is the model unable to synthesize from the chunks?

If retrieval is the bottleneck: improve retrieval (better embeddings, better rerankers).
If synthesis is the bottleneck: long-context or hybrid.

---

## 8. Worked Meridian example

Meridian's per-workload decision.

### 8.1 The Care Coordinator workload

Knowledge base: ~50M tokens (clinical notes, protocols, formulary).

Doesn't fit in any context window.

Decision: RAG.

Architecture:
- Retrieval: top-7 relevant chunks (~3500 tokens).
- Per-call cost: ~$0.025.
- Per-call latency: ~1.5s TTFT.
- Quality: 96% pass rate.

Long-context alternative considered:
- Full KB doesn't fit.
- Not viable.

### 8.2 The clinical decision support workload

Workload: specific clinical decision questions.

Knowledge base: same as Care Coordinator (~50M tokens), doesn't fit.

Decision: Hybrid.

Architecture:
- Retrieval: 30 chunks (~15k tokens).
- Reranker: prioritize.
- Per-call cost: ~$0.06.
- Per-call latency: ~3s TTFT.
- Quality: 98% pass rate.

The extra cost justified by safety-critical nature; comprehensive context helps the model reason carefully.

### 8.3 The patient record review workload (new feature)

Workload: review a single patient's complete record.

Patient record: ~30k-100k tokens (varies).

Decision: Long-context.

Architecture:
- Full patient record in context.
- No retrieval needed (one patient; record is bounded).
- Per-call cost: ~$0.10-$0.30.
- Latency: ~3-8s.
- Quality: 99% pass rate.

Justified: per-call value high (clinician reviews; reduces their time); record fits.

### 8.4 The Patient API chat workload

Workload: patient-facing chat.

Knowledge base: ~10M tokens.

Decision: RAG.

Architecture: standard RAG; top-3 chunks (~2000 tokens).

Volume: 12k turns/day. Cost: ~$30/day.

Long-context unviable: KB doesn't fit; volume too high; latency too tight.

### 8.5 The document classification workload

Workload: classify documents into categories.

Per-document size: ~3000 tokens.

Decision: Long-context per document.

Architecture: full document in prompt; no retrieval needed (no external KB; classification from document).

Per-call cost: ~$0.01 (self-hosted Llama).
Per-call latency: ~1s.

Document IS the context.

### 8.6 The analytics warehouse copilot

Workload: SQL generation from natural language.

Knowledge: ~1M tokens of schema documentation.

Decision: Hybrid.

Architecture:
- Retrieve relevant schema documents (top 10; ~5000 tokens).
- Plus schema metadata in system prompt (~2000 tokens).
- Long context for the assembled chunks.

Quality: 97% pass rate on SQL generation.

### 8.7 The decision matrix summary

```
Workload                    Decision    KB-fit?  Volume  Per-call value
─────────────────────────────────────────────────────────────────────
Care Coordinator            RAG         No       High    Medium
Clinical decision support   Hybrid      No       Medium  High (safety)
Patient record review       Long        Yes      Low     High
Patient API chat            RAG         No       High    Low
Document classification     Long        Yes      Med     Low
Analytics SQL copilot       Hybrid      No       Med     Medium
```

Per workload; data-driven; documented.

### 8.8 The annual review

Q1 each year:

- Re-evaluate per workload.
- Has KB grown? (Care Coordinator: yes; doesn't change decision — still doesn't fit.)
- Has model cost dropped? (Yes; long-context more affordable.)
- Has model quality improved? (Yes; less "lost in the middle.")
- Decision still right? (Mostly yes.)

### 8.9 The lessons

- RAG wins for most workloads.
- Long-context for bounded-data workloads (single patient, single document).
- Hybrid for synthesis-required workloads.
- Per-workload eval is essential; assumptions about quality vary.

---

## 9. Anti-patterns

### 9.1 The "1M context; let's just stuff everything" reflex

**Pattern.** Use long-context for everything because the capacity exists. Cost balloons; latency suffers.

**Corrective.** Per-workload decision per §2.

### 9.2 The "RAG is bad; switch to long-context" misdiagnosis

**Pattern.** RAG quality is poor; conclusion is to abandon. Reality: RAG retrieval may be poorly tuned.

**Corrective.** Diagnose retrieval per §2.5; consider hybrid.

### 9.3 The pure-RAG-for-everything dogma

**Pattern.** RAG by default; no consideration of long-context. Miss the cases where long-context is better.

**Corrective.** Per-workload evaluation.

### 9.4 The "we don't need reranking" oversimplification

**Pattern.** Hybrid without reranking; top-K is noisy. Quality below expected.

**Corrective.** Reranker per §5.4.

### 9.5 The cost-ignorant long-context

**Pattern.** Use long-context; don't realize per-call cost is 15x RAG. Bill surprises.

**Corrective.** Cost analysis per §6.

### 9.6 The "lost in the middle" surprise

**Pattern.** Long-context used; quality regressions for content in middle. Don't realize the cause.

**Corrective.** Eval distribution of where in context the relevant information appears.

### 9.7 The "we'll evaluate quality later" deferral

**Pattern.** Decision made by intuition; eval comes later; bad decision sticks.

**Corrective.** Per-approach eval before committing.

### 9.8 The "RAG misses 30%; that's fine" acceptance

**Pattern.** RAG quality below acceptable; team accepts.

**Corrective.** Diagnose retrieval; or upgrade to hybrid.

### 9.9 The "we'll add retrieval later" deferral for new product

**Pattern.** New feature ships with long-context; "RAG is a v2 thing"; cost balloons.

**Corrective.** RAG from launch unless long-context is clearly justified.

### 9.10 The hybrid without thought

**Pattern.** "Hybrid is the best of both"; pick it without analysis. Wastes the savings of pure RAG.

**Corrective.** Per-workload decision per §2.

---

## 10. Findings (sprint-assignable)

**ARCH-LCR-001 (P0). No per-workload decision framework for RAG vs long-context.** Choice ad-hoc. Adopt framework per §2. Owner: AI platform.

**ARCH-LCR-002 (P0). Per-workload eval not done for both approaches.** Choice lacks evidence. Per §7. Owner: AI platform + eval.

**ARCH-LCR-003 (P0). Long-context used by default without cost analysis.** Bill surprises. Per §6. Owner: AI platform + FinOps.

**ARCH-LCR-004 (P1). RAG retrieval quality not diagnosed.** Conclusion to switch may be wrong. Per §2.5. Owner: AI platform.

**ARCH-LCR-005 (P1). Hybrid pattern not considered.** Defaulting to pure RAG or pure long-context. Per §5. Owner: AI platform.

**ARCH-LCR-006 (P1). Annual review of approach not scheduled.** Drift; model improvements not captured. Per §2.7. Owner: AI platform.

**ARCH-LCR-007 (P1). Prompt-caching for long-context not used.** Cost higher than necessary. Per §6.6. Owner: AI platform.

**ARCH-LCR-008 (P1). Per-tenant retrieval (RAG) not used for multi-tenant.** Long-context risks cross-tenant. Per §4.6. Owner: AI platform.

**ARCH-LCR-009 (P2). "Lost in the middle" not monitored.** Quality regressions undetected. Per §7.2. Owner: AI platform.

**ARCH-LCR-010 (P2). Per-workload tier-routing absent.** Same model for all approaches. Per §5. Owner: AI platform.

**ARCH-LCR-011 (P2). Cost-per-approach not tracked.** Hard to see the architectural cost. Per §6.7. Owner: observability + FinOps.

**ARCH-LCR-012 (P2). Quality comparison per approach not maintained.** Decisions get stale. Per §7.7. Owner: AI platform + eval.

**ARCH-LCR-013 (P2). RAG and long-context A/B testing absent.** Better choice unknown for marginal cases. Per [prompt-ab-testing.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-ab-testing.md). Owner: AI platform.

**ARCH-LCR-014 (P2). Hybrid retrieval params not tuned.** k_initial / k_final defaults not workload-aware. Per §5.4. Owner: AI platform.

**ARCH-LCR-015 (P3). Per-workload decision not documented.** Rationale lost. Per §8.7. Owner: AI platform.

**ARCH-LCR-016 (P3). Cost-per-call comparison stale.** Provider pricing has changed. Per §6.1. Owner: FinOps.

**ARCH-LCR-017 (P3). Long-context KB-fit calculation outdated.** Models with larger contexts available. Per §2.4. Owner: AI platform.

**ARCH-LCR-018 (P3). Customer-facing latency expectations not aligned with approach.** Long-context's slowness surprises users. Per §4.3 and customer comms. Owner: product + AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Define per-workload signals (§2.2).** Document for each workload.
- [ ] **Eval each approach per workload (§7.7).**
- [ ] **Cost analysis per approach per workload (§6).**
- [ ] **Document decision per workload (§8.7).**
- [ ] **Implement chosen approach.**
- [ ] **Hybrid pattern for synthesis-required workloads (§5).**
- [ ] **Prompt-caching for long-context (§6.6).**
- [ ] **Annual review of approach (§2.7).**
- [ ] **Track per-approach cost and quality.**

---

## 12. References

**In this folder.**
- [system-prompt-architecture.md](./system-prompt-architecture.md) — context's other allocations.
- [context-window-budgeting.md](./context-window-budgeting.md) — budget framework.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — assembly mechanics.
- [chat-history-architecture.md](./chat-history-architecture.md) — history is another context concern.

**Elsewhere in this repo.**
- [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) — broader RAG patterns.
- [reference-patterns/hybrid-retrieval-patterns.md](../reference-patterns/hybrid-retrieval-patterns.md) — retrieval architecture.
- [cost-and-performance-architecture/token-economics.md](../cost-and-performance-architecture/token-economics.md) — cost framework.
- [data-architecture-for-ai/vector-store-architecture.md](../data-architecture-for-ai/vector-store-architecture.md) — RAG infrastructure.

**Sibling repos.**
- [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — RAG engineering.
- [ai-engineering-reference-architecture / rag-engineering / reranking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/reranking-engineering.md) — reranking.

**External.**
- "Lost in the Middle" paper (Liu et al., 2023).
- Provider long-context documentation (Anthropic 1M, Google Gemini 2M).
- Provider prompt-caching documentation.
