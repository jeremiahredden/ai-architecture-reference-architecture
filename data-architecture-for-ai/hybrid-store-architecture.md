# Hybrid Store Architecture

> **Audience.** Architects designing the retrieval store for a RAG-shaped feature where vector-only isn't quite right. Tech leads weighing OpenSearch vs Vespa vs dual-engine architectures. Anyone whose vector search returns "semantically similar but factually wrong" results. **Scope.** The *architectural* decision: combining lexical (BM25), vector, and structured-filter retrieval in one query path. Single-engine vs dual-engine; rank fusion approaches; the data-modeling decisions that determine whether hybrid lifts quality. Not the engineering of the retrieval pipeline (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/retrieval-engineering.md`). Not the broader vector-store choice (see [vector-store-architecture.md](./vector-store-architecture.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Pure vector retrieval has known weaknesses. It excels at semantic similarity ("documents about X") but struggles with:

- **Lexical specificity.** "Find document containing the exact phrase 'patient_id 12345'" — vector retrieval may find semantically similar documents about patient identifiers without finding the specific one.
- **Rare-token recall.** Domain-specific terms, named entities, codes (ICD-10, NPI numbers) — vector models may not represent these distinctively.
- **Filtering on structured attributes.** "Documents from tenant X, in the last 30 days, with status=published" — vector retrieval combined with metadata filters needs careful architecture.
- **Out-of-distribution content.** Documents the embedding model wasn't trained on (highly specialised jargon, code, new domains).

Hybrid retrieval — combining lexical (typically BM25), vector, and structured filtering in one query path — addresses these weaknesses. A query "find recent encounters mentioning patient_id 12345 about cardiology" benefits from: BM25 for the exact ID match, vector for the semantic "cardiology" understanding, structured filter for "recent."

The architectural decision: how to combine these signals. Single-engine systems (OpenSearch, Vespa, Elasticsearch with vector plugins) handle all three within one engine. Dual-engine architectures (separate vector store + separate lexical store, results merged) provide more flexibility at higher complexity. The choice affects operational shape, query latency, cost, and the data-modeling decisions the team makes.

The decision is consequential. The wrong architecture either over-engineers (dual-engine when single-engine would suffice) or under-delivers (single-engine when the workload genuinely needs flexibility). Most teams' hybrid retrieval journeys involve picking one architecture, encountering its limits, and either migrating or accepting the limits.

This document is opinionated about four things:

1. **Pure vector isn't always enough; hybrid is often the right starting point.** Many production RAG features ship with hybrid because vector alone leaves quality on the table.
2. **The single-engine pattern is the right default.** Operational simplicity matters; dual-engine is justified by specific requirements, not assumed.
3. **Rank fusion is engineering, not magic.** RRF (Reciprocal Rank Fusion) is the common default; tuning matters; eval is the decision input.
4. **Data modeling determines whether hybrid lifts quality or just adds cost.** Ad-hoc hybrid often underperforms; deliberate modeling captures the structure that hybrid retrieval can use.

Structure: (2) what hybrid retrieval is and isn't; (3) the architecture variants; (4) rank fusion approaches; (5) data modeling for hybrid; (6) operational considerations; (7) migration paths between variants; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. What hybrid retrieval is and isn't

Clarify the scope before discussing architecture.

### 2.1 What hybrid combines

Three primary signals:

- **Lexical (BM25).** Token-based matching with frequency weighting. Excels at exact and near-exact matches; handles rare terms; deterministic.
- **Vector (embedding-based).** Semantic similarity. Excels at conceptual matches; handles paraphrasing; not deterministic across model upgrades.
- **Structured filters.** Metadata-based selection. Tenant, date range, document type, status, custom fields. Exact match on attributes.

A hybrid query uses two or three of these signals; the results are combined.

### 2.2 What hybrid is not

