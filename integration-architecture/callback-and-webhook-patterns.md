# Callback and Webhook Patterns

> **Audience.** Architects designing how long-running AI work returns to the caller. Tech leads whose agent task takes 40 seconds but the request handler times out at 30. Anyone whose AI feature produces results that multiple consumers need to act on later. **Scope.** The *architectural* decision: the four return-path patterns (callback URL, webhook, polling endpoint, durable workflow); callback URL design (signing, replay protection, retry semantics); webhook patterns for AI (signed payloads, idempotency keys, subscriber design); polling endpoints (when polling beats callbacks, completion-status schema); durable workflow frameworks (Temporal, Step Functions, Inngest, durable function frameworks); choosing among the four. Not the integration-shape decision itself (see [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md)). Not the event-driven pub/sub patterns for AI consumers (see [event-driven-ai-integration.md](./event-driven-ai-integration.md)). Not the human-in-the-loop boundary (see [human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Most production AI work doesn't fit request/response. The reasons are familiar:

- The work takes longer than the request timeout. Agent loops, multi-step pipelines, long-context summarization, batch enrichment — typical run times measured in tens of seconds, sometimes minutes.
- The caller doesn't need the result immediately. Bulk classification jobs run overnight; embedding pipelines run hourly; the user who submitted the request has moved on.
- Multiple consumers need the result. The completion of an AI task should reach the originating UI, an audit log, a downstream workflow, and an analytics warehouse — five places, one event.
- The work is durable: it must complete even if the originating connection drops, the originating user logs out, or the originating service restarts. Holding the result on a synchronous connection is fragile.

For all of these, the integration needs a *return path* that's decoupled from the original request. The caller submits work; the result arrives later, through a different channel. The choice of channel is architectural.

There are four channels: callback URL (server pushes result to a URL the caller supplied), webhook (server pushes to URLs the caller registered ahead of time), polling endpoint (caller asks "is it done yet?"), and durable workflow (the work is itself the integration; framework handles the wait/notify).

Each has trade-offs across delivery reliability, infrastructure overhead, caller complexity, and operational properties. None is universally right. The architectural decisions are:

- Which channel for which work? The choice is per use case, not per platform.
- How is delivery secured? Signed payloads, mutual TLS, replay protection.
- How is duplicate delivery handled? Idempotency keys; consumer dedup.
- How is failed delivery handled? Retry policy, dead-letter, eventual consistency vs strict.
- How does the system survive the receiver being slow or down? Buffer, backoff, give up.

These decisions compound. Cheap mistakes early (e.g., unsigned webhooks; "we'll add signing in v2") become expensive later (security incident, customer trust loss). Mature architectures think through them at design time.

This document is opinionated about four things:

1. **The default for non-trivial agent work is durable workflow.** Temporal, Step Functions, Inngest, durable function frameworks — these solve the multi-step, long-running, observable-on-replay problem cleanly. Bolting callbacks onto an HTTP API for agent work usually produces something that works for the demo and breaks at scale.
2. **Webhooks must be signed from day one.** Unsigned webhooks are not webhooks; they're an unauthenticated push to an attacker-discoverable URL. Adding signatures later requires every consumer to deploy verification, simultaneously, while the producer is still accepting unsigned traffic.
3. **The receiver's backpressure is the producer's responsibility.** Webhook delivery storms during a receiver outage are a producer-side architectural failure, not a receiver-side one. The producer must respect receiver 429s, back off, and surface delivery-failure to the receiver's subscription manager.
4. **Polling and callbacks both exist; neither is universally better.** Polling beats callbacks when the receiver can't accept inbound (NAT, firewall), when the receiver is high-volume (callbacks would swamp it), or when caller-controlled timing matters. Callbacks beat polling when freshness matters and the receiver is webhook-capable. The choice is a property of the receiver, not the producer's preference.

Structure: (2) the four return-path patterns; (3) callback URL design; (4) webhook patterns for AI; (5) polling endpoints; (6) durable workflow frameworks; (7) choosing among the four; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four return-path patterns

Four channels cover the space. Each has a canonical shape.

### 2.1 Callback URL

Caller submits work and supplies a URL to call back to:

```
POST /jobs
Body: { task: "summarize", payload: {...}, callback_url: "https://caller.example.com/result/abc123" }

Response: 202 Accepted
Body: { job_id: "job-xyz", estimated_seconds: 30 }

... later ...

POST https://caller.example.com/result/abc123
Body: { job_id: "job-xyz", status: "completed", result: {...} }
```

**Properties.**
- One-off subscription. Each request has its own callback URL.
- Caller controls the URL (different URLs per request, different teams own different endpoints).
- No subscription state on the producer side beyond the in-flight job.
- Delivery happens once; if the receiver is down at delivery time, the work might be lost (or retried, depending on producer policy).

**When right.** Per-request return paths. Different requests need different return endpoints. The caller is responsible for its own routing.

**When wrong.** Long-lived subscriptions ("send me every result of this type"). The webhook pattern fits that better.

### 2.2 Webhook

Caller registers a long-lived subscription. Producer delivers events matching the subscription to the registered URL:

```
POST /subscriptions
Body: { event_types: ["task.completed", "task.failed"], url: "https://caller.example.com/webhooks/ai" }

... later, on every matching event ...

POST https://caller.example.com/webhooks/ai
Body: { event_type: "task.completed", payload: {...}, signature: "...", timestamp: "..." }
```

**Properties.**
- Long-lived subscription. One URL receives many events over time.
- Subscription state on the producer side (subscribers table, event types, delivery preferences).
- Delivery has retry semantics (producer attempts delivery N times, dead-letters after failure).
- Often combined with signing (HMAC of payload + secret).

**When right.** Long-running integrations where the receiver wants every event of a class. Cross-system integration (Stripe → CRM, GitHub → CI, AI platform → downstream consumers).

**When wrong.** One-off per-request return paths. Use callback URL.

### 2.3 Polling endpoint

Caller submits work; receives a job ID; polls a status endpoint until the job completes:

```
POST /jobs
Body: { task: "summarize", payload: {...} }

Response: 202 Accepted
Body: { job_id: "job-xyz", poll_url: "/jobs/job-xyz/status", poll_after_ms: 5000 }

... caller polls ...

GET /jobs/job-xyz/status
Response: { status: "running", estimated_remaining_seconds: 15 }

... eventually ...

GET /jobs/job-xyz/status
Response: { status: "completed", result: {...} }
```

**Properties.**
- Caller-controlled timing. Caller polls when it wants.
- No inbound network requirement on caller's side. Works through NATs, firewalls, restrictive networks.
- Producer state lives on producer's side; caller just holds a job ID.
- Higher cumulative request volume (every poll is a request) but each request is small.

**When right.** Caller can't or won't accept inbound (no public endpoint, behind firewall). Caller-controlled timing matters. Result freshness can tolerate poll interval.

**When wrong.** Many concurrent callers polling frequently — produces a load multiplier on the producer.

### 2.4 Durable workflow framework

The work itself is a durable workflow; the framework owns the entire lifecycle including the wait/notify pattern:

```python
@workflow.defn
class SummarizeTask:
    @workflow.run
    async def run(self, task_input: dict) -> dict:
        context = await workflow.execute_activity(fetch_context, task_input["document_id"])
        summary = await workflow.execute_activity(call_llm_summarize, context)
        await workflow.execute_activity(notify_downstream, summary)
        return summary
```

The framework (Temporal, Step Functions, Inngest, etc.) handles:
- Persisting workflow state across crashes and restarts.
- Retrying failed activities per-policy.
- Resuming after worker failure on a different worker.
- Exposing workflow state for query (caller can poll for status, see step-by-step progress).
- Triggering downstream signals/events on completion.

**Properties.**
- Strongest durability guarantees. Work survives any single-node failure.
- Strongest observability. The framework exposes step-by-step state.
- Heaviest operational footprint. The framework is a system in itself; cluster to operate, SDK to learn.
- The "return path" is whatever the workflow does at completion: trigger a webhook, write to a result store, signal a waiting client.

**When right.** Multi-step AI work (agents, multi-LLM pipelines). Long-running work where mid-task failure must be recoverable. Workloads where step-by-step observability matters.

**When wrong.** Single-step, short, low-value work. The framework overhead doesn't pay off for a 200ms classification call.

### 2.5 The patterns compose

Real systems usually combine. A durable workflow runs the work; on completion, it triggers a webhook to subscribers; subscribers may have come from a polling-style "where is my job" query earlier. Meridian's Care Coordinator uses three of the four patterns in different parts of the system.

---

## 3. Callback URL design

The single-shot return path. The architectural decisions are about delivery reliability and security.

### 3.1 The signing requirement

The callback URL is supplied by the caller; the producer hits it later. By that time, anyone with the URL can also hit it. Without signing, an attacker who learns a callback URL (network sniffing, log access, social engineering) can deliver fake results.

**Pattern.** HMAC signature over payload + timestamp.

```
POST <callback_url>
Headers:
  X-Signature: sha256=<HMAC>
  X-Timestamp: 1716840000
Body: {...}
```

HMAC is computed with a shared secret. The shared secret is established at job submission time:

```
POST /jobs
Body: { ..., callback_url: "...", callback_secret: "<random>" }
```

The caller stores `(job_id → callback_secret)`. On callback, the receiver looks up the secret by job_id, recomputes the HMAC, compares.

**Alternative.** Producer-owned signing key. The producer signs every callback with its own private key (asymmetric). The receiver verifies with the producer's public key (well-known, published). No per-request secret to share. Lower key-management overhead for the caller; the same key is reused for all jobs.

**Choice.** Per-request secret when callers are external and identity is per-request. Producer-owned key when callers are internal and a single trust relationship is established.

### 3.2 Replay protection

A signed payload is still replayable. An attacker who captures a valid callback can replay it later, potentially confusing the receiver.

**Pattern.** Timestamp + window. The callback includes a timestamp; the receiver rejects callbacks older than N minutes. Limits the replay window.

**Pattern.** Nonce / job_id uniqueness. The receiver tracks delivered callbacks by job_id; duplicate delivery (whether intentional replay or producer retry) is dedup'd at the receiver.

**Pattern.** TLS pinning or mTLS. The receiver only accepts callbacks from the producer's specific TLS certificate. Limits which network endpoints can deliver.

### 3.3 Delivery retry semantics

The receiver might be down when the producer attempts delivery. The producer's policy:

**No retry.** Deliver once; on failure, log and give up. The receiver must poll if it needs the result.

- Simple; no producer-side retry state.
- Receivers must defensively poll anyway in case of missed delivery.
- Acceptable when the work isn't critical or when the receiver has a polling fallback.

**Retry with backoff.** Deliver; on failure, retry with exponential backoff up to a maximum.

- Producer holds retry state per job.
- Receiver doesn't need to poll defensively.
- Standard pattern; matches what webhooks (§4) typically do.

**Retry until acknowledged with TTL.** Deliver; retry indefinitely until receiver acknowledges or until job TTL expires.

- Strongest delivery guarantee; high producer-side state.
- Useful when the receiver is critical and TTL is short (e.g., 1 hour).

The choice depends on how critical the callback is to the receiver, and how long the producer is willing to hold state.

### 3.4 Timeout and abandonment

A receiver that never responds (TCP connection hangs, application doesn't ack within a window) consumes producer resources.

**Pattern.** Hard timeout per delivery attempt. 30 seconds is a reasonable default; the receiver should ack quickly (process async after ack if needed). Beyond timeout, the delivery is considered failed and (if retrying) requeued.

**Pattern.** Dead-letter after N retries. The job moves to a "delivery failed" state; an operator can investigate or replay manually.

### 3.5 The "callback URL goes stale" problem

The caller submitted a job with `callback_url=https://caller.example.com/result/abc123`. Two hours later, the caller has redeployed; the URL no longer routes. The producer attempts delivery and gets 404.

**Mitigation.**

- Polling fallback. The caller can poll for the result if the callback failed; the producer must hold the result for a TTL.
- Subscription update. The caller can update the callback URL post-submission; the producer uses the latest URL on delivery.
- TTL on the result. If the receiver hasn't acked within N hours, the producer discards the result. Document the TTL.

### 3.6 The "caller's URL must be publicly reachable" assumption

Callbacks require the caller to host an inbound endpoint. For SaaS-to-SaaS integration this is fine; for enterprise-behind-firewall it's often not.

**Mitigation.**

- Use polling for receivers that can't accept inbound.
- Use a webhook relay service (the producer pushes to a relay; the receiver polls the relay).
- Use a long-poll endpoint (the receiver holds a connection open; the producer pushes through it).

---

## 4. Webhook patterns for AI

Long-lived subscriptions. The architectural decisions are about subscription management, delivery reliability, and security at scale.

### 4.1 Subscription management

Webhooks need a subscription model: which events go to which URLs.

**Pattern.** Per-subscriber subscription record:

```
{
  subscriber_id: "sub_abc",
  url: "https://example.com/webhooks/ai",
  event_types: ["task.completed", "task.failed"],
  secret: "<random>",
  active: true,
  created_at: "...",
  delivery_stats: { ... }
}
```

Producers store subscription records; on every event, the matching subscribers' URLs receive the payload.

**Pattern.** Filtering at subscription. Subscribers can filter by event attributes (e.g., only events for tenant T, or only events with classification = critical). Reduces unnecessary delivery.

**Pattern.** Lifecycle management. Subscriptions can be paused, updated, or deleted. A subscriber that's been failing for too long is auto-paused.

### 4.2 Signing at scale

Same as callback URL signing (§3.1) but with one shared secret per subscription. The signing pattern is standardized across the producer; the secret rotation is per-subscription.

**Pattern.** Secret rotation. The producer supports two valid secrets per subscription (current + previous); the subscriber rotates the secret via an API; old signatures are accepted for a grace window.

**Pattern.** Algorithm versioning. The signature header includes the algorithm: `X-Signature: sha256=...` allows future upgrade to a different algorithm without breaking subscribers.

### 4.3 Delivery reliability at scale

A producer with thousands of subscribers delivers many events. Reliability requires:

**Pattern.** Per-subscription delivery worker. Each subscription has its own delivery queue. A slow subscriber doesn't block others.

**Pattern.** Per-subscription circuit breaker. If a subscriber's delivery success rate drops below threshold, pause delivery to that subscriber for a window; notify them via a different channel.

**Pattern.** Exponential backoff with jitter. Retries spread out so retry storms don't synchronize.

**Pattern.** Delivery audit log. Every delivery attempt (success or failure) is logged. The subscriber can query their delivery log to investigate missed events.

### 4.4 At-least-once delivery + receiver-side idempotency

Webhook delivery is at-least-once. The receiver must dedup.

**Pattern.** Event ID in the payload. Receiver tracks processed event IDs; duplicate delivery is recognized and ignored.

**Pattern.** Idempotency key header:

```
X-Event-Id: evt_abc123
```

Receiver dedups on this key.

**Pattern.** Document the at-least-once contract in webhook docs. "You may receive the same event more than once; dedup using `event_id`."

### 4.5 Receiver-side backpressure

The receiver may be slow. Webhook delivery storms during receiver overload can cascade:

**Pattern.** Receiver returns 429 with `Retry-After`. The producer respects the header; slows delivery to this subscriber; resumes when the timeout expires.

**Pattern.** Receiver returns 503 or 502. The producer treats as transient; retries with backoff.

**Pattern.** Receiver returns 4xx (other than 429). The producer treats as permanent failure for this event; doesn't retry. Surfaces to subscription's delivery log.

**Pattern.** Producer-side rate limit per subscriber. Even without receiver-side signals, the producer can cap delivery rate per subscriber (e.g., 100 events/second max) to prevent overwhelming slow consumers.

### 4.6 Webhook payload design for AI events

**Decision.** Fat or thin payload (analogous to §6.2 in [event-driven-ai-integration.md](./event-driven-ai-integration.md)).

- **Fat:** the full LLM output, structured data, metadata, all in the payload.
- **Thin:** event metadata + ID, receiver fetches via callback URL.

**Recommendation.** Hybrid. The payload includes the structured summary (classification, key fields, status); large content (full document text, embedding vectors) is referenced by URL or ID.

**Pattern.** Schema versioning. The event payload has a `schema_version` field. Subscribers know which version they're parsing. Breaking changes require a new version; the producer can deliver multiple versions to subscribers that prefer different versions (during migration).

**Pattern.** PII filtering. Webhook payloads cross network boundaries; PII must be filtered at the producer. Subscribers that need PII must fetch from an access-controlled store, not receive in the webhook.

### 4.7 The webhook-as-event-bus anti-pattern

Some teams use webhooks as a poor man's event bus: every consumer registers a webhook for every event type; producer fans out to all.

**Why it's wrong.** Webhooks are designed for one-or-few external consumers, not for fan-out to many internal consumers. The latter is what event-driven architecture (Kafka, SNS+SQS, EventBridge) is for.

**Corrective.** For internal fan-out, use the event-driven patterns ([event-driven-ai-integration.md](./event-driven-ai-integration.md)). For external integration, use webhooks. Don't conflate.

---

## 5. Polling endpoints

The receiver-driven return path. Polling is often denigrated as "wasteful" but is the right answer in several common cases.

### 5.1 When polling beats callbacks

- **Receiver can't accept inbound.** Behind a firewall, NAT, mobile network without persistent connection. Polling works; callbacks don't.
- **Receiver-controlled timing matters.** Receiver wants to check at a specific time (e.g., page load, user action). A callback fires when ready, possibly when the UI isn't visible.
- **Receiver is high-volume.** A receiver getting 100k webhooks/sec is operationally complex; polling on demand may be lighter.
- **Caller doesn't want to manage subscriptions.** Polling has no subscription state; caller just submits and polls.

### 5.2 Completion-status schema

Standard polling response:

```json
{
  "job_id": "job-xyz",
  "status": "running" | "completed" | "failed" | "canceled",
  "progress": {
    "percent_complete": 60,
    "current_step": "extracting_findings",
    "steps_completed": 3,
    "total_steps": 5,
    "estimated_remaining_seconds": 12
  },
  "result": null,  // populated when status == "completed"
  "error": null,   // populated when status == "failed"
  "poll_after_ms": 5000
}
```

**Pattern.** Status field, not HTTP status code, indicates job state. HTTP 200 always; status field tells caller what happened.

**Pattern.** Progress is optional but valuable for long-running jobs. UI can render a progress bar.

**Pattern.** `poll_after_ms` is producer-recommended next-poll delay. Allows producer to throttle aggressive pollers gracefully.

### 5.3 Polling cadence and backoff

Naive polling: caller polls every second. Producer gets DDoSed during peak.

**Pattern.** Producer-recommended cadence. The response includes `poll_after_ms`; caller respects it.

**Pattern.** Exponential backoff. Caller starts at short interval (200ms) and backs off if status remains "running" (1s, 5s, 30s).

**Pattern.** Cap on polling duration. After N minutes without completion, caller stops polling and considers the job lost (or falls back to callback registration).

**Pattern.** Adaptive cadence. Caller's polling interval adjusts based on `estimated_remaining_seconds`. If estimated remaining is 30 seconds, poll every 5 seconds; if estimated is 5 minutes, poll every 30 seconds.

### 5.4 Server-sent events (SSE) as an alternative

Polling alternative: the server holds a connection open and streams status updates as they happen.

**Pattern.** Receiver opens GET to a streaming endpoint. Server emits SSE events (status updates, progress, completion). Receiver consumes the stream.

**Pros.** Real-time updates without polling. Lower latency for completion notification.

**Cons.** Requires connection-holding infrastructure. Less suited for very long jobs (connection life). Same NAT/firewall concerns as callbacks.

**When right.** Interactive UIs where the user is actively watching. The completion latency matters; the connection is short (seconds to minutes).

**When wrong.** Hours-long jobs. Receivers behind aggressive proxies that close idle connections.

### 5.5 The result store and TTL

After completion, the result is held server-side for some TTL so the receiver can fetch it.

**Pattern.** TTL of 24 hours by default; configurable per job. Receivers fetch within TTL; older results purged.

**Pattern.** Result location. Small results in the polling response; large results in a separate object store with a pre-signed URL.

**Pattern.** Replay endpoint. If receiver wants to fetch the result again (e.g., user refreshed page), polling endpoint returns the cached result until TTL.

---

## 6. Durable workflow frameworks

The pattern that solves the hard problems (multi-step, long-running, resumable on crash) at the cost of operational complexity.

### 6.1 What the framework provides

- **Workflow persistence.** The workflow's state (current step, intermediate results, history) is durably stored. A worker crash doesn't lose state; another worker resumes.
- **Activity execution.** Workflow steps run as activities, scheduled and tracked by the framework. Activity failures trigger configured retry policies.
- **Determinism guarantees.** Workflow code is deterministic; the framework can replay history to reconstruct state. Side effects must go through activities.
- **Observability.** The framework's UI shows workflow state, activity history, retry counts. Debugging is much easier than custom orchestration.
- **Signals and queries.** External code can signal a workflow (e.g., "user approved") or query its state.

### 6.2 The major frameworks

**Temporal.** Open-source; cloud-hosted variant (Temporal Cloud); strong durability and observability; rich SDK in multiple languages. The most full-featured choice. Requires Temporal cluster (self-host or cloud).

**AWS Step Functions.** Managed; tight AWS integration; state-machine model (JSON or Amazon States Language); good for AWS-native workloads. Less ergonomic for complex agent loops.

**Inngest.** Managed; serverless; webhook-based event triggers; lower operational overhead than Temporal; less rich for very complex workflows.

**Restate.** Open-source; lightweight; durable functions model. Newer; less mature ecosystem than Temporal but lower overhead.

**Cloudflare Workflows.** Managed; integrates with Cloudflare Workers; suitable for Workers-based stack.

**Choosing.** Temporal if maximum flexibility and complex workflows; Step Functions if AWS-native; Inngest if low operational overhead and webhook-driven triggers; Cloudflare Workflows if already on Cloudflare.

### 6.3 The agent-as-workflow pattern

Long-running agent work fits cleanly into the workflow model.

```python
@workflow.defn
class CareCoordinatorTask:
    @workflow.run
    async def run(self, task: CareTask) -> CareResult:
        context = await workflow.execute_activity(
            fetch_patient_context, task.patient_id,
            retry_policy=RetryPolicy(max_attempts=3)
        )

        eligibility = await workflow.execute_activity(
            check_eligibility, task, context,
            retry_policy=RetryPolicy(max_attempts=2)
        )

        if not eligibility.is_eligible:
            await workflow.execute_activity(
                notify_ineligible, task, eligibility.reason
            )
            return CareResult(status="ineligible", reason=eligibility.reason)

        draft = await workflow.execute_activity(
            draft_referral, task, context,
            retry_policy=RetryPolicy(max_attempts=2),
            heartbeat_timeout=timedelta(seconds=60)
        )

        await workflow.execute_activity(
            write_to_ehr_staging, draft
        )

        await workflow.execute_activity(
            notify_clinician_for_review, task, draft
        )

        clinician_decision = await workflow.wait_condition(
            lambda: self.clinician_decision is not None,
            timeout=timedelta(hours=48)
        )

        if clinician_decision == "approved":
            await workflow.execute_activity(commit_referral, draft)
        else:
            await workflow.execute_activity(cancel_draft, draft)

        return CareResult(status=clinician_decision)
```

Each step is an activity; activity retries are independent; the wait-for-clinician step survives worker restarts.

### 6.4 The return path from a workflow

When the workflow completes, the result must reach the caller. Several patterns:

**Signal a waiting client.** The client opened a long-poll or SSE connection; the workflow's completion signals it.

**Write to a result store; client polls.** The workflow's completion writes to a result store; the client polls. Acts like the polling pattern (§5).

**Trigger a webhook.** The workflow's completion triggers a webhook delivery to subscribers. Acts like the webhook pattern (§4).

**Direct callback.** The workflow's completion calls a callback URL the client provided at submission. Acts like the callback pattern (§3).

The workflow framework is orthogonal to the return-path choice; it owns the work, and a step in the workflow handles delivery.

### 6.5 The operational cost

- **Cluster operation.** Self-hosted Temporal requires a Temporal cluster (Cassandra/PostgreSQL backend, history service, worker pool). Operational footprint comparable to running a Kafka cluster.
- **SDK adoption.** Developers learn the SDK's idioms (deterministic workflow code, side effects through activities). Onboarding is ~1-2 weeks.
- **Versioning workflows.** Long-running workflows survive code deploys; if the workflow code changes mid-run, the framework needs versioning policy. Adds complexity to deploy practices.
- **Cost.** Managed services (Temporal Cloud, Step Functions, Inngest) charge per execution. For high-volume workloads, cost can be material; do the math before committing.

For multi-step AI work, the cost is usually justified by the operational properties. For single-step short work, it isn't; use a simpler pattern.

---

## 7. Choosing among the four

The choice is per use case. The decision tree:

### 7.1 The decision tree

**Step 1: Is the work multi-step or long-running (> 30 seconds expected)?**

- **Yes.** Use durable workflow framework. Other patterns can wrap it (workflow completion triggers webhook, etc.).
- **No.** Continue.

**Step 2: Is the receiver one-or-few external integrations, or a single-request return path?**

- **Single-request return path.** Use callback URL.
- **One-or-few long-lived external integrations.** Use webhook.
- **Continue.**

**Step 3: Can the receiver accept inbound HTTP?**

- **No.** Use polling endpoint.
- **Yes.** Continue.

**Step 4: How does the receiver prefer to be notified?**

- **Real-time, low-latency.** Use callback URL or webhook.
- **Receiver-controlled timing.** Use polling endpoint.
- **Either is fine.** Polling is operationally simpler; pick that.

### 7.2 The choices compose

A workflow framework usually owns the work; one of the other three patterns delivers the result.

| Use case | Work runs as | Result delivery |
| --- | --- | --- |
| User-facing chat | Direct LLM call (synchronous; no return path) | Streaming response |
| Document classification batch | Durable workflow | Webhook to downstream |
| Care Coordinator agent | Durable workflow | Webhook to clinician portal + audit + EHR (§2.5 fan-out) |
| Bulk embedding | Durable workflow | Polling endpoint for status; webhook on completion |
| Background enrichment for analytics | Direct call from event consumer | None (direct write to warehouse) |
| Single-shot summarization with caller-supplied callback | Durable workflow | Callback URL provided at submission |

### 7.3 The "use the simpler pattern" default

When the choice is genuinely close, prefer the simpler pattern:

- Polling over callbacks (no producer-side delivery infrastructure).
- Callbacks over webhooks (no subscription management).
- Direct call over workflow (no framework overhead).

Complexity must justify itself. If a single-shot 5-second LLM call works fine synchronously, don't put it in a workflow.

---

## 8. Worked Meridian example

Meridian's Care Coordinator long-running agent task uses three of the four return-path patterns in different parts of the system.

### 8.1 The task lifecycle

A clinician submits a care-plan-draft request through the web portal. The task:

1. Web portal POSTs to the Care Coordinator API. Request includes patient ID, care goals, and a webhook URL for completion notification.
2. API enqueues a durable workflow. Returns 202 Accepted with `job_id` and `poll_url`.
3. Workflow runs (typically 20-90 seconds). Web portal polls `poll_url` every 5 seconds (caller-supplied poll interval; respects `poll_after_ms`).
4. On workflow completion: writes draft to EHR-staging; sends webhook to portal (which dismisses the polling); fires webhook to audit log subscriber; fires webhook to compliance archive subscriber.

### 8.2 The architecture

```
Web Portal                                          Backend
    │                                                  │
    │  POST /care-tasks                                │
    │  { patient_id, goals, callback_url }             │
    ├──────────────────────────────────────────────────►│
    │                                                  │  ┌──────────────────┐
    │                                                  ├──► Temporal workflow │
    │  202 Accepted { job_id, poll_url }               │  │  (Care Coord.    │
    │◄─────────────────────────────────────────────────┤  │   agent)         │
    │                                                  │  └─────────┬────────┘
    │  GET /care-tasks/{job_id}/status                 │            │
    ├──────────────────────────────────────────────────►│           │
    │  { status: "running", percent: 45, ... }         │            │
    │◄─────────────────────────────────────────────────┤            │
    │                                                  │            │
    │  ... polling continues ...                       │            │
    │                                                  │            │
    │                                                  │  ┌─────────▼────────┐
    │                                                  │  │ Workflow completes│
    │                                                  │  │ Triggers webhook  │
    │                                                  │  │ delivery service  │
    │                                                  │  └─────────┬────────┘
    │                                                  │            │
    │                                  ┌───────────────┼────────────┤
    │                                  │               │            │
    │  POST <callback_url>             │               │            │
    │  { status: "completed", ... }    │               │            │
    │◄─────────────────────────────────┤               │            │
    │                                                  │            │
    │                                          POST <audit-url>     │
    │                                          { ... }              │
    │                                                  │            │
    │                                          POST <compliance-url>│
    │                                          { ... }              │
```

### 8.3 The pattern choices and why

**Durable workflow (Temporal) for the agent.** Multi-step (fetch context → check eligibility → draft → write → notify), long-running (median 35s, P99 90s), must survive worker crashes (clinician would notice if their request died silently). Temporal is the right tool; cross-link to §6.3.

**Polling endpoint for the originating portal.** The portal is the originating client; it knows the job_id; it wants to render a "drafting your care plan..." spinner with progress updates. Polling fits cleanly:

- Portal polls every 5 seconds (server-suggested `poll_after_ms`).
- Polling response includes percent_complete and current_step (workflow exposes this via Temporal query API).
- When status changes to "completed", the polling response includes the result; portal dismisses polling.

**Callback URL to the portal as a backup.** The portal submitted with `callback_url`. If the polling client misses the completion (browser tab closed, network blip), the callback delivers the completion asynchronously. The portal's callback endpoint stores the result; when the user returns to the page, the stored result is rendered.

**Webhook to audit and compliance subscribers.** Audit log and compliance archive are long-lived subscribers; they want every completion event, regardless of which clinician initiated. They subscribe once; receive every event. Webhooks fit; callbacks would mean per-request subscription, which is wrong for a long-lived integration.

### 8.4 The reliability properties

**If the workflow worker crashes mid-task:** Temporal resumes on a different worker. The polling endpoint shows "still running" longer than expected but eventually completes. Portal still gets the result via polling (or callback). Subscribers still get webhook on completion.

**If the portal's callback endpoint is down:** The portal still has polling open; the portal still sees the result via polling. The callback delivery fails and retries with backoff; eventually the callback delivery DLQs (logged for investigation) — but the user's experience is unaffected.

**If an audit log subscriber is down:** Webhook delivery to that subscriber retries with backoff; eventually the subscription's circuit breaker opens; an alert fires; the operator investigates. Other subscribers (compliance archive) and the portal callback are unaffected.

**If the polling endpoint is overloaded:** Each polling response includes `poll_after_ms`; the polling endpoint can dynamically increase this to throttle. Portal respects it. No DDoS pattern.

**If the clinician's webhook subscription URL becomes invalid:** Delivery fails; producer retries; eventually marks the subscription as failing; operator investigates; the receiver-side team is notified.

### 8.5 What the architecture costs

- **Temporal cluster:** 3-node deployment (managed via Temporal Cloud in production). Cost: ~$2000/month for the cluster + workflow execution cost.
- **Webhook delivery service:** a small consumer pool that drains the webhook outbox. Operational overhead: shared with other subscription-based features.
- **Polling endpoint:** standard HTTP endpoint backed by the result store (DynamoDB; sub-millisecond lookups by job_id).
- **Total ops cost:** ~$3000/month for the infrastructure; ~0.5 SRE FTE for ongoing maintenance.

### 8.6 What the architecture prevents

- **Lost work from worker crashes:** zero in the last six months (Temporal handles).
- **Lost completions from receiver outages:** zero (multiple delivery paths; outbox-based webhook).
- **DDoS by aggressive polling clients:** zero (server-suggested cadence is respected; would be enforced at the gateway if not).
- **Webhook delivery storms during receiver outage:** mitigated by per-subscription circuit breaker.

### 8.7 The migration story

The original architecture (before this design):

- The agent ran inside the API handler. 30-second timeout meant 35% of tasks failed mid-run.
- No polling; portal showed a spinner that eventually timed out.
- Notification to downstream services was a synchronous call from the agent; failures of downstream blocked the agent's completion from the portal's perspective.
- Audit was a separate process that scanned the DB every 5 minutes; latency to audit was 5-10 minutes; missed entries were not uncommon.

Migration timeline:

- Week 1: introduced Temporal; agent code refactored to workflow + activities. Test in staging.
- Week 2: rolled out to production for new tasks; existing tasks continued in old path. Verified.
- Week 3: deprecated old path.
- Week 4: added webhook delivery for audit and compliance.
- Week 5: portal updated to use polling + callback.

The migration was ~5 weeks total. The before/after task success rate: 65% → 99.7%. The latency-to-audit: 5-10 minutes → <500ms (audit subscribes to webhook).

---

## 9. Anti-patterns

### 9.1 The unsigned webhook

**Pattern.** Webhook delivery has no signature. Receiver verifies "is this from us" by checking nothing. An attacker who learns the URL can deliver fake events.

**Corrective.** Signing from day one (§4.2). HMAC over payload + timestamp; receiver verifies.

### 9.2 The polling client that polls every 100ms

**Pattern.** Naive client polls aggressively. Producer is DDoSed during peak.

**Corrective.** Server-suggested cadence in response; gateway rate-limit as backstop. Client SDK respects `poll_after_ms`.

### 9.3 The callback URL with no replay protection

**Pattern.** Callback is signed but has no timestamp / nonce. An attacker who captures a valid callback can replay it.

**Corrective.** Timestamp + window check (§3.2). Nonce uniqueness at receiver.

### 9.4 The webhook subscriber that ignores at-least-once

**Pattern.** Webhook documentation says "at-least-once delivery." Receiver processes every webhook as if it's the only delivery. Duplicate deliveries produce duplicate side effects.

**Corrective.** Receiver dedups on event ID. Producer documents the at-least-once contract prominently.

### 9.5 The "we'll add the durable workflow later" deferral

**Pattern.** First version of a multi-step agent runs inline in an API handler. "We'll move to Temporal when we need to." When the team needs to, the migration is painful (existing tasks need to be drained; in-flight state is lost).

**Corrective.** Adopt the workflow framework from the start for any multi-step long-running work. The framework overhead is much cheaper than the migration.

### 9.6 The webhook used as event bus

**Pattern.** Webhooks fan out to many internal consumers. Subscription management is hand-maintained; delivery is unreliable; backpressure is hard.

**Corrective.** Use proper event-driven infrastructure (Kafka, SNS+SQS, EventBridge) for internal fan-out. Webhooks for external integration only. Cross-link to [event-driven-ai-integration.md](./event-driven-ai-integration.md).

### 9.7 The callback URL that's never tested for failure

**Pattern.** Callback delivery is implemented but never tested under failure. First production failure: receiver was down; producer attempted delivery once; gave up; result is lost. Customer complaint.

**Corrective.** Chaos test the callback delivery path. Verify retry policy, dead-letter behavior, TTL.

### 9.8 The polling endpoint with no progress information

**Pattern.** Polling response is just `{status: "running"}`. UI can't show progress; user waits without feedback for a minute and abandons.

**Corrective.** Include progress in polling response (percent, current step, ETA). The workflow framework exposes the underlying state; surface it.

---

## 10. Findings (sprint-assignable)

**ARCH-CWP-001 (P0). Multi-step long-running agent runs inline in API handler.** Worker crashes lose tasks; partial completions orphan state. Adopt durable workflow framework (Temporal / Step Functions / Inngest); refactor agent to workflow + activities. Owner: agent platform.

**ARCH-CWP-002 (P0). Webhook delivery is unsigned.** Attacker who learns URL can deliver fake events. Add HMAC signature with per-subscription secret; document verification in subscriber docs. Owner: platform.

**ARCH-CWP-003 (P0). No idempotency on webhook receiver.** Duplicate delivery produces duplicate side effects. Receiver dedups on `event_id`; document contract in producer's webhook docs. Owner: receiver teams + platform.

**ARCH-CWP-004 (P0). Polling endpoint has no `poll_after_ms` field; clients poll arbitrarily.** Producer DDoSed during peak. Add server-suggested cadence to polling response; gateway rate-limit as backstop. Owner: API team.

**ARCH-CWP-005 (P1). Callback URL has no replay protection.** Captured valid callback can be replayed. Add timestamp + window check; receiver enforces. Owner: receiver teams + platform.

**ARCH-CWP-006 (P1). No per-subscription circuit breaker for webhook delivery.** Failing subscriber's deliveries pile up; storms during recovery. Add per-subscription circuit breaker; pause delivery on failure rate threshold; notify subscriber. Owner: webhook delivery service.

**ARCH-CWP-007 (P1). No delivery audit log for webhooks.** Subscriber can't investigate missed events. Log every delivery attempt (success/failure); expose via API; document. Owner: webhook delivery service.

**ARCH-CWP-008 (P1). Polling response has no progress information.** UI can't show progress; users abandon. Include percent, current step, ETA in polling response; surface from workflow's state. Owner: API team + workflow team.

**ARCH-CWP-009 (P1). Webhook payload contains PII.** Compliance risk; broad access to PII via webhook log. Filter PII at producer; subscribers fetch from access-controlled store with webhook payload as pointer. Owner: producer + security.

**ARCH-CWP-010 (P1). No callback URL TTL or fallback polling path.** Receiver outage during delivery loses result. Hold result for TTL on producer side; polling fallback for receivers that miss callback. Owner: API team.

**ARCH-CWP-011 (P2). No schema versioning on webhook payloads.** Breaking changes break all subscribers simultaneously. Add `schema_version` field; support multiple versions during migration. Owner: producer team.

**ARCH-CWP-012 (P2). Webhook secret rotation requires subscriber downtime.** Rotation is operationally expensive. Support two valid secrets during grace window; subscribers rotate without coordination. Owner: webhook delivery service.

**ARCH-CWP-013 (P2). No per-subscription rate limit.** Slow subscriber receives delivery faster than they can process. Add producer-side per-subscriber rate limit (e.g., 100 events/sec max). Owner: webhook delivery service.

**ARCH-CWP-014 (P2). Workflow versioning is ad-hoc.** Workflow code changes mid-run break in-flight workflows. Adopt workflow versioning discipline (Temporal patches, Step Functions version IDs); document deploy procedure. Owner: workflow team.

**ARCH-CWP-015 (P2). Webhook delivery storm during receiver recovery.** Recovering subscriber gets backlog all at once; falls over again. Add jitter; rate-limit replay; document receiver capacity expectations. Owner: webhook delivery service.

**ARCH-CWP-016 (P3). No alternative return path documented for receivers behind firewalls.** Receivers can't integrate; falls back to manual processes. Document polling and webhook-relay patterns for restrictive networks. Owner: developer relations.

**ARCH-CWP-017 (P3). Long-poll / SSE alternative not offered.** UI clients have to choose between polling lag and webhook complexity. Add SSE endpoint for active-UI scenarios; document. Owner: API team.

**ARCH-CWP-018 (P3). No chaos test for callback / webhook failure paths.** First production failure is the first test. Pre-production chaos: callback receiver returns 503, webhook receiver returns 429, polling endpoint times out. Verify each is handled. Owner: SRE + QA.

---

## 11. Adoption sequencing checklist

For a team adopting callback / webhook / polling / workflow patterns for an AI feature, in order:

- [ ] **Identify the work's properties.** Multi-step? Long-running? Resumable required? Single-request return or long-lived subscription?
- [ ] **Apply the decision tree (§7.1).** Choose primary pattern; identify whether other patterns compose.
- [ ] **If multi-step long-running, adopt durable workflow framework (§6).** Temporal / Step Functions / Inngest / equivalent. Plan onboarding.
- [ ] **For webhook receivers, design subscription model (§4.1).** Subscription records, filtering, lifecycle management.
- [ ] **Sign every webhook and callback from day one (§4.2, §3.1).** HMAC with rotating secret; document verification.
- [ ] **Add replay protection (§3.2).** Timestamp + window; nonce dedup at receiver.
- [ ] **Define delivery retry policy (§3.3).** Exponential backoff; dead-letter after N attempts.
- [ ] **For polling, define completion-status schema (§5.2).** Status, progress, ETA, `poll_after_ms`.
- [ ] **Implement per-subscription circuit breaker for webhooks (§4.3).** Pause on failure rate; notify subscriber.
- [ ] **Build delivery audit log (§4.3).** Every attempt logged; queryable by subscriber.
- [ ] **Document at-least-once contract (§4.4) prominently in receiver docs.** Receivers must dedup.
- [ ] **Filter PII at producer (§4.6).** Subscribers fetch from access-controlled store.
- [ ] **Add schema_version to webhook payloads (§4.6).** Plan for breaking changes.
- [ ] **Support per-subscription rate limit (§4.5).** Backpressure protection.
- [ ] **Pre-production chaos test all return paths.** Receiver outages, slow receivers, callback URL stale, polling timeout, workflow worker crash.
- [ ] **Document return-path conventions** for new feature teams. Decision tree, recommended pattern per use case.

---

## 12. References

**In this folder.**
- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — the integration-shape decision; callback/webhook/polling are the "async" mechanisms.
- [event-driven-ai-integration.md](./event-driven-ai-integration.md) — internal fan-out via event bus (companion; webhooks are for external integration).
- [backpressure-and-queueing.md](./backpressure-and-queueing.md) — queue topology that often sits behind a workflow framework.
- [integration-failure-patterns.md](./integration-failure-patterns.md) — failure handling at integration boundaries; callbacks/webhooks/polling all participate in the failure model.
- [tool-call-architecture.md](./tool-call-architecture.md) — agent tools that themselves use callbacks/webhooks/workflows.
- [human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md) — where humans participate in long-running async workflows.

**Elsewhere in this repo.**
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose return paths are described here.
- [reference-systems/patient-api-ai-assist.md](../reference-systems/patient-api-ai-assist.md) — SaaS feature with webhook subscribers.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant rate limits for webhook delivery.

**Sibling repos.**
- [ai-engineering-reference-architecture / agent-engineering / agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md) — engineering-side agent loop that runs inside a workflow framework.
- [ai-engineering-reference-architecture / agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — partial-failure handling that composes with workflow durability.
- [ai-engineering-reference-architecture / observability-and-telemetry / trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md) — tracing across workflow boundaries.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — threats against callback / webhook integrations (URL leak, replay, signature bypass).

**External.**
- Stripe API webhook docs — gold-standard webhook signing and dedup pattern.
- GitHub webhook docs — similar reference; rich event taxonomy.
- Temporal documentation — the canonical durable workflow reference.
- AWS Step Functions documentation — managed state-machine workflows.
- Inngest documentation — managed durable functions.
- Cloudflare Workflows documentation — Workers-native durable orchestration.
- IETF Server-Sent Events spec — SSE alternative to polling.
