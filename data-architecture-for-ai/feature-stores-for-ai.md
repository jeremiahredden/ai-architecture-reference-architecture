# Feature Stores for AI

> **Audience.** Architects considering a feature store for an AI workload. Teams with an existing ML feature store wondering how it fits with LLM-era systems. Anyone whose AI feature needs personalisation, user state, or structured context that supplements retrieval. **Scope.** The *architectural* decision — when a feature store earns its place, when alternatives (context caches, profile DBs) suffice, and the integration shape with agent or RAG pipelines. Not the engineering of feature pipelines (covered by feature-store vendor documentation). Not the broader ML feature engineering practice. **Worked client.** Meridian Health.

---

## 1. Why this document exists

The feature store concept comes from traditional ML: a centralised system that serves consistent features to both training (offline) and serving (online) pipelines, solving the train/serve skew problem. Examples include Tecton, Feast, Hopsworks, AWS SageMaker Feature Store, and Databricks Feature Store.

LLM-era systems have different needs. The "feature" concept is less central because:

- LLMs don't typically need engineered numerical features at inference time.
- The dominant context-injection pattern is retrieval (RAG), not feature lookup.
- Training-time vs serving-time skew is less of a concern because most teams don't train; they use frontier or fine-tuned base models with prompts and retrieval.

But feature stores haven't become irrelevant. The legitimate use cases in LLM systems:

- **Personalisation signals.** Per-user preferences, recent behaviours, demographic / segmentation attributes used to customise prompts or filter retrieval.
- **User state.** Current session context, in-flight task state, recent conversation summary that lives across conversations.
- **Structured context that supplements retrieval.** "Patient's current medications" as a structured fact set alongside retrieved free-text notes.
- **Cross-feature consistency.** When multiple AI features need the same per-user context, a feature store provides one source of truth.

The architectural decision is whether a dedicated feature store is the right pattern for these needs, or whether simpler alternatives (a profile DB, a cache, semantic memory in the agent layer) suffice. The wrong choice in either direction is costly: a feature store where it isn't needed adds operational complexity for marginal benefit; the absence of one where it is needed produces inconsistent per-user experiences and duplicated infrastructure.

This document is opinionated about four things:

1. **The default for new AI features is no feature store.** Direct retrieval + per-feature profile reads + agent memory cover most cases. The feature store is added when cross-feature consistency or scale demands it.
2. **Feature stores compete with semantic memory and profile DBs.** The choice depends on access patterns (per-feature vs cross-feature), update cadence (real-time vs batch), and the team's existing infrastructure.
3. **When adopted, the feature store is part of the AI architecture, not separate.** Integration patterns matter; ad-hoc bolting on produces inconsistent behaviour.
4. **Online/offline parity matters even when no training is involved.** The same feature served at inference time should match what eval consumes; mismatch breaks evaluation.

Structure: (2) what feature stores actually offer in LLM-era systems; (3) when feature stores fit; (4) feature store vs alternatives; (5) integration patterns with AI workloads; (6) online/offline parity; (7) operational and cost considerations; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. What feature stores actually offer in LLM-era systems

The capabilities, calibrated for LLM-era use cases.

### 2.1 The core capabilities

A feature store provides:

- **Feature definitions as code.** Features are defined declaratively; transformations are reusable.
- **Online serving.** Low-latency lookups (typically sub-10ms) for inference-time feature retrieval.
- **Offline storage.** Historical feature values for training, eval, and analysis.
- **Online/offline consistency.** The same feature computed at inference time matches the value seen at training/eval time.
- **Backfill.** Compute historical feature values from raw data.
- **Materialised features.** Pre-computed and stored; ready for serving.
- **On-demand features.** Computed at request time from raw inputs.
- **Versioning and lineage.** Feature definitions are versioned; lineage tracks data flow.

For LLM-era systems, the most relevant capabilities are online serving, online/offline consistency, and versioning. Backfill and materialisation matter for some use cases.

### 2.2 The features in LLM context

In an LLM workload, a "feature" might be:

- `user.last_5_search_queries`
- `patient.current_medications`
- `tenant.product_tier`
- `user.preferred_language`
- `session.conversation_summary`
- `account.subscription_status`

These are structured per-entity data that supplements the unstructured retrieval (RAG) or memory (per [memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) in the engineering sibling).

### 2.3 The "feature" vs "context" distinction

