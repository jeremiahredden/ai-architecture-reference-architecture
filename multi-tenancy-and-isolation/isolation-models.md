# Isolation Models for AI Systems

> **Audience.** Architects scoping a multi-tenant AI platform. Tech leads renegotiating an AI system's tenancy posture in response to a new contract, regulatory change, or premium-customer commitment. **Scope.** The *architectural* framework for choosing the multi-tenant isolation model for an AI system; the three models, their trade-offs, migration paths between them. Not the per-layer implementation patterns (see [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) for the retrieval-layer pattern; [per-tenant-prompt-and-context.md](./), [cross-tenant-leakage-prevention.md](./), [noisy-neighbor-mitigation.md](./), [per-tenant-fine-tuning.md](./), [data-residency-patterns.md](./) for the others — all coming). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Multi-tenancy in AI systems is harder than multi-tenancy in conventional software. The data lives in opaque places (vector stores, conversation histories, embeddings) where the convenient enforcement primitives of conventional databases (row-level security, schema-per-tenant) do not directly apply. The model itself has no concept of tenancy and will gladly use any context it is given. A single misconfigured retrieval filter, a single prompt-context leak, a single tool that bypasses scoping — and tenant A's data appears in tenant B's response.

So the *isolation model* — the overall architectural posture toward how much each tenant is isolated from every other tenant — is one of the most consequential decisions in any multi-tenant AI platform. It is also one of the most expensive decisions to reverse later. A platform that started shared-everything and is now under audit pressure to move to per-tenant infrastructure faces a multi-quarter migration.

This document is the framework for making the choice deliberately rather than by default. The patterns documented in the rest of this folder (per-tenant vector namespacing, per-tenant prompt and context, cross-tenant leakage prevention, etc.) all assume an isolation model is chosen; this document is the one that helps you choose.

This document is opinionated about three things:

1. **The choice is between three discrete models**, not a continuum. The decision should be explicit and documented, with the trade-offs understood and accepted.
2. **The choice is per-tenant-class, not per-tenant.** A platform serves multiple tenant classes (standard, premium, regulated, custom-contract) — different classes can occupy different models.
3. **Migration paths are deliberate projects**, not "we'll just refactor." Each migration direction has known difficulty; budget accordingly.

Structure: (2) the questions; (3) the three models in depth; (4) per-class assignment; (5) migration paths; (6) the per-layer enforcement that backs the chosen model; (7) worked Meridian example; (8) anti-patterns; (9) findings; (10) adoption checklist.

---

## 2. The questions

Four questions. Answer in writing before reading the model catalogue. The answers usually point at one or two candidate models per tenant class.

1. **What is the regulatory / contractual bar per tenant class?** HIPAA / financial / public-sector / SOC 2 — what each customer requires from you. Some customers have specific contract clauses about dedicated infrastructure; others accept shared infrastructure with strong logical isolation; consumer-shaped customers have no per-tenant requirements at all.
2. **What is the per-tenant data sensitivity?** Clinical PHI is different from public-domain content. The more sensitive the data, the higher the bar for isolation guarantees.
3. **What is the cost economics?** Per-tenant infrastructure has a per-tenant cost floor; shared infrastructure scales economically across many tenants. What is each tenant's revenue contribution, and what does that allow as infrastructure cost?
4. **What is the team's operational capacity?** Each isolation model has its own operational shape. Per-tenant infrastructure requires per-tenant capacity planning, patching, backup verification — at scale this is a meaningful platform investment.

The answers shape the model selection. The framework in section 4 maps the answers to model recommendations.

---

## 3. The three models

### 3.1 Model A: Shared-everything with filtering

**Shape.** All tenants share the same vector store (one index, one namespace), the same conversation store (one schema), the same retrieval pipeline, the same model deployment, the same audit log. Tenant isolation is enforced by *filters* — every query includes a `WHERE tenant_id = ?` clause, every retrieval result is checked for tenant attribution, every prompt is scoped to the right tenant context.

