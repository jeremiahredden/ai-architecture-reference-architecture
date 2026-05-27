# Caching Tiers

> **Audience.** Architects designing the AI feature's cache layer. Tech leads deciding where caches sit in the stack and which tiers to deploy. Anyone whose "we should cache more" is becoming a strategic priority but the tier landscape is unclear. **Scope.** The *architectural* decisions: which of the four caching tiers to deploy per workload; where the caches sit in the stack; cross-cache coherency; freshness-vs-cost trade-offs; cache placement (per-tenant, per-region, shared); the cache topology. Not the engineering mechanics of each tier (see sibling [cost-and-finops/caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md)). Not the per-call cost analysis (see [token-economics.md](./token-economics.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The engineering-side document covers how each cache works. This document covers the architectural decisions about which caches to deploy, where to place them, and how they interact.

The architectural questions:

- Which caching tiers fit this workload?
- Where do caches sit in the stack (per-tenant, per-region, shared)?
- How do multiple caches coexist (coherency)?
- What's the freshness-vs-cost trade-off for this workload?
- What's the cache invalidation strategy at the architectural level?

Without these decisions, caching becomes either under-deployed (cost left on table) or chaotic (caches contradict each other; stale content; complexity).

This document is opinionated about four things:

1. **All four caching tiers can compose.** They serve different purposes and operate at different layers. Use what fits.
2. **Cache placement is an architectural decision.** Per-tenant isolation, per-region pinning, shared caches — each has trade-offs.
3. **Cache invalidation is architectural; don't defer to engineering.** Design the invalidation strategy at architecture time.
4. **Freshness-vs-cost is per-workload.** Some workloads can serve hour-stale results; some need sub-second freshness.

Structure: (2) the four tiers recap; (3) tier deployment decision; (4) cache placement in the stack; (5) cross-cache coherency; (6) freshness-vs-cost design; (7) architectural cache invalidation; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four tiers recap

Brief recap; engineering details in sibling doc.

### 2.1 Prompt-prefix cache (provider-side)

Provider caches the static prefix of prompts. Hits get a discount (e.g., 90%).

- Architectural placement: at the LLM API boundary.
- Configuration: cache-control markers in the prompt.
- Cross-tenant: per-tenant isolation by API key.

### 2.2 Response cache (exact-match)

Same input → same response, served from cache.

- Architectural placement: in front of LLM call.
- Cache key: hash of (model, prompt, parameters, tenant).
- Storage: in-memory + Redis typical.

### 2.3 Semantic cache (similarity)

Embedding-similarity match.

- Architectural placement: in front of LLM call; orthogonal to response cache.
- Storage: vector index + response store.
- Risk: wrong-cache-hit if threshold too low.

### 2.4 Retrieval cache

Cache the retrieved chunks for a query.

- Architectural placement: in front of vector store.
- Cache key: query embedding hash + tenant + parameters.

### 2.5 The tier interaction

For a complete AI request:

```
User query →
  Semantic cache hit? → Return cached response (no LLM, no retrieval).
  No → Retrieval cache hit? → Use cached retrieval.
  No → Vector store query.
  → Assemble prompt (with prompt-prefix caching at API boundary).
  → LLM call.
  → Response cache (for future identical queries).
```

Layered; each catches different cases.

---

## 3. Tier deployment decision

Which tiers fit which workloads.

### 3.1 The deployment matrix

```
Workload                       Prompt-prefix   Response   Semantic   Retrieval
─────────────────────────────────────────────────────────────────────────────
Care Coordinator agent         ✓ (stable prefix)  ×        ×          ✓ (per-session)
Patient API chat               ✓              ✓          ✓          ✓
FAQ chat                       ✓              ✓          ✓✓         ✓
Document classification        ✓              ✓✓ (immutable docs)  ×  ×
Real-time clinical decision    ✓              ×          ×          ✓
Embedding generation           ×              ✓✓         ×          ×
```

