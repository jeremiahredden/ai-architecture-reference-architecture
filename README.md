# AI Architecture Reference Architecture

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Maintained](https://img.shields.io/badge/Maintained-yes-green.svg)
![Sibling repo: ai-engineering-reference-architecture](https://img.shields.io/badge/sibling-ai--engineering--reference--architecture-orange.svg)
![Sibling repo: ai-security-reference-architecture](https://img.shields.io/badge/sibling-ai--security--reference--architecture-orange.svg)
![Sibling repo: appsec-reference-architecture](https://img.shields.io/badge/sibling-appsec--reference--architecture-orange.svg)
![Sibling repo: cloud-security-reference-architecture](https://img.shields.io/badge/sibling-cloud--security--reference--architecture-orange.svg)

### A practitioner's toolkit for designing AI systems — RAG and agent patterns, model strategy, data and integration architecture, cost and performance, multi-tenancy, designed-in guardrails, and worked reference systems for regulated workloads

**Jeremiah Redden** | Senior AI/AppSec Security Architect | CISSP | [github.com/jeremiahredden](https://github.com/jeremiahredden)

---

This repository is the *design* side of the AI build problem. Its closest sibling, [**ai-engineering-reference-architecture**](https://github.com/jeremiahredden/ai-engineering-reference-architecture), is the *shipping and operating* side — eval engineering, agent engineering, RAG engineering, model lifecycle, observability, reliability, CI/CD, and FinOps. The third closely-related sibling, [**ai-security-reference-architecture**](https://github.com/jeremiahredden/ai-security-reference-architecture), covers how to attack and defend the systems described here. All three are designed to be read together. Where this repo says *"pattern X has these moving parts and these non-functional properties,"* the engineering sibling says *"and here is how you ship and operate it,"* and the security sibling says *"and here is the threat model and the controls."*

The split is deliberate. AI architecture is its own discipline now — distinct from data architecture, distinct from cloud architecture, and emphatically distinct from "we'll bolt an LLM into the existing system and call it AI." The decisions that matter most for a real AI workload — which retrieval pattern to commit to, how to tier models for cost and latency, where to put guardrails so they survive a refactor, how to isolate tenants in a system whose primary state lives in opaque embeddings — are not addressed by the cloud architecture canon, the data architecture canon, or the application architecture canon. They need their own reference set.

A second motivation: most public AI architecture material is either marketing-grade ("here is a reference architecture diagram with 14 boxes and three arrows") or research-grade ("here is a 47-page paper on retrieval augmentation"). Neither helps a platform engineer who has been told "ship a clinical agent for the patient-intake workflow, in this regulated environment, with this budget, in this quarter." This repo is calibrated to that practitioner — opinionated about which patterns matter, explicit about non-functional trade-offs, and biased toward designs that can be landed incrementally on top of an existing stack rather than as a green-field build.

---

## Table of Contents

- **[reference-patterns/](./reference-patterns/)** — The pattern catalogue. RAG variants (naive, hybrid, agentic, graph-augmented, query-rewrite), agent topologies (single-agent loop, supervisor/worker, swarm, planner-executor), multi-model orchestration (router patterns, tier-routing, ensemble), and the decision frameworks for picking between them. The first deep document, [rag-architecture-decision-guide.md](./reference-patterns/rag-architecture-decision-guide.md), is landed.
- **[model-strategy/](./model-strategy/)** *(coming)* — Frontier-vs-open-weights decision frameworks, model routing and tiering, model catalogue design, build-vs-fine-tune-vs-RAG decision tree, capability vs cost vs latency trade-offs, and the migration patterns for moving between models as new generations land (which, in 2026, is roughly every quarter).
- **[context-and-prompt-architecture/](./context-and-prompt-architecture/)** *(coming)* — System-prompt architecture, prompt assembly patterns (templating vs structured composition vs retrieval-into-prompt), context-window budgeting across system / instructions / retrieval / history / tools, the long-context vs RAG decision, and the prompt-as-API design discipline that lets prompts evolve without breaking downstream consumers.
- **[data-architecture-for-ai/](./data-architecture-for-ai/)** *(coming)* — Vector stores, knowledge graphs, hybrid stores, feature stores for AI, embedding strategy (model selection, dimensions, normalization, versioning), data contracts for retrieval sources, lineage and provenance tracking, and the freshness-vs-staleness architectural choices that determine how often the system tells the user something wrong.
- **[integration-architecture/](./integration-architecture/)** *(coming)* — Sync vs async vs streaming response patterns, event-driven AI integration, tool-call architecture (in-process vs MCP vs HTTP), human-in-the-loop boundary design, backpressure and queueing, and the integration shapes that let an AI feature land inside an existing application without rewriting the application.
- **[cost-and-performance-architecture/](./cost-and-performance-architecture/)** *(coming)* — Token economics at the architectural level, throughput planning, GPU strategy for self-hosted inference, caching tiers (semantic cache, embedding cache, response cache), latency budgets, the streaming-vs-batched-vs-async decision, and the cost-control patterns that prevent a successful AI feature from generating a six-figure surprise invoice.
- **[multi-tenancy-and-isolation/](./multi-tenancy-and-isolation/)** *(coming)* — Tenant isolation patterns for AI systems where most of the state lives in opaque embeddings, retrieved documents, and conversation histories. Per-tenant namespacing in vector stores, per-tenant fine-tuning vs shared base models, cross-tenant data leakage prevention at the retrieval layer, noisy-neighbor mitigation, and the data-residency patterns that B2B SaaS AI features have to honor.
- **[guardrails-and-policy-architecture/](./guardrails-and-policy-architecture/)** *(coming)* — Where guardrails belong architecturally — input filters, output filters, intermediate-step validation, tool-call authorization, retrieval scope enforcement, and the placement decisions that determine whether guardrails survive a refactor or are routed around. Build-side framing only; the operational side (detection, response, red-teaming) lives in [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture).
- **[reference-systems/](./reference-systems/)** — End-to-end worked reference architectures. The first deep document, [meridian-care-coordinator.md](./reference-systems/meridian-care-coordinator.md), is the full architecture for a clinical agent system in a regulated healthcare SaaS — the showcase that ties the pattern catalogue, model strategy, data architecture, and guardrail placement into one coherent system. Additional reference systems (patient-API AI assist, de-identified analytics warehouse copilot) are planned.

---

## How to Use This Repo

**If you are an AI architect or staff engineer** designing a new AI feature or platform, start with [reference-patterns/](./reference-patterns/) and [reference-systems/](./reference-systems/). The pattern catalogue is the vocabulary; the reference systems are how the vocabulary actually composes into a working architecture. Most architecture mistakes I see come from choosing a pattern (often "agent") that the actual problem does not require — the reference patterns catalogue is built to make those decisions visible rather than implicit.

**If you are a platform engineer** building the platform that other product teams will build AI features on, [model-strategy/](./model-strategy/), [data-architecture-for-ai/](./data-architecture-for-ai/), and [multi-tenancy-and-isolation/](./multi-tenancy-and-isolation/) are the three platform-shaped decisions you cannot push down to product teams. Get those right and product teams have room to iterate; get them wrong and every product team will end up reimplementing them inconsistently.

**If you are a product engineer** building one AI feature inside a larger application, [integration-architecture/](./integration-architecture/) and [context-and-prompt-architecture/](./context-and-prompt-architecture/) are the two most-leveraged sections. The integration shape determines whether your feature is a bolt-on or a citizen of the application; the context architecture determines whether the prompt is a string you maintain forever or an API you can evolve.

**If you are an engineering leader** trying to scope an AI initiative, [cost-and-performance-architecture/](./cost-and-performance-architecture/) is where the budget-relevant decisions live, and [reference-systems/](./reference-systems/) is where you can see one end-to-end. The worked Meridian Care Coordinator example is sized to a 350-engineer regulated SaaS — close enough to most enterprise AI scopes to be a useful baseline.

**If you are responsible for AI safety or governance**, [guardrails-and-policy-architecture/](./guardrails-and-policy-architecture/) is the build-side companion to the [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) sibling. The placement decisions here determine whether the controls described in the security repo have a real surface to attach to.

**If you are a hiring manager** evaluating my work, the worked reference architecture in [reference-systems/meridian-care-coordinator.md](./reference-systems/meridian-care-coordinator.md) and the decision framework in [reference-patterns/rag-architecture-decision-guide.md](./reference-patterns/rag-architecture-decision-guide.md) are closer to my deliverables than a resume will ever be. Read those first.

---

## Philosophy

Four principles drive every piece of content in this repository. They are the same four that govern the sibling repos, restated for AI architecture.

**1. AI architecture should make product and platform teams faster, not slower.** A reference architecture that requires six new services, two new vendor contracts, and a six-month platform migration before the first feature ships is not an architecture — it is a procurement plan in disguise. The patterns here are designed for incremental adoption. The expectation is that a team can pick one pattern, land it for one feature, and ship it inside a sprint; the platform consolidations come later, after the product value is proven. I optimize for designs that compose with what the team already has, even when a green-field design would be cleaner.

**2. The right AI pattern is the one that ships this sprint.** The single biggest failure mode in AI architecture in 2026 is over-engineering — picking the agent topology when a single LLM call would do, picking the graph-augmented RAG when hybrid search would do, picking the fine-tune when prompt engineering would do. Complexity in AI systems compounds nonlinearly: every additional component is more eval surface, more failure modes, more cost, and more latency. Every pattern in this repo names the simpler version that should be tried first and the signals that justify graduating to the more complex one. The 80% degraded-mode pattern that ships next week beats the gold-standard architecture that ships next quarter.

**3. Every AI design decision needs a concrete next step with an owner.** A reference architecture that ends with "consider implementing RAG" has failed. One that ends with "use hybrid BM25+vector retrieval over the chunked clinical-guideline corpus, embed with `text-embedding-3-large` at 1536 dimensions, rerank top-50 to top-10 with `cohere-rerank-3.5`, store in pgvector with per-tenant namespace partitioning, owner: ai-platform-eng, tracked as ARCH-014, by sprint 47" has succeeded. Every worked example here is written to that standard — concrete components, concrete sizing, concrete owners, concrete tracking IDs. Decisions without owners decay into tribal knowledge and then into rewrites.

**4. An AI architecture that only exists in a slide deck doesn't exist. Show your work.** Architecture diagrams, capability matrices, and vendor-comparison tables are evidence of thought, not of system behavior. AI systems in particular have a way of failing in ways the architecture diagram does not predict — the retrieval pattern looks fine in the diagram and is wrong in production because the chunking strategy interacts badly with the actual document distribution. Where the patterns here can be turned into runnable artifacts — a sample retrieval pipeline, an eval suite that verifies the pattern, a concrete prompt and tool schema — they are. Every architectural claim should be backed by an artifact a reviewer can execute. The sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture) repo is where those artifacts live in depth.

---

## What this repo is not

- **A vendor walkthrough.** Where specific tools are named (Claude, GPT, Gemini, pgvector, Pinecone, Weaviate, Cohere rerankers, MCP servers, LangChain, LlamaIndex, Vercel AI SDK), it is because the pattern is easier to explain concretely. Substitute the vendor of your choice; the pattern is what matters. Vendor lifetimes are short — a 2026-vintage reference architecture that names one vendor as the answer will be obsolete by 2027.
- **A research paper survey.** The literature on retrieval, agents, and reasoning has more good papers per quarter than any practitioner can read. Where it matters, this repo names the technique and points at the canonical reference; it does not reproduce the literature.
- **An AI safety treatise.** The placement of guardrails is in scope. The full taxonomy of harms, the alignment debate, and the policy and governance superstructure are out of scope — they need their own treatment and they are not what a platform engineer building a clinical agent needs to read at 3pm on a Tuesday.
- **AI engineering or AI security.** The shipping-and-operating side lives in [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture); the attack-and-defend side lives in [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture). Use all three together; the boundaries are deliberate.

---

## License & Attribution

All content in this repository is released under the MIT License unless otherwise noted. Templates, reference architectures, decision frameworks, and worked examples may be used, modified, and adapted freely — including in commercial engagements and client deliverables. Attribution is appreciated but not required.

If you find something useful, I would like to hear about it. Open an issue, reach out on LinkedIn, or send a pull request with your improvements.