```
                Application code
                       │
                       ▼
        ┌──────────────────────────────────┐
        │     Shared infrastructure         │
        │     (one index, one model        │
        │      deployment, one audit log)   │
        │                                   │
        │   tenant_id field on every record │
        │   filter clause on every query    │
        └──────────────────────────────────┘
```

**Sweet-spot workload.**
- Consumer / B2C workloads where there is no per-user infrastructure expectation.
- Internal-tooling workloads inside a single organization.
- Workloads with low regulatory bar and low per-tenant data sensitivity.

**What it gives you.**
- **Cheapest.** No per-tenant operational floor; shared infrastructure cost amortizes across all tenants.
- **Simplest operationally.** One thing to patch, monitor, capacity-plan.
- **Most flexible.** Cross-tenant queries (admin analytics, platform-level operations) are easy.

**What it costs.**
- **Highest blast radius.** A single missing filter exposes every tenant's data to every other tenant. The defense is convention and testing; convention degrades over time and turnover.
- **Weakest audit story.** "Tenants are logically isolated by metadata filters in queries" is harder to attest than "tenants are physically isolated."
- **No per-tenant capacity isolation.** One tenant's traffic spike affects all tenants.

**When this model is wrong.** Any regulated workload (HIPAA, financial, public-sector). Any customer contract requiring isolation guarantees. Any workload where a single cross-tenant leak would be a material incident.

### 3.2 Model B: Shared-model with isolated data

**Shape.** Model inference is shared across tenants (same provider deployment, same model version, same prompt structure), but data is isolated per-tenant. Vector stores use per-tenant namespaces or per-tenant logical partitions. Conversation history is per-tenant. The audit log is per-tenant-scoped. Prompts include explicit tenant context. Tools scope by tenant. Retrieval filters by tenant. The infrastructure is mostly shared but the *data* is logically isolated end-to-end.

```
              ┌─── shared model deployment ───┐
              │   (one Anthropic endpoint,    │
              │    one OpenAI endpoint, etc.) │
              └───────────────┬───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       tenant-A storage  tenant-B storage  tenant-C storage
       (namespace /      (namespace /      (namespace /
        partition /       partition /       partition /
        schema)           schema)           schema)
```

**Sweet-spot workload.**
- Regulated B2B SaaS with many tenants (the Meridian Care Coordinator default).
- Workloads where per-tenant data isolation is required but per-tenant model deployment is not justified.
- Workloads where the cost economics demand shared model deployment.

**What it gives you.**
- **Strong logical isolation of data.** Per-tenant namespacing in the vector store, per-tenant partitioning in databases, per-tenant scoping in conversation history and audit log.
- **Shared model cost economics.** Most of the cost is model inference; sharing the model deployment amortizes that across tenants.
- **Per-tenant operational levers (limited).** Per-tenant metrics, per-tenant policies, per-tenant rate limits — all configurable without per-tenant infrastructure.
- **Strong audit story for most regulated bars.** "Tenant data is in tenant-specific storage primitives" attests cleanly.

**What it costs.**
- **Defense-in-depth is the engineering investment.** Layered enforcement across all the per-layer patterns (retrieval client wrapper, prompt scoping, tool scoping, audit scoping) is what makes the isolation real. Half-built defense-in-depth produces gaps.
- **Shared-model failure modes affect everyone.** A model outage, a model deprecation, a model auto-upgrade affects all tenants simultaneously.
- **Noisy-neighbor mitigation is its own work.** Without per-tenant rate limits and capacity isolation, one tenant's traffic affects others.

**When this model is right.** Multi-tenant regulated SaaS at scale. The Meridian Care Coordinator for 240 standard-tier hospital tenants. The default for most B2B AI SaaS in 2026.

### 3.3 Model C: Dedicated infrastructure

**Shape.** Each tenant (or each tenant class) gets its own infrastructure — its own model deployment (or its own provider account), its own data stores, its own audit log, its own observability stack. Either fully separate, or "premium tier on dedicated, standard tier on shared." The premium customer pays for the dedicated posture; the platform team operates a smaller fleet of dedicated stacks.

