# Integration Failure Patterns

> **Audience.** Architects designing how AI features fail safely at integration boundaries. Tech leads whose AI feature occasionally produces partial output, hallucinated tool calls, or mid-stream errors. Anyone whose AI feature's failure is visible to a customer, a regulator, or a system of record. **Scope.** The *architectural* decisions: the failure taxonomy at AI integration boundaries; timeout-and-fallback architecture; partial-success in multi-step pipelines; "model failed but we owe the caller something" patterns; dual-write, saga, and outbox patterns for AI side effects; idempotency contracts at the integration boundary. Not the integration-shape decision itself (see [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md)). Not the engineering-side resilience patterns inside the consumer (see sibling [reliability-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/reliability-engineering)). Not the agent-internal partial-failure recovery (see sibling [agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI features fail in ways that conventional service integrations don't. A REST call either returns 200 with a parseable body or returns an error code; the failure modes are mostly known and the integration patterns (retry, circuit-breaker, fallback) cover them. An LLM call returns 200 with a body that may be:

- Truncated mid-output because of a token limit, a network blip, or a stream reset.
- Syntactically valid JSON whose semantics violate the consumer's expectations.
- A confident-sounding tool call invocation that names a tool that doesn't exist.
- A confident-sounding answer based on retrieved content that contradicts the consumer's source of truth.
- A refusal that returns no usable content at all, while the caller's UI was promised a paragraph.
- The wrong model's output because mid-stream the provider failed over to a different model in their fleet.
- A 200 OK followed by zero bytes because the streaming connection was torn down.

These failures don't fit the request/response model the caller's code expects. The caller's code was written for "succeed or fail"; the LLM operates in "succeed, fail, or produce something in between." The "in between" is where most production AI integration bugs live.

The architectural framing matters because AI failure modes have second-order effects that conventional failures don't:

- **A truncated LLM response that the caller treats as complete** writes a partial EHR note, a partial code change, or a partial customer email. The downstream cleanup is more expensive than the LLM call itself.
- **A hallucinated tool call** can execute a destructive action with confidence. The integration's job is to refuse before execution, not to trust the call.
- **A model fallback to a cheaper model mid-stream** changes the output's quality without changing the caller's metadata. The consumer thinks they got Sonnet output; they got Haiku output for the second half.
- **A "model failed" response that surfaces as a 500 to the user** in a customer-visible feature produces support tickets and churn. The "what to return when the model failed" decision is a UX decision as well as an engineering decision.

This document covers the integration-level decisions that determine how AI features fail. It is the architectural complement to the engineering-side patterns in the sibling reliability-engineering folder: those cover *how* a consumer retries, falls back, or degrades inside its boundary; this covers *what* the integration contract guarantees about partial success, what the consumer owes the caller when things go wrong, and how AI-driven side effects participate in saga / outbox / idempotency patterns that the rest of the system uses.

This document is opinionated about four things:

1. **The LLM call's success is not the integration's success.** A successful LLM call can produce unparseable output, half-completed work, or a confidently-stated wrong answer. The integration's contract is what's delivered to the caller — not the provider's HTTP status code.
2. **Partial work must be either reversible or committed atomically with the rest.** "The agent did step 1 and step 2, then failed on step 3" produces orphaned state unless step 1's and step 2's side effects are reversible (saga compensation) or were buffered until the whole sequence committed (outbox).
3. **Idempotency keys at integration boundaries are not the same as idempotency keys inside the LLM consumer.** The caller's idempotency key is per *request*; the consumer's idempotency key is per *attempt*. Conflating them produces double-charges, double-writes, and replay confusion.
4. **The "model failed" path is a load-bearing feature, not an exception case.** Designing it as an afterthought ("we'll throw a 500") produces brittle systems. Designing it as a first-class response shape (degraded answer, cached answer, refusal-with-context, structured error) produces robust ones.

Structure: (2) the failure taxonomy at AI integration boundaries; (3) timeout-and-fallback architecture; (4) partial success in multi-step pipelines; (5) the "model failed but we owe the caller something" pattern; (6) dual-write, saga, and outbox patterns for AI side effects; (7) idempotency contracts at the integration boundary; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The failure taxonomy at AI integration boundaries

The first move in any AI integration failure design is having a vocabulary for the failure modes. The classes below are the ones every production AI integration eventually encounters.

### 2.1 Pre-call failures

**Network unreachable to provider.** Classic transient. Retry resolves it. Caller-visible: 503 from your side.

**Authentication failure.** Provider rejected the API key. Almost always a config issue; retry won't help. Caller-visible: 500 from your side; page the operator.

**Pre-flight validation failure.** Your consumer rejected the request before calling the model (input too large, content-policy violation in the input, missing required field). Caller-visible: 400 with a structured error explaining what's invalid.

**Rate-limit budget exhausted at your side.** The fleet-wide budget (see [backpressure-and-queueing.md](./backpressure-and-queueing.md)) is empty; the consumer doesn't even try the provider. Caller-visible: 429 with Retry-After.

### 2.2 In-call failures

**Provider 4xx errors (excluding 429).** Usually a request-format problem (the consumer constructed an invalid prompt or tool schema). Should not retry; fix the construction. Caller-visible: 500 from your side (this is a consumer bug, not a caller bug).

**Provider 429.** Rate limit hit. Retry with backoff (provider-aware; see [backpressure-and-queueing.md §4](./backpressure-and-queueing.md)). Caller-visible: depends — short bursts may be transparent (you queue and retry); sustained 429s should surface as 429 to caller.

**Provider 5xx.** Provider-side server error. Retry; if persistent, fall back (cheaper model, cached response, degraded path) or surface as caller-visible failure.

**Network timeout during call.** Connection started but didn't complete. The LLM call may or may not have happened on the provider's side (idempotency consideration). Retry with caution; the LLM might charge you for the failed call.