In traditional ML, a feature is a numerical input to a model. In LLM systems, the "feature" is structured context injected into a prompt or used to filter retrieval. The mechanics differ:

- Traditional ML: feature value → model's tensor input.
- LLM: feature value → rendered as text in the prompt OR used as a filter parameter for retrieval/tools.

The feature store's purpose (consistent, low-latency feature serving) is similar; the integration is different.

### 2.4 What feature stores don't typically offer

- **Unstructured content storage.** That's the corpus / vector store role.
- **Conversational memory.** That's the agent's semantic memory layer (per the engineering sibling).
- **Real-time streaming context updates.** Some feature stores support this; many don't.
- **Cross-feature aggregation logic for prompts.** The team builds this on top.

### 2.5 The "feature store vs vector store" misconception

These are not alternatives. They serve different purposes:

- Vector store: similarity search over unstructured content.
- Feature store: keyed lookup of structured per-entity attributes.

A typical LLM workload uses both: vector retrieval for content + feature lookup for per-user context.

### 2.6 The "feature store as silver bullet" pitfall

Vendor marketing sometimes positions feature stores as the answer to AI data architecture broadly. The pitch oversells; feature stores solve a specific problem and don't replace retrieval, memory, or structured databases.

Evaluate against specific use cases; not as a general-purpose platform.

---

## 3. When feature stores fit

The decision criteria for adopting a feature store in an AI workload.

### 3.1 The "yes" criteria

Adopt when:

- **Cross-feature consistency matters.** Multiple AI features need the same per-user context (preferences, segments, recent behaviour); a shared feature store prevents inconsistency.
- **Online/offline consistency matters.** The same feature value served at inference time must match the value at eval / training time (e.g., a fine-tuned model used a feature; that feature must be available consistently).
- **The team has scale that justifies the operational investment.** Tens of features × millions of entities × high serving QPS.
- **The team has existing feature store infrastructure.** If Tecton / Feast is already deployed for traditional ML, extending to AI workloads is incremental.
- **Personalisation is a major axis.** AI features customised heavily per user; the customisation reuses per-user signals.

### 3.2 The "no" criteria

Stay simple when:

- **Single feature, small scale.** One AI feature, narrow personalisation; a direct DB read is simpler.
- **No cross-feature reuse.** Per-feature profile data; no other consumer.
- **Real-time conversational memory only.** That's agent memory, not features.
- **The team lacks operational capacity for another platform.** Adding a feature store has overhead.

### 3.3 The diagnostic questions

Before adopting:

1. How many AI features will use this per-entity context? If one, alternatives suffice.
2. What's the read QPS? If < 100 QPS, alternatives suffice.
3. Is online/offline consistency a real requirement (e.g., for fine-tune training)? If not, alternatives suffice.
4. Does the team have feature-store operational experience? If not, the learning curve is real.
5. What's the maintenance cost? Feature stores require ongoing care.

Honest answers reduce feature-store adoption to the cases where it genuinely fits.

### 3.4 The "we'll need it eventually" hypothetical

Some teams adopt a feature store anticipating future needs. The pattern often backfires: the feature store is operationally costly upfront; the imagined future needs don't materialise as expected; the team carries the burden for years.

Better pattern: adopt simpler alternatives now; migrate to a feature store when needs are concrete.

### 3.5 The greenfield vs brownfield distinction

Greenfield (no existing ML platform): the bar for adopting a feature store is high; alternatives have to materially fall short.

Brownfield (existing ML platform with feature store): the bar is lower; extending the existing system is cheaper than building parallel infrastructure.

This document's framing assumes a greenfield decision is the harder choice.

### 3.6 The vendor landscape

In 2026:

- **Tecton.** Mature; managed offering; expensive at scale; strong cross-source feature pipelines.
- **Feast.** Open-source; lighter weight; community-supported.
- **Hopsworks.** Open-source + managed; broader ML platform.
- **Databricks Feature Store.** Bundled with Databricks; convenient for Databricks-centric teams.
- **AWS SageMaker Feature Store.** Bundled with SageMaker; AWS-centric.
- **In-house.** Some teams build their own; non-trivial investment.

The vendor choice is secondary to whether to use a feature store at all.

---

## 4. Feature store vs alternatives

The alternatives, evaluated.

### 4.1 Direct DB read

The simplest alternative. The AI feature reads from the application's database directly.

