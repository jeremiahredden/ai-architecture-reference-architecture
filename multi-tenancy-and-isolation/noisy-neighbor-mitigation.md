# Noisy Neighbor Mitigation

> **Audience.** Architects of multi-tenant AI platforms. SaaS teams whose first paying customer just turned into a small portfolio of customers and the second incident report from customer C reads "we got slower when customer A onboarded their new feature." Anyone who has to explain to a regulated customer why their tenant's latency depends on what tenants A and B are doing today. **Scope.** The *architectural* decisions: the dimensions of multi-tenant contention in AI systems (token, RPS, GPU, vector-index hot-key, downstream side-effect); per-tenant budget enforcement; per-tenant model-tier routing; per-tenant cost circuit-breaker; detection and observability of noisy-neighbor events; customer-facing assurance and SLA contracts. Not the queue topology and fairness mechanics themselves (see [integration-architecture/backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md), which is the companion). Not the per-tenant vector-store separation (see [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md)). Not the cross-tenant leakage controls (see [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Multi-tenant AI platforms have specific failure modes that single-tenant systems don't. The general principle — "one tenant's bad day shouldn't be another tenant's outage" — is well-understood in SaaS. The mechanisms for guaranteeing it in AI systems are different.

The distinctive issues:

- **Shared model provider.** All tenants typically share one provider account (Anthropic, OpenAI, Google). The provider enforces account-level rate limits. Tenant A's burst exhausts the account's TPM; tenants B and C get 429s. No tenant-level isolation at the provider.
- **Shared GPU pool (self-hosted).** All tenants share the inference cluster. Tenant A's long-context batch saturates the GPU; tenants B and C wait. No tenant-level isolation at the hardware.
- **Shared vector store.** All tenants' embeddings live in the same (logically) index. Tenant A's high-volume queries warm a cache that displaces tenant B's. Tenant A's index updates cause re-indexing pauses that affect tenant B's reads.
- **Shared downstream side-effect stores.** All tenants' write rate hits the same downstream (warehouse, audit log, notification service). Tenant A's burst saturates downstream; tenant B's writes back up.
- **Cost contagion.** Tenant A's runaway agent generates $40k in unplanned LLM cost; that cost lands on your AWS bill, not theirs. Without per-tenant cost caps, financial damage is uncapped.

Each of these dimensions has its own mitigation; collectively, they constitute the noisy-neighbor mitigation strategy. The architecture has to address all of them, because a system that's airtight on five dimensions and leaky on one is still vulnerable on the leaky one.

The companion document, [backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md), covers the *queueing topology* — how requests are organized, the fairness disciplines, the shed-load patterns. This document covers the *isolation* perspective — per-tenant budget enforcement across all dimensions, detection of noisy neighbors before SLA-violation pages, customer-facing assurance, and the runbook for when isolation isn't holding.

This document is opinionated about four things:

1. **Noisy-neighbor mitigation is multi-dimensional.** A single-dimension solution (per-tenant RPS only) leaves four other dimensions unprotected. Tenant A respecting RPS can still exhaust TPM, monopolize GPU, hot-key the vector index, or spike $/day. The architecture must address every dimension that matters.
2. **Per-tenant cost cap is non-negotiable past the first paying customer.** The "tenant runaway agent" scenario is not hypothetical; it has happened at every multi-tenant AI vendor that's been in market for more than a year. The cost cap doesn't require sophistication; it just has to exist.
3. **Detection precedes mitigation.** "We have per-tenant rate limits" without "we know when noisy-neighbor events happen" means the first you'll hear of it is the customer complaint. The dashboard that correlates per-tenant traffic with per-tenant latency must exist before the first multi-tenant incident.
4. **The customer contract is the source of truth.** Multi-tenant SLA terms determine what isolation guarantees are commercially load-bearing. Architecture serves the contract; the contract serves the customer. Vague isolation promises produce vague isolation, which produces incidents.

Structure: (2) the dimensions of multi-tenant contention; (3) per-tenant budget enforcement; (4) per-tenant model-tier routing; (5) per-tenant cost circuit-breaker; (6) detection and observability; (7) customer-facing assurance; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The dimensions of multi-tenant contention

The first move in any multi-tenant AI mitigation strategy is enumerating which dimensions of contention exist for your system. The list below is the typical universe; your system may have more.

### 2.1 Request rate (RPS / RPM)

The most familiar dimension. Tenant A makes too many requests; provider's RPM is exhausted; everyone gets 429s.

**Where it bites.**
- Hosted-model provider's account-level RPM.
- Your API gateway's per-IP or per-endpoint RPM.
- Downstream side-effect APIs' per-caller RPM.

**Why AI workloads make it tricky.** A single tenant request can fan out: a chat session is one user message but may produce 5-10 LLM calls (retrieval, planning, tool calls, final response). RPS at the *user* level is a poor predictor of RPS at the *LLM provider* level. Per-tenant LLM-call RPM must be tracked separately from per-tenant API RPM.

### 2.2 Token consumption (TPM / TPS)

Less familiar to teams new to AI. Tenants don't all consume tokens at the same rate per request; long-context workloads (RAG, summarization, agents) consume 10-100x more tokens per request than short-form classification.

**Where it bites.**
- Provider's account-level TPM (often the binding constraint for long-context workloads).
- Your self-hosted GPU's effective token-throughput.