```
        Premium customer A:
        ┌────────────────────────────────────┐
        │  Dedicated provider account         │
        │  Dedicated model deployment         │
        │  Dedicated data stores              │
        │  Dedicated audit log                │
        │  Dedicated observability            │
        └────────────────────────────────────┘

        Premium customer B:
        ┌────────────────────────────────────┐
        │  Dedicated provider account         │
        │  Dedicated model deployment         │
        │  ...                                │
        └────────────────────────────────────┘
```

**Sweet-spot workload.**
- Premium B2B contracts where the customer pays for dedicated infrastructure as a procurement commitment.
- Strict regulatory requirements (FedRAMP High, certain government / defense workloads, certain healthcare regulators) that require physical isolation.
- Workloads with such high per-tenant volume that dedicated infrastructure is cost-comparable to shared anyway.

**What it gives you.**
- **Strongest isolation.** Cross-tenant exposure is physically impossible at the infrastructure layer. The defense-in-depth investment is much smaller because the storage layer cannot be misconfigured into cross-tenant scope.
- **Per-tenant capacity isolation.** One tenant's traffic spike cannot affect others.
- **Per-tenant operational levers.** Re-deploy this tenant, patch this tenant, restart this tenant — without affecting others.
- **Strongest audit story.** "Customer A's data is in Customer A's dedicated infrastructure" is the simplest possible attestation.

**What it costs.**
- **Operational overhead per tenant.** Each tenant is a fleet to operate. Patching, monitoring, capacity planning, backup verification — all per-tenant.
- **Cost overhead per tenant.** Each tenant has a base cost floor regardless of usage. Small tenants pay more per-unit than they would on shared infrastructure.
- **Cross-tenant operations are awkward.** Platform-wide work (a model deprecation, an embedding model migration) becomes "do this once per tenant."

**When this model is right.** Premium-contract customers willing to pay for it. Regulated workloads where the regulator requires physical isolation. Very large tenants whose data volume itself justifies dedicated infrastructure.

---

## 4. Per-class assignment

Most real platforms have multiple tenant classes occupying multiple isolation models. The framework:

### 4.1 The decision table

| Tenant class | Recommended model | Why |
|---|---|---|
| Consumer / B2C | Shared-everything with filtering | Per-user infrastructure is not economically viable; no per-user regulatory requirement |
| Standard B2B (regulated, e.g. HIPAA) | Shared-model with isolated data | Logical isolation meets the regulatory bar; cost economics support shared inference |
| Premium B2B (with dedicated-infra contract) | Dedicated infrastructure | The customer contracted for it; the additional cost is in the pricing |
| Regulated public-sector (FedRAMP High, sovereign-cloud) | Dedicated infrastructure (often in a separate cloud account / region) | Physical isolation is explicitly required |
| Internal-tooling tenants | Shared-everything | No external regulatory bar; speed of iteration is the priority |

A platform may operate two, three, or four of these simultaneously. The Meridian Care Coordinator operates three: shared-everything for internal tooling (~5 tenants), shared-model-with-isolated-data for the 240 standard-tier hospitals, dedicated-infrastructure for the 1 premium-contract hospital.

### 4.2 The mixed-model platform discipline

When multiple models coexist:

- **The shared-model components serve the shared-model and shared-everything tenants** uniformly. The premium-tier tenant's dedicated stack is entirely separate.
- **The platform code is shared.** The same retrieval-client wrapper, the same agent-loop runner, the same gateway code runs in all environments. Configuration differs; logic does not.
- **The operational documentation is per-environment.** Standard-tier runbooks vs premium-tier runbooks vs internal-tooling runbooks. The runbooks share most content but have environment-specific deviations.

The discipline keeps the platform's complexity tractable. The alternative — separate codebases per tenant class — produces unmanageable divergence.

### 4.3 The pricing alignment

Tenant class is a business model decision as well as a technical one. The pricing should reflect the cost-to-serve:

- Standard tier pricing covers the shared-model cost per tenant.
- Premium tier pricing covers the dedicated-infrastructure cost per tenant plus margin.
- Internal tooling has no external pricing but should be cost-attributed for internal chargeback.

