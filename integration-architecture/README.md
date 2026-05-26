# Integration Architecture

## What this folder is

The architectural decisions about how an AI feature attaches to the rest of a system. The material here is what I put in front of a platform team when the question is: *the AI feature itself works in the notebook — now how do we land it inside the actual application without rewriting the application?*

## The organizing principle

AI features have integration properties that conventional service integration does not address. They are slow (seconds-to-tens-of-seconds per call), expensive per call, occasionally fail in opaque ways, sometimes need to stream partial output to feel acceptable, sometimes need to wait for a human in the middle of a workflow, and frequently need to call back out to existing tools and data sources. None of these properties are fatal, but each one rules out a default integration shape — synchronous request/response over HTTP, the implicit pattern most application code reaches for first.

So the patterns here are arranged around the *integration shape decision*: synchronous, streamed, asynchronous-with-callback, event-driven, queued, polling, and human-in-the-loop. Each shape has different latency, reliability, observability, and UX properties; each is the right answer for a different class of feature. Choosing the wrong shape is the single most common AI integration mistake and is also the easiest to avoid if the choice is made explicitly rather than by default.

## Planned documents

- **sync-vs-async-vs-streaming.md** *(coming)* — The integration-shape decision tree. Synchronous when latency budget allows and feature is single-shot. Streaming when perceived-latency is the constraint and the UX can render incrementally. Asynchronous when the operation runs longer than 30 seconds, when retries are necessary, or when the caller does not need the result immediately. With worked examples for chat UI (streaming), bulk document classification (async-with-callback), and clinical-decision support (sync with strict latency budget).
- **tool-call-architecture.md** *(coming)* — Where tools should live: in-process functions, HTTP services, MCP servers, or hybrid. The trade-offs (latency, security boundary, language coupling, deployment independence), the tool-discovery pattern, and the tool-schema-as-contract discipline. Includes the MCP-adoption decision tree (when MCP is worth the operational cost, when local function-call is sufficient).
- **event-driven-ai-integration.md** *(coming)* — AI consumers and producers on an event bus. Patterns for AI-as-event-handler (document ingested → classify → annotate), AI-as-event-producer (agent finished task → emit completion event), and AI-step-in-a-saga. The retry, ordering, and idempotency disciplines that event-driven AI integration introduces, with worked examples on SNS/SQS, EventBridge, and Kafka.
- **[human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md)** — Where the human is in the workflow: approver before action (clinical agent suggests order, clinician approves), reviewer after action (agent drafts, human edits), escalation point on uncertainty (low-confidence cases route to human queue), or always-on copilot (human is the operator, AI is the assistant). The UX, latency, and operational implications of each, and the failure mode where the human becomes a rubber stamp.
- **backpressure-and-queueing.md** *(coming)* — Rate limits on the model provider side, queueing patterns that absorb upstream bursts without dropping work, priority lanes (real-time chat vs batch backfill), and the fairness disciplines for multi-tenant systems where one tenant's traffic spike should not starve another tenant.
- **integration-failure-patterns.md** *(coming)* — Timeout-and-fallback, partial-success in multi-step pipelines, the "model failed but we owe the caller something" pattern, observability hooks, and the integration anti-patterns (the eight-second blocking call inside a 200ms SLO endpoint, the agent step that retries the entire conversation, the streaming response that swallows mid-stream errors).
- **callback-and-webhook-patterns.md** *(coming)* — When async results come back: callback URLs, webhooks with verification, polling endpoints, and the durable-workflow pattern (Temporal, AWS Step Functions, durable function frameworks) that often becomes the cleanest answer for long-running agent work.

## How to use this section

**If you are integrating an AI feature into an existing application**, start with `sync-vs-async-vs-streaming.md`. The integration shape is the load-bearing decision; getting it right makes everything else manageable, and getting it wrong forces you to fight the shape forever.

**If your AI feature is agent-shaped** (multi-step, tool-using, sometimes-slow), `tool-call-architecture.md` and `callback-and-webhook-patterns.md` together describe how the agent should be wired into the rest of the system. The combination of "tool calls go out, results come back, the agent runs as a durable workflow" is the 2026 default for non-trivial agents.

**If your feature involves a human decision step**, `human-in-the-loop-boundaries.md` is non-optional. The boundary design determines whether the human adds safety or becomes a bottleneck that the system silently routes around.

## What this section is not

- **A microservices design guide.** General service-integration patterns (saga, choreography vs orchestration, CDC, etc.) are well-covered elsewhere. This folder is about the AI-specific overlays on top.
- **An MCP / agent-framework tutorial.** Operational depth on MCP and on specific agent frameworks lives in the engineering sibling's `agent-engineering/` folder. This folder addresses the *integration-shape* decision around them.