**Pros.**
- Simple; no new system.
- Always-fresh data.
- Existing DB operational expertise.

**Cons.**
- DB schema changes affect the AI feature; coordination required.
- No cross-feature consistency layer.
- DB load increases with AI feature usage.
- No built-in offline/online parity.

**When right.** Small-scale, single-feature use cases.

### 4.2 Profile / state DB (purpose-built)

A dedicated DB (Redis, DynamoDB, custom) holding per-entity profiles. Read by the AI feature.

**Pros.**
- Decoupled from application DB; AI feature load doesn't affect it.
- Fast reads (Redis sub-ms; DynamoDB single-digit ms).
- Custom schema for AI-feature needs.

**Cons.**
- Sync layer needed (data from source → profile DB).
- Eventual consistency to account for.
- Schema management is the team's job.

**When right.** Multiple AI features sharing per-entity context, but without the full feature-store operational investment.

### 4.3 Cache (Redis, Memcached)

A cache layer holds derived per-user context.

**Pros.**
- Fast reads.
- Implementation simple.

**Cons.**
- Cache invalidation complexity.
- Cold-start latency.
- Not a system of record.

**When right.** Augmenting a profile DB; not as the primary store.

### 4.4 Agent's semantic memory (per the engineering sibling)

The agent's own per-user memory layer (per [memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md)).

**Pros.**
- Lives within the agent's scope; integrates naturally.
- Conversational and behavioural context.
- Owned by the AI feature team.

**Cons.**
- Per-agent; doesn't share across features.
- Memory is specific to the agent's interactions, not a complete profile.

**When right.** Conversational context, agent-specific memory. Not a substitute for full per-entity profile.

### 4.5 The choice matrix

| Use case | Best fit |
| --- | --- |
| Single AI feature, small scale | Direct DB read |
| Multiple AI features, simple shared profile | Profile DB |
| Multiple AI features, complex features, scale | Feature store |
| Conversational state per-user | Agent semantic memory |
| Hot path with cache hit > 95% | Cache layer |

Most production architectures combine these (e.g., feature store for shared profile features + agent memory for conversational state + cache layer for hot reads).

### 4.6 The "feature store for everything" anti-pattern

Some teams try to use the feature store for all per-entity data including conversational state, ephemeral session data, and unstructured content.

The pattern over-extends the tool. Conversational state belongs in agent memory; ephemeral data in caches; unstructured content in vector stores. Feature stores are for structured per-entity features that benefit from feature-store capabilities.

### 4.7 The "we have a feature store from ML; let's use it" alignment

When a feature store already exists for traditional ML workloads, extending to AI workloads is often the right pattern — the operational investment is sunk; adding features is incremental.

The opposite is rare: starting an ML platform investment specifically for AI workloads usually doesn't justify a feature store; simpler alternatives suffice.

---

## 5. Integration patterns with AI workloads

How feature stores connect to RAG, agent, and workflow patterns.

### 5.1 Pattern A — Features as prompt context

Feature values are formatted as text and included in the prompt.

```
User query → 
  fetch per-user features (preferences, recent activity, segments) →
  format as text → 
  include in system prompt → 
  generation
```

Example:

```
User context:
- Preferred language: Spanish
- Recent searches: cardiology specialist, diabetes management
- Subscription tier: premium
- Account age: 18 months

[User's query follows]
```

**When right.** Personalisation that benefits from the model's reasoning over user context.

### 5.2 Pattern B — Features as retrieval filter

Feature values constrain the retrieval scope.

```
User query →
  fetch user features →
  use as retrieval filters (tenant, language, content tier) →
  vector retrieval with filters →
  generation
```

Example: a user's language preference filters retrieval to that language's content; the user's subscription tier filters to authorized content.

**When right.** Per-user filtering of large corpora; pre-retrieval narrowing.

### 5.3 Pattern C — Features as tool input

Features become input arguments to tools the agent calls.

```
Agent loop →
  agent decides to call fetch_recommended_content tool →
  tool fetches per-user features as context →
  retrieves recommendations based on features →
  returns to agent
```

**When right.** Agent-shaped workflows where the model's tool calls benefit from per-user context.

### 5.4 Pattern D — Features for re-ranking

Features inform post-retrieval reranking.

```
User query →
  retrieve top-N candidates →
  fetch per-user features →
  rerank using features (user's domain, recent interest, etc.) →
  return top-K →
  generation
```

