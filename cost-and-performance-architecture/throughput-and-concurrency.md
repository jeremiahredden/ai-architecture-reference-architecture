# Throughput and Concurrency

> **Audience.** Architects whose AI workload's traffic is growing and the provider's rate limit suddenly matters. Tech leads designing for thousands-of-requests-per-second when current is hundreds. Anyone whose multi-tenant platform discovered that "we'll scale when we need to" arrives faster than expected. **Scope.** The *architectural* design for throughput and concurrency in AI systems: concurrency-per-instance, request-batching patterns, the model-provider rate-limit shape (RPM, TPM, both), patterns for scaling across multiple provider tenancies (Bedrock cross-account, Azure OpenAI multi-deployment), queue-and-fairness disciplines for multi-tenant systems. Not the engineering-side rate-limiting mechanics (see [ai-engineering-reference-architecture / cost-and-finops / cost-aware-rate-limiting.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-aware-rate-limiting.md)). Not the capacity planning details (see [ai-engineering-reference-architecture / reliability-engineering / capacity-planning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md)). Not the noisy-neighbor fairness (see [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI throughput planning differs from conventional service throughput in three specific ways:

- **The bottleneck is usually the provider, not your infrastructure.** Adding consumers doesn't help if the provider's RPM is binding.
- **Tokens-per-minute is a real constraint.** Beyond RPM, providers cap aggregate token consumption, which scales with prompt size.
- **Multi-account / multi-deployment is the primary throughput-scaling lever for hosted models.** Sharding across provider tenancies is how you scale beyond a single account's limit.

These differences mean:

- A team's "we'll scale by adding pods" plan stalls at provider rate limits.
- A team's TPM calculation that assumed 1000-token prompts breaks when long-context workloads ship.
- A team's "we have one Anthropic account" architecture hits a wall when traffic exceeds account limits.

This document covers the architectural patterns: concurrency-per-instance design; batching patterns; understanding the provider rate-limit shape; multi-account scaling; queue-and-fairness for multi-tenant.

This document is opinionated about four things:

1. **Throughput planning starts with the provider's rate limits, not your infrastructure.** Account-level RPM/TPM is the binding constraint for hosted models; design around it.
2. **Multi-account / multi-deployment is normal at scale.** Bedrock cross-account, Azure OpenAI multi-deployment, multiple Anthropic accounts — these are the scaling levers.
3. **Continuous batching is the throughput primitive for self-hosted.** vLLM's continuous batching produces 2-5x throughput improvement over naive batching.
4. **Concurrency-per-instance is not the bottleneck.** It's almost always the upstream (provider rate limit) or downstream (vector store, side-effect store) that binds.

Structure: (2) the throughput dimensions; (3) provider rate-limit shape; (4) concurrency-per-instance design; (5) request batching patterns; (6) multi-account / multi-deployment scaling; (7) queue-and-fairness for multi-tenant; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The throughput dimensions

What "throughput" means for AI workloads.

### 2.1 Requests per second (RPS) / per minute (RPM)

The classic metric:

- How many distinct requests can the system serve per unit time.
- Provider rate-limits commonly RPM.

### 2.2 Tokens per minute (TPM)

For AI workloads, often the binding constraint:

- Aggregate token throughput.
- Account-level limits at ~1-10M TPM for major providers.

### 2.3 Concurrent connections

How many simultaneous in-flight requests:

- For streaming, each holds a connection.
- Provider may cap concurrent connections separately.

### 2.4 Per-tenant throughput

For multi-tenant: per-tenant aggregate throughput:

- Sub-allocation of platform throughput.
- Per-tenant rate-limit budgets (cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)).

### 2.5 The dimension interaction

Workload character affects which dimension binds:

- High-volume short prompts: RPM-bound.
- Long-context workloads: TPM-bound.
- Long-running agents: concurrency-bound.

Plan for the binding dimension; over-provision for the others.

### 2.6 The throughput vs latency interaction

Higher throughput often means higher latency:

- Concurrent requests batched on GPU → individual latency rises.
- Queue depth grows → latency grows.

Architecture's job: balance both.

### 2.7 The peak vs sustained distinction

- Peak: rare burst (Monday morning, launch day).
- Sustained: continuous load.

Architecture must handle both:

- Capacity sized for sustained + reasonable peak.
- Queue or shed-load for extreme peak.

---

## 3. Provider rate-limit shape

Understanding what hosted providers offer.

### 3.1 The rate-limit dimensions

Major providers cap on:

- **RPM (requests per minute).**
- **TPM (tokens per minute).**

Both apply simultaneously; whichever is binding constrains throughput.

### 3.2 The tier system

Provider tiers:

- **Free tier.** Very limited; typical 60 RPM, 50k TPM.
- **Pay-as-you-go.** Moderate; 500 RPM, 500k TPM.
- **Production tier.** Higher; 5000 RPM, 2M TPM.
- **Enterprise / commit tier.** Negotiated; can be very high.

Each tier has its own RPM/TPM.

### 3.3 The 2026 hosted-provider rate limits (illustrative)

```
Anthropic (production tier):
  Claude Sonnet 4.6: 5000 RPM, 2M TPM
  Claude Haiku 4.5: 5000 RPM, 5M TPM (input rate higher)
  Claude Opus 4.7: 4000 RPM, 1M TPM (lower for premium)

OpenAI:
  GPT-4o: 5000 RPM, 1.5M TPM
  GPT-4o-mini: 30000 RPM, 6M TPM

AWS Bedrock (varies by region / model):
  Anthropic via Bedrock: similar to direct
  Cohere via Bedrock: variable

Azure OpenAI (per-deployment):
  Per-deployment quota; multi-deployment scaling
```

(Verify for current data; updates frequent.)

### 3.4 The per-deployment vs per-account question

Some providers (Azure OpenAI) bill at deployment level:

- Multiple deployments per account → aggregate throughput.
- Each deployment has its own quota.

Others (Anthropic, OpenAI direct) bill at account level:

- One account = one quota.
- Multi-account for more throughput.

### 3.5 The "we can't get more" reality

Sometimes the provider can't (or won't) grant more rate limit:

- Capacity-constrained.
- Customer's commit tier insufficient.
- Provider's account-level cap.

Mitigation: multi-account (§6); multi-region; self-host (cross-link to [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md)).

### 3.6 The rate-limit headers

Providers expose current usage (cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-aware-rate-limiting.md §6.1](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-aware-rate-limiting.md)):

- `X-RateLimit-Remaining-Requests`.
- `X-RateLimit-Remaining-Tokens`.
- `Retry-After`.

Architecture consumes these for headroom monitoring.

### 3.7 The provider tier upgrade path

Growing workload → upgrade tier:

- Start: pay-as-you-go.
- Volume threshold: upgrade to production tier.
- Higher volume: enterprise / commit.

Each tier has its own pricing; negotiate.

### 3.8 The "we're 80% headroom; what now"

When headroom is consistently low:

- Negotiate higher tier (timeline: weeks).
- Multi-account (timeline: days).
- Move some workloads off provider (timeline: weeks).
- Self-host high-volume workloads (timeline: months).

Plan ahead.

---

## 4. Concurrency-per-instance design

What "concurrency" means for AI consumers.

### 4.1 The per-instance limit

For a consumer pod / service:

- How many concurrent LLM calls can it serve?
- Limited by: thread pool, connection pool, memory.

Typical: 10-100 concurrent per pod.

### 4.2 The connection-pool sizing

For HTTP-based providers:

```
connection_pool_size = expected_concurrent_requests × buffer
```

Typically 1.5-2x expected concurrent.

### 4.3 The thread-pool sizing

Async services with thread pools:

- Threads sized for concurrent LLM calls.
- Plus some for background tasks.

For mostly-async workloads, thread count can be lower than concurrent calls (event loop multiplexes).

### 4.4 The memory budget

Each in-flight LLM call holds:

- Connection state.
- Request / response buffers (streaming: larger).

Memory limits the concurrent count per instance.

### 4.5 The "horizontal scaling" lever

Beyond per-instance concurrency:

- Add more pod replicas.
- Each adds another batch of concurrent slots.

Subject to upstream (provider) constraint.

### 4.6 The "pods are idle but provider is rate-limited" pattern

If provider RPM is binding:

- Adding more pods doesn't help.
- Each pod sees rate-limit pressure.

Detect this; don't over-provision.

### 4.7 The auto-scaling

For dynamic load:

- Scale-up on queue depth or latency.
- Scale-down on idle.

Considerations:

- Cold start time (CPU-based pods: seconds; GPU pods: 5-15 min — cross-link to [ai-engineering-reference-architecture / reliability-engineering / capacity-planning.md §7](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md)).
- Headroom for spike accommodation.

### 4.8 The "queue depth as scaling signal" preference

For AI workloads, queue depth is often the right signal:

- CPU / memory utilization: too coarse.
- Queue depth: directly correlated to load.

Scale-up when depth > N per replica.

---

## 5. Request batching patterns

How to maximize throughput.

### 5.1 The naive batching pattern

Multiple requests collected; sent as one batch:

```
batch = []
while batch.size < BATCH_SIZE:
    batch.append(receive())
send_batch(batch)
```

**Pros.** Reduces per-request overhead; better throughput for self-hosted.
**Cons.** Latency suffers (wait for batch to fill).

### 5.2 The dynamic batching pattern

Batch up to a max size or max wait:

```
batch = []
deadline = now + MAX_WAIT
while batch.size < MAX_SIZE and now < deadline:
    if has_request():
        batch.append(receive())
send_batch(batch)
```

**Pros.** Latency capped; throughput when batches fill quickly.
**Cons.** Still some latency cost.

### 5.3 The continuous batching pattern (self-hosted)

The vLLM / TGI pattern:

- GPU processes multiple requests simultaneously.
- New requests join the batch as they arrive.
- Existing requests continue.
- No artificial wait.

**Pros.** 2-5x throughput improvement; near-zero latency penalty.
**Cons.** Specific to compatible inference servers (vLLM, TGI, SGLang).

### 5.4 The provider-side batching (Anthropic / OpenAI Batch API)

For batch-eligible workloads:

- Submit many requests as a JSONL file.
- Provider processes asynchronously.
- 50% discount (cross-link to [ai-engineering-reference-architecture / cost-and-finops / batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md)).

**Pros.** Cost; throughput.
**Cons.** 24-hour SLA; not for real-time.

### 5.5 The when-to-batch decision

| Workload | Batch? |
| --- | --- |
| Real-time chat | No (latency-sensitive) |
| Streaming | No (incompatible) |
| Background classification | Yes (latency-tolerant) |
| Bulk embedding | Yes |
| Agent task | No (per-step) |

### 5.6 The batch-size tuning

For self-hosted continuous batching:

- vLLM's max-batch-size: tuned based on GPU memory + model.
- Higher: more throughput; risk of OOM.
- Lower: safer; lower throughput.

Per-GPU tuning.

### 5.7 The "batch but stream out" pattern

For some workloads, batch on input but stream on output:

- Batch multiple inputs onto one GPU pass.
- Each request streams its output.

Combines throughput benefit with streaming UX.

---

## 6. Multi-account / multi-deployment scaling

The primary lever beyond a single account.

### 6.1 The multi-account pattern

For Anthropic / OpenAI direct:

- Account A: 5000 RPM, 2M TPM.
- Account B (separate): another 5000 RPM, 2M TPM.
- Combined: 10000 RPM, 4M TPM.

