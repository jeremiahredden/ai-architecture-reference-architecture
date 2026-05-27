# Model Strategy

## What this folder is

The architectural decisions about *which models* a system commits to, how it routes between them, and how it migrates between them as new generations land. The material here is what I put in front of an architecture review board when the question is: *we cannot just pick "Claude" or "GPT" and call it strategy — what is the actual model strategy for this platform?*

## The organizing principle

In 2026, model strategy is the single most consequential and the single most volatile architecture decision in an AI system. Consequential because model choice drives capability ceiling, cost floor, latency profile, vendor concentration risk, regulatory exposure (data residency, BAA coverage, FedRAMP boundary), and the prompt and eval surface every other team works against. Volatile because the model frontier moves quarterly — a model strategy that was correct in January is often suboptimal by July, and a system architecture that hard-codes one model anywhere downstream of the API boundary will be in a migration project within a year of shipping.

So the patterns here are biased toward *strategies that survive model churn*: model-as-dependency discipline (pin versions, list models as first-class dependencies in a manifest), abstraction at the right layer (not too thin, not too thick), tier routing (cheap first, escalate on signal), portfolio thinking (multiple models in production for resilience, not one), and migration playbooks rather than one-time selections.

## Planned documents

- **[frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md)** — The three-way decision framework: when to use a frontier hosted model (Claude, GPT, Gemini), when to use open weights (Llama, Mistral, Qwen, DeepSeek), when to fine-tune (and on which base), and when to do nothing yet. With cost / latency / capability / regulatory comparisons, decision tree, and worked Meridian Health examples (clinical agent uses frontier-hosted-with-BAA; bulk de-identification uses open weights on internal GPUs; classification uses a fine-tuned smaller model).
- **[model-routing-and-tiering.md](./model-routing-and-tiering.md)** — Router architecture: rule-based, classifier-based, LLM-as-router, and hybrid. Tier escalation patterns (try Haiku, fall back to Sonnet, escalate to Opus on uncertainty). The cost / quality / latency curves and where the inflection points actually are. The router-as-platform-component vs router-as-application-code decision.
- **[model-catalogue-and-registry.md](./model-catalogue-and-registry.md)** — The platform pattern of treating models as first-class catalogued dependencies with owners, version pins, allowed contexts, BAA coverage status, deprecation dates, and per-model usage telemetry. Catalogue schema, integration with the model lifecycle pipeline (sibling engineering repo), and the governance model that lets the platform say "yes" to model adoption without losing control.
- **[build-vs-buy-decision.md](./build-vs-buy-decision.md)** — When to use a hosted API, when to host open weights yourself, when to use a managed inference provider (Bedrock, Vertex, Azure OpenAI, Together, Fireworks), and the criteria — cost at scale, latency requirements, regulatory boundary, capability ceiling, custom-fine-tune needs — that should drive the choice. The hosted-with-data-residency-add-on pattern that solves most regulated workloads without going self-hosted.
- **[model-migration-playbook.md](./model-migration-playbook.md)** — The repeatable playbook for migrating from model A to model B without a quality regression: parallel-running shadow traffic, eval-suite cross-check, prompt-port discipline, fallback configuration, the rollout sequence, and the rollback criteria. Includes the worked example of migrating Meridian Care Coordinator from a previous generation Claude to current generation.
- **[capability-vs-cost-vs-latency-tradeoffs.md](./capability-vs-cost-vs-latency-tradeoffs.md)** — The non-functional comparison framework. Tokens-per-second, cost-per-million-tokens, context-window, structured-output reliability, tool-use reliability, multilingual coverage, and the regulatory characteristics. Calibrated for 2026 frontier and open-weight options.

## How to use this section

**If you are setting platform-level model policy**, read `frontier-vs-open-weights-vs-fine-tune.md` and `model-catalogue-and-registry.md` together. The first defines the strategy; the second defines the platform mechanism that enforces it.

**If you are designing a high-volume feature**, `model-routing-and-tiering.md` and `capability-vs-cost-vs-latency-tradeoffs.md` are the cost-relevant pair. Routing typically saves 40–70% of cost on a tiered workload; this is one of the highest-leverage architectural choices available.

**If you are facing a model deprecation deadline**, `model-migration-playbook.md` is the runbook. Treat model migration as a structured project with a parallel-run, an eval gate, and a rollback, not as an inline swap.

## What this section is not

- **A model benchmark report.** Benchmarks are useful and a poor proxy for production performance on a specific workload. The patterns here are about how to *evaluate models against your workload* (with help from the engineering sibling's `eval-engineering/` folder), not about which model scored highest on MMLU last week.
- **A vendor recommendation.** Vendor lifetimes are short. Where models are named, they are illustrative; the strategy patterns apply to whatever vendor mix is current.
