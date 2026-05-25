# Sync vs Async vs Streaming

> **Audience.** Architects integrating an AI feature into an existing application. Tech leads choosing the integration shape before any prompt engineering happens. **Scope.** The *architectural* decision about how AI calls integrate with the rest of the system. Not the engineering implementation of streaming (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `observability-and-telemetry/llm-call-instrumentation.md` covers streaming instrumentation). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI calls do not fit the default HTTP request-response shape that most application code is built for. They are slow (seconds for chat; tens of seconds for agent loops). They occasionally fail in opaque ways. They sometimes need to stream partial output to feel responsive. They sometimes need to wait minutes or hours for a human-in-the-loop step. They sometimes need to call back out to existing tools and data sources mid-flight.

Choosing the wrong integration shape is the single most common AI integration mistake. The team builds the chat feature as synchronous HTTP request-response; users see a "loading..." spinner for 8 seconds; UX is poor. Or the team builds bulk classification as a synchronous endpoint; the request times out at 30 seconds when the job needs 5 minutes; the team retrofits async after the fact. Or the team builds an agent feature as streaming but the agent needs HITL approval mid-flight; the integration cannot accommodate the pause.

This document is the framework for choosing the integration shape *before* any AI implementation work begins. The shape is the load-bearing architectural decision; getting it right makes everything downstream easier.

This document is opinionated about three things:

1. **The shape is decided per-feature**, not per-platform. The Care Coordinator chat is streaming; the inline order-entry is also streaming but tighter; the async coordination tasks are async-with-callback; the patient-API assist is synchronous request-response. Different features need different shapes within the same platform.
2. **The shape constrains everything downstream.** Latency budgets, eval suites, error handling, observability, infrastructure all flow from the shape choice. Reverse-engineering the shape later is much more expensive than choosing deliberately upfront.
3. **HITL crossings are first-class.** A workflow that includes human approval mid-flight is async by nature; the integration shape must accommodate it. Hiding the HITL crossing in synchronous code produces the worst of both worlds.

Structure: (2) the three shapes; (3) the decision questions; (4) the decision tree; (5) when shapes hybrid (workflow with embedded shapes); (6) HITL integration patterns; (7) failure-mode considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The three shapes

### 2.1 Synchronous request-response

**Shape.** The client makes an HTTP request; the server processes; the response returns. Standard HTTP semantics. The client blocks (or shows a loading state) until the response arrives.

```
Client → HTTP request → Server processes → HTTP response → Client renders
```

**Sweet-spot workload.**
- Latency under 3 seconds total.
- Single-shot AI call (one LLM invocation, possibly with one retrieval).
- The user is willing to wait briefly.
- The response is a complete unit (no value in partial rendering).

**What it gives you.**
- Simplest integration. Existing HTTP infrastructure works.
- Familiar error handling (HTTP status codes).
- Easy to cache, gate, route, log via existing patterns.

**What it cannot do well.**
- Long latency (>3s) feels unresponsive to interactive users.
- Cannot show intermediate progress.
- HTTP timeouts (30s default in most stacks) cap the operation length.
- Cannot accommodate HITL mid-flight.

**Examples in Meridian.** The patient-API AI assist (lookup-shaped questions, sub-3-second target).

### 2.2 Streaming

**Shape.** The client makes a request; the server streams the response over time (Server-Sent Events, WebSocket, or HTTP chunked transfer). The client renders progressively as content arrives.

```
Client → HTTP request → Server starts processing
                              │
                              ▼
                        Server streams chunk 1 → Client renders
                              │
                              ▼
                        Server streams chunk 2 → Client renders
                              │
                              ▼
                        ...
                              │
                              ▼
                        Server closes stream → Client done
```

**Sweet-spot workload.**
- Interactive chat where perceived latency matters more than total latency.
- Total response generation > 3 seconds but the response is producible incrementally.
- Users benefit from seeing partial output (text response, search results, agent reasoning).
- The response is a sequence (sentences, structured items, agent turns).

**What it gives you.**
- Excellent perceived latency. Time-to-first-token < 1.5 seconds; user is seeing content quickly.
- Total response can be longer than synchronous (10s-30s is fine for chat) without UX cost.
- Naturally fits chat UX patterns.
- Supports backpressure (the client can disconnect; the server stops generating).

**What it cannot do well.**
- Operations that take minutes (the client connection cannot remain open that long reliably).
- Operations that pause for HITL (the stream cannot resume after a multi-hour pause).
- Caching is harder (the cache key is the prompt; the cache value is the full response).
- Proxies and middleboxes can interfere with long-running connections.