**Why AI workloads make it tricky.** A tenant with 100 RPM but 8000-token average prompts consumes 800k TPM. A tenant with 1000 RPM and 200-token average prompts consumes 200k TPM. RPM-only budgets systematically misallocate; the first tenant's smaller-RPM-but-larger-TPM workload silently dominates.

### 2.3 GPU pool saturation (self-hosted models)

For workloads with self-hosted models, the GPU pool is the bottleneck. A tenant with long-context batch jobs holds GPU memory and compute; concurrent tenants' interactive requests queue behind it.

**Where it bites.**
- vLLM / TGI / Triton's internal queue.
- Per-pod GPU memory (a long-context request can OOM a pod).
- Cross-tenant batch-size mixing inefficiencies (vLLM's continuous batching helps; not all systems support it).

**Why AI workloads make it tricky.** GPU contention shows up as latency, not as 429s. Tenant B's P99 climbs while tenant A is running batch; neither system reports an "error." Detection requires explicit per-tenant latency monitoring.

### 2.4 Vector store / retrieval index contention

Multi-tenant retrieval workloads share index infrastructure. Contention shows up as:

- Query QPS limits at the vector store (per-tenant or global).
- Cache eviction. Tenant A's high-volume queries warm the vector store's cache; tenant B's queries miss cache and pay slow retrieval.
- Index update interference. Tenant A's bulk re-indexing blocks tenant B's reads on shared infrastructure (depending on the store's locking semantics).
- Hot-key effects. Tenant A's queries cluster around a few high-frequency keys; tenant B's queries hit different keys with worse cache locality.

**Where it bites.**
- Pinecone / Weaviate / pgvector at scale.
- OpenSearch / Elasticsearch with k-NN under heavy update load.

**Why AI workloads make it tricky.** Retrieval latency doesn't map cleanly to per-tenant ownership; it's an emergent property of cross-tenant access patterns. Mitigation requires either per-tenant index (operationally heavy) or careful capacity planning + access-pattern monitoring on shared indices.

### 2.5 Downstream side-effect store contention

The AI's output writes to downstream stores (warehouse, audit log, notification service, EHR-staging). Tenants share these stores' write capacity.

**Where it bites.**
- Downstream APIs' per-caller rate limits.
- Database write contention (per-table or per-partition).
- Message broker per-topic throughput.

**Why AI workloads make it tricky.** The AI's output volume is non-uniform; one tenant's "ingest 1M historical documents" backfill produces 1M downstream writes that compete with another tenant's steady 100/sec interactive load.

### 2.6 Cost ($/day)

The financial dimension. Tenant A's misbehaving agent (runaway loop, fine-tuning attempt, abuse) produces unbounded LLM spend.

**Where it bites.**
- Your AWS bill.
- The provider's monthly cap (if you have one and the tenant blows through it).

**Why AI workloads make it tricky.** A single misbehaving agent can spend $1000 in an hour. Conventional API rate limits don't catch it (they limit calls per second; agent fits within those limits but each call is expensive). Cost cap must be on $/period, not on RPS.

### 2.7 The dimensions matrix

| Dimension | What it limits | Detection metric | Common cause |
| --- | --- | --- | --- |
| RPM | Provider account, API gateway | 429 rate | High-volume tenant |
| TPM | Provider account | TPM exhaustion, latency | Long-context tenant |
| GPU (self-hosted) | Inference cluster | Latency, queue depth | Long-context or batch tenant |
| Vector index | QPS, cache hit, indexing pause | Retrieval latency, cache hit rate | Heavy query or bulk update tenant |
| Downstream side-effect | DB writes, downstream API | Downstream latency, error rate | Bulk writer tenant |
| Cost ($/day) | Your bill | $/tenant/period | Runaway agent or abuse |

Most platforms address one or two; the airtight ones address all six.

---

## 3. Per-tenant budget enforcement

The mechanism for "tenant A cannot exceed N." Spans all dimensions in §2.

### 3.1 The budget catalog

For each tenant, the architecture maintains explicit budgets:

```yaml
tenant: meridian-southwest
budgets:
  api_rpm: 500
  api_rpm_burst: 1000  # 1-min burst
  llm_rpm: 200
  llm_tpm: 600_000
  gpu_concurrent_long_context: 3
  vector_qps: 100
  downstream_write_rps: 50
  cost_daily_usd: 200
  cost_monthly_usd: 5000
priority: standard
```

Each budget is enforced by a separate mechanism. RPM by API gateway throttling. TPM by token-bucket rate limiter in the AI consumer. GPU by per-tenant concurrency cap. Vector QPS by vector-store-side per-tenant key. Cost by daily/monthly cost circuit-breaker (§5).

### 3.2 The enforcement mechanisms

**API rate limit (RPM/RPS).** Standard API gateway pattern. Tenant identification via API key or JWT claim; rate limit per tenant. AWS API Gateway, Kong, Envoy, custom — any production gateway supports this.

**LLM rate limit (RPM and TPM).** Per-tenant token bucket, typically Redis-backed. The AI consumer acquires from the tenant's bucket before calling the provider; if empty, queue or shed. Tracked separately from API rate limit because LLM call rate ≠ API call rate.

**GPU concurrency limit.** Per-tenant semaphore on the inference scheduler. Tenant A can have at most N concurrent long-context requests in the GPU pool; further requests queue.

**Vector QPS limit.** Vector-store-side, where supported (Pinecone supports per-API-key throttle). Otherwise, application-side rate limiter wrapping the vector store calls.

**Downstream write limit.** Per-tenant token bucket wrapping the downstream API calls. Or downstream-side per-tenant API key rate limit if downstream supports it.

**Cost limit.** Per-tenant cost circuit-breaker; see §5.

### 3.3 Budget allocation policy

Budgets are not arbitrary; they should reflect commercial reality:

- **Sum of budgets > fleet capacity.** Tenants rarely use their full budget simultaneously; over-provisioning is normal. The fairness layer ensures no single tenant can take more than its budget when capacity is tight.
- **Tiered budgets by commercial plan.** Free, standard, premium — each tier has different budgets. Budgets are advertised; tenants know what they're paying for.
- **Burst above sustained.** Each budget has a sustained rate (e.g., 200 RPM) and a burst rate (e.g., 1000 RPM for 1 minute). Bursts accommodate normal traffic spikes.
- **Negotiated budgets.** Enterprise tenants negotiate specific budgets; the architecture supports tenant-specific overrides on the standard tier.

### 3.4 The "soft" vs "hard" enforcement choice

**Soft enforcement.** Tenant exceeds budget; warning logged; request proceeds. Useful during ramp-up; risky in production (no actual isolation).

**Hard enforcement.** Tenant exceeds budget; request rejected with 429 (or queued). Isolation guarantee holds.

**Hybrid.** Hard enforcement for the binding budgets (cost, GPU, downstream); soft enforcement with auto-escalation for the soft budgets (RPM warning at 80%, hard at 100%).

**Recommendation.** Hard enforcement on all budgets for production. Soft enforcement is a transitional posture during ramp-up to gather data on real usage patterns.

### 3.5 Tenant-visible budget metadata

Tenants should be able to see their own budgets and current consumption:

```http
GET /api/v1/account/quotas

{
  "tenant": "meridian-southwest",
  "tier": "standard",
  "budgets": {
    "api_rpm": { "limit": 500, "current": 124, "reset_at": "..." },
    "llm_rpm": { "limit": 200, "current": 87, "reset_at": "..." },
    "cost_daily_usd": { "limit": 200, "spent": 47.32, "reset_at": "..." }
  }
}
```

Without visibility, tenants can't self-manage; they hit limits without warning, file support tickets, and you spend time explaining what they could have seen on their own dashboard.

### 3.6 Pre-flight cost estimation

For high-cost workloads, the architecture can pre-flight-estimate cost before committing:

- "This agent task will cost approximately $0.85 (based on expected input + output tokens at current model rates)."
- "Your remaining daily budget is $12.45."
- "Proceed?"

Useful for batch jobs and large requests; protects tenants from accidental large spends. Implementation: token count of input + estimated output tokens × per-token cost. Surfaces as response metadata or as a dedicated estimation endpoint.

---

## 4. Per-tenant model-tier routing

Premium tenants get larger / better / faster models by default. The architecture must enforce this; "we route premium to Sonnet" without enforcement is a hope, not a guarantee.

### 4.1 The pattern

Each tenant has a configured model tier:

```yaml
tenants:
  meridian-southwest:
    primary_model: claude-sonnet-4-5
    fallback_model: claude-haiku-4-5
    tier: standard
  meridian-coastal:
    primary_model: claude-opus-4-7
    fallback_model: claude-sonnet-4-5
    tier: premium
  acme-startup-trial:
    primary_model: claude-haiku-4-5
    fallback_model: null  # no fallback for free tier
    tier: free
```

The AI consumer resolves the model from the tenant's configuration, not from a global default. Premium tenants get premium models; free tenants get the cheap tier.

### 4.2 The fairness implications

Premium-only models are *capacity-separated*: premium model deployments don't share with standard. If premium uses Opus and standard uses Sonnet, the Opus capacity is isolated. A standard tenant's burst doesn't impact premium because they don't share the same model deployment's rate limit.

**Pattern.** Separate provider accounts per tier. Premium tenants on one Anthropic account; standard on another; free on a third. Provider rate limits apply per account; cross-tier contention is impossible at the provider level.

**Pattern.** Shared provider account with per-tier reserved capacity. Single account; rate-limit budget split (e.g., 30% reserved for premium). Less operational overhead; provides isolation at the consumer-side budget level.

**Pattern.** Same model for all tiers but different concurrency budgets. All tenants use Sonnet, but premium tenants get higher concurrency. Simpler; less differentiation; appropriate when model quality doesn't differ across tiers.

### 4.3 The model upgrade ladder

Tenants can upgrade tiers. The architecture must support tier change without code change:

- Tenant tier is in configuration, not hardcoded.
- Tier change is reflected at next request (or at end of current session, depending on tier-change semantics).
- Tier downgrade is reversible (didn't lose data; just lower model).

### 4.4 The "premium model failed; fall back to standard" decision

When the premium model is unavailable, options:

- **Fall back to standard model.** Premium tenant gets standard quality during the outage. Acceptable if the tenant's contract allows degradation during outage.
- **Refuse.** Premium tenant gets a structured error indicating the premium model is unavailable. Conservative; explicit; preserves contract terms.
- **Wait and retry.** Hold the request until premium is available. Risky; can produce long latency.

The choice is per-feature and per-contract. Customer-facing chat: fall back. Regulated workflow: refuse. The contract should be explicit.

### 4.5 The cost differential

Premium models cost more. The architecture must:

- Charge premium tenants differently (per-tenant cost attribution to surface the differential).
- Avoid accidentally using premium models for non-premium tenants (the "intern accidentally hardcoded the model" failure mode).

**Pattern.** Audit per-request: which model was used, which tier requested it. Drift (e.g., a free-tier tenant whose calls are landing on premium) surfaces as an alert.

---

## 5. Per-tenant cost circuit-breaker

The financial backstop. Without it, a runaway agent can produce thousands of dollars of unplanned spend on a free tenant.

### 5.1 The two-threshold pattern

Each tenant has a cost budget with two thresholds:

- **Warning threshold (80% of budget).** Notification to tenant ("you've used 80% of your daily budget"); operator alert; no enforcement action.
- **Hard threshold (100% of budget).** Subsequent requests refused with structured error; tenant notified; operator paged.

Budgets configurable per tier; defaults set by commercial policy.

### 5.2 The enforcement mechanism

Cost tracking happens at LLM call time. Each call records:

- Tenant ID.
- Cost (input_tokens × input_rate + output_tokens × output_rate, where rates are per-model).
- Timestamp.

Aggregated per tenant per period (typically per UTC day; per month for monthly caps).

**Pattern.** Real-time aggregation in Redis (sorted set or counter); per-call decrement of remaining budget.

**Pattern.** Pre-flight check before each LLM call. If estimated call cost exceeds remaining budget, refuse before incurring spend.

**Pitfall.** Estimation accuracy. Estimated cost (based on input tokens, expected output tokens) can be wrong; actual cost may exceed estimate. Add a safety margin (estimate 1.5x). Reconcile actuals post-call.

### 5.3 The "what counts as cost" definition

**LLM calls only.** Simplest; covers the dominant cost.

**LLM + embedding + retrieval.** Broader; includes vector store query cost and embedding generation cost. Closer to true cost.

**LLM + embedding + retrieval + downstream APIs.** Full attribution; closest to true cost; most complex to implement.

Start narrow; expand as the cost components grow. Most platforms start with LLM-only and expand to include embedding within the first six months.

### 5.4 The tenant-isolated circuit

The critical property: tenant A hitting the cap does not affect tenant B. The circuit-breaker is per-tenant, not global.

**Pattern.** Per-tenant cost-tracking key; per-tenant breaker state. Tenant A's breaker tripping has no effect on tenant B's cost tracking or breaker state.

**Pitfall.** Shared cost-tracking infrastructure that fails or is slow can affect all tenants. The cost-tracking service must have per-tenant isolation in its own architecture (sharded by tenant ID, or strict latency budgets so a slow tenant doesn't block others).

### 5.5 The "what happens to in-flight work" decision

When the breaker trips, what happens to currently-running multi-step agents?

**Option A: Let in-flight complete; refuse new.** The 5-step agent that's on step 3 finishes; new submissions are refused. Smoothest tenant experience; allows tenant to recover cleanly.

**Option B: Kill in-flight at breaker trip.** The agent on step 3 is killed; partial state cleaned up. Strongest cost protection; tenant experience is rough.

**Option C: Per-step check.** Each step checks the cost budget before running. The agent fails at the next step boundary if the breaker is tripped. Middle ground.

**Recommendation.** Option A for most workloads (graceful) with Option C overlay for very high-cost agents (specific runaway-agent protection).

### 5.6 The cost dispute path

Tenants will dispute. "Why was I billed $X?" The architecture must:

- Per-call cost log queryable by tenant (each call's input tokens, output tokens, model, cost).
- Cost reconciliation API: "show me the calls that drove cost on date Y."
- Audit trail tying call to user / agent / feature.

Without this, the support burden during the first cost dispute is significant. With it, tenants self-serve.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) and [cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) in the engineering sibling for the implementation-level mechanics.

---

## 6. Detection and observability

Without detection, the first time you hear about a noisy-neighbor event is from the affected tenant. The architecture's job is to surface the event internally before it becomes a customer complaint.

### 6.1 The metrics that matter

**Per-tenant traffic metrics.** RPS, TPM, LLM call rate, vector queries, downstream writes, cost. Per-tenant, per-minute granularity.

**Per-tenant latency metrics.** P50, P95, P99 of end-to-end request latency, per tenant. Distinct from global latency; the goal is to spot per-tenant degradation.

**Cross-tenant correlation.** When tenant A's traffic spikes, does tenant B's latency degrade? The correlation, displayed visually, is the noisy-neighbor signal.

**Per-tenant budget utilization.** Percentage of each budget consumed; trends over time; tenants approaching limits.

**Provider rate-limit headroom.** Account-level RPM/TPM utilization; how much room remains.

### 6.2 The dashboard layout

A noisy-neighbor dashboard typically has these panels:

1. **Top tenants by traffic (last hour).** RPS, TPM, cost. Stack-ranked. Spikes are visible.
2. **Per-tenant latency heatmap.** Y-axis: tenants. X-axis: time. Color: P99 latency. Heat-shifts during incidents.
3. **Cross-tenant impact panel.** Tenant A's traffic overlaid with tenant B's latency. Correlation is visible.
4. **Provider rate-limit headroom.** Time series; alerts when headroom < 20%.
5. **Per-tenant budget consumption.** Tenants near limits highlighted.
6. **Cost spike detection.** Per-tenant $/hour anomaly detection; alerts on spikes.

### 6.3 The alerts

- **Per-tenant traffic spike.** Tenant's traffic > X std dev above its own baseline. Often the first signal of a runaway agent or abuse.
- **Per-tenant latency degradation.** Tenant's P99 > Y% above its own baseline. Often the first signal of cross-tenant impact.
- **Cross-tenant correlation.** When two unrelated tenants degrade simultaneously, the shared infrastructure (provider, GPU, vector store) is the likely cause.
- **Provider rate-limit headroom low.** Account-level utilization > 80%.
- **Cost spike per tenant.** $/hour > Z above tenant's baseline. Runaway agent signal.
- **Per-tenant budget breach.** Hard threshold hit. Page on-call.

### 6.4 The runbook

When a noisy-neighbor event is detected:

1. **Identify the noisy tenant.** Top-traffic panel; cost-spike panel.
2. **Identify affected tenants.** Per-tenant latency heatmap; correlation panel.
3. **Decide the mitigation.**
   - Tighten the noisy tenant's budget (temporary cap reduction).
   - Apply a hold on the noisy tenant pending investigation.
   - Communicate to affected tenants ("we're investigating elevated latency").
4. **Investigate the noisy tenant.**
   - Misconfigured client?
   - Abuse?
   - Runaway agent (loop, misbehaving prompt)?
   - Legitimate growth?
5. **Resolve.**
   - If misconfigured: notify tenant; advise correction.
   - If abuse: invoke abuse policy.
   - If runaway: stop the offending workflow; investigate root cause.
   - If legitimate growth: negotiate budget increase.

### 6.5 Post-incident review

Every noisy-neighbor event that affected another tenant gets a post-incident review:

- What was the impact (how many tenants, how long, how much latency degradation)?
- What was the cause (runaway agent, legitimate spike, misconfigured client)?
- What detection signal fired first; how long until human action?
- What can be done to detect earlier or mitigate faster?
- Is a budget adjustment or architecture change warranted?

Treating each event seriously, even if no customer noticed, prevents the slow drift toward "noisy neighbors are normal."

---

## 7. Customer-facing assurance

Multi-tenant customers want to know they're protected from other tenants. The architecture serves the commercial promise.

### 7.1 The SLA terms that matter

Typical SLA elements for multi-tenant AI:

- **Latency SLO.** P99 < N seconds for interactive requests. Per-tier different (premium tighter than standard).
- **Availability SLO.** Service available 99.9% (or 99.95% for premium) per month.
- **Cross-tenant isolation guarantee.** "Your performance is not affected by other tenants' load." (Strong guarantee; requires architecture to back it.)
- **Quota guarantee.** "Your provisioned budgets are guaranteed available; you can use up to your budget without queue contention from other tenants."

The architecture must back each promise. "We have per-tenant rate limits" doesn't fully back "your performance is not affected by other tenants' load" — vector index contention and shared GPU still produce cross-tenant impact.

### 7.2 The customer-facing dashboard

Premium customers expect dashboards. Standard customers benefit from them too:

- Per-tenant usage (matching the budget catalog).
- Per-tenant latency (the customer's own P50, P95, P99 over the last 24 hours).
- Per-tenant cost (current spend, projected month-end).
- Per-tenant incident history (any operator-acknowledged events that affected this tenant).

Without customer-facing visibility, customers infer isolation from anecdotes ("seemed slow yesterday"). With it, customers see facts and trust the architecture.

### 7.3 The compliance assurance

Regulated customers (HIPAA, SOC 2, ISO 27001) need formal assurance of isolation:

- **Architecture documentation:** the layered isolation patterns; the budget enforcement; the cross-tenant leak controls (cross-link to [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md)).
- **Audit log:** per-tenant access log showing only the tenant's own data was accessed; cross-tenant access is logged and reviewed.
- **Annual or quarterly attestation:** an independent review or self-attestation that the isolation controls are operating.
- **Incident notification:** in the event of a cross-tenant leak (rare; should be zero), notification per regulatory or contractual obligation.

### 7.4 The "dedicated infrastructure" option

For customers whose isolation requirements exceed what shared infrastructure can provide:

- Dedicated model deployment (separate provider account or separate self-hosted cluster).
- Dedicated vector store (separate index or separate cluster).
- Dedicated GPU pool (separate inference deployment).
- Dedicated downstream sinks (separate database or schema).

The trade-off: dedicated infrastructure is expensive (often 3-10x shared); only some customers' compliance or contractual requirements justify it. Cross-link to [isolation-models.md](./isolation-models.md) for the broader architecture choice.

### 7.5 The "what we won't promise" honesty

Some isolation guarantees are infeasible at acceptable cost on shared infrastructure:

- **Bit-perfect performance isolation.** Vector index latency depends on cross-tenant access patterns; eliminating cross-tenant effect requires per-tenant index, which is expensive.
- **Per-tenant model versioning.** All tenants typically use the same model versions; per-tenant model versioning is expensive.
- **Per-tenant security patch timing.** Patches affect the shared infrastructure; per-tenant timing is infeasible.

The contract should be honest about what's guaranteed and what's best-effort. "Latency SLO is contractual; per-query consistency in vector retrieval is best-effort" is more honest than vague isolation promises.

---

## 8. Worked Meridian example

Meridian's Patient API AI-Assist SaaS runs 12 paying tenants on shared AI infrastructure. The tenant mix ranges from a 50-user small clinic to a 5000-user regional health system. The architecture has had three near-misses in the last 18 months; the controls have prevented all three from becoming SLA incidents.

### 8.1 The tenant catalog

```
12 paying tenants:
- 4 free trial tenants (each < 100 users)
- 5 standard tenants (each 100-500 users)
- 2 premium tenants (1000-2000 users each)
- 1 enterprise tenant (5000 users; negotiated dedicated capacity)
```

The enterprise tenant has dedicated provider account + dedicated vector index + dedicated downstream sinks. The others share infrastructure.

### 8.2 The budget catalog

Defaults by tier; per-tenant overrides where negotiated:

```yaml
tiers:
  free:
    api_rpm: 60
    llm_rpm: 30
    llm_tpm: 50_000
    cost_daily_usd: 5
    cost_monthly_usd: 50
    primary_model: haiku
    fallback_model: null

  standard:
    api_rpm: 500
    llm_rpm: 200
    llm_tpm: 300_000
    cost_daily_usd: 100
    cost_monthly_usd: 2500
    primary_model: sonnet
    fallback_model: haiku

  premium:
    api_rpm: 2000
    llm_rpm: 1000
    llm_tpm: 1_500_000
    cost_daily_usd: 500
    cost_monthly_usd: 12_000
    primary_model: opus
    fallback_model: sonnet

  enterprise:
    # negotiated per contract
    # generally: 5-10x premium budgets
    # dedicated provider account
    # dedicated vector index
```

### 8.3 The enforcement infrastructure

- **API rate limit:** AWS API Gateway with per-tenant API key + usage plan.
- **LLM rate limit (RPM and TPM):** Redis-backed token bucket; AI consumer acquires before each call.
- **GPU concurrency:** N/A; all tenants on hosted models (no self-hosted GPU).
- **Vector index:** Pinecone with per-tenant namespace and per-tenant API key for QPS isolation; enterprise tenant on dedicated index.
- **Downstream write:** per-tenant token bucket wrapping downstream API calls.
- **Cost:** per-tenant cost tracker with two-threshold breaker (warning at 80%, hard at 100%).

### 8.4 The three near-misses

**Incident 1: Standard tenant's misconfigured chatbot (March 2026).**

A standard tenant deployed a chatbot that incorrectly looped on user input parsing. The chatbot fired 12 RPM (within budget) but each request had 8 LLM calls (retrieval, planning, 3 tool calls, validation, response), so effective LLM-RPM was ~96 — under the 200 LLM-RPM budget but the calls averaged 4000 tokens each, producing ~380k TPM (over the 300k TPM budget).

The TPM bucket exhausted; the tenant's subsequent requests were 429'd at the consumer side. Other tenants unaffected. The tenant noticed within 15 minutes; called support; we identified the loop in their code; they fixed and redeployed within 2 hours.

**What worked.** Separate TPM and RPM budgets caught what RPM alone would have missed.

**What we improved.** Added per-tenant alert ("you're at 80% of your TPM budget") so the tenant could have seen the issue before hitting the cap.

**Incident 2: Free trial tenant's runaway agent (January 2026).**

A free trial tenant was experimenting with an agent that called itself recursively. The agent invoked LLM in a loop with no termination condition. Inside 20 minutes, the tenant generated $4.30 of LLM spend.

The cost circuit-breaker fired at the $5/day limit. Tenant's subsequent agent invocations were refused. Tenant was notified ("you've hit your daily cost limit"). Operator was paged. We investigated, found the recursive loop, contacted the tenant; they fixed.

Without the cost circuit-breaker, the tenant could have generated hundreds of dollars before anyone noticed.

**What worked.** Cost circuit-breaker as financial backstop.

**What we improved.** Added pre-flight cost estimation for agent invocations; the tenant's UI now shows "this agent task will cost approximately $0.85" before running.

**Incident 3: Premium tenant's bulk re-indexing (May 2026).**

A premium tenant ran a bulk re-indexing of their entire document corpus (~80k documents) during business hours. The re-indexing produced 80k embedding calls + 80k vector-index writes in ~2 hours.

The embedding calls were within the tenant's TPM budget (premium has 1.5M TPM). The vector writes were within the tenant's QPS budget. But the vector index's cache was warmed for the re-indexing tenant's new keys; other tenants' query cache-hit rate dropped from 75% to 40% during the re-indexing; their P99 retrieval latency climbed from 80ms to 220ms.

The cross-tenant correlation panel showed the impact. Operator was paged. We negotiated with the premium tenant to move the re-indexing to overnight. They agreed.

**What worked.** Detection caught what budgets didn't.

**What we improved.** Added "bulk operation off-peak preference" to the customer docs; added a feature flag for tenants to schedule re-indexing during low-traffic windows.

### 8.5 The architecture cost

- Redis cluster for budget tracking: ~$300/month.
- Pinecone per-tenant namespaces: included in standard Pinecone pricing.
- Per-tenant observability infrastructure (dashboards, alerts): ~$200/month (Datadog).
- Cost-tracking infrastructure: ~$100/month.
- Total noisy-neighbor mitigation cost: ~$600/month for the platform.
- One-time engineering investment: ~6 weeks (1 engineer) to build out the budget enforcement and observability layers.

### 8.6 The customer-facing dashboard

Each tenant has access to:

- Usage panel: RPM, TPM, cost (current, projected month).
- Latency panel: their own P50/P95/P99 over last 7 days.
- Budget panel: each budget's current consumption with warning thresholds.
- Incident history panel: operator-acknowledged events that affected this tenant (empty for most tenants most of the time).

Customer-facing dashboard reduced support tickets about "performance" by ~60%; customers self-serve the answer.

---

## 9. Anti-patterns

### 9.1 The single-dimension budget

**Pattern.** Per-tenant RPM only. Tenant respects RPM, exhausts TPM. Other tenants get 429s.

**Corrective.** Budgets across all dimensions in §2. RPM, TPM, GPU, vector QPS, downstream writes, cost.

### 9.2 The "we trust tenants" deferred enforcement

**Pattern.** Budgets exist in config but are not enforced. "Tenants are reasonable; we'll add enforcement when needed." First non-reasonable tenant produces an SLA incident.

**Corrective.** Enforce from day one. Soft warnings during ramp-up; hard enforcement in production.

### 9.3 The shared provider account at scale

**Pattern.** All tenants on one Anthropic account. Tenant A's burst exhausts the account; tenants B-Z all 429'd.

**Corrective.** Either per-tenant rate budgets on a shared account (with rigorous enforcement), or per-tier separate accounts, or dedicated accounts for enterprise tenants. The "all tenants on one account with no tenant-level budget" pattern is failure-prone.

### 9.4 The runaway agent with no cost cap

**Pattern.** Free-tier tenant deploys an agent that loops. $40k of spend in a day before anyone notices.

**Corrective.** Per-tenant cost circuit-breaker. Daily and monthly. Hard limit, not just warning.

### 9.5 The premium tier as a label

**Pattern.** Premium tenants are marketed as "priority" but land in the same queue as standard. No differentiation in latency or capacity.

**Corrective.** Premium tier must change runtime behavior measurably (separate model, separate capacity, weighted scheduling). Cross-link to backpressure-and-queueing §6.4.

### 9.6 The "we monitor traffic but not per-tenant" gap

**Pattern.** Global RPS and latency dashboards exist. Per-tenant breakdown doesn't. Cross-tenant impact is invisible; first signal is the customer complaint.

**Corrective.** Per-tenant traffic and latency metrics. Per-tenant dashboards. Cross-tenant correlation panel.

### 9.7 The cost circuit-breaker that's "global"

**Pattern.** Cost cap exists but is global (or per-day-across-tenants). One tenant's spend tripping the breaker stops the platform for everyone.

**Corrective.** Per-tenant cost cap. Per-tenant breaker state. Tenant A's breaker has no effect on tenant B.

### 9.8 The compliance attestation that "the architecture is isolated" without evidence

**Pattern.** Sales claims the platform is multi-tenant isolated. Audit asks for evidence. Architecture diagram says "per-tenant" but operational reality is "shared with hope."

**Corrective.** Each isolation claim is backed by an enforcement mechanism. Audit trail demonstrates per-tenant access. Annual attestation reviews controls actually operating, not just designed.

---

## 10. Findings (sprint-assignable)

**ARCH-NNM-001 (P0). No per-tenant cost cap.** Runaway agent on free tenant generates uncapped LLM spend. Implement per-tenant daily and monthly cost cap with two-threshold breaker; hard enforcement. Owner: AI platform + FinOps.

**ARCH-NNM-002 (P0). Per-tenant RPM only, no per-tenant TPM.** Long-context tenant exhausts account TPM; other tenants 429'd. Add per-tenant TPM budget enforced via Redis token bucket. Owner: AI platform.

**ARCH-NNM-003 (P0). All tenants share one provider account with no tenant-level rate enforcement at consumer.** First multi-tenant burst is an SLA incident. Implement per-tenant rate budgets at consumer level; sum of budgets ≤ account budget × safety factor. Owner: AI platform.

**ARCH-NNM-004 (P0). No detection of cross-tenant impact.** First signal of noisy neighbor is the affected tenant's complaint. Build per-tenant latency dashboard + cross-tenant correlation panel; alert on per-tenant degradation. Owner: SRE + AI platform.

**ARCH-NNM-005 (P1). No pre-flight cost estimation for high-cost workloads.** Tenant submits expensive agent task without knowing the cost; surprise bill. Add pre-flight estimation (input tokens × rate + estimated output tokens × rate); surface to tenant. Owner: AI platform + product.

**ARCH-NNM-006 (P1). No per-tenant vector-index isolation beyond namespacing.** Bulk re-indexing tenant displaces other tenants' cache. Add per-tenant QPS budget for vector store; document bulk operation off-peak preference; offer dedicated index for enterprise. Owner: AI platform.

**ARCH-NNM-007 (P1). No tenant-visible budget metadata.** Tenants can't self-serve; support burden. Add GET /account/quotas endpoint; tenant dashboard. Owner: API team + product.

**ARCH-NNM-008 (P1). Premium tier is a label, not a guarantee.** Marketing claims priority not backed by architecture. Implement per-tenant model-tier routing; verify premium uses separate capacity or higher-weight scheduling. Owner: AI platform + product.

**ARCH-NNM-009 (P1). No per-tenant cost log queryable by tenant.** First cost dispute is the first audit. Add per-call cost log queryable by tenant; reconciliation API; surface in tenant dashboard. Owner: API team + product.

**ARCH-NNM-010 (P1). No per-tenant alerts approaching budget limits.** Tenants surprised by 429s. Alert at 80% of any budget; notify tenant via configured channel (email, webhook). Owner: product + AI platform.

**ARCH-NNM-011 (P2). Downstream side-effect store contention not budgeted per-tenant.** Tenant's bulk write delays other tenants' writes. Add per-tenant write rate budget; enforce at consumer wrapping downstream APIs. Owner: AI platform.

**ARCH-NNM-012 (P2). Cost circuit-breaker kills in-flight work.** Tenant's expensive but legitimate agent terminates abruptly. Adopt graceful policy: in-flight completes; new submissions refused. Owner: AI platform.

**ARCH-NNM-013 (P2). No noisy-neighbor runbook.** First incident is operator improvisation. Document runbook: identify noisy tenant, mitigate, investigate, resolve, post-incident review. Owner: SRE.

**ARCH-NNM-014 (P2). No annual review of per-tenant budgets vs actual usage.** Budgets drift from reality; tenants either constrained or wasteful. Annual review; adjust per-tier defaults; negotiate per-tenant overrides. Owner: product + AI platform.

**ARCH-NNM-015 (P2). Free tier budgets allow expensive operations.** $5/day cap with agent loops can generate the full $5 in minutes. Restrict free tier to lower-cost operations (smaller model, no agents); document. Owner: product + AI platform.

**ARCH-NNM-016 (P3). No cross-tenant correlation in observability tooling.** Operator can't visually correlate tenant A spike with tenant B latency. Build correlation panel; document use in runbook. Owner: SRE.

**ARCH-NNM-017 (P3). No customer-facing incident notification for cross-tenant events.** Customers don't know when their experience was affected by another tenant. Add per-tenant incident history; notify on operator-acknowledged events. Owner: SRE + product.

**ARCH-NNM-018 (P3). Compliance attestation claims isolation without operational evidence.** Architecture diagram doesn't match operational reality. Each isolation claim backed by enforcement mechanism; audit trail demonstrates per-tenant access; annual control review. Owner: compliance + security + AI platform.

---

## 11. Adoption sequencing checklist

For a team adopting noisy-neighbor mitigation, in order:

- [ ] **Enumerate the contention dimensions (§2) for your system.** Six is typical; your system may have more.
- [ ] **Define per-tenant budgets across all dimensions.** Defaults per tier; per-tenant overrides where negotiated.
- [ ] **Implement per-tenant API rate limit at gateway.** AWS API Gateway / Kong / Envoy.
- [ ] **Implement per-tenant LLM rate limit (RPM and TPM) at consumer.** Redis-backed token bucket.
- [ ] **Implement per-tenant cost circuit-breaker (§5).** Daily and monthly; two thresholds (warning + hard).
- [ ] **Implement per-tenant vector-index throttling.** Either provider-side (Pinecone per-key) or application-side wrapper.
- [ ] **Implement per-tenant downstream write throttling.** Per-tenant token bucket on the downstream API client.
- [ ] **Implement per-tenant model-tier routing (§4).** Premium gets premium model; enforced.
- [ ] **Build per-tenant traffic, latency, and cost metrics (§6.1).** Per-tenant, per-minute granularity.
- [ ] **Build noisy-neighbor dashboard (§6.2).** Top tenants, per-tenant latency, cross-tenant correlation, budget consumption, cost.
- [ ] **Set up per-tenant alerts (§6.3).** Traffic spike, latency degradation, budget breach, cost spike.
- [ ] **Build tenant-facing dashboard (§7.2).** Usage, latency, cost, incident history.
- [ ] **Document customer SLA terms (§7.1).** What's contractually guaranteed; what's best-effort.
- [ ] **Document noisy-neighbor runbook (§6.4).** Detection → identification → mitigation → investigation → resolution.
- [ ] **Pre-production test:** simulate a noisy tenant (high RPM, high TPM, expensive agent, bulk indexing). Verify per-tenant isolation holds.
- [ ] **Annual budget review:** adjust per-tier defaults to reflect real usage.

---

## 12. References

**In this folder.**
- [isolation-models.md](./isolation-models.md) — the three isolation models; this doc applies to shared-model-isolated-data and shared-everything-with-filtering.
- [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) — vector-store-side per-tenant separation; composes with vector QPS isolation here.
- [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md) — security-focused multi-tenant isolation; complementary to performance-focused isolation here.

**Elsewhere in this repo.**
- [integration-architecture/backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md) — the queueing topology and fairness mechanics that underpin per-tenant enforcement (companion).
- [integration-architecture/event-driven-ai-integration.md](../integration-architecture/event-driven-ai-integration.md) — per-tenant fairness in event-driven AI consumers.
- [integration-architecture/integration-failure-patterns.md](../integration-architecture/integration-failure-patterns.md) — failure handling at integration boundaries, including 429 propagation.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — model-tier routing primitives that per-tenant tiering builds on.
- [reference-systems/patient-api-ai-assist.md](../reference-systems/patient-api-ai-assist.md) — the multi-tenant SaaS worked example.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) — engineering-side implementation of the cost circuit-breaker.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-tenant cost attribution mechanics.
- [ai-engineering-reference-architecture / cost-and-finops / tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — engineering-side tier routing.
- [ai-engineering-reference-architecture / observability-and-telemetry / cost-dashboards.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/cost-dashboards.md) — per-tenant cost dashboards.
- [ai-engineering-reference-architecture / observability-and-telemetry / alerting-and-paging-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/alerting-and-paging-design.md) — alert design including per-tenant fairness alerts.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — abuse and DoS scenarios that noisy-neighbor mitigation must defend against.

**External.**
- AWS API Gateway usage plans and per-key throttling docs.
- Pinecone API key throttling and per-namespace QPS docs.
- Anthropic / OpenAI rate-limit documentation (provider-specific account-level constraints).
- Stripe's metering API patterns (per-customer usage tracking) — strong reference for per-tenant cost tracking.
- SaaS Capacity Management literature (e.g., AWS Well-Architected SaaS Lens).