Architecture:

- Routing layer (cross-link to [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md)) distributes across accounts.
- Each account's rate limits tracked separately.

### 6.2 The Bedrock cross-account pattern

AWS Bedrock supports cross-account model invocation:

- Account A has Bedrock provisioned-throughput.
- Account B (separate AWS account) configured for cross-account access.
- Same model, but two quotas.

Architecture leverages multiple AWS accounts.

### 6.3 The Azure OpenAI multi-deployment

Azure OpenAI's per-deployment quotas:

- Deployment 1 (East US): 100k TPM.
- Deployment 2 (West US): 100k TPM.
- Deployment 3 (East US 2): 100k TPM.

Combined: 300k TPM.

Routing layer distributes; deployment selection based on rate-limit headroom.

### 6.4 The routing algorithm

For multi-account / multi-deployment:

```python
def select_target(workload, accounts):
    eligible = [a for a in accounts if has_headroom(a, workload)]
    if not eligible:
        return None  # all at capacity
    # Choose based on policy: round-robin, least-loaded, weighted
    return weighted_choice(eligible)
```

Common policies:
- Round-robin: simple; even distribution.
- Least-loaded: prefer accounts with most headroom.
- Weighted: per-account priority.

### 6.5 The "sticky session" consideration

If a request is part of a longer interaction (conversation history; streaming context):

