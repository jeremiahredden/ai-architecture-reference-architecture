# Reference Patterns

## What this folder is

The pattern catalogue for AI system design. The material here is the vocabulary I use in front of a whiteboard when the question is: *we know we want to add an AI feature here, what shape should the AI part actually take?* It is opinionated about which patterns are worth committing to in production, and which are research curiosities or vendor demos.

## The organizing principle

Most AI architecture mistakes come from picking a pattern that the actual problem does not require — most often, picking "agent" when a single LLM call would do, or picking "graph RAG" when hybrid search would do. Complexity in AI systems compounds nonlinearly. Every additional component is more eval surface, more failure modes, more cost, more latency, and more on-call exposure. So the patterns here are arranged in roughly-increasing order of operational cost, and every pattern names the simpler version it should be evaluated against before adoption.

The folder treats the pattern selection itself as the architectural decision. The implementation of a chosen pattern is the engineering sibling's territory; this folder's job is to make the decision visible — what each pattern actually requires, what failure modes come with it, and what production signals justify graduating from the simpler version to the more complex one.

## Planned documents

- **[rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md)** — Decision framework for RAG variants: naive single-source RAG, hybrid (BM25 + vector), reranked, query-rewriting, multi-step / agentic RAG, graph-augmented RAG, and retrieval-free long-context. Each variant with its sweet-spot use case, the failure modes that justify graduating to it, the eval surface required to validate it, and worked Meridian Health examples. Includes the *"start with hybrid, graduate only on signal"* adoption playbook, the RAG-anti-patterns checklist, and sprint-assignable ARCH-RAG findings.
- **[agent-topologies.md](./agent-topologies.md)** — Single-agent loop, supervisor / worker, planner / executor, swarm, hierarchical agents, and the agent-vs-workflow decision. When an agent is the right shape and when a deterministic workflow (with optional LLM steps) is the right shape — the most consequential decision in 2026 agent architecture.
- **[multi-model-orchestration.md](./multi-model-orchestration.md)** — Router patterns (rules-based, LLM-as-router, classifier-as-router), tier routing (cheap-first then escalate, capability-first then specialize), ensemble patterns, and the model-portfolio architecture that makes a system resilient to single-model degradation or deprecation.
- **[structured-output-patterns.md](./structured-output-patterns.md)** — JSON Schema with tool-calling, constrained decoding, output validation + repair loops, structured-output failure modes (truncation, schema drift, hallucinated fields), and the architectural choice between strict schemas and freeform-with-extraction.
- **[hybrid-retrieval-patterns.md](./hybrid-retrieval-patterns.md)** — Hybrid lexical + semantic search at the architectural level: when to combine BM25 and vector vs use one alone, multi-stage retrieval (recall → rerank → context-fit), metadata filtering as first-class, and the index-shape decisions that determine retrieval cost and quality.
- **[pattern-anti-patterns.md](./pattern-anti-patterns.md)** — The six patterns I see chosen wrongly most often: "agent for everything," "graph RAG because the data is graph-shaped," "fine-tune as the first move," "router as the first move," "every step is an LLM call," and "monolithic prompt as architecture." Each with the failure mode and the corrective pattern.

## How to use this section

**If you are scoping a new AI feature**, read [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) first if the feature is retrieval-shaped, or `agent-topologies.md` if it is action-taking-shaped. The decision frameworks are calibrated to *fastest pattern that meets the requirements*, not *most-capable pattern that could meet them*.

**If you are inheriting an AI system you did not design**, `pattern-anti-patterns.md` is the diagnostic tool. Most inherited systems show one or two of the documented anti-patterns; the document includes the corrective sequencing for each.

**If you are deciding between an LLM-shaped solution and a non-LLM solution**, the agent-topologies and structured-output documents both have explicit "do not use an LLM here" sections. Half of good AI architecture is choosing where *not* to put the AI.

## What this section is not

- **A survey of every RAG / agent paper.** The research literature ships faster than any reference can track. Where a paper genuinely shifts the practice, it is cited; otherwise, this folder stays focused on patterns that have been shipped in production by enough teams to have documented failure modes.
- **A framework comparison.** LangChain vs LlamaIndex vs Vercel AI SDK vs raw SDK is a useful conversation but it is not an architecture conversation. The patterns here are framework-agnostic; the engineering sibling repo's `agent-engineering/` and `rag-engineering/` folders address framework choice where it matters.