Misalignment (e.g., a standard-tier customer demanding premium-tier isolation without the premium price) is a sales conversation, not an engineering decision. The framework forces the conversation to happen explicitly.

---

## 5. Migration paths

Migrating between isolation models is a structured project. Each direction has known difficulty.

### 5.1 Shared-everything → shared-model-with-isolated-data

**Triggers.** New customer with regulatory requirements; existing customer asking for stronger isolation; audit pressure; cross-tenant incident.

**Difficulty.** High. Most of the data-side patterns need to be retrofitted: per-tenant namespacing in the vector store, per-tenant scoping in retrieval client, per-tenant partitioning in conversation history, per-tenant audit logging. Existing data needs to be migrated to the new structure.

**Approach.**
1. New data lands in the isolated-data structure immediately (parallel-write).
2. Backfill existing data into the new structure (often a multi-week project; reads can be served from either during the transition).
3. Switch reads to the new structure; verify with canary suite (per [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) section 6).
4. Decommission the old shared structure.

**Typical timeline.** 6–12 weeks for a moderately-sized system.

### 5.2 Shared-model-with-isolated-data → dedicated infrastructure (single premium tenant)

**Triggers.** Premium customer contract; regulatory escalation; tenant volume so large that shared infrastructure has noisy-neighbor issues.

**Difficulty.** Medium. The platform code is unchanged; what changes is the infrastructure deployment and the routing logic.

**Approach.**
1. Stand up the dedicated infrastructure (new provider account, new data stores, new audit log).
2. Migrate the tenant's data from the shared infrastructure (one-time copy + parallel-write during transition).
3. Update the routing layer so this tenant's calls route to the dedicated stack.
4. Decommission the tenant's data in the shared infrastructure.

**Typical timeline.** 4–8 weeks for a single tenant migration; the platform's first such migration takes longer (operational patterns being developed); subsequent migrations are routine.

### 5.3 Dedicated infrastructure → shared-model-with-isolated-data (consolidation)

**Triggers.** Customer downgrade from premium to standard; cost-reduction initiative; tenant volume dropped to where dedicated is no longer economic.

**Difficulty.** Medium. The migration is essentially the reverse of 5.2.

