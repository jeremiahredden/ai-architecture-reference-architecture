# Latency Budgets and Streaming

> **Audience.** Architects whose AI feature is "too slow" — and who want a diagnostic framework rather than a model swap. Tech leads designing a streaming UX for the first time. Anyone whose retrieval is fast, model is fast, but end-to-end latency is somehow slow. **Scope.** The *architectural* practice of latency design for AI: latency-budget discipline (TTFT, TTUO, total response); the budget cascade across system / retrieval / model / tools / answer; when streaming is required vs not; streaming-implementation patterns that survive proxies, gateways, CDNs; SSE vs WebSocket choice. Not the engineering-side timeout calibration (see [ai-engineering-reference-architecture / reliability-engineering / timeout-strategy.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/timeout-strategy.md)). Not the integration-shape decision (see [integration-architecture/sync-vs-async-vs-streaming.md](../integration-architecture/sync-vs-async-vs-streaming.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI feature latency is rarely a model problem. It's almost always an architecture problem.

A typical latency breakdown for a "slow" AI feature:

```
Total latency: 8.2s
  - Network / load balancer:    150ms
  - Authentication:             80ms
  - Pre-call rate-limit check:  20ms
  - Vector store retrieval:     350ms  ← 4% of total; not the bottleneck
  - Prompt assembly:            50ms
  - LLM call:                   4200ms ← 51% of total; the dominant factor
  - Output validation:          80ms
  - Side-effect writes:         600ms
  - Response serialization:     50ms
  - Network return:             100ms
  - Frontend rendering:         2500ms ← 30% of total; second-largest
```

The "the model is slow" diagnosis misses that the frontend render is also a major contributor; that the side-effect writes are significant; that the retrieval is fine.

Better latency requires:

- A budget per stage.
- Measurement of each stage.
- Optimization where it matters.

And for user-facing interactive workloads, streaming changes the equation: perceived latency is what matters, and streaming delivers first tokens in 1-2 seconds while total response takes 8.

This document covers the architectural patterns:

- The latency budget discipline.
- The cascade across system stages.
- When streaming is required (and when it isn't).
- The implementation patterns for streaming (SSE, WebSocket, what survives proxies).
- Optimization where it matters most.

This document is opinionated about four things:

1. **Latency is a budget, not a number.** Total target divided across stages; each stage has an allocation; exceeding any stage exceeds the total.
2. **Perceived latency matters more than total latency for interactive workloads.** Streaming delivers perceived latency under 2 seconds while total may be 8.
3. **The model is rarely the slowest stage.** Optimize where latency actually lives — usually retrieval, side-effects, or frontend.
4. **Streaming implementation is harder than it looks.** Proxies buffer; gateways close; CDNs cache. The implementation must work end-to-end through your stack.

Structure: (2) the latency budget discipline; (3) the cascade across stages; (4) when streaming is required; (5) the streaming implementation patterns; (6) optimization techniques; (7) per-workload latency targets; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The latency budget discipline

The discipline that makes latency work.

### 2.1 The budget per workload

Per feature, define:

- Total latency target (e.g., P99 < 5s).
- TTFT target (for streaming; e.g., P99 < 2s).
- Allocated budget per stage.

The total budget = sum of stage allocations + safety margin.

### 2.2 The standard stages

For most AI features:

```
Stage                       Typical allocation
─────────────────────────────────────────────
Network / load balancer     50-200ms
Authentication              30-100ms
Authorization checks        30-100ms
Pre-call cost / rate check  10-50ms
Retrieval                   200-500ms
Prompt assembly             10-50ms
LLM call                    1-8s
Tool calls (if any)         100-500ms each
Output validation           50-200ms
Side-effect writes          100-500ms
Response serialization      20-100ms
Network return              50-200ms
Frontend rendering          0-3s (varies wildly)
```

Allocate per stage; sum to total.

### 2.3 The "is the budget realistic" check

Sum of stage allocations must equal target with margin:

```
Target P99: 5000ms
  Stages sum: 4500ms
  Margin: 500ms (10%)
  
  Achievable? Depends on each stage's actual performance.
```

If not, either:
- Increase target (tell customer 7s, not 5s).
- Reduce stages (skip retrieval; faster model).
- Optimize specific stages.

### 2.4 The "no margin" trap

When stage allocations equal total exactly:

- One slow stage breaks the whole budget.
- No room for variance.

Always reserve margin (5-15%).

### 2.5 The stage-specific budget enforcement

Each stage has its own timeout (cross-link to [ai-engineering-reference-architecture / reliability-engineering / timeout-strategy.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/timeout-strategy.md)):

- Retrieval timeout: 800ms (allows P99 of 500ms with margin).
- LLM call timeout: 6000ms (allows P99 of 4000ms with margin).
- Side-effect timeout: 1000ms.

If a stage exceeds its timeout, fail-fast; degraded mode (cross-link to [degraded-mode-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md)) kicks in.

### 2.6 The budget review

Quarterly or per-feature deploy:

- Measured actual P99 per stage.
- Compare to allocation.
- Tune budgets.

The budget is a hypothesis; production data validates.

### 2.7 The per-stage observability

For the budget to work, per-stage measurement:

- Per-stage latency metric.
- Per-stage P50, P95, P99.
- Per-stage SLO.

Cross-link to [ai-engineering-reference-architecture / observability-and-telemetry / trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md).

### 2.8 The "where is the latency" diagnostic

When latency is too high:

- Open the per-stage panel.
- Find the stage exceeding budget.
- Investigate that stage specifically.

The cascade tells you where to look.

---

## 3. The cascade across stages

How latency flows; where to look.

### 3.1 The dependency graph

Most stages are sequential (each must finish before the next):

```
Network → Auth → AuthZ → Retrieval → Prompt → LLM → Validation → Side-effect → Network
```

Each contributes additively.

### 3.2 The parallel opportunities

Some stages can run in parallel:

- Auth + AuthZ (often).
- Retrieval + (other prep work).
- Side-effect writes (after response sent; doesn't affect user-visible latency).

Identify parallel opportunities; design for them.

### 3.3 The retrieval-then-LLM cascade

For RAG workloads:

```
Retrieval → LLM (with retrieved context)
```

Total: retrieval P99 + LLM P99.

If retrieval is fast (200ms P99) and LLM is slow (3000ms P99), the LLM dominates.

If retrieval is slow (1000ms P99) and LLM is medium (1500ms P99), retrieval is a significant factor.

Measure both; optimize the slower.

### 3.4 The agent cascade

For agent workloads:

```
LLM call 1 → Tool 1 → LLM call 2 → Tool 2 → ... → LLM call N
```

N steps; each adds latency.

Typical agent: 5 steps × (3s LLM + 500ms tool) = ~17 seconds.

Reductions:
- Fewer steps (better prompt engineering).
- Parallel tool calls when possible.
- Faster individual stages.

### 3.5 The side-effect-after-response pattern

For features where the response is what the user sees, side effects can happen after:

```
User request → ... → Response to user (3s)
                  → Side-effect writes (in background; ~500ms)
                  → Notification to downstream (async)
```

User-visible latency: 3s. Background work: 500ms+ separately.

Cross-link to [integration-architecture/callback-and-webhook-patterns.md](../integration-architecture/callback-and-webhook-patterns.md).

### 3.6 The frontend rendering as a stage

Often forgotten:

- Backend returns in 3s.
- Frontend takes 2s to render.
- User-visible latency: 5s.

Frontend rendering should be in the latency budget; measure it.

### 3.7 The network path

Geographic distance contributes:

- US East to US West: ~70ms RTT.
- US to EU: ~80ms.
- US to APAC: ~150ms.

For high-traffic features, regional deployment reduces network latency.

### 3.8 The cumulative tail

P99 is not additive:

- Stage A P99: 200ms.
- Stage B P99: 300ms.
- Aggregate P99: ~500ms (not necessarily; depends on correlation).

Actual aggregate P99 should be measured, not computed.

### 3.9 The "the model is slow" misdiagnosis

When latency is too high:

- Default assumption: "the model is slow; swap to a faster one."
- Reality: often other stages contribute as much.

The cascade reveals where.

---

## 4. When streaming is required

The streaming decision.

### 4.1 The interactive case

Streaming is required when:

- User is actively waiting.
- UX displays output as it arrives.
- Perceived latency matters.

Examples: chat, copilot, search-as-you-type.

### 4.2 The non-interactive case

Streaming is not required when:

- Output is consumed batch-style.
- User is not actively waiting.
- Total latency is what's measured.

Examples: API consumer that parses full JSON; batch processing; agent steps consumed internally.

### 4.3 The streaming benefit

For interactive:

- First token in 1-2s (vs full response in 5-8s).
- User sees feedback quickly.
- Perceived latency dramatically lower.

### 4.4 The streaming cost

- Implementation complexity (more code; harder testing).
- Infrastructure considerations (proxies, gateways).
- Caching complications (can't easily cache streaming).
- Some downstream consumers can't handle.

### 4.5 The "should we stream" decision

Stream when:

- User is interactive (waiting in front of the UI).
- TTFT < 2s with streaming; full response > 5s.
- Frontend can render as tokens arrive.

Don't stream when:

- Batch / async use case.
- Total response is short anyway.
- Downstream consumer doesn't support it.

### 4.6 The "we stream but cache the full result" pattern

Hybrid:

- Stream to user for UX.
- Cache the full result after streaming completes.
- Next identical query hits cache; no streaming needed.

Best of both: fast first response (cache); streaming on miss.

### 4.7 The streaming-as-perceived-latency optimization

For chat workloads:

- Median full-response latency: 4-8s.
- Median TTFT with streaming: 1-2s.
- Perceived latency: based on TTFT.

User perceives the feature as fast even though total is slower than baseline non-streaming.

---

## 5. The streaming implementation patterns

Implementation that works through your stack.

### 5.1 The SSE (Server-Sent Events) pattern

```
Server → HTTP response with Content-Type: text/event-stream
       → Streams chunks separated by \n\n
       → Client reads each chunk as event
```

**Pros.**
- HTTP-based; works with most proxies and gateways (with care).
- One-way (server → client); simpler than WebSocket.
- Native browser support.

**Cons.**
- Doesn't support bi-directional.
- HTTP/2 multiplexes but HTTP/1.1 holds connection.
- Some proxies buffer; defeats streaming.

### 5.2 The WebSocket pattern

```
Server ↔ Client over WebSocket
       Bidirectional message stream
```

**Pros.**
- Bidirectional.
- Lower overhead per message.
- Good for chat-like.

**Cons.**
- More complex; more infrastructure considerations.
- Not all proxies support.
- Connection management overhead.

### 5.3 The SSE-vs-WebSocket decision

- One-way streaming, simple UX: SSE.
- Bidirectional, complex chat with user-typing-indicator: WebSocket.
- Most AI streaming: SSE works fine.

### 5.4 The proxy-buffering issue

Many proxies buffer HTTP responses:

- nginx default: buffers up to 4-8KB before flushing.
- Some CDNs: buffer entire response.
- Load balancers: usually OK.

For streaming to work:

- Disable buffering on the path: `X-Accel-Buffering: no` (nginx).
- Configure CDN to not cache or buffer streaming endpoints.
- Test through the full stack.

### 5.5 The "streaming works in dev but not prod" surprise

Common cause: proxy in prod that's not in dev. Symptoms:

- Dev: tokens flow as expected.
- Prod: response arrives as one chunk after total latency.

Test through the full stack from day 1.

### 5.6 The HTTP/2 vs HTTP/1.1 consideration

- HTTP/1.1 + SSE: each stream holds a connection.
- HTTP/2 + SSE: multiplexed; many streams per connection.

For high-concurrent streaming, HTTP/2 is essential.

### 5.7 The disconnection handling

Streaming connections can drop:

- Network blip.
- Mobile network change.
- Client disconnects.

Server must:

- Detect disconnection.
- Stop processing (don't continue generating for disconnected client).
- Save partial state if applicable.

Cross-link to [retry-strategy.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/retry-strategy.md) for the resumption pattern.

### 5.8 The in-band error pattern

Streaming may fail mid-stream:

```
event: token
data: "Hello"

event: token
data: " world"

event: error
data: "{\"code\": \"internal_error\", \"message\": \"...\"}"
```

Client parses error event; can handle gracefully.

Cross-link to [integration-architecture/integration-failure-patterns.md §9.3](../integration-architecture/integration-failure-patterns.md).

### 5.9 The streaming-protocol consistency

For consumers of multiple AI providers:

- Anthropic SSE format.
- OpenAI SSE format (slightly different).
- Each has its own event types.

The integration layer normalizes; consumer code parses one format.

### 5.10 The streaming-test discipline

Pre-launch test:

- Send a request through full prod stack.
- Verify tokens arrive in chunks (not all at once).
- Verify TTFT meets target.
- Verify disconnection handled.
- Verify in-band errors handled.

---

## 6. Optimization techniques

How to actually move the latency line.

### 6.1 The retrieval optimization

For RAG workloads with slow retrieval:

- Index optimization (HNSW parameter tuning).
- Reduce k (top-k results).
- Index-side pre-filtering before vector search.
- Tiered retrieval (broad + narrow).

Cross-link to [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md).

### 6.2 The prompt-shortening

Shorter prompts → faster TTFT (less input to process):

- Compress retrieved context.
- Drop redundant instructions.
- Remove unused few-shot examples.

Trade-off with quality; eval-driven.

### 6.3 The smaller-model substitution

For latency-sensitive workloads:

- Haiku P99: 1.5s.
- Sonnet P99: 3.5s.

If Haiku quality is acceptable, latency drops 2x.

Cross-link to [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md).

### 6.4 The streaming-from-day-1

Don't retrofit streaming. Build the feature streaming-aware from the start:

- Backend supports stream.
- Frontend renders progressively.
- Tests cover streaming path.

Retrofitting is much harder than building it in.

### 6.5 The parallel tool calls

For agent workloads:

- Tool 1 and Tool 2 don't depend on each other → parallel.
- Saves N × tool_latency per agent step.

Cross-link to [ai-engineering-reference-architecture / agent-engineering / agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md).

### 6.6 The cache layer

For repeat queries:

- Response cache (cross-link to [caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md)).
- Cache hits served in milliseconds.

40-60% cache hit rates achievable on FAQ workloads → median latency drops by 50%+ on cache-hit traffic.

### 6.7 The pre-warm of connections

Connection-setup is slow (TCP + TLS = ~50-200ms):

- Connection pooling.
- HTTP/2 multiplexing.
- Keep-alive.

Cross-link to [timeout-strategy.md §9](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/timeout-strategy.md).

### 6.8 The regional deployment

For users far from your region:

- Network RTT contributes 100-300ms.
- Regional deployment reduces RTT.

For latency-sensitive workloads, multi-region.

### 6.9 The "we optimized but it's still slow" insight

If all stages are optimized and latency is still high:

- Model is the actual bottleneck.
- Either accept; use smaller model; or stream.

Streaming reduces perceived latency without changing total.

---

## 7. Per-workload latency targets

Different workloads have different latency expectations.

### 7.1 Real-time chat

```
TTFT: < 2s P99
Total response: < 10s P99
```

User is interactive; streaming required.

### 7.2 Background enrichment

```
Total: < 24h
```

User isn't waiting; latency-tolerant; cost-optimized.

### 7.3 Clinical decision support

```
TTFT: < 5s P99
Total: < 15s P99
```

Safety-critical; needs careful response; allowance for thoroughness.

### 7.4 Agent task

```
TTFT (first turn): < 5s
Per-step: < 15s
Total: < 60s
```

Multi-step; user expects wait; progress indicators useful.

### 7.5 Batch processing

```
Total: < 24h
```

No user waiting; throughput-optimized.

### 7.6 Search-as-you-type

```
TTFT: < 100ms (sub-second perception)
```

Very latency-sensitive; usually requires self-hosted small model.

### 7.7 Voice assistant

```
TTFT: < 300ms
Total: < 2s (perceptible delay)
```

Audio UX; very latency-sensitive.

### 7.8 The per-workload latency budget table

```
Workload                          Total target   TTFT target  Streaming?
─────────────────────────────────────────────────────────────────────────
Real-time chat                    5-10s P99      <2s          Yes
Clinical decision support         <15s P99       <5s          Optional
Care Coordinator agent task       <60s P99       <5s          Optional (progress indicator)
Document classification (batch)   <24h           N/A          No
Embedding generation              <100ms each    N/A          No
Search-as-you-type                <500ms total   <100ms       Yes
Internal copilot                  <8s P99        <3s          Yes
```

---

## 8. Worked Meridian example

Meridian's per-workload latency design.

### 8.1 Patient API chat (US)

Latency target:
- TTFT: P99 < 2s.
- Total response: P99 < 10s.

Streaming: SSE.

Stage budget:
```
Auth + AuthZ:             100ms
Per-tenant rate-limit:    20ms
Retrieval (Pinecone):     200ms
Prompt assembly:          30ms
LLM call (Sonnet) TTFT:   1200ms median; 1800ms P99
LLM call streaming rate:  ~80 tokens/sec
Side-effect writes:       (after stream complete; doesn't affect user latency)
Total TTFT:               ~1700ms P99 (within 2s budget)
Total response (400 tok): ~7s P99
```

Optimization actions taken:
- Pinecone QPS optimized; P99 dropped from 350ms to 200ms.
- Retrieval-then-LLM (not parallel because LLM needs retrieved context).
- Pre-warmed connection pool to Anthropic.

### 8.2 Care Coordinator agent task

Latency target:
- TTFT: P99 < 5s.
- Per-agent-step: P99 < 15s.
- Total task: P99 < 60s.

Streaming: progress indicator (not token stream; agent task UI shows step-by-step).

Stage budget per step:
```
Pre-step setup:            100ms
LLM call (Sonnet):         3000ms median; 7000ms P99
Tool call (avg):           400ms
Post-step state update:    50ms
Per-step P99:              ~7500ms
```

5 steps × 7.5s = 37.5s median; 75s P99.

Hit the P99 target by:
- Reducing avg agent steps from 7 to 5 (better prompt).
- Parallel tool calls (when possible).
- Prompt-prefix caching (faster LLM TTFT).

### 8.3 Document classification (batch)

Latency target: < 24h total.

Architecture: batch processing.

```
Document arrives → Queue → Batch processing (every 30 min) → Classification → Index
```

Per-document: ~5s (Llama 3 70B self-hosted).
Per-batch (1000 docs): ~30 min on 4 GPUs.

User-visible latency: < 60 min (worst case waiting for next batch + processing).

Acceptable; user isn't waiting.

### 8.4 The Q1 2026 retrieval optimization

Patient API chat P99 was 12s (target 10s).

Investigation per the cascade:
- Auth/AuthZ: 100ms ✓
- Retrieval: 600ms (above 300ms target)
- LLM call: 4200ms ✓
- Other: 200ms ✓

Retrieval was the over-budget stage.

Root cause: index size grew; HNSW parameter not tuned for new size.

Fix: parameter tuning; retrieval P99 dropped to 200ms.

Total P99 returned to 8s.

### 8.5 The Q2 2026 streaming-buffering surprise

Patient API chat moved to a new CDN. Streaming stopped working — tokens arrived as one chunk after total latency.

Root cause: CDN buffered SSE responses.

Fix: bypass CDN for `/api/stream/*` endpoints. CDN config update.

Took 4 hours from report to fix; would have been caught if tested through full stack.

### 8.6 The voice-assistant exploration

A planned feature: voice assistant for clinical scribe.

Target latency: TTFT < 300ms.

Analysis:
- Hosted Sonnet: median TTFT 1200ms; way too slow.
- Hosted Haiku: median TTFT 500ms; still too slow.
- Specialized provider (Groq): median TTFT 150ms; fits.

Architecture decision: use Groq for the voice workload; even though it's a different provider, the latency requirement is binding.

Cross-link to [multi-provider-failover.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/multi-provider-failover.md).

### 8.7 The "we built it streaming from day 1" benefit

Patient API chat was streaming-first:
- Backend designed for streaming.
- Frontend renders progressively.
- Tests cover streaming.
- Production worked from launch.

Vs Care Coordinator agent (built non-streaming initially):
- Retrofitted progress indicator later.
- Took 2x as long to add.

### 8.8 The infrastructure cost

- Streaming infrastructure: minimal additional cost (SSE is HTTP).
- Multi-region deployment: ~$8k/month for additional regional deployment.
- Connection pooling / HTTP/2: free architectural improvement.

### 8.9 The customer feedback

Voice-of-customer:
- "The chat feels really fast" (perceived latency from streaming).
- "The Care Coordinator takes a long time but I can see what it's doing" (progress indicator).

Both workloads pass the user-experience bar through different mechanisms.

### 8.10 The lessons

- The cascade is the diagnostic; without it, latency optimization is guessing.
- Streaming changes the perceived latency game.
- Implementation through the full stack matters; dev != prod.
- Some workloads have hard latency limits; pick the provider that meets them.

---

## 9. Anti-patterns

### 9.1 The "the model is slow; swap to a faster one" reflex

**Pattern.** Total latency high; first move is model swap. Model isn't the bottleneck.

**Corrective.** Cascade diagnostic per §2.8; find the actual slow stage.

### 9.2 The no-budget approach

**Pattern.** No latency budget defined; nobody knows what good looks like.

**Corrective.** Per-feature budget per §2.1.

### 9.3 The single-stage-optimization

**Pattern.** Retrieval optimized; ignored that LLM is now dominant. Marginal improvement.

**Corrective.** Optimize the bottleneck per §2.8; iterate.

### 9.4 The retrofit-streaming

**Pattern.** Feature built non-streaming; streaming added later. Painful refactor.

**Corrective.** Streaming-from-day-1 per §6.4.

### 9.5 The "streaming works in dev" assumption

**Pattern.** Streaming works in dev; doesn't in prod due to proxy / CDN buffering.

**Corrective.** Test through full stack per §5.10.

### 9.6 The no-frontend-rendering account

**Pattern.** Latency budget covers backend; frontend takes another 3s.

**Corrective.** Include frontend rendering per §3.6.

### 9.7 The "we'll measure latency later" deferral

**Pattern.** Feature launches without per-stage observability. Latency complaints arrive; nobody knows where to look.

**Corrective.** Per-stage measurement per §2.7.

### 9.8 The cumulative-tail underestimate

**Pattern.** P99 budget computed by summing P99s. Actual P99 is lower (independence) or higher (correlation).

**Corrective.** Measure actual aggregate P99 per §3.8.

### 9.9 The "we'll add caching later" miss

**Pattern.** Repeat queries served by full LLM call; cache could have served in milliseconds.

**Corrective.** Caching in §6.6.

### 9.10 The streaming-error-handling gap

**Pattern.** Streaming works on happy path; errors don't propagate cleanly.

**Corrective.** In-band error pattern per §5.8.

---

## 10. Findings (sprint-assignable)

**ARCH-LAT-001 (P0). No per-feature latency budget defined.** Latency drifts without governance. Per §2 per workload. Owner: AI platform + product.

**ARCH-LAT-002 (P0). No per-stage latency observability.** Cascade diagnostic impossible. Per §2.7. Owner: observability-eng.

**ARCH-LAT-003 (P0). Streaming tested in dev only; not through prod stack.** First prod fire reveals proxy buffering. Per §5.10. Owner: AI platform + SRE.

**ARCH-LAT-004 (P1). Interactive features not streaming.** Perceived latency higher than necessary. Per §4.1. Owner: AI platform + product.

**ARCH-LAT-005 (P1). Latency target ignores frontend rendering.** Backend optimized; total still slow. Per §3.6. Owner: AI platform + frontend.

**ARCH-LAT-006 (P1). Connection pooling absent.** Connection setup adds 50-200ms per request. Per §6.7. Owner: AI platform.

**ARCH-LAT-007 (P1). Multi-region deployment absent.** Non-local users see RTT contribution. Per §6.8. Owner: AI platform.

**ARCH-LAT-008 (P1). Agent tools serialized when could be parallel.** Per-agent-step latency 2-3x higher than necessary. Per §6.5. Owner: agent platform.

**ARCH-LAT-009 (P2). Cache not implemented for repeat queries.** Cache-hittable queries pay full latency. Per §6.6. Owner: AI platform.

**ARCH-LAT-010 (P2). Smaller-model alternative not evaluated.** Slow workload using premium model when smaller would meet capability bar. Per §6.3. Owner: AI platform.

**ARCH-LAT-011 (P2). Side-effects in user-visible path.** User waits for downstream writes that could be after-response. Per §3.5. Owner: AI platform + feature teams.

**ARCH-LAT-012 (P2). Streaming-error handling absent.** Mid-stream failures don't surface cleanly. Per §5.8. Owner: AI platform.

**ARCH-LAT-013 (P2). HTTP/1.1 used for streaming.** Connection-per-stream overhead. Per §5.6. Owner: AI platform.

**ARCH-LAT-014 (P2). Voice / sub-second workloads on standard providers.** Latency target unmeetable. Per §6.3 and §8.6. Owner: AI platform + product.

**ARCH-LAT-015 (P3). Disconnection handling absent in streaming.** Server continues generating for disconnected client. Per §5.7. Owner: AI platform.

**ARCH-LAT-016 (P3). Quarterly latency budget review absent.** Drift undetected. Per §2.6. Owner: AI platform + observability.

**ARCH-LAT-017 (P3). Latency SLOs not per-stage.** Per §2.5; per-stage SLOs. Owner: SRE + AI platform.

**ARCH-LAT-018 (P3). Frontend rendering not measured.** User-visible latency unknown. Per §3.6. Owner: frontend + observability.

---

## 11. Adoption sequencing checklist

- [ ] **Define per-feature latency budget (§2).**
- [ ] **Implement per-stage observability (§2.7).**
- [ ] **Choose streaming where appropriate (§4).**
- [ ] **Test streaming through full prod stack (§5.10).**
- [ ] **Disable proxy buffering on streaming endpoints (§5.4).**
- [ ] **Implement connection pooling (§6.7).**
- [ ] **Implement in-band error handling for streaming (§5.8).**
- [ ] **Implement disconnection handling (§5.7).**
- [ ] **Add cache layer where applicable (§6.6).**
- [ ] **Per-stage SLO + alerts (§2.5).**
- [ ] **Multi-region deployment for latency-sensitive workloads (§6.8).**
- [ ] **Quarterly latency budget review.**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — cost dimension (companion).
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — concurrency under latency constraints.
- [caching-tiers.md](./caching-tiers.md) — caching as latency lever.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) — self-hosted latency profile.

**Elsewhere in this repo.**
- [integration-architecture/sync-vs-async-vs-streaming.md](../integration-architecture/sync-vs-async-vs-streaming.md) — integration-shape decision.
- [integration-architecture/integration-failure-patterns.md](../integration-architecture/integration-failure-patterns.md) — failure handling including streaming.
- [integration-architecture/callback-and-webhook-patterns.md](../integration-architecture/callback-and-webhook-patterns.md) — async patterns.
- [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md) — latency in model selection.

**Sibling repos.**
- [ai-engineering-reference-architecture / reliability-engineering / timeout-strategy.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/timeout-strategy.md) — per-stage timeout calibration.
- [ai-engineering-reference-architecture / observability-and-telemetry / trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md) — per-stage tracing.
- [ai-engineering-reference-architecture / observability-and-telemetry / llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md) — LLM-stage observability.
- [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — retrieval optimization.
- [ai-engineering-reference-architecture / agent-engineering / agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md) — agent loop latency.

**External.**
- IETF Server-Sent Events spec.
- WebSocket protocol RFC.
- Provider SSE documentation (Anthropic, OpenAI).
- HTTP/2 RFC and best practices.
- Web Performance literature.