**When right.** Reranking benefits from per-user signal; common in recommendation-shaped retrieval.

### 5.5 The latency budget

Feature lookup is part of the request latency. For interactive features:

- Feature store p99 lookup: typically 5-20ms.
- Per-feature lookup; multiple features per request may be batched.
- Total feature lookup overhead: 10-50ms for typical workloads.

Compared to embedding (~50-100ms) and LLM call (~500-2000ms), feature lookup is small. Acceptable for most workloads.

### 5.6 The "feature value changed mid-conversation" issue

In multi-turn conversations, a user's feature value may change between turns (user updates preference; behaviour changes their segment). The agent's context may be stale.

Mitigations:

- Re-fetch features per turn (cost).
- Cache for the conversation's duration (stale within conversation).
- Hybrid: stable features cached; volatile features re-fetched.

The choice depends on which features change and how much it matters.

### 5.7 The privacy and access boundary

Per-user features are PII. The feature store must:

- Enforce access controls per the calling context (user, tenant, feature).
- Audit access.
- Support deletion for right-to-be-forgotten.
- Per [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) considerations.

The feature store's role in privacy compliance is significant; engineer accordingly.

---

## 6. Online/offline parity

When features are used at training/eval time, parity matters.

### 6.1 The parity problem

Traditional ML: a feature computed differently at training vs serving (different SQL aggregation; different time window; bug in one path) produces a "train/serve skew" — the model performs differently in production than in eval.

In LLM systems, the equivalent: a feature used in eval (when running golden cases) is computed differently than at production serving. The eval doesn't predict production behaviour.

### 6.2 When parity matters in LLM workloads

- **Fine-tuning.** If a fine-tune was trained with feature X computed via method M, production must also use M.
- **Eval against golden cases.** The eval simulates production; features in eval should match production.
- **A/B testing.** The two variants should see consistent feature definitions.
- **Cross-team evaluations.** Different teams should see the same feature values.

### 6.3 When parity matters less

- **Single-call inference with simple personalisation.** A user preference fetched directly; parity isn't a concern because the same path serves training and production.
- **Stateless prompts.** No persistent per-user state.

### 6.4 The feature store's parity guarantee

Feature stores enforce parity via:

- One feature definition; one computation logic.
- Same code path serves online (low-latency lookup) and offline (historical query).
- Versioned features; changes are tracked.

This is the feature store's strongest argument. Teams with serious eval/production parity needs benefit.

### 6.5 The "we'll just be careful" alternative

Some teams maintain parity through discipline rather than infrastructure:

- Feature computation in shared code; both eval and production import.
- Tests verify both paths produce same output.
- Code review enforces.

The discipline approach works for small teams with few features; scales poorly.

### 6.6 The "we don't need parity" stance

Some workloads genuinely don't need parity. Stateless prompts; per-user simple lookups; one-off features.

For these, the feature store's parity capability is unused; the cost is paid for capability that isn't needed.

---

## 7. Operational and cost considerations

The infrastructure layer.

### 7.1 The operational footprint

A feature store typically requires:

- Online store infrastructure (Redis, DynamoDB, or similar; sub-10ms reads).
- Offline store infrastructure (data warehouse; historical features).
- Pipeline infrastructure (Spark, Flink, or vendor-specific; computes and materialises features).
- Feature definition repository.
- Monitoring and observability.

For a Tecton or Hopsworks managed deployment, much of this is provided. For self-hosted Feast or in-house, the team owns it.

### 7.2 The cost