**Approach.**
1. Migrate the tenant's data into the shared infrastructure's per-tenant namespace.
2. Update routing to point at the shared infrastructure.
3. Customer-facing communication (typically the consolidation is the customer's choice, so they are aware).
4. Decommission the dedicated infrastructure.

**Typical timeline.** 4–8 weeks per tenant.

### 5.4 Shared-everything → dedicated (skipping shared-model)

**Triggers.** A high-stakes customer onboarding from a shared-everything platform onto dedicated infrastructure directly.

**Difficulty.** High (in a different way than 5.1). The platform may not have the shared-model-with-isolated-data patterns built yet, so the migration is "build the new pattern + build the dedicated variant simultaneously."

**Approach.** Usually best done as two sequential projects: first establish the shared-model-with-isolated-data pattern for all tenants (gives the platform the right primitives), then carve out the dedicated infrastructure for the specific customer.

**Typical timeline.** 3–6 months for the platform-wide upgrade + dedicated stack-up.

### 5.5 The reverse direction (shared-model → shared-everything) is almost never right

The reverse direction (a regulated tenant moving to shared-everything to save costs) is a regulatory regression. Almost never the right move. If cost is the issue, address it within the isolation model (tier routing, caching, per-tenant cost circuits) rather than weakening the isolation posture.

---

## 6. Per-layer enforcement

The chosen isolation model has to be enforced at every layer of the AI stack. This is where the other documents in this folder fit.

### 6.1 The layered enforcement map

| Layer | Pattern | Document |
|---|---|---|
| Storage (vectors) | Per-tenant namespace or per-tenant index | [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) |
| Storage (conversation history) | Per-tenant partitioning | [per-tenant-prompt-and-context.md](./) (coming) |
| Retrieval | Mandatory filter in retrieval-client wrapper | [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) |
| Prompt assembly | Tenant context as required field | [per-tenant-prompt-and-context.md](./) (coming) |
| Tool calls | Per-tenant scope in tool registry | [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) |
| Cost | Per-tenant cost circuits | [cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) |
| Capacity | Per-tenant rate limits, noisy-neighbor mitigation | [noisy-neighbor-mitigation.md](./) (coming) |
| Observability | Per-tenant attribution on every trace | [trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md) |
| Audit | Per-tenant audit log scoping | [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) |
| Cross-tenant prevention | Layered defense + canary testing | [cross-tenant-leakage-prevention.md](./) (coming) |
| Fine-tuning (if applicable) | Per-tenant model versions | [per-tenant-fine-tuning.md](./) (coming) |
| Residency | Region-pinning at infrastructure layer | [data-residency-patterns.md](./) (coming) |

Each layer is its own pattern with its own document. The isolation-model decision (this document) is the parent; the per-layer patterns are the children that implement it.

### 6.2 The enforcement must compose

A platform that enforces tenant isolation at the storage layer but not at the prompt-assembly layer has gaps. A platform that enforces at every layer but does not have cost-isolation has noisy-neighbor risk. The layered-defense pattern only works when all layers are enforcing.

The chosen isolation model determines *which layers need which level of enforcement*. Shared-everything skips most of the layers (it accepts the risk). Shared-model-with-isolated-data requires the full layered defense at each per-tenant point. Dedicated-infrastructure pushes most enforcement to the infrastructure layer (the dedicated provider account is the enforcement).

---

## 7. Worked Meridian Health example

### 7.1 The three tenant classes

Meridian Care Coordinator operates three tenant classes:

| Class | Count | Isolation model | Notes |
|---|---|---|---|
| Internal tooling | ~5 | Shared-everything | Engineering / dogfood / staging |
| Standard hospital tier | ~240 | Shared-model with isolated data | The bulk of customers |
| Premium hospital tier | 1 | Dedicated infrastructure | Contracted-for-dedicated; canary for the dedicated-infra operational pattern |

### 7.2 Per-class architecture differences

| Aspect | Internal | Standard | Premium |
|---|---|---|---|
| Vector store | Shared pgvector, no namespace | pgvector per-tenant namespace (the 240+ partitions described in `per-tenant-vector-namespacing.md`) | Dedicated Aurora cluster |
| Model deployment | Shared (the standard endpoints) | Shared (the standard endpoints) | Dedicated provider account |
| Audit log | Shared sink, low retention | Per-tenant scoped, 7-year retention | Dedicated audit log, 7-year retention |
| Observability | Shared traces, no special handling | Shared traces with tenant attribution | Dedicated trace stream |
| Cost circuits | Per-feature only | Per-tenant per-day + per-feature | Per-tenant per-day (higher) + dedicated alerting |
| Rate limits | None | Per-tenant default | Higher per-tenant |
| Operational pages | Standard | Standard | Higher priority |

### 7.3 The shared platform code

The same Python codebase serves all three classes. The configuration (per section 6.3 of [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md)) determines which infrastructure each call routes to. Tenant ID is the first-class dimension; tenant class is metadata on the tenant.

The discipline: no per-tenant-class code branches. Configuration drives behavior. The codebase is the same; the deployment shape differs.

### 7.4 The migration history

- **2024.** Pilot phase: shared-everything with filtering for 3 pilot hospitals. Acceptable for the pilot.
- **2025-Q1.** Decision to commit to shared-model-with-isolated-data for the GA scale-out. Migration project: ~10 weeks. Per-layer enforcement patterns landed.
- **2025-Q3.** First premium-contract customer signs; dedicated-infrastructure variant built. Migration project: ~6 weeks for this one customer.
- **Ongoing.** Standard-tier onboarding follows the shared-model-with-isolated-data pattern; new premium-contract customers go to the dedicated variant.

### 7.5 The cost-of-isolation conversation

The pricing tiers reflect the cost-to-serve:

- Internal tooling: not billed (cost is engineering overhead).
- Standard hospital tier: per-bed pricing covers shared-model cost + margin.
- Premium hospital tier: dedicated-infrastructure surcharge above the standard per-bed pricing.

A standard-tier hospital asking for dedicated infrastructure has the conversation: "the dedicated tier is X% more per bed; here is what you get for that." The technical conversation is downstream of the commercial one.

---

## 8. Anti-patterns

### 8.1 "Shared-everything for regulated workloads"

The team picked shared-everything because it was the fastest to ship. Months later, the first customer compliance review asks for evidence of isolation; the team scrambles to build defense-in-depth retrospectively; the audit finding is "implementation is in progress."

**Corrective.** Match the isolation model to the regulatory bar at design time. The cost premium of shared-model-with-isolated-data over shared-everything is small; the cost of a compliance miss is enormous.

### 8.2 "Dedicated infrastructure for everyone"

The team picked dedicated infrastructure for the strongest isolation, then discovered the per-tenant operational cost at scale. 240 tenants means 240 patching cycles, 240 capacity planning runs, 240 backup verification checks.

**Corrective.** Reserve dedicated infrastructure for tenants whose pricing covers it. Use shared-model-with-isolated-data for the bulk.

### 8.3 "Isolation by hope"

The team adopted shared-model-with-isolated-data and called the per-layer enforcement "best practices" rather than "required." Some layers got the enforcement; others did not; the gaps are isolation failures waiting to happen.

**Corrective.** The layered enforcement is the isolation model. Half-implemented defense-in-depth is a gap; commit to all the layers or commit to a different model.

### 8.4 "Per-class isolation models without configuration discipline"

The team has standard-tier and premium-tier tenants on different isolation models, implemented as per-class code branches scattered throughout the codebase. The premium-tier variant diverges over time; standard-tier features do not land in premium-tier.

**Corrective.** Single codebase; per-class configuration drives behavior. No code branches.

### 8.5 "Migration without canary"

The team migrates from shared-everything to shared-model-with-isolated-data with a flag flip. No canary suite verifies the migration; cross-tenant leak risks are not tested; production failures emerge.

**Corrective.** Canary suite per [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) section 6 runs through the migration. Cross-tenant tests are gates, not nice-to-haves.

### 8.6 "Premium-tier dedicated infrastructure without the operational platform"

The team built the dedicated-infrastructure variant for the premium tenant and then realized they have no per-tenant patching pipeline, no per-tenant backup verification, no per-tenant capacity monitoring. The dedicated stack drifts.

**Corrective.** Dedicated infrastructure requires a per-tenant operational platform. If the team cannot build / staff that platform, the premium contract should not promise dedicated infrastructure.

### 8.7 "Cross-tenant queries inside the regular code path"

Platform-wide analytics queries that span tenants are implemented inside the regular tenant-scoped code path with a "skip tenant filter" flag. The flag is convenient for engineering and a tenant-isolation hazard.

**Corrective.** Cross-tenant queries go through a separate, audited, explicitly-authorized client (per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) §9.3). Application code cannot import it.

