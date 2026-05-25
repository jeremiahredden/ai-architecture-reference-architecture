# Lineage and Provenance

> **Audience.** Architects designing the data side of a regulated-AI workload. Tech leads who have been asked "where did this specific claim in the answer come from?" and could not answer. **Scope.** The *architectural* pattern for end-to-end lineage from source document → chunk → embedding → retrieval → context → answer. Not the engineering practice of the ingestion pipeline (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/`). Not the contract pattern (see [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Regulated AI workloads — clinical, financial, legal — require that every user-facing claim be traceable back to its source. A clinical decision-support system that says "the recommended dose is 5mg twice daily" must let the clinician click through to the AHA 2024 Heart Failure Guideline section 3.2 that contains that recommendation. Without that traceability, the AI's output is unsupportable; the regulatory posture is incomplete; the clinician cannot trust the system in high-stakes decisions.

Lineage and provenance is the architectural pattern that produces this traceability. It is not "log the source documents somewhere"; it is a *first-class flow* through the system where every artifact carries the identifier of its predecessor and the chain is queryable end-to-end.

The Care Coordinator's `ARCH-CARE-015` finding (citation / source attribution not tracked) is the canonical instance. The RAG decision guide's `ARCH-RAG-015` is the analog. This document is the architectural depth on the pattern.

This document is opinionated about three things:

1. **Lineage is first-class, not retrofitted.** The artifacts that flow through the pipeline (source documents, chunks, embeddings, retrievals, contexts, answers) each carry provenance metadata. Adding lineage to an existing pipeline that did not design for it is much harder than designing for it.
2. **The lineage is queryable.** Given an answer, the team can produce the source documents. Given a source document, the team can find every answer that cited it. Given an embedding, the team can identify the chunk and document. The queries are routine, not forensic.
3. **The lineage supports both user-facing citations and after-the-fact audit.** The same metadata serves the in-product "show citation" feature and the regulatory-audit "prove this claim is supported" workflow.

Structure: (2) what lineage is and is not; (3) the lineage chain; (4) provenance metadata at each stage; (5) the lineage store; (6) the citation-attribution surface; (7) integration with retrieval scope enforcement; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. What lineage is and is not

### 2.1 What it is

**Lineage** is the chain of artifacts connecting a user-facing output to its sources. Each artifact in the chain carries identifiers that let you traverse to the predecessors.

**Provenance** is the metadata about an artifact's origin — when it was created, by what process, from what predecessor, by what version of the producing component. Provenance is recorded alongside the artifact and travels with it.

### 2.2 What it is not

- **Not just a citation feature.** Citations are the user-facing surface of lineage, but lineage is the underlying chain that supports more than citations (audit, debugging, contamination detection).
- **Not just for the retrieval-augmented answer.** Lineage applies to every artifact: training data, fine-tune data, evals, prompts, retrieved chunks, answers.
- **Not "blame tracking" for AI mistakes.** Lineage is for traceability; understanding which source contributed to a claim. Blame is a separate concern.

### 2.3 The four use cases

Lineage serves four distinct use cases. Each shapes the metadata captured.

**Citation.** "The answer's claim X is supported by source S." User-facing. Citations point users to the documents that grounded the answer.

**Audit.** "On date D, the system produced answer A for interaction I; the answer was supported by sources S1, S2, S3 retrieved from corpus C version V." Regulatory. Auditors verify that claims are supported by retrievable sources.

**Debugging.** "The answer was wrong; which retrieved chunk contributed to the wrong claim?" Operational. Engineers trace incorrect outputs back to specific sources.

**Contamination detection.** "Did this eval case appear in our training data?" Engineering discipline. Detect cross-contamination of training / eval / retrieval data.

Each use case has different latency, retention, and access-control requirements. The architectural pattern serves all four with one unified lineage system.

---

## 3. The lineage chain

The pipeline produces a chain of artifacts. Each link captures the predecessor.

### 3.1 The standard chain (RAG-shaped workload)

```
Source document  →  Chunk  →  Embedding  →  Retrieval  →  Context  →  Answer

(at each → arrow, the downstream artifact records the upstream artifact's identifier)
```

Each arrow is a process step. Each downstream artifact records the upstream(s) it was derived from.

### 3.2 Chain detail

**Source document.** A document in the corpus. Identifier: doc_id (corpus-specific). Provenance: source-corpus version, ingestion date, upstream origin.

**Chunk.** A piece of a source document. Identifier: chunk_id (often `{doc_id}:{chunk_offset}`). Provenance: parent doc_id, chunk strategy, chunk version (which chunking algorithm version produced it), creation date.

**Embedding.** A vector representation of a chunk. Identifier: embedding_id. Provenance: parent chunk_id, embedding model version, dimension count, normalization scheme.

**Retrieval.** A retrieval call's result set. Identifier: retrieval_id (often the trace's retrieval span ID). Provenance: query, query rewriter version, retriever configuration, top-K, reranker version, returned doc/chunk IDs with scores.

**Context.** The portion of retrieved content used in the prompt. Identifier: context_id. Provenance: parent retrieval_id(s), context-assembly version, included chunk IDs, truncation decisions.

**Answer.** The user-facing output. Identifier: answer_id (often the trace's interaction ID). Provenance: parent context_id, model version, prompt version, generation parameters.

### 3.3 The chain for agent workloads

Agent workloads have a more complex chain because the agent's tool calls produce intermediate context updates:

```
Source documents  →  Chunks  →  Embeddings
                                    │
                                    ▼
                              Retrieval (turn 1)  →  Context (turn 1)
                                                          │
                                                          ▼
                                                    Agent decision
                                                          │
                                                          ▼
                                                    Tool call → tool result
                                                          │
                                                          ▼
                                                    Retrieval (turn 2)  →  Context (turn 2)
                                                                                │
                                                                                ▼
                                                                          Final answer
```

Each retrieval and each context update is its own lineage link. The final answer's lineage traces back through every turn's retrievals.

### 3.4 The chain for non-RAG workloads

Workloads that do not use retrieval still have lineage, just shorter:

```
Prompt + variables  →  Context  →  Answer
```

The answer's lineage records the prompt version and the variable values that produced it. No retrieval link; the chain is shorter.

---

## 4. Provenance metadata at each stage

The metadata required at each stage. Each artifact's record is small but structured.

### 4.1 Source document

```yaml
doc_id: clinical-guideline:aha-hf-2024:section-3.2
corpus: clinical-guidelines
corpus_version: 2026-Q2-1
ingested_at: 2026-04-15T08:30:00Z
ingested_from: aha-publisher-feed
ingest_pipeline_version: 2.3.1
content_hash: sha256:9c2a1f8b...
metadata:
  publisher: American Heart Association
  publication_date: 2024-09-15
  section: 3.2
  topic_tags: [heart-failure, discharge, post-acute-care]
```

### 4.2 Chunk

```yaml
chunk_id: clinical-guideline:aha-hf-2024:section-3.2:chunk-0042
doc_id: clinical-guideline:aha-hf-2024:section-3.2
chunk_index: 42
content_hash: sha256:7fa3b1...
created_at: 2026-04-15T08:31:23Z
chunk_strategy: document-structure-aware-v2
chunk_strategy_version: 2.1.0
chunk_position:
  start_offset: 12450
  end_offset: 13180
content_length_tokens: 287
```

### 4.3 Embedding

```yaml
embedding_id: emb-clinical-guideline:aha-hf-2024:section-3.2:chunk-0042-v3
chunk_id: clinical-guideline:aha-hf-2024:section-3.2:chunk-0042
embedding_model: text-embedding-3-large
embedding_model_version: 2024-01-25
dimensions: 1536
normalization: l2
created_at: 2026-04-15T08:32:10Z
```

### 4.4 Retrieval

```yaml
retrieval_id: ret-2026-05-25-a7b8-1
trace_id: t-2026-05-25-3a4b
interaction_id: interaction-2026-05-25-a7b8
query: "What is the post-discharge follow-up protocol for a CHF patient on the new pathway?"
query_rewriter_version: 1.1.0  # null if not rewritten
retriever_type: hybrid
retriever_config:
  bm25_weight: 0.4
  vector_weight: 0.6
  rerank: cohere-rerank-3.5@2025-09-01
returned:
  - chunk_id: clinical-guideline:aha-hf-2024:section-3.2:chunk-0042
    rank: 1
    bm25_score: 8.7
    vector_score: 0.91
    rerank_score: 0.96
  - chunk_id: tenant-protocol:mercy-cleveland:hf-22:chunk-0007
    rank: 2
    bm25_score: 7.2
    vector_score: 0.87
    rerank_score: 0.92
  # ...
```

### 4.5 Context

```yaml
context_id: ctx-2026-05-25-a7b8-1
trace_id: t-2026-05-25-3a4b
parent_retrieval_ids: [ret-2026-05-25-a7b8-1]
assembly_strategy: structured-with-citations-v3
included_chunks:
  - chunk_id: clinical-guideline:aha-hf-2024:section-3.2:chunk-0042
    position: 1
    truncated: false
  - chunk_id: tenant-protocol:mercy-cleveland:hf-22:chunk-0007
    position: 2
    truncated: false
total_context_tokens: 1240
```

### 4.6 Answer

```yaml
answer_id: ans-2026-05-25-a7b8
trace_id: t-2026-05-25-3a4b
interaction_id: interaction-2026-05-25-a7b8
parent_context_id: ctx-2026-05-25-a7b8-1
generated_at: 2026-05-25T14:32:18Z
model: claude-opus-4-7@2026-04-12
prompt_version: care_coordinator_clinical_knowledge@1.8.0
generation_parameters:
  temperature: 0.2
  max_tokens: 2000
content_hash: sha256:c8d9f2...
cited_chunks:
  - chunk_id: clinical-guideline:aha-hf-2024:section-3.2:chunk-0042
    claim_position: ["sentences 2-3"]
  - chunk_id: tenant-protocol:mercy-cleveland:hf-22:chunk-0007
    claim_position: ["sentence 4"]
```

### 4.7 The structure consistency

Every record has the same shape: identifier, parent identifier(s), creation metadata, content-fingerprint hash, processing-version. Tooling can traverse the chain uniformly.

---

## 5. The lineage store

Lineage records live in a queryable store. The store is the lineage system's persistence.

### 5.1 What the store provides

- **Insertion.** Pipeline stages write lineage records as artifacts are produced.
- **Lookup.** Given an artifact ID, retrieve its provenance.
- **Traversal.** Given an artifact, walk up or down the chain.
- **Query.** "Find all answers that cited chunk X." "Find all chunks derived from document Y." "Find all interactions on date D using prompt version Z."
- **Retention.** Lineage records are retained per the regulatory requirement (Meridian: 7 years for clinical lineage).

### 5.2 The store choice

Three options:

**Relational database with lineage tables.** Postgres tables for each artifact type with foreign-key relationships. Strong query support; well-understood operations.

**Document database.** MongoDB / DynamoDB with denormalized lineage. Faster individual lookups; weaker complex queries.

**Graph database.** Neo4j / DGraph with explicit lineage edges. Most expressive for graph traversal queries; operational overhead.

For Meridian: relational (Postgres). Most lineage queries are tree-shaped (walk up the chain) and SQL handles them well. The relational option keeps the data-platform footprint small.

### 5.3 The write pattern

Lineage writes are eventually consistent. The pipeline stage produces the artifact and writes the lineage record asynchronously (often to a queue → batch insert into the store). The pipeline does not block on lineage writes.

The exception: for high-stakes lineage (clinical citations), the lineage write is synchronous — the answer is not surfaced until the lineage record is persisted. This ensures that citations in the user-facing answer always resolve to a persisted record.

### 5.4 The hash chain

Each lineage record contains a content hash of the artifact. The hash chains let the team detect tampering:

- The answer's record includes the content hash of the context.
- The context's record includes the content hashes of the included chunks.
- The chunks' records include the content hashes of their source documents.

If a source document is modified after retrieval, the answer's lineage chain reveals the discrepancy. This is the foundation of audit-grade traceability.

---

## 6. The citation-attribution surface

The user-facing citations are the visible product of lineage. The architecture:

### 6.1 The citation contract

A citation is:
- A pointer to a specific chunk (chunk_id).
- A description of what claim in the answer it supports (claim_position).
- A surface format suitable for display (text link, structured popover, hyperlinked footnote).

### 6.2 The citation production

Citations are produced at answer-generation time:
- The model is prompted to cite sources for each claim (the prompt includes instructions like "for every claim, cite the source chunk by ID").
- The answer parser extracts citations from the model's output.
- The citation IDs are validated against the context's chunk IDs (a citation that does not reference a context chunk is a hallucinated citation).
- Validated citations are persisted to the answer's lineage record.

### 6.3 The citation validation

Validation is a guardrail:
- Citation must reference a chunk in the context.
- Citation's claim_position must point to a real position in the answer.
- For high-stakes workloads (Meridian clinical), citation accuracy is itself an eval criterion.

Failed validations are not silently dropped; they are surfaced to the team and trigger eval-suite additions.

### 6.4 The user-facing rendering

The application's UI uses the citation metadata to render:
- Inline footnote markers ([1], [2], ...).
- Hover popovers showing the cited chunk's content.
- Click-throughs to the source document.
- "View all sources" panels.

The rendering is application-side; the lineage system provides the data.

### 6.5 The citation cache invalidation

When a source document is updated (a new corpus version landed), citations in previously-cached answers may point to outdated chunks. The application's caching strategy:

- Cache invalidation on corpus version change for cached answers.
- "Source updated since this answer was generated" warning if a stale citation is surfaced.
- Refresh policy for stale answers.

---

## 7. Integration with retrieval scope enforcement

Lineage and scope enforcement (per [retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md)) are paired patterns.

### 7.1 Scope enforcement produces the lineage tokens

The retrieval wrapper, when it enforces scope, records the scope dimensions in the retrieval lineage record. The lineage record shows not only which chunks were returned but also the scope context that authorized their return.

### 7.2 Lineage validates scope after-the-fact

When investigating a potential cross-scope incident, the lineage records are the evidence:
- "On date D, retrieval R for tenant T returned chunks with tenant attribution {T1, T2}" — the discrepancy is visible.
- The lineage chain shows the answer's chunks; the chunks' source documents; the documents' tenants.
- A scope violation appears as a chain whose terminal documents belong to a different tenant than the requesting context.

The lineage is the forensic trail for scope-enforcement integrity.

### 7.3 Cross-tenant lineage queries

For the rare legitimate cross-tenant operations (admin analytics), the lineage system supports cross-tenant queries with separate audit:
- The query is authorized.
- The query's execution is itself logged in the audit trail.
- The query's results are scoped to the operation's purpose.

---

## 8. Worked Meridian Health example

### 8.1 The lineage architecture

Meridian's lineage store is a Postgres database (separate from the operational stores) with tables for: documents, chunks, embeddings, retrievals, contexts, answers. Each table has a primary identifier, foreign keys to predecessors, creation metadata, and content hash.

Write pattern: pipelines write via a lineage-writer service that batches and ingests. Reads via a lineage-query API.

### 8.2 The Care Coordinator citation flow

When the Care Coordinator answers a clinical question:

1. Retrieval-client wrapper executes the retrieval; lineage record `ret-X` is persisted with the returned chunk IDs.
2. The context assembler builds the prompt context from retrieved chunks; lineage record `ctx-X` is persisted with the included chunk IDs.
3. The LLM generates the answer with citation instructions in the prompt. Answer text includes citation markers.
4. The answer parser extracts citations. Each citation is validated (the cited chunk_id must be in the context's chunk list).
5. The answer's lineage record `ans-X` is persisted with the validated citations.
6. The application's chat panel renders the answer with hyperlinked citations. Clicking a citation opens the source document at the cited chunk's position.

### 8.3 An audit example

A regulatory auditor asks: "On 2026-05-15, the Care Coordinator gave clinician John Doe an answer about CHF discharge protocols. Show the sources."

1. The auditor accesses the audit interface; provides the interaction ID.
2. The interface queries the lineage system; retrieves the answer record.
3. The answer record's `cited_chunks` field lists the chunks.
4. For each cited chunk, the interface walks up the chain: chunk → source document → corpus version.
5. The interface displays: the answer; the cited chunks' full content; the source documents (with publication dates and versions); the corpus version that was current at the time.

The auditor verifies that each claim in the answer is supported by the cited source content. The audit takes minutes, not days.

### 8.4 A debugging example

The team is debugging a quality regression: a clinician reported that an answer was wrong on 2026-05-22. The investigation:

1. Pull the interaction's trace.
2. From the trace, get the answer's lineage record.
3. Walk up the chain to the cited chunks and source documents.
4. Verify the source documents are correct (yes, the AHA 2024 guideline section is current).
5. Verify the retrieved chunks contain the expected content (yes).
6. Verify the context included the right chunks (yes).
7. Verify the model's output reflects the chunks (no — the model misinterpreted a clinical recommendation).
8. Diagnosis: the supervisor prompt's clinical-reasoning instructions led to a misinterpretation. Fix: prompt-engineer the supervisor prompt.

The lineage made the investigation possible. Without it, the team would have rerun the question and seen variable output without knowing what went wrong.

### 8.5 A contamination detection

The team's quarterly contamination audit:

1. Sample 30 cases from the clinical eval golden set.
2. For each case, query the lineage system: "did any chunk derived from a document related to this case appear in training data or in any system that the model has seen?"
3. The query walks: case → expected source document → chunks derived from that document → any retrieval that returned those chunks → any context that included them → any training-data ingestion that involved them.
4. Findings: 0 contamination in this quarter.

Without lineage, the contamination audit is "we don't know; we hope."

### 8.6 The lineage store size

For Meridian's volume (~3K Care Coordinator interactions/day + retrievals, embeddings, chunks, documents):

- Documents: ~50K (clinical guidelines, protocols, drug interactions). Growth: slow.
- Chunks: ~250K. Growth: with re-chunking strategies, occasional bumps.
- Embeddings: ~250K (matches chunks). Growth: re-embedding events.
- Retrievals: ~20K/day (multiple per interaction). Growth: with feature usage.
- Contexts: ~3K/day (one per interaction). 
- Answers: ~3K/day (one per interaction).

Aggregate over 7-year retention: ~30M retrieval records, ~9M answer records. Storage is meaningful but bounded; Postgres handles it cleanly.

### 8.7 The platform discipline

- Every pipeline stage writes lineage records.
- Lineage queries are first-class APIs.
- Citation validation is in the answer-generation path.
- Audit interface is built and rehearsed.
- Quarterly contamination audit.
- Quarterly lineage-store health review.

---

## 9. Anti-patterns

### 9.1 "Citations are model-generated, not validated"

The model is instructed to cite sources; the cites are inserted in the output text; the application renders them. Some citations are hallucinated (cite a chunk that was never retrieved). Users click; the link 404s.

**Corrective.** Validate citations against the retrieval lineage; reject hallucinated citations; surface validation results in eval.

### 9.2 "Lineage is logs, not structured"

Lineage information is in application logs as informal text. Querying the chain is impossible.

**Corrective.** Lineage as structured records in a queryable store. The schema is the contract.

### 9.3 "Lineage is per-feature, not platform"

Each feature implements its own lineage scheme. The chat feature's chunk IDs do not align with the analytics feature's chunk IDs even when they refer to the same chunks.

**Corrective.** Platform-wide lineage schema. The same source document, chunk, embedding produces consistent IDs across all consumers.

### 9.4 "Lineage retention is the database default"

Lineage records are retained 30 days. The team cannot satisfy 7-year regulatory audits.

**Corrective.** Retention aligned with regulation. Tiered (hot in primary store; cold in compliance-grade archive).

### 9.5 "No content hash in lineage"

Lineage records reference IDs but not content hashes. If a source document is modified, the lineage gives no signal of tampering.

**Corrective.** Hash every artifact; the lineage chain is hash-linked; modifications are detectable.

### 9.6 "Lineage writes are blocking and slow"

Pipeline stages block on synchronous lineage writes; throughput is bottlenecked on the lineage store.

**Corrective.** Asynchronous writes; batch ingestion; synchronous only for high-stakes paths.

### 9.7 "No contamination audit"

The team has lineage but never queries it for contamination. Eval-data leakage into training / retrieval is undetected.

**Corrective.** Quarterly contamination scans per [eval-engineering-playbook.md](../../ai-engineering-reference-architecture/eval-engineering/eval-engineering-playbook.md) section 9.

### 9.8 "Lineage queries require engineering"

The audit interface requires engineering to run lineage queries. Auditors and compliance teams must file tickets.

**Corrective.** Self-service audit interface for authorized non-engineering users.

---

## 10. Findings (sprint-assignable)

### ARCH-LIN-001 — Severity: Critical
**Finding.** Lineage from source document → answer is not tracked; user-facing claims cannot be traced to sources.
**Recommendation.** Build the lineage chain per section 3; pipeline stages write lineage records; queryable store.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-LIN-002 — Severity: Critical
**Finding.** Citations in user-facing answers are model-generated and not validated; hallucinated citations have been observed.
**Recommendation.** Citation validation per section 6.3; reject citations not present in context; surface validation rate in eval.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-LIN-003 — Severity: High
**Finding.** Lineage retention is the database default (30-90 days); regulatory requirement is 7+ years.
**Recommendation.** Tiered retention per section 5.1; hot in primary store, cold in compliance-grade archive.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+2.

### ARCH-LIN-004 — Severity: High
**Finding.** Lineage schema is per-feature; cross-feature lineage queries are not possible.
**Recommendation.** Platform-wide schema per section 4; consistent IDs across features.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-LIN-005 — Severity: High
**Finding.** Content hashes are not part of lineage; source-document tampering is undetectable.
**Recommendation.** Hash every artifact per section 5.4; chain hashes for tamper detection.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-LIN-006 — Severity: High
**Finding.** Audit interface does not exist; auditors require engineering to run lineage queries.
**Recommendation.** Self-service audit interface for authorized non-engineering users.
**Owner.** ai-platform-eng + compliance, sprint N+3.

### ARCH-LIN-007 — Severity: High
**Finding.** Quarterly contamination audit is not scheduled; eval contamination is undetected.
**Recommendation.** Schedule per [eval-engineering-playbook.md](../../ai-engineering-reference-architecture/eval-engineering/eval-engineering-playbook.md) section 9; use lineage queries.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-LIN-008 — Severity: Medium
**Finding.** Lineage writes are synchronous and blocking; pipeline throughput is bottlenecked.
**Recommendation.** Asynchronous writes per section 5.3; synchronous only for high-stakes paths.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-LIN-009 — Severity: Medium
**Finding.** Citation rendering in the UI does not surface staleness when source documents are updated after answer generation.
**Recommendation.** Citation cache invalidation per section 6.5; "source updated" warnings.
**Owner.** ai-platform-eng + product, sprint N+3.

### ARCH-LIN-010 — Severity: Medium
**Finding.** Agent-loop intermediate retrievals are not captured in lineage; only the final answer's lineage is recorded.
**Recommendation.** Per-turn retrieval lineage per section 3.3; agent chains have multi-link lineage.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-LIN-011 — Severity: Medium
**Finding.** Cross-tenant lineage queries (for legitimate platform analytics) lack separate audit.
**Recommendation.** Cross-tenant lineage operations through a separate authorized API; explicit audit per section 7.3.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### ARCH-LIN-012 — Severity: Medium
**Finding.** Lineage records lack source-corpus version; investigating "what was the corpus state when this answer was produced" requires correlation.
**Recommendation.** Capture corpus_version on every retrieval lineage record.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-LIN-013 — Severity: Medium
**Finding.** Embedding model version is not propagated through the chain; "which embedding model produced these vectors" requires manual correlation.
**Recommendation.** Per section 4.3, embedding lineage records carry the embedding model version.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-LIN-014 — Severity: Medium
**Finding.** Lineage store health (write success rate, query latency) is not monitored.
**Recommendation.** Lineage SLIs and alerting; quarterly health review.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ARCH-LIN-015 — Severity: Low
**Finding.** Prompt version is not in the answer lineage; correlating "which prompt produced this output" requires the trace, not the lineage.
**Recommendation.** Capture prompt_version on every answer lineage record.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-LIN-016 — Severity: Low
**Finding.** Lineage schema is undocumented; cross-team consumers cannot reason about the available fields.
**Recommendation.** Schema documentation; sample queries; commit alongside the implementation.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-LIN-017 — Severity: Low
**Finding.** Lineage store does not support efficient "find all answers citing chunk X" queries.
**Recommendation.** Add index / denormalization to support reverse lookups.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-LIN-018 — Severity: Low
**Finding.** Lineage-store size growth projections are not modeled; capacity planning is reactive.
**Recommendation.** Quarterly capacity projection based on growth trends; alert on storage-pressure.
**Owner.** ai-platform-eng + sre, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team without lineage tracking:

- [ ] **Sprint 0 — design.** Define the lineage schema. Decide store choice. Define the use cases driving the design (citations, audit, debugging, contamination detection).
- [ ] **Sprint 1 — store and writer.** Build the lineage store and the lineage-writer service.
- [ ] **Sprint 1 — first chain.** Instrument the source → chunk → embedding chain (the ingestion pipeline).
- [ ] **Sprint 2 — retrieval lineage.** Instrument the retrieval wrapper to write retrieval lineage records.
- [ ] **Sprint 2 — answer lineage.** Instrument the answer-generation path to write answer lineage records.
- [ ] **Sprint 3 — citation validation.** Validate model-produced citations against the context's chunks; reject hallucinated citations.
- [ ] **Sprint 3 — audit interface.** Self-service interface for authorized auditors.
- [ ] **Sprint 4 — retention.** Tiered retention; cold archive for compliance.
- [ ] **Sprint 4 — contamination audit.** Use lineage to detect eval-train cross-contamination.
- [ ] **Sprint 5 — citation UI.** User-facing citation rendering; click-throughs; staleness warnings.
- [ ] **Ongoing — discipline.** Lineage writes on every artifact; quarterly contamination audit; capacity reviews.

A team that completes this sequence has the audit-grade traceability that turns "we hope the sources support the claims" into "we can prove which sources support which claims." For regulated workloads this is not optional.

---

## 12. References

- HIPAA Security Rule §164.312(b) — the audit requirements that drive Meridian's clinical lineage.
- OpenLineage specification — the open-source standard for data lineage; the AI patterns here are similar in spirit.
- ML reproducibility patterns (DVC, MLflow tracking) — analogous lineage for ML training pipelines.
- This repo: [data-architecture-for-ai/data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — the corpus-as-contract pattern that complements lineage.
- This repo: [data-architecture-for-ai/vector-store-architecture.md](./vector-store-architecture.md) (coming) — the storage-layer design lineage references.
- This repo: [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — the parallel pattern; lineage is what gives scope enforcement its forensic trail.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — ARCH-CARE-015 is the cross-link finding.
- This repo: [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) — ARCH-RAG-015 is the analog finding from the RAG side.
- Sibling repo: [ai-engineering-reference-architecture/eval-engineering/eval-engineering-playbook.md](https://github.com/jeremiahredden/eval-engineering-reference-architecture/blob/main/eval-engineering-playbook.md) — section 9 on contamination prevention, which uses lineage queries.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/rag-engineering) — the engineering pipelines that produce lineage records.
- Sibling repo: [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — the threat model for data lineage (tampering, injection via lineage manipulation).
