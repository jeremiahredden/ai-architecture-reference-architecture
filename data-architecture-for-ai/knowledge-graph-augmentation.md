# Knowledge Graph Augmentation

> **Audience.** Architects deciding whether to add a knowledge graph to a RAG-shaped feature. Tech leads whose team has been pitched "add a knowledge graph and quality improves." Teams that have a knowledge graph and want to understand how to integrate it with AI systems. **Scope.** The *architectural* decision and integration patterns — when a KG belongs in the architecture, when it doesn't, the graph-as-retrieval-source and graph-as-tool patterns, and the maintenance burden of the graph itself. Not the graph construction depth (out of scope for this folder). Not the entity-extraction engineering (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Knowledge graphs (KGs) have an unusual position in 2026 AI architectures: simultaneously hyped (every vendor pitches a "GraphRAG" feature; conference talks lionise their use) and unevenly adopted (most production AI features ship without one). The asymmetry reflects a real ambiguity: a KG can dramatically improve quality on the right workload, and can be an expensive maintenance burden on the wrong one.

The decision is architectural. It depends on:

- Whether the team's content has stable, identifiable entities and relationships worth modelling.
- Whether the queries genuinely benefit from traversal (multi-hop relationships, recursive entity exploration) vs flat retrieval.
- Whether the team can sustain the KG's construction and maintenance.
- Whether the alternatives (better retrieval, better re-ranking, better chunking) would deliver similar quality at lower operational cost.

The team that adds a KG without examining these often discovers, after months of construction work, that the KG-augmented retrieval doesn't materially outperform the vector retrieval it replaced — and now the team has two systems to maintain instead of one. The team that doesn't consider a KG when one genuinely fits ships a system whose quality is held back by the lack of structured relationship modelling.

This document is opinionated about four things:

1. **The default is no knowledge graph.** Most workloads ship well with vector retrieval + good chunking + good re-ranking. KGs are added when there's a specific reason, not by default.
2. **The graph's value lives in the queries that use it.** A graph without queries that benefit from traversal is overhead. The integration pattern (how queries actually use the graph) is what determines whether the investment pays off.
3. **The graph is a maintained dataset, not a one-time construction.** Schema evolves; entity definitions drift; relationships need updates. The maintenance cost is ongoing and significant.
4. **Hybrid (vector + graph) is the right pattern when KG is right.** Pure graph retrieval rarely outperforms hybrid; the graph complements vector retrieval rather than replacing it.

Structure: (2) when a KG fits and when it doesn't; (3) the three integration patterns; (4) graph extraction from unstructured text; (5) the maintenance burden; (6) the hybrid pattern; (7) graph stores and operational considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. When a knowledge graph fits — and when it doesn't

The decision criteria.

### 2.1 The "yes" criteria

A KG fits when the team's workload has all (or nearly all) of:

- **Stable, identifiable entities.** "Patient," "medication," "encounter," "procedure" — clear entity types with consistent identifiers across sources.
- **Meaningful, queryable relationships.** "Patient X was prescribed Medication Y by Provider Z on Date D" — relationships that the queries care about, not just incidental connections.
- **Queries that benefit from traversal.** "What medications are contraindicated with the patient's current prescriptions, considering their conditions?" — requires hopping across multiple entities. Flat retrieval can't easily answer this; graph traversal can.
- **Existing structured data sources.** A canonical data source (an EHR with codified prescriptions, an enterprise catalog with structured product data, a CRM with relationship records) provides the seed entities and relationships.
- **A team that can own the graph as a dataset.** Schema design, ongoing maintenance, conflict resolution.

When all are present, a KG can meaningfully improve quality on the queries that benefit from traversal.

### 2.2 The "no" criteria

A KG is the wrong investment when:

- **Content is primarily unstructured text without clear entities.** "Summarise these meeting notes" — KG adds little.
- **Queries are flat (single-hop).** "Find documents about topic X" — vector retrieval handles this; the graph adds nothing.
- **The team doesn't have a stable schema vocabulary.** Building the KG requires deciding entity types and relationships; without domain consensus, the KG is over-engineered or wrong.
- **The team can't sustain construction and maintenance.** A KG that decays in quality is worse than no KG (consumers come to trust it, then are misled).
- **Vector retrieval + hybrid lexical-vector + reranking can deliver the quality.** Many workloads that "feel like they need a KG" can be served better by improving the existing retrieval stack.