### 8.8 "Reverse migration: weakening isolation to save cost"

A customer asks to move from premium-tier dedicated to standard-tier shared to save cost. The team considers it without revisiting the regulatory posture. The customer's data may now exist in shared infrastructure with weaker contractual isolation than promised.

**Corrective.** Weakening isolation requires renegotiating the contract. Cost reduction inside the existing isolation model (tier routing, caching, optimized prompts) is the right answer.

---

## 9. Findings (sprint-assignable)

### ARCH-ISO-001 — Severity: Critical
**Finding.** Regulated workload is on shared-everything-with-filtering; isolation is by application convention only.
**Recommendation.** Migrate to shared-model-with-isolated-data per the migration path in section 5.1; the layered enforcement patterns must all land.
**Owner.** ai-platform-eng + security-eng, sprint N+1 (planning), N+2..N+6 (execution).

### ARCH-ISO-002 — Severity: Critical
**Finding.** No documented isolation-model decision for the platform; new features adopt different patterns ad-hoc.
**Recommendation.** Document the chosen model per tenant class; circulate for sign-off; new features conform.
**Owner.** ai-platform-eng + security-eng + product, sprint N+1.

### ARCH-ISO-003 — Severity: High
**Finding.** Premium-tier customer contracted for dedicated infrastructure but is operationally served from shared infrastructure with "logical isolation."
**Recommendation.** Build out the dedicated-infrastructure variant per section 5.2; align with the contract; communicate the migration timeline.
**Owner.** ai-platform-eng + customer-success + security-eng, sprint N+1 (planning), N+2..N+6 (execution).