**Streaming connection torn down mid-output.** The consumer received N tokens; then the connection dropped. The N tokens may be partial JSON, half a sentence, or half a structured response.

### 2.3 Post-call failures (the "200 OK but wrong" class)

**Output schema violation.** The LLM returned valid JSON but with the wrong fields, wrong types, or wrong values. The structured-output enforcement (function-calling, JSON schema) reduces but doesn't eliminate this. Caller-visible: depends on the contract — you can retry, you can salvage the partial, or you can fail.

**Truncated output.** The LLM hit max_tokens and stopped mid-sentence (or mid-JSON-object). Detectable from the `stop_reason` (or `finish_reason`) field. Caller-visible: must not be presented as complete; either continue (multi-turn) or fail explicitly.

**Hallucinated tool call.** The LLM emitted a tool call invocation for a tool that doesn't exist, with arguments that don't match any tool's schema. The integration must reject this; not execute it.

**Refusal.** The LLM declined to answer. Detectable from response content and (sometimes) from a refusal flag in the response metadata. Caller-visible: depends on the workload; sometimes a refusal is correct behavior, sometimes it's a misclassified safety filter.

**Wrong-model output (silent fallback).** Some providers will silently route to a backup model when the primary is overloaded. The output is semantically valid but lower-quality than expected; the response metadata indicates the actual model used. Often unnoticed in development; caller-visible only via the metadata.

**Content-policy refusal.** The provider's safety system refused. May come back as a special status or as a generic-sounding refusal. Distinct from a content-aware refusal (where the model itself decided not to answer); the policy refusal is enforced before the model sees the input.

### 2.4 Side-effect failures

**Side effect succeeded; LLM call result lost.** The LLM call succeeded; the consumer wrote the side effect (e.g., updated the DB); the consumer crashed before returning to the caller. From the caller's perspective, the request failed; from the system's perspective, the side effect happened. Without idempotency, the caller's retry duplicates.

**Side effect partially succeeded.** Multi-step side effect (e.g., agent that writes to three downstream systems); some writes succeeded, some failed. The system is in an inconsistent state.

**Side effect performed but the wrong side effect.** The LLM's output was wrong (hallucinated tool argument, misclassified routing) and the consumer dutifully executed it. Reversibility becomes the question; cross-link to §4.

### 2.5 Downstream failures

**Downstream consumer rejected the AI output.** Downstream service returns an error when the AI consumer tries to write. Could be schema, could be permission, could be rate-limit at the downstream.