- **Not a vector store with a metadata filter.** That's vector + filter; hybrid implies vector + lexical (and optionally filter).
- **Not a separate keyword search alongside vector.** The integration is in the query path; the user sees one result set.
- **Not always needed.** Pure vector suffices for many workloads (semantic question-answering over a corpus where lexical specificity isn't important).
- **Not the same as "RAG with reranking."** Reranking is post-retrieval; hybrid is during retrieval.

### 2.3 When hybrid is right

- **Mixed-content corpus.** Some content benefits from semantic matching; some needs exact/rare-term matching.
- **Domain-specific terminology.** Medical codes, legal citations, product SKUs — terms vector models may not embed well.
- **Multi-attribute queries.** Queries naturally combine semantic intent with structured constraints.
- **Recall matters.** The team can't afford to miss results that lexical retrieval would have caught.

### 2.4 When pure vector is right

- **Conversational QA over knowledge bases.** Questions and answers tend to be semantic, not lexical.
- **Summarisation-shaped retrieval.** "Documents about X" — pure semantic.
- **Corpus the embedding model handles well.** Mainstream content, English text, common domains.
- **Operational simplicity matters more than the quality gain.**

### 2.5 The eval reveals the answer

The team's eval on the team's content is the decision input. Run vector-only, lexical-only, and hybrid baselines. Compare quality on the eval set. If hybrid materially outperforms, adopt. If not, keep it simple.

In practice: most enterprise / domain-specific workloads benefit from hybrid; most consumer / mainstream-content workloads don't need it. The pattern is not universal.

### 2.6 The "lexical is obsolete" misconception

A common framing: "vector retrieval superseded lexical." The framing is wrong. Vector and lexical are complementary; each has cases the other misses. The 2026 production state is that both are alive and often used together.

---

## 3. The architecture variants

Three primary architectures.

### 3.1 Variant A — Single-engine

One search engine handles lexical, vector, and structured filtering. Examples: OpenSearch (with k-NN plugin), Elasticsearch (with dense vector field), Vespa, Solr (with vector plugin), Weaviate (with BM25 + vector).

```
Query → single engine → 
  internal: BM25 score + vector score + filter →
  combined ranking → results
```

**Pros.**
- Operational simplicity (one system).
- Lower latency (one round trip).
- Built-in rank fusion.
- Mature; established.

**Cons.**
- Engine-specific optimisations may favour one signal over another.
- Vector capabilities may lag dedicated vector stores.
- Scaling vector and lexical workloads together rather than independently.

**When right.** Default for most hybrid workloads. The simplicity warrants it unless a specific requirement forces dual-engine.

### 3.2 Variant B — Dual-engine

Separate vector store and lexical store; query both; merge results in application code.

```
Query → 
  branch 1: vector store → vector results
  branch 2: lexical store → BM25 results →
  application: rank fusion → combined results
```

**Pros.**
- Best-in-class engine per modality (purpose-built vector store + mature lexical store).
- Independent scaling of vector and lexical workloads.
- Engine choices can change independently.

**Cons.**
- Operational complexity (two systems).
- Higher latency (two round trips or parallel queries).
- Rank fusion is application code (engineering investment).
- Cross-store consistency (a document indexed in vector store may not yet be in lexical store).

**When right.** When vector workload genuinely benefits from a dedicated vector store (extreme scale, specific features); when lexical workload benefits from a mature engine (Elasticsearch-specific features the single-engine option doesn't have); when independent scaling matters.

### 3.3 Variant C — Vector-with-structured-filter

The simplest hybrid: a vector store with structured-attribute filtering. Not full hybrid (no BM25) but addresses the metadata-filter case.

```
Query → vector store with filter → results matching filter, ranked by vector similarity
```

**Pros.**
- Operational simplicity.
- Most vector stores support this natively (pgvector with WHERE clauses, Pinecone with metadata filters, Weaviate with where clauses).
- Lower latency than dual-engine.

**Cons.**
- Doesn't address lexical-specificity needs.
- Filter performance varies by store; large result sets may be expensive.

**When right.** When the workload's needs are vector + structured filter (no lexical); this is the cheapest hybrid that delivers value.

### 3.4 The decision matrix

| Criterion | Variant A (single-engine) | Variant B (dual-engine) | Variant C (vector + filter) |
| --- | --- | --- | --- |
| Operational simplicity | High | Low | High |
| Query latency | Medium | Higher | Lower |
| Lexical retrieval | Yes | Yes (best-in-class) | No |
| Vector retrieval | Yes (engine-dependent) | Yes (best-in-class) | Yes |
| Structured filter | Yes | Yes (varies) | Yes |
| Independent scaling | No | Yes | No |
| Best for default | Yes | When justified | When lexical not needed |

Most production hybrid workloads land on Variant A. Variant B is for specific requirements; Variant C for simpler needs.

### 3.5 The migration path

Teams often start at Variant C (vector + filter), discover the lexical need, migrate to Variant A. Few teams migrate from A to B; the operational cost increase rarely justifies.

Plan the migration path before committing; section 7 covers the details.

### 3.6 The "Vespa as one-size-fits-all" temptation

Vespa is a powerful single-engine option (lexical + vector + ML ranking). Pitched as a complete solution for retrieval needs.

The pitch is sometimes right. Vespa fits high-end workloads that benefit from its sophisticated ranking and scale.

The pitch is sometimes overkill. For small-to-medium workloads, simpler engines (OpenSearch, pgvector with extensions) suffice with lower operational cost.

Evaluate against scale and complexity needs, not vendor positioning.

### 3.7 The OpenSearch / Elasticsearch consideration

OpenSearch and Elasticsearch are mature single-engine choices. Considerations:

- License: OpenSearch is fully open-source (Apache 2.0); Elasticsearch's license changes have created uncertainty.
- Vector capabilities: improving rapidly; in 2026 both are credible for many workloads (10M-100M vectors); for extreme scale (1B+ vectors), purpose-built vector stores may still outperform.
- Ecosystem: large communities, many integrations.

For most teams: OpenSearch is the safer default; Elasticsearch where the team has existing investment.

---

## 4. Rank fusion approaches

How signals combine matters. Several common approaches.

### 4.1 Reciprocal Rank Fusion (RRF)

The default. For each document, compute a score from each ranking system based on rank position (not raw scores):

```
RRF_score(doc) = sum over rankings r of 1 / (k + rank_in_r(doc))
```

Where k is typically 60 (a small constant). Documents that rank high in multiple systems get high RRF scores.

**Pros.**
- Doesn't require score normalisation (works with any score scale).
- Robust to score-distribution differences.
- Simple to implement; tunable via k.

**Cons.**
- Ignores score magnitudes (high-confidence and low-confidence matches treated similarly).
- One-size-fits-all weighting.

The default for most hybrid implementations.

### 4.2 Weighted score combination

Linear combination of normalised scores:

```
final_score = w_lex * normalised_bm25_score + w_vec * normalised_vector_score
```

Weights tuned via eval. Requires score normalisation (BM25 scores are unbounded; vector cosine scores are typically -1 to 1).

**Pros.**
- Captures score magnitudes.
- Tunable per-query-class.

**Cons.**
- Normalisation is engineering-heavy and brittle.
- Weights can overfit to eval; underperform on production.

Used when teams have invested in tuning and have specific weighting needs.

### 4.3 Learned fusion (learning-to-rank)

A model learns to combine signals based on training data. Common in mature retrieval systems (Vespa has built-in ML ranking; OpenSearch has Learning to Rank plugin).

**Pros.**
- Captures non-linear combinations.
- Adapts to per-query-class patterns.
- Best quality when properly trained.

**Cons.**
- Training data requirement (graded relevance judgments).
- Engineering complexity.
- Model maintenance ongoing.

Used at high scale where the investment is justified.

### 4.4 Cascading retrieval

First-stage retrieval (cheap) returns top-N; second-stage (expensive) reranks. Per-stage scoring approaches can differ.

```
Stage 1: hybrid (BM25 + vector) → top 100
Stage 2: reranker (cross-encoder LLM judge) → top 10
```

Common production pattern; the rerank addresses fusion quality issues at small additional cost.

### 4.5 Per-query-class fusion

Different queries benefit from different fusion weights:

- "Find documents about X" → heavier vector weight.
- "Find documents containing 'XYZ-12345'" → heavier lexical weight.

Implementation: query classification → per-class weights. Adds complexity; worth it when query classes have materially different optimal weights.

### 4.6 The "tune fusion to my eval" trap

A common pattern: tune fusion weights extensively on the eval set; achieve high eval score; ship; production reveals the weights don't generalise.

Mitigation: holdout eval set never used for tuning; production-trace eval for ongoing validation; periodic re-tuning rather than one-shot.

### 4.7 The default recommendation

For most teams: start with RRF, default k=60. Evaluate. Move to weighted combination or learned ranking only if eval shows RRF is leaving meaningful quality on the table.

---

## 5. Data modeling for hybrid

Whether hybrid lifts quality depends on what's in the index.

### 5.1 The fields matter

A document indexed for hybrid retrieval has multiple fields:

- **Full-text field for BM25.** The content the team wants lexically matchable. Often the entire chunk or document text.
- **Vector field for embedding.** The same content (or a summary) embedded.
- **Structured metadata fields.** Tenant_id, document_type, date, status, custom attributes.
- **Identifiers.** Document ID, chunk ID, source URL.

The fields are designed deliberately; not "throw everything in."

### 5.2 The chunking decision

Hybrid retrieval works at chunk granularity (typically). The chunking strategy affects both signals:

- **Chunks too small.** Lexical matching may match isolated words without context; vector retrieval misses broader semantic context.
- **Chunks too large.** Both signals dilute; relevance is averaged.

The chunking is engineered per the engineering sibling's [chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md). For hybrid: typically 200-800 tokens per chunk, depending on content.

### 5.3 The "what to embed" decision

Sometimes the embedded content differs from the lexically-indexed content:

- Embed a summary; index the full text lexically. Reduces vector storage; lexical handles term-specific queries.
- Embed the text; index both text and a metadata-derived "context" field lexically. The metadata adds search-anchored content.

The deliberate engineering produces better results than the default "embed and index the same text."

### 5.4 The structured metadata schema

The structured fields are the team's metadata schema. Decisions:

- **Tenant_id.** Always present; used for filtering and isolation (per [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/)).
- **Document type / source.** Enables source-filtered retrieval ("only clinical guidelines").
- **Date fields.** Enable recency filtering and sorting.
- **Status / visibility.** Published vs draft; active vs deprecated.
- **Domain-specific fields.** Per the workload (patient_id for clinical, account_id for CRM, etc.).

Schema evolution per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md).

### 5.5 The boost fields

Some engines support per-field boost weights (lexical fields get different importance):

- **Title field.** Boosted; matches in title weighted higher.
- **Body field.** Standard weight.
- **Tags field.** Boosted; structured tags carry intent.

Tuned via eval; per-field boost is one of the data-modeling levers.

### 5.6 The synonyms and analyzers

For lexical retrieval, analyzer choice matters:

- **Tokenisers.** How text breaks into tokens.
- **Synonyms.** Domain-specific synonym lists (medications by brand/generic; diagnoses by code/name).
- **Stemming / lemmatisation.** Reduce word forms to roots.

The analysis pipeline is engineering; default analyzers often underperform for domain content.

### 5.7 The "hybrid for everything" failure

Some teams enable hybrid retrieval on all queries even when only one signal helps. The lexical work is wasted; the cost compounds.

Mitigation: per-query-type hybrid decisions (some queries are lexical-only, some vector-only, some hybrid). Adds complexity; worth it when query types are distinct.

---

## 6. Operational considerations

The infrastructure layer.

### 6.1 Capacity planning

For a hybrid store:

- **Storage.** Vector dimensions × document count for vector field; full-text storage for BM25; metadata for structured fields.
- **RAM.** Indexes need RAM; HNSW vector indexes are RAM-intensive; lexical indexes have caches.
- **CPU.** Query throughput is CPU-bound for both signals.

Capacity depends on the workload; benchmark with realistic data.

### 6.2 Indexing throughput

Hybrid indexing is more expensive than single-signal:

- The document is both lexically analyzed (tokenisation, analysis) and embedded (LLM API call).
- Index updates for both signals.

Bulk indexing is typically 100-1000 documents per second on commodity hardware; varies with engine, document size, embedding model.

### 6.3 Query latency

Per-query overhead:

- Single-engine: ~50-200ms typical for hybrid retrieval over 10M documents.
- Dual-engine: parallel queries + merge; ~100-300ms.
- Plus embedding time for the query (50-100ms for provider-hosted, 5-50ms for co-located).

For interactive features, this is acceptable; for very-low-latency requirements, optimisation needed.

### 6.4 Cost

Hybrid costs:

- Storage cost (vector + lexical indexes).
- Query cost (compute per query).
- Embedding cost (per indexed document and per query).

For a typical 10M-document corpus with hybrid retrieval: $1k-5k/month infrastructure; varies widely.

### 6.5 Observability

Per [retrieval-observability.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-observability.md) in the engineering sibling. Hybrid-specific:

- Per-signal contribution to final ranking (which results came from lexical, which from vector, which from both).
- Per-query: what was the query class, which signals fired, what was the latency contribution per signal.
- Rank fusion behaviour (are signals agreeing or disagreeing?).

The observability supports diagnosing quality issues and tuning fusion.

### 6.6 Multi-tenancy

Per [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) considerations. Tenant_id as a structured filter is the common pattern; the hybrid query always filters by tenant.

### 6.7 Backup and DR

The index is reconstructible from source; backup is optional for the index itself (re-indexing from source is the fallback). Backup of metadata and configuration is necessary.

### 6.8 Engine upgrades

Single-engine: engine upgrades may affect both lexical and vector behaviour.

Dual-engine: each engine upgrades independently; coordination needed.

In either case: test upgrades on non-prod; monitor eval scores post-upgrade.

---

## 7. Migration paths between variants

The team's hybrid architecture may evolve over time.

### 7.1 From vector-only to Variant C

Add structured metadata fields to existing vector store; queries add filters. Most vector stores support this natively; minimal migration work.

### 7.2 From Variant C to Variant A

The team realises lexical retrieval is needed. Options:

- **Add a lexical engine alongside.** Becomes Variant B (dual-engine); operational complexity.
- **Migrate to a hybrid engine.** Re-index all content in the new engine; non-trivial migration.

The migration to single-engine is the cleaner long-term path. The team that picks dual-engine often regrets the operational complexity.

### 7.3 From Variant A to Variant B

The team has scaling or feature needs the single-engine can't meet. Migration involves:

- Choose vector store and lexical store.
- Re-index in each.
- Implement rank fusion in application code.
- Migrate gradually (one corpus at a time).

The migration is significant; cost / benefit analysis warranted.

### 7.4 Across single-engine choices

E.g., from OpenSearch to Vespa. The migration:

- New engine indexed in parallel.
- Eval comparison (does the new engine outperform on the team's queries?).
- Tenant-by-tenant cutover.
- Old engine retired after stable period.

### 7.5 The migration cost

Re-indexing a large corpus:

- Embedding cost (if changing embedding alongside).
- Lexical indexing cost (CPU and storage).
- Engineering time.
- Risk during the transition.

A 100M-document migration is typically a 2-3 month effort. Budget accordingly.

### 7.6 The "we'll just live with current limitations" alternative

Sometimes the right answer is to live with limitations. The migration cost may not justify the gain. The team's eval data informs this.

If eval shows current architecture's limitations are minor for the team's workload, the answer is "stay." If eval shows significant gain available, migration is justified.

---

## 8. Worked Meridian example

Meridian's hybrid store architecture.

### 8.1 The choice (made Q1-25)

- **Variant A (single-engine):** OpenSearch with k-NN plugin.
- **Why this choice:** Meridian had existing OpenSearch deployment for application logs; vector capabilities were adequate for the projected scale; operational team familiar.
- **Why hybrid:** Clinical content benefits from both lexical (medication codes, ICD codes, exact names) and vector (semantic question-answering). Pure-vector eval showed 4.05 vs hybrid 4.22 on the team's golden set.

### 8.2 The corpora

| Corpus | Documents | Hybrid? | Rationale |
| --- | --- | --- | --- |
| Clinical guidelines | ~80k | Yes | Both signals beneficial; rare clinical terms |
| Payer policies | ~150k | Yes | Policy IDs and codes need lexical |
| Internal procedures | ~25k | Yes | Stable terms need lexical |
| Patient records | ~30M | Yes | Patient IDs and codes are essential lexical matches |
| Code samples (developer copilot) | ~5k | Hybrid (different engine) | Voyage code embedding + lexical code search |
| API documentation | ~12k | Vector-only | Semantic question-answering; no specific term needs |

Most corpora use hybrid; one is vector-only. The decision is per-corpus.

### 8.3 The data modeling

Each document has fields:

```yaml
fields:
  - id: chunk_id (keyword)
  - tenant_id: keyword (always filtered)
  - document_id: keyword
  - document_type: keyword (clinical_guideline, payer_policy, etc.)
  - source: keyword
  - published_date: date
  - status: keyword (active, archived)
  - title: text (boosted 2x in BM25)
  - body: text (BM25 standard weight)
  - body_vector: dense_vector (1024 dim, from text-embedding-3-large@2024-01)
  - tags: keyword[] (for filtering)
  - patient_id: keyword (for patient records only; for cross-record traversal)
```

The schema is per-corpus with shared core fields.

### 8.4 The rank fusion

RRF with k=60. Tuned briefly at launch (k=40 vs k=60 vs k=100 on eval; k=60 won marginally).

Re-tuning attempted twice in 16 months; no statistically-significant gain from alternative weights. Sticking with RRF default.

### 8.5 The query flow

```
User query → 
  embedding (OpenAI text-embedding-3-large, 1024 dim) →
  OpenSearch hybrid query (BM25 over title + body, k-NN over body_vector, structured filter on tenant + status + date range) →
  top 25 results →
  reranker (Cohere rerank-3) →
  top 10 results →
  to model context
```

The reranker is a post-retrieval step; not part of the hybrid retrieval itself. Adds ~50ms latency, ~$0.001 cost; quality gain ~3%.

### 8.6 The capacity

- ~30M documents indexed.
- Storage: ~200 GB total (vector + lexical + metadata).
- RAM: ~50 GB across the cluster.
- Hardware: 6 nodes, ~$3k/month infrastructure.

Scaling target: 100M documents over the next year; current architecture handles with cluster expansion.

### 8.7 The query performance

- p50 latency: 120ms.
- p99 latency: 280ms.
- Throughput: ~500 queries/sec sustained.

Within budget for interactive features.

### 8.8 What worked

- **Single-engine simplicity.** Operational burden manageable; the team handles upgrades, capacity, monitoring as part of broader OpenSearch operations.
- **Per-corpus decisions.** Code samples on a different engine (specialised); API docs vector-only (no lexical need); both data-driven decisions.
- **RRF default.** Simple; effective; didn't need elaborate tuning.
- **Reranker as polish.** Cheap; meaningful quality gain; standard pattern.

### 8.9 What didn't work initially

- **Pure-vector first.** Initial deployment was vector-only; clinical staff reported "I searched for medication 'metformin' and the system returned similar diabetes documents but not the metformin-specific monograph." Lexical was added after this feedback.
- **Default analyzers.** Initial setup used OpenSearch defaults; clinical synonyms (brand/generic medication names) weren't matching. Synonym lists added per the clinical-content team's expertise.
- **No per-query observability.** Diagnosing "why did this query return this result" was hard. Per-signal contribution metadata added in Q2-25.

### 8.10 The future path

The team considers:

- **Migration to Vespa.** Examined Q4-25 for the analytics-warehouse use case (which has very large scale); decided OpenSearch's capabilities are sufficient for current workload; revisit annually.
- **Learned ranking.** Considered for the care-coordinator's query mix; the eval gain projected isn't sufficient to justify the engineering investment yet.

Neither is planned; both are watched.

---

## 9. Anti-patterns

### 9.1 "Pure vector everywhere"

The team adopts vector retrieval exclusively; never evaluates whether lexical would help.

**Corrective.** Per-corpus eval per section 2.5; adopt hybrid where it helps.

### 9.2 "Dual-engine by default"

The team picks dual-engine without specific need; carries unnecessary operational complexity.

**Corrective.** Single-engine default per section 3.1; dual-engine when justified.

### 9.3 "Tune RRF on the eval set"

Fusion weights tuned extensively on eval; overfit; production underperforms.

**Corrective.** Holdout discipline per section 4.6.

### 9.4 "Hybrid on all queries"

Every query runs the full hybrid pipeline even when only one signal helps.

**Corrective.** Per-query-class hybrid decisions per section 5.7.

### 9.5 "Default analyzers"

Lexical retrieval uses out-of-box analyzers; domain-specific terms don't match well.

**Corrective.** Analyzer engineering per section 5.6; domain synonyms.

### 9.6 "Embed everything; index nothing lexically"

Documents are embedded but full text isn't lexically indexed; lexical retrieval can't help even though hybrid was intended.

**Corrective.** Full text in both signals per section 5.3.

### 9.7 "No reranker"

Hybrid retrieval is the final step; the team accepts top-K from rank fusion without rerank polish.

**Corrective.** Reranker as cheap quality step; per section 4.4.

### 9.8 "No per-signal observability"

When quality issues emerge, the team can't tell which signal underperformed.

**Corrective.** Per-signal contribution metadata per section 6.5.

---

## 10. Findings (sprint-assignable)

### ARCH-HYBRID-001 — Severity: Critical
**Finding.** Pure-vector retrieval used for content that benefits from lexical; rare-term recall poor.
**Recommendation.** Per-corpus eval per section 2.5; adopt hybrid where it helps.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-HYBRID-002 — Severity: High
**Finding.** Dual-engine architecture adopted without specific need; operational complexity high.
**Recommendation.** Migration to single-engine evaluated per section 7.2; simpler architecture if justified.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-HYBRID-003 — Severity: High
**Finding.** Rank fusion not implemented; lexical and vector results merged ad-hoc; quality variable.
**Recommendation.** RRF as default per section 4.1; evaluated against alternatives.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-HYBRID-004 — Severity: High
**Finding.** Lexical analyzer defaults used; domain terms (medical codes, synonyms) don't match.
**Recommendation.** Analyzer engineering per section 5.6; domain synonym lists.
**Owner.** ai-platform-eng + domain experts, sprint N+2.

### ARCH-HYBRID-005 — Severity: High
**Finding.** Per-signal contribution observability absent; quality diagnoses slow.
**Recommendation.** Per-signal metadata per section 6.5; per-query diagnostics.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-HYBRID-006 — Severity: Medium
**Finding.** Hybrid runs on all queries; cost higher than needed for queries that benefit from one signal only.
**Recommendation.** Per-query-class decisions per section 5.7; cost evaluated.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HYBRID-007 — Severity: Medium
**Finding.** Reranker absent; hybrid retrieval results not polished; quality below achievable.
**Recommendation.** Reranker step per section 4.4; eval impact measured.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HYBRID-008 — Severity: Medium
**Finding.** Per-corpus hybrid decisions not made; some corpora may not benefit from hybrid.
**Recommendation.** Per-corpus eval per section 2.5; vector-only where sufficient.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HYBRID-009 — Severity: Medium
**Finding.** Tenant_id filtering not enforced at query time; cross-tenant leakage risk.
**Recommendation.** Always-applied tenant filter per section 6.6 and [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/).
**Owner.** ai-platform-eng + security, sprint N+3.

### ARCH-HYBRID-010 — Severity: Medium
**Finding.** Per-field boost not tuned; title matches treated same as body matches.
**Recommendation.** Per-field boost per section 5.5; tuned via eval.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-HYBRID-011 — Severity: Medium
**Finding.** Embedding model and lexical analyzer not coordinated; vector and lexical signals diverge in unexpected ways.
**Recommendation.** Coordinated data modeling per section 5; verify both signals see appropriate content.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HYBRID-012 — Severity: Medium
**Finding.** Engine upgrade caused subtle ranking behaviour changes; eval regressed silently.
**Recommendation.** Pre/post-upgrade eval per section 6.8; monitor.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-HYBRID-013 — Severity: Medium
**Finding.** Capacity planning didn't account for HNSW RAM usage; cluster sized for cold storage only.
**Recommendation.** Per section 6.1; benchmark and size with realistic data.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-HYBRID-014 — Severity: Medium
**Finding.** Migration path from current architecture not documented; future changes will be improvised.
**Recommendation.** Document path per section 7; review annually.
**Owner.** architecture, sprint N+4.

### ARCH-HYBRID-015 — Severity: Low
**Finding.** Fusion weights tuned once at launch; never revisited.
**Recommendation.** Periodic re-tuning per section 4.6.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-HYBRID-016 — Severity: Low
**Finding.** Holdout eval not used for fusion tuning; risk of overfit.
**Recommendation.** Holdout discipline per section 4.6.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-HYBRID-017 — Severity: Low
**Finding.** Cross-engine consistency not verified; new documents may be in one store but not the other temporarily.
**Recommendation.** Eventual-consistency awareness per section 3.2; monitor staleness.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-HYBRID-018 — Severity: Low
**Finding.** Learned ranking not considered; significant quality gain may be available at high scale.
**Recommendation.** Evaluate per section 4.3 when scale and stable workload justify.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team starting from pure vector:

- [ ] **Sprint 0 — gap analysis.** Where does pure vector fall short? Document the cases.
- [ ] **Sprint 0 — eval baseline.** Vector-only score on the team's golden set.
- [ ] **Sprint 1 — variant choice.** Variant A typically; B or C if justified.
- [ ] **Sprint 1 — engine deployment.** Single-engine (OpenSearch typical) or dual-engine setup.
- [ ] **Sprint 1 — data modeling.** Fields per section 5.1; chunking strategy.
- [ ] **Sprint 2 — indexing.** Both signals; full text + vector + structured.
- [ ] **Sprint 2 — fusion.** RRF default per section 4.7.
- [ ] **Sprint 2 — eval.** Hybrid score vs vector-only; verify improvement.
- [ ] **Sprint 3 — analyzers.** Domain synonyms; analyzer tuning.
- [ ] **Sprint 3 — observability.** Per-signal contribution; per-query diagnostics.
- [ ] **Sprint 4 — reranker.** Post-retrieval polish.
- [ ] **Sprint 5 — multi-tenancy + filtering.** Tenant scope; structured filters.
- [ ] **Ongoing — quarterly review.** Per-corpus quality; fusion tuning; capacity.

For a team with an existing hybrid implementation:

- [ ] **Sprint 0 — audit.** What's the architecture, what's working, what's not.
- [ ] **Sprint 1 — fix worst gaps.** Often analyzers or per-signal observability.
- [ ] **Sprint 2 — eval coverage.** Vector-only, hybrid, with-rerank baselines.
- [ ] **Sprint 3 — tuning.** Fusion weights, field boosts.
- [ ] **Sprint 4 — migration planning if needed.** If current architecture has limits.

A team that completes the sequence has hybrid retrieval delivering measurable quality gain at managed operational cost. A team that doesn't has retrieval whose limitations show up as quality issues with no obvious path forward.

---

## 12. References

- [vector-store-architecture.md](./vector-store-architecture.md) — the vector side; hybrid is an extension.
- [embedding-strategy.md](./embedding-strategy.md) — embeddings feed the vector signal.
- [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — schema and corpus discipline that hybrid depends on.
- [knowledge-graph-augmentation.md](./knowledge-graph-augmentation.md) — KG as an alternative or complement to hybrid.
- [lineage-and-provenance.md](./lineage-and-provenance.md) — provenance tracking through hybrid retrieval.
- [freshness-architecture.md](./freshness-architecture.md) — index freshness for both signals.
- [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — per-tenant isolation in hybrid context.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — engineering depth on retrieval.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/reranking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/reranking-engineering.md) — rerank step in the pipeline.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/chunking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/chunking-engineering.md) — chunking decisions feed both signals.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/retrieval-observability.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-observability.md) — observability for retrieval.
- OpenSearch, Elasticsearch, Vespa, Solr, Weaviate documentation — single-engine options.
- Reciprocal Rank Fusion (Cormack, Clarke, Buettcher, 2009) — the seminal RRF paper.
- "Learning to Rank for Information Retrieval" (Liu, 2009) — broader literature on learned ranking.