### ARCH-ISO-004 — Severity: High
**Finding.** Per-layer enforcement is partial: vector-store is per-tenant-namespaced but conversation history is in a shared partition without tenant scoping.
**Recommendation.** Land the missing per-layer enforcement patterns (conversation-history partitioning, prompt-context scoping, tool-scope, audit-log scoping, cost circuits).
**Owner.** ai-platform-eng, sprint N+2 through N+4.

### ARCH-ISO-005 — Severity: High
**Finding.** Per-tenant-class differences are implemented as code branches; standard-tier features lag in the premium-tier variant.
**Recommendation.** Refactor to configuration-driven; tenant-class is metadata, not a code switch.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-ISO-006 — Severity: High
**Finding.** Cross-tenant administrative queries are performed inside the regular code path with a `skip_tenant_filter=True` parameter.
**Recommendation.** Separate audited `AdminClient` per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md); remove the filter-skip parameter.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-ISO-007 — Severity: High
**Finding.** Migration from one isolation model to another has been performed without a canary test suite verifying cross-tenant isolation.
**Recommendation.** Canary suite per [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) section 6 is a required gate for any isolation-model migration.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-ISO-008 — Severity: High
**Finding.** Dedicated-infrastructure variant exists but the per-tenant operational platform (provisioning, patching, backup, monitoring) is not in place.
**Recommendation.** Build the per-tenant operational platform; or reduce the number of dedicated-infrastructure tenants to what current capacity can operate.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-ISO-009 — Severity: Medium
**Finding.** Pricing tiers do not align with isolation-model cost-to-serve; the premium tier's surcharge does not cover the dedicated-infrastructure cost.
**Recommendation.** Realign pricing with cost-to-serve; conversation with sales / finance / leadership.
**Owner.** product + finance + ai-platform-eng, sprint N+3.

### ARCH-ISO-010 — Severity: Medium
**Finding.** Tenant onboarding does not include explicit isolation-model assignment; new tenants land in the default class without intentional review.
**Recommendation.** Tenant onboarding workflow includes isolation-class selection; review by security-eng for regulated tenants.
**Owner.** ai-platform-eng + customer-success, sprint N+3.

### ARCH-ISO-011 — Severity: Medium
**Finding.** A request to weaken isolation (e.g., regulated tenant to shared-everything for cost reasons) was considered without revisiting the regulatory / contractual implications.
**Recommendation.** Establish the policy: weakening isolation requires explicit contract renegotiation and security-eng sign-off.
**Owner.** customer-success + security-eng + ai-platform-eng team lead, sprint N+3.

### ARCH-ISO-012 — Severity: Medium
**Finding.** Operational runbooks are unified across tenant classes; standard-tier and dedicated-tier incident response is the same runbook.
**Recommendation.** Per-class runbooks for differences; shared runbook for shared behaviors; documented per-class operational SLAs.
**Owner.** ai-platform-eng + sre, sprint N+4.

### ARCH-ISO-013 — Severity: Medium
**Finding.** Per-tenant cost attribution is not used to inform pricing or class assignment.
**Recommendation.** Per-tenant cost reports inform: pricing tier review, class assignment review, dedicated-vs-shared decisions.
**Owner.** ai-platform-eng + finops + product, sprint N+4.

### ARCH-ISO-014 — Severity: Medium
**Finding.** Internal-tooling tenants run on the production shared infrastructure; isolation between internal tooling and customer tenants is by convention only.
**Recommendation.** Either separate internal-tooling infrastructure or formalize the internal-tooling tenant class with the same isolation patterns as customer tenants.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-ISO-015 — Severity: Medium
**Finding.** A tenant class with high data residency requirements (e.g., EU customers) is served from US-region infrastructure.
**Recommendation.** Build the per-region isolation variant per [data-residency-patterns.md](./) (coming); or restrict that class to dedicated-region infrastructure.
**Owner.** ai-platform-eng + legal + customer-success, sprint N+4.

