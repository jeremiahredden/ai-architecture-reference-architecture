# Data Architecture for AI

## What this folder is

The architectural decisions about how data flows into and out of AI systems. The material here is what I put in front of a data platform team when the question is: *we have a data warehouse, an operational database, a document store, a CMS, and now an AI feature needs to retrieve across all of them — what is the data architecture that makes this safe, fresh, and inexpensive?*

## The organizing principle

AI workloads are a new shape on top of an existing data estate. They are read-heavy, latency-sensitive, schema-permissive, freshness-sensitive in some places and tolerant in others, and they introduce new storage primitives (vector indexes, embedding caches, graph stores) that did not exist in the canonical data platform pattern. They also introduce a new failure mode that data engineering teams are not used to: *silent quality degradation*. A retrieval source that drifts in schema does not throw an error — it returns documents the model uses but cannot use well, and the symptom shows up days later as a quality regression in evals.

So the patterns here treat AI data architecture as *first-class data architecture* (data contracts, lineage, freshness SLOs, ownership) rather than as an appendage of the AI feature. The vector store is a database with all the usual database-design questions; the retrieval corpus is a published dataset with all the usual dataset-versioning questions; the embedding model is a *piece of infrastructure* whose version change can silently invalidate every embedding downstream of it.

## Planned documents

- **vector-store-architecture.md** *(coming)* — Choosing and sizing a vector store: pgvector (extends Postgres, simple), Pinecone / Weaviate / Qdrant (purpose-built), OpenSearch / Elasticsearch (hybrid with lexical). Capacity planning, multi-tenancy patterns (per-tenant namespace, per-tenant index, shared with metadata filter), the cost / latency trade-offs, and the migration path between vector store choices.
- **embedding-strategy.md** *(coming)* — Model selection, dimension choice (cost / quality trade-off), normalization, batching for ingestion throughput, the rebuild-vs-incremental update pattern, embedding versioning as a first-class artifact, and the embedding-model-deprecation playbook. Includes the worked example of changing embedding models without invalidating the entire production retrieval system in one weekend.
- **knowledge-graph-augmentation.md** *(coming)* — When a knowledge graph belongs in the architecture and when it does not. The graph-as-retrieval-source pattern (retrieve subgraph, format into context), graph-as-tool pattern (model calls a graph query tool), graph extraction from unstructured text, and the failure mode where the graph itself becomes an expensive maintenance burden disconnected from the actual data.
- **hybrid-store-architecture.md** *(coming)* — The increasingly-common pattern of running BM25, vector, and structured filters in the same query path. Architecture variants (single-engine OpenSearch / Vespa, dual-engine with merging, RRF rank fusion), the operational trade-offs, and the data-modeling decisions that determine whether hybrid lifts quality or just adds cost.
- **feature-stores-for-ai.md** *(coming)* — When a feature store (Tecton, Feast, internal) is the right pattern for AI workloads — typically for personalization signals, user state, and structured context that supplements retrieval. Feature store vs context cache vs profile DB, and the integration shape with the agent or RAG pipeline.
- **data-contracts-for-retrieval.md** *(coming)* — Treating retrieval corpora as published datasets with data contracts: schema, freshness, deduplication, content-type guarantees, owner team, change protocol. The pattern that prevents an upstream CMS change from silently degrading downstream AI quality. Aligned with the engineering sibling's `data-engineering-for-ai/` folder.
- **lineage-and-provenance.md** *(coming)* — Tracking provenance from source document → chunk → embedding → retrieval → answer, so a user-facing answer can be traced back to the exact source text. Critical for regulated workloads (clinical, legal, financial), and useful everywhere as the debugging tool of last resort. Integration with observability sibling content.
- **freshness-architecture.md** *(coming)* — The freshness-vs-staleness architectural choices: real-time vs near-real-time vs nightly vs on-demand ingestion. Cache invalidation patterns. When stale is acceptable, when stale is catastrophic, and the per-source freshness SLO pattern.

## How to use this section

**If you are designing a RAG system from scratch**, read `vector-store-architecture.md` and `embedding-strategy.md` first. Those two decisions constrain everything else and are the most expensive to change after launch.

**If you have a RAG system in production with quality regressions of unknown cause**, `data-contracts-for-retrieval.md` and `lineage-and-provenance.md` are the diagnostic pair. The cause is usually an upstream data change that the AI team did not know about.

**If you are deciding whether to add a knowledge graph**, `knowledge-graph-augmentation.md` is explicit about the cases where it pays back and the (more common) cases where it adds operational cost without adding quality.

## What this section is not

- **A vector-database benchmark.** Vector DB performance varies more by workload shape than by product. The patterns here help you evaluate against your workload; they do not declare a winner.
- **A data warehouse design guide.** General data warehousing (Snowflake / BigQuery / Redshift modeling, dbt patterns) is a mature discipline with its own canon. This folder is only about the AI-specific overlays on top.