**Examples in Meridian.** The Care Coordinator chat panel, the inline order-entry assist.

### 2.3 Asynchronous with callback

**Shape.** The client makes a request; the server returns an immediate acknowledgment (the request was accepted, here is a task ID). The server processes the work asynchronously (potentially over minutes or hours). On completion, the server notifies the client via callback (webhook), polling, or a separate notification channel.

```
Client → HTTP request → Server returns task_id immediately
                              │
                              ▼ (later)
                        Server processes (async, durably)
                              │
                              ▼ (on completion)
                        Server → Webhook to Client OR Client polls for status
                              │
                              ▼
                        Client renders result
```

**Sweet-spot workload.**
- Operations that take longer than ~30 seconds.
- Operations that involve HITL mid-flight.
- Bulk operations (fan-out across many items).
- Operations whose results are not needed immediately by the user.

**What it gives you.**
- Unbounded operation length. Days, weeks if needed.
- Durable orchestration (Step Functions, Temporal); the task survives infrastructure failures.
- Natural fit for HITL crossings (the task waits for the human action).
- Natural fit for bulk / fan-out operations.

**What it cannot do well.**
- Poor UX for interactive workflows (the user starts the task and walks away; there's no real-time progress).
- More infrastructure (durable orchestration, callbacks, status tracking).
- Failure handling is more complex (partial failures in multi-step async).

**Examples in Meridian.** The async coordination tasks (process N patients in a worklist).

---

## 3. The decision questions

Five questions. Answer in writing before reading the decision tree.

### 3.1 What is the latency budget?

- Under 3 seconds: synchronous is the only viable shape.
- 3-30 seconds: streaming if interactive UX; synchronous if non-interactive.
- 30 seconds - 5 minutes: async-with-callback; streaming if the user will watch the whole stream.
- Over 5 minutes: async-with-callback.

### 3.2 What does the user do during the operation?

- **Wait actively** (chat-style; the user is staring at the screen): streaming or fast synchronous.
- **Watch progressively** (agent reasoning, multi-step coordination): streaming.
- **Move on to other work** (bulk operations, long-running tasks): async.
- **Receive notification later** (background processing): async.

### 3.3 Does the operation involve HITL mid-flight?

- Yes: async. (Streaming or synchronous cannot pause for hours of human review.)
- No: any shape can work.

### 3.4 Is the operation a single unit or fan-out?

- Single unit: sync or streaming.
- Fan-out (process N items in parallel): async with map-reduce orchestration.

### 3.5 What is the failure tolerance?

- Strict (e.g., regulated workflow that must complete): async with durable orchestration; survives infrastructure failures.
- Loose (e.g., interactive lookup; user can retry): sync or streaming; retry on failure.

---

## 4. The decision tree

```
Start
  │
  ▼
Is the latency budget < 3s?
  │ YES → Synchronous request-response.
  │
  ▼ NO
Does the operation involve HITL mid-flight?
  │ YES → Async with callback.
  │
  ▼ NO
Is the operation a fan-out over many items?
  │ YES → Async with map-reduce orchestration.
  │
  ▼ NO
Does the user wait actively?
  │ YES → Streaming.
  │
  ▼ NO
Async with callback.
```

The tree is a rough guide; specific cases may have additional considerations (regulatory, cost, etc.).

---

## 5. Hybrid: workflow with embedded shapes

Some features have multiple integration shapes within them. The workflow is the outer shape; embedded steps may have their own shape.

### 5.1 Streaming chat with embedded async tools

The chat is streaming. The user types; the agent responds incrementally. Mid-response, the agent invokes a tool that is itself async (e.g., kicking off a long-running database query). The agent's response includes "I'm running a query; I'll let you know when it's ready."

The streaming response continues — telling the user about the async kickoff — and then the async result is delivered via a separate channel (a new chat message when the query completes).

This pattern accommodates both shapes within one user interaction.

### 5.2 Async coordination with embedded sync sub-tasks

An async coordination task orchestrates N per-item sub-tasks. Each per-item sub-task may itself be synchronous (a single LLM call with retrieval); the outer orchestration is async.

The outer task tracks completion; the inner tasks are short and synchronous.

### 5.3 Sync with embedded streaming

Rare. A synchronous endpoint that internally streams the LLM response for instrumentation purposes but returns the complete response at the end. The outer interaction is sync; the inner LLM call is streamed.

This is usually a sign that the outer shape should be streaming all the way through; collapsing it to sync at the boundary loses the UX benefits of streaming.

---

## 6. HITL integration patterns

When the workflow includes human-in-the-loop steps, the integration shape must accommodate the pause.

### 6.1 The proposal-then-approval pattern

The AI produces a proposal (draft, suggestion); the human reviews and approves or rejects; the system commits on approval.

- AI side: short, fast (synchronous or streaming).
- Human side: arbitrary latency (minutes to days).
- Commit side: triggered by human action; short.

Each phase has its own integration shape; the overall workflow is async.

### 6.2 The interruption pattern

The AI is in a streaming response; mid-stream, it determines it needs human input (a question, a confirmation). The stream pauses; a UI prompt appears; the user responds; the stream resumes.

This is rare and operationally complex. Most teams avoid it by structuring the workflow to ask all questions upfront (proposal-then-approval).

### 6.3 The async queue pattern

The AI produces work items to a queue; humans process the queue at their own pace; processed items trigger downstream work.

This is the bulk-coordination pattern. The user's relationship to any single item is async (they did not initiate it directly).

---

## 7. Failure-mode considerations

Each shape has characteristic failure modes.

### 7.1 Synchronous failure modes

- **Timeout.** The 30s HTTP timeout fires before the operation completes. The user sees an error; the operation may have completed server-side but the user doesn't know.
- **Network interruption.** The connection drops mid-response. The user retries; the operation runs again (no idempotency, the user pays twice).
- **Server crash.** The operation is lost; no recovery.

Mitigations: short operations, idempotency keys, status endpoints.

### 7.2 Streaming failure modes

- **Mid-stream connection drop.** Some content was rendered; the rest is lost. The user sees a partial answer.
- **Proxy interference.** Some proxies / load balancers buffer the response, defeating streaming. Time-to-first-token becomes time-to-complete-response.
- **Backpressure mismatch.** The server produces faster than the network can deliver; or the client renders slower than the network delivers; buffer overflows.
- **Server crash mid-stream.** The stream closes; the user sees an incomplete answer.

Mitigations: idempotency for retries, server-side checkpointing for resume, proxy configuration verified.

### 7.3 Async failure modes

- **Lost callback.** The webhook firing fails; the client never knows the operation completed. Mitigation: durable callback retries; client-side polling as backup.
- **Partial completion of multi-step async.** Some sub-tasks completed; some failed. Mitigation: durable workflow with explicit failure handling per sub-task.
- **Stale task IDs.** The client lost the task ID; cannot retrieve the result. Mitigation: persistent task ID storage on the client; status query by client-known reference.
- **Cost runaway in long-running tasks.** A multi-step async task hits the cost circuit-breaker partway. Mitigation: per-task cost budget; checkpoint and resume capability.

Mitigations: durable orchestration (Step Functions, Temporal); explicit failure handling per sub-task; cost budgets at task level.

### 7.4 The failure-handling pattern matrix

| Shape | Primary failure handling | Recovery pattern |
|---|---|---|
| Sync | Timeouts; idempotency keys | Retry with same idempotency key |
| Streaming | Resume from checkpoint | Re-stream from last successful chunk |
| Async | Durable orchestration | Retry sub-tasks; restart whole task from checkpoint |

---

## 8. Worked Meridian Health example

The Care Coordinator and related Meridian AI features illustrate the four shapes in one platform.

### 8.1 Patient-API AI assist → Synchronous request-response

**Workload.** Patient asks a question via the patient app. The system retrieves from patient-education content and answers.

**Five questions answered.**
1. Latency budget: sub-3-second target.
2. User waits actively (patient watching the loading state).
3. No HITL.
4. Single-unit operation.
5. Loose failure tolerance (patient can retry).

→ **Synchronous.** HTTP POST with the question; HTTP response with the answer.

**Implementation.** Sub-2-second p99 latency. Cached responses for FAQ-shaped questions. Simple HTTP; no streaming complexity.

### 8.2 Care Coordinator chat → Streaming

**Workload.** Clinician asks a question in the chat panel. The system answers with cited clinical reasoning.

**Five questions answered.**
1. Latency budget: total < 6 seconds; TTFT < 1.5 seconds.
2. User waits actively; benefits from seeing progressive response.
3. No HITL within the chat interaction itself (side-effect operations are separate).
4. Single-unit response.
5. Loose failure tolerance (retry).

→ **Streaming.** SSE from the API to the chat panel. The supervisor's final consolidation streams to the user.

**Implementation.** SSE-over-HTTPS; explicit keepalive; resume-on-disconnect for transient network issues. TTFT and total-latency tracked as SLIs.

### 8.3 Care Coordinator side-effect path (draft patient message) → Async with HITL

**Workload.** Clinician asks the Coordinator to draft a follow-up message. The AI creates a draft; the clinician reviews; the clinician sends.

**Five questions answered.**
1. Latency budget: AI portion is short (~3s); human portion is arbitrary (minutes to hours).
2. User initiates and walks away; comes back to review.
3. HITL is the core of the workflow.
4. Single draft per request.
5. Strict failure tolerance (clinical workflow; drafts must not be lost).

→ **Async with HITL.** The AI creates the draft synchronously (returned immediately to the clinician's UI). The draft is queued for HITL review. The clinician reviews on their own time. The send action triggers a separate notification.

**Implementation.** Draft creation is a single fast API call. The draft persists in a queue. The HITL UI is the existing clinical platform's outbound-message review panel. The send action goes through standard clinical workflow.

### 8.4 Async coordination tasks → Async with map-reduce

**Workload.** Care coordinator initiates "for every CHF patient discharged in the last 7 days, draft an outreach message." Fan-out across N patients.

**Five questions answered.**
1. Latency budget: minutes (fan-out across N patients, each ~30s).
2. User initiates and goes to other work.
3. HITL is per-draft (each draft awaits clinician review).
4. Fan-out over N items.
5. Strict failure tolerance (regulated workflow; drafts must complete).

→ **Async with map-reduce orchestration.** Step Functions orchestrates per-patient Lambda invocations. Each Lambda creates one draft. Completion notifies the coordinator UI.

**Implementation.** Step Functions handles orchestration, retries, partial failures. Per-patient drafts queued for HITL. Aggregate completion webhook fires when all sub-tasks complete (or after a timeout with explicit partial-result handling).

### 8.5 The integration-shape matrix

| Feature | Shape | TTFT | Total budget | Failure-handling |
|---|---|---|---|---|
| Patient-API assist | Sync | n/a | <3s | Retry-on-error |
| Care Coordinator chat | Streaming | <1.5s | <6s | Resume-on-disconnect |
| Inline order-entry assist | Streaming | <1s | <4s | Resume-on-disconnect |
| Side-effect draft creation | Async with HITL | n/a | days (HITL window) | Durable queue |
| Bulk coordination tasks | Async with map-reduce | n/a | minutes-hours | Durable workflow |
| Patient record summarization | Async with batch | n/a | minutes (batch) | Retry-from-checkpoint |

Each shape was chosen deliberately to match the workload characteristics. Mixing shapes for a single feature would be over-engineering; using a uniform shape for all features would be under-engineering.

### 8.6 The platform discipline

- Per-feature integration-shape decisions are documented (an ADR per feature; or a section in the feature's design doc).
- The four shapes are first-class platform capabilities (the streaming infrastructure, the async orchestration, the HITL queue).
- New AI features go through the integration-shape decision review.
- Quarterly: the chosen shapes are reviewed against actual operational data (latency, success rates, user satisfaction).

---

## 9. Anti-patterns

### 9.1 "Sync for everything"

The team uses synchronous HTTP for every AI feature because "it's simpler." The chat feature has 8-second responses with a loading spinner; bulk operations time out at 30 seconds; HITL workflows are jury-rigged with polling.

**Corrective.** Match the shape to the workload. Synchronous is right for some features; the wrong shape elsewhere is a poor UX or a broken workflow.

### 9.2 "Streaming for everything"

The team builds every AI feature as streaming because "streaming is modern." Background batch jobs are streamed for no UX benefit; the streaming infrastructure complexity bleeds into features that did not need it.

**Corrective.** Streaming is for interactive workloads where perceived latency matters. Background work is async.

### 9.3 "Async for everything"

The team builds every AI feature as async to "future-proof." Simple lookup features now have queue-and-callback complexity; UX is delayed by orchestration overhead; debugging is harder.

**Corrective.** Async is for long-running, fan-out, or HITL workflows. Simple short interactive features are sync or streaming.

### 9.4 "HITL inside a streaming response"

The team tries to handle HITL mid-stream — the stream pauses, a UI prompt appears, the stream resumes after the human action. The pattern works in a demo and breaks in production (proxies time out the connection; users walk away; the stream cannot resume reliably).

**Corrective.** HITL crossings are async boundaries. The AI's output before the HITL is one workflow phase; the human action and post-HITL work are separate phases.

### 9.5 "Synchronous endpoint for async work"

The team builds a synchronous endpoint that internally takes 5 minutes (a bulk job, a multi-step agent). HTTP times out at 30 seconds; the client sees an error; the work may or may not have completed server-side.

**Corrective.** Operations longer than ~25 seconds (leaving margin for the 30s timeout) are async.

### 9.6 "Streaming through a proxy that buffers"

The team implements streaming server-side; the CDN or proxy buffers the response; the client sees the entire response at once after the server finishes. The streaming UX benefit is lost.

**Corrective.** Verify the full request path supports streaming (proxies, load balancers, CDNs). Disable buffering or use direct connections.

### 9.7 "Async without durable orchestration"

The team implements async with in-memory job queues. A server crash loses all in-flight tasks. Recovery is manual.

**Corrective.** Durable orchestration (Step Functions, Temporal, or similar). The orchestrator survives infrastructure failures.

### 9.8 "Integration-shape decision deferred"

The team builds the AI prompt and the UI; the integration shape is decided last, often based on what is easiest to wire up. The result is a mismatch — the shape is convenient to implement but wrong for the workload.

**Corrective.** Shape is the first decision, before any AI work. The five questions in section 3 produce the answer.

---

## 10. Findings (sprint-assignable)

### ARCH-SHAPE-001 — Severity: Critical
**Finding.** Production AI feature uses synchronous HTTP for an operation that takes >25 seconds; client timeouts produce false errors.
**Recommendation.** Migrate to async with callback; client gets task_id immediately, polls or receives webhook.
**Owner.** ai-platform-eng + product, sprint N+1.

### ARCH-SHAPE-002 — Severity: High
**Finding.** Interactive chat feature is implemented as synchronous request-response; users see 8-second loading spinners.
**Recommendation.** Migrate to streaming per section 2.2; TTFT becomes the user-visible latency.
**Owner.** ai-platform-eng + product, sprint N+2.

### ARCH-SHAPE-003 — Severity: High
**Finding.** Workflow with HITL is shoe-horned into a streaming or synchronous shape; the HITL step is jury-rigged.
**Recommendation.** Restructure as async with HITL per section 6.1; the proposal phase and the approval phase are explicit.
**Owner.** ai-platform-eng + product, sprint N+2.

### ARCH-SHAPE-004 — Severity: High
**Finding.** Streaming infrastructure does not handle proxy buffering; production deployment sees responses delivered in one chunk after generation completes.
**Recommendation.** Verify and configure the request path (CDN, load balancer, proxies) for streaming; integration-test the full path.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ARCH-SHAPE-005 — Severity: High
**Finding.** Bulk async operations are implemented with in-memory queues; server crashes lose in-flight work.
**Recommendation.** Durable orchestration (Step Functions, Temporal) per section 2.3; in-flight tasks survive infrastructure failures.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ARCH-SHAPE-006 — Severity: High
**Finding.** Async callback failures (webhook delivery failures) are not retried; clients miss completion notifications.
**Recommendation.** Durable retry of callbacks; client-side polling as backup; both implemented.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-SHAPE-007 — Severity: Medium
**Finding.** Integration-shape decision was made by default (whatever was easiest), not by the five-question framework.
**Recommendation.** ADR per feature documenting the integration-shape decision and rationale.
**Owner.** ai-platform-eng + product, sprint N+3.

### ARCH-SHAPE-008 — Severity: Medium
**Finding.** Synchronous endpoints do not have idempotency keys; retries can produce duplicate work or duplicate side effects.
**Recommendation.** Idempotency-key contract on all synchronous AI endpoints.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SHAPE-009 — Severity: Medium
**Finding.** Streaming responses do not support resume-on-disconnect; transient network issues lose the rest of the response.
**Recommendation.** Server-side checkpointing within the streaming response; resume token allows client to reconnect.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SHAPE-010 — Severity: Medium
**Finding.** Async tasks do not have per-task cost budgets; long-running tasks can consume meaningful cost without a circuit.
**Recommendation.** Per-task cost budget; checkpoint and resume capability when budget approaches.
**Owner.** ai-platform-eng + finops, sprint N+3.

### ARCH-SHAPE-011 — Severity: Medium
**Finding.** HITL approval surface and the AI portion of the workflow are operationally separate but their failure correlation is not tracked.
**Recommendation.** End-to-end observability for HITL workflows; track time-to-approval and approval rates per feature.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ARCH-SHAPE-012 — Severity: Medium
**Finding.** Fan-out operations have no per-partition failure isolation; one partition's failure affects others.
**Recommendation.** Per-partition failure handling in the orchestration; partial results returned with explicit failure marking.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SHAPE-013 — Severity: Medium
**Finding.** Streaming response error handling is implicit; clients see connection-closed without classification of the failure.
**Recommendation.** Server emits a final error chunk before closing; client renders the error appropriately.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SHAPE-014 — Severity: Medium
**Finding.** Async task statuses are not surfaced in user-facing UI; users do not know whether their initiated task is still running.
**Recommendation.** Status polling endpoint; status indicator in UI; notification on completion.
**Owner.** ai-platform-eng + product, sprint N+4.

### ARCH-SHAPE-015 — Severity: Medium
**Finding.** Async task IDs are not durable client-side; users who close the browser lose access to their task results.
**Recommendation.** Server-side task registry with user attribution; user can query tasks they initiated.
**Owner.** ai-platform-eng + product, sprint N+4.

### ARCH-SHAPE-016 — Severity: Low
**Finding.** Time-to-first-token (TTFT) is not a tracked SLI for streaming features; UX regressions are detected by user complaints.
**Recommendation.** TTFT as an SLI per [llm-call-instrumentation.md](../../ai-engineering-reference-architecture/observability-and-telemetry/llm-call-instrumentation.md).
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ARCH-SHAPE-017 — Severity: Low
**Finding.** Quarterly review of integration-shape decisions against operational data is not scheduled.
**Recommendation.** Schedule quarterly review; re-evaluate shapes against current latency / failure / UX data.
**Owner.** ai-platform-eng team lead, sprint N+5.

### ARCH-SHAPE-018 — Severity: Low
**Finding.** Integration-shape patterns are not documented as a platform capability; new feature teams reinvent.
**Recommendation.** Document the four shapes (sync, streaming, async, async-with-HITL) as platform capabilities; include code examples and operational patterns.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team designing a new AI feature:

- [ ] **Sprint 0 — decide.** Answer the five questions (section 3). Run through the decision tree (section 4). Document the chosen shape and rationale (ADR).
- [ ] **Sprint 0 — verify infrastructure.** Confirm the chosen shape is supported by the existing infrastructure (streaming through proxies, durable orchestration for async).
- [ ] **Sprint 1 — implementation.** Build the feature with the chosen shape. Use existing platform capabilities (streaming infrastructure, Step Functions, HITL queues).
- [ ] **Sprint 1 — failure handling.** Implement the failure-handling pattern for the chosen shape (idempotency for sync, resume for streaming, durable orchestration for async).
- [ ] **Sprint 2 — observability.** Trace and instrument per [trace-and-span-design.md](../../ai-engineering-reference-architecture/observability-and-telemetry/trace-and-span-design.md). TTFT for streaming; per-step latency for async.
- [ ] **Sprint 2 — UX integration.** Ensure the UX matches the shape (loading states for sync; progressive rendering for streaming; status indicators for async).
- [ ] **Sprint 3 — review.** Review the operational data after some production traffic; verify the shape is right.
- [ ] **Ongoing — quarterly review.** Re-evaluate the shape against current data; migrate if mismatch is evident.

A team that completes this sequence has the integration-shape discipline that produces good UX, predictable operational characteristics, and clean failure handling. A team that defaults to the easiest-to-implement shape pays the mismatch tax in production.

---

## 12. References

- HTTP/1.1 and HTTP/2 specifications for the underlying transport.
- Server-Sent Events (SSE) specification.
- WebSocket protocol.
- AWS Step Functions, Temporal documentation for durable orchestration patterns.
- This repo: [integration-architecture/human-in-the-loop-boundaries.md](./) (coming) — depth on HITL integration.
- This repo: [integration-architecture/callback-and-webhook-patterns.md](./) (coming) — depth on async callback patterns.
- This repo: [integration-architecture/integration-failure-patterns.md](./) (coming) — the failure-mode catalog this document references.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the multi-shape platform example.
- Sibling repo: [ai-engineering-reference-architecture/observability-and-telemetry/llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md) — the streaming-instrumentation engineering practice.
- Sibling repo: [ai-engineering-reference-architecture/reliability-engineering/fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) — the per-shape failure handling.
