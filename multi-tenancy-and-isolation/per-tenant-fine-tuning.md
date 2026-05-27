# Per-Tenant Fine-Tuning

> **Audience.** Architects of multi-tenant AI platforms whose customers are asking "can we fine-tune the model on our data?" Tech leads weighing the operational cost of per-tenant model variants against the alleged benefit. Anyone whose roadmap shows "per-tenant fine-tuning" as a Q3 deliverable and is not yet sure whether that's the right answer. **Scope.** The *architectural* decisions: the small number of cases where per-tenant fine-tuning is genuinely right; the alternatives that usually win (strong RAG + per-tenant overlay; adapters and LoRAs as a middle ground); the per-tenant economics; the operational cost (per-tenant model lifecycle, eval, rollback, deprecation); the adapter architecture if you go that route; per-tenant migration when you have N models instead of one. Not the fine-tuning *engineering* itself (see sibling [model-lifecycle / fine-tuning-operations](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle), planned). Not the build-vs-buy decision (see [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md), companion). Not the frontier-vs-open-weights choice (see [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](../model-strategy/frontier-vs-open-weights-vs-fine-tune.md), foundational). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Per-tenant fine-tuning is one of the most-requested and least-justified features in multi-tenant AI platforms. Customers ask for it because the phrase "fine-tuned on our data" is intuitive and sounds powerful. Sales teams add it to roadmaps because it differentiates from competitors that don't offer it. Engineering teams sometimes implement it because the request count is loud.

What customers usually want when they ask for "fine-tuning on our data":

- The model understands our domain (medical, legal, financial, retail).
- The model knows our specific terminology, products, and procedures.
- The model responds in our style and voice.
- The model doesn't reference competitors or irrelevant content.
- The model gives accurate, current information about our business.
- Our data is private and not used to train models for other customers.

Of these six, exactly zero require per-tenant fine-tuning in 2026. All of them are addressable with:

- A capable frontier model.
- Strong tenant-aware retrieval over the tenant's content.
- A per-tenant prompt overlay (see [per-tenant-prompt-and-context.md](./per-tenant-prompt-and-context.md)).
- Per-tenant data isolation (see [isolation-models.md](./isolation-models.md), [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md)).
- Per-tenant tool access (see [per-tenant-prompt-and-context.md §5](./per-tenant-prompt-and-context.md#5-per-tenant-tool-authorization-at-the-assembly-layer)).

The architectural failure mode: building per-tenant fine-tuning infrastructure because customers asked for it, then discovering that the fine-tuned model performs no better than the RAG-augmented frontier model — at 10-100x the operational cost.

That doesn't mean per-tenant fine-tuning is never right. It does mean the bar should be high. The right framing is "we will fine-tune for tenant X only when we can articulate, with evidence, why RAG + overlay isn't enough." The wrong framing is "tenant X asked, so we'll fine-tune."

This document covers the architectural decisions: when per-tenant fine-tuning is genuinely the right answer; what RAG-based alternatives look like and why they usually win; the per-tenant economics; the operational cost the team will inherit; the adapter / LoRA architecture if you commit; the migration mechanics when you have N tenant models instead of one.

This document is opinionated about four things:

1. **The default answer is no.** Per-tenant fine-tuning has substantial fixed operational cost (per-tenant lifecycle, eval, rollback, deprecation). The cost is rarely worth it. RAG + per-tenant overlay handles ~95% of "we want a model that knows our data" requests at a fraction of the cost.
2. **Adapters / LoRAs are not "fine-tuning lite"; they are a different commitment.** They reduce the per-tenant inference cost dramatically (adapters can be loaded on demand on a shared base model). They do not reduce the per-tenant data, eval, or lifecycle cost — those are still there. Adapters change the inference economics; they don't change the operational economics.
3. **When per-tenant fine-tuning is genuinely right, it's because the workload has properties RAG cannot deliver.** Specific signal: the tenant's domain has terminology, syntax, or output style that prompt-context cannot reliably elicit. Most "tenant-specific knowledge" needs are not in this category.
4. **The eval suite scales with the model count, not the tenant count.** N tenant models = N eval suites, each maintained, each run on every base-model change, each updated when the tenant's domain evolves. This is the dominant hidden cost; teams that miss it discover it 18 months in.

Structure: (2) the cases where per-tenant fine-tuning is genuinely right; (3) the alternatives that usually win; (4) the per-tenant economics; (5) the operational cost; (6) the adapter / LoRA architecture; (7) per-tenant model migration; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The cases where per-tenant fine-tuning is genuinely right

The valid cases share specific properties. If your tenant's request doesn't match one of these, the default answer is no.

### 2.1 Domain-specific syntax or output structure that prompt can't reliably elicit

Some domains have output forms that are hard to coax from a frontier model via prompting alone — even with detailed instructions and few-shot examples.

**Examples.**

- A legal document drafting workflow where the output must conform to a specific firm's brief structure, with citations in a specific format, and clause numbering in a specific scheme. Few-shot examples in the prompt help; reliability is still ~80%; fine-tuning gets it to ~98%.
- A clinical coding workflow where the output is a specific ICD-10/CPT code list with a specific confidence ranking. Prompt-engineered to ~85% accuracy; fine-tuned to ~95%.
- A code-generation workflow for a specific in-house framework with un-documented conventions. Prompt-with-examples reaches ~70%; fine-tune reaches ~90%.

**Test for this category.** Does the workload's required output have specific structural rules that the frontier model gets wrong frequently even with extensive prompt engineering? If yes, fine-tuning may help. If no (the model gets the output structure right when prompted well), fine-tuning won't fix the actual gap.

### 2.2 Capability augmentation that exceeds the frontier model's capability

Rare; the frontier models in 2026 are very capable. But there are workloads where domain expertise compresses the input enough to make a smaller model competitive.

**Example.** A medical coding workflow where the input is a 50-page clinical note and the output is a structured code list. A frontier model with full prompt handles it well but costs $0.40 per case. A fine-tuned smaller model (Llama 3 8B fine-tuned on millions of (note, codes) pairs) reaches comparable accuracy at $0.04 per case — 10x cost reduction.

**Test for this category.** Is there a fine-tunable smaller model whose post-fine-tune accuracy matches or exceeds the frontier model's accuracy on this workload? If yes, fine-tuning is a cost play. If no, fine-tuning is an exercise in lower-accuracy-at-lower-cost.

### 2.3 Strong regulatory requirement for "model trained on customer's data"

Some regulated industries have customers whose compliance teams insist on a model that was trained on the customer's data, regardless of the technical merit. The argument is regulatory or contractual, not technical.

**Test for this category.** Is the requirement contractual? Is the customer's compliance posture genuinely changed by per-tenant fine-tuning (e.g., they can claim "the model is trained on our data" in their own SOC 2 / HIPAA assessment)? If yes, fine-tuning may be required regardless of technical merit; the architecture must support it. If no (the compliance team is wrong about the technical merit), educate the customer rather than build infrastructure.

### 2.4 Latency requirement that only a smaller model can meet

The smaller model is faster (lower latency per token, higher tokens-per-second). Fine-tuning a smaller model to be domain-competent makes it a viable latency-optimized choice.

**Example.** Voice assistant for a specific domain (medical scribe, legal note-taker) where round-trip latency must be < 500ms. Frontier model is too slow; fine-tuned small model is fast enough.

**Test for this category.** Does the workload have a strict latency budget that no frontier model meets, even with streaming? If yes, fine-tuning a smaller model is a viable path. If no, fine-tuning isn't the latency lever.

### 2.5 Volume that makes per-call cost the dominant cost

For very high-volume workloads (millions of calls per day), per-call cost dominates. A fine-tuned smaller model at $0.001/call beats a frontier model at $0.040/call by 40x on cost. The fine-tuning cost is amortized quickly.

**Test for this category.** Is the workload high-volume enough that per-call cost × volume dominates total cost? If yes, fine-tuning is a cost play. If no, the per-call savings don't justify the infrastructure cost.

### 2.6 The "none of the above" case

If the tenant's request for fine-tuning doesn't match §2.1-§2.5, the request is probably for tenant-specific knowledge, terminology, or style — which RAG + overlay handles. The architectural answer is "no, but here's how we'll address what you actually need."

This conversation, conducted well, is education and trust-building. Conducted poorly, it sounds dismissive and the tenant churns. The framing matters: "we don't offer per-tenant fine-tuning because the architecture we use delivers what you need without it" is different from "we don't offer that feature."

---

## 3. The alternatives that usually win

The default architecture for "tenant wants the model to understand their domain" is RAG + per-tenant overlay. Properly built, it delivers most of what fine-tuning promises at much lower cost and operational overhead.

### 3.1 Strong tenant-scoped RAG

The first lever. The tenant's documents — manuals, procedures, terminology, historical examples — are ingested into a tenant-scoped retrieval index. The model is prompted to use the retrieved context to answer.

**What this addresses.**

- Domain terminology (the retrieved chunks include the tenant's terminology in context).
- Specific facts about the tenant's business (the retrieved chunks have the facts).
- Current information (re-indexing keeps it current).
- Tenant data privacy (retrieval is tenant-scoped; cross-link to [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md)).

**What it doesn't address.**

- Output structure rules that prompt + examples can't reliably elicit.
- Latency requirements that the frontier model can't meet.
- Per-call cost for very high-volume workloads.

For workloads with property "wants to know our specific business," RAG addresses 80-90% of the gap. The remaining 10-20% can usually be closed with §3.2.

### 3.2 Per-tenant prompt overlay

A small, curated prompt addendum for each tenant. Covers terminology preferences, persona, output style, tenant-specific rules. Cross-link to [per-tenant-prompt-and-context.md §3](./per-tenant-prompt-and-context.md#3-the-layered-prompt-assembly-architecture).

**What this addresses.**

- Tenant voice and terminology.
- Tenant-specific compliance constraints.
- Persona ("respond as a Meridian assistant").
- Output format preferences.

**What it doesn't address.**

- Knowledge that's not in the overlay or retrieved (overlay must stay small).
- Output structure with strict rules (rules can be in the overlay; reliability isn't 100%).

Combined with RAG, the overlay handles "model knows our voice and our specific business." Most tenant requests for "fine-tuning" reduce to RAG + overlay.

### 3.3 Few-shot examples in the prompt

For workloads with specific output structure, in-context examples often suffice. 3-5 examples of "input → expected output" in the prompt elicits the structure.

**What this addresses.**

- Structured output where 3-5 examples cover the pattern space.
- Specific style requirements (tone, terminology, structure).

**What it doesn't address.**

- Workloads with vast structural variability (no 5 examples cover the space).
- Workloads where the structure is genuinely complex (e.g., needing to learn rules, not just patterns).

For ~70% of "model needs to produce a specific format" workloads, few-shot examples + structured-output enforcement (function calling, JSON schema) reaches sufficient reliability.

### 3.4 Structured output enforcement

The model is told to output JSON matching a schema; the structured-output API enforces. Cross-link to [reference-patterns/structured-output-patterns.md](../reference-patterns/structured-output-patterns.md).

**What this addresses.**

- Output that must be parseable.
- Output with specific field requirements.
- Reduction in malformed output errors.

**What it doesn't address.**

- The content quality of the output (structured can still be wrong).
- Very complex output schemas (deeply nested, with conditional fields).

Combined with few-shot examples, structured output handles most "specific format" needs.

### 3.5 Cascading: RAG + overlay + few-shot + structured output

The combined stack:

```
Tenant request
    ↓
RAG (tenant-scoped retrieval pulls relevant context)
    ↓
Prompt assembly:
  - Shared base (model role, safety)
  - Tenant overlay (terminology, persona, rules)
  - Retrieved context (tenant's documents)
  - Few-shot examples (5 input→output for this workload)
  - Current user request
    ↓
Model call with structured-output enforcement
    ↓
Output validation (cross-link to per-tenant-prompt-and-context.md §6)
    ↓
Response to user
```

For workloads where RAG + overlay + few-shot + structured output is insufficient, fine-tuning is on the table. The fine-tuning decision starts after this stack is exhausted, not before.

### 3.6 When the alternatives don't suffice

The honest cases (matching §2.1-§2.5) remain valid:

- Strict output-structure requirements that examples can't elicit reliably.
- Latency requirements only a smaller model can meet.
- Volume where per-call cost dominates.
- Contractual / regulatory requirement for fine-tune.

For these, fine-tuning is on the table. The architectural conversation moves to §4-§7.

---

## 4. The per-tenant economics

Fine-tuning has direct costs (compute, data preparation, eval) and indirect costs (lifecycle, infrastructure, drift). Per-tenant fine-tuning multiplies both.

### 4.1 The direct cost per tenant

For a tenant getting their own fine-tuned model:

- **Data preparation.** Labeling, cleaning, formatting, validation. Highly variable; typical range: $5k-$50k per tenant for the initial training set.
- **Fine-tune compute.** Provider-side fine-tuning cost; depends on model size, training tokens, epochs. Typical range: $500-$10k per fine-tune run.
- **Eval suite construction.** A bespoke eval suite for this tenant's workload; typically $5k-$25k initial labor.
- **Initial eval runs.** Running the eval suite against the fine-tuned model; small per run; significant cumulatively across iterations.

Initial cost per tenant: $10k-$85k. Highly variable.

### 4.2 The recurring direct cost

Per tenant per year:

- **Re-fine-tune cycles.** Models are re-fine-tuned when the tenant's domain shifts, when the base model changes, or quarterly as a discipline. 4-12 times per year. $500-$10k per cycle.
- **Eval re-runs.** Every re-fine-tune triggers eval re-runs. ~$500-$2k per cycle.
- **Drift monitoring.** Production monitoring to catch quality drift. Per-tenant overhead.

Recurring cost per tenant per year: $5k-$50k.

### 4.3 The indirect cost

The dominant hidden cost is the operational cost (§5): per-tenant model lifecycle, per-tenant eval, per-tenant rollback. This translates to engineering FTE allocated to "operating per-tenant fine-tunes."

For 10 tenants on per-tenant fine-tuning: ~1 FTE allocated to ops. At loaded cost ($250k/year), that's $25k/year per tenant just for ops.

### 4.4 The break-even analysis

Per-tenant fine-tuning breaks even when:

```
(per-call cost saving × volume) > (fine-tune direct + indirect cost per tenant)
```

Example: tenant makes 10M calls/year. Frontier model: $0.04/call → $400k/year. Fine-tuned model: $0.004/call → $40k/year. Per-call saving: $360k/year. Fine-tune cost: $50k/year (rough). Break-even: ~7x.

Example: tenant makes 100k calls/year. Frontier model: $4k/year. Fine-tune model: $400/year. Saving: $3.6k/year. Fine-tune cost: $50k/year. Break-even: never (loses $46k/year).

The first example justifies fine-tuning. The second does not. Volume is the dominant input.

### 4.5 The "free" fine-tune trap

Some providers offer "free" fine-tuning (the per-fine-tune cost is zero or minimal). The lifecycle cost still applies. "Free fine-tune" reduces one column of the cost; it doesn't eliminate the others. Don't let "free" cloud the per-tenant economic analysis.

### 4.6 The cost-attribution discipline

Per-tenant fine-tuning costs should be attributed to the tenant in your cost analytics. Without per-tenant cost attribution, the cost of fine-tuning per tenant is invisible, and the decision to fine-tune for tenant X is made without knowing what it costs. Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md).

---

## 5. The operational cost

The dominant hidden cost. Most teams that build per-tenant fine-tuning miss this until 12-18 months in.

### 5.1 Per-tenant model lifecycle

Each tenant model goes through the same lifecycle as a shared model:

- Training data preparation and approval.
- Fine-tune run.
- Eval against tenant's eval suite.
- Staging deployment.
- Canary or shadow rollout.
- Full rollout.
- Production monitoring.
- Periodic re-fine-tune.
- Deprecation when superseded.

For 10 tenants × 4 lifecycle events per year = 40 lifecycle events per year. Each event is process overhead (review, approval, eval, deploy, monitor). Cross-link to [ai-engineering-reference-architecture / model-lifecycle](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle).

### 5.2 Per-tenant eval suite

Each tenant model needs its own eval suite. The suite captures:

- The tenant's specific output requirements.
- The tenant's edge cases.
- Regression tests for past failures.
- Quality metrics specific to the tenant's workload.

Each suite must be maintained as the tenant's workload evolves. Test cases age out; new cases get added; the suite grows.

For 10 tenants: 10 eval suites. Each suite has dozens to hundreds of test cases. Each suite must run on every base-model change, every fine-tune iteration, every regression review.

The eval suite is often the single largest engineering investment in per-tenant fine-tuning. Underbudgeting it produces low-quality eval, which produces undetected quality regressions in production.

### 5.3 Per-tenant rollback

When a fine-tuned model has a quality regression, rollback means returning to the previous tenant model. The infrastructure must:

- Track per-tenant model versions.
- Support per-tenant rollback without affecting other tenants.
- Have a runbook for "tenant X's model is regressing; roll back to vN-1."

The rollback infrastructure is generally not different from non-tenant model rollback; the difference is the per-tenant blast radius and the per-tenant communication ("we're rolling back your model; here's why").

Cross-link to [ai-engineering-reference-architecture / model-lifecycle / rollback-procedures](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle) (planned).

### 5.4 Per-tenant deprecation

When a tenant's fine-tuned model is superseded (new base model, new training methodology), the old model is deprecated. The customer is notified; migration to the new model happens (cross-link to §7); the old model is removed.

For 10 tenants × occasional deprecations: a steady stream of communication and migration work.

### 5.5 The base-model change cascade

When the base model is updated (new Claude Sonnet, new Llama, etc.), every tenant's fine-tune needs to be re-evaluated:

- Does the new base model + tenant's adapter/LoRA still meet quality bars?
- Should the tenant be re-fine-tuned on the new base?
- Should the tenant's adapter be migrated to the new base?
- What's the staging-to-production rollout per tenant?

For 10 tenants × 2-4 base-model changes per year: 20-40 per-tenant re-evaluations per year.

This is the cascade that makes per-tenant fine-tuning expensive. Each base-model change isn't one operation; it's N tenant operations.

### 5.6 Per-tenant drift monitoring

Quality drift in a fine-tuned model can come from:

- Distribution shift in the tenant's input data.
- Schema drift in the tenant's domain.
- The model's own degradation over time (rare for transformers; possible for some architectures).

Detection requires per-tenant production monitoring: per-tenant quality metrics, per-tenant eval-set replay, per-tenant complaint analysis. Cross-link to [ai-engineering-reference-architecture / observability-and-telemetry / quality-drift-detection.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/quality-drift-detection.md).

### 5.7 The FTE math

A reasonable rule of thumb: 1 FTE per ~10 actively-managed per-tenant fine-tuned models, assuming the model is a meaningful workload (not abandoned).

At 50 tenants on per-tenant fine-tuning: 5 FTE in ops. At loaded cost ($250k/FTE): $1.25M/year. That cost is rarely visible at the "let's offer per-tenant fine-tuning" decision.

---

## 6. The adapter / LoRA architecture

If per-tenant fine-tuning is the right answer, adapters / LoRAs are the architectural pattern that makes it operationally feasible.

### 6.1 What adapters change

An adapter (or LoRA — Low-Rank Adaptation) is a small parameter delta on top of a base model. Instead of fine-tuning the full model (resulting in a 70B-parameter file per tenant), the adapter is a small file (~100MB) that's loaded on top of a shared base model.

**What this changes about the inference architecture.**

- One base model serves all tenants.
- Each tenant has their own small adapter file.
- At inference time, the appropriate tenant's adapter is loaded into the base.
- vLLM, TGI, and similar frameworks support adapter loading (some with millisecond switching).

**What this changes about cost.**

- Inference cost is similar to non-fine-tuned base model (slight overhead for adapter switching).
- Storage cost per tenant: small (~100MB per adapter vs ~140GB per full model).
- Training cost per tenant: lower (only the adapter parameters are trained, not the full model).

**What this does not change.**

- Eval suite cost (still per tenant).
- Lifecycle cost (still per tenant).
- Drift monitoring (still per tenant).
- The decision of whether to do fine-tuning (still applies).

### 6.2 The adapter inference architecture

```
                   ┌──────────────────────┐
                   │  Adapter store        │
                   │  (per-tenant LoRAs)  │
                   └──────────┬───────────┘
                              │
                              │ (loaded on request)
                              │
                   ┌──────────▼───────────┐
                   │  Inference server     │
                   │  (vLLM with          │
                   │   multi-LoRA         │
                   │   support)           │
                   │                       │
                   │  Base model (Llama   │
                   │   8B) loaded once    │
                   └──────────┬───────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼────┐         ┌─────▼─────┐         ┌─────▼─────┐
   │Tenant A │         │ Tenant B  │         │ Tenant C  │
   │adapter  │         │ adapter   │         │ adapter   │
   └─────────┘         └───────────┘         └───────────┘
```

The inference server holds the base model in GPU memory. When a tenant's request arrives, the tenant's adapter is loaded (often pre-loaded for active tenants); inference runs against base + adapter.

### 6.3 The adapter operational architecture

- **Adapter storage.** Adapter files stored in a per-tenant directory; access controlled.
- **Adapter loading.** Inference server loads adapters on demand; LRU eviction for cold tenants.
- **Adapter versioning.** Each tenant has multiple adapter versions; the active version is referenced by tenant configuration.
- **Adapter deployment.** New adapter version deployed via the per-tenant model lifecycle (§5.1).

### 6.4 When adapters work and when they don't

**Adapters work well when.**

- The fine-tune objective is incremental on the base model (style adjustment, domain terminology).
- The base model is open-weight (Llama, Mistral, Qwen).
- The inference platform supports multi-LoRA (vLLM, TGI).
- The training data is in the "small data" regime (hundreds to low thousands of examples).

**Adapters work poorly when.**

- The fine-tune objective requires substantial parameter change (large capability shift).
- The base model is closed (hosted providers — most don't expose adapter-compatible APIs).
- The training data is very large (full fine-tune may converge better).

### 6.5 Adapters + hosted models

Some hosted providers (e.g., OpenAI's fine-tuning, Anthropic's planned fine-tuning support, AWS Bedrock model customization) offer fine-tuning that's effectively adapter-based under the hood. The provider manages adapter storage and loading; you submit training data and get an endpoint.

**Pro.** No GPU infrastructure to manage.

**Con.** Per-tenant cost on the hosted provider; less control over the adapter; usually no way to inspect or migrate.

For most teams in 2026, the hosted adapter route is the path of least resistance — if fine-tuning is the right answer at all.

---

## 7. Per-tenant model migration

When you have N tenant models instead of one, migration becomes an N-times problem.

### 7.1 The migration triggers

- New base model released; you want to use it.
- Existing base model deprecated by provider.
- Fine-tuning methodology improves; you want to re-fine-tune.
- Regulatory or compliance change requires re-training.
- Tenant's domain evolved; old fine-tune is stale.

Each trigger may apply to all tenants or a subset. Migration must be per-tenant decidable.

### 7.2 The migration playbook

For each tenant migration:

1. **Construct the new model.** Re-fine-tune the new base on the tenant's training data (or migrate the adapter).
2. **Run the tenant's eval suite.** Verify quality bar is met.
3. **Stage deployment.** Deploy new model in staging; run shadow traffic.
4. **Canary or A/B.** Roll out to a fraction of tenant's traffic; compare quality and latency.
5. **Full deployment.** Replace old model with new for the tenant.
6. **Decommission old.** After confirmation period, remove old model.
7. **Document.** Update tenant-facing release notes (if visible to tenant).

Each step is per tenant. The playbook is the same; the execution multiplies.

### 7.3 The parallel-migration architecture

Some platforms migrate tenants in parallel — staging, canary, and full rollout happen for multiple tenants simultaneously, gated on the eval suite.

**Pro.** Throughput; faster cumulative migration.

**Con.** Risk concentration; if a base-model change has a systemic issue, multiple tenants are affected simultaneously.

**Recommendation.** Sequential canary (first tenant fully through staging-to-full before second tenant starts canary); parallel rollout once canary signal is good. Balances throughput and risk.

### 7.4 The rollback mid-migration

If a tenant's migration shows regression in canary, rollback is per-tenant: tenant's traffic returns to old model; other tenants continue on new model. Per-tenant rollback infrastructure is required (§5.3).

### 7.5 The "tenant's old adapter doesn't work on new base" reality

Adapters are coupled to their base model. A LoRA trained on Llama 3 8B doesn't transfer to Llama 4 8B. Migration to a new base means re-training the adapter on the new base — full fine-tuning cost, just per-tenant.

Some research suggests adapter-portability across base versions; in practice (2026), portability is not robust enough to rely on. Plan for re-training, not migration.

### 7.6 The communication burden

Per-tenant migrations require per-tenant communication. Each tenant's account manager:

- Knows the migration schedule.
- Communicates to the tenant ("we're upgrading your model on X date").
- Coordinates if the tenant wants to test in advance.
- Handles concerns ("our team relies on the old model's specific quirks").

For 10 tenants × 2-4 migrations per year: 20-40 tenant communications per year. Real cost in account-management time.

---

## 8. Worked Meridian example

Meridian Health's experience with per-tenant fine-tuning illustrates the architectural decisions and the alternatives.

### 8.1 The initial request

In late 2025, Meridian's largest tenant (5000-user regional health system, enterprise tier) asked: "can you fine-tune the model on our clinical notes? We want the AI to be familiar with our specific care protocols."

The technical team's first reaction: implement per-tenant fine-tuning infrastructure. The customer was paying premium; the request was direct; the architecture conversation was deferred to a "yes, we can do that."

### 8.2 The architectural pause

Before building, the team asked: what does the customer actually need? Conversation with the customer's clinical informatics team revealed:

- They want the AI to use their care-protocol terminology (e.g., "DKA protocol v3.1" not "DKA management").
- They want the AI to reference their specific guidelines when relevant.
- They want responses styled in their voice.
- They want the AI to know their formulary preferences.
- They want assurance that their data is private.

All five are addressable with RAG + per-tenant overlay. The team proposed:

- Ingest the customer's care protocols, guidelines, formulary, and clinical reference materials into a tenant-scoped retrieval index.
- Build a tenant-specific overlay covering terminology and voice preferences.
- Per-tenant tool catalog excluding tools that conflict with their workflows.
- Per-tenant output validation against their entity schema.
- Document the data isolation guarantees.

### 8.3 The trial

The customer agreed to a 90-day trial of RAG + overlay before committing to fine-tuning. The team:

- Spent 4 weeks ingesting and embedding the customer's content (~80k documents).
- Spent 1 week curating the overlay with the customer's clinical informatics team.
- Spent 1 week building the tenant-specific eval suite.
- Ran the suite + production validation for 60 days.

Results after 90 days:

- Clinical accuracy on the tenant's eval suite: 94% (vs 91% without RAG, 95% with theoretical perfect fine-tune).
- Tenant terminology adherence: 96% (vs 60% without overlay).
- Customer satisfaction (clinical informatics team): high; "responses feel like they understand us."
- Cost: standard tier rates × usage; ~$8k/month for this tenant.

The customer agreed not to fine-tune. The 3% accuracy gap to "theoretical perfect" was not worth the lifecycle cost.

### 8.4 The one case where fine-tuning was right

In Q1 2026, a different tenant (mid-size specialty practice, ~500 users, premium tier) had a workflow that didn't fit RAG: medical billing code suggestion from clinical notes. The output had to be a precise list of ICD-10 and CPT codes ranked by confidence, in a specific format for downstream billing system import.

Few-shot examples got the structure right ~85% of the time. The remaining 15% had hallucinated codes or wrong format. RAG didn't help because the codes themselves are well-known; the issue was structured-output reliability.

Per the test in §2.1, this was a valid fine-tuning case: domain-specific output structure with reliability gap that prompt engineering couldn't close.

The team:

- Fine-tuned a Llama 3 8B model on ~25k (note, code list) pairs from the tenant's anonymized historical data.
- Hosted on the team's GPU pool with vLLM + LoRA support.
- Built tenant's eval suite (~500 test cases).
- Deployed with per-tenant lifecycle.

Results:

- Structured-output reliability: 99.4% (vs 85% with few-shot).
- Cost per call: $0.003 (vs $0.040 with frontier model).
- Volume: 50k calls/month → $150/month vs $2000/month with frontier.
- Annual savings: ~$22k.
- Annual operational cost (FTE allocation): ~$25k.

Break-even: just barely. The decision was justified by the *workflow* (the structured-output requirement couldn't be met otherwise), not by the cost. The cost was a wash.

### 8.5 The decision rule going forward

Meridian's platform team adopted a documented rule:

> Per-tenant fine-tuning is not offered as a standard feature. It will be considered for individual tenants when:
> - The workload has a specific structural output requirement that prompt-engineering cannot reliably elicit, **or**
> - The workload's per-call cost × volume exceeds $30k/year and a fine-tuned model could plausibly reduce cost by 5x or more, **and**
> - The tenant agrees to the lifecycle commitment (eval suite, rollback responsibility, drift monitoring participation).
>
> The default architecture for tenant-specific knowledge needs is RAG + per-tenant overlay + few-shot + output validation.

The rule has been applied to ~15 inbound requests since adopted. 14 were addressed with RAG + overlay; 1 (billing-code workflow) became a fine-tune.

### 8.6 What the team avoided

By not building per-tenant fine-tuning as a general feature:

- Avoided ~12 unnecessary fine-tune pipelines.
- Avoided ~1 FTE in ongoing per-tenant ops.
- Avoided the multi-tenant eval-suite multiplication.
- Avoided the base-model-cascade migration cost.

Estimated avoided cost: $300-500k/year in ops + infrastructure. The team has 5 engineers; this avoided cost is ~20% of the team's capacity.

---

## 9. Anti-patterns

### 9.1 The "tenant asked, so we'll build" reflex

**Pattern.** Tenant requests fine-tuning. Engineering builds per-tenant fine-tuning infrastructure. 18 months later, only 2 tenants actually use it; the rest of the infrastructure is dead weight.

**Corrective.** Architecture conversation before infrastructure. Understand what the tenant actually needs; assess whether RAG + overlay handles it; build fine-tuning only when justified by the §2 tests.

### 9.2 The "free fine-tune is free" miscalculation

**Pattern.** Provider offers free fine-tuning. Team interprets as "no cost." Misses the lifecycle, eval, and ops costs that aren't waived.

**Corrective.** Full cost accounting includes direct + indirect. "Free fine-tune" reduces one column; the others still apply.

### 9.3 The eval suite shared across tenants

**Pattern.** N tenants on fine-tuned models. One eval suite shared across them. The suite tests generic quality; each tenant's specific quirks are not tested. Drift in tenant-specific behavior is undetected.

**Corrective.** Per-tenant eval suite. Maintained per tenant; run on every change affecting that tenant's model.

### 9.4 The adapters-without-discipline experiment

**Pattern.** Adopt LoRAs because they're cheap to train. Build hundreds of per-tenant adapters without lifecycle discipline. After 12 months, nobody knows which adapters are in use, which are stale, which need re-training.

**Corrective.** Same lifecycle for adapters as for full fine-tunes. Adapters are cheap to *train*; they're not cheap to *operate*.

### 9.5 The fine-tune that solved a prompt problem

**Pattern.** Fine-tune because "prompt engineering couldn't solve it." Six months later, a junior engineer reads the prompt and realizes it was missing one line of instruction. Fine-tune was unnecessary.

**Corrective.** Exhaust prompt engineering (with senior review) before committing to fine-tune. Document why prompt engineering was insufficient.

### 9.6 The hardcoded model in fine-tune pipeline

**Pattern.** Fine-tune pipeline hardcodes Llama 3 8B. When Llama 4 lands, migration is a rewrite, not a config change.

**Corrective.** Base model is a parameter, not a hardcode. Migration to new base is a config change + re-train, not a code change.

### 9.7 The "we'll deprecate later" model proliferation

**Pattern.** Old fine-tuned models accumulate. "We'll deprecate them later" never happens. Storage costs grow; eval-suite confusion grows; nobody knows which model a given tenant is on.

**Corrective.** Active lifecycle management. Old versions retired on schedule (e.g., N+2 old version retired). Per-tenant model version tracked; one active version per tenant.

### 9.8 The customer who fine-tuned-and-left

**Pattern.** Customer requests fine-tuning. Team builds it. Customer churns 6 months later. Fine-tune infrastructure orphaned but kept running because nobody dares delete it.

**Corrective.** Tie fine-tune infrastructure to active contracts. When a tenant churns, decommission their fine-tune within N days. Periodic audit of "still-active fine-tunes vs still-paying tenants."

---

## 10. Findings (sprint-assignable)

**ARCH-PTF-001 (P0). Per-tenant fine-tuning offered without justification framework.** Inbound requests build infrastructure before assessing whether RAG + overlay suffices. Adopt documented decision rule (§2 tests) for per-tenant fine-tuning. Owner: AI platform + product.

**ARCH-PTF-002 (P0). No per-tenant eval suite for fine-tuned models.** Quality drift in tenant-specific behavior is undetected. Build per-tenant eval suite for each fine-tuned tenant; run on every model change. Owner: AI platform.

**ARCH-PTF-003 (P0). No per-tenant model lifecycle (deploy, rollback, deprecate).** First fine-tune is operational chaos. Implement per-tenant lifecycle infrastructure (versioned models, per-tenant deploy, per-tenant rollback). Owner: AI platform.

**ARCH-PTF-004 (P0). Per-tenant fine-tuning cost not attributed to tenant.** Decision to fine-tune for tenant X made without knowing cost. Implement per-tenant cost attribution including fine-tune lifecycle costs. Owner: FinOps + AI platform.

**ARCH-PTF-005 (P1). RAG + overlay not exhausted before considering fine-tuning.** Fine-tuning chosen when prompt-engineering would have sufficed. Adopt requirement: prompt + RAG + overlay + few-shot + structured output must be tested before fine-tune approved. Owner: AI platform + tenant ops.

**ARCH-PTF-006 (P1). Adapters used without lifecycle discipline.** Hundreds of adapters accumulate; nobody knows which are active. Apply same lifecycle to adapters as to full fine-tunes. Owner: AI platform.

**ARCH-PTF-007 (P1). Base model hardcoded in fine-tune pipeline.** Migration to new base requires code change. Parameterize base model; migration is config + re-train. Owner: AI platform.

**ARCH-PTF-008 (P1). No deprecation schedule for old tenant models.** Old versions accumulate; storage costs grow. Active lifecycle: N+2 old version retired. Owner: AI platform.

**ARCH-PTF-009 (P1). Fine-tune infrastructure runs for churned tenants.** Orphaned infrastructure costs money; nobody dares delete. Tie fine-tune infrastructure to active contracts; periodic audit. Owner: AI platform + revenue ops.

**ARCH-PTF-010 (P1). No production monitoring of per-tenant model quality.** Drift in tenant-specific behavior undetected until customer complains. Implement per-tenant quality metrics; per-tenant eval-set replay in production; per-tenant alerting. Owner: AI platform + observability.

**ARCH-PTF-011 (P2). No documented break-even analysis for per-tenant fine-tune.** Fine-tune offered without business case. Document per-tenant economic analysis template; require before approval. Owner: product + FinOps.

**ARCH-PTF-012 (P2). Migration playbook for tenant fine-tunes is ad-hoc.** Each migration improvised. Document standard playbook: prep, eval, stage, canary, deploy, decommission. Owner: AI platform.

**ARCH-PTF-013 (P2). No per-tenant rollback runbook.** First fine-tune regression is improvised crisis. Per-tenant rollback runbook; tested in pre-production. Owner: SRE + AI platform.

**ARCH-PTF-014 (P2). Per-tenant adapter loading not optimized for cold tenants.** First request after cold-load has elevated latency. Pre-load active tenants' adapters; LRU eviction for cold; warm-up procedure for newly active. Owner: AI platform.

**ARCH-PTF-015 (P2). No customer-facing documentation of per-tenant model lifecycle.** Customer surprised by deprecation notice. Document lifecycle to customers; SLA terms include migration cadence. Owner: product + customer success.

**ARCH-PTF-016 (P3). Adapter storage not access-controlled per tenant.** Cross-tenant adapter access would be a data exfiltration vector. Per-tenant adapter directory with access controls; audit logging on access. Owner: AI platform + security.

**ARCH-PTF-017 (P3). No "is adapter portable to new base" research before base migration.** Base migration assumes adapter is portable; discovers it isn't mid-migration. Test adapter portability in advance; plan re-train if not portable. Owner: AI platform.

**ARCH-PTF-018 (P3). FTE cost of per-tenant fine-tuning ops not budgeted.** Team takes on fine-tuning without budgeting ops capacity; gets stretched. Document FTE-per-tenant cost; budget against tenant count. Owner: engineering management.

---

## 11. Adoption sequencing checklist

For a team considering per-tenant fine-tuning, in order:

- [ ] **Adopt the §2 decision framework.** Per-tenant fine-tuning is offered only when one of the tests matches.
- [ ] **Build the RAG + overlay + few-shot + structured-output stack first.** Per-tenant fine-tuning starts where this stack ends.
- [ ] **Document the per-tenant economic analysis template (§4).** Required before any per-tenant fine-tune decision.
- [ ] **Build per-tenant cost attribution.** Visibility into fine-tune lifecycle cost per tenant.
- [ ] **If committing to per-tenant fine-tuning, adopt adapter / LoRA architecture (§6).** Don't build per-tenant full-fine-tunes if you can avoid it.
- [ ] **Build per-tenant model lifecycle infrastructure (§5).** Versioned models, per-tenant deploy, per-tenant rollback.
- [ ] **Build per-tenant eval suite for each fine-tuned tenant.** Run on every model change.
- [ ] **Build per-tenant production monitoring.** Per-tenant quality metrics; drift detection.
- [ ] **Document migration playbook (§7).** Standard procedure for base-model changes and re-fine-tune cycles.
- [ ] **Plan FTE capacity.** Budget ~1 FTE per ~10 actively fine-tuned tenants for ops.
- [ ] **Adopt deprecation schedule (§5.4).** Old versions retired; tenant communication built in.
- [ ] **Audit fine-tune infrastructure against active contracts.** Decommission orphaned fine-tunes.
- [ ] **Document customer-facing lifecycle.** Customer SLA covers migration cadence, deprecation notice.
- [ ] **Pre-production test the rollback path.** Per-tenant rollback works; doesn't affect other tenants.

---

## 12. References

**In this folder.**
- [isolation-models.md](./isolation-models.md) — the three isolation models; per-tenant fine-tuning is the dedicated-infrastructure extreme.
- [per-tenant-prompt-and-context.md](./per-tenant-prompt-and-context.md) — the RAG + overlay architecture that handles most "tenant-specific" needs without fine-tuning.
- [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) — the data-layer namespacing that underpins tenant-scoped RAG.
- [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md) — the broader leakage-path catalog.
- [noisy-neighbor-mitigation.md](./noisy-neighbor-mitigation.md) — performance isolation for shared infrastructure.

**Elsewhere in this repo.**
- [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](../model-strategy/frontier-vs-open-weights-vs-fine-tune.md) — foundational decision framework that informs per-tenant fine-tuning.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — routing patterns that compose with per-tenant model selection.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — companion; per-tenant fine-tuning is the extreme case of "build."
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — the catalog tracks per-tenant model variants.
- [reference-systems/patient-api-ai-assist.md](../reference-systems/patient-api-ai-assist.md) — the worked multi-tenant reference system.

**Sibling repos.**
- [ai-engineering-reference-architecture / model-lifecycle / model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md) — engineering-side model registry that tracks per-tenant variants.
- [ai-engineering-reference-architecture / eval-engineering / eval-engineering-playbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/eval-engineering-playbook.md) — per-tenant eval suite construction.
- [ai-engineering-reference-architecture / observability-and-telemetry / quality-drift-detection.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/quality-drift-detection.md) — per-tenant drift monitoring.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-tenant cost attribution including fine-tune costs.

**External.**
- LoRA paper (Hu et al., 2021) and subsequent multi-LoRA inference research.
- vLLM multi-LoRA documentation.
- TGI multi-LoRA documentation.
- Anthropic / OpenAI / AWS Bedrock fine-tuning service documentation.
- Provider pricing pages for fine-tuning service economics.