Per-workload selection.

### 3.2 The deployment criteria

**Prompt-prefix cache:** deploy if prompts have stable prefixes (>500 tokens). Almost always applicable; near-free.

**Response cache:** deploy if same inputs are common (FAQ-shaped, classification with repeat docs).

**Semantic cache:** deploy if workload is FAQ-shaped with paraphrasing.

**Retrieval cache:** deploy if retrieval is repeated within sessions or across users.

### 3.3 The "we deploy all four" maximalism

For some workloads, all four apply. Combined hit rates can reach 70%+.

For others, only one or two apply.

Per-workload audit.

### 3.4 The "should we deploy this tier" cost-benefit

Per tier:

```
deployment_cost = infrastructure + engineering setup + ongoing maintenance
benefit = (hit_rate × per_call_cost) - (overhead per call)
```

Deploy when benefit > cost.

### 3.5 The "we have no idea our hit rate" reality

Without monitoring:

- Cache deployed; unknown if working.

Cross-link to [caching-for-cost.md §6](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).

### 3.6 The "every tier has overhead" reminder

Each cache adds:

- Implementation complexity.
- Operational overhead.
- Storage cost.
- Invalidation complexity.

Justify per tier; don't deploy reflexively.

### 3.7 The per-feature differentiation

Different features may use different tier combinations:

- High-volume FAQ: all 4 tiers.
- Medium-volume agent: prompt-prefix + retrieval.
- Low-volume clinical: prompt-prefix only.

Per-feature.

---

## 4. Cache placement in the stack

Where caches physically sit.

### 4.1 The per-tenant cache

For multi-tenant: per-tenant cache instance.

- Tenant A's cache; tenant B's cache.
- Cross-tenant isolation by design.
- More cache instances; more operations.

Cross-link to [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md §7.2](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md).

### 4.2 The shared cache with tenant-keyed entries

Single cache instance:

- Keys include tenant_id.
- One physical cache; logically per-tenant.
- Lower operational overhead; risk of key-collision bugs.

### 4.3 The per-region cache

For multi-region deployment:

- Each region has its own cache.
- Cross-region cache access avoided.
- Per-region hit rates.

Cross-link to [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md).

### 4.4 The "shared globally" cache anti-pattern

A single cache across regions:

- Cross-region network latency.
- Residency violations possible.
- Hit rate doesn't compensate.

**Corrective.** Per-region.

### 4.5 The L1 + L2 tiered cache

Layered:

- L1: in-process memory (fastest; smallest).
- L2: Redis (fast; shared across processes).
- L3 (optional): persistent storage (slower; larger).

Read flow:
- Check L1 first.
- If miss, check L2; populate L1.
- If miss, full call; populate L2 and L1.

### 4.6 The cache-aside vs read-through

**Cache-aside:** application checks cache; on miss, calls LLM; populates cache.

**Read-through:** cache layer intercepts; transparent.

Cache-aside more common; explicit; flexible.

### 4.7 The cache for streaming responses

For streaming workloads:

- Response cache: cache the final assembled response.
- Replay: stream from cache (server emits tokens at expected rate).

Implementation complexity; not all platforms.

### 4.8 The cache size budget per location

```
Per-tenant L1: 100MB.
Per-tenant L2: 5GB.
Per-region L2: 100GB total.
```

Bounded; eviction policy per location.

### 4.9 The shared-vs-isolated trade-off

Shared cache:
- Lower operational cost.
- Higher hit rate (cross-tenant FAQ shared).
- Risk: cross-tenant leakage if not carefully keyed.

Isolated per-tenant:
- Higher operational cost.
- Lower hit rate (no cross-tenant share).
- Stronger isolation.

For PHI-bearing data: isolated. For non-sensitive FAQ: shared with tenant keys.

---

## 5. Cross-cache coherency

When multiple caches must agree.