### 2.3 The diagnostic questions

Before committing to a KG:

1. Can you write 5 specific queries the KG should answer? If not, the use case isn't clear.
2. For each, can you describe the traversal the KG would do? If "just look up X," the KG isn't needed.
3. Where would the entity data come from? If "we'd have to construct it," is the construction cost justified?
4. Who maintains the KG ongoing? If "nobody designated," the project will fail.
5. What's the alternative (better retrieval, better re-ranking, fine-tuning)? If alternatives might suffice, try them first.

Honest answers to these questions reduce KG adoption to the cases where it actually helps.

### 2.4 The vendor pitch

KG vendors (Neo4j, TigerGraph, Memgraph, AWS Neptune, in 2026 also vector-graph hybrids like LlamaIndex's GraphRAG, Microsoft GraphRAG) pitch KGs as broadly applicable. The pitch often comes with impressive demos on curated content.

Production results are more mixed. Many teams that adopted KGs based on vendor pitches discovered the maintenance burden exceeded the quality gain on their actual workloads.

The corrective: evaluate against the team's own workload, with the team's own data, on the team's own eval set. Demos are insufficient evidence.

### 2.5 The "GraphRAG paper" effect

Microsoft's 2024 GraphRAG paper popularised a specific pattern: use an LLM to extract entities and relationships from unstructured text, build a graph, query the graph for context. The paper showed quality gains on certain task types.

The popularisation produced both legitimate adoption (teams whose workloads genuinely benefit) and cargo-cult adoption (teams that built GraphRAG-shaped systems without confirming the fit). The latter is the common failure pattern.

If considering GraphRAG specifically: read the paper carefully, identify whether your workload matches its task profile, evaluate on your eval set before committing to the architecture.

### 2.6 The "if we had a KG" hypothetical

Some teams imagine "if we had a KG, we could do X." The hypothetical often skips:

- The construction cost.
- The maintenance cost.
- The integration complexity.
- The eval to verify X actually improves with KG.

The corrective: rather than the hypothetical, prototype the KG-augmented version on a sample of the workload; measure the quality difference; decide based on data.

### 2.7 The category-killer alternative: just better retrieval

Many "we need a KG" intuitions are met by:

- Better chunking (per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) and the engineering sibling's `chunking-engineering.md`).
- Hybrid lexical-vector retrieval (per [hybrid-store-architecture.md](./hybrid-store-architecture.md)).
- Reranking (per the engineering sibling's `reranking-engineering.md`).
- Query rewriting (per the engineering sibling's `query-rewriting.md`).

Try these first. If they suffice, skip the KG.

---

## 3. The three integration patterns

When a KG is right, three patterns govern how queries use it.

### 3.1 Pattern A — Graph as retrieval source

The KG is queried for relevant entities; the entities (and their context) are formatted as part of the model's input.

```
User query → entity recognition → graph traversal → subgraph extraction → 
  format subgraph as text → include in model prompt → generation
```

The retrieved subgraph is rendered as text (often a list of facts: "Patient X was diagnosed with Y on Date D; X is currently on medications A, B, C; X's allergies include P, Q"). The model consumes the text alongside any vector-retrieved chunks.

**Pros.**
- The graph's structure produces compact, precise context.
- Hop-based traversal captures relationships flat retrieval misses.
- The text format is model-friendly.

**Cons.**
- Format engineering required (how to render the subgraph as text).
- The subgraph extraction logic must be tuned per query type.
- The graph's gaps show up as missing context.

**When right.** When the team's queries map to specific traversal patterns; subgraph rendering is tractable.

### 3.2 Pattern B — Graph as tool

The graph is exposed as a tool the model can call. The agent or LLM decides when to query the graph and what to query for.

```
User query → agent loop → decision to call graph tool with query → 
  graph returns results → agent processes → final answer
```

This is a tool in the [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md) sense from the engineering sibling. The model decides the query.

**Pros.**
- Model autonomy: agent decides when graph helps.
- Flexible: agent can issue different graph queries on different turns.
- No need to anticipate every query pattern upfront.

**Cons.**
- Agent-shaped problem (per the sibling repo's [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md)); operational burden.
- Tool design matters; the graph-query tool needs careful design.
- Cost: agent loops cost more than direct retrieval.

**When right.** When queries are diverse and the right graph query is hard to predict; the workload is already agent-shaped.

### 3.3 Pattern C — Graph as filter / scope

The graph is queried first to determine which documents are relevant; vector retrieval then runs against those documents.

```
User query → entity recognition → graph identifies relevant documents → 
  vector retrieval scoped to those documents → generation
```

The graph narrows the retrieval space; vector retrieval finds the specific chunks.

**Pros.**
- Combines graph's structural understanding with vector's semantic matching.
- Reduces noise (vector retrieval against a smaller, more relevant set).
- Often improves precision without sacrificing recall.

**Cons.**
- Two-stage retrieval (graph + vector) adds latency.
- The graph's scoping must be reliable; missed entities mean missed documents.

**When right.** When the team has both graph structure (entities, relationships) and unstructured content (documents, notes); the queries benefit from scoping.

### 3.4 The choice among patterns

| Criterion | Pattern A | Pattern B | Pattern C |
| --- | --- | --- | --- |
| Workload shape | Workflow / RAG | Agent | Workflow / RAG |
| Query diversity | Bounded | Diverse | Mixed |
| Operational complexity | Medium | High | Medium |
| Latency | Lower | Higher | Medium |
| Precision benefit | High when traversal matches query | High when agent reasons well | High via scoping |

Most production teams use Pattern A or Pattern C; Pattern B is reserved for genuinely agent-shaped workloads.

### 3.5 The "Cypher / SPARQL query as tool" specialisation

In Pattern B, the graph tool's interface matters. Common choices:

- **Natural language query.** Tool accepts "What medications is patient X on?"; tool layer translates to Cypher / SPARQL; executes; returns. The translation layer is itself an engineering investment.
- **Structured query.** Tool accepts structured parameters (`entity_type: medication, related_to: patient_id, relation: prescribed`); tool layer generates the query.
- **Templated query.** Tool exposes specific query templates; agent picks the template and fills parameters.

Templated queries are typically the right starting point — bounded surface, predictable behaviour, easier to eval. Natural-language-to-graph is harder to engineer reliably; structured queries are between.

### 3.6 The hybrid pattern (combinations)

Patterns A, B, C are not mutually exclusive. A production system might:

- Use Pattern A as the primary retrieval (graph subgraph + vector chunks in the prompt).
- Expose Pattern B as a tool for follow-up clarification queries.
- Use Pattern C to filter the vector search.

Most production KG-augmented systems use 2 of the 3 patterns in combination.

---

## 4. Graph extraction from unstructured text

When the source data is unstructured, building the graph is itself an engineering project.

### 4.1 The extraction pipeline

```
Unstructured text → entity recognition → relationship extraction → 
  entity disambiguation → graph storage
```

Each step has its own challenges.

### 4.2 Entity recognition

Identify spans in the text that refer to entities of interest. Approaches:

- **Rule-based / pattern matching.** Reliable for known patterns (drug names, dates, codes); brittle for free-form references.
- **Pre-trained NER models.** Generic entity types (PERSON, ORG, LOC); often need domain adaptation.
- **LLM-based extraction.** Prompt-engineered extraction; flexible; expensive at scale; quality varies.
- **Fine-tuned models.** Domain-specific NER; best quality for stable domains; investment required.

The 2026 default for many domains is LLM-based extraction with structured output (per the engineering sibling's [structured-output-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/structured-output-engineering.md)). Quality is good; cost is the constraint.

### 4.3 Relationship extraction

Identify relationships between entities. Same approaches as entity recognition; harder problem because relationships are often implicit or distributed across multiple sentences.

The LLM-based approach is common in 2026; quality depends on the relationships' clarity in the source text.

### 4.4 Entity disambiguation

"Dr. Smith" in two documents — is it the same Dr. Smith? Disambiguation is the hardest part of graph construction. Approaches:

- **Identifier matching.** When sources have unique IDs (NPI numbers for clinicians, product codes), use them.
- **Context matching.** Same hospital, same specialty, same time period → likely the same person.
- **LLM-based disambiguation.** Provide candidate matches; LLM judges.

Wrong disambiguation produces wrong graph relationships; downstream quality suffers in ways that are hard to detect.

### 4.5 The construction cost

Building a KG from unstructured text is expensive:

- LLM calls for extraction (per document × extraction prompt cost).
- Engineering effort for the pipeline.
- Quality validation and disambiguation tuning.
- Storage and indexing.

For 1M documents at $0.05 per document extraction: $50,000 just for the LLM calls. Plus engineering time. Plus ongoing maintenance.

The cost has to be justified by the quality gain on the team's eval set.

### 4.6 The construction-on-demand alternative

For some workloads, building the graph on-demand from query-relevant documents is cheaper:

- Vector retrieval finds the top-N documents for a query.
- LLM extracts entities and relationships from those documents only.
- The mini-graph is used for the query, then discarded.

The pattern (sometimes called "lazy GraphRAG") avoids the massive upfront construction cost. Trade-off: per-query latency and cost go up; the graph isn't reusable across queries.

For some workloads this is the right pattern — especially exploratory, low-volume use cases where the upfront construction would be over-investment.

### 4.7 The "the graph is wrong" failure mode

Extraction is imperfect. Some fraction of entities and relationships will be wrong (typos, misidentifications, hallucinations from LLM extraction). The error rate compounds: a 95%-accurate extraction over multi-hop queries produces results with much lower accuracy.

Mitigations:

- Validation against known-good sources where possible.
- Confidence scores on extractions; downstream consumers consider confidence.
- Iterative improvement: production usage reveals errors; corrections feed back into the extraction pipeline.

Total elimination of errors is infeasible; design the graph's downstream consumers to tolerate some level of error.

---

## 5. The maintenance burden

The KG isn't a one-time construction. The maintenance is ongoing.

### 5.1 What requires maintenance

- **New entities.** As the source data grows, new entities appear; the graph needs to incorporate them.
- **Changing entities.** Patient's address changes; medication formulary updates; provider's affiliation shifts.
- **Schema evolution.** New entity types or relationships emerge; the schema accommodates.
- **Disambiguation corrections.** Errors discovered in production are corrected.
- **Quality monitoring.** Drift in extraction quality, in disambiguation, in coverage.

### 5.2 The maintenance cost

For a production KG with millions of entities and tens of millions of relationships, maintenance typically:

- 0.5-2 FTE engineering ongoing.
- Storage and compute costs (relative to graph size).
- Periodic eval and re-extraction (every 6-12 months for a stable domain; quarterly for dynamic).

The team that doesn't budget for maintenance discovers, 12 months in, that the graph is drifting and quality is degrading.

### 5.3 The schema-drift failure

The schema changes over time:

- New entity types emerge (the team adds "device" entities to a patient graph that previously only tracked patients and medications).
- Relationship types change (the team realises "prescribed_by" should distinguish initial prescription from refills).

Each change requires:

- Updates to the extraction pipeline.
- Migration of existing data.
- Updates to query patterns.
- Updates to consumer code.

The drift cost compounds. Schema discipline (per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md)) helps but doesn't eliminate.

### 5.4 The disconnection from source

A common failure: the KG becomes disconnected from the source data. The source updates; the KG doesn't. The graph reflects the world six months ago; queries return stale information.

Mitigations:

- Change-data-capture (CDC) integration: source changes flow to the KG.
- Periodic re-construction (more expensive but simpler).
- Per-entity freshness SLO: track when each entity was last updated.

The integration patterns are the same as broader data freshness (per [freshness-architecture.md](./freshness-architecture.md)).

### 5.5 The KG as orphan project

KGs are often built as projects, then orphaned when the project ends. The team that built it moves on; nobody owns it; it decays.

The mitigation: assign ownership before construction; budget for ongoing maintenance; treat the KG as a long-term commitment, not a one-time project.

If ongoing ownership can't be committed, don't build the KG.

### 5.6 The KG retirement decision

Sometimes the right answer is to retire the KG:

- Maintenance burden exceeds the quality gain.
- Alternatives (better retrieval, fine-tuned models) deliver similar quality at lower cost.
- The team's priorities shift.

Retirement is honourable. A maintained-or-retired choice is better than the slow decay that ends as a quality incident.

---

## 6. The hybrid pattern (vector + graph)

When KG is right, the hybrid (vector + graph) is usually the production architecture.

### 6.1 Why hybrid

- **Vector excels at semantic similarity.** "Documents about the patient's condition" — vector retrieval finds the relevant chunks.
- **Graph excels at structured traversal.** "All medications prescribed to the patient by their cardiologist" — graph traversal is precise.

Combined, they cover both modes.

### 6.2 The architecture

```
User query → query understanding → 
  vector retrieval (for semantic content) +
  graph traversal (for structured relationships) → 
  merged context →
  generation
```

The query understanding step routes parts of the query to the appropriate engine; the results merge into a unified context.

### 6.3 The merging

Two outputs (vector chunks + graph subgraph) need to merge into the model's prompt. Common patterns:

- **Concatenation with sections.** "Structured context (from graph): ... Relevant passages (from search): ..."
- **Interleaved by relevance.** Reranking the merged result; top-N regardless of source.
- **Graph-as-filter (Pattern C from section 3.3).** Graph scopes the vector retrieval; only vector results are passed to the model.

Each has tradeoffs; the choice is per workload.

### 6.4 The query understanding step

The hybrid relies on knowing which parts of the query need which engine. Implementations:

- **Heuristic / pattern-based.** If query mentions specific entities, use graph; if topical, use vector.
- **LLM-based routing.** A small LLM call decides per query.
- **Always-both.** Send to both engines; merge. Simplest; sometimes wasteful.

For most workloads, always-both is the starting point. Optimisation toward routing comes if the cost or latency matters.

### 6.5 The eval for hybrid

Hybrid eval needs to verify:

- The vector-only baseline (no graph).
- The graph-only baseline (no vector).
- The hybrid combination.

If hybrid doesn't materially outperform vector-only, the graph isn't pulling its weight. The eval surfaces this; the team decides.

### 6.6 The hybrid's operational complexity

Two engines means two operational stacks. Two failure modes. Two cost lines. Two upgrade paths.

The complexity is real. Budget for it. The hybrid's quality gain has to justify the complexity over vector-only.

---

## 7. Graph stores and operational considerations

The infrastructure layer.

### 7.1 The graph store categories

| Category | Examples | Characteristics |
| --- | --- | --- |
| Native graph DB | Neo4j, TigerGraph, Memgraph, ArangoDB | Purpose-built for graph queries; mature query languages (Cypher, GSQL) |
| Multi-model DB | ArangoDB, OrientDB, Cosmos DB Gremlin | Graph + document + key-value in one store |
| RDF triple stores | Stardog, GraphDB, Virtuoso | SPARQL-based; semantic web heritage |
| Cloud-managed graph | AWS Neptune, Azure Cosmos DB Gremlin | Managed service; lower operational burden |
| Graph-on-vector or hybrid stores | LlamaIndex GraphRAG, Microsoft GraphRAG | Newer; AI-specific; varying maturity |

The store choice is secondary to whether to use a graph at all; once decided, the choice is similar to vector-store-architecture's logic (managed vs self-hosted, capacity, multi-tenancy).

### 7.2 The query language

- **Cypher.** Neo4j's language; widely supported; declarative.
- **SPARQL.** RDF triple stores; older; powerful for semantic queries.
- **Gremlin.** TinkerPop; widely supported; more imperative.
- **GraphQL.** API layer over graphs; common as a frontend.

The query language is part of the store choice; switching is non-trivial.

### 7.3 Capacity and scale

Graph stores scale differently than relational. Considerations:

- **Vertex and edge counts.** Order of magnitude affects choice; some stores handle billions, others struggle past hundreds of millions.
- **Query patterns.** Deep traversal (5+ hops) is harder than shallow traversal.
- **Write throughput.** High-write workloads may need different store than read-heavy.

Capacity planning per the team's projected scale and query patterns.

### 7.4 The integration with vector stores

The graph store and vector store often live in different systems. The integration:

- Cross-system queries (orchestration code in the application layer).
- Co-located deployment (latency considerations).
- Shared identifiers (entities in the graph map to documents in the vector store).

The integration is the team's engineering investment; not something the stores do automatically.

### 7.5 Multi-tenancy

Per [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) considerations apply to graph stores too. Per-tenant subgraphs, tenant-scoped queries, cross-tenant leakage prevention.

The graph store may or may not have first-class multi-tenancy features; check before adopting.

### 7.6 Observability

Per-query latency, hit counts, error rates — standard observability. Graph-specific:

- Per-traversal-depth metrics.
- Per-entity-type query frequencies.
- Graph staleness (last-update age per entity).

The metrics feed dashboards and alerts in the same way as other infrastructure.

### 7.7 Backup and disaster recovery

Graphs are state. They need backup. They need DR procedures. The standard data-platform discipline.

For graphs constructed from source data, the construction process itself is a form of recovery (rebuild from source). But the construction is expensive; backups of the constructed graph are still worth maintaining.

---

## 8. Worked Meridian example

Meridian's KG architecture decisions — including the decision not to build one for most workloads.

### 8.1 Care-coordinator: no knowledge graph

The care-coordinator agent was evaluated for KG augmentation. The team built a prototype using GraphRAG-style extraction from clinical notes.

Evaluation results:

- Vector retrieval + reranking: 4.21 on golden set.
- Vector + KG (Pattern A): 4.27.
- Cost: KG-augmented adds ~$0.04 per query (graph traversal + LLM extraction overhead).

The 1.4% quality gain didn't justify the cost premium and the construction/maintenance burden. The KG project was retired; the team focused on retrieval improvements instead.

This is the "no KG" outcome — explicitly evaluated, explicitly rejected, documented. Not the "we never considered it" pattern.

### 8.2 Patient-graph for the patient-API copilot: yes, hybrid (Pattern C)

The patient-API copilot helps developers integrate with Meridian's APIs. It uses a patient-graph for one specific purpose: when a developer asks about a specific patient's data flow ("what happens when this patient's record is updated?"), the graph identifies relevant API endpoints and dependencies.

Pattern: graph as filter (Pattern C). The graph scopes which API endpoints are relevant to the patient's data type; vector retrieval finds the documentation chunks for those endpoints.

The graph is small (~5k entities — APIs, schemas, dependencies) and stable (changes when APIs change, weekly cadence). Construction is partly manual (the team curates the API entity list); partly automated (dependencies extracted from API specs).

Outcome: developer query quality improved 8% over vector-only on the graph-relevant query slice. The investment (~3 weeks construction, ~5% of one engineer's time ongoing) justified.

### 8.3 Clinical-knowledge graph: considered, rejected

Meridian considered building a comprehensive clinical-knowledge graph (diseases, medications, procedures, interactions) for use across all features.

Rejection reasons:

- Existing clinical knowledge bases (RxNorm, SNOMED, ICD-10) provide much of this; building a custom graph would duplicate.
- Maintenance burden for a comprehensive graph would have required 2+ FTE ongoing.
- Most use cases were served by retrieving from RxNorm / SNOMED via API tools rather than a unified graph.
- The team's eval suggested marginal quality gain vs the tool-based retrieval.

The team uses tool-based access to standard clinical knowledge bases (RxNorm, SNOMED-CT, etc.) instead of a custom graph. Lower operational cost; comparable quality.

### 8.4 The decision pattern

Meridian's pattern is consistent: evaluate KG against the specific workload; reject when alternatives are sufficient; adopt when the KG genuinely outperforms.

Two adoptions across all features:

- patient-API copilot's patient-graph (Pattern C).
- analytics-warehouse copilot's schema-graph (relationships between warehouse tables; used to inform SQL generation; Pattern B as a tool).

Several rejections:

- Care-coordinator KG (insufficient quality gain).
- Comprehensive clinical-knowledge graph (existing knowledge bases sufficed).
- Patient-records KG (too large for the value; vector retrieval over chunked records worked).

### 8.5 The maintenance investment

Two production graphs:

- Patient-API graph: ~5% of one engineer's time ongoing.
- Warehouse schema graph: ~3% of one engineer's time ongoing (mostly auto-updated from schema changes).

Total: <10% of one FTE. Sustainable. The team won't build a third graph that would push past this.

### 8.6 The owner

Each graph has an explicit owner (the API team for the patient-API graph; the analytics team for the warehouse-schema graph). No "graph platform team" — graphs are workload-specific, owned by their workload's team.

### 8.7 What worked

- **Evaluation before commitment.** Each KG decision was data-driven.
- **Per-workload patterns.** No org-wide KG mandate; each workload decided independently.
- **Explicit retirement.** The care-coordinator KG was retired explicitly when eval showed insufficient ROI.
- **Lean maintenance.** Two small graphs are much cheaper to maintain than one large one.

### 8.8 What didn't work initially

- **Initial bias toward KG.** Early-2025 thinking was "we should have a KG"; took 6 months of evaluation to converge on "only where it actually helps."
- **One team's enthusiasm.** Sometimes a single engineer's interest pushed for adoption past the eval-justified threshold; the team's eval discipline corrected.
- **Underestimated extraction cost.** The clinical-knowledge graph prototype's construction cost was 3× the initial estimate; one factor in the rejection.

---

## 9. Anti-patterns

### 9.1 "Default to KG"

The team adopts a KG without examining whether the workload benefits. Construction and maintenance costs accumulate; quality gain doesn't materialise.

**Corrective.** Decision criteria per section 2; the default is no KG.

### 9.2 "KG without queries that benefit"

The graph is built; nobody can describe queries that benefit from its traversal. The graph is overhead.

**Corrective.** Diagnostic questions per section 2.3; if no specific queries, no KG.

### 9.3 "Pure-graph retrieval"

Replacing vector retrieval with graph-only retrieval. Misses semantic similarity; quality drops vs hybrid.

**Corrective.** Hybrid per section 6; the graph complements, doesn't replace.

### 9.4 "GraphRAG by cargo-cult"

The team builds GraphRAG because it's the trendy pattern; their workload doesn't match the paper's profile.

**Corrective.** Evaluate against the team's own workload per section 2.5.

### 9.5 "No maintenance budget"

The KG is built as a project; nobody owns it ongoing; it decays.

**Corrective.** Ownership and maintenance budget per section 5.5; before construction.

### 9.6 "Disconnected from source"

The KG was constructed once; the source data updates; the graph doesn't. The graph reflects stale state.

**Corrective.** CDC or periodic re-construction per section 5.4.

### 9.7 "Wrong query language commitment"

The team picks a query language that doesn't fit their needs (e.g., SPARQL for a workload that's mostly property-graph queries). Migration is expensive.

**Corrective.** Match query language to workload per section 7.2.

### 9.8 "Extraction errors propagate silently"

The graph has known error rate; downstream consumers treat graph data as ground truth; user-visible errors result.

**Corrective.** Confidence scores per section 4.7; consumers handle uncertain data appropriately.

---

## 10. Findings (sprint-assignable)

### ARCH-KG-001 — Severity: Critical
**Finding.** KG was adopted without per-workload evaluation; quality gain unverified; maintenance burden unowned.
**Recommendation.** Decision review per section 2; if KG not justified, plan retirement.
**Owner.** architecture + feature team, sprint N+1.

### ARCH-KG-002 — Severity: Critical
**Finding.** Production KG has no designated owner; quality drifting.
**Recommendation.** Owner assignment per section 5.5; ongoing maintenance budget.
**Owner.** architecture + leadership, sprint N+1.

### ARCH-KG-003 — Severity: High
**Finding.** KG disconnected from source data; staleness causing user-visible errors.
**Recommendation.** CDC or re-construction per section 5.4.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-KG-004 — Severity: High
**Finding.** Pure-graph retrieval used; quality below hybrid alternative.
**Recommendation.** Hybrid per section 6; evaluate.
**Owner.** ai-platform-eng + feature team, sprint N+2.

### ARCH-KG-005 — Severity: High
**Finding.** Graph extraction errors propagate without confidence scores; downstream consumers act on uncertain data.
**Recommendation.** Confidence per section 4.7; consumer-side handling.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-KG-006 — Severity: Medium
**Finding.** Considering KG for a workload without diagnostic questions answered; risk of unjustified investment.
**Recommendation.** Diagnostic per section 2.3; prototype + evaluate before commitment.
**Owner.** architecture + feature team, sprint N+3.

### ARCH-KG-007 — Severity: Medium
**Finding.** KG construction cost not budgeted; project will stall or overspend.
**Recommendation.** Cost projection per section 4.5; budget approved upfront.
**Owner.** architecture + finance, sprint N+3.

### ARCH-KG-008 — Severity: Medium
**Finding.** Schema evolution informal; changes break consumers silently.
**Recommendation.** Schema discipline per section 5.3; versioned schema; consumer notifications.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-KG-009 — Severity: Medium
**Finding.** No graph eval; quality measured anecdotally.
**Recommendation.** Per-workload eval per section 6.5; vector-only, graph-only, hybrid baselines.
**Owner.** ai-platform-eng + feature team, sprint N+3.

### ARCH-KG-010 — Severity: Medium
**Finding.** Multi-tenancy on the graph not implemented; cross-tenant leakage possible.
**Recommendation.** Per-tenant scoping per section 7.5; storage-level enforcement.
**Owner.** ai-platform-eng + security, sprint N+3.

### ARCH-KG-011 — Severity: Medium
**Finding.** Tool design for graph-as-tool pattern (Pattern B) ad-hoc; agent picks wrong queries.
**Recommendation.** Tool design per section 3.5; templated queries preferred.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-KG-012 — Severity: Medium
**Finding.** No alternative-retrieval baseline (better chunking, reranking) evaluated before KG investment.
**Recommendation.** Alternatives per section 2.7 evaluated; KG only if alternatives insufficient.
**Owner.** architecture + ai-platform-eng, sprint N+4.

### ARCH-KG-013 — Severity: Medium
**Finding.** Graph observability minimal; query latency, staleness, error rates not tracked.
**Recommendation.** Observability per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-KG-014 — Severity: Medium
**Finding.** Graph store choice locked in before workload understood; migration would be expensive if requirements change.
**Recommendation.** Capacity planning per section 7.3 before committing.
**Owner.** architecture, sprint N+4.

### ARCH-KG-015 — Severity: Low
**Finding.** Graph backup and DR procedures absent; the graph is irreplaceable state but unprotected.
**Recommendation.** Per section 7.7; standard data-platform DR discipline.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-KG-016 — Severity: Low
**Finding.** Construction-on-demand alternative (lazy GraphRAG) not considered for low-volume workloads.
**Recommendation.** Section 4.6 alternative; lower upfront cost.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-KG-017 — Severity: Low
**Finding.** Entity disambiguation runs without periodic correction loop; errors accumulate.
**Recommendation.** Correction feedback loop per section 4.4 / 4.7.
**Owner.** ai-platform-eng + feature team, sprint N+5.

### ARCH-KG-018 — Severity: Low
**Finding.** Graph retirement criteria undocumented; KGs may persist past usefulness.
**Recommendation.** Annual review per section 5.6; retirement is a valid outcome.
**Owner.** architecture, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team evaluating whether to add a KG:

- [ ] **Sprint 0 — diagnostic.** Walk through section 2.3 questions; document answers.
- [ ] **Sprint 0 — alternatives.** Per section 2.7; evaluate whether better retrieval suffices.
- [ ] **Sprint 1 — prototype.** Small-scale KG on a subset; integrate with one query pattern.
- [ ] **Sprint 1 — eval.** Vector-only baseline; KG-augmented; hybrid.
- [ ] **Sprint 2 — decision.** Promote, prototype-extension, or retire.

If promoting:

- [ ] **Sprint 2 — owner.** Designated; maintenance budget approved.
- [ ] **Sprint 2 — extraction pipeline.** Production-grade with quality monitoring.
- [ ] **Sprint 3 — integration.** Pattern A, B, or C; eval-validated.
- [ ] **Sprint 3 — multi-tenancy.** If applicable; storage-level enforcement.
- [ ] **Sprint 4 — observability.** Per section 7.6.
- [ ] **Sprint 4 — backup / DR.** Per section 7.7.
- [ ] **Sprint 5 — production rollout.** Gradual; eval-gated.
- [ ] **Ongoing — quarterly health check.** Quality, freshness, owner status.
- [ ] **Annually — re-evaluation.** Retire if no longer justified.

For a team with an existing KG of unclear ROI:

- [ ] **Sprint 0 — audit.** Quality gain vs alternatives; maintenance burden; query patterns actually used.
- [ ] **Sprint 1 — decision.** Continue, refactor, or retire.

A team that completes the sequence has KG investment that's justified by data. A team that doesn't builds graphs whose value is asserted rather than measured.

---

## 12. References

- [vector-store-architecture.md](./vector-store-architecture.md) — the vector side of the hybrid pattern.
- [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — corpus discipline applies to graph as well.
- [embedding-strategy.md](./embedding-strategy.md) — embeddings used in the hybrid pattern.
- [hybrid-store-architecture.md](./hybrid-store-architecture.md) — broader hybrid (vector + lexical) architecture.
- [lineage-and-provenance.md](./lineage-and-provenance.md) — graph data should also be lineage-tracked.
- [freshness-architecture.md](./freshness-architecture.md) — KG freshness as case.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant considerations apply to graphs.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — retrieval-side patterns the hybrid integrates with.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/reranking-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/reranking-engineering.md) — often the right alternative to KG.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md) — Pattern B's tool design.
- Microsoft GraphRAG paper (2024) — origin of the LLM-extracted-KG pattern.
- Neo4j, TigerGraph, AWS Neptune, Memgraph — major graph databases.
- RxNorm, SNOMED-CT, ICD-10 — standard clinical knowledge bases (alternatives to custom KG in healthcare).
- "Toward a Reading-and-Writing Knowledge Graph" — broader literature on KG in NLP / AI systems.
