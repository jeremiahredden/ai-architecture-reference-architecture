# Embedding Strategy

> **Audience.** Architects choosing the embedding model for a RAG-shaped feature or evaluating whether to change one. Tech leads who have been asked "should we move from OpenAI ada to Voyage v3" and want a structured answer. **Scope.** The *architectural* decisions about embeddings — model selection, dimension choice, normalization, the migration playbook for changing models without invalidating the production retrieval system. Not the engineering of the ingestion pipeline (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/embedding-pipeline-engineering.md`). Not the eval methodology for embeddings (sibling's `eval-engineering/`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Embeddings are infrastructure. The choice of embedding model determines what's representable in the vector store, what's retrievable, what cost the team carries per million chunks indexed and per million queries served, and — most operationally — what happens when the team wants to change the choice.

The embedding model is upstream of every retrieval the system does. A change to the model invalidates the existing index: the new model produces different vectors for the same input; queries embedded with the new model don't match documents embedded with the old. The team that didn't plan for this discovers, when they try to upgrade, that the upgrade is a multi-week index rebuild affecting tens to hundreds of millions of vectors.

The discipline this document covers is making the embedding choice deliberately, sizing the dimension space to balance cost and quality, treating the embedding model as a versioned piece of infrastructure (per [lineage-and-provenance.md](./lineage-and-provenance.md)), and planning the migration path before the team needs it.

The 2026 embedding landscape has multiple viable model families: OpenAI text-embedding-3 variants, Voyage embeddings (Voyage 3, voyage-3-lite), Cohere embed-v4, BGE / E5 open-weight families, Anthropic-future-state options, and specialised domain models (clinical-BERT-derivative, code embeddings, multi-modal). The "right" choice depends on the workload's content shape, the quality bar, the cost envelope, and the team's operational appetite for self-hosting vs managed.

This document is opinionated about four things:

1. **Embedding choice is a per-workload architectural decision, not an org-wide standard.** Different features may rightly use different models; the standardisation tax is real and not always worth paying.
2. **Dimension is a cost lever, not a quality dial.** Higher dimensions don't reliably improve quality; lower dimensions cost less and often suffice. Matryoshka-style trainable dimensionality is the right pattern when available.
3. **The migration plan is part of the choice.** Pick a model whose migration path the team can execute; if migration is genuinely impossible (e.g., proprietary model with no successor), the choice carries a long-term risk premium.
4. **The embedding model's version is a first-class artefact.** Embeddings carry their model version; queries embedded with one version do not match documents embedded with another; the system must handle this with discipline.

Structure: (2) the embedding choice space; (3) dimension as cost lever; (4) the model-selection decision; (5) versioning embeddings as artefacts; (6) the migration playbook; (7) operational and cost considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The embedding choice space

The 2026 embedding model landscape, characterised architecturally.

### 2.1 The model categories

| Category | Examples | Characteristics |
| --- | --- | --- |
| Provider-hosted general | OpenAI text-embedding-3-large/small, Voyage 3, Cohere embed-v4 | Managed; per-call pricing; consistent quality; vendor risk |
| Provider-hosted specialised | Voyage code, Cohere multilingual | Domain-tuned; same hosting characteristics |
| Open-weight general | BGE-M3, E5-large, gte-modernbert-base | Self-host; flat cost (infra); ongoing maintenance |
| Open-weight specialised | clinical-BERT-derivatives, financial-BERT, code-specific | Domain-tuned; self-host |
| Multi-modal | CLIP variants, NomicEmbed, ImageBind | Image + text in unified space; specialised storage; less common |

The four general-purpose patterns (provider general, provider specialised, open-weight general, open-weight specialised) cover most production workloads.

### 2.2 Provider-hosted general

The default for most teams in 2026.

**Pros.**
- Zero operational burden (no model serving, no GPU management).
- Consistent quality (the provider invests in model improvement).
- Pricing is per-token; aligns with usage.
- Quick to adopt; no infrastructure investment.

**Cons.**
- Per-token cost scales with volume; at high scale, can exceed self-hosted cost.
- Provider risk (price changes, model deprecation, API outages, terms changes).
- Latency depends on provider; network round-trip per call.
- Data crosses the provider's API (compliance / privacy considerations).

**When right.** Most teams' starting point. Especially right for: variable volume, fast iteration, teams without ML platform investment, workloads where data-egress concerns are manageable.

### 2.3 Provider-hosted specialised

Same operational characteristics as general; tuned for specific domains.

**When right.** Workloads in the specialised domain (code retrieval, multilingual content) where the specialised model demonstrably outperforms general models on the team's evals.

### 2.4 Open-weight general

Self-hosted on the team's infrastructure.

**Pros.**
- Flat infrastructure cost; doesn't scale with query volume.
- Data stays in the team's environment.
- Full control over latency (co-located deployment).
- No provider dependency.

**Cons.**
- Operational burden (GPU serving, scaling, monitoring).
- Quality may lag provider-hosted (depends on model; gap is narrowing in 2026).
- Upgrade work is the team's burden.
- Capacity planning required.

**When right.** High-volume workloads (where per-token cost would dominate); strict data-residency requirements; teams with mature ML platform.

### 2.5 Open-weight specialised

Domain-specific open models.

**When right.** Specialised domain workloads where the open model is comparable to or better than provider general models, and the team has the infrastructure to host.

### 2.6 Multi-modal

CLIP-family models embed text and images in a unified space.

**When right.** Multi-modal retrieval (search across documents and images); product catalog search; medical imaging with text-based queries. Specialised; not the default for text-only workloads.

### 2.7 The "which is best" question is wrong

There is no universally best embedding model. The choice depends on:

- The team's content domain.
- The team's quality bar.
- The team's volume.
- The team's operational capacity.
- The team's compliance constraints.
- The team's cost envelope.

The right choice is evaluated against the team's specific eval set, not against published benchmarks. MTEB and other benchmarks are a starting filter, not a decision.

---

## 3. Dimension as cost lever

The model's output dimension affects cost, latency, and capacity.

### 3.1 What dimension affects

- **Storage.** Each vector takes `dimension × 4 bytes` (float32) or `dimension × 1 byte` (int8) per document. For 100M documents at 1536 dimensions float32: ~615 GB. At 768 dimensions int8: ~77 GB. The 8× difference matters at scale.
- **Index size.** The vector index (HNSW or similar) scales with dimension and document count. Larger dimensions → larger index → more RAM.
- **Query latency.** Higher dimensions slow distance computation; effect is modest at modern hardware but real.
- **Query cost** (for provider-hosted). Some providers charge by dimension; lower dimensions are cheaper per query.
- **Quality.** Higher dimensions *can* improve quality, but the curve is often flat past a point.

### 3.2 The quality-vs-dimension curve

Empirically, for most general-purpose embedding models in 2026:

| Dimension | Quality (relative on typical RAG eval) | Cost (storage + compute) |
| --- | --- | --- |
| 256 | 88% | 100% baseline |
| 512 | 93% | 200% |
| 768 | 95% | 300% |
| 1024 | 96% | 400% |
| 1536 | 96.5% | 600% |
| 3072 | 97% | 1200% |

Numbers are illustrative; shape is typical. The quality-per-dimension marginal gain flattens fast. Cost grows linearly with dimension.

The lesson: lower dimensions often suffice. Many teams ship at the model's max dimension by default and pay 6× the cost for 1.5% quality gain over a sized-down alternative.

### 3.3 Matryoshka embeddings

Modern embedding models (OpenAI text-embedding-3, Voyage 3, BGE-M3) are trained with Matryoshka Representation Learning (MRL) — the model produces vectors where the first N dimensions are themselves a useful embedding. The team can truncate at runtime to the right dimension for the workload.

The pattern:

1. Generate full-dimension embeddings at ingestion.
2. Store both full and truncated dimensions (or compute truncated on demand).
3. Use the truncated dimension for the index and queries.
4. Optionally re-rank top results with the full-dimension embedding for precision on the head.

The two-stage pattern (truncated for recall, full for precision) often captures the quality of full dimensions at most of the cost saving.

### 3.4 Quantisation

Beyond dimension, quantisation reduces storage further:

- **int8 quantisation.** 4× storage reduction vs float32. Quality impact usually <1%.
- **binary quantisation.** 32× reduction. Quality impact higher but acceptable for some workloads (large-scale first-stage retrieval).
- **product quantisation (PQ).** Variable reduction; complex; used in some vendor implementations.

The dimension + quantisation choices together are the cost lever; treat them as such.

### 3.5 The "we used the default" failure

Many teams ship with the embedding model's default dimension (often the max). The cost is 3–5× what a deliberate choice would have produced.

The corrective: per workload, run eval at multiple dimensions; pick the dimension where quality drops below the threshold. For a stable workload, the choice is once-and-done.

---

## 4. The model-selection decision

The decision matrix.

### 4.1 The criteria

For each candidate model:

- **Quality on the team's eval set.** The proxy that matters; published benchmarks are filter, eval is decision.
- **Cost.** Per-token for provider-hosted; infra cost for self-hosted; project across volume estimates.
- **Latency.** Embedding latency per call; ingestion throughput; query latency.
- **Dimension.** With Matryoshka, multiple dimension options per model.
- **Domain fit.** General vs specialised; if specialised available for the domain, test it.
- **Operational fit.** Does the team have the platform for self-hosting if open-weight?
- **Compliance fit.** Data residency, provider terms, audit requirements.
- **Migration path.** What does it take to switch later?

### 4.2 The decision process

1. **Filter on compliance and operational fit.** Eliminate models that don't meet hard constraints.
2. **Filter on cost envelope.** Eliminate models whose projected cost exceeds the budget.
3. **Eval the remaining candidates.** On the team's golden set; report quality, latency, cost.
4. **Decide.** Quality, cost, latency tradeoff; document the decision rationale.
5. **Re-evaluate quarterly.** New models emerge; old models change pricing; the choice may shift.

### 4.3 The "stay on the current default" inertia

Teams often stay on their initial embedding choice for years past optimal. Inertia reasons:

- Migration is expensive (perceived).
- "The current model works."
- No one is owning the embedding choice.

The corrective: a designated owner reviews the embedding strategy annually. The review may conclude "stay" or "migrate"; either is fine; the review is the discipline.

### 4.4 The org-wide standard temptation

Some teams standardise on one embedding model org-wide. The benefit: shared infrastructure, shared tooling, one thing to upgrade.

The downside: different workloads have different needs. The team's chatbot may not need the same embedding quality (or pay the same cost premium) as the clinical-document retrieval.

Recommendation: a recommended default with explicit room for per-workload deviation. The platform team supports the default; deviations are justified by per-workload data.

### 4.5 The model-family commitment

Some teams commit to a model family (all OpenAI; all Voyage; all open-weight). The reason: cross-model migrations are harder than within-family migrations (OpenAI ada → text-embedding-3 is easier than OpenAI → Voyage).

The commitment isn't free — it constrains future choices. Make it consciously, not by default.

### 4.6 The hybrid: multiple embedding models in one system

Some workloads use multiple embedding models in parallel:

- One for general retrieval; one for code retrieval.
- One small for first-stage; one large for re-ranking.
- One per language for multilingual workloads.

Multiple models mean multiple indexes, multiple ingestion pipelines, multiple migration plans. The complexity is real; only adopt when the quality gain justifies.

---

## 5. Versioning embeddings as artefacts

Embeddings inherit the embedding model's version. Treat this explicitly.

### 5.1 The embedding model version

Every embedding the team stores was produced by a specific model version:

- "OpenAI text-embedding-3-large@2024-01" (or whatever versioning the provider uses).
- "BGE-M3@v1.5".
- "internal-clinical-bert@2025-03-14".

The version is stored alongside the vector. Without it, the team can't tell which model produced a given embedding; cross-model queries silently underperform.

### 5.2 The model-version registry

A registry maps the embedding model version → its endpoint / weights / characteristics:

```yaml
embedding_models:
  - id: openai/text-embedding-3-large
    version: "2024-01"
    dimensions: 3072
    matryoshka_dimensions: [256, 512, 768, 1024, 1536, 3072]
    provider: openai
    api: https://api.openai.com/v1/embeddings
    cost_per_million_input_tokens_usd: 0.13
    
  - id: openai/text-embedding-3-large
    version: "2024-07-pruned"
    dimensions: 3072
    deprecated: true
    replacement: openai/text-embedding-3-large@2024-08
```

The registry supports query-time and ingestion-time decisions: which model to use; whether a stored embedding is current or deprecated.

### 5.3 The "embedding model deprecated" event

Providers occasionally deprecate models. When they do:

- The team's existing embeddings are produced by a model that's going away.
- New ingestion can use the replacement.
- Queries against old embeddings need to either be re-embedded with the replacement model or remain on the deprecated model (until removal).

The architectural choice: re-embed everything (expensive but clean) or run both old and new for a transition period (operationally complex).

### 5.4 The "embedding model upgraded" event

Even without a forced deprecation, the team may want to upgrade for better quality. Same dynamic: existing embeddings need a strategy; queries embedded with the new model don't match old embeddings.

### 5.5 The query-time decision

For each query, the system decides which embedding model to use:

```
For corpus C, retrieve documents indexed with model_version V:
  - Embed the query with V
  - Query the index
  - Return results
```

The system tracks per-corpus the embedding model version; queries are embedded accordingly. Cross-version queries don't work — the system errors loudly rather than silently mixing.

### 5.6 The lineage

Per [lineage-and-provenance.md](./lineage-and-provenance.md), each embedding's lineage records:

- Source document (and source document version).
- Chunking method (and version).
- Embedding model (and version).
- Ingestion time.

When the team debugs "this document isn't being retrieved," the lineage answers "what embedding produced its vector, when, with what method." Without lineage, debugging is guesswork.

---

## 6. The migration playbook

The hardest operational task in embedding strategy: changing the model without breaking production.

### 6.1 The naïve approach (don't do this)

"Just switch to the new model" — change the configuration; new ingestion uses the new model; old documents have old embeddings; queries embedded with the new model don't match old embeddings; retrieval quality plummets overnight.

The naïve approach is what teams discover when they don't plan migration. The fix is a multi-week index rebuild done under pressure.

### 6.2 The planned approach

The migration is staged.

**Stage 1: dual-write.** New ingestion produces embeddings with both old and new models. Old documents remain on the old model. The index has documents from both models (tagged with their model version).

**Stage 2: backfill.** Old documents are re-embedded with the new model. Backfill runs over weeks (rate-limited to control cost and load). Each document's "embeddings_new" field populates.

**Stage 3: dual-query during transition.** Queries are embedded with the new model. The index supports queries against documents that have the new embedding (which grows during backfill). Documents still on old-only embedding are excluded until they're backfilled, or queries are issued against both indexes and results merged.

**Stage 4: cutover.** When backfill is complete (every document has new embeddings), queries shift to new-only. Old embeddings are retained for rollback.

**Stage 5: cleanup.** After a stable period, old embeddings are deleted.

The full migration typically takes 4–12 weeks depending on corpus size and rate-limits.

### 6.3 The backfill cost

Backfilling 100M documents at $0.13 per million tokens (OpenAI text-embedding-3-large pricing) at ~500 tokens per document is:

100M × 500 / 1M × $0.13 = $6,500.

Plus the new storage for the new embeddings (~615 GB for 1536-dim float32, less with quantisation).

The cost is non-trivial; budget for it as part of migration planning.

### 6.4 The eval-during-migration

During the migration, eval compares old-model vs new-model performance on the team's golden set:

- Verify the new model is better (or at least equivalent) before committing.
- Verify quality across all corpus subsets, not just the average.
- If eval reveals the new model underperforms on a specific subset, decide: fix the new model's behaviour on that subset, or stay with the old model.

### 6.5 The rollback path

The migration plan includes the rollback path:

- If the new model proves worse in production, revert to old (queries embedded with old model; documents still have old embeddings).
- Old embeddings retained until well past the stable period (typically 30–90 days after cutover).

The retention is insurance; storage cost is small relative to the risk of "we cut over and can't go back."

### 6.6 The per-corpus migration

Some workloads have multiple corpora (clinical guidelines vs payer policies vs internal procedures). Each can migrate independently:

- Corpus A migrates first; quality is verified; team builds confidence.
- Corpus B migrates next; benefits from corpus A's experience.
- Corpus C migrates last; or stays on the old model if migration isn't worth it.

Per-corpus migration reduces blast radius. Useful when corpora have different criticality.

### 6.7 The migration as recurring exercise

The first migration is the hardest; subsequent migrations benefit from the pattern. A team that has done one migration well has the playbook for the next. A team that has never migrated will rediscover the lessons under pressure.

Recommend: do a small migration deliberately within the first year of running RAG, even if not strictly needed. The exercise builds the muscle.

---

## 7. Operational and cost considerations

The operational layer beyond model choice.

### 7.1 Ingestion throughput

For backfill or bulk-ingestion: how many documents per second can the team embed?

- Provider-hosted: limited by API rate limits; typically 1k-10k documents/sec for chunks-sized inputs.
- Self-hosted: limited by GPU throughput; varies with model size and hardware.

The throughput determines how long backfills take. A 100M-document backfill at 1k/sec is ~28 hours of continuous embedding; rate-limit headroom and operational windows extend this to days.

### 7.2 Query latency

Per query, the embedding step is:

- Provider-hosted: ~30–200ms (network + processing).
- Self-hosted co-located: ~5–50ms.

For features where query latency matters (interactive chat), the embedding step is part of the budget. Self-hosting can be the right choice for latency-sensitive features.

### 7.3 Cost projection

Per ingestion-and-query workload:

```
ingestion_cost = documents × avg_tokens_per_doc / 1M × rate
ongoing_ingestion_cost = (per-day new docs × avg tokens) / 1M × rate × 30 (monthly)
query_cost = (queries/month × avg_tokens_per_query) / 1M × rate

total_monthly_embedding_cost = ongoing_ingestion + query_cost
```

For Meridian at ~100k queries/day × 100 tokens/query × $0.13/M tokens = $39/month for queries. Plus ~$500/month for ongoing ingestion of new content. Modest at this scale; can grow significantly with scale.

### 7.4 The cost-vs-quality re-evaluation

At higher scale, the cost equation may favour self-hosting:

- At 10M queries/day, the query cost grows; self-hosting starts looking cheaper.
- The crossover point is workload-dependent; calculate per-team.

The crossover isn't just direct cost — it includes ML platform investment, GPU operations cost, and the opportunity cost of building infra vs spending on the team's core work.

### 7.5 The "embedding service is down" scenario

If the embedding endpoint (provider or self-hosted) is unavailable:

- New ingestion fails; backlog grows.
- Queries fail; user-visible degradation.

Mitigations:

- Multi-region failover for self-hosted.
- Multi-provider failover (the team has accounts at two providers; the system can fail over).
- Embedding cache (per [caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops) *(planned)*) — recently-queried inputs cached; some queries served from cache during outages.
- Graceful degradation pattern — if embedding fails, the feature returns "unable to retrieve" cleanly rather than crashing.

### 7.6 Observability

Per workload:

- Embedding calls per minute; latency p50/p99.
- Cost per day.
- Errors and retries.
- Per-model version usage (during transitions).

The observability supports cost control and incident response. Per [retrieval-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/retrieval-instrumentation.md) in the engineering sibling.

### 7.7 The dimension reduction at scale

For very large corpora (1B+ vectors), even with low dimensions, storage is significant. Architectures:

- Two-stage retrieval: small-dimension index for first-stage; full-dimension for re-rank.
- Sharded indexes per logical partition.
- Hierarchical compression (binary first-stage; int8 re-rank; float32 only for top-N).

The complexity grows with scale; the architectural pattern justifies it only when scale demands.

---

## 8. Worked Meridian example

Meridian's embedding strategy.

### 8.1 The choice (made Q4-24)

- **Model:** OpenAI text-embedding-3-large.
- **Dimension:** 1024 (Matryoshka-truncated from 3072).
- **Why this model:** Compliance-fit (Meridian's BAA with OpenAI was already in place), quality on the team's eval (4.2/5 vs alternatives), cost manageable for projected scale, mature provider API.
- **Why this dimension:** The team ran eval at 256, 512, 768, 1024, 1536, 3072; quality at 1024 was within 0.5% of 3072 at 1/3 the storage cost. The choice was 1024 for storage + adequate quality.

### 8.2 The corpora

| Corpus | Documents | Embedding model | Dimension |
| --- | --- | --- | --- |
| Clinical guidelines | ~80k | text-embedding-3-large@2024-01 | 1024 |
| Payer policies | ~150k | text-embedding-3-large@2024-01 | 1024 |
| Internal procedures | ~25k | text-embedding-3-large@2024-01 | 1024 |
| Patient records (encounter summaries) | ~30M | text-embedding-3-large@2024-01 | 1024 |
| Code samples (developer copilot) | ~5k | voyage-code-3 | 1024 |

The fourth corpus is the largest by far; storage and ingestion economics dominate.

Note: code samples are on a different model. The team evaluated and found Voyage's code-specific model materially better on code retrieval; the deviation from the org default is justified per workload.

### 8.3 The storage

Patient records dominate:
- 30M docs × 1024 dim × 4 bytes (float32) = ~120 GB.
- Quantised to int8: ~30 GB.

Total embedding storage across all corpora: ~150 GB. Manageable.

### 8.4 The ongoing ingestion

- New encounter summaries: ~80k/day × 500 tokens × $0.13/M = ~$5.20/day. ~$160/month.
- New clinical guidelines / policies: ~100/day; negligible cost.
- Total ongoing ingestion: ~$200/month.

### 8.5 The query load

- ~100k queries/day across all features.
- Average ~100 tokens per query.
- Cost: ~$1.30/day, ~$40/month.

### 8.6 The migration story (Q3-25)

The team migrated from `text-embedding-ada-002` (the legacy choice from 2023) to `text-embedding-3-large` over Q3-25.

The migration plan:

- Stage 1 (Week 1-2): dual-write configured; new ingestion produces both ada-002 and 3-large embeddings.
- Stage 2 (Week 2-6): backfill scheduled; rate-limited to 5M docs/day; new corpus indexed alongside old.
- Stage 3 (Week 6-8): query routing switched per-corpus as backfill completed; old corpus queries still on ada-002 during transition.
- Stage 4 (Week 8): cutover complete; new embeddings serve all queries; old embeddings retained.
- Stage 5 (Week 16): old embeddings deleted after 8 weeks of stable operation.

The migration cost: ~$8,200 in embedding-API charges (3-large is more expensive than ada-002; new index storage doubled during transition).

The quality outcome: +6% on the team's eval set; +12% on the clinical-guidelines slice specifically (the new model handled domain-specific terminology better).

### 8.7 The dimension story

The initial deployment used 3072 dimensions (the default for text-embedding-3-large). After 3 months, the team ran the dimension eval:

- 256: quality 4.05 (vs 4.21 at 3072)
- 768: quality 4.18
- 1024: quality 4.19
- 1536: quality 4.20
- 3072: quality 4.21

The decision: reduce to 1024 for storage savings (4× reduction) with 0.5% quality loss.

The re-indexing took a weekend (just a truncate operation since the embeddings are Matryoshka — the dimensions are stored in the same vector; the team just stopped using the higher dimensions).

### 8.8 The owner

The platform team's "RAG infrastructure" engineer owns the embedding strategy. Annual review documented; quarterly health check; deprecation watch.

### 8.9 What worked

- **Eval before committing.** The dimension choice and the migration choice were both data-driven; production didn't surprise the team.
- **Staged migration.** No production incident during the migration; rollback ready (and untested in practice — but ready).
- **Matryoshka.** The dimension flexibility saved a separate migration effort when the team chose to reduce dimensions.
- **Per-corpus deviation.** The code-corpus uses Voyage Code; the cost of operational complexity (managing two models) is justified by the quality gain.

### 8.10 What didn't work initially

- **Org-wide standard temptation.** Early platform-team push to use ada-002 for everything was reversed when the code copilot's eval showed materially better results on voyage-code-3.
- **Started with 3072 dimensions.** Saved the storage cost of the dimension reduction by waiting for production data to inform the decision; could have been done at launch.
- **First migration was reactive.** The ada-002 → 3-large migration happened because of a quality plateau; the team learned the migration playbook the hard way. The next migration will be planned proactively.

---

## 9. Anti-patterns

### 9.1 "Use the default model and default dimension"

The team adopts whatever the framework defaults to; dimensions, model, all unexamined. Discovers later that the cost is 4× what a deliberate choice would have been.

**Corrective.** Decision process per section 4.2; dimension eval per section 3.5.

### 9.2 "Org-wide single model"

Standardisation forced even when per-workload data shows alternatives outperform.

**Corrective.** Recommended default + room for justified deviation per section 4.4.

### 9.3 "No migration plan"

Embedding model is treated as immutable; no plan for upgrades; team discovers migration is a multi-quarter project when they want it.

**Corrective.** Migration playbook per section 6; do a small migration deliberately to build the muscle.

### 9.4 "Mixed embeddings in one index without tracking"

Documents in the same index were embedded by different models; query-time confusion; quality silently degrades.

**Corrective.** Per-embedding model version tracking per section 5; system errors loudly on cross-version queries.

### 9.5 "Naïve cutover"

Switch the model overnight; old documents have old embeddings; quality plummets; emergency response.

**Corrective.** Staged migration per section 6.2.

### 9.6 "No eval against alternatives"

The team committed to a model 3 years ago; never re-evaluated; the landscape has moved.

**Corrective.** Quarterly or annual re-evaluation per section 4.2.

### 9.7 "Multi-modal without need"

A text-only workload adopts a multi-modal model "for future-proofing"; pays the cost of multi-modal complexity for no current benefit.

**Corrective.** Adopt multi-modal only when multi-modal is genuinely needed.

### 9.8 "No embedding lineage"

The team can't tell which model produced which embedding; can't debug retrieval issues; can't migrate cleanly.

**Corrective.** Lineage per [lineage-and-provenance.md](./lineage-and-provenance.md); embedding metadata per section 5.6.

---

## 10. Findings (sprint-assignable)

### ARCH-EMBED-001 — Severity: Critical
**Finding.** Embedding model and dimension were defaults; no deliberate decision documented.
**Recommendation.** Decision process per section 4.2; dimension eval per section 3.5; document rationale.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-EMBED-002 — Severity: Critical
**Finding.** No migration plan; the team cannot upgrade the embedding model without major incident.
**Recommendation.** Migration playbook per section 6; do a small migration deliberately to validate.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-EMBED-003 — Severity: Critical
**Finding.** Embeddings in indexes lack model-version tagging; cross-version mixing produces silent quality regression.
**Recommendation.** Per-embedding version metadata per section 5; system errors loudly on cross-version queries.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-EMBED-004 — Severity: High
**Finding.** Dimension chosen by default (model's max); cost is 3-5× a deliberate choice.
**Recommendation.** Dimension eval per section 3.5; reduce where quality holds.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-EMBED-005 — Severity: High
**Finding.** Org-wide single embedding model when per-workload data shows alternatives outperform.
**Recommendation.** Per-workload decision per section 4.4 / 4.6; justified deviations from default.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-EMBED-006 — Severity: High
**Finding.** No embedding-strategy owner; no annual review; strategy ossifies.
**Recommendation.** Designated owner per section 4.3; annual review documented.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-EMBED-007 — Severity: High
**Finding.** Embedding lineage absent; debugging retrieval issues requires guesswork.
**Recommendation.** Lineage per [lineage-and-provenance.md](./lineage-and-provenance.md); embedding metadata per section 5.6.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-EMBED-008 — Severity: Medium
**Finding.** No Matryoshka pattern used despite available models; dimension reduction would require re-ingestion.
**Recommendation.** Use Matryoshka-trained models per section 3.3; truncation as runtime decision.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-EMBED-009 — Severity: Medium
**Finding.** No quantisation; storage cost 4× achievable.
**Recommendation.** Quantisation per section 3.4; int8 for most workloads; eval quality impact.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-EMBED-010 — Severity: Medium
**Finding.** Embedding endpoint single point of failure; no failover strategy.
**Recommendation.** Multi-region or multi-provider failover per section 7.5.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-EMBED-011 — Severity: Medium
**Finding.** Per-workload embedding cost not tracked; cost-optimisation invisible.
**Recommendation.** Cost attribution per workload per section 7.3; dashboards.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-EMBED-012 — Severity: Medium
**Finding.** Migration plan is theoretical; never exercised; first real migration would be high-risk.
**Recommendation.** Deliberate small migration per section 6.7; validates the playbook.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-EMBED-013 — Severity: Medium
**Finding.** Specialised-domain workload (code, multilingual) using general-purpose embedding; specialised alternatives unexamined.
**Recommendation.** Eval per workload per section 4.2; consider domain-specific models.
**Owner.** ai-platform-eng + feature team, sprint N+4.

### ARCH-EMBED-014 — Severity: Medium
**Finding.** No backfill strategy for large corpora; the migration cost projection is unknown.
**Recommendation.** Backfill plan per section 6.3; rate-limit; cost projection.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-EMBED-015 — Severity: Low
**Finding.** Embedding observability lacks per-call latency tracking; embedding-side regressions invisible.
**Recommendation.** Per-call observability per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-EMBED-016 — Severity: Low
**Finding.** Two-stage retrieval (truncated first-stage, full re-rank) not used despite scale that would benefit.
**Recommendation.** Two-stage pattern per section 7.7 for high-scale workloads.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-EMBED-017 — Severity: Low
**Finding.** Deprecation watch absent; provider deprecation announcement could surprise team.
**Recommendation.** Quarterly review includes provider-deprecation status.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-EMBED-018 — Severity: Low
**Finding.** Embedding cost projections not updated when volume changes; budget surprises.
**Recommendation.** Quarterly cost re-projection per section 7.3.
**Owner.** ai-platform-eng + finance, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team choosing embeddings for a new feature:

- [ ] **Sprint 0 — requirements.** Quality bar, volume estimate, cost envelope, compliance constraints.
- [ ] **Sprint 0 — candidates.** Filter by compliance and operational fit; 3-5 candidates.
- [ ] **Sprint 1 — eval.** Per candidate on team's golden set; quality, latency, cost.
- [ ] **Sprint 1 — dimension eval.** Per chosen model, at multiple dimensions.
- [ ] **Sprint 1 — decision.** Document rationale.
- [ ] **Sprint 2 — implementation.** Index, ingestion pipeline (per engineering sibling).
- [ ] **Sprint 2 — lineage.** Per-embedding model version tracking.
- [ ] **Sprint 2 — observability.** Per-call latency, cost.
- [ ] **Sprint 3 — migration plan.** Documented; rehearsed in non-prod.
- [ ] **Sprint 3 — owner assignment.** Designated.
- [ ] **Ongoing — annual review.** Strategy re-evaluated.

For a team retrofitting embedding strategy:

- [ ] **Sprint 0 — audit.** What model, what dimension, what cost, what migration capability.
- [ ] **Sprint 1 — dimension eval.** If using default, evaluate reduction.
- [ ] **Sprint 2 — lineage.** Backfill model-version metadata if missing.
- [ ] **Sprint 3 — migration plan.** Build (even if not migrating soon).
- [ ] **Sprint 4 — first deliberate migration.** Build the muscle.

A team that completes the sequence has embedding strategy under deliberate control. A team that doesn't has constraints they don't yet know about.

---

## 12. References

- [vector-store-architecture.md](./vector-store-architecture.md) — vector store is the consumer of the embeddings.
- [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — corpus is the input to embedding.
- [lineage-and-provenance.md](./lineage-and-provenance.md) — embedding metadata is part of lineage.
- [knowledge-graph-augmentation.md](./knowledge-graph-augmentation.md) — graph as complement to vector retrieval.
- [hybrid-store-architecture.md](./hybrid-store-architecture.md) — embeddings in hybrid contexts.
- [freshness-architecture.md](./freshness-architecture.md) — re-embedding cadence interacts with freshness.
- [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — per-tenant index considerations interact with embedding choice.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — analogous decision for inference models.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/embedding-pipeline-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/embedding-pipeline-engineering.md) — pipeline depth.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md) — chunking affects what gets embedded.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-workload embedding cost tracking.
- MTEB (Massive Text Embedding Benchmark) — published benchmarks as filter (not decision).
- OpenAI text-embedding-3 documentation — Matryoshka pattern reference.
- Voyage embeddings, Cohere embed-v4 documentation — provider alternatives.
- BGE-M3, E5 papers — open-weight alternatives.
- Matryoshka Representation Learning (Kusupati et al., 2022) — the underlying training approach.
