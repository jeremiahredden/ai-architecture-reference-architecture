# Multi-tenancy and Isolation

## What this folder is

The architectural patterns for AI systems where more than one tenant shares the same model, the same retrieval infrastructure, or the same agent platform. The material here is what I put in front of a SaaS architecture review when the question is: *we are about to onboard a second customer onto the same AI platform — what isolation guarantees do we have, and where will they fail?*

## The organizing principle

Multi-tenancy in conventional software is a well-understood problem with mature patterns (per-tenant DB schemas, row-level security, encrypted-per-tenant storage). Multi-tenancy in AI systems is harder because the state lives in opaque places. The retrieved documents are in a shared vector index. The conversation history is in a shared store. The system prompt is shared and the per-tenant overlay is small. The model itself has no concept of tenancy and will gladly use any context it is given. A single misconfigured retrieval filter, a single tenant ID missing from a metadata field, a single prompt injection that exfiltrates context, and tenant A's data shows up in tenant B's response.

So the patterns here treat isolation as a *layered architectural requirement* — at the retrieval layer (per-tenant namespacing or filtering), at the prompt-assembly layer (tenant ID in context, output validation against tenant scope), at the storage layer (per-tenant encryption, per-tenant chunk attribution), at the agent layer (tool-call scoping), and at the cost layer (per-tenant accounting and rate limiting so noisy neighbors do not starve others).

The companion concern is *data residency*: B2B SaaS AI features in 2026 routinely face requirements that tenant data not leave specific regions, that some tenants get dedicated infrastructure, and that the audit trail be tenant-scoped. The patterns here are designed to make those guarantees architectural rather than operational.

## Planned documents

- **isolation-models.md** *(coming)* — The three isolation models for AI systems: shared-everything-with-filtering (cheapest, most failure-prone), shared-model-isolated-data (typical SaaS sweet spot), and dedicated-infrastructure (expensive, regulatory-grade). Decision framework, cost / risk trade-offs, and the migration paths between them as a tenant's requirements escalate.
- **per-tenant-vector-namespacing.md** *(coming)* — The patterns for tenant separation in vector stores: per-tenant namespace (Pinecone, Weaviate), per-tenant index (operationally heavier), shared index with mandatory metadata filter (cheap but high-blast-radius if the filter is missed). The testing pattern that catches missing-filter bugs before they become incidents.
- **per-tenant-prompt-and-context.md** *(coming)* — Tenant identity in the prompt, per-tenant system-prompt overlays, per-tenant tool authorization, and the design that prevents one tenant's retrieved context from being formatted into another tenant's response. Includes the "tenant ID as first-class context field" discipline.
- **[cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md)** — The leakage paths and the controls for each: retrieval misconfiguration (mandatory filters, deny-by-default), prompt injection exfiltration (input filtering, output sanitization, the sibling AI-security repo's controls), conversation-state leakage (per-tenant store isolation), cache pollution (tenant-aware cache keys), and the audit-log pattern that lets a customer's compliance team verify isolation.
- **[noisy-neighbor-mitigation.md](./noisy-neighbor-mitigation.md)** — Per-tenant rate limits, per-tenant cost caps, queue fairness, the priority-tier pattern for premium tenants, and the architecture that prevents one tenant's traffic spike (or runaway agent) from degrading service for others. Includes the per-tenant model-tier routing pattern (premium tenants get larger model by default).
- **per-tenant-fine-tuning.md** *(coming)* — When per-tenant fine-tuning is the right pattern (very few cases), when adapters / LoRAs make it cheaper, and the operational cost (per-tenant model lifecycle, per-tenant eval, per-tenant rollback) that usually steers regulated workloads back toward strong RAG + shared base model.
- **data-residency-patterns.md** *(coming)* — Region-pinning at the infrastructure layer (provider region, model deployment region, data store region), the per-region model-availability matrix that constrains the design, and the architecture for tenants whose data may not cross specific borders. Includes the GovCloud / sovereign-cloud variant and the FedRAMP boundary pattern.

## How to use this section

**If you are designing a multi-tenant AI platform from scratch**, read `isolation-models.md` first. The model you commit to constrains everything else — and graduating from shared-everything to dedicated-infrastructure later is much harder than designing for the right level up front.

**If you have a multi-tenant AI feature in production and are not certain about isolation**, `cross-tenant-leakage-prevention.md` is a self-audit. Walk through every leakage path; verify each control.

**If a customer is asking for data residency or dedicated infrastructure**, `data-residency-patterns.md` is the menu. Most customer requirements map to one of three or four patterns; the document names them.

## What this section is not

- **A SOC 2 / HIPAA control mapping.** Compliance mapping for the patterns here is in the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture). This folder is about the architecture; the controls and the evidence are elsewhere.
- **A multi-tenant database design guide.** General multi-tenant DB patterns are well-documented; this folder is only about the AI-specific overlays on top.
