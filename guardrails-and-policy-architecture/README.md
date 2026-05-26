# Guardrails and Policy Architecture

## What this folder is

The architectural decisions about *where guardrails live* in an AI system, not how to write them and not how to operate them. The material here is what I put in front of an architecture team when the question is: *we know we need guardrails — input filters, output validation, tool-call authorization, retrieval scope enforcement — but where in the architecture should each one actually sit so it stays in place when the system evolves?*

This is the build-side companion to the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture). That repo addresses the threat model, the attack patterns, the detection content, and the response runbooks. This folder addresses only the architectural placement decision: *given that a control needs to exist, where should it sit so it cannot be silently removed by a refactor, and so it composes with the rest of the system?*

## The organizing principle

Most AI guardrails fail in production for the same reason: they are bolted onto the side of the system rather than placed inside it. The classic failure mode is the moderation API call that wraps the LLM call in the chat handler — and is silently skipped by the new batch-processing handler that the team built three months later. The control existed; it just was not where the new code paths went through.

So the patterns here treat guardrail placement as a *system-design problem*. The two architectural questions are *(1) what is the smallest set of choke points that all AI invocations must go through?* and *(2) which control belongs at which choke point?* The answer is almost always a small platform-level abstraction (an "AI gateway" or "model client wrapper") that every invocation goes through, with the controls placed inside it rather than around it. That abstraction is the surface where the security sibling's detection content, the engineering sibling's observability content, and this repo's design patterns all attach.

The folder is opinionated about three things specifically. First, that input filtering and output validation belong *inside* the model-client abstraction, not in calling code. Second, that tool-call authorization belongs in the *tool registry* (centralized, audited, declaratively configured per tool), not in tool-implementation code. Third, that retrieval scope enforcement (the "only see this tenant's documents" guarantee) belongs in the *retrieval client abstraction*, not in the prompt assembly logic.

## Planned documents

- **[guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md)** — The architectural map: input guardrails (PII detection, prompt-injection screening, abuse / off-topic detection), output guardrails (PII leak prevention, format validation, claim verification), intermediate-step guardrails (agent action approval, tool-call authorization), and retrieval guardrails (scope enforcement, freshness checks, source allow-listing). Decision framework for which goes where, and the layered-defense pattern that combines them.
- **[ai-gateway-pattern.md](./ai-gateway-pattern.md)** — The architectural pattern of fronting all model invocations with a thin gateway that handles authentication, per-tenant routing, rate limiting, logging, cost accounting, and input / output filtering. Build-vs-buy (Anthropic / OpenAI gateways, AWS Bedrock Guardrails, Azure Content Safety, third-party LLM gateways like Portkey or Helicone, custom). The internal-API contract that survives model swaps.
- **[tool-call-authorization.md](./tool-call-authorization.md)** — The pattern for tool authorization as a first-class concern: which user / agent / tenant can call which tool with which arguments, evaluated centrally, logged centrally, and surfaced to the agent loop as a polite refusal rather than an error. The integration with the agent-engineering sibling content and with the security sibling's agent-permission-model.
- **[retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md)** — The architecture that makes "this query can only see documents in tenant X's namespace" an enforced invariant rather than a documented convention. Per-tenant retrieval client wrappers, query rewriting that injects mandatory filters, audit trails that prove enforcement, and the test pattern that catches scope failures before they become incidents.
- **[policy-as-code-for-ai.md](./policy-as-code-for-ai.md)** — Encoding AI policies as code: which prompts are allowed, which tools are allowed for which features, which models are approved for which data classifications, which output classes trigger human review. OPA / Cedar / custom evaluation, the policy-bundle pattern, and the deployment integration that keeps policy current with code.
- **[content-moderation-architecture.md](./content-moderation-architecture.md)** — Where moderation sits: pre-model (filter inputs), post-model (filter outputs), inline (constrained decoding), human-in-the-loop (high-risk classes route to review). Vendor moderation (OpenAI Moderation, Azure Content Safety, Bedrock Guardrails, Perspective API) vs custom classifiers vs LLM-as-moderator. The performance and cost implications.
- **[refusal-and-escalation-design.md](./refusal-and-escalation-design.md)** — The architectural design of refusal and escalation paths: when the model should refuse, what the refusal should look like (canonical message, escalation hint, helpful-but-bounded alternative), how refusals are observable for product feedback, and the escalation paths to human-in-the-loop or to a more capable model. Refusal-as-feature, not refusal-as-friction.

## How to use this section

**If you are starting a new AI platform**, the `ai-gateway-pattern.md` document is the highest-leverage early decision. Without a gateway, controls bolt onto calling code and decay; with a gateway, controls have a home and survive refactors.

**If you have an AI system in production with inconsistent guardrails** (some features have moderation, some do not; some have rate limits, some do not), `guardrail-placement-decision-framework.md` is the diagnostic and the refactor target.

**If you are shipping agents** (multi-step, tool-using), `tool-call-authorization.md` and `retrieval-scope-enforcement.md` are non-negotiable. The agent failure modes that show up in production are overwhelmingly authorization failures and scope failures, not capability failures.

## What this section is not

- **An AI-security playbook.** Threat modeling, attack patterns, red-teaming methodology, detection content, and incident response live in the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture). This folder addresses only architectural placement.
- **An AI-safety treatise.** The full taxonomy of harms, the alignment debate, and the policy and governance superstructure are out of scope.
- **A vendor moderation comparison.** Where moderation vendors are named, they are illustrative; the architectural placement question is independent of vendor choice.
