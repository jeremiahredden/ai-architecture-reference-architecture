# Vector Store Architecture

> **Audience.** Architects choosing the vector store for a new RAG-shaped feature, or evaluating whether to migrate from one. Tech leads who have been asked "why pgvector vs Pinecone" and want a structured answer. **Scope.** The *architectural* decision: which vector store, sized how, with what tenancy posture. Not the engineering of ingestion / embedding pipelines (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/`). Not the per-tenant isolation implementation (see [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Choosing a vector store is the second-most-consequential data-architecture decision in a RAG workload (after the corpus design itself). The choice constrains capacity, latency, cost, multi-tenancy posture, operational shape, and migration paths. A wrong choice is recoverable but expensive — moving 50M vectors from one store to another is a multi-quarter project.

Most teams pick their vector store by default — the team uses Postgres so they use pgvector, or the team adopts the most-popular vendor of the month — and discover the constraints later. The corrective discipline: pick deliberately, calibrated to the workload's actual characteristics, with the operational team's capacity in mind.

This document is opinionated about three things:

1. **The choice is between a small number of architectural patterns**, not between vendors. pgvector (Postgres extension), purpose-built vendor (Pinecone, Weaviate, Qdrant), hybrid in OpenSearch / Elasticsearch / Vespa, fully self-hosted. The vendor within each pattern is replaceable; the pattern is the architectural commitment.
2. **The decision is per-workload.** Different features within the same platform can use different vector stores. The Meridian Care Coordinator uses pgvector; a hypothetical search feature might use OpenSearch; an experimental low-volume feature might use a managed service. Picking one platform-wide is the simpler choice but not always the right one.
3. **Migration paths are part of the choice.** The team commits to a pattern but should know how to migrate if requirements change. The patterns have known migration costs; budget accordingly.

Structure: (2) the four patterns; (3) the decision questions; (4) capacity planning; (5) multi-tenancy considerations; (6) operational considerations; (7) migration paths; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The four patterns

Vector stores fall into four architectural patterns. Each has its own characteristic properties.

### 2.1 Pattern A: Vector extension in an existing database

**Examples.** pgvector (Postgres extension), Azure Cosmos DB vector search, MongoDB Atlas Vector Search.

**Shape.** A vector index lives inside the team's existing database. Vector retrieval is a query against the database. The database's operational story applies — backup, replication, monitoring, patching are unified with everything else in the database.

**Sweet-spot workload.** Teams already operating Postgres (or the relevant database). Small-to-medium vector counts (under ~50M). Workloads where joining vectors with relational data is valuable (per-chunk metadata, tenant attribution).

**What it gives you.**
- **Operational unification.** One database to operate. Existing backup / monitoring / DR patterns apply.
- **Relational joins.** Vector retrieval can join with the database's relational tables (tenant tables, document metadata, user permissions). Reduces application-side data assembly.
- **Per-tenant partitioning is native.** Postgres declarative partitioning gives the per-tenant storage-layer guarantee that [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) describes.
- **Familiar cost model.** Database storage and compute; no per-vector-store-specific pricing.

**What it costs.**
- **Lower throughput ceiling than purpose-built stores.** pgvector's HNSW index is competent but not optimized for very high query throughput.
- **Approximate-nearest-neighbor algorithms are limited.** Purpose-built stores have more recent ANN algorithms (DiskANN, ScaNN) and faster vector arithmetic primitives.
- **Vertical scaling limit.** Postgres can scale to large vector counts but at the high end the operational complexity grows.

**Cost shape.** Cheapest at small-medium volume. Crosses over to more expensive as volume grows past the database's comfortable limit.

### 2.2 Pattern B: Purpose-built vector vendor

**Examples.** Pinecone, Weaviate, Qdrant.

**Shape.** A managed service (or self-hosted distributed service) purpose-built for vector retrieval. Vectors live in the vendor's system; the application calls the vendor's API.

**Sweet-spot workload.** Large vector counts (>50M). High query throughput. Teams that prefer managed services over operational ownership. Workloads with sophisticated vector-search requirements (high-dimensional vectors, multi-vector indexes, real-time updates).

**What it gives you.**
- **Throughput at scale.** Built for vector workloads; can handle thousands of queries per second.
- **Recent ANN algorithms.** State-of-the-art vector indexing.
- **Managed operational footprint.** Backups, scaling, patching are the vendor's job.
- **Per-namespace tenancy primitive.** Most vendors have first-class multi-tenancy (Pinecone namespaces, Weaviate multi-tenancy collections).

**What it costs.**
- **Per-vector pricing.** Often more expensive than self-hosted equivalents at the same scale.
- **Vendor dependency.** Migration costs are real; vendor lock-in is a consideration.
- **Cross-store joins are application-side.** Vector retrieval returns IDs; the application joins with relational data separately. Adds an integration layer.
- **Regulatory / residency story varies.** Not all vendors have full BAA / FedRAMP / data-residency coverage; check carefully for regulated workloads.

**Cost shape.** Higher per-vector cost at small scale; competitive at large scale; vendor-specific.

### 2.3 Pattern C: Hybrid search engine

**Examples.** OpenSearch, Elasticsearch, Vespa.

**Shape.** A search engine that supports both lexical (BM25) and vector retrieval. Hybrid queries (combining lexical + vector) are first-class.

**Sweet-spot workload.** Workloads that need hybrid retrieval (most enterprise RAG). Teams already operating Elasticsearch / OpenSearch for non-AI search. Workloads with sophisticated query requirements (faceting, structured filtering, aggregations).

**What it gives you.**
- **Hybrid retrieval native.** Both BM25 and vector in one query; no application-side merging.
- **Mature search features.** Aggregations, faceting, sorting, complex filtering.
- **Familiar operational model** if the team already operates Elasticsearch / OpenSearch.

**What it costs.**
- **Operational complexity.** Search clusters are harder to operate than relational databases for teams without that experience.
- **Vector search is newer and may be less performant** than purpose-built vector stores at very high volume.
- **Multi-tenancy primitives are less mature** than purpose-built vector stores; per-tenant indices vs metadata filtering trade-offs.

**Cost shape.** Self-hosted Elasticsearch / OpenSearch can be cheap; managed (Elastic Cloud, AWS OpenSearch Service) is more expensive.

### 2.4 Pattern D: Self-hosted purpose-built

**Examples.** Self-hosted Weaviate / Qdrant / Milvus / Chroma.

**Shape.** A purpose-built vector store deployed on the team's infrastructure. Full operational ownership.

**Sweet-spot workload.** Teams with strong infrastructure ops capability. Very high scale where managed pricing is prohibitive. Strict data-residency requirements that limit managed-service options. Need for customization at the vector-store level.

**What it gives you.**
- **Full control.** Configuration, deployment topology, scaling decisions are all in the team's hands.
- **Cost efficiency at large scale.** Avoid per-vector vendor markups.
- **Data residency guaranteed.** Vectors stay in the team's infrastructure.

**What it costs.**
- **Full operational ownership.** Backups, patching, capacity planning, on-call.
- **Skill investment.** The team needs vector-store operational expertise.
- **Slower feature adoption.** Self-hosted versions lag managed services on new features.

**Cost shape.** Lowest at very high scale; highest operational overhead per unit of value.

### 2.5 The summary table

| Pattern | Best for | Operational shape | Cost profile | Multi-tenancy |
|---|---|---|---|---|
| Pattern A (extension) | Small-medium scale; Postgres-native teams | Existing database ops | Cheapest small; expensive at scale | Native (partitioning) |
| Pattern B (vendor) | Large scale; managed-service preference | Vendor handles | Per-vector pricing | Native (namespaces) |
| Pattern C (hybrid) | Hybrid retrieval; existing OpenSearch / Elastic | Search-cluster ops | Self-hosted cheap; managed expensive | Configurable |
| Pattern D (self-hosted purpose-built) | Very large scale; strict residency | Full ownership | Cheapest at scale; high ops | Native (varies by product) |

---

## 3. The decision questions

Six questions. Answer in writing before reading the catalogue.

### 3.1 What is the projected vector count?

- Under 1M: any pattern works; pick on other criteria.
- 1M–50M: pgvector or vendor patterns viable; hybrid (OpenSearch) viable; self-hosted overkill.
- 50M–500M: vendor or hybrid; pgvector beginning to strain.
- Over 500M: self-hosted purpose-built or top-tier vendor; pgvector usually not viable.

### 3.2 What is the query throughput?

- Under 10 QPS: any pattern.
- 10-100 QPS: most patterns; pgvector requires care.
- 100-1000 QPS: vendor or hybrid; pgvector with care.
- Over 1000 QPS: vendor with sufficient capacity or self-hosted with the right architecture.

### 3.3 Do you need hybrid retrieval (BM25 + vector)?

- If yes: hybrid pattern (OpenSearch / Vespa) or vendor patterns that support it (Pinecone hybrid, Qdrant with full-text payload). Pure pgvector requires application-side merging.
- If no (vector-only is sufficient): any pattern.

### 3.4 What multi-tenancy posture do you need?

- Per-tenant index (strongest isolation): vendor patterns and self-hosted support this; pgvector requires per-tenant database (operationally heavy).
- Per-tenant namespace (the common SaaS choice): vendor patterns native; pgvector partition native; hybrid patterns configurable.
- Shared with metadata filter (lowest isolation): any pattern supports.

### 3.5 What is the regulatory posture?

- HIPAA-regulated: confirm vendor BAA coverage; some vendors (Pinecone, Weaviate enterprise) have BAA; others do not. Self-hosted or pgvector keeps the data in your control.
- FedRAMP / GovCloud: very few vendors; usually self-hosted or in-region pgvector.
- Data residency (EU, etc.): confirm vendor region coverage; otherwise self-hosted in the right region.

### 3.6 What is the team's operational capacity?

- Single-language Python team with existing Postgres: pgvector is the lowest-friction choice.
- DevOps-mature team with strong K8s / cluster ops: any pattern.
- Small team without strong ops: vendor managed service.
- Multi-language platform with existing OpenSearch: hybrid pattern.

---

## 4. Capacity planning

### 4.1 Vector count

- Source documents × average chunks per document = chunk count.
- Chunk count × 1 (one embedding per chunk) = vector count.
- Plan for 3-5x growth (corpus expands; re-chunking strategies produce more chunks; re-embedding events double-count during migration).

For Meridian: ~50,000 chunks across clinical-guidelines + protocols + drug-interactions. With 5x growth headroom: ~250K vectors. Well within pgvector's comfort zone.

### 4.2 Dimension count

- Embedding model determines dimensions. `text-embedding-3-large` is 1536 dimensions; can be configured down to 256 for cost reduction at quality cost.
- Higher dimensions = more storage and slightly slower queries; usually negligible at small-medium scale.

### 4.3 Storage size

- Per vector: dimensions × 4 bytes (float32) + index overhead.
- 1536-dim × 4 bytes = 6,144 bytes per vector raw + ~50% HNSW index overhead = ~9KB per vector.
- 250K vectors × 9KB = ~2.25GB. Trivial.
- 50M vectors × 9KB = ~450GB. Meaningful but manageable.
- 500M vectors × 9KB = ~4.5TB. Requires careful storage planning.

### 4.4 Query throughput sizing

- p99 query latency target × concurrent queries = throughput requirement.
- A 100ms p99 latency target with 10 concurrent queries = 100 QPS sustained.
- Plan for spike capacity (typically 3-5x sustained).

### 4.5 Re-embedding events

- When the embedding model changes, every vector must be re-embedded.
- Re-embedding 250K vectors: a few hours.
- Re-embedding 50M vectors: a few days (bounded by embedding-pipeline throughput).
- Re-embedding 500M vectors: weeks; requires staged rollout.

This is a meaningful capacity planning consideration. Choose embedding models with longevity in mind.

---

## 5. Multi-tenancy considerations

The multi-tenancy choice is documented in depth in [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md). Brief here on how the vector-store pattern interacts:

### 5.1 Pattern × isolation model

| Pattern | Shared-everything | Per-tenant namespace | Per-tenant index |
|---|---|---|---|
| Pattern A (pgvector) | Single index, filter | Postgres partition or namespace column | Per-tenant database |
| Pattern B (vendor) | Single namespace, filter | Vendor namespace primitive | Multiple vendor indices |
| Pattern C (hybrid) | Single index, filter | Routing or alias | Multiple indices |
| Pattern D (self-hosted) | Single index, filter | Vendor namespace primitive | Multiple deployments |

The per-tenant namespace is the SaaS sweet spot; the per-tenant index is the strongest isolation. All patterns support all three models with varying operational shapes.

### 5.2 Per-tenant operational overhead

Per-tenant index = operational overhead per tenant. At 240 tenants:
- Pattern A: 240 databases to operate (a lot).
- Pattern B: 240 vendor indices (vendor pricing + per-index overhead).
- Pattern C: 240 OpenSearch indices (more manageable; aliases simplify).
- Pattern D: 240 self-hosted deployments (significant).

For Meridian's 240-tenant scale, per-tenant namespace inside pgvector partitioning is the operational sweet spot.

---

## 6. Operational considerations

### 6.1 Backup and disaster recovery

- Pattern A: inherits the database's backup strategy. PITR (point-in-time recovery) typically available.
- Pattern B: vendor provides backups; verify retention and recovery SLOs.
- Pattern C: cluster snapshots; cross-region replication possible.
- Pattern D: full ownership; the team designs.

### 6.2 Monitoring

- Pattern A: existing database monitoring (Datadog, CloudWatch, etc.). Add vector-specific metrics (HNSW index size, query latency by index).
- Pattern B: vendor metrics + integration with team's APM.
- Pattern C: search-cluster monitoring; well-understood patterns.
- Pattern D: full ownership.

### 6.3 Capacity scaling

- Pattern A: vertical scaling within the database; or read replicas for query throughput.
- Pattern B: vendor handles; cost-aware (auto-scaling can produce surprise bills).
- Pattern C: cluster expansion; rolling.
- Pattern D: full ownership; horizontal scaling depends on the product.

### 6.4 Vector-index maintenance

- Vector indices (HNSW, IVF, DiskANN) need periodic rebuilds as data churns.
- Pattern A: pgvector index rebuilds; lock during rebuild (or REINDEX CONCURRENTLY in newer Postgres).
- Pattern B / D: depends on vendor / product.
- Pattern C: index optimization passes.

---

## 7. Migration paths

The team can migrate between patterns; each direction has known cost.

### 7.1 Pattern A → Pattern B (growing scale)

Most common migration. The platform outgrows pgvector's comfort zone.

**Approach.**
1. Stand up the new vendor service.
2. Re-embed in parallel (if new embedding model) or copy vectors (if same model).
3. Dual-write during transition (new chunks written to both).
4. Cut reads over.
5. Decommission pgvector.

**Typical timeline.** 4-8 weeks for a moderate-scale system.

### 7.2 Pattern B → Pattern D (cost optimization at large scale)

When vendor pricing exceeds the operational cost of self-hosting.

**Approach.** Similar to 7.1 — stand up the new system, copy vectors, dual-write, cut over.

**Timeline.** 6-12 weeks. The operational platform-up takes the longer end.

### 7.3 Pattern C → Pattern B (specialization)

The team's hybrid search engine is straining under vector workload; the vector portion gets its own purpose-built store.

**Approach.** Continue using the search engine for BM25; route vector retrieval to the new vendor; the application merges. Hybrid retrieval becomes application-side instead of search-engine-side.

**Trade-off.** Operational complexity (two stores instead of one) for performance gains.

### 7.4 The migration cost rules of thumb

- 1M vectors: a sprint.
- 10M vectors: a month.
- 100M vectors: a quarter.
- 1B vectors: multiple quarters; project-grade.

Plan accordingly. The "we'll just migrate later" is usually under-budgeted.

---

## 8. Worked Meridian Health example

### 8.1 The decision

Meridian's Care Coordinator answered the six questions:

1. **Vector count.** ~50K initially, projected 250K at 5x growth headroom. Comfortably in Pattern A territory.
2. **Query throughput.** Peak ~30 QPS at GA scale. Easily handled by pgvector.
3. **Hybrid retrieval needed.** Yes (BM25 + vector + reranker per the RAG decision guide). pgvector with application-side merging is fine at this scale.
4. **Multi-tenancy.** 240 tenants on shared-model-with-isolated-data per the isolation-models doc. Per-tenant Postgres partitioning fits.
5. **Regulatory.** HIPAA. pgvector keeps the data in Meridian's AWS account; no third-party vendor BAA needed.
6. **Team operational capacity.** ai-platform-eng (6 people) operates the existing Aurora Postgres footprint. Adding pgvector is incremental, not a new operational discipline.

→ **Pattern A: pgvector in Aurora Postgres.** Per-tenant partitioning for isolation.

### 8.2 The deployment topology

- Primary cluster: Aurora Postgres in us-east-1 (single-AZ writer, multi-AZ readers).
- Per-tenant partitioned table `chunks` with HNSW index per partition.
- 240+ partitions (one per hospital tenant + one for global content).
- Embeddings live in the same database alongside chunk metadata and document metadata.

The premium customer's dedicated infrastructure (per the isolation-models example) uses a separate Aurora cluster with the same pgvector design, single-tenant.

### 8.3 The capacity planning

- 50K chunks × 1536 dimensions × 4 bytes + index overhead = ~450MB total storage.
- 240 partitions = ~2MB per partition on average (varies by tenant size).
- 30 QPS sustained; Aurora's read-replica capacity handles it with headroom.
- Re-embedding events: 50K chunks × 90ms per embedding × 1 worker = ~75 minutes. Not a project; a maintenance window.

### 8.4 The hybrid retrieval shape

The retrieval-client wrapper invokes:
1. BM25 query against Postgres FTS (in the same database, no separate index needed).
2. Vector query against pgvector (HNSW).
3. Application-side merging (RRF) of the two result sets.
4. Reranking via Cohere (network call to external service).

The application-side hybrid is operationally simpler than running a separate hybrid search engine; at this scale, the latency cost is acceptable.

### 8.5 The migration consideration

If the Care Coordinator's volume grows past where pgvector is comfortable (~10x current scale projected to ~5M vectors and 300 QPS), the team has a migration plan:
- Candidate replacement: Pinecone or Weaviate. Both have HIPAA BAA available.
- Migration timeline: ~6 weeks for the volume.
- Decision criteria: pgvector p95 query latency exceeds 100ms sustained, OR pgvector capacity utilization is sustained > 70%.

The criteria are documented; the team is not committing to the migration; the trigger is monitored.

### 8.6 The platform discipline

- `meridian.retrieval` (the wrapper) is the only path to pgvector; lint enforces.
- HNSW index parameters tuned per the Aurora performance team's guidance.
- Per-partition metrics and per-partition health monitoring.
- Quarterly review of pgvector performance against the migration criteria.

---

## 9. Anti-patterns

### 9.1 "Vendor-by-default because everyone uses Pinecone"

The team picks the popular vendor without sizing the workload. Cost is higher than needed; vendor lock-in is real; the team has not considered whether their volume justifies the vendor's pricing.

**Corrective.** Run the six questions. For small-medium scale, pgvector is usually the right answer.

### 9.2 "pgvector at scale where it cannot keep up"

The team starts with pgvector for one feature; the workload grows; query latency degrades; the team scales up Postgres until it is expensive and still slow.

**Corrective.** Monitor against the migration criteria; commit to migrating when criteria are met; do not let pgvector strain forever.

### 9.3 "Self-hosted before proving the need"

A small team picks self-hosted Weaviate / Milvus for "future scale." Operational overhead consumes the team's capacity; managed alternatives would have served the actual scale fine.

**Corrective.** Self-hosted is for teams with strong operational capability AND demonstrated scale need.

### 9.4 "Multi-store sprawl"

Different features use different vector stores ad-hoc. The team operates four different vector stores; expertise is diluted; migrations are per-feature instead of platform-wide.

**Corrective.** Default to one store platform-wide; deviate only with explicit justification. Coordinate deviations to avoid sprawl.

### 9.5 "No capacity planning"

The team deploys vector storage with default settings and discovers capacity limits in production.

**Corrective.** Capacity planning per section 4 before launch; monitoring against capacity in production; documented re-evaluation triggers.

### 9.6 "Vector store as application database"

The team puts non-vector data in the vector store (user records, configurations, transactions) because "everything is together." The vector store is mis-used; performance degrades.

**Corrective.** Vector store for vector retrieval. Relational data in relational stores. Cross-references via foreign keys or application-side joins.

### 9.7 "Re-embedding without planning"

The team changes embedding models without planning the re-embedding event. Production retrieval breaks during the migration; users see degraded behavior.

**Corrective.** Re-embedding is a planned operation. New vectors go to a separate index/namespace; the application reads from the new only after re-embedding completes.

### 9.8 "BAA / regulatory check after vendor selection"

The team picks Pinecone, builds the integration, and then discovers their HIPAA BAA tier requires a more expensive plan than the team budgeted for. Or the data-residency requirement was missed entirely.

**Corrective.** Regulatory check is question 5 in the decision; it filters vendor options before integration starts.

---

## 10. Findings (sprint-assignable)

### ARCH-VEC-001 — Severity: High
**Finding.** Vector store was selected without sizing the workload; pattern may be misaligned with actual scale and operational capacity.
**Recommendation.** Answer the six decision questions (section 3); document the rationale; revisit if mismatch is evident.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-VEC-002 — Severity: High
**Finding.** No capacity planning; vector counts, throughput, storage projections are not in place.
**Recommendation.** Capacity planning per section 4; alert thresholds against capacity; documented re-evaluation criteria.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### ARCH-VEC-003 — Severity: High
**Finding.** Hybrid retrieval is required but the chosen store does not support it natively; application-side merging is implemented inconsistently.
**Recommendation.** Standardize the hybrid pattern (application-side merging vs search-engine-native); document.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-VEC-004 — Severity: High
**Finding.** Multi-tenancy posture in the vector store is not implemented at the storage layer; relies on filter-only isolation.
**Recommendation.** Per-tenant namespace or per-tenant index per [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md).
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-VEC-005 — Severity: High
**Finding.** Regulatory / BAA coverage of the vendor is not verified for the workload's data sensitivity.
**Recommendation.** Verify vendor BAA / regulatory posture; document; switch vendors if mismatch.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+2.

### ARCH-VEC-006 — Severity: Medium
**Finding.** Multiple vector stores are deployed across the platform without coordination; expertise is diluted.
**Recommendation.** Default platform vector store; deviation requires explicit justification.
**Owner.** ai-platform-eng team lead, sprint N+3.

### ARCH-VEC-007 — Severity: Medium
**Finding.** Vector index parameters are at defaults; query performance is suboptimal for the workload.
**Recommendation.** Tune HNSW / IVF parameters against the workload; re-tune when corpus characteristics change.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-VEC-008 — Severity: Medium
**Finding.** Re-embedding events are unplanned; embedding-model changes cause production disruption.
**Recommendation.** Re-embedding is a planned operation; new vectors go to separate namespace during transition; cut over after completion.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-VEC-009 — Severity: Medium
**Finding.** Migration criteria from the current pattern to an alternative are not documented; growth past the comfortable limit is not anticipated.
**Recommendation.** Document migration triggers per section 8.5; monitor against them.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ARCH-VEC-010 — Severity: Medium
**Finding.** Vector store backups are not aligned with the regulatory retention requirement.
**Recommendation.** Confirm backup retention; align with regulation; verify DR procedures.
**Owner.** ai-platform-eng + sre + compliance, sprint N+3.

### ARCH-VEC-011 — Severity: Medium
**Finding.** Vector store monitoring is limited to availability; query latency, capacity utilization, error rates per index are not surfaced.
**Recommendation.** Vector-specific SLIs; per-index dashboards; alerting.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ARCH-VEC-012 — Severity: Medium
**Finding.** Vector store is used as a general application database (non-vector data stored there); performance is affected.
**Recommendation.** Vector store for vector retrieval only; relational data in relational stores; cross-reference via IDs.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-VEC-013 — Severity: Medium
**Finding.** Read replicas are not used to scale query throughput; all queries hit the writer.
**Recommendation.** Read replicas for query traffic; writer for ingestion only.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-VEC-014 — Severity: Medium
**Finding.** Vector store version (Postgres + pgvector version, or vendor version) drift is not tracked.
**Recommendation.** Version registry; upgrade cadence documented; eval-gated upgrade pattern.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-VEC-015 — Severity: Low
**Finding.** Cost of the vector store is not separately tracked; finops cannot attribute spend.
**Recommendation.** Per-feature cost attribution per the FinOps pattern.
**Owner.** ai-platform-eng + finops, sprint N+4.

### ARCH-VEC-016 — Severity: Low
**Finding.** Application-side hybrid merging strategy is not documented; new engineers learning the system have to read the code.
**Recommendation.** Document the hybrid merging strategy and rationale.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-VEC-017 — Severity: Low
**Finding.** Vector store DR runbook is undocumented; failover procedures are ad-hoc.
**Recommendation.** Document DR runbook; rehearse quarterly.
**Owner.** ai-platform-eng + sre, sprint N+5.

### ARCH-VEC-018 — Severity: Low
**Finding.** Capacity-planning projections are not updated quarterly; growth trends may exceed plan without notice.
**Recommendation.** Quarterly capacity review; re-project against current trends.
**Owner.** ai-platform-eng team lead, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team selecting a vector store:

- [ ] **Sprint 0 — answer the questions.** Six decision questions in section 3. Document the answers.
- [ ] **Sprint 0 — pick the pattern.** Match the answers to a pattern (section 2).
- [ ] **Sprint 0 — pick the vendor or implementation within the pattern.** Verify regulatory coverage.
- [ ] **Sprint 1 — capacity planning.** Project counts, throughput, storage. Document.
- [ ] **Sprint 1 — initial deployment.** Stand up the store. Configure for the chosen multi-tenancy posture.
- [ ] **Sprint 2 — retrieval client integration.** Per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md); the wrapper is the only path.
- [ ] **Sprint 2 — monitoring.** Vector-specific SLIs; per-index dashboards.
- [ ] **Sprint 3 — backups and DR.** Backup retention, recovery procedures, DR runbook.
- [ ] **Sprint 3 — migration criteria.** Document the triggers that would justify migrating to a different pattern.
- [ ] **Sprint 4 — production hardening.** Tune index parameters; verify capacity headroom; rehearse DR.
- [ ] **Ongoing — quarterly review.** Capacity, performance, cost, migration criteria.

A team that completes this sequence has a deliberate vector-store choice that fits the workload and the operational capacity. A team that defaults to whatever is popular pays in either cost (over-provisioned vendor) or operational pain (under-provisioned self-hosted).

---

## 12. References

- pgvector documentation; Postgres declarative partitioning docs.
- Pinecone, Weaviate, Qdrant, Milvus, Chroma documentation.
- OpenSearch, Elasticsearch, Vespa vector search documentation.
- HNSW, IVF, DiskANN, ScaNN — vector index algorithms.
- This repo: [data-architecture-for-ai/embedding-strategy.md](./) (coming) — the embedding-model decision that pairs with this.
- This repo: [data-architecture-for-ai/data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — the corpus-as-contract pattern.
- This repo: [data-architecture-for-ai/lineage-and-provenance.md](./lineage-and-provenance.md) — the lineage pattern that vector storage supports.
- This repo: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — the implementation depth on multi-tenant patterns.
- This repo: [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — the wrapper that sits in front of the vector store.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — ADR-CARE-004 commits to pgvector for the worked example.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/rag-engineering) — the engineering practice for the pipelines that feed the vector store.