### 5.1 The coherency problem

If response cache says "answer is X" but retrieval cache says "documents are Y" — what if X doesn't reflect Y?

Multi-cache systems need coherency strategy.

### 5.2 The "each cache independent" approach

Simple:

- Each cache has its own TTL.
- They don't coordinate.
- Stale: each may be stale on its own.

Easiest; some staleness.

### 5.3 The "linked invalidation" approach

When one cache invalidates, related caches do too:

- Document update → retrieval cache invalidates for that document.
- Retrieval cache invalidation → response cache invalidates entries that used that retrieval.

Linked; consistent; complex.

### 5.4 The "version tags" approach

Each cache entry has a version:

- Knowledge base version.
- Prompt version.
- Model version.

Entries with stale versions are invalidated.

Cross-link to [prompt-as-api-discipline.md §3](../context-and-prompt-architecture/prompt-as-api-discipline.md).

### 5.5 The freshness hierarchy

Some caches are inherently fresher:

- Prompt-prefix (5-60 min TTL): freshest.
- Retrieval cache (10 min - hours): medium.
- Semantic cache (hours - days): older.
- Response cache (hours - days): older.

Design with hierarchy in mind.

### 5.6 The "we made the data fresh but the response is stale" failure

User updates document. Retrieval re-indexes. Next call retrieves the updated content. But response cache has the old response.

Mitigation:

- Cascade invalidation.
- Or: shorter response cache TTL.

### 5.7 The cache-aware response generation

For some workloads:

- Response references retrieval IDs.
- If retrieval invalidates, response cache verifies.
- Cache hit returns only if retrievals are still valid.

Complexity; selective.

### 5.8 The "we accept some inconsistency" pattern

For non-critical workloads:

- Eventual consistency.
- Stale responses for the cache TTL.
- Acceptable to user (they don't notice immediately).

### 5.9 The strict-consistency case

For some workloads (clinical, financial):

- Strict consistency.
- Invalidate aggressively.
- Lower hit rate; higher correctness.

---

## 6. Freshness-vs-cost design

The trade-off.

### 6.1 The freshness requirements per workload

```
Workload                  Acceptable staleness
─────────────────────────────────────────────────────
FAQ chat                  Hours to days (FAQs don't change).
Patient API chat          Minutes (clinical context).
Care Coordinator          Real-time (patient's current state).
Document classification   Immutable (per document).
News / current events     Seconds.
Internal copilot          Minutes.
```

Per-workload; informs TTL.

### 6.2 The TTL per cache per workload

```
Care Coordinator:
  Prompt-prefix: 5 min default
  Retrieval: 5 min (clinical context)
  No response cache (real-time required)

Patient API chat:
  Prompt-prefix: 60 min (extended TTL)
  Retrieval: 30 min
  Response: 30 min (FAQ portion)
  Semantic: 24 hours

Document classification:
  Response: 7 days (documents are immutable)
```

Per-workload TTLs.

### 6.3 The "stale data is fine" UX

For some workloads:

- User sees "as of X minutes ago" indicator.
- Acceptable stalleness.

### 6.4 The "real-time required" workload

For clinical, financial, safety-critical:

- Aggressive invalidation.
- Sub-minute TTL or no cache.
- Cost vs safety trade-off.

### 6.5 The cost of strict freshness

Strict freshness = low cache hit rate = higher cost.

For real-time workloads: cost premium accepted.

### 6.6 The cache as eventual-consistency

By design, cache lags:

- Update event → invalidation event → all caches catch up.
- Latency: 100ms to seconds.
- Acceptable for most workloads.

### 6.7 The cache-staleness alert

Track cache age:

- Time since update.
- Time since cache fresh.

If exceeds expectation: alert.

### 6.8 The "we want sub-second freshness; cache won't work" reality

For some workloads: caching doesn't fit.

Accept; don't force-fit.

---

## 7. Architectural cache invalidation

How invalidation is designed.

### 7.1 The invalidation triggers

What invalidates cache:

- Time (TTL).
- Document update (retrieval cache).
- Prompt change (response cache).
- Model version change (response cache + prompt-prefix cache).
- Tenant configuration change (overlay change → response cache).
- Per-request override (force-fresh flag).

### 7.2 The invalidation event design

Cache invalidation is an event:

```
Event: document_updated
  document_id: D123
  tenant_id: T1
  
Subscribers:
  - Retrieval cache (invalidates entries for D123)
  - Response cache (invalidates entries derived from D123)
```

Pub-sub for invalidation.

### 7.3 The invalidation cascade

Linked caches cascade:

- Document update → retrieval cache invalidation → response cache invalidation.

Cross-link to §5.3.

### 7.4 The invalidation latency

Time from update to all caches invalidated:

- Local (in-process): instant.
- Redis: <100ms.
- Cross-region: 1-10 seconds.

Workload tolerates within latency.

### 7.5 The "we forgot to invalidate" failure

Update happens; cache not invalidated:

- Stale content served.
- May be hours / days before noticed.

Mitigation:

- All update events propagate to cache layer.
- Periodic full-cache validation (sample-check freshness).

### 7.6 The mass-invalidation pattern

When prompts version-up (cross-link to [prompt-as-api-discipline.md](../context-and-prompt-architecture/prompt-as-api-discipline.md)):

- All cache entries for that prompt version invalidated.
- Cache empties; fills with new version's responses.

Brief cost-spike; expected.

### 7.7 The "we keep stale on purpose" exception

For some non-critical workloads:

- Stale cache acceptable.
- Periodic refresh outside critical path.

Cache lag explicit.

### 7.8 The "manual invalidation" operator action

Operators can:

- Force-invalidate per pattern.
- Clear entire cache.

For incident response.

### 7.9 The audit log of invalidations

Track:

- When; what; why.
- Sample: "Cache for D123 invalidated at 14:23; reason: D123 updated."

Forensic / debugging.

---

## 8. Worked Meridian example

Meridian's cache architecture.

### 8.1 The architecture map

```
Patient API chat:
  Prompt-prefix: 60 min TTL (Sonnet provider-side)
  Response cache: 30 min TTL; tenant-keyed; Redis L2
  Semantic cache: 24 hour TTL; per-tenant index; threshold 0.92
  Retrieval cache: 30 min TTL; per-tenant; sub-keyed by query

Care Coordinator:
  Prompt-prefix: 5 min TTL
  Retrieval cache: 5 min TTL; per-session
  No response cache (real-time required)
  No semantic cache (clinical specifics; risk of wrong-hit)

Document classification:
  Prompt-prefix: 1 hour TTL (Llama 3 70B)
  Response cache: 7 days (documents immutable)
  No semantic cache (each document unique)
  No retrieval cache (no retrieval for classification)

Embedding (BGE-large self-hosted):
  Response cache: 30 days (embeddings stable for content)
```

Per-workload tier combinations and TTLs.

### 8.2 The deployment topology

```
Cache layer:
  L1 (in-process): 50MB per service instance; LRU.
  L2 (Redis cluster):
    Per-tenant size limits.
    Per-region (US, CA).
    ~30GB total.
  Persistent (PostgreSQL):
    For response cache (long-term FAQ).
    ~10GB per region.
```

Multi-tier; per-region.

### 8.3 The cross-cache coherency design

Document update workflow:

```
EHR → Document update event →
  Vector store re-index →
    Emit: document_updated event
      Subscribers:
        Retrieval cache: invalidate entries for document
        Response cache (per affected query patterns): invalidate
```

Cascade invalidation.

### 8.4 The Q1 2026 freshness incident

A clinician updated a patient's record. Care Coordinator's retrieval cache hadn't yet picked up.

For 4 minutes: Care Coordinator's retrieval returned old document.

Mitigation:

- 5-minute TTL on Care Coordinator retrieval cache.
- Real-time invalidation event on update.

After fix: invalidation event integrated; TTL stays as safety net.

### 8.5 The hit rates achieved

```
Patient API chat:
  Prompt-prefix: 55% hit rate
  Response cache: 18% hit rate
  Semantic cache: 22% hit rate
  Retrieval cache: 35% hit rate
  Combined: ~70% requests serve at least partially from cache.

Care Coordinator:
  Prompt-prefix: 60% hit rate
  Retrieval cache: 28% hit rate
  No response or semantic.
  Combined: ~85% of input tokens cached.

Document classification:
  Response cache: 22% (documents that are reclassified after updates)
```

Substantial cost savings.

### 8.6 The cost impact

Pre-cache: $90k/month total LLM cost.
Post-cache: $48k/month.
Savings: $42k/month (47%).

(Cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md §9](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).)

