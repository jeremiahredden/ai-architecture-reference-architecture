# Context and Prompt Architecture

## What this folder is

The architectural decisions about what goes into the model's context window, how it gets there, and how it evolves. The material here is what I put in front of a team when the question is: *the prompt has grown to 4,000 tokens of system instructions, the chat history is 8,000 more, retrieval adds another 6,000, and the bill / latency / quality are all suffering — what is the actual context architecture?*

## The organizing principle

The context window is a shared, finite, expensive resource that every AI feature competes for. Most production AI systems treat it as an afterthought — system prompt gets bigger over time as edge cases are patched, retrieval pulls more chunks "to be safe," chat history accumulates without summarization, tool schemas are dumped wholesale. The result is a system that costs more every quarter, gets slower every quarter, and quality-regresses in ways that are hard to attribute because the prompt has become an undocumented monolith.

So the patterns here treat the context window as a *budgeted resource with sub-allocations* — system, instructions, retrieval, history, tools, and answer — each with an owner, a budget, and a strategy for what to do when the budget is exceeded (truncate, summarize, defer, escalate to a larger-context model). And they treat the prompt itself as a *versioned API*, not a string, so it can evolve without silently breaking the consumers (chains, evals, downstream extractions) that depend on it.

## Planned documents

- **[system-prompt-architecture.md](./system-prompt-architecture.md)** — The system prompt as architecture: identity, behavior, refusal policy, formatting contract, tool-use policy. The decomposition pattern that lets a single platform-level system prompt host multiple feature-level overlays. The "prompts as components" model and the testing pattern that catches regressions when one overlay changes.
- **[context-window-budgeting.md](./context-window-budgeting.md)** — Per-allocation budgets (system / instructions / retrieval / history / tools / answer), what to do when an allocation overflows (priority-based truncation, summarization, tiered-context routing), context-budget-as-SLO discipline, and the patterns for systems where context is genuinely unbounded (long-conversation agents, repository-aware coding agents).
- **[long-context-vs-rag.md](./long-context-vs-rag.md)** — The decision framework for when long-context (single large prompt with everything stuffed in) beats RAG, when RAG beats long-context, and when the right answer is RAG-into-long-context. Calibrated for current frontier models (1M-context) with the workload signals that determine which is right.
- **[prompt-assembly-patterns.md](./prompt-assembly-patterns.md)** — Templating (Jinja-style), structured composition (function-built prompts), retrieval-into-prompt, prompt-as-DSL, and the patterns for prompts that have to vary by tenant, user, locale, or feature flag. Includes the "do not concatenate strings at request time" anti-pattern.
- **[prompt-as-api-discipline.md](./prompt-as-api-discipline.md)** — The discipline of treating prompts as versioned APIs: pinning versions in releases, deprecation lifecycles, breaking-change communication, the prompt-changelog pattern, and the integration with the engineering sibling's prompt-engineering folder. Backwards-compatible prompt evolution patterns.
- **[chat-history-architecture.md](./chat-history-architecture.md)** — Short-term vs long-term memory at the architectural level. Verbatim history (cheap, fills context), running summary (fixed size, loses detail), tool-and-fact extraction (structured, loses voice), hybrid. The retention policy as an architectural choice that determines what the system can remember about a user across sessions.
- **[few-shot-vs-fine-tune-vs-system-prompt.md](./few-shot-vs-fine-tune-vs-system-prompt.md)** — When each is right: few-shot examples (flexible, expensive per call), fine-tune (cheap per call, slow to update), system prompt (cheap, hits ceiling). The decision tree and the migration paths between them.

## How to use this section

**If you are designing a new AI feature**, read `system-prompt-architecture.md` and `context-window-budgeting.md` first. The system prompt and the context budget are the two decisions that are most expensive to undo later — every feature built on top inherits them.

**If you have a system with a runaway system prompt** (4,000+ tokens of accumulated edge-case patches), `system-prompt-architecture.md` and `prompt-as-api-discipline.md` together describe the refactoring path. The pattern is almost always "decompose into platform-base + feature-overlays, version both."

**If you are weighing whether to use a 1M-context model or RAG**, `long-context-vs-rag.md` is the explicit decision tree. The 2026 answer is usually still RAG for cost and latency reasons; the document names the cases where long-context wins.

## What this section is not

- **A prompt-engineering tutorial.** How to write a good prompt (chain-of-thought, role priming, few-shot example design) is the engineering sibling's `prompt-engineering/` folder. This folder is about the *architectural* choices that frame those prompts.
- **A specific-model best-practices guide.** Each model vendor publishes those (Anthropic's prompt engineering guide, OpenAI's, Google's). They are vendor-specific and they update. The patterns here are about the architecture under those guides.
