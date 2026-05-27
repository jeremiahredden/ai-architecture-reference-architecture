# Backpressure and Queueing

> **Audience.** Architects designing how AI workloads absorb traffic bursts and propagate "we're full" upstream. Platform teams whose model provider returns 429s at peak. Anyone running a multi-tenant AI platform where one tenant's spike can starve others. **Scope.** The *architectural* decisions: where backpressure originates in AI systems; the queue-topology options (single, per-tenant, per-priority, hybrid); rate-limit propagation discipline; shed-load patterns; priority lanes; per-tenant fairness disciplines. Not the event-driven shape itself (see [event-driven-ai-integration.md](./event-driven-ai-integration.md)). Not the cross-tenant isolation guarantees beyond fairness (see [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)). Not the engineering-side cost mechanics (see sibling [cost-and-finops/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/cost-and-finops)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Conventional service architecture has a mature backpressure playbook: bounded queues, token buckets, circuit breakers, load shedding. Most of it transfers to AI systems unchanged. Some of it doesn't.

What's different about AI workloads:

- **The bottleneck is upstream of you.** The model provider's TPM/RPM, not your CPU or your DB connections, is the binding constraint for hosted-model workloads. You cannot autoscale your way out of it.
- **The bottleneck has price tags attached.** Each call costs money. Queueing strategy is also a cost-control strategy. A queue that delivers 100% of requests during a burst at 3x the steady-state rate just spent 3x the money.
- **The downstream is sometimes slower than the LLM.** Vector stores, embedding indexes, and write-heavy DBs can be slower than the LLM that's producing input for them. Backpressure has to flow *forward* (downstream tells the AI consumer to slow) as well as backward (the AI consumer tells the upstream producer to slow).
- **The unit of work is non-uniform.** A 200-token prompt and an 8000-token prompt count as one request each by RPM but consume 40x different TPM budget. Queueing strategies based on request count alone produce systematically wrong allocations.
- **Latency budgets are operator-set, not provider-set.** Most hosted providers respond with consistent first-token latency until they're near rate limit; then everything degrades together. The application's latency budget (e.g., 800ms for an interactive endpoint) is the queue's binding constraint, not the provider's median latency.

For multi-tenant AI platforms, all of the above compound: one tenant's burst on a shared provider account degrades every other tenant. The first incident with a tenant whose lawyer reads the SLA is the incident that turns "per-tenant fairness" from a backlog item into a P0.

This document covers the architectural decisions: where backpressure originates and propagates; how to choose a queue topology; how to communicate rate limits up the chain; how to shed load when shedding is the right answer; how to give premium-tier tenants real priority; how to make per-tenant fairness load-bearing instead of aspirational.

This document is opinionated about four things:

1. **The bottleneck is the model provider, not your infrastructure.** Tuning your fleet's concurrency without modeling the provider's TPM/RPM produces 429 storms — a noisier failure mode than steady-state queueing would have been.
2. **Per-tenant queueing is not optional past two paying customers.** A single shared queue means tenant A's burst is tenant B's outage. Per-tenant queueing — even if it's just per-tenant token buckets in front of a shared executor — is the line between "single-tenant system that has tenants" and "multi-tenant platform."
3. **Queueing has a cost beyond latency.** Each retained request costs (compute to hold, memory, potentially the LLM call that's already been made). Bounded queues with explicit shed-load policy are correct; unbounded queues with "we'll figure it out" policy are how systems become $400k surprise-bill stories.
4. **Priority is a contract, not a hint.** "Premium tier gets priority" must mean something measurable: latency SLO, queue admission preference, dedicated model tier, dedicated rate budget. Vague priority schemes drift to no-priority in the first incident.

Structure: (2) where backpressure originates in AI systems; (3) the queue-topology choices; (4) rate-limit propagation discipline; (5) shed-load patterns; (6) priority lanes; (7) per-tenant fairness disciplines; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. Where backpressure originates in AI systems

The first move in any backpressure design is being honest about which thing is full. Most AI workloads have at least three candidate bottlenecks; only one binds at a time but which one changes with traffic mix.

### 2.1 The model provider (hosted models)

The dominant source of backpressure for almost all production AI in 2026. Providers quote:

- **RPM** (requests per minute) — usually the binding constraint for low-token, high-volume workloads (classification, short-form chat).
- **TPM** (tokens per minute) — usually the binding constraint for long-context workloads (RAG, document summarization, agent loops with long context).
- **Concurrency limit** — some providers cap simultaneous in-flight requests separately; binding for long-running calls.

**Signals you're hitting it.**

- HTTP 429 with `Retry-After` header.
- Degraded latency before the 429 (some providers slow you down before refusing).
- The provider's status dashboard shows your account near limit.

**What backpressure looks like here.** The provider returns 429s. Your consumer must stop pulling work, hold or shed in-flight, and back off until `Retry-After` elapses. Aggressive retry without backoff worsens the problem — every retry counts toward the rate limit.

**Architectural mitigation.** Maintain a fleet-wide rate-limit budget (Redis-backed token bucket) that the consumers consult before making a call. The budget tracks RPM and TPM separately. When the bucket is empty, the consumer doesn't ask the provider — it queues, backs off, or sheds.

### 2.2 Your GPU pool (self-hosted models)

For self-hosted or fine-tuned models, the bottleneck is the GPU inference cluster — vLLM, TGI, Triton, or equivalent.

**Signals.**

- Queue depth in the inference server (vLLM exposes this metric).
- Time-to-first-token degrading as concurrency rises.
- GPU utilization at 95%+ sustained.

**What backpressure looks like.** The inference server's own queue grows. If you don't bound it, latency rises until requests time out at the consumer.

**Architectural mitigation.** Two-tier queueing: the inference server has a bounded internal queue; the consumer-side queue absorbs overflow with explicit policy (block, shed, route to alternate model). Cross-link to [model-strategy / model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) for the "fall back to hosted when self-hosted is saturated" pattern.

### 2.3 The retrieval / vector store

For RAG-heavy workloads, the vector store is often slower than the LLM. Pinecone, Weaviate, OpenSearch with k-NN, pgvector — each has its own QPS and latency profile under load.

**Signals.**

- Retrieval P99 latency climbing while LLM latency is steady.
- Vector-store-side rate limits triggering.
- Cache hit rate dropping (if you have a retrieval cache).

**What backpressure looks like.** Retrieval calls queue at the vector store; the LLM call waits on retrieval; end-to-end latency grows.

**Architectural mitigation.** Retrieval cache (hot queries served from cache). Vector-store autoscaling where supported. Pre-fetch retrieval results for predictable queries. Degrade gracefully to coarser retrieval (fewer documents, shorter context) when the store is saturated.

### 2.4 Downstream consumers of LLM output

When the LLM's output rate exceeds what downstream can ingest, the downstream becomes the bottleneck. Examples:

- LLM emits 1000 embeddings/sec; vector store can ingest 200/sec.
- LLM produces 100 summaries/sec; downstream warehouse can write 30 rows/sec.
- Agent makes 50 tool calls/sec; downstream tool service can serve 10 RPS.

**Signals.**

- Downstream-side rate limits triggering or queue depth growing.
- LLM completes quickly but downstream completes slowly.
- End-to-end latency dominated by downstream latency.

**What backpressure looks like.** The intermediate queue between LLM output and downstream consumer grows. Eventually either the queue's capacity is reached (drop) or the LLM consumer is told to slow.

**Architectural mitigation.** Intermediate buffer with downstream-aware draining. The AI consumer reads the downstream's queue depth (or response latency) as a slowdown signal and decreases its own pull rate.

### 2.5 Your own concurrency limits

The simplest bottleneck: your consumer's connection pool, thread pool, or pod count is saturated. Often the first to bind in early-stage systems; usually masked once provider limits start binding.

**Architectural mitigation.** Standard service capacity planning. Set explicit concurrency bounds; don't rely on resource exhaustion to throttle.

### 2.6 Reading the bottleneck

Most operators get this wrong by treating the first symptom (high latency) as the diagnosis. The discipline:

- Per-stage latency breakdown: consumer queue wait, retrieval, LLM call, downstream write.
- Per-stage saturation metric: queue depth, in-flight count, rate-limit headroom.
- Cross-stage correlation: when LLM latency is normal but end-to-end is slow, the downstream is the bottleneck.

The architecture's job is to make the bottleneck legible. The operator's job is to respond to the right one.

---

## 3. The queue-topology choices

Four queue topologies cover essentially all AI backpressure architectures. The choice has compound effects: ordering, fairness, replay, cost, and operational complexity all depend on it.

### 3.1 Single shared queue

One queue. All requests land in it. Consumers pull in FIFO order.

**Pros.**

- Simplest to operate. One depth metric to watch, one DLQ to drain.
- Trivially correct ordering (within the broker's per-partition guarantee).
- Easy capacity planning: one queue, one consumer pool.

**Cons.**

- No fairness. Tenant A's burst pushes tenant B's requests to the back.
- No priority. Interactive requests wait behind batch requests.
- All-or-nothing failure mode. Consumer outage stops everyone equally.

**When right.**

- Single-tenant or tenant-of-one systems.
- Internal-only tools where fairness isn't a requirement.
- Pre-revenue MVP where the operational simplicity outweighs the future migration cost.

**Migration trap.** Single-queue systems are easy to ship and hard to evolve. By the time you have three tenants, two priorities, and two SLOs, retrofitting per-tenant or per-priority queueing is a substantial refactor.

### 3.2 Per-priority queue

One queue per priority lane. Typically two-tier (interactive vs batch) or three-tier (interactive / standard / batch). Consumers pull from higher-priority queues first.

**Pros.**

- Interactive workloads don't wait behind batch.
- Capacity allocation is explicit per priority.
- Different SLOs are enforceable per queue (interactive has tight depth alert; batch has looser one).

**Cons.**

- Priority inversion if not designed carefully (a low-priority request holding a shared resource blocks high-priority).
- More queues to operate; more DLQs; more metrics.
- Doesn't address per-tenant fairness — tenant A's interactive burst still starves tenant B's interactive requests.

**When right.**

- Mixed-workload systems (real-time + batch sharing a provider).
- Single-tenant systems with distinct latency tiers.
- A bridge architecture before adopting per-tenant queueing.

**Implementation note.** Strict-priority scheduling (always drain high before touching low) can starve low-priority queues entirely. Weighted-round-robin (e.g., 4:1 in favor of high) prevents starvation while still giving high priority real preference.

### 3.3 Per-tenant queue

One queue per tenant. Consumers pull round-robin (or weighted-round-robin) across tenant queues.

**Pros.**

- Per-tenant fairness by construction. Tenant A's burst affects only tenant A's queue.
- Per-tenant SLOs enforceable.
- Per-tenant capacity caps simple to implement.
- Premium tenants can have their own dedicated consumer pool.

**Cons.**

- Operational overhead grows with tenant count. 100 tenants = 100 queues = 100 DLQs.
- Per-queue overhead at the broker can become real at high tenant counts.
- Implementation requires the producer to know the tenant ID and route accordingly.

**When right.**

- Multi-tenant SaaS with revenue diversity (small + medium + enterprise tenants).
- Regulated workloads where per-tenant audit and isolation are required.
- Systems where one tenant's outage must not affect others.

**Variant: per-tenant logical queue on shared physical queue.** Instead of N physical queues, use one queue with a tenant ID in each message and a tenant-aware consumer that round-robins across tenant groups (Kafka consumer groups with custom assignor; SQS with per-tenant message group ID in FIFO queues). Operationally simpler; loses some properties (per-tenant DLQ requires extra plumbing).

### 3.4 Hybrid: per-tenant × per-priority

Both dimensions. Tenant A has interactive + batch queues; tenant B has the same. Consumers pull with per-tenant fairness on the outer loop, per-priority preference on the inner loop.

**Pros.**

- Most flexible. Premium tenants can have priority over standard tenants on interactive queues; batch is fairly shared.
- Maps directly to commercial tiering (per-tenant fairness + premium tenants get higher SLO).

**Cons.**

- Most operational complexity. N tenants × M priorities = N×M queues.
- Pull algorithm is more complex; bugs in the pull policy produce subtle unfairness.
- Often more topology than the workload actually needs.

**When right.**

- Mature multi-tenant platforms with both revenue tiering and workload tiering.
- Regulated multi-tenant systems where per-tenant fairness + per-priority SLOs are both contractual.

**Pragmatic default.** Per-tenant queues with two priorities per tenant (interactive + batch). Pull policy: round-robin across tenants, prefer interactive 4:1.

### 3.5 Decision summary

| Topology | Fairness | Priority | Complexity | When right |
| --- | --- | --- | --- | --- |
| Single shared | None | None | Lowest | Single-tenant, MVP |
| Per-priority | None | Yes | Low-medium | Mixed-workload single-tenant |
| Per-tenant | Yes | None | Medium | Multi-tenant SaaS |
| Per-tenant × per-priority | Yes | Yes | High | Mature multi-tenant with tiering |

**Migration ordering.** Single shared → per-priority is easier than → per-tenant. Per-tenant → per-tenant × per-priority is incremental. The painful migration is single shared → per-tenant, which requires producer-side routing changes and consumer-side queue-discovery changes. Build for per-tenant from the start if multi-tenant is on the roadmap.

---

## 4. Rate-limit propagation discipline

When the system is at capacity, "we're full" must propagate up the chain. The discipline is about how you communicate it.

### 4.1 The three communication patterns

**Pattern A: 429 with Retry-After (synchronous APIs).**

The HTTP standard. Caller receives 429, reads `Retry-After`, waits, retries. Suitable when the caller is human-driven (mobile app, web frontend) and can show "try again in a moment."

**Pitfall.** Many clients ignore `Retry-After` and retry immediately. The architecture should assume some fraction of callers will retry hot; rate-limit them more aggressively or block them at the gateway.

**Pattern B: 202 Accepted with polling URL (long-running async).**

Caller submits work; server accepts and returns a poll URL. Caller polls. Server controls when work runs (in queue) and how the result is delivered (callback or poll response). Suitable when the caller is a system that can hold state.

**Pitfall.** Polling intervals not capped. Naive clients poll every 100ms; server gets DDoSed by its own clients during peak. Set minimum poll interval; reject too-frequent polls with 429-on-poll.

**Pattern C: Webhook with subscription (event-driven).**

Caller subscribes once; server pushes results when done. Caller doesn't poll. Suitable when the caller can receive callbacks (server-to-server, durable subscribers).

**Pitfall.** Webhook delivery during backpressure. If the receiver is slow, webhook attempts pile up. Receiver-side backpressure (the receiver returns 429 to the webhook sender) must be supported.

Detail on Pattern B and C is in [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md). This document focuses on the queueing behind whichever pattern you choose.

### 4.2 Rate-limit headers as load-bearing contract

If your service is a rate-limited API to its callers, your headers are the contract. The convention (RFC 6585, RFC 9001 draft, IETF Rate-Limit headers draft):

- `X-RateLimit-Limit`: caller's per-window budget.
- `X-RateLimit-Remaining`: caller's remaining budget in current window.
- `X-RateLimit-Reset`: epoch seconds when the window resets.
- `Retry-After`: seconds (or HTTP date) the caller should wait before retrying.

**Pattern.** Return these headers on every response, not just 429s. Well-behaved clients pre-empt rate limits by slowing as `Remaining` shrinks. Returning only on 429 forces clients to learn the limit by hitting it.

**Pattern.** For multi-tier limits (per-tenant + per-API-key + per-endpoint), document which limit produced the 429. Header convention isn't standardized; use a custom header (e.g., `X-RateLimit-Scope: tenant`) so the client knows which budget they're hitting.

### 4.3 Upstream propagation

When your service is at capacity, you may need to slow *your own* upstream producers. Two patterns:

**Pattern A: producer pulls from queue.** Producer reads queue depth or rate-limit headers from a control plane; producer slows or stops producing when downstream signals capacity exhaustion.

**Pattern B: producer is paged.** Producer is a request handler; you return 429 to the caller and let the caller's own backoff propagate. Suitable when "the caller" is an upstream API that respects 429.

The pattern depends on whether you control the producer. If yes (internal pipeline stage), Pattern A is cleaner. If no (external caller), Pattern B is your only option.

### 4.4 Fleet-wide rate-limit budget

For provider-side rate limits, the consumer fleet must share a single budget. A 10-consumer fleet hitting a 1000 RPM provider limit at 100 RPM each works in steady state and explodes during the first burst when several consumers happen to send simultaneously.

**Implementation.**

- Redis-backed token bucket. Consumers acquire tokens before calling the provider.
- The bucket's refill rate is the provider's RPM/TPM divided by the consumer count's expected burst-overlap factor (usually leave 10-15% headroom).
- TPM bucket is separate from RPM bucket; both must have available tokens before a call.
- Token acquisition has a timeout; if no token in N seconds, the consumer queues (or sheds).

**Pitfall.** Token-acquisition contention at scale. 100 consumers all calling Redis SETNX on every LLM call adds latency. Use Lua script for atomic decrement-and-check; consider client-side batching (acquire 10 tokens at once, use over several calls).

---

## 5. Shed-load patterns

When capacity is exhausted, three things can happen: the work waits (queue), the work degrades (smaller model, less context, cached response), or the work is refused (drop or 429). The first instinct is always "queue more," and it's almost always wrong past a certain point.

### 5.1 The four shed-load options

**Option A: Drop.**

The request is dropped; the caller gets 503 or 429. Suitable for low-value workloads (best-effort recommendations, optional enrichment).

**When right.** The request has no business value if delayed (real-time ad bid). The workload is best-effort by SLA.

**When wrong.** The request is mandatory (user-facing operation). The customer is paying for it.

**Mechanism.** Queue admission control. New requests are rejected when queue depth exceeds a threshold or when capacity is provably exhausted.

**Option B: Degrade.**

The request is processed with reduced quality. Smaller model, less context, cached response, simpler prompt. Suitable when the workload can tolerate quality reduction.

**When right.** The user value of a degraded response is higher than the user value of an error or a long wait. Cross-link to [reliability-engineering / fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) in the engineering sibling.

**When wrong.** The workload requires the full model (regulated, contractual, safety-critical).

**Mechanism.** The consumer detects backpressure (cache lookup first, fallback to smaller model, return cached or templated response). Cross-link to [model-strategy / model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md).

**Option C: Queue.**

The request waits. Suitable when the workload is latency-tolerant.

**When right.** Batch workloads. Async workloads where the user is not waiting.

**When wrong.** Latency-sensitive workloads (user-facing chat, real-time scoring). The queue's depth determines the user's wait; an unbounded queue produces unbounded waits.

**Mechanism.** Bounded queue with explicit admission control. The queue's depth is monitored; when depth exceeds threshold, fall back to drop or degrade.

**Option D: Refuse with retry-later.**

Return 429 with `Retry-After`; the caller's responsibility to come back. Suitable for system-to-system traffic where the caller is well-behaved.

**When right.** API consumers that respect rate-limit headers. Internal pipeline consumers.

**When wrong.** Human-driven traffic that doesn't retry (the user sees an error, gives up). Naive clients that retry hot.

**Mechanism.** Rate-limit at the API gateway; 429 with header; document the policy in API docs.

### 5.2 Choosing per-workload

The shed-load policy is a per-workload decision, not a platform-wide one.

| Workload class | Default shed policy | Rationale |
| --- | --- | --- |
| Real-time clinical decision support | Refuse with retry-later, or degrade to last-known-good | Latency-sensitive; cannot queue indefinitely; safety-critical so cannot drop silently |
| Interactive chat | Degrade to smaller model | User present; degraded response > error |
| Document classification batch | Queue, then drop oldest | Latency-tolerant; backlog clears overnight |
| Real-time ad scoring | Drop | Each request is low-value individually |
| Compliance audit batch | Queue with no shed | Cannot drop; latency-tolerant |
| Premium customer interactive | Queue with priority + dedicated capacity | SLA contractually requires processing |

The architecture's job is to make these per-workload policies explicit and configurable. The anti-pattern is "everything queues forever" or "everything drops on overflow" — both produce wrong behavior for workloads that don't fit the policy.

### 5.3 Detecting "shed" conditions

Three signals usually combine to indicate shed-load is appropriate:

1. **Queue depth or growth rate exceeds threshold.** Static threshold for absolute depth; rate threshold for "depth growing 100/sec faster than drain rate."
2. **Provider rate-limit headroom is low.** Fleet-wide token bucket below 10% remaining.
3. **Latency SLO at risk.** P99 latency over the last minute approaching the SLO ceiling.

A single signal can be noise; two or more correlated signals warrant action. The action depends on the workload's shed policy (§5.2).

### 5.4 Communicating shed to the caller

When shed happens, the caller deserves to know. Three patterns:

- **Soft fail (default value).** Return a sentinel response indicating shed ("service temporarily degraded; this is a default response"). Caller can decide whether to retry or use the default.
- **Hard fail with structured error.** Return 429/503 with a body that describes what happened, what was tried, what to retry, and when.
- **Graceful degradation indicator.** Return a normal response with metadata indicating "this was served by the fallback path." Caller can choose to surface or hide.

The right pattern depends on the caller. Internal callers benefit from structured errors. External APIs often benefit from soft-fail (don't break callers that don't handle errors). User-facing apps benefit from graceful degradation indicators (so the UI can show "showing cached results").

---

## 6. Priority lanes

Priority is a contract. The architecture must enforce it; "we try to prioritize premium customers" is not priority, it's hope.

### 6.1 What priority must guarantee

For priority to be load-bearing, the system must guarantee:

- **Queue admission preference.** High-priority requests are admitted to the queue before low-priority requests when capacity is tight.
- **Capacity reservation.** Some fraction of capacity is reserved for high-priority requests, not available to low-priority requests.
- **Distinct SLOs.** High-priority has a latency SLO; low-priority has a different (looser) latency SLO or no latency SLO.
- **Distinct shed-load behavior.** Under load, low-priority sheds first.

A "priority" tier that doesn't meet at least the first two of these is marketing, not architecture.

### 6.2 Two-tier priority (interactive vs batch)

Most production systems start here. The two tiers are:

- **Interactive.** User is waiting. SLO: P99 < N seconds (typical: 2-10s).
- **Batch.** No user waiting. SLO: completion within a window (typical: hours).

**Implementation.**

- Per-priority queue (§3.2). Consumers prefer interactive 4:1 or stricter.
- Capacity reservation: at least X% of consumer capacity is held for interactive (idle when no interactive work). Prevents batch from monopolizing.
- Shed policy: batch can queue deep; interactive sheds to degraded response when queue grows.

### 6.3 Three-tier priority (real-time / standard / batch)

Adds a "real-time" tier for the highest-criticality workloads (clinical safety, financial trading, fraud detection).

**Implementation.**

- Real-time has dedicated capacity (not shared). The dedicated capacity may be a separate model deployment, separate provider account, or separate consumer pool.
- Standard is the default; shares capacity with batch but with priority preference.
- Batch is the lowest; sheds first; can be drained to dedicated batch infrastructure overnight.

**Pitfall.** "Real-time" tier proliferation. Every team wants their workload to be real-time. The architecture should make real-time expensive (dedicated capacity has explicit cost), so teams self-select.

### 6.4 Per-customer priority (premium tier)

Multi-tenant SaaS with revenue tiering: premium customers get faster service.

**Implementation.**

- Premium tenants get either: (a) higher weight in per-tenant round-robin (premium = weight 4, standard = weight 1), or (b) admission preference (premium goes to a dedicated queue with reserved capacity), or (c) a higher model tier (premium = larger, more expensive model by default).
- The premium contract documents what's guaranteed. "Premium gets priority" is vague; "premium gets 4x weight in round-robin and dedicated 20% of capacity during peak" is contractual.

**Pitfall.** Premium tenants on the same provider account as free tenants. If the free tier exhausts the provider's RPM, premium tenants are blocked too. Architectural fix: separate provider account or separate API key per tier; provider-side rate limit applies per key.

### 6.5 The priority-inversion trap

When a high-priority request depends on a low-priority resource, the low-priority resource's saturation blocks the high-priority request.

**Example.** Interactive chat request needs retrieval. Retrieval service is at capacity because of a batch backfill that's running through the same retrieval pool. Interactive chat waits behind batch backfill.

**Mitigation.**

- Reserve capacity in shared services for high-priority callers.
- Tag requests with priority and propagate the tag through the call chain.
- The shared service prioritizes its own queue by caller's tag, not just by arrival time.

---

## 7. Per-tenant fairness disciplines

Per-tenant fairness is the line between a single-tenant system that hosts tenants and a multi-tenant platform. The architecture has to enforce it; "we trust tenants to behave" is not a strategy.

### 7.1 The fairness dimensions

For AI workloads, fairness applies on several dimensions:

- **Request rate (RPS).** No tenant exceeds N requests per second.
- **Token consumption (TPM / TPS).** No tenant exceeds N tokens per minute.
- **Cost ($/period).** No tenant exceeds $X per month or $Y per hour.
- **Queue position.** Tenants are served round-robin (or weighted-round-robin), not first-come-first-served.
- **Retrieval-index hot-key.** No tenant's queries dominate the vector store's cache.
- **Side-effect store writes.** No tenant's writes monopolize the downstream sink.

Single-dimension fairness (e.g., per-tenant RPS) is incomplete; a tenant respecting RPS can still exhaust the TPM budget with long-context requests, or exhaust the cost budget with expensive-model calls.

### 7.2 Per-tenant budgets

The mechanism:

- Each tenant has explicit per-dimension budgets (RPM, TPM, $/day).
- Budgets are enforced by token buckets, leaky buckets, or weighted-round-robin scheduling.
- When a tenant exhausts a budget, that tenant's requests are queued, shed, or returned 429 — *that tenant only*.
- Tenant budgets are visible to the tenant (API to query current consumption) and to ops.

**Pattern.** Budgets sum to greater than fleet capacity. If 100 tenants each have a 1000 RPM budget and fleet capacity is 50000 RPM, the system can be over-provisioned in nominal case (tenants rarely use full budget) but fairness ensures no single tenant can take more than its budget when capacity is tight.

**Pattern.** Burst budgets. Short-term burst above sustained budget is allowed; long-term burst is not. Token bucket with bucket size = burst limit, refill rate = sustained limit.

### 7.3 Weighted-fair queueing

When tenants have different commercial tiers, weighting captures that:

- Tenant A (free): weight 1.
- Tenant B (standard): weight 3.
- Tenant C (premium): weight 10.

Round-robin scheduler picks from each tenant's queue in proportion to weight. Premium tenants get 10x more turns than free.

**Implementation.** Deficit-round-robin or weighted-fair-queueing algorithm from networking literature; both are well-studied and easy to implement. The key property is anti-starvation: even the lowest-weight tenant gets served eventually.

### 7.4 Anti-starvation

A naive priority scheme starves low-priority. A fair scheme guarantees liveness even for the lowest tier.

**Pattern.** Strict round-robin across tenants, with priority only as a tiebreaker. Tenant A always gets at least one turn per round, even if tenant B is premium.

**Pattern.** Aging: requests that wait too long get a priority bump. A free-tier request that has waited 30 seconds gets priority equivalent to standard for the next round. Prevents indefinite starvation.

### 7.5 Detection of "noisy neighbor" events

The signal you want before the SLA-violation page:

- Per-tenant traffic dashboard: tenants ranked by RPM, TPM, $/hour over the last window.
- Per-tenant SLO compliance: which tenants are experiencing degraded latency.
- Cross-tenant correlation: when tenant A's RPM spikes, does tenant B's latency degrade?

A spike in tenant A correlated with degradation for tenant B is a fairness incident. The architecture's job is to make the correlation visible; the operator's job is to act on it (tighten tenant A's budget, page on-call, escalate to commercial).

Detail on the multi-tenancy patterns beyond fairness is in [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

---

## 8. Worked Meridian example

Meridian Health runs three workload classes on overlapping AI infrastructure:

- **Real-time Care Coordinator** (interactive clinical agent, 200-800 concurrent users in business hours).
- **Document ingestion pipeline** (40k-110k docs/day, batch with overnight peak).
- **Patient API AI-assist** (multi-customer SaaS feature, 12 paying tenants ranging from 50 to 5000 daily users).

The three share one Anthropic account, two model tiers, and one vector store cluster. Without per-tenant and per-priority queueing, one tenant's burst or one batch backfill could degrade clinical care. Here's how it's structured.

### 8.1 The topology

```
                         ┌─────────────────────────┐
                         │   Per-tenant × per-     │
                         │   priority queue layer  │
                         │   (hybrid §3.4)         │
                         └──────────┬──────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ↓                           ↓                           ↓
[real-time queues]          [standard queues]            [batch queues]
        │                           │                           │
        │                  ┌────────┴───────┐                   │
        │                  ↓                ↓                   │
        │            [internal tenants] [SaaS tenants]          │
        │                                                        │
        ↓                                                        ↓
[dedicated consumer pool]                            [shared consumer pool]
        │                                                        │
        ↓                                                        ↓
[reserved 30% of provider RPM]                  [70% of provider RPM]
```

- Real-time queue feeds a dedicated consumer pool with reserved 30% of the Anthropic account's RPM/TPM. Care Coordinator's interactive calls land here. No other workload can starve it.
- Standard queues feed the shared pool. Internal tools (analytics-warehouse-copilot) and SaaS tenants share, with per-tenant weighted-round-robin (§7.3).
- Batch queues feed the same shared pool but with lower priority (4:1 vs standard). Document ingestion lands here.
- SaaS tenants are tiered: free (weight 1), standard (weight 3), premium (weight 10). Two premium tenants have additional 5% RPM dedicated, on top of fair-share.

### 8.2 The budget structure

- **Provider account total:** 5000 RPM, 2M TPM (Anthropic enterprise tier).
- **Real-time reservation:** 30% = 1500 RPM, 600k TPM. Reserved at all times; idle when no clinical work.
- **Standard / batch shared:** 70% = 3500 RPM, 1.4M TPM. Allocated by weighted-round-robin.
- **Per-tenant SaaS budgets:**
  - Free tenants: 100 RPM, 50k TPM each.
  - Standard tenants: 500 RPM, 250k TPM each.
  - Premium tenants: 1500 RPM, 750k TPM each + 5% RPM dedicated.
- **Per-tenant cost cap:** $/day cap per tenant; circuit-breaker kicks in at 80%, hard-stop at 100%. Cross-link to [cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) in engineering sibling.

### 8.3 The shed policies

| Workload | Shed policy | Trigger | Behavior |
| --- | --- | --- | --- |
| Care Coordinator (real-time) | Degrade then refuse | Real-time queue depth > 50 or P99 > 8s | Fall back to smaller model (Haiku); if still saturated, refuse with structured error to clinician UI |
| Document ingestion | Queue, drop oldest at depth cap | Batch queue depth > 10000 | Drop oldest events (with audit); operator paged |
| SaaS tenant interactive | Per-tenant 429 with Retry-After | Tenant's RPM bucket empty | Return 429 to that tenant; other tenants unaffected |
| SaaS tenant batch | Queue, drop on tenant cap | Tenant's batch queue exceeds tenant cap | Drop with notification to tenant |

### 8.4 What this prevents

Three specific incidents this topology has prevented (Q4 2025 — Q1 2026):

**Incident A: Free-tier tenant's runaway agent.** A free-tier tenant deployed an agent that misbehaved and looped, attempting 200 RPM. The per-tenant bucket capped them at their 100 RPM budget; they received 429s; no other tenant experienced degradation. The incident lasted 4 hours (until the tenant noticed and fixed the loop) but was invisible to all other tenants and to Care Coordinator.

**Incident B: Backfill of 1M documents during business hours.** A new ingestion stage required re-classifying 1M historical documents. Without queueing topology, this would have exhausted provider capacity. With the topology, the backfill ran on the batch queue, was throttled to a fraction of shared capacity, and took 18 hours; Care Coordinator was unaffected and SaaS tenants saw a slight (within-SLO) latency increase.

**Incident C: Provider degradation during clinical peak.** Anthropic had a 90-minute period of elevated latency. The fleet-wide rate-limit budget detected the degradation (latency rising despite token availability) and the real-time path's degrade policy fell back to the Haiku model for non-critical Care Coordinator calls. Critical-path clinical work continued on the primary model; non-critical (background suggestions, summarization) ran on the fallback. Zero clinical-care impact.

### 8.5 What the topology costs

- Operational complexity: 14 logical queues (3 real-time + 3 standard internal + 12 standard SaaS + 12 batch SaaS, plus DLQs). Managed via tenant-aware control plane that auto-provisions queues on tenant onboarding.
- Rate-limit infrastructure: Redis cluster sized for ~50 ops/sec per request (a few hundred RPS at peak). One small cluster.
- Developer onboarding: ~2 hours to understand the topology; new workloads must declare their priority tier and tenant scope at deployment time.

The topology emerged over four iterations:

1. **Single shared queue** at MVP (one tenant, internal only). Stable until tenant 2 onboarded.
2. **Per-priority queue** when document ingestion started competing with Care Coordinator. Solved interactive-vs-batch.
3. **Per-tenant queues** when SaaS rollout hit 3 paying tenants. Solved fairness.
4. **Per-tenant × per-priority hybrid** when premium tier launched. Current state.

Each iteration was a migration; the lesson is to build for the topology you'll need in 18 months, not the one you need on launch day.

---

## 9. Anti-patterns

### 9.1 The unbounded queue

**Pattern.** Queue has no depth cap. Memory grows during burst; restart loses everything; latency grows without bound.

**Corrective.** Explicit depth cap. Define behavior at the cap (drop oldest, drop newest, refuse new, page operator).

### 9.2 The single bucket for all tenants

**Pattern.** Fleet-wide token bucket; first-come-first-served. Tenant A's burst exhausts the bucket; tenants B, C, D get 429s.

**Corrective.** Per-tenant buckets feeding a fairness scheduler. Cross-link to §7.

### 9.3 The "we'll add priority later" deferral

**Pattern.** Single queue at launch; "premium tier is roadmap." Premium tier launches; the SLA is signed. The first incident shows premium is not actually faster than free.

**Corrective.** Priority is a contract. Either deliver it architecturally or don't sell it. Deferring priority to v2 makes the v2 migration painful and creates a window where you're billing for a guarantee you can't make.

### 9.4 The retry storm that masks capacity issues

**Pattern.** Provider returns 429. Consumer retries immediately. Retry hits 429. Retry again. Each retry counts toward rate limit, prolonging the saturation. The retry storm masks the underlying issue (the provider's rate limit), and the operator sees "high error rate" without understanding it's self-inflicted.

**Corrective.** Provider-aware backoff. Respect `Retry-After`. Surface "retries due to backpressure" as a distinct metric from "retries due to transient error."

### 9.5 The hidden synchronous call in the async path

**Pattern.** The async pipeline has a "small" synchronous call to fetch context. Under normal load, the call is 50ms. Under backpressure, the call is 5 seconds. The pipeline's apparent latency budget triples because of the hidden sync.

**Corrective.** All inter-service calls in the async path must have explicit timeouts and explicit fallback behavior. Profile end-to-end during synthetic backpressure.

### 9.6 The shared retry budget across workload classes

**Pattern.** Retry budget is one number for the whole consumer. A retry storm in low-priority workload exhausts the budget; high-priority retries fail.

**Corrective.** Per-priority retry budgets. High-priority retries from a reserved budget that low-priority can't touch.

### 9.7 The "premium tier is just a label" trap

**Pattern.** Premium and standard tenants land in the same queue. Premium tenants are tagged but not given preference. Marketing material claims premium is "priority access."

**Corrective.** Priority must change runtime behavior measurably. Reserved capacity, distinct queue, weighted scheduling. If you can't measure the difference between premium and standard, you don't have a premium tier.

### 9.8 The cost-budget circuit-breaker that nobody tests

**Pattern.** Per-tenant cost cap exists in config. Nobody tests it. First time it fires (a tenant's runaway agent hits the cap), it fires on tenant A but also blocks tenants B and C because the implementation has a bug.

**Corrective.** Test cost-budget circuit-breakers in pre-production with synthetic traffic. Per-tenant means per-tenant; verify with a multi-tenant test.

---

## 10. Findings (sprint-assignable)

**ARCH-BPQ-001 (P0). Single shared queue serving multiple tenants.** First multi-tenant incident is the migration trigger; better to migrate before. Add per-tenant logical queues (per-tenant message group ID or per-tenant queues) with weighted-round-robin scheduling. Owner: platform.

**ARCH-BPQ-002 (P0). No depth cap on critical queues.** Unbounded queue depth produces unbounded latency and surprise restarts. Set depth caps; define behavior at cap (drop oldest / drop newest / refuse). Owner: platform.

**ARCH-BPQ-003 (P0). No fleet-wide rate-limit budget for provider calls.** 429 storms during natural bursts. Implement Redis-backed token bucket sized to provider RPM/TPM; all consumers acquire before calling. Owner: AI platform.

**ARCH-BPQ-004 (P0). No per-tenant cost cap.** Runaway agent on a free tenant can rack up unlimited bill. Implement per-tenant $/day cap with circuit-breaker at 80% warning, 100% hard stop. Owner: AI platform + FinOps.

**ARCH-BPQ-005 (P1). No priority queue separation between interactive and batch.** Batch workloads degrade interactive latency. Add per-priority queues; consumers prefer interactive 4:1; reserve N% capacity for interactive. Owner: platform.

**ARCH-BPQ-006 (P1). Retry policy doesn't respect Retry-After.** Retry storms during provider degradation. Implement provider-aware backoff: read Retry-After; back off entire fleet; surface as backpressure metric. Owner: consumer team.

**ARCH-BPQ-007 (P1). No shed-load policy per workload.** Default-queue-forever produces unbounded latency. Define shed policy per workload (drop / degrade / queue / refuse); document; test. Owner: feature teams + platform.

**ARCH-BPQ-008 (P1). Premium-tier customers in same queue as standard.** Premium SLO not enforceable. Add per-tier weighted-round-robin or reserved capacity for premium. Owner: platform + product.

**ARCH-BPQ-009 (P1). No per-tenant TPM tracking, only RPM.** Long-context workloads bypass RPM throttle and exhaust TPM. Track and budget TPM separately; both must have available headroom for a call. Owner: AI platform.

**ARCH-BPQ-010 (P2). Rate-limit headers not returned on successful responses.** Clients can't pre-empt rate limits; learn them by hitting 429. Return `X-RateLimit-*` headers on every response. Owner: API team.

**ARCH-BPQ-011 (P2). No detection of cross-tenant correlation.** "Tenant A spike caused tenant B degradation" is invisible. Build dashboard that overlays per-tenant traffic with per-tenant latency. Owner: SRE.

**ARCH-BPQ-012 (P2). Hidden synchronous call in async path.** Pipeline latency triples during backpressure because of an un-timeouted sync call. Audit async paths for sync calls; add timeouts; profile end-to-end under synthetic backpressure. Owner: pipeline teams.

**ARCH-BPQ-013 (P2). Shared retry budget across priorities.** Low-priority retry storm starves high-priority retries. Split retry budget per priority class. Owner: consumer team.

**ARCH-BPQ-014 (P2). Strict-priority scheduling without anti-starvation.** Lowest priority can be starved indefinitely. Switch to weighted-round-robin or deficit-round-robin with anti-starvation guarantees. Owner: platform.

**ARCH-BPQ-015 (P2). No aging / priority bump for long-queued requests.** Free-tier requests can wait indefinitely. Implement aging: priority bump after waiting longer than N seconds. Owner: platform.

**ARCH-BPQ-016 (P3). No structured-error response on shed.** Caller can't distinguish shed from upstream error. Define structured error body (reason, retry guidance, alternative actions). Owner: API team.

**ARCH-BPQ-017 (P3). Per-tenant budget not visible to tenants.** Tenants can't self-manage; surprise 429s. Expose per-tenant budget and current consumption via API. Owner: API team + product.

**ARCH-BPQ-018 (P3). Cost-budget circuit-breakers not tested in pre-production.** First production fire is the first test. Add synthetic multi-tenant load tests that exercise cost caps; verify per-tenant isolation. Owner: SRE + QA.

---

## 11. Adoption sequencing checklist

For a team adopting backpressure and queueing in an AI platform, in order:

- [ ] **Identify the binding bottleneck (§2).** Provider RPM, provider TPM, self-hosted GPU, vector store, downstream consumer. Different mitigations for each.
- [ ] **Choose queue topology (§3) based on tenant count and roadmap.** Build for per-tenant if multi-tenant is on the roadmap; the migration is expensive.
- [ ] **Implement fleet-wide rate-limit budget (§4.4) backed by Redis token bucket.** RPM and TPM tracked separately. All consumers acquire before calling.
- [ ] **Set explicit depth caps on every queue (§5.1).** Define shed behavior at cap.
- [ ] **Define shed policy per workload (§5.2).** Real-time degrades; batch queues; etc. Document the policy.
- [ ] **Implement priority lanes (§6) if mixing latency-sensitive and latency-tolerant workloads.** Reserve capacity for high-priority; verify with synthetic load.
- [ ] **Implement per-tenant budgets (§7.2) if multi-tenant.** RPM, TPM, $/day, all enforced independently.
- [ ] **Implement weighted-fair queueing (§7.3) if tenant commercial tiering exists.** Premium = higher weight or dedicated capacity.
- [ ] **Implement provider-aware backoff (§4.4).** Respect Retry-After. Surface as metric.
- [ ] **Return rate-limit headers on every response (§4.2).** Document the contract in API docs.
- [ ] **Build cross-tenant correlation dashboard (§7.5).** Per-tenant traffic vs per-tenant latency.
- [ ] **Implement per-tenant cost circuit-breaker (§8.2).** 80% warning, 100% hard stop, per tenant.
- [ ] **Run synthetic backpressure tests.** Exercise shed policies, priority scheduling, per-tenant isolation. Page on regressions.
- [ ] **Document the topology** as a one-page diagram in the platform runbook. New team members should understand priority/fairness in <15 minutes.

---

## 12. References

**In this folder.**
- [event-driven-ai-integration.md](./event-driven-ai-integration.md) — how backpressure propagates in event-driven AI systems (companion).
- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — the integration-shape decision; backpressure mechanics differ per shape.
- [integration-failure-patterns.md](./integration-failure-patterns.md) — failure-mode taxonomy at AI integration boundaries.
- [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md) — long-running AI work as a backpressure mechanism.
- [tool-call-architecture.md](./tool-call-architecture.md) — tool calls inherit the agent's backpressure regime.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — companion doc on multi-tenant isolation beyond fairness (queueing is necessary but not sufficient).
- [multi-tenancy-and-isolation/isolation-models.md](../multi-tenancy-and-isolation/isolation-models.md) — the three isolation models; queueing topology depends on which model.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — the model-tier fallback pattern used in shed-degrade.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose queueing topology is described here.

**Sibling repos.**
- [ai-engineering-reference-architecture / reliability-engineering / fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md) — the model-fallback pattern composing with shed-degrade.
- [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) — per-tenant cost caps that compose with per-tenant queue budgets.
- [ai-engineering-reference-architecture / cost-and-finops / tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — model-tier routing under cost pressure, which composes with shed-degrade.
- [ai-engineering-reference-architecture / observability-and-telemetry / alerting-and-paging-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/alerting-and-paging-design.md) — alerts for queue depth, rate-limit headroom, per-tenant fairness.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — DoS / abuse patterns that backpressure must defend against.

**External.**
- IETF Rate-Limit Headers draft (`draft-ietf-httpapi-ratelimit-headers`).
- RFC 6585 §4 (HTTP 429 Too Many Requests).
- Weighted-fair queueing literature (Demers, Keshav, Shenker; deficit-round-robin).
- Anthropic / OpenAI / Google rate-limit documentation for provider-specific RPM/TPM models.
- Netflix Hystrix patterns (historical; the concepts apply even though the library is deprecated).
- AWS SQS / SNS / EventBridge service docs for AWS-native queue mechanics.