### 8.7 The infrastructure cost

- Redis cluster (cache layer): $1500/month.
- Vector index for semantic cache: $400/month.
- PostgreSQL for response cache: $200/month.
- Engineering: ~6 weeks initial setup (4 tiers + per-feature design).

Total: ~$2100/month + initial. Net savings: $40k/month.

### 8.8 The semantic-cache threshold tuning incident (Q2 2026)

(Cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md §9.8](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).)

Threshold lowered from 0.92 to 0.88 in experiment:

- Wrong-hit rate jumped from <1% to 8%.
- Detected within 4 hours by quality drift detection.
- Threshold restored; affected entries purged.

Lesson: threshold changes require eval gate.

### 8.9 The lessons

- Multi-tier caching delivers substantial savings.
- Per-workload tier selection matters.
- Coherency design is essential.
- Threshold tuning carefully.

---

## 9. Anti-patterns

### 9.1 The "we'll cache everything" zeal

**Pattern.** Deploy all 4 tiers without per-workload analysis. Operational overhead exceeds benefit.

**Corrective.** Per-workload per §3.

### 9.2 The "single cache; no isolation" multi-tenant leak

**Pattern.** Cache without tenant_id in key. Cross-tenant leakage.

**Corrective.** Per-tenant scoping per §4.1.

### 9.3 The "we deployed cache but no monitoring" gap

**Pattern.** Cache running; unknown hit rate. Cost-saving unverified.

**Corrective.** Per-cache hit-rate observability per §3.5.

### 9.4 The "stale data is fine" silent failure

**Pattern.** Cache stale; users notice; complaint surfaces. Mitigation requires aggressive invalidation.

**Corrective.** Per-workload freshness requirement per §6.1.

### 9.5 The "we forgot invalidation" cascading staleness

**Pattern.** Update event; cache not invalidated; stale until TTL.

**Corrective.** Pub-sub invalidation per §7.2.

### 9.6 The wrong-cache-hit (semantic)

**Pattern.** Semantic threshold too low; semantically-similar-but-different queries hit; wrong response.

**Corrective.** Threshold tuning per [caching-for-cost.md §4.2](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).

### 9.7 The cross-region cache without residency consideration

**Pattern.** Cache shared across regions; data crosses borders.

**Corrective.** Per-region cache per §4.3.

### 9.8 The "cache for streaming" complexity rush

**Pattern.** Try to cache streaming responses; complexity exceeds benefit.

**Corrective.** Cache the assembled response per §4.7; replay if needed.

### 9.9 The cache-coherency-deferred

**Pattern.** Multiple caches; coherency strategy "for v2." Stale combinations.

**Corrective.** Linked invalidation per §5.3.

### 9.10 The "cache size unbounded" growth

**Pattern.** No size limits; cache grows; storage cost balloons.

**Corrective.** Per-location budget per §4.8.

---

## 10. Findings (sprint-assignable)

