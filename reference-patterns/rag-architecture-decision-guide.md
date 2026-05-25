# RAG Architecture Decision Guide

> **Audience.** Architects and staff engineers choosing the RAG shape for a new feature, or reviewing the RAG shape of an existing one. **Scope.** The architectural pattern decision and the graduation signals between patterns — not implementation, vendor selection, or operational practice (those live in the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/` folder). **Worked client.** Meridian Health, the fictional regulated healthcare SaaS used across all three sibling reference architecture repos.

---

## 1. Why this document exists

RAG is the most over-engineered shape in 2026 AI architecture. The reference diagrams almost all show the same hybrid-retrieval-with-reranker-and-query-rewriter-and-graph-augmentation pattern, regardless of whether the workload needs any of it. Teams adopt the full stack on day one because the diagrams said so, then spend the next two quarters fighting the cost, latency, and operational load that the choice imposed — without ever measuring whether the additional components actually lifted quality on their workload.

The corrective discipline is the same as in every other engineering domain: *pick the simplest pattern that meets the requirements, graduate to more complex patterns only on signal*. The signals are the failure modes the simpler pattern cannot address, demonstrated on the actual workload by evaluation, not anticipated by the architecture diagram.

So this guide is organized in roughly-increasing order of pattern complexity. Each pattern includes:

- **What it is.** The minimum architecture and the moving parts.
- **Sweet-spot workload.** Where it earns its operational cost.
- **What it cannot do well.** The failure modes that justify graduating to the next pattern.
- **Required eval surface.** What a team needs in place to validate this pattern and decide whether to keep it.
- **Cost and latency shape.** The order-of-magnitude operational profile.
- **What it looks like for Meridian Health.** Worked use within the cross-repo fictional client.

The pattern catalogue is followed by an explicit *graduation signals* table, three worked Meridian Health architectures at different complexity levels, and an anti-pattern checklist.

---

## 2. The questions to answer before picking a pattern

Every RAG architecture decision is shaped by the answers to five questions. Answer these in writing before reading the pattern catalogue; the right pattern usually becomes obvious.

1. **What is the *retrieval corpus*?** A static, curated, structured knowledge base (clinical guidelines, product documentation, regulatory filings) is a different workload from an open-ended document graveyard (every internal wiki page ever written), which is a different workload from a structured operational store (the patient database). The architecture follows the corpus, not the other way around.
2. **What does *good* look like, and who decides?** A retrieval-augmented answer can be correct-but-unverifiable, correct-with-citation, or correct-with-required-source. The latency, eval surface, and architectural choices change with the answer.
3. **What is the *latency budget*?** Interactive chat (under 3 seconds total), human-in-the-loop assistance (under 8 seconds), or batch / async (minutes acceptable). Latency budget rules out entire pattern classes — multi-step agentic RAG cannot meet a 2-second budget, period.
4. **What is the *cost budget per interaction*?** A 5-cent budget rules out reranker + long-context + agentic-RAG. A 50-cent budget opens every option. The cost shape changes with retrieval depth, model tier, and how many model calls happen per user-visible interaction.
5. **Who *owns* the retrieval corpus, and how often does it change?** A corpus owned by a publishing team with weekly updates is a different operational problem from a corpus that gets one rebuild a quarter, or from a corpus that updates continuously (a CMS hooked to the embedding pipeline).

The answers determine which patterns are even on the table. Trying to pick a pattern without these answers is the most common source of over-engineering — teams pick the "safe" complex pattern because they have not done the work to know they could pick simpler.

---

## 3. The pattern catalogue

### 3.1 Pattern A: Naive single-source RAG

**What it is.** One vector store, one embedding model, top-K retrieval, retrieved chunks formatted into the prompt, single LLM call to generate the answer. No reranking, no query rewriting, no agent loop.

```
        ┌──────────────┐
query → │  Embed query │ → vector search top-K → format chunks into prompt → LLM → answer
        └──────────────┘                                                      │
                                                                              ▼
                                                                          (optional)
                                                                          citation
```

**Sweet-spot workload.** A small-to-medium corpus (under a million chunks), with documents that are reasonably uniform in topic and structure, where users ask questions whose answers tend to live in one or two chunks. Examples: a product-documentation Q&A, an internal-wiki assistant for a small team, a customer-support knowledge-base lookup.

**What it cannot do well.**
- **Lexical-precise queries.** Vector search alone is bad at queries dominated by rare keywords, codes, or identifiers (drug names, ICD-10 codes, SKU numbers, regulation paragraph numbers). The semantic similarity score does not give them enough weight.
- **Conversational follow-ups.** "What about the dosing for pediatric patients?" depends on the prior turn for context; naive RAG embeds and retrieves on the literal question, which fails.
- **Multi-hop questions.** Questions whose answer is constructed from multiple sources ("what is the interaction between drug A and condition B in patients on medication C?") rarely have all the relevant chunks in the top-K of a single retrieval.
- **Retrieval-quality ceiling.** Top-K from a single embedding model has a ceiling; if your eval shows recall@10 stuck at 70%, no amount of prompt-tuning will get the answer right when the answer is not in the top-10.

**Required eval surface.** A 30–100 case golden set with expected answers and expected source documents. Retrieval recall (was the right chunk in the top-K) and answer correctness (judged separately, manually or by LLM-as-judge). Without this you cannot tell whether a quality problem is a retrieval problem or a generation problem.

**Cost and latency shape.** Cheapest and fastest RAG pattern. One embedding call (~5ms), one vector search (~50–100ms), one LLM call (1–3s for chat-tier models). Total budget: 2–4 seconds, $0.005–$0.02 per interaction depending on chunk count and model tier.

**Meridian Health example.** The patient-education content lookup on the patient-API AI assist: a curated corpus of about 8,000 patient-education articles, lookup-shaped questions ("what should I expect from a colonoscopy?"), single-chunk answers in most cases. Naive RAG is correct here, and adding a reranker or agentic flow would add cost without lifting quality.

---

### 3.2 Pattern B: Hybrid (BM25 + vector)

**What it is.** Two retrievers run in parallel — a lexical retriever (BM25, OpenSearch, Elasticsearch) and a vector retriever — and the results are merged (reciprocal rank fusion, weighted sum, or score normalization with cutoff). The merged top-K goes into the prompt.

```
              ┌──── BM25 / lexical retrieval ────┐
              │                                  ▼
query → split │                              ┌───────┐
              │                              │ merge │ → format chunks → LLM → answer
              │                              └───────┘
              └──── vector retrieval ────────────▲
```

**Sweet-spot workload.** Any workload that includes named entities, codes, identifiers, exact phrasing requirements, or jargon — which in practice is almost every enterprise workload. Hybrid retrieval is the *default* for production RAG in 2026 because the lexical+vector combination consistently beats either one alone on real-world enterprise corpora.

**What it adds over Naive.**
- Catches lexical-precise queries that vector alone misses (the exact drug name, the exact regulation section).
- Improves recall on questions that mention rare terms.
- Provides a sanity check: when both retrievers agree on a result, confidence is higher; when they disagree, the merging strategy needs to decide whose vote wins.

**What it cannot do well.**
- Still has the multi-hop problem.
- Still has the conversational-follow-up problem.
- The merging strategy needs eval — RRF is a reasonable default, but a poorly-chosen merge can underperform either retriever alone.
- Operational complexity goes up: now two retrievers, two indices, two failure modes, two operational SLOs.

**Required eval surface.** Everything Naive needs, plus a measurement of whether hybrid actually lifts recall over either retriever alone. Hybrid usually lifts recall by 5–15 percentage points; if you measure no lift on your corpus, drop back to vector-only and save the operational cost.

**Cost and latency shape.** Two retrievals in parallel (~100–200ms total, fan-out fan-in), LLM call (1–3s). Total budget: 2–4 seconds, $0.01–$0.03 per interaction.

**Meridian Health example.** The clinical-guidelines retrieval inside the Care Coordinator: clinical guidelines are dense with named medications, dosing intervals, condition names, and CPT codes. BM25 alone misses paraphrased clinical questions; vector alone misses queries dominated by the rare drug name or exact ICD-10 code. Hybrid lifts retrieval recall from ~74% (vector-only) to ~89% on the Care Coordinator's eval set.

---

### 3.3 Pattern C: Hybrid + Reranker

**What it is.** Hybrid retrieval returns a larger candidate set (top-50 or top-100), and a reranker model (Cohere Rerank, BGE reranker, or in-context LLM rerank) scores each candidate against the query and returns the top-N (typically 5–10) to format into the prompt.

```
query → hybrid retrieval (top-50) → reranker (top-10) → format chunks → LLM → answer
```

**Sweet-spot workload.** Workloads where the top-50 candidates contain the right answer most of the time, but the right answer is rarely in the top-5 of either retriever's raw scores. This is common on heterogeneous corpora (mixed-format documents, mixed-recency content, mixed-topic chunks within the same document).

**What it adds over Hybrid.**
- Lifts precision-at-K significantly when the corpus is heterogeneous. Reranker scores are *query-aware* in ways the retriever scores are not — the reranker can recognize that a chunk that scored medium on BM25 and medium on vector is actually a strong match for the specific question, while a chunk that scored high on both is actually about a different topic.
- Provides a calibrated relevance score the prompt-formatter can use to decide how many chunks to include and how to weight them.

**What it cannot do well.**
- Adds latency (50–500ms for the reranker pass) and cost ($0.001–$0.01 per query for hosted rerankers, or compute time for self-hosted).
- Does *not* fix retrieval-recall problems — if the right chunk is not in the top-50 of hybrid retrieval, the reranker cannot rescue it.
- Reranker score calibration drifts as the corpus changes; needs periodic recalibration of cutoff thresholds.

**Required eval surface.** Hybrid eval plus rerank-pass evaluation: does the reranker actually place the gold-standard chunk higher than hybrid retrieval did? If not, the reranker is adding cost without quality lift and should be removed.

**Cost and latency shape.** Adds 50–500ms and ~$0.001–$0.01 per query over Hybrid. Total budget: 2.5–5 seconds, $0.015–$0.04 per interaction.

**Meridian Health example.** The Care Coordinator's clinical-guidelines retrieval after corpus growth past about 50,000 chunks. Without the reranker, the right guideline was reaching the top-10 about 78% of the time; with the reranker (Cohere Rerank 3.5), the right guideline reaches the top-5 about 91% of the time. The reranker pays for itself by the resulting reduction in cases that escalate to the human clinician for clarification.

---

### 3.4 Pattern D: Query Rewriting

**What it is.** Before retrieval, an LLM call rewrites the user query — expanding it with synonyms, decomposing it into sub-queries, or contextualizing it with the recent conversation turns. The rewritten query (or queries) goes through retrieval.

```
user query + recent turns → LLM (rewrite) → rewritten query(s) → retrieval → format → LLM → answer
```

**Sweet-spot workload.** Conversational systems where follow-up questions depend on prior context. Workloads where users phrase questions in shorthand that does not match the corpus vocabulary. Multi-hop questions where decomposition into sub-queries enables parallel retrievals that the prompt can then reason over.

**What it adds.**
- Solves the conversational-follow-up problem ("what about the pediatric dosing?" becomes "what is the pediatric dosing for [medication discussed in the previous turn]?").
- Improves recall on shorthand queries by expanding to corpus vocabulary.
- Enables multi-hop retrievals by decomposition.

**What it cannot do well.**
- Adds an LLM call to the critical path (200ms–2s depending on model tier).
- Can hurt quality if the rewriter hallucinates context that was not actually in the conversation.
- Requires its own eval — the rewriter's output is now a debuggable surface, and rewrite failures look like retrieval failures in the trace if not separately instrumented.

**Required eval surface.** Hybrid eval plus a per-rewrite eval: did the rewrite preserve the user's intent? Did the rewrite improve recall over the raw query? If not, drop the rewriter for that query class.

**Cost and latency shape.** Adds one LLM call (200ms–2s, $0.001–$0.01 depending on tier). Total budget: 3–6 seconds, $0.02–$0.05 per interaction.

**Meridian Health example.** The Care Coordinator uses query rewriting only for conversational turns after the first ("what about the pediatric dosing?" → rewritten using the previous-turn medication). For first-turn queries the cost is not justified. The rewriter runs on Haiku-tier; using a more capable model showed no measurable lift in rewrite quality.

---

### 3.5 Pattern E: Multi-step / Agentic RAG

**What it is.** The retrieval step is itself an agent loop. The LLM is given retrieval as a tool, decides what to retrieve based on the query, evaluates the results, decides whether to retrieve again with a different query, possibly decomposes into sub-questions, and only generates a final answer when it judges that it has enough information.

```
query → agent loop:
        ├── retrieve tool (may call multiple times with different queries)
        ├── evaluate retrieved info
        ├── decide: retrieve more? answer now? give up?
        └── eventually: format final answer
```

**Sweet-spot workload.** Multi-hop questions whose retrieval cannot be planned up front. Research-shaped workflows where the user expects the system to "dig in" rather than answer from a single retrieval. Question-answering over heterogeneous corpora where different question classes require different retrieval strategies.

**What it adds.**
- Handles questions whose retrieval plan emerges as information is gathered.
- Can recover from initial retrieval failures by trying different queries.
- Naturally produces a citation trail of what was retrieved and used.

**What it cannot do well.**
- Operationally expensive. Each loop turn is a model call plus a retrieval. A 5-turn loop costs 5x a single-call answer and takes 5x the latency.
- Hard to evaluate. Trajectory eval is required (not just outcome eval), and the trajectory varies between runs of the same query.
- Failure modes are agent failure modes — loops, runaway cost, wrong-tool-call — and need agent-grade observability.
- Latency is incompatible with sub-3-second chat budgets.

**Required eval surface.** Agent-grade evals (see the engineering sibling's `agent-engineering/` folder). Trajectory and outcome eval. Cost-per-completion as a tracked metric. Turn-count as a tracked metric.

**Cost and latency shape.** 3x–10x the cost and latency of single-call RAG. Total budget: 8–30 seconds, $0.10–$1.00 per interaction. Should be async or batch, not real-time chat.

**Meridian Health example.** The analytics-warehouse copilot uses agentic RAG to answer multi-step analytic questions ("show me the readmission rate trend for CHF patients on the new care pathway, broken out by hospital size") — the system retrieves table schemas, sample rows, query history, then constructs the query, then sometimes retrieves additional schema after seeing the result. Not used in the Care Coordinator (latency-incompatible) or the patient-API assist (cost-incompatible).

---

### 3.6 Pattern F: Graph-augmented RAG

**What it is.** A knowledge graph (entities and relationships extracted from the corpus or maintained separately) is part of the retrieval surface. Retrieval can navigate the graph (find connected entities) in addition to or instead of similarity search over chunks.

```
query → entity extraction → graph traversal → subgraph + chunks → format → LLM → answer
```

**Sweet-spot workload.** Workloads where the *relationships* between entities matter more than the textual content (drug-interaction graphs, supply-chain dependency graphs, organizational reporting graphs, citation networks). Workloads where the corpus already has graph structure that vector search cannot exploit.

**What it adds.**
- Handles relationship-shaped queries that vector or hybrid retrieval cannot ("who reports indirectly to X and worked on Y?").
- Enables structured reasoning over connected entities.
- Provides explicit, debuggable retrieval — the subgraph that informed the answer is visible.

**What it cannot do well.**
- Requires building and maintaining the graph. This is the largest operational cost in the pattern, and most teams underestimate it. The graph drifts as the corpus drifts; entity extraction quality directly bounds graph quality.
- Adds another retrieval surface to debug and another failure mode (entity-extraction failure, graph-traversal failure).
- Most workloads are not graph-shaped. The temptation to add a graph because "our data has relationships" is the largest source of over-engineering in this pattern.

**Required eval surface.** All of the above plus graph-quality eval (entity recall, relationship accuracy) and graph-traversal eval (did the right subgraph get retrieved for the right question).

**Cost and latency shape.** Variable. Graph traversal can be fast (milliseconds) but graph construction and maintenance is the dominant cost.

**Meridian Health example.** The Care Coordinator uses a small, hand-curated drug-interaction graph (about 12,000 nodes, 80,000 edges, sourced from FDA structured-product-labels) as a *targeted* augmentation — when the question is interaction-shaped, the graph is queried; otherwise it is not. A general-purpose knowledge graph over the entire clinical-guidelines corpus was considered and rejected: the maintenance load on a graph that size (continuous corpus updates, entity-extraction drift) exceeded the projected quality lift, and a hybrid+reranker pattern was retained for the general case.

---

### 3.7 Pattern G: Retrieval-free long-context

**What it is.** The relevant material is small enough to fit entirely in the model's context window, so no retrieval is needed — the whole corpus (or the whole document set) is dumped into the prompt. With 1M-context frontier models in 2026, this is a real option for some workloads.

```
query + entire corpus / document set → LLM → answer
```

**Sweet-spot workload.** Single-document or small-collection workloads where the document fits in context (under ~500K tokens, leaving room for system / question / answer). Reading-comprehension shapes. Workflows where the corpus *is* the user's uploaded document.

**What it adds.**
- No retrieval to design, build, eval, or maintain.
- Model sees the entire context — no risk of missed retrieval, no chunking artifacts, no embedding-version drift.
- Simpler operational model: there is no vector store, no embedding pipeline, no rebuild discipline.

**What it cannot do well.**
- Cost: input tokens dominate, and 500K input tokens at frontier-model pricing is several dollars per call.
- Latency: time-to-first-token on a 500K-input prompt is 5–15 seconds; total latency is incompatible with chat budgets.
- Does not scale beyond what fits in context — the moment the corpus exceeds the window, the team has to build retrieval anyway.
- Prompt-prefix caching can offset much of the cost for repeated queries over the same context, but caching strategy is now an architectural concern.

**Required eval surface.** Standard answer-correctness eval; retrieval recall is not applicable. *Lost-in-the-middle* eval is critical — long-context models often miss information in the middle of the prompt, and the eval must catch this.

**Cost and latency shape.** Most expensive per call. $0.50–$5 per interaction depending on context size and model tier. Latency 8–30 seconds.

**Meridian Health example.** Used for the patient-record summarization workflow ("summarize this patient's recent care history"): a single patient's record is uploaded, fits in context, and the workflow is async (batch, end-of-day summary generation) so latency is acceptable. Not used for any interactive workflow.

---

## 4. The graduation signals

The decision is not "which pattern is best in theory" — it is "what is the smallest pattern that meets requirements, and what signals should trigger graduation to a more complex one?" The table below names the signals.

| Currently using | Graduate to | Trigger signal (observed in eval, not anticipated in design) |
|---|---|---|
| **Naive** | **Hybrid** | Eval shows recall@10 stuck below 75% on queries with named entities or codes; users complain about missed exact-match results. |
| **Naive** | **Long-context** | Corpus is small and fits in context; cost / latency budget allows; team wants to skip building retrieval. |
| **Hybrid** | **Hybrid + Reranker** | Eval shows recall@50 is high (>90%) but precision@5 is low (<70%); chunks the model uses to answer are often at rank 20+. |
| **Hybrid** | **Query Rewriting** | Conversational workload; eval on second-turn queries shows recall significantly worse than first-turn; corpus vocabulary differs from user vocabulary. |
| **Hybrid + Reranker** | **Agentic RAG** | Multi-hop question class makes up >20% of traffic; latency budget allows multi-second answers; eval shows single-shot retrieval has a quality ceiling on those questions. |
| **Hybrid** | **Graph-augmented** | Question class is genuinely relationship-shaped (interaction, dependency, network); workload volume justifies building and maintaining the graph; eval shows hybrid retrieval cannot answer the relationship queries. |
| **Any** | **Long-context for a subset** | Per-user-document workflow (reading-comprehension over an uploaded file); latency / cost budget allows. |

Two patterns to *avoid graduating to* on intuition alone:

- **Agentic RAG before measuring.** The temptation to add an agent loop because "the agent can decide whether to retrieve again" is strong. Measure first: how often does the eval show that a single hybrid retrieval was insufficient and a second one would have helped? If under 10%, the agent is over-engineering.
- **Graph augmentation because the data is relational.** Lots of data is relational. The question is whether the *queries* are graph-shaped. Most are not.

---

## 5. Worked Meridian Health architectures

Three Meridian Health workloads use three different RAG architectures. The differences are not arbitrary — they fall out of the answers to the five questions in section 2.

### 5.1 Patient-API AI assist → Naive RAG

**Workload.** Patients use the Meridian patient app to ask questions about their upcoming appointments, their medications, their patient-education materials, and routine logistics ("when is my colonoscopy?", "can I eat before the procedure?", "what does my new medication do?"). High volume (~50,000 interactions/day), low risk (no clinical decision support, no order entry), conversational but mostly single-turn lookup-shaped.

**Five questions, answered.**
1. Corpus: ~8,000 patient-education articles, curated and reviewed by clinical educators, structured by topic.
2. Good: answer matches a sentence or paragraph in the source article; citation back to the article is shown to the user.
3. Latency budget: sub-3-second chat response.
4. Cost budget: ~$0.01 per interaction (high volume, low margin).
5. Corpus changes: weekly editorial updates by the patient-education team.

**Architecture.** Naive RAG. Single embedding model (`text-embedding-3-large` at 1536 dimensions), pgvector for storage (per-tenant namespace by hospital), top-5 retrieval, formatted into the prompt with citation metadata, single Haiku-class LLM call to generate the answer. No reranker, no query rewriting, no agent loop, no graph.

**Why not more.** Hybrid would lift recall by perhaps 5 points on the eval, but the corpus vocabulary is already aligned with patient vocabulary (the educators write for patients) so the lift is smaller than usual. The cost budget is tight; the operational simplicity of one retriever is worth the ~5-point recall trade. Conversational turns are rare in this workflow (most patient questions are stand-alone), so query rewriting is not justified. The reranker would add cost the per-interaction budget does not allow.

**Eval surface.** 80-case golden set, refreshed quarterly. Retrieval recall, answer correctness (manual review by clinical educators on a 20% sample), citation accuracy. Quality SLO: 95% answer-correct, 90% retrieval-recall@5.

**Architecture sketch.**

```
patient question
     │
     ▼
[ AI gateway: PHI filter, per-tenant routing, cost accounting ]
     │
     ▼
[ Embed query (text-embedding-3-large) ]
     │
     ▼
[ pgvector search, top-5, filter by hospital_id ]
     │
     ▼
[ Format chunks into prompt with citation metadata ]
     │
     ▼
[ Haiku-class LLM call ]
     │
     ▼
[ Output guardrail: PHI re-check, citation validation, format check ]
     │
     ▼
patient answer (with citations)
```

### 5.2 Care Coordinator clinical agent → Hybrid + Reranker + Query-rewrite + targeted Graph

**Workload.** Clinical staff use the Care Coordinator to coordinate patient care — looking up clinical guidelines, drug interactions, scheduling logistics, internal-protocol references, and (with human approval) drafting clinical orders. Lower volume (~3,000 interactions/day across all customer hospitals), higher risk (clinical decision support, agent-style behavior, eventual order entry), heavily conversational, mixed question shapes.

**Five questions, answered.**
1. Corpus: ~50,000 chunks across clinical guidelines (curated, dense with codes and named medications), internal-protocol documents (hospital-tenant-specific), and a hand-curated drug-interaction graph (~12,000 nodes from FDA structured-product-labels).
2. Good: answer matches a clinical guideline or protocol with explicit citation; for any drug-interaction claim, the answer must cite the FDA-source graph; for any order-entry assistance, the clinician approves before the order is created.
3. Latency budget: 6 seconds for chat-shaped questions, 12 seconds for multi-step coordination tasks.
4. Cost budget: ~$0.20 per interaction (lower volume, higher value, clinical context).
5. Corpus changes: clinical guidelines updated quarterly; hospital protocols updated by hospital-tenant editors continuously; drug-interaction graph updated monthly from FDA SPL feed.

**Architecture.** Hybrid retrieval (BM25 + vector) + Cohere Rerank-3.5 for the general clinical / protocol retrieval. Query rewriting (Haiku-tier) for second-turn-and-later conversational queries. A *targeted* graph augmentation: when the question is interaction-shaped (classified by a small upfront LLM call), the drug-interaction graph is queried and results are formatted into the prompt alongside the retrieved chunks. Supervisor / worker agent topology — Opus-class for clinical reasoning and the supervisor role, Sonnet-class for the orchestration / planning sub-agent, Haiku-class for the classifier and the query rewriter. (Topology details in [meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md).)

**Why not less.** Naive RAG measured ~74% recall on a 200-case clinical golden set (clinical jargon is too keyword-heavy for vector alone). Hybrid lifted to 89%. Reranker (Cohere) lifted precision@5 from 71% to 88%. Query rewriting lifted second-turn recall from 64% to 86% on a 50-case conversational subset. Graph augmentation specifically for interaction queries lifted *interaction-question* answer-correctness from 81% to 96%; it was rejected as a general retrieval surface because the maintenance load on a corpus-wide graph would have required two FTEs the team did not have.

**Why not more.** Long-context RAG was considered for the per-patient summarization sub-task and rejected on cost (with 240 hospital customers and a coordinator interaction per patient per shift, the per-interaction cost dominates). Full agentic RAG (the agent decides retrieval queries dynamically inside its loop) was considered and is in fact used for a *sub-class* of multi-hop coordination queries; for the bulk of single-step clinical lookups, the deterministic hybrid+rerank+optional-graph flow is faster and cheaper.

**Eval surface.** 200-case clinical golden set + 50-case conversational subset + 60-case interaction-question subset, refreshed quarterly with new cases added from production incidents. Retrieval recall, retrieval precision, answer correctness (LLM-as-judge calibrated against clinician review on a 10% sample), citation accuracy, faithfulness. Quality SLOs: 95% answer-correct on clinical, 98% on interaction. Hard refusal rate (escalated to human) tracked as a separate metric.

### 5.3 Analytics-warehouse copilot → Agentic RAG (workflow-bounded)

**Workload.** Internal analysts and product managers use the copilot to ask analytic questions over the de-identified analytics warehouse ("show me the readmission-rate trend for CHF patients, broken out by hospital size, last 4 quarters"). Lower volume (~400 interactions/day), variable risk (no PHI in the warehouse but potential re-identification risk on certain joins), batch / interactive hybrid latency posture.

**Five questions, answered.**
1. Retrieval corpus: table schemas (~280 tables, ~4,500 columns), sample rows (per-table 10-row anonymized samples), query history (~6,000 historical queries + their text labels), and a curated semantic layer (~120 documented metrics).
2. Good: SQL that executes successfully against the warehouse, returns a result that matches the user's intent, and avoids columns / joins that would risk re-identification (governance policy enforced as a tool-call authorization).
3. Latency budget: 15–60 seconds for the typical interactive question, longer for complex queries.
4. Cost budget: ~$0.30 per query (lower volume, higher per-query value).
5. Corpus changes: table schemas updated weekly from the warehouse; semantic layer updated continuously by the analytics-platform team; query history updated continuously.

**Architecture.** Multi-step agentic RAG, but workflow-bounded — the agent's loop has a fixed structure (retrieve schema → propose SQL → validate → optionally retrieve more schema / sample rows → produce SQL → execute in sandbox → format result), with the LLM choosing only *which retrievals* to run inside that fixed structure, not the overall plan. Turn budget: 6 (most queries complete in 2–3). Cost budget: $1.00 per query as the circuit breaker. Tool-call authorization rejects any query referencing potentially-reidentifying join columns.

**Why agentic here when not for the others.** Multi-hop is the median case, not the edge case — most analytics questions require retrieving multiple schema pieces and reasoning about joins. The latency budget is much more permissive (analysts will wait 30 seconds for a useful result). The cost budget per query is higher because each query has more decision value.

---

## 6. Anti-patterns

The six RAG architecture mistakes I see most often, and the corrective pattern for each.

### Anti-pattern 1: "Build the full stack on day one"

The team reads the reference architectures online (hybrid + rerank + query-rewrite + agentic + graph) and builds all of it for the first feature. Result: an operational surface that the team cannot keep running, with quality wins they cannot attribute to any specific component, and a cost line that is 4x the necessary baseline.

**Corrective pattern.** Start with Naive. Measure. Graduate on signal. Each graduation step should be a separate landed change with its own before/after eval.

### Anti-pattern 2: "Reranker by default"

The reranker is added "for safety" without measuring whether it lifts quality on the actual eval. It adds 50–500ms latency and $0.001–$0.01 cost per query, and on uniform corpora it often does not earn either.

**Corrective pattern.** Add the reranker only when eval shows recall@50 is high but precision@5 is low. If that gap does not exist on your workload, the reranker is not earning its cost.

### Anti-pattern 3: "Agent for everything"

Every retrieval becomes an agent loop because "the agent can decide whether to retrieve again." The single-question workload that costs $0.01 with naive RAG now costs $0.15 with the agent loop, takes 8 seconds instead of 2, and has worse quality on the simple cases because the agent over-thinks them.

**Corrective pattern.** Workflow-shaped problems get workflows. Agentic-shaped problems get agents. The architecture sibling's [agent-vs-workflow-decision.md](../reference-patterns/) (when published) explicitly draws this line.

### Anti-pattern 4: "Graph because the data is relational"

The corpus has documents that reference each other, so the team builds a knowledge graph. Graph construction and maintenance now occupies a meaningful slice of the team's capacity, the graph drifts as the corpus drifts, and the quality lift over hybrid retrieval is marginal because the *queries* are not actually graph-shaped — they are document-shaped.

**Corrective pattern.** Build the graph only when the queries are relationship-shaped *and* the relationship answer cannot be found by retrieving and reading the right document. Targeted, narrow graphs (the drug-interaction graph in the Meridian example) beat sprawling corpus-wide graphs.

### Anti-pattern 5: "Chunking on autopilot"

The team uses the default chunking strategy from their framework (often fixed-size, 1000 tokens, no awareness of document structure). On documents with tables, lists, code blocks, or clinical structure, default chunking splits semantic units mid-content, producing chunks that are not individually retrievable and not coherently embeddable.

**Corrective pattern.** Chunking is a workload-specific eval problem. Try at least two strategies; pick the one that lifts retrieval recall on your eval. Document-structure-aware chunking (split on section boundaries, preserve table integrity, preserve code-block integrity) almost always beats fixed-size on real document corpora.

### Anti-pattern 6: "Retrieve more chunks if quality is bad"

When retrieval quality is poor, the team raises top-K from 5 to 10 to 20 to 50. This dilutes the prompt with irrelevant context, raises cost, raises latency, and often *lowers* answer quality because the LLM gets distracted by the low-relevance chunks.

**Corrective pattern.** Diagnose the cause. Low retrieval recall is not solved by retrieving more chunks; it is solved by improving the retriever (hybrid, reranker, query rewriting). Low precision is not solved by retrieving more chunks; it is solved by reranking. Retrieving more is rarely the right answer past the original top-5 to top-10.

---

## 7. Findings (sprint-assignable)

These are the canonical RAG-architecture findings I write into review documents. Each has an ID (`ARCH-RAG-NNN`), severity, finding, recommendation, and a sprint owner template (replace with the actual team / sprint when adopting).

### ARCH-RAG-001 — Severity: High
**Finding.** RAG pattern selection was made before the workload's retrieval-recall ceiling was measured.
**Recommendation.** Build a 30–100 case golden set; measure naive RAG recall; only graduate to more complex patterns on signal.
**Owner template.** ai-platform-eng, sprint N+1.

### ARCH-RAG-002 — Severity: High
**Finding.** Production retrieval is vector-only over a corpus that contains named entities, codes, or jargon.
**Recommendation.** Adopt hybrid (BM25 + vector) with RRF merging; re-measure recall.
**Owner template.** ai-platform-eng, sprint N+1.

### ARCH-RAG-003 — Severity: Medium
**Finding.** Reranker is deployed without measurement that it lifts precision over the underlying retrieval.
**Recommendation.** Add the reranker-pass eval; remove the reranker if no lift is measured.
**Owner template.** ai-platform-eng, sprint N+2.

### ARCH-RAG-004 — Severity: High
**Finding.** Conversational follow-up queries are embedded and retrieved verbatim without query rewriting.
**Recommendation.** Add a Haiku-tier query-rewrite step for second-turn-and-later queries; eval on a conversational subset.
**Owner template.** ai-platform-eng, sprint N+1.

### ARCH-RAG-005 — Severity: Medium
**Finding.** Chunking uses framework defaults without workload-specific evaluation.
**Recommendation.** Compare fixed-size vs document-structure-aware chunking on the golden set; adopt the higher-recall strategy.
**Owner template.** ai-platform-eng, sprint N+2.

### ARCH-RAG-006 — Severity: High
**Finding.** Agentic RAG is in use for query classes that single-step retrieval handles equivalently.
**Recommendation.** Route the simple-class queries to single-step retrieval; reserve agentic flow for the multi-hop class identified by eval.
**Owner template.** ai-platform-eng, sprint N+3.

### ARCH-RAG-007 — Severity: High
**Finding.** Knowledge graph in use has unclear ownership and an undocumented refresh process.
**Recommendation.** Assign a maintaining team; document the refresh process; measure graph-traversal lift over hybrid retrieval and remove the graph if no lift.
**Owner template.** ai-platform-eng + data-platform, sprint N+2.

### ARCH-RAG-008 — Severity: High
**Finding.** Multi-tenant workload uses a shared vector index without mandatory metadata filter.
**Recommendation.** Implement per-tenant namespace or mandatory metadata filter, enforced inside the retrieval client wrapper (not in calling code). Add the test pattern that catches missing-filter bugs.
**Owner template.** ai-platform-eng, sprint N+1.
**Cross-link.** Architecture: `multi-tenancy-and-isolation/per-tenant-vector-namespacing.md` (coming).

### ARCH-RAG-009 — Severity: Medium
**Finding.** Retrieval results are not instrumented (no trace of what was retrieved, scores, what was used).
**Recommendation.** Add retrieval-span instrumentation per the engineering sibling's `observability-and-telemetry/retrieval-instrumentation.md`.
**Owner template.** ai-platform-eng, sprint N+2.

### ARCH-RAG-010 — Severity: Medium
**Finding.** Embedding model version is not pinned; SDK auto-upgrade has caused silent re-embedding incidents.
**Recommendation.** Pin embedding model version in code and in deployment config; treat embedding-model changes as breaking changes that require full re-embedding with eval cross-check.
**Owner template.** ai-platform-eng, sprint N+1.

### ARCH-RAG-011 — Severity: High
**Finding.** Top-K was raised from 5 to 20 in response to quality issues, without diagnostic measurement.
**Recommendation.** Roll back to top-5–10; measure retrieval recall and precision; address the underlying retrieval problem (hybrid, reranker, query rewrite) rather than diluting the prompt.
**Owner template.** ai-platform-eng, sprint N+1.

### ARCH-RAG-012 — Severity: Medium
**Finding.** Retrieval corpus has no documented refresh SLO or owner team.
**Recommendation.** Establish corpus ownership; document per-source freshness SLO; add freshness monitoring.
**Owner template.** ai-platform-eng + corpus-owning team, sprint N+2.
**Cross-link.** Architecture: `data-architecture-for-ai/data-contracts-for-retrieval.md` (coming).

### ARCH-RAG-013 — Severity: High
**Finding.** No eval distinguishes retrieval failures from generation failures, so diagnosis of quality issues is guesswork.
**Recommendation.** Split eval into retrieval-stage (recall, precision) and generation-stage (correctness given correct context); diagnose accordingly.
**Owner template.** ai-platform-eng, sprint N+1.
**Cross-link.** Engineering: `eval-engineering/eval-of-rag.md` (coming).

### ARCH-RAG-014 — Severity: Medium
**Finding.** Long-context RAG is in use for an interactive workload where the cost / latency profile is incompatible with the user-facing SLO.
**Recommendation.** Move the workload to hybrid retrieval; reserve long-context for async / batch workflows where its trade-offs are acceptable.
**Owner template.** ai-platform-eng, sprint N+3.

### ARCH-RAG-015 — Severity: High
**Finding.** Citation / source attribution is not tracked from retrieval to answer, so user-facing claims cannot be traced to their source documents.
**Recommendation.** Implement lineage tracking (doc → chunk → embedding → retrieval → answer); surface citations to users where the workload allows it.
**Owner template.** ai-platform-eng, sprint N+2.
**Cross-link.** Architecture: `data-architecture-for-ai/lineage-and-provenance.md` (coming).

---

## 8. Adoption sequencing checklist

For a team starting from "we want to add RAG to feature X":

- [ ] **Sprint 0 — define.** Answer the five questions in section 2. Write the answers down; circulate to stakeholders for agreement. Most over-engineering is prevented at this step.
- [ ] **Sprint 1 — golden set.** Build a 30–100 case golden set with expected answers and expected source documents. Get clinical / domain reviewers to sign off on the cases.
- [ ] **Sprint 1 — naive RAG.** Ship the naive-RAG pattern. Measure retrieval recall and answer correctness on the golden set.
- [ ] **Sprint 2 — graduate if signal.** If recall < 75% or precision < 70%, graduate to hybrid. Re-measure.
- [ ] **Sprint 3 — graduate if signal.** If precision@5 is still low and recall@50 is high, add reranker. Re-measure.
- [ ] **Sprint 3 — instrumentation.** Add retrieval observability (sibling `observability-and-telemetry/retrieval-instrumentation.md` patterns) before launch. Without it, post-launch quality regressions are not diagnosable.
- [ ] **Pre-launch — eval gate.** Add the eval suite to CI per sibling `cicd-and-eval-gates/eval-gate-design.md`. Block deploys on regression.
- [ ] **Post-launch — measure and graduate.** Online evals (judge-pass-rate, retry-rate, citation-accuracy) on production traffic. Graduate further only on signal from these metrics.

The sequencing is biased toward *shipping the simplest pattern first* and graduating on measured signal. Teams that follow this sequence routinely arrive at a simpler architecture than teams that start from the "full reference" pattern, with equivalent or better quality and dramatically lower operational cost.

---

## 9. References

- Karpukhin et al., *Dense Passage Retrieval for Open-Domain Question Answering* (2020) — foundational vector retrieval reference.
- Lin et al., *Pyserini: A Python Toolkit for Reproducible Information Retrieval Research with Sparse and Dense Representations* (2021) — hybrid retrieval baselines.
- Cormack et al., *Reciprocal Rank Fusion outperforms Condorcet and Individual Rank Learning Methods* (2009) — RRF reference for hybrid merging.
- Lewis et al., *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* (2020) — RAG canonical reference.
- Anthropic, Contextual Retrieval (2024) — practical contextual-chunking pattern.
- Vendor documentation for Cohere Rerank, Pinecone, Weaviate, pgvector, OpenSearch, Anthropic Claude (cited inline where relevant).
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/rag-engineering) — the engineering practice for the patterns chosen here.
- Sibling repo: [ai-security-reference-architecture/llm-application-security/](https://github.com/jeremiahredden/ai-security-reference-architecture/tree/main/llm-application-security) — security implications of retrieval (prompt-injection via poisoned documents, retrieval-as-side-channel).
