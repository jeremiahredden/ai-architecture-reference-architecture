# Hybrid Retrieval Patterns

> **Audience.** Architects designing the retrieval layer for a RAG system. Tech leads whose pure-vector retrieval is missing the obvious lexical matches; or whose pure-lexical retrieval is missing semantic intent. Anyone whose retrieval has multiple stages and isn't sure whether each is earning its cost. **Scope.** The *architectural* patterns: when hybrid (BM25 + vector) beats single-modality; multi-stage retrieval (recall → rerank → context-fit); metadata filtering as first-class; index-shape decisions. Not the data-architecture-level store choice (see [data-architecture-for-ai/hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md)). Not the engineering of the retrieval pipeline (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A typical RAG system's retrieval started as one of:

- **Pure vector.** Embed everything; semantic similarity search. Works well for conceptual queries; misses lexical specifics.
- **Pure lexical (BM25 / Elasticsearch).** Token-based matching. Works well for exact terms; misses semantic intent.
- **Naive concat.** Run both; concatenate top-K from each. Doesn't deduplicate; doesn't rerank; usually adds noise.

The architectural pattern that consistently outperforms in production is *multi-stage hybrid*: first-stage recall combines lexical and vector signals; second-stage rerank ranks the merged set; third-stage context-fit selects the chunks that actually go to the model. Metadata filtering is first-class throughout, not an afterthought.

The pattern is well-understood but rarely well-implemented. Common failures:

- The first stage retrieves too much (recall good; precision bad); the model is overwhelmed with noisy context.
- The reranker is missing (or is the wrong reranker for the workload).
- Metadata filters happen after retrieval rather than during (expensive; sometimes wrong).
- Each stage is tuned in isolation; the end-to-end quality isn't what each stage's metrics suggest.
- The context-fit step is ad-hoc (just "take top K"); doesn't consider response budget or relevance distribution.

This document covers the architectural pattern: how the stages compose; what each does; how to decide on per-workload variations; how to evaluate the full pipeline rather than per-stage.

This document is opinionated about four things:

1. **Multi-stage is the production default.** Single-stage retrieval underperforms for most workloads. The stage architecture is the right starting point.
2. **Each stage has a different job.** Recall (don't miss relevant); rerank (order by relevance); context-fit (select what to send). Conflating them produces worse results than separating them.
3. **Metadata filtering is at retrieval, not after.** Storage-level filtering is much cheaper and more correct than post-retrieval filtering.
4. **End-to-end eval is the only meaningful measurement.** Per-stage metrics (recall@100 at stage 1; precision at stage 2) are diagnostic; only end-to-end answer quality decides if the pipeline works.

Structure: (2) the multi-stage architecture; (3) the recall stage; (4) the rerank stage; (5) the context-fit stage; (6) metadata filtering; (7) the index-shape decisions; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The multi-stage architecture

The pattern in detail.

### 2.1 The three stages

```
Query →
  Stage 1: Recall (high-recall, lower-precision; combines lexical + vector + filter)
    → Top 50-200 candidates
  Stage 2: Rerank (precision-focused; cross-encoder or LLM)
    → Top 10-20 candidates
  Stage 3: Context-fit (token-budget aware selection)
    → Top 3-8 chunks → into prompt
```

Each stage narrows; each stage optimises for a different property.

### 2.2 Why three stages

The cost / precision curve isn't linear. A single retrieval at top-3 trying to be both high-recall and high-precision underperforms a multi-stage that's high-recall first, then high-precision after:

- **Single-stage top-3.** Misses relevant; lower quality.
- **Three-stage top-3.** Catches relevant in stage 1; selects best in stage 2-3; higher quality.

The multi-stage's total cost is often comparable to a single high-quality call.

### 2.3 The stages' jobs

- **Recall.** Don't miss anything relevant. Wide net; lower precision OK.
- **Rerank.** Order by true relevance. Each candidate is examined more carefully.
- **Context-fit.** Select for what fits in the prompt budget AND maximises coverage. The final selection problem.

### 2.4 The "do I need three stages?" question

Some workloads work with two:

- **Recall + Rerank only.** When response budget is generous; context-fit is just "take top 5."
- **Rerank + Context-fit.** When the corpus is small enough that recall isn't needed (rerank can examine everything).

For most production RAG workloads, three stages.

### 2.5 The stages and existing infrastructure

The recall stage maps to the hybrid retrieval engine (per [hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md)). The rerank stage maps to a dedicated reranker (Cohere rerank, voyage rerank, or LLM-as-reranker). The context-fit stage is application logic.

The architectural pattern is "use the right tool for each stage" rather than "build one tool that does everything."

### 2.6 The per-stage observability

Each stage has its own metrics:

- Stage 1: recall@N (did we get the relevant ones?), latency, cost.
- Stage 2: NDCG@K (are top-K ordered well?), latency, cost.
- Stage 3: token utilization, relevance distribution.

Per-stage metrics diagnose where quality issues come from. End-to-end metric (answer quality) decides if the pipeline works overall.

### 2.7 The "no rerank" failure

Many production RAG systems skip the rerank stage; pass top-K from recall directly to the model. Quality suffers:

- Top-K from recall often has 30-50% irrelevant chunks (recall optimised for not missing; precision suffers).
- Model is overwhelmed; hallucinates from noise; misses key relevant chunks.

Adding a rerank step typically lifts answer quality 8-15% — one of the highest-ROI retrieval improvements.

---

## 3. The recall stage

First-stage: don't miss relevant.

### 3.1 The recall surface

For each query, retrieve top 50-200 candidates. The "right" number depends on:

- Corpus size (larger corpus → larger top-N may be warranted).
- Query complexity (multi-faceted queries benefit from larger top-N).
- Rerank's capacity (rerank must process all top-N).

50-200 is the typical range; ~100 is the common default.

### 3.2 The signals: lexical + vector + filter

A modern recall stage combines:

- **BM25 lexical search.** Token-based; matches exact and stemmed terms.
- **Vector similarity search.** Embedding-based; matches semantic intent.
- **Metadata filter.** Per-tenant, date range, document type, status (per section 6).

The three signals applied together at the storage layer (per [hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md)).

### 3.3 The fusion at recall

How lexical and vector signals merge:

- **RRF (Reciprocal Rank Fusion).** Default; combines rankings without normalisation.
- **Weighted score combination.** Combines normalised scores with tuned weights.
- **Always-both with set union.** Both signals' top-K combine; merged set passed to rerank.

RRF is the production default for first-stage merge; weighted is for tuned workloads; always-both is the simplest.

### 3.4 The "vector only" failure for some workloads

Pure-vector recall misses lexical specifics:

- Drug names (vector models may not distinguish well between similar drug names).
- Codes (ICD-10, CPT, NPI).
- Provider names.
- Exact phrases.

For clinical, legal, financial workloads — anywhere domain terminology matters — hybrid recall significantly outperforms vector-only.

### 3.5 The "lexical only" failure

Pure-lexical recall misses semantic intent:

- Paraphrases ("heart attack" vs "myocardial infarction" vs "MI").
- Synonyms not in the index's synonym list.
- Conceptual matches ("complications" vs specific complication terms).

Hybrid catches both.

### 3.6 The cost of recall

Per recall query:

- Embedding the query: ~50ms (provider-hosted) to ~10ms (co-located).
- Storage lookup: ~50-200ms for 10M-document corpus.
- Cost: ~$0.0001-0.0005 per query.

The recall cost is small; not usually the optimization target.

### 3.7 The recall accuracy: recall@N

For each query (in an eval set), what fraction of relevant chunks appear in the top-N? Target: recall@100 > 0.95 for most workloads.

If recall@100 < 0.90, the recall stage is the bottleneck; tune it before the others.

### 3.8 The recall expansion: query rewriting

Sometimes the original query doesn't match the corpus's vocabulary. Query rewriting (per the engineering sibling's [query-rewriting.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/query-rewriting.md)) expands or rephrases before retrieval.

The pattern fits the recall stage; the rewritten query (or multiple variations) feeds the hybrid retrieval.

---

## 4. The rerank stage

Second-stage: order by true relevance.

### 4.1 What rerank does

Takes the top 50-200 from recall; re-orders by a more accurate (and more expensive) relevance signal. Returns top 10-20.

### 4.2 The reranker options

- **Cohere rerank (rerank-3 in 2026).** Mature; managed; widely used. ~$0.001-0.003 per query of 100 candidates.
- **Voyage rerank.** Comparable quality; alternative.
- **Open-source cross-encoders.** ms-marco family models; self-hostable.
- **LLM-as-reranker.** A small LLM judges relevance per query-candidate pair. Flexible; expensive; slow.

For most production workloads: vendor reranker (Cohere or Voyage). Self-hosting if cost / volume justifies.

### 4.3 The cross-encoder advantage

Recall uses bi-encoders (query and chunk embedded separately; cosine similarity). Rerank uses cross-encoders (query + chunk processed together; relevance scored).

Cross-encoders are slower per pair but materially more accurate. Using them in the rerank stage (limited to ~100 candidates) is feasible; using them in recall (millions of candidates) isn't.

### 4.4 The rerank's impact

Rerank typically lifts NDCG@10 by 0.05-0.15 over hybrid-only retrieval. End-to-end answer quality typically up 8-15%.

For the cost (~$0.002 per query) the rerank is one of the highest-ROI retrieval investments.

### 4.5 Per-workload rerank tuning

The reranker has limits:

- Vendor rerankers are general-purpose; may not capture domain-specific relevance perfectly.
- Self-hosted cross-encoders can be fine-tuned on domain data; quality gain on specialized workloads.

Most teams: vendor default + monitor; consider fine-tune if quality plateau.

### 4.6 The LLM-as-reranker

For some workloads, an LLM does the reranking via prompt:

```
You are evaluating which of these passages is most relevant to the query.
Query: ...
Passage 1: ...
Passage 2: ...
...
Return JSON: ordered list of passage indices by relevance.
```

**Pros.**
- Captures complex relevance (reasoning, context-aware).
- No model training; just a prompt.

**Cons.**
- Cost: $0.005-0.05 per query depending on candidates and model.
- Latency: 500-2000ms.
- Stochastic.

**When right.** High-stakes workloads where the rerank quality matters more than cost / latency. Reserve for cases that justify.

### 4.7 The rerank's recall responsibility

Rerank doesn't add candidates; it can only rank what recall provided. If recall misses something, rerank can't find it.

This is why recall@N is the upstream constraint; rerank quality is bounded by it.

### 4.8 The rerank cost cap

Rerank cost scales with candidates × queries. At 1k queries/day × 100 candidates × $0.001: ~$30/month for vendor rerank. Manageable.

At 1M queries/day × 100 candidates: ~$3k/month. Worth optimizing (smaller candidate sets; tier rerank; etc.).

---

## 5. The context-fit stage

Third-stage: select what goes to the model.

### 5.1 What context-fit does

Takes the top 10-20 reranked candidates; selects 3-8 to actually include in the prompt. The selection optimises for:

- **Token budget.** Total prompt size must fit in the model's context window with response budget remaining.
- **Coverage.** Selected chunks should cover the query's facets, not just the top-ranked.
- **Redundancy.** Avoid duplicate or near-duplicate content.

### 5.2 The naive context-fit (just take top K)

The simplest pattern: top K from rerank → into prompt.

**Pros.**
- Simple.
- Predictable cost (K chunks).

**Cons.**
- May include near-duplicates (e.g., top 5 are 3 different versions of the same source).
- May miss facets (all top 5 cover one aspect; other aspects unaddressed).
- May exceed token budget for long chunks.

**When right.** Simple workloads; default starting point.

### 5.3 The diversity-aware context-fit

MMR (Maximal Marginal Relevance) or similar:

- Select first chunk: highest relevance.
- Select second chunk: high relevance AND different from first.
- Continue: high relevance AND different from already-selected.

The selection covers more facets at the cost of slightly lower top-relevance.

### 5.4 The token-budget-aware context-fit

The selection is constrained by total tokens:

```
Selected = []
Budget = max_prompt_tokens
For chunk in reranked_candidates:
    if Budget - chunk.tokens >= 0:
        Selected.append(chunk)
        Budget -= chunk.tokens
    if len(Selected) >= max_chunks:
        break
return Selected
```

Constraints: total tokens ≤ budget; chunks ≤ max count.

### 5.5 The "context fits everything" misconception

Models with 200K-1M context windows technically can hold many chunks. But:

- Bigger context costs more (per-token cost).
- Bigger context degrades quality on the bottom edge (the "lost in the middle" effect).
- Bigger context increases latency.

Just because it fits doesn't mean it should be there. Context-fit selects deliberately.

### 5.6 The chunk-stitching pattern

When the relevant content spans multiple chunks (e.g., page 4 and page 5 of a document), context-fit may stitch:

- Adjacent chunks combined.
- Cross-references preserved.
- Source attribution maintained.

The stitching is engineering effort but improves quality on long-source content.

### 5.7 The context-fit eval

- **Coverage.** Does the selected context contain the information needed to answer?
- **Token efficiency.** Is the context budget well-used?
- **Diversity.** Are facets covered?

Measured against the eval set.

### 5.8 The "include the citation" requirement

For workloads requiring citations, context-fit includes the source metadata with each chunk:

- Document ID, section, page (where applicable).
- Used by the model to cite back to source.

The architecture decides where citation metadata sits (in the chunk text vs as a separate field); affects how the model produces citations.

---

## 6. Metadata filtering

First-class throughout, not an afterthought.

### 6.1 Why metadata matters

Every retrieval has implicit or explicit filters:

- Tenant scope (per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md)).
- Document type ("clinical guidelines only").
- Date range ("recent encounters").
- Status ("published only").
- Custom (per-workload).

These filters are part of the retrieval; not post-filtering.

### 6.2 Filtering at the storage layer

The storage engine applies filters during search:

```sql
SELECT * FROM corpus 
WHERE tenant_id = 'X' 
  AND document_type = 'clinical_guideline'
  AND date >= '2026-01-01'
ORDER BY hybrid_score(query)
LIMIT 100;
```

The filter is part of the index query; not applied after.

### 6.3 The "filter after retrieval" anti-pattern

```python
# DON'T:
candidates = retriever.search(query, top_k=1000)
filtered = [c for c in candidates if c.tenant_id == tenant_id][:100]
```

Problems:
- Wasteful (retrieved 1000 to filter to 100; or fewer than needed pass filter).
- Wrong recall (the 1000 are best matches without filter; the 100 best with filter may not be in the 1000).

Correct pattern: filter at storage; query returns 100 already-filtered results.

### 6.4 The filter-as-required pattern

Some filters are non-negotiable:

- Tenant scope (always required).
- Status (only published).
- Permissions (only this user's authorised content).

The retrieval client refuses queries without these filters. Per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md).

### 6.5 The filter performance considerations

Filter performance depends on the storage engine:

- **Highly selective filters early.** A filter that eliminates 99% of data should apply before the costly ranking.
- **Index design supports filters.** Filtered fields should be indexed; otherwise the filter scans.
- **Filter cardinality matters.** Date ranges, tenant IDs work well as filters; free-text fields don't.

The storage engineer's job; the architecture acknowledges.

### 6.6 The metadata schema

Per [data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md), the metadata schema is a contract:

- Fields present per document.
- Types and constraints.
- Versioned.

Schema drift breaks filters. The schema discipline matters.

### 6.7 The compound filter

Complex queries combine filters:

- "Clinical guidelines + published + Q1 2026 + status=active + (specialty=cardiology OR cardiology adjacent)"

The storage engine handles compound filters efficiently; the architecture doesn't need to chain.

### 6.8 The "filter as scope, not as restriction" framing

Filters express the user's authorised scope (what they can see), not "restrict the search to this." The framing matters:

- Filter = "this is what's available to you."
- Restriction = "we're not letting you search broadly."

Same mechanism; different framing in error messages and user understanding.

---

## 7. The index-shape decisions

The data-modelling that determines retrieval cost and quality.

### 7.1 Chunking decisions

Chunk size, overlap, splitting strategy. Per the engineering sibling's [chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md):

- Smaller chunks: more precise; less context per chunk.
- Larger chunks: more context; less precise.
- Overlap: catches mid-chunk relevance.

For hybrid: typically 200-800 tokens; per workload.

### 7.2 What goes in each field

A document indexed has multiple fields:

- **Title** (often boosted in lexical).
- **Body** (the main content; both lexical and vector indexed).
- **Tags / metadata** (structured filters).
- **Summary** (sometimes embedded separately).
- **Identifiers** (document_id, source URI, etc.).

The schema is decided per workload; affects all three retrieval stages.

### 7.3 The boost design

For lexical retrieval, per-field boosts:

- Title matches: 3× weight.
- Body matches: 1× (baseline).
- Tag matches: 2× weight.

The boosts are tuned per workload; eval informs.

### 7.4 The embedding decisions

For vector retrieval, per [embedding-strategy.md](../data-architecture-for-ai/embedding-strategy.md):

- Embedding model (text-embedding-3-large, BGE-M3, etc.).
- Dimension (often Matryoshka-truncated).
- Quantization (int8 vs float32).

These decisions affect storage, query cost, recall.

### 7.5 The synonym discipline

Lexical retrieval benefits from synonym lists:

- Domain-specific (drug brand / generic names; ICD codes / descriptions).
- Acronyms (MI / myocardial infarction).
- Common variants.

The synonym list is curated; updated by domain experts.

### 7.6 The analyzer choice

Stemming, lemmatization, tokenization:

- Domain-specific analyzers (medical, legal).
- Language-specific.
- Custom for specific terms.

Affects what matches as "the same word."

### 7.7 The chunk metadata

Each chunk carries:

- Source document ID.
- Position within document.
- Section/heading context.
- Date.
- Tenant scope.
- Custom domain fields.

The metadata supports filtering AND citation in responses.

### 7.8 The eval reveals the design

The right index shape isn't theoretical; it's data-driven:

- Run eval on the golden set.
- Tune chunking; eval.
- Tune embedding dimension; eval.
- Tune boosts; eval.
- Continue until quality plateaus.

The end-to-end eval is the optimisation target.

---

## 8. Worked Meridian example

Meridian's hybrid retrieval architecture.

### 8.1 The stack

- **Engine:** OpenSearch with k-NN plugin (per [hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md)).
- **Embeddings:** OpenAI text-embedding-3-large at 1024 dimensions (per [embedding-strategy.md](../data-architecture-for-ai/embedding-strategy.md)).
- **Reranker:** Cohere rerank-3.
- **Application code:** Custom Python in `meridian-retrieval` service handles the multi-stage flow.

### 8.2 The recall stage

Per Care Coordinator query:

- Hybrid query (BM25 over title + body; vector over body_vector; structured filter on tenant + status + date range).
- RRF fusion with k=60.
- Top 100 candidates returned.
- Latency p99: 180ms.
- Cost: ~$0.0005 per query (embedding for query).

### 8.3 The rerank stage

- Top 100 from recall → Cohere rerank-3.
- Returns top 25.
- Latency p99: 220ms.
- Cost: ~$0.002 per query.

### 8.4 The context-fit stage

- Top 25 from rerank → application logic:
  - Token-budget aware (limit context to 4k tokens).
  - Diversity-aware (deduplicate near-similar; cover query facets).
  - Returns top 5-8 chunks.
- Citation metadata included.

### 8.5 The metadata filter

Every query has required filters:

- `tenant_id` (always required; enforced by retrieval client).
- `status = 'active'` (deprecated content excluded).
- `published_date <= now` (no draft content).

Optional filters per request:

- `document_type` (clinical_guideline, payer_policy, etc.).
- `language` (per patient's preferred language).

### 8.6 The chunking

- ~500-token chunks with 50-token overlap.
- Section boundaries respected (chunks don't span major sections).
- Per [chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md).

### 8.7 The end-to-end metrics

Per Care Coordinator interaction:

- Recall@100 (at the chunk level): 0.93.
- NDCG@10 (after rerank): 0.87.
- Answer quality (LLM-judge): 4.21/5.

The pipeline is calibrated to these targets; quarterly review.

### 8.8 The cost

Per Care Coordinator interaction's retrieval:

- Recall: $0.0005.
- Rerank: $0.002.
- Total retrieval: ~$0.0025.

Compared to total interaction cost (~$0.10-0.18), retrieval is ~2-3% of cost. Worth optimizing; not the dominant cost.

### 8.9 The optimization history

- **Initial:** vector-only at top-5; answer quality 3.81.
- **Add lexical (hybrid):** answer quality 4.05.
- **Add rerank:** answer quality 4.21.
- **Add MMR context-fit:** answer quality 4.21 (same; cost slightly down).

Each stage's addition gave material lift; cumulative ~10% lift over vector-only.

### 8.10 What worked

- **Multi-stage from the start (after the initial vector-only iteration).** Each stage doing its job.
- **Vendor rerank.** Easy to adopt; meaningful quality lift.
- **MMR context-fit.** Modest but real benefit; reduces context waste.
- **Per-stage observability.** Diagnostics when quality drops.

### 8.11 What didn't work initially

- **No rerank for the first 6 months.** Answer quality plateau'd; rerank was the unlock.
- **Naive context-fit (just top 5).** Saw near-duplicates in context; MMR fixed.
- **Filter-after-retrieval.** Bug discovered: some tenant queries silently returned cross-tenant content. Moved filter to storage layer.
- **Default analyzer.** Missed domain synonyms; custom synonym list added.

---

## 9. Anti-patterns

### 9.1 "Single-stage retrieval"

No rerank, no context-fit. Top-K from initial retrieval directly to model. Quality leaves significant value on the table.

**Corrective.** Multi-stage per section 2.

### 9.2 "Vector only" or "lexical only"

Either pure-vector or pure-lexical when the workload benefits from both.

**Corrective.** Hybrid per section 3.

### 9.3 "Filter after retrieval"

Filter applied to retrieved results; wasteful and sometimes wrong.

**Corrective.** Filter at storage per section 6.2.

### 9.4 "Naive context-fit"

Just take top K; doesn't deduplicate; doesn't respect token budget.

**Corrective.** Token-budget-aware + diversity-aware per section 5.

### 9.5 "Per-stage eval only"

Each stage tuned to its own metric; end-to-end quality not measured.

**Corrective.** End-to-end eval per section 2.6 / 7.8.

### 9.6 "LLM-as-reranker for everything"

Expensive reranker used routinely; cost and latency cost.

**Corrective.** Vendor reranker default per section 4.2; LLM-as-reranker for high-stakes only.

### 9.7 "Skip rerank because vector retrieval is good"

Pure vector + top-K skipping rerank. Wastes the biggest available quality lever.

**Corrective.** Add rerank per section 4.

### 9.8 "Schema drift breaks filters"

Metadata schema changes; filters silently fail.

**Corrective.** Per [data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md).

---

## 10. Findings (sprint-assignable)

### ARCH-HRP-001 — Severity: Critical
**Finding.** Single-stage retrieval (no rerank); answer quality leaves 10-15% on the table.
**Recommendation.** Add rerank stage per section 4; vendor reranker (Cohere or Voyage).
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-HRP-002 — Severity: Critical
**Finding.** Filter-after-retrieval pattern; risk of cross-tenant leakage; wasteful.
**Recommendation.** Filter at storage layer per section 6.2; storage enforcement per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-HRP-003 — Severity: High
**Finding.** Pure-vector retrieval on workload with significant lexical specificity (drug names, codes); recall on specific terms poor.
**Recommendation.** Hybrid per section 3.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-HRP-004 — Severity: High
**Finding.** End-to-end eval not run; per-stage metrics suggest quality but answer quality unknown.
**Recommendation.** End-to-end eval per section 7.8.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-HRP-005 — Severity: High
**Finding.** Naive context-fit (top K); near-duplicates and budget overflow.
**Recommendation.** Token-budget + MMR per section 5.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-HRP-006 — Severity: Medium
**Finding.** Recall@N below target on golden set; rerank limited by what recall provides.
**Recommendation.** Tune recall per section 3; raise top-N or improve fusion.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HRP-007 — Severity: Medium
**Finding.** Domain synonyms missing; lexical recall low on domain terms.
**Recommendation.** Per section 7.5; synonym list curation.
**Owner.** ai-platform-eng + domain experts, sprint N+3.

### ARCH-HRP-008 — Severity: Medium
**Finding.** Per-stage observability absent; quality issues hard to localise.
**Recommendation.** Per-stage metrics per section 2.6.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-HRP-009 — Severity: Medium
**Finding.** Reranker vendor chosen by reputation; not eval'd against alternatives on team content.
**Recommendation.** Per section 4.5; eval Cohere vs Voyage vs alternatives.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HRP-010 — Severity: Medium
**Finding.** Citation metadata not included in retrieved chunks; model can't cite back to source reliably.
**Recommendation.** Per section 5.8; include metadata with chunk.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HRP-011 — Severity: Medium
**Finding.** Chunking strategy hasn't been re-evaluated; production may benefit from different chunk size.
**Recommendation.** Per section 7.1; eval chunk sizes.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HRP-012 — Severity: Medium
**Finding.** Rerank cost not tracked; cost surprise possible at scale.
**Recommendation.** Per section 4.8; cost attribution.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HRP-013 — Severity: Medium
**Finding.** Context-fit doesn't consider response budget; long chunks crowd response space.
**Recommendation.** Per section 5.4; budget-aware selection.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HRP-014 — Severity: Medium
**Finding.** Metadata schema not versioned; filter behaviour can change silently.
**Recommendation.** Per [data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md).
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HRP-015 — Severity: Low
**Finding.** Query rewriting not used; some queries' lexical form doesn't match corpus's vocabulary.
**Recommendation.** Per section 3.8; query rewriting in recall stage.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HRP-016 — Severity: Low
**Finding.** Chunk-stitching not implemented; multi-chunk relevance handled poorly.
**Recommendation.** Per section 5.6; stitch where relevance crosses chunks.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-HRP-017 — Severity: Low
**Finding.** Boost weights not tuned; default fields treated uniformly.
**Recommendation.** Per section 7.3; eval per-field boost.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-HRP-018 — Severity: Low
**Finding.** Self-hosted reranker not considered for scale; vendor cost will dominate at very high volume.
**Recommendation.** Per section 4.2; self-host evaluation when volume warrants.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building hybrid retrieval:

- [ ] **Sprint 0 — eval baseline.** Vector-only recall+top-K quality.
- [ ] **Sprint 1 — hybrid recall.** BM25 + vector + RRF fusion.
- [ ] **Sprint 1 — metadata filter at storage.** Enforced; tested.
- [ ] **Sprint 2 — rerank stage.** Vendor reranker; integrated.
- [ ] **Sprint 2 — eval impact.** Measure end-to-end quality lift.
- [ ] **Sprint 3 — context-fit stage.** Token-budget + diversity-aware.
- [ ] **Sprint 3 — per-stage observability.** Recall@N, NDCG@K, token utilization.
- [ ] **Sprint 4 — domain synonyms.** Lexical recall improvement.
- [ ] **Sprint 4 — chunk-metadata for citations.** Source attribution in chunks.
- [ ] **Sprint 5 — query rewriting (if needed).** For queries that miss corpus vocabulary.
- [ ] **Ongoing — quarterly review.** End-to-end eval; per-stage tuning; capacity.

For a team with existing retrieval:

- [ ] **Sprint 0 — audit.** Current stages; per-stage metrics; end-to-end eval.
- [ ] **Sprint 1 — fix worst gap.** Often missing rerank or filter-after-retrieval.
- [ ] **Sprint 2 — eval-driven tuning.** Per-stage; end-to-end.

A team that completes the sequence has retrieval that materially outperforms naive approaches. A team that doesn't has retrieval that's a quality bottleneck.

---

## 12. References

- [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) — RAG decision framework; hybrid is one variant.
- [agent-topologies.md](./agent-topologies.md) — agent retrieval interacts with these patterns.
- [multi-model-orchestration.md](./multi-model-orchestration.md) — retrieval + orchestration combine.
- [structured-output-patterns.md](./structured-output-patterns.md) — structured output for retrieval responses.
- [pattern-anti-patterns.md](./pattern-anti-patterns.md) — pattern-level anti-patterns.
- [data-architecture-for-ai/hybrid-store-architecture.md](../data-architecture-for-ai/hybrid-store-architecture.md) — data-architecture perspective.
- [data-architecture-for-ai/vector-store-architecture.md](../data-architecture-for-ai/vector-store-architecture.md) — vector store choices.
- [data-architecture-for-ai/embedding-strategy.md](../data-architecture-for-ai/embedding-strategy.md) — embeddings.
- [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md) — metadata schema discipline.
- [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — filter as scope enforcement.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — engineering depth.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/reranking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/reranking-engineering.md) — rerank engineering.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md) — chunking.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/query-rewriting.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/query-rewriting.md) — query rewriting.
- Cohere rerank, Voyage rerank — vendor reranker options.
- Reciprocal Rank Fusion (Cormack, Clarke, Buettcher, 2009) — RRF paper.
- "Lost in the Middle" (Liu et al., 2023) — long-context quality degradation.
- MMR (Maximal Marginal Relevance; Carbonell & Goldstein, 1998) — diversity-aware selection.