**ARCH-CT-001 (P0). Caching not deployed despite high-volume cacheable workload.** Cost left on table. Per-workload deployment per §3. Owner: AI platform.

**ARCH-CT-002 (P0). Multi-tenant cache without tenant scoping.** Leakage risk. Per §4.1, §4.2. Owner: AI platform + security.

**ARCH-CT-003 (P0). Cache invalidation events not designed.** Stale content; cascading failures. Per §7.2. Owner: AI platform.

**ARCH-CT-004 (P0). Cross-region cache without residency awareness.** Data crosses borders. Per §4.3. Owner: AI platform + security.

**ARCH-CT-005 (P1). Per-cache hit-rate monitoring absent.** Cache effectiveness unknown. Per §3.5. Owner: observability-eng.

**ARCH-CT-006 (P1). Cross-cache coherency design absent.** Stale combinations. Per §5. Owner: AI platform.

**ARCH-CT-007 (P1). Per-workload TTL not defined.** Either too aggressive or too lax. Per §6.2. Owner: AI platform.

**ARCH-CT-008 (P1). Per-tenant cache size limits not set.** One tenant fills shared cache. Per §4.8. Owner: AI platform.

**ARCH-CT-009 (P1). Semantic cache threshold not tuned to workload.** Wrong-hit rate. Per §3.5 and [caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md). Owner: AI platform.

**ARCH-CT-010 (P2). L1/L2 tiered cache not deployed.** Sub-optimal performance. Per §4.5. Owner: AI platform.

**ARCH-CT-011 (P2). Cache-aside vs read-through decision ad-hoc.** Inconsistent integration. Per §4.6. Owner: AI platform.

**ARCH-CT-012 (P2). Cache-staleness alert absent.** Drift undetected. Per §6.7. Owner: observability + AI platform.

**ARCH-CT-013 (P2). Manual invalidation API absent for incident response.** Operators have no lever. Per §7.8. Owner: AI platform + SRE.

**ARCH-CT-014 (P2). Invalidation audit log absent.** Forensic / debugging gap. Per §7.9. Owner: AI platform.

**ARCH-CT-015 (P3). Streaming cache pattern not explored.** Some workloads miss caching benefit. Per §4.7. Owner: AI platform.

**ARCH-CT-016 (P3). Cache infrastructure cost not surfaced.** Hidden in operations budget. Per §8.7. Owner: FinOps.

**ARCH-CT-017 (P3). Annual cache-tier-review absent.** Tiers may be over- or under-deployed. Owner: AI platform.

**ARCH-CT-018 (P3). Cross-tier cache size accounting absent.** Hard to budget. Per §4.8. Owner: AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Per-workload tier deployment decision (§3).**
- [ ] **Cache placement design (per-tenant, per-region) (§4).**
- [ ] **Per-tenant scoping (§4.1).**
- [ ] **Cross-cache coherency (§5).**
- [ ] **Per-workload TTL (§6.2).**
- [ ] **Invalidation event design (§7.2).**
- [ ] **Per-cache hit-rate monitoring (§3.5).**
- [ ] **Per-cache size budget (§4.8).**
- [ ] **Eviction policy per cache.**
- [ ] **Manual-invalidation API (§7.8).**
- [ ] **Cache staleness alerts (§6.7).**
- [ ] **Annual cache-tier review.**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — caching as cost lever.
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — cache hit reduces latency.
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — caching reduces provider load.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) *(coming)* — caching composes with routing.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) — self-host capacity considerations.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md) — per-tenant scoping.
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — per-region.
- [context-and-prompt-architecture/prompt-assembly-patterns.md](../context-and-prompt-architecture/prompt-assembly-patterns.md) — cache invalidation on prompt change.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md) — engineering mechanics.
- [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — retrieval cache.

**External.**
- Provider prompt-caching documentation (Anthropic, OpenAI).
- Redis documentation.
- Cache-invalidation literature.