**Downstream consumer accepted but processed incorrectly.** Downstream wrote the data but later processing failed. The failure is visible later (or never, if it's silent).

**Cascade failure.** The AI consumer's failure took down the downstream consumer (e.g., AI consumer flooded downstream with retries until downstream became unavailable).

### 2.6 The taxonomy as a contract

The architecture must commit to handling each class explicitly. The default behavior — "treat any non-2xx as 'try again later' and any 2xx as success" — produces wrong behavior for at least four of the classes above:

- Truncated output isn't an error; it's a 200 OK with incomplete content.
- Hallucinated tool calls aren't an error; they're a 200 OK with a destructive instruction.
- Silent model fallback isn't an error; it's a 200 OK with quality degradation invisible to the caller.
- Side-effect-succeeded-call-lost isn't an error from the system's perspective but is from the caller's.

A consumer's failure-handling code should have a branch for each class in §2.1-§2.5. If a class isn't handled, the team should be able to explain why explicitly — "we accept silent fallback because we don't differentiate Sonnet vs Haiku output for this use case" is a fine answer; "we never thought about it" is not.

---

## 3. Timeout-and-fallback architecture

How long to wait, and what to return when the wait is over without a useful response.

### 3.1 The latency budget cascade

Every AI integration has a latency budget. The caller has one (the user expects a response in N seconds); the consumer has a sub-budget (the LLM call must complete in M < N seconds to leave room for the consumer's own work); the LLM call itself has a sub-budget (the provider's median first-token latency, the average tokens-per-second, the expected output length).

**Pattern.** Express the budget as a cascade:

```
Caller budget:               2000 ms
  Consumer overhead:           50 ms (parsing, validation, side-effect prep)
  Side-effect write:          200 ms
  Headroom for retry:         300 ms
  Available for LLM call:    1450 ms
```

The LLM call gets a hard timeout at 1450ms. If it exceeds, the consumer falls back rather than continuing to wait.

**Pitfall.** No headroom for retry. If the budget for the LLM call equals the caller's full budget, a single timeout means the caller waits the full budget and gets nothing. Reserve at least one retry's worth of time, or commit to "no retry" explicitly.

### 3.2 Choosing the timeout

**Static timeout.** "LLM calls timeout at 1500ms." Easy to implement; wrong for variable-cost calls. A short summarization gets the same timeout as a long agent loop.

**Per-call-class timeout.** Each call type (classify, summarize, agent-step, batch) has its own timeout. More accurate; more configuration to maintain.

**Adaptive timeout.** The timeout is derived from the call's expected work — e.g., `expected_output_tokens / provider_tokens_per_second × safety_factor`. The most accurate; the hardest to debug when wrong.

**Default recommendation.** Per-call-class timeout, with the cascade (§3.1) determining each class's timeout from the caller's budget.

### 3.3 The fallback ladder

When the primary path fails (timeout, error, refusal), the fallback ladder defines the next attempt:

```
Primary:        Sonnet, full context, full output
↓ on timeout/error
Fallback 1:     Haiku, full context, full output
↓ on timeout/error
Fallback 2:     Cached response (if available)
↓ on miss
Fallback 3:     Templated / default response
↓ on inability to template
Fallback 4:     Structured error to caller
```

Each rung has its own latency budget; the cumulative budget must fit within the caller's budget.

**Pattern.** Parallel fallback. Issue the primary call; in parallel, prepare the fallback (cache lookup, smaller-model call). If the primary fails, the fallback is already ready or nearly so. Costs more (you pay for both) but reduces caller-visible latency on failure.

**Pattern.** Sequential fallback. Try primary; on failure, try fallback 1; on failure, try fallback 2. Cheaper; longer cumulative latency on failure.

The choice depends on cost sensitivity. For high-volume, low-cost workloads, sequential is fine. For latency-sensitive, low-volume, high-value calls (clinical decision support, financial transactions), parallel is worth the cost.

Detail on the fallback patterns inside the consumer is in [reliability-engineering / fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) in the engineering sibling.

### 3.4 What "fallback to error" looks like

Sometimes there is no acceptable fallback. The integration must communicate failure to the caller cleanly.

**Pattern.** Structured error with action guidance:

```json
{
  "status": "error",
  "error_class": "model_unavailable",
  "error_detail": "Primary and fallback models both unavailable",
  "retry_after_ms": 5000,
  "alternative_action": "Try again, or contact support if the issue persists",
  "incident_id": "incident_2026_05_27_3a4f"
}
```

The caller's UI can render an actionable message. The incident_id ties the customer's complaint to the operator's logs.

**Anti-pattern.** Bare 500. Caller's UI shows "An error occurred." Caller's support team can't correlate to anything. Operator's log shows a generic exception. Diagnosis takes hours.

### 3.5 The streaming-timeout special case

Streaming complicates timeouts. The connection is open; tokens are flowing; the question is "are they flowing fast enough?"

**Pattern.** Time-to-first-token (TTFT) timeout. If no tokens have arrived in N seconds, fail; the call is stuck.

**Pattern.** Inter-token timeout. If tokens have arrived but the gap between tokens exceeds N seconds, fail; the stream is hung.

**Pattern.** Total-duration ceiling. Even if tokens are flowing, fail if the total call exceeds the budget; don't let a verbose generation overshoot.

All three should be configured; relying on one alone produces edge cases (a stream that produces one token, then waits 60 seconds, then produces another — each token "in time" by inter-token timeout but the total far over the budget).

---

## 4. Partial success in multi-step pipelines

Multi-step AI pipelines — agent runs, multi-LLM-call workflows, sequenced enrichment — fail in the middle more often than at the end. The integration must define what's owed to the caller when step N of M succeeded but step N+1 failed.

### 4.1 The three resolution strategies

**Strategy A: All-or-nothing.** If any step fails, the whole pipeline fails. Side effects from successful steps must be rolled back (compensation) or must not have been committed (outbox pattern).

**Pros.** Caller's mental model is simple: it worked or it didn't. No partial state to reason about.

**Cons.** Lost work — successful expensive LLM calls are discarded. Cost penalty on the next retry.

**When right.** Workflows where partial completion is meaningless (a referral that was only partially routed to half the providers it should have reached) or actively dangerous (a financial transaction with one side recorded and the other side missing).

**Strategy B: Partial-OK.** Return whatever's been completed plus a "step N failed" indicator. Caller decides whether to use the partial result, retry the failed step, or abandon.

**Pros.** No wasted LLM calls. Caller can salvage value.

**Cons.** Caller must handle partial results; many callers don't. Easy to silently drift to "treat partial as complete."

**When right.** Workflows where partial completion has real value (a customer support summary that has the timeline but not the recommendation) and the caller is sophisticated enough to handle it.

**Strategy C: Retry-from-failure.** Persist all successful step outputs; on retry, resume from the failed step. Caller waits longer but gets a complete result without recomputing successful steps.

**Pros.** No lost work; complete result for the caller.

**Cons.** Requires durable state for intermediate results; more complex implementation; the retry might still fail.

**When right.** Long-running pipelines where individual steps are expensive (agent loops, multi-LLM-call enrichment). The standard pattern for production agent infrastructure.

### 4.2 Choosing the strategy

The choice is per-workflow, not per-platform. Different workflows in the same system can use different strategies.

| Workflow | Strategy | Rationale |
| --- | --- | --- |
| Single-call classification | All-or-nothing | One step; no partial state |
| Agent that drafts an email | Retry-from-failure | Expensive steps; persistent state easy to manage |
| Multi-stage document enrichment | Partial-OK | Each enrichment has independent value |
| Financial transaction processing | All-or-nothing with saga compensation | Partial state is dangerous |
| Clinical decision support pipeline | Retry-from-failure | Caller is waiting; resume is faster than restart |
| Background analytics enrichment | Partial-OK | Value of partial is high; cost of retry is wasted |

### 4.3 Persistence of intermediate state

For Strategy C, the architecture must persist intermediate state durably. Options:

**Inline in the request context.** Each step's output is in memory; passed to the next step. Lost on crash. Acceptable for short pipelines.

**Per-pipeline state store.** A row per pipeline run, with one column per step's output (or a JSON blob). Durable; manual to manage; common for moderate-complexity pipelines.

**Durable workflow framework.** Temporal, Step Functions, Inngest, durable function frameworks. The framework handles persistence; the pipeline is expressed as a workflow function. The standard production pattern for non-trivial agent infrastructure.

**Pattern.** Durable workflow for pipelines that have more than three steps or that run longer than a few seconds. The framework cost is justified by the operational properties (resume on crash, retry per step, observability, replay).

### 4.4 The compensation problem

When all-or-nothing requires rollback, what gets rolled back, and how?

**Reversible side effects.** Database writes (delete the row), file writes (delete the file), in-flight notifications (cancel before send). Reversible at low cost.

**Quasi-reversible side effects.** Webhook delivery (recipient already processed it; you can send a "cancel" event but they may not handle it), email send (you can send a "disregard" follow-up; it's confusing). Reversible only with effort and only partially.

**Irreversible side effects.** Money sent, physical action taken, message posted to a public channel. Not reversible. The architecture must either commit only at the end (no irreversible side effect mid-pipeline) or accept the risk.

**Pattern.** Two-phase commit at the workflow level. All steps that have irreversible side effects are deferred until phase 2 (final commit). If phase 1 (reversible steps + draft of irreversible) completes, phase 2 commits everything. If phase 1 fails partway, rollback is bounded to reversible steps.

**Pattern.** Outbox for irreversible side effects. The step writes to an outbox (reversible) rather than performing the side effect directly. An outbox-publisher runs after the pipeline commits; if the pipeline aborts, the outbox entry is deleted before publication. Cross-link to §6.

---

## 5. The "model failed but we owe the caller something" pattern

A common failure mode: the LLM call failed (or returned nothing useful), but the caller is waiting and expects content. The integration must decide what to return.

### 5.1 The four response shapes

**Shape A: Degraded answer.**

A response generated by a cheaper, faster, or simpler path. Could be a smaller model's answer (fallback ladder), a cached answer from a similar query, or a templated response.

**Pros.** Caller gets *something* useful. UX is preserved.

**Cons.** Quality is lower; may be wrong; the caller may not know it's degraded.

**When right.** UX-driven contexts (chat, copilot, search) where any answer beats no answer.

**When wrong.** Decision-making contexts (clinical, financial, legal) where a wrong-but-confident answer is worse than no answer.

**Implementation.** Return the degraded answer with a metadata flag indicating it's a fallback. The UI can render it with a "less confident" indicator or use it silently depending on the contract.

**Shape B: Cached answer.**

A previously generated answer (for the same query, or a similar one) served from cache.

**Pros.** Zero LLM cost; sub-millisecond latency; quality is whatever the original answer's quality was.

**Cons.** May be stale; may not exactly match the current query.

**When right.** Read-heavy workloads with high query overlap. Cache hit during model outage is a reliability win.

**Implementation.** Cache keyed on a normalized form of the input. Cache entry includes timestamp and source-model metadata; consumer can decide whether to serve stale.

**Shape C: Refusal-with-context.**

The integration explicitly declines to answer, with context about why and what to try next.

```json
{
  "status": "unable_to_respond",
  "reason": "Primary model is unavailable and no acceptable fallback found",
  "suggested_actions": ["Retry in 30 seconds", "Contact your administrator if persistent"],
  "incident_id": "incident_2026_05_27_a3f"
}
```

**Pros.** Honest; doesn't fabricate. Caller can decide what to do.

**Cons.** UX cost; user sees a "we couldn't" message.

**When right.** Safety-critical contexts where a fabricated answer is worse than a refusal. Regulated workloads where the audit trail matters.

**Implementation.** Always return as structured error, not free-text. The structure lets the caller render appropriately.

**Shape D: Structured null.**

Return a response in the expected shape but with empty/null values, plus a metadata indicator that the response is incomplete.

```json
{
  "summary": null,
  "key_points": [],
  "status": "incomplete",
  "reason": "model_timeout"
}
```

**Pros.** Caller's parsing code works unchanged; they handle nulls.

**Cons.** Easy for caller to forget to check the status field and treat nulls as valid.

**When right.** Internal APIs where caller is sophisticated and the shape-stability is valuable.

**When wrong.** Customer-facing APIs where naive callers might miss the status flag.

### 5.2 Choosing the shape per call class

| Call class | Default shape | Rationale |
| --- | --- | --- |
| Customer-facing chat | Degraded answer | UX-critical; small quality loss acceptable |
| Search query | Cached answer | High query overlap; cache hit common |
| Clinical decision support | Refusal-with-context | Wrong answer is dangerous; explicit refusal is safe |
| Document classification (batch) | Refusal-with-context | Downstream handles "unclassified" cleanly |
| Tool call from agent | Refusal-with-context to the agent | Agent decides how to recover |
| Internal enrichment API | Structured null | Sophisticated callers; shape stability matters |
| Financial advisory | Refusal-with-context | Regulatory and liability constraints |
| Marketing content draft | Degraded answer (smaller model) | Drafts are reviewed by humans; degradation is OK |

### 5.3 The "always return JSON" discipline

Whatever shape, the integration's response should always be parseable. The caller's code should never have to handle "200 OK with HTML error page" or "200 OK with empty body."

**Pattern.** Every response, including errors, is the same content type and parseable in the same way. Errors are wrapped in the same response envelope; success and failure are distinguished by a status field, not by HTTP code alone.

**Pattern.** Schema validation at the integration boundary. Before returning to the caller, the integration validates its own response against a schema. If validation fails, return a structured error rather than the malformed response.

### 5.4 The "be honest about confidence" practice

When returning a degraded or cached response, mark it.

```json
{
  "summary": "The customer asked about refund eligibility...",
  "confidence": "medium",
  "source": "fallback_model",
  "primary_model_status": "unavailable"
}
```

Callers can choose to use the metadata (render with a visual indicator) or ignore it. Either way, the truth is in the response — if downstream consumers later need to know that this answer was degraded, the metadata is there.

The anti-pattern is to silently degrade. If the caller can't distinguish a primary-model answer from a fallback-model answer, the system has built-in deception. Downstream analytics will conflate the two; quality metrics will silently degrade.

---

## 6. Dual-write, saga, and outbox patterns for AI side effects

When the AI feature performs side effects, the same patterns that govern conventional service side effects apply — but with AI-specific overlays.

### 6.1 The dual-write problem

The classic dual-write problem: a service writes to two systems (e.g., database + message queue). The first write succeeds; the second fails; the system is inconsistent.

For AI features, the analog is:

- LLM call succeeded; side effect write failed.
- Side effect write succeeded; result publication failed.
- Multiple side effects (agent writes to N downstream systems); some succeeded, some failed.

The dual-write problem is well-understood; the solution is the outbox pattern (write to one system transactionally; have a publisher read from that system and produce the other writes).

### 6.2 The outbox pattern for AI

**Standard outbox.** The service writes the work it intends to publish to an outbox table in its own DB, transactionally with its other state. A publisher polls the outbox and publishes to the downstream system (broker, API, etc.). After successful publish, the outbox entry is marked done.

**AI variant.** The consumer makes the LLM call. The LLM's output, plus the intended side effects, are written transactionally to:

- The result table (the LLM's structured output, keyed by request ID).
- The outbox table (one row per intended downstream publish).

The publisher reads the outbox and publishes. The result table is the system of record; the outbox is the durable intent.

**Pattern.** Idempotent publish. The publisher includes the outbox row's ID in the published payload. Downstream consumers dedup on this ID. Even if the publisher publishes twice (e.g., it crashed between publish and outbox-mark-done), downstream sees one effective write.

**Pattern.** Outbox-as-audit. The outbox table is also the audit log. Once written, it's the durable record of "the LLM said X, and we intended to act on it." Useful for compliance and post-incident investigation.

### 6.3 The saga pattern for AI

When multi-step side effects must be coordinated (write to system A, then system B, then system C), saga is the standard. Each step is a service call; if any step fails, the prior steps' compensating actions are invoked.

**AI-step in saga.** When the AI is one step in a saga, the saga must understand:

- The AI step's success criteria. Did the LLM call succeed *and* produce parseable output? A 200 from the provider isn't enough; the output must be validated.
- The AI step's compensation. If the saga rolls back, what undoes the AI step? Often nothing — the LLM call's "side effect" is just an output (the side effect happens in the next step). But if the AI step itself wrote to a system (called a tool that has side effects), that write needs compensation.
- The AI step's idempotency. On saga retry, will the AI step re-call the LLM? If yes, costs and non-determinism issues. If no, what's the cache key?

**Pattern.** The AI step writes its result to a result store keyed by saga ID + step ID. On saga retry, the step checks the result store first; if present, reuse; if absent, call the LLM.

**Pattern.** AI steps with downstream side effects (tools) must run with an idempotency key that includes the saga ID + step ID. The downstream service dedups; the AI step is safe to retry.

### 6.4 The compensating-action pattern for AI

**Forward action.** Step writes to system A. Step succeeds.

**Compensating action.** If the saga aborts later, the compensating action for "write to system A" runs (e.g., delete the row, mark the write as canceled).

**AI-specific compensating actions.**

- **AI generated a draft document; downstream system stored it.** Compensation: mark the document as canceled or delete it. The LLM call's cost is sunk; the document is reversible.
- **AI sent a notification.** Compensation: usually impossible (the recipient has already received it). Often the compensating action is to send a follow-up notification with a correction.
- **AI updated a customer record.** Compensation: revert to the prior value. Requires the prior value to be captured before the update.

**Pattern.** Snapshot before write. Before any side effect, snapshot the prior state. The compensation restores the snapshot.

**Pattern.** Soft-state writes. Side effects are marked as "draft" or "pending" until the saga commits. Downstream consumers ignore draft state. On saga commit, the state is promoted to "committed."

### 6.5 The "wait, did the LLM call happen?" question

Network timeouts during an LLM call leave the consumer in a state where it doesn't know whether the call happened. If it happened, the consumer was charged. If it happened and produced output, the output is lost.

**Mitigation.** Idempotency on the provider side. Most providers support idempotency keys (an HTTP header) that dedup retries. The consumer generates the key per request; retries use the same key; the provider returns the cached result of the first call.

**Mitigation.** Optimistic record. Before calling the LLM, write a "call attempted" row. On timeout, the row is the record of the attempt; the consumer can query the provider's logs (where supported) to determine whether the call completed.

**Mitigation.** Accept the cost. For low-value calls, just retry and accept the occasional double-charge. For high-value calls ($0.40+), the idempotency-key approach is worth the setup cost.

---

## 7. Idempotency contracts at the integration boundary

Idempotency at integration boundaries is a contract between caller and callee. AI integration adds nuance.

### 7.1 What "idempotent" means at the integration boundary

The standard definition: a request can be safely retried; the same request produces the same effective state.

For AI calls, the same input doesn't produce the same output (temperature, non-determinism, model drift). So "same effective state" must mean *side effects* are deduplicated, not *output* is identical.

**Pattern.** Idempotency key in the request header. Caller generates a UUID per logical request. Retries reuse the UUID. The integration dedups: first request with a given key gets processed; subsequent requests with the same key return the first response without re-processing.

**Pattern.** Idempotency key from the caller's natural key. If the request is naturally keyed (e.g., "summarize document X" — the document ID is the natural key), use the natural key. Avoid hashing the entire request payload, which is fragile to formatting changes.

### 7.2 The dedup window

Idempotency dedup typically applies within a time window. After the window, the same key might be reused for a different request.

**Pattern.** Configurable window per use case. Long for one-shot operations (24 hours for "create this report"); short for high-volume operations (5 minutes for streaming chat).

**Pattern.** Store the idempotency key, the request hash, and the response together. On retry, verify the request hash matches; if not, the caller is reusing the key incorrectly (return an error). If matches, return the stored response.

### 7.3 Idempotency at boundary vs idempotency inside consumer

A common conflation. The integration boundary's idempotency is per *request from caller*. The consumer's idempotency (when calling the LLM provider) is per *attempt*.

**Caller-facing.** Caller submits request with idempotency key K. Caller retries with K. Integration returns the first attempt's result.

**Consumer-facing.** Integration processes request K. Integration calls LLM provider with idempotency key K' (different from K). If the LLM call fails and is retried, the same K' is reused. The provider dedups.

K and K' are distinct because:

- K is in the caller's namespace; K' is in the consumer's namespace.
- Caller's retry policy is different from consumer's retry policy.
- One caller request may generate multiple LLM calls (agent loop, multi-step pipeline); each gets its own K'.

Confusing them produces double-billing (consumer didn't dedup at provider) or double-processing (integration didn't dedup at caller boundary).

### 7.4 Replay-safe AI calls

Caller wants to replay the same request and get the same response (e.g., to render the same UI on page refresh). The integration must serve the cached response for the same idempotency key, even if days have passed.

**Pattern.** Long-window cache for replay. Store the response by idempotency key for a long window (days or weeks for archival). Replay hits the cache.

**Pattern.** Cache versioning. The cached response is tied to the model version, the prompt version, and the input. If any change, the cache miss triggers a fresh call. Replay still works (because the key includes versions), but historical responses are reproducible.

### 7.5 The "idempotent but expensive" trade-off

Full idempotency for high-cost LLM calls (especially agent loops) requires storing a lot of intermediate state. The trade-off:

- **Store everything.** Maximum idempotency; high storage cost; replay works at all granularities.
- **Store only inputs and outputs.** Moderate idempotency; lower storage cost; replay works at request level but not at step level.
- **Store nothing.** Cheapest; no idempotency; replay always re-runs.

The decision depends on the cost ratio: storage vs LLM call. For agent loops costing $1+ per call, storing intermediate state is cheaper than re-running.

---

## 8. Worked Meridian example

Meridian's Care Coordinator agent demonstrates the patterns. The agent does multi-step work; some steps have irreversible side effects (notifications to clinicians); some have reversible ones (drafts in the EHR). The architecture is designed so that mid-task failures don't produce orphaned state or duplicate notifications.

### 8.1 The agent's failure modes

A Care Coordinator task ("draft a referral letter for patient X to specialist Y") involves:

1. **Fetch patient context** (EHR read; idempotent; safe to retry).
2. **Fetch specialist directory entry** (API read; idempotent; safe to retry).
3. **Verify referral eligibility** (LLM call: checks insurance, prior authorization status; outputs a structured eligibility decision).
4. **If eligible: draft the referral letter** (LLM call; outputs structured letter content).
5. **Write the draft to EHR-staging** (database write; the draft is in "pending review" state).
6. **Notify the clinician for review** (queued notification; sent on the next polling cycle).

Failures observed in the first six months of production:

- Step 3 LLM call times out → unclear whether the eligibility check happened or not. Re-running risks re-running the eligibility check.
- Step 4 returns truncated output (max_tokens hit on a long referral case) → step 5 would write a truncated draft.
- Step 5 succeeds but the agent's runner crashes before step 6 → the draft is in EHR but no notification fires.
- Step 4 succeeds; step 5 fails due to EHR-staging being unavailable → no draft, no notification, but the LLM cost is sunk.
- Step 6's notification fires; the clinician reviews; clinician approves; the draft is committed; a duplicate of step 6 fires (notification retry) → clinician gets two notifications and is confused.

### 8.2 The architecture that handles them

**Idempotency layer (per task).** Every task has a `task_id` (UUID, caller-generated). All steps use this as the idempotency root. Step outputs are stored in a per-task result table keyed on `(task_id, step_id)`. On retry, the step checks the result table first; if a result exists for this `(task_id, step_id)`, use it; if not, run the step.

**Durable workflow framework.** The agent runs as a Temporal workflow. Each step is a Temporal activity. Temporal handles persistence, retries, and resume on crash.

**Outbox for irreversible side effects.** Step 6 (notify clinician) doesn't directly send the notification. It writes to an `agent_notifications` outbox table with the notification's payload, the `task_id`, and the `step_id`. A separate outbox-publisher polls the table and sends notifications via the notification service.

**Two-phase commit at workflow level.** The agent's steps 1-5 run in "draft" mode. The draft in EHR-staging is marked as "agent-draft, pending commit." Step 6 (notify) is the commit point — the notification is sent, the draft is promoted to "agent-draft, pending clinician review." Until the agent reaches step 6, the work is reversible (cancel the task → discard the draft → no notification fired).

**Provider-side idempotency on LLM calls.** Each LLM call (steps 3 and 4) includes an idempotency key derived from `task_id + step_id`. On retry, the provider returns the cached result of the first call; no double-billing.

**Structured-output validation.** Step 4's output is validated against a schema before step 5 acts on it. If validation fails (truncated, wrong shape), step 4 retries (up to 2 retries). If still failing, the agent fails the task with a structured error to the requester.

### 8.3 How each failure mode is now handled

- **Step 3 timeout:** Workflow retries step 3 with the same idempotency key. Provider returns the original result (charge once). Workflow continues.
- **Step 4 truncated output:** Schema validation fails; step 4 retries with a larger max_tokens or a different prompt template. If still failing after 2 retries, the agent fails the task; caller sees a structured error indicating "draft generation incomplete; please retry or contact support."
- **Step 5 success, runner crash before step 6:** Temporal resumes the workflow on a different worker. The result table shows step 5 succeeded; the workflow proceeds to step 6.
- **Step 4 succeeds, step 5 fails (EHR-staging unavailable):** Workflow retries step 5 (with exponential backoff). If still failing after the retry budget, the agent fails the task; the LLM output is stored in the result table for later replay; the workflow surfaces an error to the requester.
- **Step 6 fires twice:** Outbox row has a unique notification ID. The notification service dedups on this ID. Clinician sees one notification.

### 8.4 What this architecture costs

- Storage: each task generates 6 result-table rows + 1 outbox row + 1 EHR-staging draft. At 5000 tasks/day, ~40k rows/day; manageable.
- Latency overhead: Temporal adds ~100ms per step for activity scheduling and result persistence. End-to-end agent task latency: ~14 seconds (median); without Temporal, ~12 seconds. 2-second overhead for the durability.
- Operational complexity: Temporal cluster to operate; activity definitions to maintain; workflow versioning to manage. The team invested ~2 weeks in onboarding.

### 8.5 Results

- Zero orphaned EHR-staging drafts in the last 4 months (down from ~30/week before the outbox pattern).
- Zero duplicate clinician notifications in the last 4 months (down from ~10/week).
- Mean recovery time after a runner crash: 8 seconds (Temporal worker picks up the workflow on the next available worker).
- Cost overhead from idempotency-key support: 0% (provider was already supporting idempotency keys; the consumer just had to use them).
- LLM cost reduction from retry-with-idempotency-key: ~$300/month (avoided re-billing on retried timeouts; small but meaningful at scale).

### 8.6 The decision summary for the Care Coordinator

| Failure class | Strategy | Tool |
| --- | --- | --- |
| LLM call timeout | Retry with provider idempotency | Provider's idempotency-key header |
| Truncated output | Retry with adjusted prompt; fail with structured error if persistent | Schema validation |
| Multi-step partial completion | Retry-from-failure (Strategy C, §4.1) | Temporal workflow with per-step result store |
| Mid-task crash | Resume on different worker | Temporal |
| Reversible side effect (draft) | Two-phase: draft → commit | EHR-staging "agent-draft, pending" state |
| Irreversible side effect (notification) | Outbox + provider-side dedup | Outbox table + notification ID |
| Duplicate caller retry | Per-task idempotency | `task_id` as the boundary idempotency key |

---

## 9. Anti-patterns

### 9.1 The eight-second blocking call inside a 200ms SLO endpoint

**Pattern.** A request handler with a 200ms latency SLO synchronously calls an LLM that routinely takes 8 seconds. Under load, the endpoint blocks; the SLO is violated systematically.

**Corrective.** The LLM call must be moved to an async path (callback, polling, webhook). The 200ms endpoint returns an "in progress" response with a poll URL. Cross-link to [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md).

### 9.2 The retry-the-whole-conversation pattern

**Pattern.** Multi-step agent fails at step 5 of 7. Retry policy restarts the agent from step 1. Steps 1-4 re-run (expensive) before retrying step 5.

**Corrective.** Persist per-step state. Retry resumes from the failed step. Cross-link to [agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) in the engineering sibling.

### 9.3 The streaming response that swallows mid-stream errors

**Pattern.** Streaming response is in flight; mid-stream, an error occurs. The stream is closed silently. The caller sees a truncated response without an error indication.

**Corrective.** Streaming responses must propagate errors via an in-band error event (e.g., a final event with type "error" and a structured payload). The client must check for the error event before treating the stream as complete.

### 9.4 The "200 OK + body parse failure" silent corruption

**Pattern.** Consumer receives 200 OK; tries to parse JSON; parse fails; consumer logs an error and returns success to the caller (because "HTTP was successful").

**Corrective.** Parse failure is a failure. Structured-output validation must happen before returning to the caller. Failed parse → retry or structured error.

### 9.5 The hallucinated-tool-call execution

**Pattern.** LLM emits a tool call for a tool that doesn't exist or with arguments that don't match the schema. The integration's tool dispatcher attempts to find a "close match" and executes something.

**Corrective.** Strict schema validation. Unknown tool → reject with structured error to the agent. Schema mismatch → reject. No fuzzy matching at the integration boundary. Cross-link to [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md).

### 9.6 The silent model fallback

**Pattern.** Provider falls back from primary to secondary model under load; doesn't tell the caller; the caller's response is lower quality without metadata reflecting this.

**Corrective.** Read the response metadata (most providers expose the actual model used). Propagate it to the caller. Surface as a metric (per-call: which model actually served). Audit drift.

### 9.7 The compensation that nobody tested

**Pattern.** Saga has compensating actions defined in code. Compensation has never run in production (no failure has triggered it). First time it fires, it has bugs.

**Corrective.** Pre-production chaos testing. Inject failures at each saga step; verify compensation runs correctly. Cross-link to [reliability-engineering](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/reliability-engineering) in the engineering sibling.

### 9.8 The "we throw 500 and call it a day" exception handler

**Pattern.** Any LLM error becomes a 500 to the caller. The caller's UI shows a generic error. The customer's support team can't correlate to anything. No incident ID. No suggested action.

**Corrective.** Structured errors with classification, incident ID, suggested action, and metadata. Cross-link to §3.4.

---

## 10. Findings (sprint-assignable)

**ARCH-IFP-001 (P0). No structured-output validation at integration boundary.** Truncated or malformed LLM output reaches downstream consumers as "success." Add schema validation; fail with structured error on validation failure. Owner: AI platform.

**ARCH-IFP-002 (P0). No idempotency key at integration boundary.** Caller retry duplicates side effects. Define idempotency key per request (caller-supplied or derived from natural key); dedup at the boundary. Owner: API team.

**ARCH-IFP-003 (P0). LLM calls don't include provider idempotency key.** Retries double-bill. Pass provider-side idempotency key on every call; verify provider supports it. Owner: AI consumer team.

**ARCH-IFP-004 (P0). No outbox pattern for irreversible side effects.** Crashes between LLM call and side effect lose work or duplicate it. Implement outbox for downstream publishes / notifications / external API calls. Owner: AI platform.

**ARCH-IFP-005 (P0). Multi-step agent retries from step 1 on any failure.** Wasted LLM cost; slower recovery. Adopt durable workflow framework (Temporal / Step Functions / Inngest); persist per-step results; resume from failed step. Owner: agent platform.

**ARCH-IFP-006 (P1). No latency-budget cascade for LLM calls.** Caller's full budget consumed by one timeout; no headroom for retry. Define cascade per call class; LLM call gets bounded sub-budget. Owner: feature team + AI platform.

**ARCH-IFP-007 (P1). No fallback ladder when primary model unavailable.** First model outage causes customer-facing 500s. Define fallback ladder (Sonnet → Haiku → cache → templated → structured error); test under synthetic provider outage. Owner: AI platform.

**ARCH-IFP-008 (P1). Streaming responses don't propagate mid-stream errors.** Caller treats truncated stream as complete. Add in-band error event to streaming protocol; client parses for it. Owner: API team.

**ARCH-IFP-009 (P1). Hallucinated tool calls executed by integration dispatcher.** Confidently-stated wrong action runs. Add strict schema validation; reject unknown tools; reject schema mismatch. Cross-link [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md). Owner: agent platform.

**ARCH-IFP-010 (P1). No detection of silent model fallback.** Quality degradation invisible. Read response metadata; surface actual-model-used metric; alert on drift. Owner: AI platform + observability.

**ARCH-IFP-011 (P1). No structured-error response shape.** Caller's UI shows generic "an error occurred." Define structured error envelope (status, class, detail, incident_id, suggested_action); use everywhere. Owner: API team.

**ARCH-IFP-012 (P2). No compensation tests for saga flows with AI steps.** First failure is the first compensation test. Pre-production chaos tests; inject failures; verify compensation runs. Owner: QA + agent platform.

**ARCH-IFP-013 (P2). Two-phase commit not used for irreversible side effects.** Mid-task failure leaves partial irreversible state. Adopt two-phase commit at workflow level; irreversible steps deferred until final commit. Owner: feature teams.

**ARCH-IFP-014 (P2). No incident_id correlation between caller error and operator logs.** Customer-reported errors take hours to diagnose. Generate incident_id on error; surface to caller; log on operator side; document the correlation. Owner: SRE.

**ARCH-IFP-015 (P2). Dedup window for idempotency key not configured.** First reuse of key after window produces unexpected dedup or unexpected processing. Document dedup window per use case; verify caller behavior matches. Owner: API team.

**ARCH-IFP-016 (P3). No replay capability for high-value calls.** Customer can't view yesterday's response; UI re-runs the LLM call. Long-window idempotency-key cache; UI's "view history" hits the cache. Owner: feature team.

**ARCH-IFP-017 (P3). Per-call-class latency budgets not differentiated.** Short calls and long calls share the same timeout. Define per-call-class timeouts; cascade from caller budget. Owner: AI platform.

**ARCH-IFP-018 (P3). Refusal vs failure not distinguished in response shape.** "We declined" and "we crashed" look the same to caller. Differentiate in the structured error envelope; surface as separate metrics. Owner: API team + observability.

---

## 11. Adoption sequencing checklist

For a team adopting integration-failure patterns in an AI feature, in order:

- [ ] **Build the failure taxonomy (§2)** specific to this feature. Enumerate each failure class and what the integration should do.
- [ ] **Define the latency-budget cascade (§3.1).** Caller's budget → consumer overhead → LLM sub-budget → headroom for retry.
- [ ] **Set per-call-class timeouts.** Static timeouts where simplicity matters; adaptive where call-cost varies wildly.
- [ ] **Implement the fallback ladder (§3.3).** Primary → cheaper-model → cache → templated → structured error.
- [ ] **Choose partial-success strategy (§4.1) per workflow.** Document the choice; it's not platform-wide.
- [ ] **For multi-step workflows, adopt a durable workflow framework** (Temporal / Step Functions / Inngest). Persist per-step results.
- [ ] **For irreversible side effects, implement the outbox pattern (§6.2).** Downstream consumers dedup on outbox row ID.
- [ ] **For workflows with mixed reversibility, implement two-phase commit (§4.4).** Irreversible deferred until final commit.
- [ ] **Define idempotency contract at integration boundary (§7).** Caller-supplied key; dedup window; replay behavior.
- [ ] **Pass provider-side idempotency keys on every LLM call.** Verify provider supports it.
- [ ] **Implement structured-output validation at boundary (§5.3).** Validate every response before returning to caller.
- [ ] **Choose "model failed" response shape per call class (§5.2).** Document; surface in API docs.
- [ ] **Define structured error envelope (§3.4).** Include incident_id correlation.
- [ ] **Pre-production chaos test:** inject provider 5xx, network timeouts, truncated outputs, schema violations, mid-stream errors. Verify each is handled.
- [ ] **Audit production for silent model fallback.** Add actual-model-used metric; alert on drift.
- [ ] **Document the failure handling** in the API docs (what callers should expect on failure).

---

## 12. References

**In this folder.**
- [event-driven-ai-integration.md](./event-driven-ai-integration.md) — DLQ patterns and poison-pill handling for event-driven AI consumers (companion).
- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — integration-shape decision; failure modes differ per shape.
- [backpressure-and-queueing.md](./backpressure-and-queueing.md) — shed-load and rate-limit propagation; the "what do we return when we're full" question.
- [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md) — long-running async patterns; how partial-success and failure surface in async return paths.
- [tool-call-architecture.md](./tool-call-architecture.md) — tool dispatch boundary, where hallucinated tool calls must be rejected.
- [human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md) — where the human is the fallback for model failure.

**Elsewhere in this repo.**
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose failure patterns are described here.
- [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) — authorization layer that rejects hallucinated or unauthorized tool calls.
- [guardrails-and-policy-architecture/refusal-and-escalation-design.md](../guardrails-and-policy-architecture/refusal-and-escalation-design.md) — refusal as a first-class response shape.
- [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) — audit trail that includes "the LLM said X, and we acted on it."

**Sibling repos.**
- [ai-engineering-reference-architecture / reliability-engineering / fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) — engineering-side implementation of fallback ladder.
- [ai-engineering-reference-architecture / agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — agent-internal partial-failure recovery.
- [ai-engineering-reference-architecture / observability-and-telemetry / debugging-from-traces.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/debugging-from-traces.md) — incident-correlation patterns.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — adversarial inputs that produce integration failures (prompt injection, oversized inputs, malformed schemas).

**External.**
- *Microservices Patterns* (Richardson, 2018) — saga, outbox, dual-write patterns; the AI overlays here sit on top.
- *Release It!* (Nygard, 2018) — circuit-breaker, bulkhead, timeout patterns; conventional reliability foundations.
- Stripe API documentation — idempotency-key as integration contract; a strong reference for the pattern.
- Temporal documentation — durable workflow patterns; the canonical implementation for multi-step durable execution.
- AWS Step Functions documentation — saga and compensation patterns; an alternative durable workflow framework.