- Managed services (Tecton, Hopsworks managed): $10k+/month for small deployments; scales with usage.
- Self-hosted (Feast on team's infrastructure): cost of underlying systems (online store cost + offline store cost + pipeline compute); typically $1k-10k/month for moderate scale.
- In-house: engineering time + infrastructure.

The cost is non-trivial; the workload must justify.

### 7.3 The engineering investment

For a new feature store adoption:

- 1-2 sprints for initial deployment.
- 2-4 sprints for feature pipeline development.
- Ongoing: feature owners, schema evolution, maintenance.

Total: 0.5-1 FTE ongoing for moderate-scale deployments.

### 7.4 The integration with the rest of the AI stack

The feature store integrates with:

- The AI feature's serving path (Pattern A/B/C/D).
- The eval infrastructure (online/offline parity).
- The agent / RAG pipeline (per the engineering sibling's depths).
- The cost attribution system (per [cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) in the engineering sibling).
- The observability stack.

Integration work is part of the adoption cost.

### 7.5 The "shadow IT" risk

Teams sometimes build mini-feature-stores per AI feature when the central feature store doesn't yet support their needs. The shadow stores diverge in capability, ownership, and quality.

Mitigations:

- Central feature store team's responsibility to support new use cases.
- Architecture review catches shadow infrastructure.
- Migration plan when shadow infrastructure is identified.

### 7.6 Disaster recovery

Feature stores hold serving-path-critical data. DR considerations:

- Online store: replicate; failover-ready.
- Offline store: standard data warehouse DR.
- Pipeline: idempotent; can re-run after failure.

Standard data-platform discipline.

---

## 8. Worked Meridian example

Meridian's feature-store decisions.

### 8.1 The starting point

Meridian had no traditional ML practice; no existing feature store. AI features ship via the platform team's prompt + RAG + agent infrastructure.

### 8.2 The decision evaluations

The team considered feature stores three times in 18 months:

**Q1-25.** Pitched for the care-coordinator's personalisation. Evaluation: only one AI feature; single source of profile data; direct DB read sufficed. Decision: no feature store.

**Q3-25.** Pitched for cross-feature personalisation as more AI features launched. Evaluation: 4 AI features now in production; some shared profile signals; but the shared signals were stable (patient demographics, tenant tier) and a profile DB satisfied. Decision: profile DB instead.

**Q4-25.** Pitched again as analytics-warehouse copilot added more dynamic personalisation needs. Evaluation: the dynamic features (recent queries, recent analytical sessions) would benefit from a feature store; but the volume and complexity didn't yet justify the operational investment. Decision: deferred; revisit in 6 months.

So far: no feature store. The team uses profile DB + agent memory + direct reads.

### 8.3 The profile DB architecture

Meridian's per-entity profiles live in:

- **PostgreSQL** for patient profiles (demographics, care preferences, current medications).
- **Redis** for session state (current conversation; recent interactions).
- **Agent semantic memory** for cross-conversation per-user context (per [memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) in the engineering sibling).

This combination serves the team's current needs. No feature store overhead.

### 8.4 The features actually used

| Feature | Source | Update cadence | Read pattern |
| --- | --- | --- | --- |
| `patient.demographics` | EHR sync to PostgreSQL | Daily | Per-conversation start |
| `patient.current_medications` | EHR sync to PostgreSQL | Real-time (CDC) | Per-care-coordinator-request |
| `patient.care_preferences` | Application writes to PostgreSQL | Real-time | Per-conversation start |
| `tenant.config` | Application writes to PostgreSQL | Real-time | Per-request |
| `user.recent_interactions` | Agent memory (semantic) | Per-interaction | Per-conversation start |
| `user.session_context` | Redis | Per-turn | Per-turn |

Each has its own store appropriate to its access pattern. No central feature store.

### 8.5 The Q1-26 revisit

The team revisited the feature-store question. Findings:

- Across 4 AI features, ~12 distinct per-entity features are used.
- About 8 of those are shared across 2+ features.
- The shared features have inconsistent schemas across features (some use snake_case, some camelCase; some include nulls, some don't); causing intermittent prompt-quality issues.
- Engineering time spent reconciling feature schemas across features: ~5% of one engineer's time per quarter.

The team is moving toward a profile-DB consolidation (centralised schemas; consistent reads) rather than a feature store. The consolidation captures most of the benefit at less operational cost.

### 8.6 The migration would be...

If Meridian decided to adopt a feature store:

- Likely choice: Feast (open-source; modest operational cost).
- Migration: 1 quarter to deploy + 2 quarters to migrate features.
- Expected benefit: cross-feature consistency formalised; backfill capability; better online/offline parity.
- Expected cost: ~0.5 FTE ongoing; infrastructure ~$2k/month.

The cost/benefit isn't compelling yet. Meridian continues with the profile-DB pattern.

### 8.7 What worked

- **Sceptical evaluation.** Three feature-store pitches; three "no" decisions. Each time, the alternative was sufficient.
- **Centralisation over standardisation.** Profile DB consolidation captures most of the benefit.
- **Per-store optimisation.** Different stores for different access patterns (PostgreSQL for stable; Redis for hot; agent memory for conversational).
- **Honest cost analysis.** The feature-store overhead is real; the team didn't romanticise the value.

### 8.8 What might change

The team would adopt a feature store if:

- Cross-feature inconsistency caused user-visible quality issues.
- Fine-tuning workloads required strict online/offline parity.
- The number of shared features grew to 20+.
- The team's scale outgrew the profile-DB approach.

None apply yet.

---

## 9. Anti-patterns

### 9.1 "Feature store as default for AI"

The team adopts a feature store for every AI feature regardless of fit. Operational complexity high; benefit marginal.

**Corrective.** Decision criteria per section 3; alternatives evaluated.

### 9.2 "Feature store as universal AI data layer"

The team uses the feature store for conversational memory, unstructured content, ephemeral state. Tool over-extended.

**Corrective.** Right tool per use case per section 4.5.

### 9.3 "Direct DB reads as 'good enough' forever"

The team avoids any infrastructure investment; AI features read application DB directly. Cross-feature inconsistency accumulates; DB load grows.

**Corrective.** Profile DB or feature store when cross-feature consistency matters per section 3.1.

### 9.4 "Shadow feature stores"

Each AI feature builds its own mini-feature-store; the team accumulates redundant infrastructure.

**Corrective.** Central ownership per section 7.5.

### 9.5 "No online/offline parity"

Features computed differently at eval vs production; eval doesn't predict production behaviour.

**Corrective.** Parity discipline per section 6; either feature store or rigorous shared code.

### 9.6 "Feature definitions in code without versioning"

Features evolve; old eval results don't reflect new feature definitions; comparisons are invalid.

**Corrective.** Versioned features per section 2.1; feature store enforces; alternative is rigorous version discipline.

### 9.7 "Stale features in long conversations"

User's feature value changed mid-conversation; agent's context is stale; user sees inconsistent treatment.

**Corrective.** Re-fetch strategy per section 5.6.

### 9.8 "Feature store cost not tracked"

Feature store is operational infrastructure; cost grows; nobody owns the budget.

**Corrective.** Cost ownership per section 7.2.

---

## 10. Findings (sprint-assignable)

### ARCH-FS-001 — Severity: High
**Finding.** Feature store adopted without per-workload justification; operational complexity exceeds benefit.
**Recommendation.** Decision review per section 3; if not justified, plan migration to alternatives.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-FS-002 — Severity: High
**Finding.** Cross-feature inconsistency in per-user data; same field returns different values to different AI features.
**Recommendation.** Profile DB centralisation per section 8.5, or feature store if scale warrants.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-FS-003 — Severity: High
**Finding.** Feature store used for conversational memory; overextended.
**Recommendation.** Move conversational state to agent memory per section 4.4 and [memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md).
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-FS-004 — Severity: High
**Finding.** Online/offline parity broken; eval results don't reflect production behaviour.
**Recommendation.** Parity discipline per section 6; shared computation paths.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-FS-005 — Severity: High
**Finding.** Shadow feature-store infrastructure built per AI feature; redundant.
**Recommendation.** Consolidate per section 7.5.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-FS-006 — Severity: Medium
**Finding.** No feature ownership; features evolve without coordination; downstream consumers break.
**Recommendation.** Per-feature owner; review cadence.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-FS-007 — Severity: Medium
**Finding.** Feature lookup adds significant latency; not budgeted for.
**Recommendation.** Latency budget per section 5.5; batching where applicable.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FS-008 — Severity: Medium
**Finding.** Stale features in long conversations; user-experience inconsistency.
**Recommendation.** Re-fetch strategy per section 5.6.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FS-009 — Severity: Medium
**Finding.** Feature schemas inconsistent across AI features (casing, nulls, types).
**Recommendation.** Schema discipline per [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md); standardise.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FS-010 — Severity: Medium
**Finding.** Feature store cost not tracked; budget unowned.
**Recommendation.** Cost ownership per section 7.2.
**Owner.** ai-platform-eng + finance, sprint N+3.

### ARCH-FS-011 — Severity: Medium
**Finding.** Privacy controls on feature store not enforced; PII accessible too broadly.
**Recommendation.** Access controls per section 5.7; per-feature and per-caller.
**Owner.** ai-platform-eng + security, sprint N+3.

### ARCH-FS-012 — Severity: Medium
**Finding.** Feature store DR not tested; production-critical data unprotected.
**Recommendation.** DR per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-FS-013 — Severity: Medium
**Finding.** Feature backfill capability unused; eval against historical feature values impossible.
**Recommendation.** Backfill per feature-store standard; eval against historical states.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-FS-014 — Severity: Medium
**Finding.** No periodic review of feature store ROI; investment may have outgrown justification or vice versa.
**Recommendation.** Annual review per section 3.4.
**Owner.** architecture, sprint N+4.

### ARCH-FS-015 — Severity: Low
**Finding.** Feature versioning informal; cross-version comparisons invalid.
**Recommendation.** Versioned features per section 2.1.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-FS-016 — Severity: Low
**Finding.** Feature documentation thin; new engineers don't know what's available.
**Recommendation.** Feature catalog; per-feature documentation.
**Owner.** ai-platform-eng + feature teams, sprint N+5.

### ARCH-FS-017 — Severity: Low
**Finding.** Cache layer absent in front of feature store; same features fetched repeatedly.
**Recommendation.** Cache where appropriate per section 4.3.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-FS-018 — Severity: Low
**Finding.** Feature store observability minimal; lookup latency, error rates not tracked.
**Recommendation.** Standard observability for the platform.
**Owner.** ai-platform-eng + ops, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team evaluating whether to adopt a feature store:

- [ ] **Sprint 0 — diagnostic.** Walk through section 3.3 questions.
- [ ] **Sprint 0 — alternatives.** Profile DB, agent memory, direct reads — evaluate against use case.
- [ ] **Sprint 0 — decision.** Adopt feature store, adopt alternative, defer.

If adopting:

- [ ] **Sprint 1 — vendor choice.** Tecton / Feast / Hopsworks / in-house. Match team's operational capacity.
- [ ] **Sprint 1 — deployment.** Initial deployment; basic infrastructure.
- [ ] **Sprint 2 — first features.** Migrate the highest-value features.
- [ ] **Sprint 2 — integration.** AI workloads consume; pattern A/B/C/D.
- [ ] **Sprint 3 — online/offline parity.** Set up if needed.
- [ ] **Sprint 3 — versioning and lineage.** Per section 2.1.
- [ ] **Sprint 4 — observability.** Per section 7.4.
- [ ] **Sprint 4 — DR.** Per section 7.6.
- [ ] **Ongoing — feature ownership.** Per-feature owner; schema discipline.
- [ ] **Annually — ROI review.** Cost vs benefit; consider retirement if no longer justified.

For a team with a feature store of unclear value:

- [ ] **Sprint 0 — audit.** What features exist; which are used; what's the cost; what's the benefit.
- [ ] **Sprint 1 — decision.** Continue, consolidate, retire.

For a team using alternatives (profile DB / direct reads):

- [ ] **Sprint 0 — gap analysis.** Where do the alternatives fall short?
- [ ] **Sprint 1 — fix gaps with alternatives first.** Centralisation, schema discipline.
- [ ] **Sprint 2 — re-evaluate.** Do the gaps remain? Does the feature store justify?

A team that completes the sequence has feature serving infrastructure matched to the workload's actual needs. A team that doesn't either over-invests or under-invests; both are costly.

---

## 12. References

- [vector-store-architecture.md](./vector-store-architecture.md) — vector retrieval; complementary to feature lookup.
- [embedding-strategy.md](./embedding-strategy.md) — embeddings for retrieval; feature store is for structured features.
- [knowledge-graph-augmentation.md](./knowledge-graph-augmentation.md) — KG as another data type alongside features.
- [hybrid-store-architecture.md](./hybrid-store-architecture.md) — hybrid retrieval; features as filters.
- [data-contracts-for-retrieval.md](./data-contracts-for-retrieval.md) — schema discipline applies to features too.
- [lineage-and-provenance.md](./lineage-and-provenance.md) — feature lineage as part of provenance.
- [freshness-architecture.md](./freshness-architecture.md) — feature freshness considerations.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant feature considerations.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) — semantic memory; the conversational alternative.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — feature store cost attribution.
- Tecton, Feast, Hopsworks, Databricks Feature Store, AWS SageMaker Feature Store — vendor landscape.
- "Feature Stores: A Foundation for Machine Learning" — broader literature on feature stores.
- "ML Feature Store Architecture" (industry pattern documentation across vendors).