- May need to stay with the original account.
- Or re-establish on a different account (if context is in our store, not provider's).

Most providers don't have account-side state; re-route is fine.

### 6.6 The cost-vs-throughput trade-off

Multi-account incurs:

- More accounts to maintain.
- More API keys.
- Per-account contracts (if applicable).

The throughput benefit justifies for high-volume systems.

### 6.7 The "we're paying for unused capacity" reconciliation

When using commit tiers per account:

- Pay for committed capacity even if unused.
- Multi-account: pay for multiple commits.
- Reconcile actual usage vs commit.

Cost discipline applies.

### 6.8 The migration path

Single → multi-account migration:

1. Set up second account.
2. Build routing layer.
3. Route 10% of traffic to second account.
4. Monitor.
5. Scale up.

Gradual; not all-at-once flip.

### 6.9 The provider-side incident isolation

Multi-account also gives:

- Independence: account A's issue doesn't affect account B.
- Insulation from one account's rate-limit issue.

Modest benefit; not primary driver.

---

## 7. Queue-and-fairness for multi-tenant

For multi-tenant platforms.

### 7.1 The fairness requirement

Multiple tenants share platform throughput:

- One tenant's burst can't exhaust others.
- Fair-share enforced.

Cross-link to [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 7.2 The queue topology

Per-tenant queues (cross-link to [integration-architecture/backpressure-and-queueing.md §3](../integration-architecture/backpressure-and-queueing.md)):

- Each tenant has a queue.
- Consumer pulls from each in turn (round-robin or weighted).

### 7.3 The fairness scheduling

Common algorithms:

- **Round-robin.** Each tenant served in turn.
- **Weighted-fair queueing.** Premium tenants get more turns.
- **Deficit-round-robin.** Adapts based on processing time.
- **Priority queue.** Premium tenants always preferred.

Per-platform choice.

### 7.4 The "we have 100 tenants" scaling

For platforms with many tenants:

- Per-tenant queues: 100 queues.
- Operational overhead grows.
- Some platforms collapse to "per-tier queues" (free / standard / premium) at scale.

### 7.5 The shared-resource budget

Beyond per-tenant queues, shared resources (provider rate limit) need allocation:

- Free tier collectively: 20% of rate limit.
- Standard tier: 50%.
- Premium tier: 30% reserved.

Per-tier allocation; per-tenant budget within tier.

### 7.6 The "premium gets dedicated" pattern

For very-premium tenants:

- Dedicated provider account / deployment.
- No sharing with other tenants.

Highest cost; highest isolation.

### 7.7 The starvation prevention

Strict priority queues can starve free tier:

- Free tier requests never served if premium is always queued.
- Anti-starvation: floor capacity for free tier.

### 7.8 The aging pattern

Long-queued requests get priority bump:

- Free tier request waited > 30s: priority bumped to standard.
- Prevents indefinite starvation.

---

## 8. Worked Meridian example

Meridian's throughput / concurrency design.

### 8.1 The throughput catalog

```
Anthropic US account (production):
  RPM: 5000
  TPM: 2,000,000
  
Anthropic Premium account (separate):
  RPM: 5000
  TPM: 2,000,000
  Reserved for premium tier tenants.

Self-hosted Llama cluster:
  Effective throughput: 200k tokens/min
  Concurrent capacity: ~30 simultaneous

Pinecone vector store:
  QPS: 500
  Per-namespace QPS: ~50

Cohere CA account:
  RPM: 500
  TPM: 200k
  Canadian-residency tenants only.
```

### 8.2 The current load (Q1 2026)

```
Total RPM: ~1500 (30% of Anthropic US capacity)
Total TPM: ~700k (35% of Anthropic US capacity)
Headroom: 65-70% — comfortable
```

Comfortable; growth has headroom.

### 8.3 The Q2 2026 forecast

```
Projected Q3:
  Total RPM: ~2500 (50% capacity)
  Total TPM: ~1.4M (70% capacity)
  Headroom dropping; renegotiate or add capacity by Q4
```

### 8.4 The multi-account decision

Current architecture: single Anthropic US account + Premium account.

Why two and not one:

- Premium customer SLA isolation.
- Premium customers get reserved capacity.

Why not three or more:

- 700k TPM total; well under combined 4M TPM capacity.
- Multi-account complexity not yet justified.

Re-evaluate annually.

### 8.5 The provider tier negotiation

Annual:

- Review usage trajectory.
- Negotiate higher tier if approaching limit.
- Q4 2026 plan: upgrade to 3M TPM tier ($X premium); negotiated based on commit.

### 8.6 The continuous batching for self-hosted

Llama cluster uses vLLM continuous batching:

- Naive batching: 50k tokens/min throughput.
- Continuous: 200k tokens/min (4x).

The 4x improvement is the architectural choice; saved infrastructure investment.

### 8.7 The per-tenant queue design

```
Tenants (12):
  4 free tier → free-queue (1 queue; per-tenant message-group-id for fairness)
  5 standard tier → standard-queue (1 queue; per-tenant message-group-id)
  2 premium tier → premium-queue (1 queue; per-tenant message-group-id; routes to Premium Anthropic account)
  1 enterprise → enterprise-dedicated-account
```

Per-tier queues; per-tenant fairness within tier; routing based on tier.

### 8.8 The Q1 2026 incident (capacity)

A customer onboarded with much higher volume than projected:

- Their projected daily volume: 200k tokens.
- Actual first-week: 800k tokens/day (4x).
- They started competing for shared standard-tier TPM.

Mitigation:
- Per-tenant TPM budget enforced; their excess requests queued.
- They were notified; agreed to either reduce or upgrade tier.
- Their tier upgraded to premium.

Architecture worked; per-tenant fairness held.

### 8.9 The Q2 2026 incident (provider degradation)

Anthropic had a regional rate-limit incident:

- US East: degraded to ~50% of normal RPM.
- Other regions: normal.

Architecture response:

- Provider headroom signal updated.
- Standard tier traffic queued; latency rose.
- Premium tier on premium account: unaffected (separate account).
- Care Coordinator queued (standard tier).
- Patient API chat queued.
- After 35 min, resolved.

Recovery: smooth.

Lessons:
- Multi-account isolation worked for premium.
- Standard tier did experience latency spike.
- Possible mitigation: multi-region for standard tier; deferred for cost reasons.

### 8.10 The capacity dashboard

```
Real-time view:
  Anthropic US account:
    RPM headroom: 70%
    TPM headroom: 65%
    [healthy]
    
  Anthropic Premium account:
    RPM headroom: 80%
    TPM headroom: 80%
    [healthy]
  
  Self-hosted Llama cluster:
    GPU utilization: 60%
    Queue depth: ~5 requests
    [healthy]
  
  Pinecone:
    QPS: 280 (56% of 500 cap)
    [healthy]
  
  Cohere CA:
    RPM headroom: 50%
    TPM headroom: 55%
    [healthy]
```

Dashboard reviewed weekly; deeper in monthly cost review.

### 8.11 The infrastructure cost

- Multi-account setup: ~1 week initial; ~0.1 FTE ongoing for management.
- Multi-region: ~3 weeks for Canadian rollout; ~0.1 FTE ongoing.
- Continuous batching infrastructure: included with vLLM.

Total: ~0.2 FTE / month for throughput / capacity / concurrency management.

### 8.12 The lessons

- Provider rate limits are the dominant constraint for hosted; understand them.
- Multi-account is the primary scaling lever.
- Continuous batching is essential for self-hosted.
- Per-tenant fairness needs to be enforced architecturally.
- Annual review of capacity trajectory prevents surprises.

---

## 9. Anti-patterns

### 9.1 The "add more pods" reflex

**Pattern.** Throughput problem; add more pods; provider RPM still binding; pods sit idle.

**Corrective.** Diagnose binding constraint per §3.

### 9.2 The single-account-at-volume

**Pattern.** One provider account at high volume; first burst exhausts; cascade.

**Corrective.** Multi-account per §6.

### 9.3 The naive batching for self-hosted

**Pattern.** Wait for batch to fill; latency penalty; 1/5 of continuous-batching throughput.

**Corrective.** vLLM / TGI / SGLang continuous batching per §5.3.

### 9.4 The "we'll add provider negotiation later" surprise

**Pattern.** Headroom drops to 5%; provider negotiation needs 2-4 weeks; production hits limit.

**Corrective.** Quarterly negotiation per §3.7.

### 9.5 The per-tenant fairness theatre

**Pattern.** Per-tenant queues exist; not actually enforced; one tenant exhausts shared budget.

**Corrective.** Real enforcement per §7.

### 9.6 The provider-headroom-not-monitored gap

**Pattern.** Provider rate-limit metric not exposed; team learns about exhaustion from 429 storm.

**Corrective.** Headroom monitoring per §3.6.

### 9.7 The dynamic-batching for latency-sensitive workloads

**Pattern.** Batched user-facing chat. Per-call latency rises 2x.

**Corrective.** No batching for latency-sensitive per §5.5.

### 9.8 The auto-scale-on-CPU for GPU

**Pattern.** GPU pods auto-scale on CPU utilization; CPU is fine; queue depth growing.

**Corrective.** Queue-depth signal per §4.8.

### 9.9 The "we have unused capacity in one region" miss

**Pattern.** One region at 90% capacity; another region at 20%; no routing across.

**Corrective.** Cross-region routing per §6.

### 9.10 The "starvation is fine" assumption

**Pattern.** Free tier never served when premium queue is busy; free customers leave.

**Corrective.** Anti-starvation per §7.7.

---

## 10. Findings (sprint-assignable)

**ARCH-TPC-001 (P0). Provider rate-limit headroom not monitored.** First exhaustion is a surprise. Per §3.6. Owner: AI platform + SRE.

**ARCH-TPC-002 (P0). Single-account-at-volume.** Bursts cascade. Multi-account per §6. Owner: AI platform + engineering management.

**ARCH-TPC-003 (P0). Per-tenant fairness not enforced.** Cross-tenant impact. Per §7. Owner: AI platform.

**ARCH-TPC-004 (P1). Self-hosted using naive batching.** Throughput sub-optimal. Continuous batching per §5.3. Owner: AI platform.

**ARCH-TPC-005 (P1). Provider tier negotiation reactive.** Surprises arrive. Annual negotiation per §3.7. Owner: engineering management + procurement.

**ARCH-TPC-006 (P1). Auto-scale signal is CPU / memory.** Doesn't reflect AI load. Queue-depth signal per §4.8. Owner: AI platform.

**ARCH-TPC-007 (P1). Multi-region routing absent.** Geographic load imbalance. Per §6. Owner: AI platform.

**ARCH-TPC-008 (P2). Multi-deployment for Azure OpenAI not used.** Throughput limited. Per §6.3. Owner: AI platform.

**ARCH-TPC-009 (P2). Sticky-session routing for conversations not designed.** Re-routing might lose context. Per §6.5. Owner: AI platform.

**ARCH-TPC-010 (P2). Provider-side batching not used for batch-eligible.** 50% cost savings missed. Cross-link to [batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md). Owner: AI platform.

**ARCH-TPC-011 (P2). Anti-starvation not implemented.** Free tier sometimes never served. Per §7.7. Owner: AI platform.

**ARCH-TPC-012 (P2). Aging pattern absent.** Long-queued requests starve indefinitely. Per §7.8. Owner: AI platform.

**ARCH-TPC-013 (P2). Per-tier throughput allocation not designed.** Tier boundaries unclear. Per §7.5. Owner: AI platform + product.

**ARCH-TPC-014 (P3). Connection pool sized too small / too large.** Either contention or memory waste. Per §4.2. Owner: AI platform.

**ARCH-TPC-015 (P3). Capacity dashboard absent.** Decision-makers blind. Per §8.10. Owner: observability-eng.

**ARCH-TPC-016 (P3). Quarterly capacity forecasting.** Sub-annual to anticipate trends. Per §8.3. Owner: AI platform + SRE.

**ARCH-TPC-017 (P3). Provider-side incident isolation not designed.** Multi-account isolation incidental. Per §6.9. Owner: AI platform.

**ARCH-TPC-018 (P3). Self-hosted GPU pool not right-sized for continuous-batching.** Under-utilized. Per §5.6. Owner: AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Build provider rate-limit headroom monitoring (§3.6).**
- [ ] **Set per-tenant fairness queue structure (§7).**
- [ ] **For self-hosted, deploy continuous batching (§5.3).**
- [ ] **Implement multi-account routing when volume warrants (§6).**
- [ ] **Quarterly capacity forecast (§8.3).**
- [ ] **Provider tier negotiation cadence (§3.7).**
- [ ] **Queue-depth-based auto-scaling for GPU (§4.8).**
- [ ] **Multi-region deployment per workload geographics.**
- [ ] **Capacity dashboard (§8.10).**
- [ ] **Document per-tenant throughput budgets.**
- [ ] **Anti-starvation enforcement (§7.7).**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — cost dimension (companion).
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — latency dimension (companion).
- [caching-tiers.md](./caching-tiers.md) *(coming)* — caching benefit.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) *(coming)* — self-hosted detail.

**Elsewhere in this repo.**
- [integration-architecture/backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md) — queue topology.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — multi-tenant fairness.
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — multi-region patterns.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — routing layer.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — self-hosted decision.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / cost-aware-rate-limiting.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-aware-rate-limiting.md) — rate-limit mechanics.
- [ai-engineering-reference-architecture / reliability-engineering / capacity-planning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md) — capacity planning operational.
- [ai-engineering-reference-architecture / reliability-engineering / multi-provider-failover.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/multi-provider-failover.md) — multi-provider patterns.

**External.**
- Provider rate-limit documentation (Anthropic, OpenAI, AWS Bedrock, Azure OpenAI).
- vLLM, TGI, SGLang continuous-batching documentation.
- Kubernetes HPA, KEDA, GPU-aware auto-scaling.
- Weighted-fair queueing literature.
