# Freshness Architecture

> **Audience.** Architects designing the data freshness layer for AI workloads — retrieval corpora, feature stores, knowledge graphs, agent memory. Tech leads whose AI feature returned an answer based on data that was correct last week but wrong today. Anyone whose answer to "is this data current?" is "I don't know." **Scope.** The *architectural* decisions: real-time vs near-real-time vs nightly vs on-demand ingestion; per-source freshness SLOs; cache invalidation patterns across the retrieval and memory stack. Not the engineering of the ingestion pipeline (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/ingestion-pipeline-engineering.md`). Not the broader data-platform freshness practice (which informs but is broader than AI-specific). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Freshness is the most under-designed dimension of AI data architecture. Teams pick a vector store, choose an embedding model, build an ingestion pipeline — and ship without a deliberate answer to "how fresh does each retrieval source need to be?" The default tends to be "ingest nightly" because that's what data engineering platforms default to.

The default is wrong for many use cases. A clinical care coordinator asking about a patient's current medications cannot use a 24-hour-old snapshot if the patient had a medication change this morning. An analytics copilot answering questions about Q4 sales cannot serve stale data when the books closed an hour ago. A customer-support agent referencing an order's status cannot return the shipping state from yesterday.

Conversely, the opposite default — "real-time everything" — is over-engineering. Many AI features serve stable content where staleness is invisible. A clinical-guideline retrieval over published research doesn't need real-time updates; the guidelines change monthly at most. A code-documentation copilot referencing API docs doesn't need real-time updates; the docs change with releases.

The architectural discipline is per-source freshness SLOs: each retrieval source has a stated freshness target; the ingestion architecture is sized to meet it; consumers can read the SLO and decide whether they can rely on the source for time-sensitive queries.

The discipline scales across the stack. Vector stores have freshness characteristics. Feature stores have freshness characteristics. Knowledge graphs have freshness characteristics. Agent memory has freshness characteristics. Caches across each layer add their own freshness considerations. The architecture explicitly models all of them.

This document is opinionated about four things:

1. **Freshness is a per-source decision, not platform-wide.** Different sources warrant different freshness; one-size-fits-all wastes resources or accepts incorrect results.
2. **Real-time is expensive; on-demand is slow; near-real-time is often right.** The four patterns (real-time, near-real-time, periodic, on-demand) have distinct cost / latency / complexity profiles. Pick the cheapest pattern that meets the SLO.
3. **Cache invalidation is the second-hardest problem in computer science; in AI systems, it has many layers.** Naïve cache invalidation produces stale answers; over-aggressive invalidation defeats caching's purpose.
4. **The freshness SLO is published; consumers depend on it.** Without published SLOs, consumers either over-trust (accepting stale data) or under-trust (avoiding the source unnecessarily).

Structure: (2) the four ingestion freshness patterns; (3) per-source freshness SLOs; (4) when stale is acceptable, when it's catastrophic; (5) cache invalidation across the stack; (6) the ingestion architecture per pattern; (7) operational and cost considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four ingestion freshness patterns

Each has its place; the choice depends on the source's SLO.

### 2.1 Real-time

Data flows from source to retrieval store as it changes. Typical latency: seconds.

```
Source update → CDC / change event → stream processor → 
  embedding pipeline → vector store
```

**Characteristics.**
- Lowest staleness window (seconds).
- Highest operational complexity (streaming infrastructure).
- Highest cost (per-event processing).
- Bounded throughput by stream processing capacity.

**When right.**
- Data that changes frequently and queries that need current state.
- Medication updates; order status; live dashboards.
- Compliance requires near-immediate reflection of source state.

### 2.2 Near-real-time

Data flows with a small delay (minutes). Typically micro-batched.

```
Source update → batched every N minutes → 
  ingestion pipeline → vector store
```

**Characteristics.**
- Staleness window typically 1-15 minutes.
- Lower operational complexity than real-time.
- Lower cost (batched processing).
- Good throughput.

**When right.**
- Data that changes frequently but where minute-level staleness is acceptable.
- Most operational AI features.
- Care coordination where minutes-level staleness on most data is acceptable.

### 2.3 Periodic (batch)

Data flows on a schedule (hourly, daily, weekly).

```
Schedule trigger → full or incremental extract → 
  ingestion pipeline → vector store
```

**Characteristics.**
- Staleness window matches the schedule.
- Simple operational pattern.
- Lowest cost.
- Predictable resource usage.

**When right.**
- Data that changes infrequently or where day-old data is fine.
- Clinical guidelines; published research; reference data.
- Most non-operational corpora.

### 2.4 On-demand

Data is fetched at query time from the source.

```
Query → fetch from source → use in response (not stored)
```

**Characteristics.**
- Always-current.
- Per-query cost and latency.
- No persistent storage of the fetched data.
- Source must support per-query fetches at scale.

**When right.**
- Highly dynamic data with low query volume.
- Data with strict freshness requirements.
- Sensitive data (where storing it raises privacy concerns).

### 2.5 The decision matrix

| Pattern | Staleness | Cost | Complexity | Best for |
| --- | --- | --- | --- | --- |
| Real-time | Seconds | High | High | Critical-fresh, high-stakes |
| Near-real-time | Minutes | Medium | Medium | Most operational |
| Periodic | Hours-days | Low | Low | Stable content |
| On-demand | Always current | Per-query | Low (mechanics) / High (rate limits) | Low-volume, must-be-current |

Match the pattern to the source's SLO; don't default to the team's favourite.

### 2.6 The hybrid: per-source different patterns

Most production architectures use multiple patterns:

- Clinical guidelines: periodic (weekly).
- Patient records: near-real-time (minutes).
- Live medication orders: real-time (seconds).
- Recent encounters: near-real-time.
- Patient demographics: periodic (daily).

The mix is the right architecture. Operational complexity is real but justified by per-source SLO adherence.

---

## 3. Per-source freshness SLOs

The discipline that makes freshness manageable.

### 3.1 What the SLO specifies

For each retrieval source:

- **Target staleness.** "Patient records are at most 5 minutes stale."
- **Measurement window.** "P99 staleness over a 24-hour rolling window."
- **Consequences of missing.** "If P99 exceeds 10 minutes, alert; if exceeds 30 minutes, page on-call."
- **Owner.** Who is responsible.
- **Consumers.** Who depends on this SLO.

The SLO is a published artefact. Consumers can read it and decide whether the source meets their needs.

### 3.2 The SLO derivation

The SLO derives from the consumers' actual needs:

- **The most demanding consumer sets the floor.** If one consumer needs 5-minute freshness and others tolerate hours, the SLO is 5 minutes (unless that consumer can be served differently).
- **The cost / SLO tradeoff is explicit.** Tightening from 1-hour to 1-minute is 10× the cost; the team decides.

### 3.3 The SLO measurement

The architecture must measure freshness. Implementations:

- **Per-document timestamps.** Source records when the document was last updated; vector store records when the document was last indexed; staleness = "now" - last-updated.
- **Heartbeat / sentinel records.** A synthetic record updated at a known cadence; ingestion lag = "now" - last seen.
- **Source-pulled monitoring.** A polling job samples source state and compares to indexed state.

Without measurement, the SLO is aspirational.

### 3.4 The SLO violation response

When the SLO is missed:

- Alert fires.
- On-call investigates: ingestion pipeline issue, source issue, capacity issue.
- Mitigation: scale up, fix the bug, route around.
- Post-incident: update the pipeline; update the SLO if the gap is structural.

The discipline is the same as any reliability SLO.

### 3.5 The SLO documentation

Published per source:

```yaml
source: patient-records
freshness_slo:
  target_staleness_p99: 5_minutes
  measurement_window: 24_hours_rolling
  alert_threshold: 10_minutes
  page_threshold: 30_minutes
owner: ai-platform-eng
consumers:
  - care-coordinator-agent
  - patient-summary-workflow
  - analytics-warehouse-copilot
```

Consumers reading the catalog see "patient-records is 5-minute fresh"; they design accordingly.

### 3.6 The SLO evolution

SLOs evolve:

- New consumer with tighter needs → tighten the SLO (and pay the cost).
- Consumer needs relax → loosen the SLO (and save cost).
- Source's structural latency improved → reflect in the SLO.
- The team learned the prior SLO was wrong → correct.

Annual review; ad-hoc updates as needed.

### 3.7 The "no SLO" failure

Without published SLOs:

- Consumers assume freshness they don't get.
- Sources don't know what's expected of them.
- Incidents reveal mismatched assumptions.
- Operational changes happen without considering consumer impact.

The discipline of writing SLOs forces clarity that's otherwise absent.

---

## 4. When stale is acceptable, when it's catastrophic

The product / business decision underlying the SLO.

### 4.1 Stale-is-fine cases

- **Reference content.** Clinical guidelines, published research, regulatory documents. Updates are infrequent; day-stale is fine.
- **Historical analysis.** Trends, aggregates over past periods. Today's data may not be in but the analysis is still valid.
- **Stable content.** Product catalogs, internal policies, configuration that changes monthly.
- **Background processing.** Batch jobs that operate on yesterday's data; design accommodates the delay.

### 4.2 Stale-is-bad cases

- **Operational decisions.** "Should this patient receive medication X?" — needs current state.
- **Financial decisions.** Account balance, transaction status, fraud signals.
- **Status / state queries.** Order status, shipment location, ticket assignment.
- **Live coordination.** Multiple parties acting on the same data must see consistent current state.

### 4.3 Stale-is-catastrophic cases

- **Safety-critical clinical decisions.** Medications, allergies, do-not-resuscitate orders.
- **Real-time compliance.** Sanctions lists, fraud rules, kill switches.
- **High-stakes time-sensitive.** Stock trades, security incident response.

For these, the architecture must guarantee freshness (or alternatively, must clearly indicate to consumers when freshness is uncertain so they take a different path).

### 4.4 The stale-data UI pattern

For data that's intermittently fresh, surface the freshness to the user:

- "This data is from 2 hours ago" — user knows.
- "Live data" badge when current.
- "Cached" indicator with refresh option.

The pattern transfers some of the freshness responsibility to the user. Useful when staleness is acceptable with awareness but unacceptable without.

### 4.5 The fallback-to-source pattern

For critical queries against potentially-stale data:

- Check the retrieval store's freshness for this record.
- If stale beyond threshold, fetch from source directly (on-demand pattern for this query).
- If source unreachable, indicate uncertainty.

The pattern adds complexity but provides a fallback for the cases where staleness would be catastrophic.

### 4.6 The "catastrophic and we didn't know" failure

The team designed for stale-is-fine; later, a use case emerged where staleness mattered; no SLO existed; users were affected before anyone noticed.

Mitigation: per-use-case freshness analysis as part of the design review. New AI features must specify their freshness needs; the data architecture must support them.

---

## 5. Cache invalidation across the stack

Multiple cache layers; each with its own invalidation challenges.

### 5.1 The cache layers in an AI system

- **Embedding cache.** Computed embeddings cached by input hash.
- **Vector store query cache.** Recent query results cached.
- **Tool result cache.** Tool calls cached by arguments.
- **Prompt cache.** Provider-side prompt caching (Anthropic, OpenAI).
- **Application response cache.** Whole-response caching at the application layer.
- **CDN / edge cache.** For features served at the edge.

Each layer's invalidation must be coordinated; un-coordinated invalidation produces stale results.

### 5.2 The "stale at one layer" failure

A document is updated. The vector store re-indexes (fresh). But the application response cache has a 1-hour TTL; a user gets the old answer for 59 more minutes.

The user-visible freshness is the *worst* freshness in the stack. Each layer's TTL contributes.

### 5.3 Invalidation strategies

**TTL-based.** Each cached entry has a time-to-live; expires after duration. Simple; coarse; doesn't react to actual updates.

**Event-based.** Source updates emit events; caches subscribe; entries invalidated on event. Precise; complex; requires event infrastructure.

**Versioned.** Cache keys include source version; new versions don't match old cache entries. Effectively per-entity TTL of "until version bumps."

**Hash-based.** Cache key is hash of inputs (including freshness-relevant inputs); content change → new hash → new entry.

**Manual.** Operators invalidate caches on demand. Last resort; error-prone.

The right strategy depends on the layer; TTL is the default; event-based is the right answer for layers where TTL would be too coarse.

### 5.4 The embedding cache

Cached by input hash. Invalidation:

- Input hash same → cached result returned.
- Input hash different → re-compute.
- If embedding model changes → all cached entries are wrong; either flush or include model version in key.

The model-version-in-key pattern is cleaner; the cache naturally invalidates when the model upgrades.

### 5.5 The vector store query cache

If the underlying index updates, cached query results may be stale. Options:

- TTL (cached for N seconds; accepts brief staleness post-update).
- Event-based (index update event invalidates query cache).
- No query cache (relying on the store's own caching).

Most teams skip the application-level query cache; the vector store's internal caching is usually sufficient.

### 5.6 The tool result cache

For idempotent read tools, caching is straightforward (TTL or hash-based). For tools whose results depend on time (e.g., "fetch_recent_orders"), the cache must include time-relevant keys (the time window the query covered).

### 5.7 The prompt cache (provider-side)

Anthropic / OpenAI prompt caching caches the prompt prefix. Invalidation: the cache has TTL (typically minutes); cache is keyed by prefix content; no manual invalidation.

Design implication: keep the prompt prefix stable for long-lived cache benefits. Frequent prefix changes defeat the cache.

### 5.8 The application response cache

Coarsest level. TTL-based usually. Decision: short TTL (low cache hit rate, fresher results) vs long TTL (high cache hit rate, staler results).

For AI features where staleness matters, the application response cache may be the wrong layer to cache at; lower-layer caches (embedding, tool) provide most of the benefit without the staleness risk.

### 5.9 The CDN / edge cache

If AI responses are served via CDN, the CDN's TTL is the user-visible staleness. Coordinate with the per-source SLO.

For personalised responses, CDN caching is typically not appropriate; the response varies per user.

### 5.10 The "we don't cache" alternative

For features where staleness is unacceptable, the simplest answer is: don't cache. Pay the cost; serve fresh.

The pattern is right for low-volume features. For high-volume features, the cost may force a different approach (smarter caching with finer invalidation).

---

## 6. The ingestion architecture per pattern

The infrastructure for each pattern.

### 6.1 Real-time ingestion

```
Source DB ←(CDC: Debezium / Maxwell)→ Kafka / Kinesis →
  stream processor (Flink / Spark Streaming) →
  embedding service →
  vector store (with streaming-friendly bulk-upsert)
```

**Considerations.**
- CDC connectors per source DB.
- Stream processing capacity sized for peak rate.
- Embedding service must handle streaming throughput.
- Vector store must support concurrent reads and writes.
- Failure handling: dead-letter queues, retry, reconciliation.

**Cost.** Stream processing + embedding API + storage. Higher than batch.

### 6.2 Near-real-time ingestion

```
Source DB → micro-batch extract (every N minutes) →
  embedding service → vector store
```

**Considerations.**
- Simpler than real-time; just a scheduled job that runs frequently.
- Batches reduce per-event overhead.
- Backpressure: if a batch falls behind, subsequent batches catch up.

**Cost.** Lower than real-time; higher than periodic.

### 6.3 Periodic ingestion

```
Source DB → scheduled extract (hourly / daily / weekly) →
  embedding service → vector store
```

**Considerations.**
- Standard ETL pattern.
- Full-refresh or incremental; design choice.
- Schedule alignment with source update windows.

**Cost.** Lowest of the patterns.

### 6.4 On-demand fetch

```
Query → application →
  fetch from source →
  embed (if needed) →
  return to model context
```

**Considerations.**
- Source must support per-query reads.
- Rate limiting at the source.
- Latency overhead (network + processing).
- Optional: cache the on-demand fetches for nearby queries.

**Cost.** Per-query.

### 6.5 The hybrid pipeline

A production system typically combines patterns:

```
Real-time for medications, orders, alerts
Near-real-time for patient encounters, recent notes
Periodic for clinical guidelines, reference data
On-demand for low-volume specialised queries
```

The pipeline infrastructure supports all four; routing rules per source.

### 6.6 The ingestion observability

Per source:

- Ingestion rate (events per second / batch).
- Ingestion latency (source-update-time to indexed-time).
- Error rate.
- Backlog.
- Freshness measurement (matches SLO definitions).

The observability supports SLO measurement and incident response.

### 6.7 The reconciliation pattern

For sources where the ingestion may miss events (stream processing failures, batch job errors), periodic reconciliation:

- Compare source state to indexed state.
- Identify deltas.
- Backfill the gaps.

Runs nightly or weekly. Catches the silent failures that ongoing monitoring may miss.

### 6.8 The schema evolution

When the source's schema changes, the ingestion must adapt:

- Per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md), schema changes go through coordination.
- The ingestion pipeline updates to handle the new schema.
- Backfill may be needed.

Schema evolution is a frequent source of ingestion lag (the pipeline is broken or slowed while updating).

---

## 7. Operational and cost considerations

The infrastructure layer.

### 7.1 The cost of freshness

Per source / pattern:

- Real-time: high (streaming infra + per-event processing).
- Near-real-time: medium.
- Periodic: low.
- On-demand: per-query.

Tightening freshness from 1-hour to 1-minute typically increases cost 5-10×. The team's SLO needs to justify.

### 7.2 The throughput considerations

- Streaming: bounded by stream processing + embedding throughput.
- Near-real-time: bounded by batch processing throughput.
- Periodic: time-bounded (must complete within the window).
- On-demand: bounded by source rate limits.

Throughput planning per source.

### 7.3 The embedding cost

For large corpora, embedding cost dominates. A 100M-document near-real-time pipeline with 80k updates/day at $0.13/M tokens × 500 tokens/update = ~$5/day; small. At higher volumes, scales.

For real-time pipelines with high event rates, embedding cost can be significant; provider rate limits may constrain.

### 7.4 The infrastructure cost

- Streaming infrastructure (Kafka cluster, Flink workers): $1k-10k/month depending on scale.
- Batch processing: usually scheduled jobs on shared infra; lower marginal cost.
- Vector store: per [vector-store-architecture.md](./vector-store-architecture.md) considerations.

### 7.5 The "freshness driving cost" conversation

When the team realises their freshness needs are pushing cost, the conversation:

- Can the SLO be relaxed? (Sometimes the original SLO was over-spec.)
- Can the source be partitioned (real-time for hot data; periodic for cold)?
- Are there optimisations (smarter chunking; less re-embedding)?

The architecture should make this conversation possible by exposing per-source cost.

### 7.6 The DR for ingestion

If the ingestion pipeline fails:

- Backlog grows; freshness violates SLO.
- Source data becomes inaccessible to AI features (or accessible but stale).
- User-visible degradation.

Mitigations:

- Multi-region pipeline.
- Reconciliation jobs that catch missed updates.
- Fallback to on-demand for critical queries when pipeline is down.

### 7.7 The "freshness during deploys"

Deployments may interrupt ingestion. The architecture should:

- Allow rolling deploys without losing events.
- Reconciliation post-deploy to catch any gaps.
- Communicate planned maintenance windows.

### 7.8 The "data deletion" interaction

Per [vector-store-architecture.md](./vector-store-architecture.md) and privacy considerations: when a record is deleted at the source, the retrieval store must also delete. The freshness architecture must include deletion events, not just updates.

Real-time / near-real-time pipelines often handle this naturally via CDC. Periodic pipelines may need explicit deletion handling.

---

## 8. Worked Meridian example

Meridian's freshness architecture.

### 8.1 The freshness landscape

| Source | Pattern | SLO P99 | Why |
| --- | --- | --- | --- |
| Patient medications | Real-time | 30 seconds | Care decisions can't use stale medication data |
| Patient allergies | Real-time | 30 seconds | Safety critical |
| Patient demographics | Near-real-time | 5 minutes | Stable mostly; minutes acceptable |
| Patient encounters | Near-real-time | 5 minutes | Operational; minutes acceptable |
| Patient lab results | Real-time | 1 minute | Clinical decisions; high stakes |
| Clinical guidelines | Periodic (weekly) | 7 days | Updated infrequently |
| Payer policies | Periodic (daily) | 1 day | Updated at most daily |
| Internal procedures | Periodic (daily) | 1 day | Same |
| Live medication orders | Real-time | 30 seconds | Order coordination |
| API documentation | Periodic (per release) | per release | Updated on developer-API releases |
| Code samples | Periodic (per release) | per release | Same |

The mix reflects the per-source needs. Real-time for safety-critical and operational; periodic for reference.

### 8.2 The infrastructure

- **Real-time pipeline:** Debezium CDC from PostgreSQL → Kafka → Flink → embedding service → OpenSearch. Latency P99: 25 seconds.
- **Near-real-time pipeline:** Scheduled job every 2 minutes; batched extract → embedding → OpenSearch. Latency P99: 4 minutes.
- **Periodic pipeline:** Standard ETL (Airflow); nightly for most periodic sources; weekly for clinical guidelines.
- **On-demand:** A few specialised queries fetch from source live (e.g., live medication order status); not the common pattern.

### 8.3 The cost breakdown

- Real-time pipeline: ~$3,000/month (Kafka cluster + Flink + always-on embedding).
- Near-real-time pipeline: ~$500/month.
- Periodic pipeline: ~$200/month (incremental cost on existing data platform).
- Total ingestion infrastructure: ~$3,700/month.
- Plus embedding cost: ~$300/month.

### 8.4 The SLO measurement

Per source, a freshness panel in Grafana shows:

- Current staleness (now - max(last_indexed_time)).
- P99 staleness over the past 24 hours.
- SLO threshold.
- Alert status.

The team's on-call dashboard prominently displays the SLO-violation count.

### 8.5 The notable freshness incidents

**Q1-25.** Real-time pipeline for medications fell behind during a database migration; staleness grew to 6 hours. The pipeline's catch-up was rate-limited; took 12 hours to catch up.

Impact: care-coordinator served stale medication data for the day; on-call detected within an hour; clinical staff notified to verify medications directly during the lag.

Corrective: pipeline catch-up rate increased; alerting threshold tightened (from 5 min to 2 min); migration runbook now includes "pause/resume pipeline" instructions.

**Q3-25.** Periodic ingestion of clinical guidelines was failing silently for 3 weeks. A schema change at the source had broken the parser; the pipeline error wasn't alerting.

Impact: clinical-guideline retrieval was 3 weeks stale; one care-coordinator query referenced a deprecated guideline.

Corrective: pipeline error alerting added; SLO measurement now includes "ingestion failure" detection; weekly review of pipeline health.

**Q4-25.** Patient demographics SLO violation; the source's update batch had failed; demographics were 24 hours stale.

Impact: minimal (demographics rarely affect query quality); but the incident surfaced that demographic-update reliability hadn't been monitored.

Corrective: per-source ingestion-health monitoring; not just freshness but also success rate.

### 8.6 The cache architecture

- Embedding cache: keyed by (input_hash, embedding_model_version). Hit rate ~60% for query embeddings; reduces embedding cost.
- Prompt cache (Anthropic): automatic for the stable system prompt portion. Hit rate ~80%; saves ~70% of input token cost on those calls.
- No application-response cache (staleness risk for clinical content).
- CDN: only for static documentation pages.

The cache strategy is conservative; clinical content's staleness risk doesn't allow aggressive caching.

### 8.7 What worked

- **Per-source SLOs.** Clear targets; clear measurement; clear alerting.
- **Pattern variety.** Different patterns for different sources; not one-size.
- **Cache discipline.** Aggressive at the embedding layer (cost saver); conservative at the application layer (avoid staleness).
- **Reconciliation.** Periodic checks catch silent failures.

### 8.8 What didn't work initially

- **Default to "ingest nightly."** Initial deployment used nightly for everything; the medication freshness incident in Q1-25 was the wake-up call.
- **No SLO documentation.** Early on, the team's freshness expectations were implicit; consumers and producers had different assumptions. SLO documents resolved.
- **No pipeline-failure alerting.** The Q3-25 silent-failure incident drove the broader alerting on pipeline health.
- **Cost surprise on real-time.** When real-time pipelines launched, cost was 3× projection. The team optimised (better batching within the streaming framework) to bring cost down.

---

## 9. Anti-patterns

### 9.1 "Default to nightly for everything"

The team uses periodic (nightly) ingestion across all sources without per-source consideration; operational features become unusable for time-sensitive queries.

**Corrective.** Per-source pattern choice per section 2; SLO-driven.

### 9.2 "Real-time for everything"

The team uses real-time ingestion across all sources without consideration; cost explodes; complexity grows.

**Corrective.** Match pattern to SLO; real-time only when justified.

### 9.3 "No SLOs"

Freshness expectations are implicit; consumers and producers misaligned; incidents reveal mismatches.

**Corrective.** SLOs per section 3; published; per-source.

### 9.4 "Cache invalidation by hope"

Caches set with long TTLs; assumption that "users will get fresh data eventually"; staleness compounds.

**Corrective.** Invalidation strategy per layer per section 5; consider event-based for layers where TTL is too coarse.

### 9.5 "Silent ingestion failures"

The ingestion pipeline fails; nobody notices for weeks; freshness violates SLO without anyone realising.

**Corrective.** Per-source ingestion-health monitoring per section 8.5; alerting on failure not just on staleness.

### 9.6 "On-demand for high-volume queries"

The team uses on-demand fetches without considering rate limits and latency; source is overwhelmed; queries slow down.

**Corrective.** Match pattern to volume per section 2.5.

### 9.7 "No deletion handling"

The ingestion pipeline propagates updates but not deletions; deleted records remain in the retrieval store.

**Corrective.** Deletion events handled per section 7.8.

### 9.8 "Freshness ignored in schema changes"

Schema evolution breaks ingestion silently; freshness violates without correlation to the schema change.

**Corrective.** Schema discipline per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md); pipeline-failure alerts.

---

## 10. Findings (sprint-assignable)

### ARCH-FRESH-001 — Severity: Critical
**Finding.** Freshness-critical content (medications, allergies, orders) served via nightly ingestion; staleness causes user-visible incidents.
**Recommendation.** Real-time or near-real-time per section 2; SLO per section 3.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-FRESH-002 — Severity: Critical
**Finding.** No published SLOs per source; consumers and producers misaligned.
**Recommendation.** SLO documents per section 3.5; published catalog.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-FRESH-003 — Severity: Critical
**Finding.** Silent ingestion failures; pipeline broken for weeks without detection.
**Recommendation.** Per-source pipeline-health alerting per section 6.6 / 8.5.
**Owner.** ai-platform-eng + ops, sprint N+1.

### ARCH-FRESH-004 — Severity: Critical
**Finding.** Deleted records not removed from retrieval store; deleted PHI / PII persists.
**Recommendation.** Deletion handling per section 7.8.
**Owner.** ai-platform-eng + privacy, sprint N+1.

### ARCH-FRESH-005 — Severity: High
**Finding.** Cache invalidation by TTL only; staleness compounds across layers.
**Recommendation.** Per-layer invalidation strategy per section 5; event-based where TTL too coarse.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-FRESH-006 — Severity: High
**Finding.** Real-time pattern used for sources where minutes-stale is acceptable; cost 5-10× justified.
**Recommendation.** Pattern matching per section 2.5; relax to near-real-time where SLO allows.
**Owner.** architecture + finance, sprint N+2.

### ARCH-FRESH-007 — Severity: High
**Finding.** No freshness measurement; SLO is aspirational, not measured.
**Recommendation.** Measurement per section 3.3.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-FRESH-008 — Severity: High
**Finding.** Reconciliation absent; missed events accumulate without detection.
**Recommendation.** Periodic reconciliation per section 6.7.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-FRESH-009 — Severity: Medium
**Finding.** Application response cache with long TTL on dynamic content; users see stale responses.
**Recommendation.** Shorter TTL or no cache at application layer per section 5.8.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FRESH-010 — Severity: Medium
**Finding.** Per-source ingestion cost not tracked; freshness-cost tradeoffs invisible.
**Recommendation.** Cost per source per section 7.5.
**Owner.** ai-platform-eng + finance, sprint N+3.

### ARCH-FRESH-011 — Severity: Medium
**Finding.** Schema evolution breaks ingestion; freshness violations correlated to upstream changes.
**Recommendation.** Schema discipline per section 6.8 and [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md).
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FRESH-012 — Severity: Medium
**Finding.** No fallback-to-source pattern; when retrieval is stale, no recovery for critical queries.
**Recommendation.** Fallback per section 4.5 for catastrophic-stale cases.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FRESH-013 — Severity: Medium
**Finding.** No surfacing of freshness in UI; users can't tell if data is current.
**Recommendation.** Stale-data UI pattern per section 4.4 where appropriate.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-FRESH-014 — Severity: Medium
**Finding.** Embedding cache doesn't include model version in key; model upgrades return stale cached embeddings.
**Recommendation.** Versioned cache key per section 5.4.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-FRESH-015 — Severity: Low
**Finding.** Deploy procedure doesn't pause/resume ingestion; freshness violations during deploys.
**Recommendation.** Per section 7.7; deploy runbook.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-FRESH-016 — Severity: Low
**Finding.** SLO documents not reviewed annually; some SLOs may be wrong for current needs.
**Recommendation.** Annual review per section 3.6.
**Owner.** architecture, sprint N+4.

### ARCH-FRESH-017 — Severity: Low
**Finding.** Prompt cache (provider-side) defeat by unstable prompt prefix; cache hit rate low.
**Recommendation.** Stable prefix discipline per section 5.7.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-FRESH-018 — Severity: Low
**Finding.** Multi-region failover for ingestion not implemented; single-region failure means freshness violation across all sources.
**Recommendation.** Multi-region per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building freshness discipline:

- [ ] **Sprint 0 — source inventory.** What sources feed AI retrieval? What's the current freshness?
- [ ] **Sprint 0 — consumer needs.** Per AI feature, what freshness is needed?
- [ ] **Sprint 0 — SLO drafts.** Per source, propose SLOs from consumer needs.
- [ ] **Sprint 1 — measurement.** Per section 3.3; freshness measurable.
- [ ] **Sprint 1 — alerting.** SLO-violation alerts.
- [ ] **Sprint 2 — pattern reassignment.** Move sources to the right pattern per section 2.
- [ ] **Sprint 2 — pipeline-health alerting.** Per source.
- [ ] **Sprint 3 — reconciliation.** Periodic checks for missed events.
- [ ] **Sprint 3 — deletion handling.** Per section 7.8.
- [ ] **Sprint 4 — cache invalidation review.** Per layer per section 5.
- [ ] **Sprint 4 — UI surfacing.** Where appropriate per section 4.4.
- [ ] **Ongoing — quarterly review.** SLOs, costs, incidents.

For a team with existing freshness issues:

- [ ] **Sprint 0 — audit.** What's the current freshness per source vs the need.
- [ ] **Sprint 1 — fix worst gaps.** Often the silent-failure or no-deletion gaps.
- [ ] **Sprint 2 — SLO documentation.** Catch up.
- [ ] **Sprint 3 — pattern reassignment.** As needed.
- [ ] **Sprint 4 — cache review.** End-to-end.

A team that completes the sequence has freshness as a designed dimension. A team that doesn't has freshness as a recurring source of surprises.

---

## 12. References

- [vector-store-architecture.md](./vector-store-architecture.md) — vector store's role in freshness; deletion considerations.
- [embedding-strategy.md](./embedding-strategy.md) — embedding model version affects cache.
- [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — schema discipline that supports stable ingestion.
- [knowledge-graph-augmentation.md](./knowledge-graph-augmentation.md) — KG freshness considerations.
- [hybrid-store-architecture.md](./hybrid-store-architecture.md) — hybrid retrieval's freshness across signals.
- [lineage-and-provenance.md](./lineage-and-provenance.md) — freshness is part of lineage.
- [feature-stores-for-ai.md](./feature-stores-for-ai.md) — feature store freshness as case.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant freshness considerations.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/ingestion-pipeline-engineering.md](https://github.com/jeremiahredden/reference-architecture/blob/main/rag-engineering/ingestion-pipeline-engineering.md) — engineering depth on ingestion.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/retrieval-observability.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-observability.md) — observability for retrieval freshness.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops) *(planned)* — caching patterns broader context.
- Debezium, Maxwell, Kafka Connect — CDC tools for real-time pipelines.
- Flink, Spark Streaming — stream processing frameworks.
- Anthropic prompt caching, OpenAI prompt caching — provider-side caching.
- "Designing Data-Intensive Applications" (Kleppmann) — broader reference on data freshness and consistency.