### ARCH-ISO-016 — Severity: Low
**Finding.** Documentation does not explain the isolation-model decision rationale to new engineers; the choice is treated as historical.
**Recommendation.** Architecture document explaining the per-class isolation model and the rationale; commit alongside the codebase; include in onboarding.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-ISO-017 — Severity: Low
**Finding.** Tenant-class metadata is not surfaced in observability; per-class behavior analysis is hard.
**Recommendation.** Add `ai.tenant.class` as a universal observability attribute; surface in dashboards.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### ARCH-ISO-018 — Severity: Low
**Finding.** Isolation-model audit-attestation documentation is informal; customer compliance teams request artifacts repeatedly.
**Recommendation.** Maintain a customer-facing isolation-architecture document; refresh annually; reference from customer agreements.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+5.

---

## 10. Adoption sequencing checklist

For a team designing the isolation model for a new multi-tenant AI platform:

- [ ] **Sprint 0 — decide.** Answer the four questions from section 2. Identify tenant classes. Choose the model per class. Document the rationale.
- [ ] **Sprint 0 — pricing alignment.** Confirm the pricing tier for each class covers the cost-to-serve of the chosen isolation model.
- [ ] **Sprint 1 — primary class.** Implement the per-layer enforcement for the highest-volume class (often the standard B2B tier). Reference the patterns in [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) etc.
- [ ] **Sprint 2 — defense-in-depth.** All the layers (vector storage, conversation history, prompt assembly, tool calls, audit, observability, cost) enforcing.
- [ ] **Sprint 3 — canary verification.** Cross-tenant canary test suite running in CI and as a release-candidate gate.
- [ ] **Sprint 4 — second class.** If multiple classes are served, implement the second class. For dedicated-infrastructure, this includes the operational platform.
- [ ] **Sprint 5 — onboarding workflow.** Tenant onboarding includes isolation-class selection; security-eng review for regulated tenants.
- [ ] **Sprint 5 — runbooks.** Per-class incident response documented; on-call walked through.
- [ ] **Ongoing — review.** Per-class metrics, per-class cost attribution, per-class onboarding cadence reviewed quarterly. Models adjusted with deliberate migration projects, not in-place refactors.

A team that completes this sequence has the isolation model that meets the regulatory bar, fits the cost economics, and stays operable. A team that skips the explicit decision discovers their isolation posture by audit — usually badly.

---

## 11. References

- HIPAA Security Rule §164.312 — the technical safeguards the regulated workload context implies.
- SOC 2 Common Criteria CC6 (logical access controls) — the framework most B2B SaaS customer compliance teams use.
- FedRAMP Moderate and FedRAMP High control families — the bar for public-sector workloads.
- Multi-tenancy patterns from the SaaS architecture canon (the AWS, Azure, GCP whitepapers on multi-tenant SaaS) — the foundation this AI-specific document builds on.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) ADR-CARE-008 — the worked tenancy decision.
- This repo: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) — the storage-layer enforcement.
- This repo: [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](./) (coming) — the prompt-layer enforcement.
- This repo: [multi-tenancy-and-isolation/cross-tenant-leakage-prevention.md](./) (coming) — the layered-defense across-surfaces pattern.
- This repo: [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](./) (coming) — the capacity-side concern.
- This repo: [multi-tenancy-and-isolation/data-residency-patterns.md](./) (coming) — the region / sovereignty concern.
- This repo: [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) — the tool-layer enforcement.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) — the cost-layer enforcement.
- Sibling repo: [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — the threat-model and attack-surface considerations.
- Sibling repo: [cloud-security-reference-architecture/landing-zones/](https://github.com/jeremiahredden/cloud-security-reference-architecture/tree/main/landing-zones) — the cloud-layer account / region isolation that dedicated-infrastructure depends on.
