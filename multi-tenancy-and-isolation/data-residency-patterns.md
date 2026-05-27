# Data Residency Patterns

> **Audience.** Architects whose AI platform has a customer asking "where exactly is our data?" Tech leads whose roadmap suddenly includes EU expansion, government accounts, or healthcare regulated for a specific jurisdiction. Anyone discovering that the model they planned to use isn't deployed in the region the customer requires. **Scope.** The *architectural* decisions: the dimensions of residency (data at rest, in transit, model deployment region, retrieval region, audit log region, embedding region); the per-region model-availability matrix that constrains design; region-pinning architecture; cross-border restriction patterns (GDPR + Schrems II, China, India, Russia); GovCloud / sovereign-cloud variants; FedRAMP boundary pattern; the "what if the model isn't available in this region" problem. Not the broader isolation models (see [isolation-models.md](./isolation-models.md)). Not the per-tenant infrastructure choice (this doc applies across tenants). Not the security-side compliance crosswalk (see the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Data residency in AI systems is harder than in conventional cloud systems for one specific reason: models are deployed in specific regions, and not all models are available in all regions. A standard SaaS architecture can be region-pinned by deploying compute and storage in the customer's required region; an AI architecture can be region-pinned only if the model is also there.

In 2026, this constraint is binding for most production AI work:

- Frontier hosted models are typically deployed in a small number of regions per provider. Anthropic, OpenAI, and Google each support a subset of AWS / GCP / Azure regions with their flagship models.
- Self-hosted models can run anywhere you have GPUs, but the operational burden and the cost differential make them a fallback for residency, not the default.
- Multi-region deployment of the same model is often available but with cost or feature trade-offs.
- New model releases stagger across regions. A model that's available in `us-east-1` may take weeks or months to reach `eu-west-1`.

The customer's residency requirement isn't always about the model — it's often about the data:

- "Our data cannot leave the EU." — GDPR + Schrems II posture.
- "Our data cannot leave the US." — federal contracts, FedRAMP, ITAR.
- "Our data must stay in Canada." — provincial privacy law (PIPEDA + PHIPA in Ontario, HIA in Alberta).
- "Our data must stay in the customer's specified region." — enterprise contracts.
- "Our data must be processed by a model that itself runs in our region." — sovereign-cloud / FedRAMP boundary requirement.

These requirements interact with the model-availability matrix. A customer needs "data + model in `ca-central-1`" but Anthropic doesn't (yet) deploy there. The architecture either uses a different model in `ca-central-1`, uses a self-hosted model in `ca-central-1`, or negotiates with the customer to use `us-east-1` with cross-border BAA / DPA terms.

This document covers the architectural decisions: the dimensions where residency matters; the per-region availability matrix as a binding constraint; the region-pinning patterns; cross-border restrictions and how to respect them; sovereign / regulated cloud patterns; the fallback when the model isn't available in the required region.

This document is opinionated about four things:

1. **Residency is multi-dimensional. "Data is in `eu-west-1`" is not the answer; "data at rest is in `eu-west-1`, model serving the data is in `eu-west-1`, retrieval index is in `eu-west-1`, audit log is in `eu-west-1`, embedding generation is in `eu-west-1`" is the answer.** Anything less leaks across boundaries.
2. **The model-availability matrix is the binding constraint, and it changes.** A residency architecture that's correct today may be unworkable tomorrow when the provider adds or removes a model in a region. Plan for the matrix to evolve; don't hardcode model-region pairs.
3. **Cross-border restriction is a legal posture, not a technical default.** GDPR + Schrems II created a regime where US-hosted model serving EU data is contentious; the architectural responses (EU-resident model, EU-resident provider, contractual terms) are all legitimate; the choice depends on legal counsel and customer requirements, not on engineering preference.
4. **Sovereign / regulated cloud (GovCloud, FedRAMP High, sovereign clouds like Bleu in France) is a different commit.** The boundary is wider than "region pinning"; it includes operators with US citizenship, isolated networks, separate update channels, and dedicated provider commitments. Treat it as a distinct architecture, not a configuration switch.

Structure: (2) the dimensions of residency; (3) the per-region model-availability matrix; (4) region-pinning architecture; (5) cross-border restriction patterns; (6) GovCloud and sovereign-cloud variants; (7) the "model not available in region" problem; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The dimensions of residency

Residency isn't a single property; it's a checklist across dimensions. Every dimension that isn't pinned is a potential leak.

### 2.1 Data at rest

The persistent storage of customer data — databases, blob stores, vector stores, file systems.

**Region-pinning mechanism.** Provision storage in the required region; configure replication to stay within the region (or stay disabled for cross-region replication).

**Pitfall.** Auto-replication enabled by default in some services. S3 cross-region replication, RDS multi-region read replicas, Azure geo-redundant storage — all can move data across regions unless explicitly configured otherwise.

### 2.2 Data in transit

The network paths data takes between components.

**Region-pinning mechanism.** All inter-service calls stay within the region; egress to external services (hosted model providers, third-party APIs) is over private network (VPC peering, PrivateLink, AWS Direct Connect, Azure ExpressRoute, GCP Interconnect) or controlled egress with audit.

**Pitfall.** CDN egress. A CDN configured for global edge caching may cache and serve content from outside the residency region. For residency-sensitive responses, CDN must be region-scoped or bypassed.

**Pitfall.** Cross-region service dependencies. A service in `us-east-1` calling a shared service in `us-west-2` for some operation crosses regions. The dependency may be invisible (a third-party SDK that calls home).

### 2.3 Model deployment region

The region where the model itself runs.

**Region-pinning mechanism.** Use a hosted provider's regional deployment (Anthropic's region selector, OpenAI's regional models, AWS Bedrock's region, GCP Vertex's region, Azure OpenAI's region) that matches the data region. For self-hosted, deploy the model in the required region.

**Pitfall.** "The provider says they're in the region" is sometimes regional API endpoint with cross-region backend. Verify the model inference itself is in the region; some providers' marketing implies region-pinning that the underlying architecture doesn't fully deliver. Request architecture documentation; verify with the provider's compliance team.

**Pitfall.** Provider's model deployment evolves. The model in your region today may be migrated, deprecated, or temporarily routed elsewhere during incidents. Customer contracts must allow for provider-side maintenance.

### 2.4 Retrieval index region

The vector store, search index, or other retrieval infrastructure.

**Region-pinning mechanism.** Per-region vector store deployment. Pinecone, Weaviate, OpenSearch, pgvector — each must be deployed in the required region.

**Pitfall.** Cross-region replication for disaster recovery. Replication targets must be within the same residency boundary or replication must be disabled. DR plans need region-aware design.

**Pitfall.** The vector store provider's managed service has its own region story. Pinecone's serverless tier had cross-region implications in early versions; the dedicated-region tier is the right choice for residency.

### 2.5 Embedding generation region

The model that generates embeddings (often a smaller model than the chat / generation model). Often forgotten because it's lower-profile than the main model.

**Region-pinning mechanism.** Embedding model runs in the residency region. Use the same provider's regional embedding model, or self-host.

**Pitfall.** Embedding models are sometimes shared infrastructure (cross-region) for efficiency. Verify the embedding generation actually happens in the required region for each call.

### 2.6 Audit log region

The logs that record system access, AI calls, user actions, data flows. Often contains PII or other sensitive content (or pointers to it).

**Region-pinning mechanism.** Audit log destination in the residency region. Cross-region aggregation for global observability must filter PII or stay within the residency boundary.

**Pitfall.** Centralized observability tools (Datadog, Splunk, New Relic) often aggregate across regions by default. Configure regional collection; data residency clauses with vendors.

### 2.7 Conversation history / state region

For agents and chat features, the conversation history is persistent; it's data at rest with conversation-specific sensitivity.

**Region-pinning mechanism.** Per-region conversation store; conversation IDs are unique per region (or globally unique but with region tagging).

**Pitfall.** Cross-region conversation portability. If a user moves between regions, their conversation history may need to follow — but residency constraints may forbid the data move. Decision: separate conversations per region, no portability.

### 2.8 The dimensions matrix

| Dimension | Common region-pinning mechanism | Common pitfall |
| --- | --- | --- |
| Data at rest | Provision in region; disable cross-region replication | Auto-replication on by default |
| Data in transit | Stay within VPC; controlled egress | CDN edge caching crosses regions |
| Model deployment | Hosted provider's regional model; self-host | API regional but backend not |
| Retrieval index | Per-region vector store | DR replication crosses boundary |
| Embedding generation | Regional embedding model | Shared cross-region infrastructure |
| Audit log | Regional log destination | Centralized observability aggregates |
| Conversation history | Per-region conversation store | Cross-region user portability |

A complete residency architecture pins all seven. Anything less is a partial residency posture; document the gaps explicitly with legal counsel.

---

## 3. The per-region model-availability matrix

The matrix is your binding constraint. Build it before designing the residency architecture.

### 3.1 What the matrix looks like

```
                    us-east-1  us-west-2  eu-west-1  eu-central-1  ap-northeast-1  ca-central-1
Claude Opus 4.7        Y          Y          Y           Y               -              -
Claude Sonnet 4.6      Y          Y          Y           Y               Y              -
Claude Haiku 4.5       Y          Y          Y           Y               Y              Y
GPT-4o                 Y          Y          Y           Y               Y              -
GPT-4o-mini            Y          Y          Y           Y               Y              Y
Gemini 2.5 Pro         Y          Y          Y           Y               Y              -
Llama 3 70B (self)     Y          Y          Y           Y               Y              Y
Llama 3 8B (self)      Y          Y          Y           Y               Y              Y
```

The columns are regions; the rows are models. The cells are "available" / "not available." This is the matrix you start with; it changes monthly.

Build the matrix for your specific providers and models; verify with the provider; update at least quarterly.

### 3.2 The matrix's effect on design

When designing for a residency requirement:

- **Pick the residency region.** Customer-driven.
- **Look up which models are available in that region.** From the matrix.
- **Select the model from the available set.** If your preferred model isn't available, you have three options: pick an available model, self-host, or negotiate with the customer to use a different region.

The model selection follows the region, not the other way around. Architects who pick the model first and then try to place the data discover that residency was a fiction.

### 3.3 The matrix changes; the design must accommodate

A model that's available in `eu-central-1` today may not be tomorrow. A model that's not available today may launch next quarter. Designs that hardcode model-region pairs accumulate technical debt as the matrix evolves.

**Pattern.** Configuration-driven model selection per region. The configuration is updated when the matrix changes; code doesn't change.

**Pattern.** Model fallback per region. If the primary model is unavailable in the region (matrix update or provider incident), fall back to the next-best available. The fallback set is per-region configuration.

### 3.4 The provider's region-deprecation cadence

Providers sometimes remove model availability from a region. Anthropic, OpenAI, AWS Bedrock have all done this. The notice period varies; the migration burden falls on you.

**Pattern.** Track region-deprecation announcements. Subscribe to provider release notes; alert when a region+model combo you depend on is announced for deprecation.

**Pattern.** Multi-model insurance. Have a fallback model in the region that you've tested for the workload. When the primary is deprecated, the fallback is already validated.

### 3.5 Self-hosting as the residency escape valve

When the matrix doesn't have a model in your required region, self-hosting an open-weight model is the escape valve. Llama, Mistral, Qwen — all available to deploy in any region with GPU capacity.

**Pro.** Region-independent. You control the deployment region.

**Con.** Operational burden. GPU infrastructure to operate, model lifecycle, latency optimization, eval suite.

**When justified.** When residency is mandatory and no hosted model meets the requirement. When the customer's contract value justifies the operational cost.

**When not justified.** When a hosted model meets residency requirements via cross-border agreement (BAA, DPA, EU Standard Contractual Clauses). Self-hosting is the harder path; only take it when easier paths are exhausted.

---

## 4. Region-pinning architecture

The infrastructure pattern for region-pinned AI deployment.

### 4.1 The deployment topology

Per-region deployment of the AI feature:

```
Customer in EU                            Customer in US
    ↓                                          ↓
EU API gateway (eu-west-1)             US API gateway (us-east-1)
    ↓                                          ↓
EU service deployment                  US service deployment
    ↓                                          ↓
EU vector store (Pinecone EU)         US vector store (Pinecone US)
    ↓                                          ↓
EU model (Claude Sonnet, EU)          US model (Claude Sonnet, US)
    ↓                                          ↓
EU audit log                          US audit log
```

Each region is a self-contained deployment. The application code is the same; the configuration differs (region, model endpoints, vector store endpoints, audit log destinations).

### 4.2 The routing layer

Customer requests must reach the correct regional deployment.

**Pattern.** Customer's account is tagged with their residency region; the API gateway routes based on the tag. The tag is set at customer onboarding and doesn't change without explicit migration.

**Pattern.** Per-region API endpoints (e.g., `eu.api.example.com`, `us.api.example.com`). The customer's SDK is configured with the regional endpoint at onboarding.

**Pitfall.** A global API endpoint that routes to the customer's region behind the scenes. The DNS / routing layer must be in the customer's region or in a global tier that's been compliance-vetted (e.g., AWS Global Accelerator with regional endpoints).

### 4.3 Cross-region operational concerns

Some operational tasks need cross-region visibility:

- Global observability and SLO tracking.
- Customer support routing (a support engineer in any region helping any customer).
- Platform team's deployment pipeline (pushing to all regions).
- Cross-region tenant management (customer that has accounts in multiple regions).

**Pattern.** Operational data (metrics, logs aggregated for SLO, support tickets) flows from regional deployments to a central observability tier. PII is filtered before leaving the region; only metrics and metadata cross.

**Pattern.** Customer support has region-scoped access. A support engineer in the US can only see US-region customer data unless granted cross-region access for a specific case (audited).

**Pattern.** Deployment pipeline is region-aware. Each region deploys independently; rolling deployments per region; canary per region.

### 4.4 The shared-control-plane separated-data-plane pattern

A common variant: the control plane (configuration, user identity, billing) is global; the data plane (customer data, model serving, retrieval) is regional.

**Pro.** Operational simplicity for control-plane operations.

**Con.** Some data crosses regions (user identity, account info, configuration). May or may not meet residency requirements depending on jurisdiction.

**When right.** When residency requirements allow control-plane sharing (e.g., "customer data stays in region" but not "every byte associated with the account stays in region").

**When wrong.** Strict residency requirements where no associated data can cross.

### 4.5 The latency consequence

Regional deployment means data and model are co-located, but customers in adjacent regions may experience higher latency than they would with a closer endpoint.

**Mitigation.** Regional deployment matched to customer geography (EU customers in EU region; US customers in US region; APAC customers in APAC region).

**Mitigation.** Latency-aware monitoring per region. Customers in regions distant from any deployment may have unacceptable latency; expand regional footprint or accept the latency.

### 4.6 The cost consequence

Regional deployment multiplies infrastructure cost. Three regions ≈ 3x the deployment cost. Even with shared control plane, the multiplier is significant.

**Mitigation.** Regional deployment only for residency-requiring customers; non-residency customers on the most cost-effective region.

**Mitigation.** Per-region capacity sized to actual regional demand. Don't over-provision EU when EU traffic is small.

---

## 5. Cross-border restriction patterns

Specific jurisdictions impose specific restrictions on AI data flows. Each has architectural implications.

### 5.1 GDPR + Schrems II (EU)

GDPR restricts personal data transfer outside the EU; Schrems II invalidated Privacy Shield, leaving Standard Contractual Clauses (SCCs) as the primary mechanism.

**Architectural responses.**

- **EU-resident deployment.** All processing in EU regions; data never leaves EU. Strongest posture.
- **SCC-based US deployment.** Data resides in US; SCCs cover the transfer; technical safeguards (encryption, key residency in EU) supplement. Acceptable for many customers; some refuse.
- **Hybrid.** Sensitive data in EU; non-sensitive in US. Operationally complex.

**Model provider implications.**

- Anthropic, OpenAI, Google all have EU regions; using EU-resident model is the cleanest answer.
- Bedrock has EU regions for Claude; Vertex has EU for Gemini; Azure OpenAI has EU.

**The bigger picture.** Some EU customers explicitly require "EU-resident model" because the SCC route is contractually undesirable. The architecture should support this without redesign.

### 5.2 Schrems II - the EU-US Data Privacy Framework (DPF)

The 2023 DPF provides a new transfer mechanism for US-based providers that self-certify. Some providers (including major hosted-AI providers) are DPF-certified, restoring the legal basis for EU→US transfer for those providers.

**Architectural implication.** Using a DPF-certified provider may allow US-region deployment for EU customer data, depending on customer contract terms.

**Pitfall.** Customer-specific contract terms often supersede DPF — the customer may have negotiated stricter requirements with their own legal counsel.

### 5.3 China (PIPL + DSL + CSL)

China's data laws are among the strictest globally. Personal Information Protection Law (PIPL), Data Security Law (DSL), Cybersecurity Law (CSL) jointly require:

- Personal data of Chinese citizens generally stays in China.
- Critical Information Infrastructure operators have additional restrictions.
- Cross-border transfer requires security assessment by CAC.
- "Important data" categorization adds further constraints.

**Architectural response.** China-region deployment is essentially mandatory for serving Chinese customers at scale. Often via a Chinese partner (Alibaba Cloud, Tencent Cloud, Huawei Cloud) due to ICP licensing and the operational complexity of cross-border operations.

**Model implications.** Frontier Western models are not generally available in mainland China. Workloads serving Chinese users typically use:
- Chinese frontier models (Qwen, Baichuan, GLM, DeepSeek).
- Self-hosted open-weight models.
- Specialized regulated-compliant deployments.

### 5.4 India (DPDP Act)

India's Digital Personal Data Protection Act (2023) restricts cross-border transfer based on government notification of restricted countries. The default is broader than China's; "data localization for sensitive personal data" applies.

**Architectural response.** India-region deployment for Indian customer data is the common posture for sensitive workloads. India regions are available from AWS, Azure, GCP; model availability is partial (varies by model).

### 5.5 Russia (Federal Law 242-FZ)

Personal data of Russian citizens must be stored on servers physically in Russia. Cross-border transfer is restricted.

**Architectural response.** Russia-region deployment, often via local partners due to sanctions and operational complexity. Western frontier models are generally not viable; self-hosted or local-provider models are the alternatives. Many Western SaaS providers have exited or paused operations.

### 5.6 Federal / FedRAMP (US)

US federal agencies require FedRAMP-authorized cloud services. FedRAMP Moderate is broadly available; FedRAMP High is for higher-sensitivity workloads; IL5/IL6 is for DoD classifications.

**Architectural response.** FedRAMP-authorized environments (AWS GovCloud, Azure Government, GCP Assured Workloads). Model availability is narrower than commercial regions; cleared personnel requirements; ATO process per system.

Detail in §6.

### 5.7 Province-level requirements (Canada, Australia)

Some Canadian provinces (Quebec under Law 25, Ontario under PHIPA for health) have province-specific residency expectations. Australia has APP-aware requirements at state level for some categories.

**Architectural response.** Region-pinning to the closest available region; explicit BAA / DPA terms; sometimes self-hosting if provincial requirements are strict.

### 5.8 The legal-tech-product triangle

Cross-border restrictions are legal posture first, architecture second. The architectural response must serve the legal interpretation, which is customer-specific. The pattern:

- Customer's legal team specifies the requirement.
- Your legal team interprets the requirement.
- Architecture proposes an implementation that meets the interpretation.
- Customer's compliance team verifies.
- Implementation proceeds.

Architects who try to interpret the legal requirements themselves produce architectures that don't satisfy the customer. Legal counsel is non-optional.

---

## 6. GovCloud and sovereign-cloud variants

Regulated cloud environments are a wider commitment than region-pinning.

### 6.1 What's different

GovCloud (AWS), Azure Government, GCP Assured Workloads, sovereign clouds like Bleu (France), Delos (Germany), and others provide:

- **Personnel restrictions.** Only cleared personnel (e.g., US persons for AWS GovCloud) can access the environment.
- **Network isolation.** Separate VPN endpoints; sometimes air-gapped; sometimes dedicated private connectivity.
- **Separate update channels.** Software updates are validated and pushed through controlled channels.
- **Dedicated compliance posture.** FedRAMP, IL5/IL6, SOC 2 Type II with regulated controls, specific ISO certifications.
- **Often higher cost.** Pricing premium of 20-100% over commercial regions.
- **Often narrower model availability.** Provider's regulated environment supports fewer models than commercial.

### 6.2 The boundary

The FedRAMP / IL5 boundary defines what's in scope for the certification. Things in the boundary are subject to the controls; things outside are not.

**Pattern.** Everything that touches customer data, customer identity, model serving, or audit logging is in the boundary. Things outside (corporate IT, dev environments, marketing systems) are not.

**Pattern.** The boundary is documented in the System Security Plan (SSP). Changes to the boundary require re-authorization in many cases.

### 6.3 Model availability in regulated environments

A typical 2026 picture for FedRAMP High:

- AWS Bedrock in GovCloud: subset of models available (Claude in GovCloud as of 2024-2025; ongoing expansion).
- Azure OpenAI in Azure Government: subset of GPT models.
- GCP Vertex in Assured Workloads: subset of Gemini.
- Self-hosted Llama / Mistral on FedRAMP-authorized infrastructure: viable but operationally heavy.

The model selection in regulated environments is constrained; design accordingly.

### 6.4 The operational difference

Operating in a regulated environment is different from operating in commercial:

- Personnel access is gated; on-call rotations require cleared personnel.
- Change management is heavier; emergency changes have explicit procedures.
- Deployment pipeline integrates with continuous-monitoring requirements.
- Audit log retention requirements (often years) drive storage architecture.
- Incident response has notification requirements to regulators.

These differences are organizational as well as architectural. A team taking on regulated cloud deployment usually needs dedicated capacity for the operational overhead.

### 6.5 The "regulated cloud as a separate platform" decision

Some organizations run their commercial and regulated platforms as separate codebases. Some maintain a single codebase with regulated-specific configuration.

**Single codebase.** Easier maintenance; risk that commercial features inadvertently work their way into regulated environments without scrutiny.

**Separate codebases.** Stricter isolation; higher maintenance cost; harder to keep features in sync.

**Hybrid.** Single codebase; regulated-specific config flags; rigorous CI to ensure regulated environment respects the flags.

For most teams, the hybrid is the right balance; the strict separation is overhead unless the regulated environment is genuinely different.

---

## 7. The "model not available in region" problem

The hardest residency problem: the customer's required region doesn't have your preferred model. Options exist; each has trade-offs.

### 7.1 Option A: use a different model that's available in the region

Find a model in the matrix that's available in the required region; verify it meets the workload's quality and capability bar.

**Pro.** Stays within the residency boundary; no cross-border data flow.

**Con.** May be a less capable or older model; quality regression possible.

**Test.** Run the workload's eval suite against the available model. If the quality bar is met (or close enough that prompt engineering can close the gap), this is the right answer.

### 7.2 Option B: self-host an open-weight model in the region

Deploy Llama, Mistral, Qwen, or similar in the required region. You control the deployment region completely.

**Pro.** Full residency control; no provider region dependency.

**Con.** Operational burden; GPU infrastructure; eval suite; lifecycle.

**Test.** Does the open-weight model meet the workload's quality bar (with reasonable fine-tuning or prompting)? If yes, viable. If no, look at Option C.

### 7.3 Option C: cross-border with contractual safeguards

Use a hosted model in a different region; cover the cross-border flow with contractual instruments (SCCs, BAA, DPA, customer-specific addendum).

**Pro.** Best model quality; lower operational overhead.

**Con.** Cross-border flow may be unacceptable to the customer; even with contracts, some customers refuse.

**Test.** Has the customer agreed to the cross-border arrangement in writing? If yes, viable. If no, return to Option A or B.

### 7.4 Option D: wait for the provider to deploy in the region

The provider's roadmap may add the required region. If the customer can wait, this is the least-impact option.

**Pro.** Eventually delivers the customer's preferred experience.

**Con.** Customer may not be willing to wait; competitors may close the gap.

**Test.** Has the provider committed to a specific region launch date that the customer accepts?

### 7.5 The fallback ladder for region-availability

The decision tree:

```
Is preferred model in customer's required region?
  Yes → Use it.
  No → Is there an acceptable alternative model in the region?
    Yes → Use the alternative; document the choice.
    No → Can a self-hosted open-weight model meet requirements?
      Yes → Self-host in the region.
      No → Is cross-border acceptable with contractual safeguards?
        Yes → Cross-border deployment with documented controls.
        No → Decline the deployment; renegotiate scope with customer.
```

### 7.6 The "multi-region per customer" complication

Some customers have multi-region presence (a US-based company with EU subsidiaries; a global enterprise with regional data sovereignty requirements). The deployment must serve them across regions while respecting per-region residency.

**Pattern.** Customer's tenant has region-scoped subsets. Each subset has its own residency posture; data doesn't flow between subsets without explicit cross-border authorization.

**Pattern.** Customer's identity system is global; their data is regional. Authentication crosses regions; data does not.

This is operationally heavy; rarely worth building unless multiple enterprise customers require it.

---

## 8. Worked Meridian example

Meridian Health's data residency journey illustrates the patterns and the trade-offs.

### 8.1 The starting posture

Initial deployment: US-only operation. All data, all processing, all model serving in `us-east-1`. Customer base: US healthcare systems. No residency constraints beyond HIPAA (which is US-federal).

Anthropic Claude was the primary model; Anthropic's `us-east-1` deployment with BAA covered HIPAA requirements.

### 8.2 The Canadian acquisition

Mid-2025: Meridian acquires a Canadian regional health system (Atlantic Maple Health). Atlantic Maple has provincial obligations:

- PIPEDA (federal Canadian privacy law).
- PHIPA (Ontario; some patients have Ontario residency).
- Atlantic-province-specific (PHIA in Nova Scotia, PHIPA NB in New Brunswick).

The province-level rules require that PHI for residents of those provinces is stored in Canada when reasonably possible. The provincial regulators have been increasingly insistent on "in Canada" for new systems.

### 8.3 The architecture decision

The team's options:

- **Option A:** keep all data in US; rely on cross-border transfer agreements. Provincial regulators were tolerating this but signaled it might not last. Customer's compliance team was nervous.
- **Option B:** deploy in `ca-central-1` (AWS Canada). But the model availability matrix at the time: Claude was not in `ca-central-1`. Available models: GPT-4o (Azure / OpenAI), Gemini (Vertex), Cohere (their native region).
- **Option C:** self-host Llama 3 70B in `ca-central-1`. Operationally heavy but full control.
- **Option D:** wait for Anthropic to deploy in Canada. Not on roadmap that anyone could confirm.

### 8.4 The path taken

The team chose a phased approach:

**Phase 1 (immediate, Q3 2025):** Atlantic Maple's data in `ca-central-1` (vector store, conversation history, audit log). Model serving still in `us-east-1` via Anthropic with explicit cross-border consent in Atlantic Maple's customer agreement. Documented the cross-border arrangement to provincial regulators.

**Phase 2 (Q1 2026):** Evaluate Cohere Command R+ on the Care Coordinator workload. Cohere is Canadian, available in `ca-central-1`, and the workload's eval suite passed at 92% (vs Claude's 95%). The 3% gap was acceptable with additional prompt engineering bringing it to 94%.

**Phase 3 (Q2 2026):** Migrate Atlantic Maple's primary model from Anthropic-US to Cohere-Canada. Cross-border model serving eliminated. Provincial regulators satisfied. Slightly lower quality accepted by Atlantic Maple's clinical team after side-by-side testing.

### 8.5 The architecture as deployed

```
US customers                              Atlantic Maple (Canadian customer)
    ↓                                          ↓
us-api.meridian.example.com         ca-api.meridian.example.com
    ↓                                          ↓
US deployment (us-east-1)            Canadian deployment (ca-central-1)
    ↓                                          ↓
US vector store (Pinecone US)        Canadian vector store (Pinecone CA)
    ↓                                          ↓
Anthropic Claude (us-east-1)         Cohere Command R+ (ca-central-1)
    ↓                                          ↓
US audit log (CloudWatch us-east-1)  Canadian audit log (CloudWatch ca-central-1)
```

Shared control plane: customer identity, billing, configuration. Documented as cross-border in customer agreements.

Separate data plane: customer data, model serving, retrieval, audit. All in respective regions.

### 8.6 The "EU customer" scenario (hypothetical, planned)

The team has a sales pipeline for European health systems. The planned architecture:

- `eu-central-1` deployment matching the Canadian pattern.
- Model selection: Anthropic Claude in EU (matrix has it; verified).
- EU-resident vector store, audit log, conversation history.
- EU data plane with documented SCC for shared control plane.
- BAA and DPA terms specific to EU customers' GDPR requirements.

The pattern is the same as Canadian; only the model selection and the contractual instruments differ.

### 8.7 The "what if Anthropic adds Canada" upside

If Anthropic deploys Claude in `ca-central-1`, the team can migrate Atlantic Maple from Cohere to Anthropic-CA. Migration playbook:

- Stage Claude-CA deployment.
- Run shadow traffic; verify quality.
- Canary 10% of Atlantic Maple's traffic.
- Promote to 100% over 2-4 weeks.
- Decommission Cohere-CA path.

The architecture's flexibility (configuration-driven model selection per region) makes this a relatively low-risk migration. The eval suite is in place; the regional deployment is the same.

### 8.8 What the architecture costs

- Operational: ~$8k/month additional infrastructure for the Canadian deployment (small relative to US-only).
- Engineering: ~3 months for the Canadian deployment + Cohere integration.
- Ongoing: per-region operational overhead distributed across the platform team.
- Customer-facing: Atlantic Maple's contracts now reference Canadian residency; customer's compliance team verified satisfactory.

### 8.9 What the architecture avoids

- Regulatory escalation from provincial authorities.
- The need to migrate later under regulatory pressure (more expensive than building in).
- Customer churn risk (Atlantic Maple had explicitly raised the question during acquisition).

---

## 9. Anti-patterns

### 9.1 The "region-pinned because the storage is in the region" claim

**Pattern.** Data at rest is in the required region; data in transit goes via US-based services; model serving is in US. Marketing claims "region-pinned"; technical reality is partial.

**Corrective.** Region-pin all seven dimensions (§2). Document the gaps explicitly if any remain.

### 9.2 The hardcoded model-region pair

**Pattern.** Code says `MODEL = "claude-sonnet-4-5-eu"`. Region migration requires code change. Matrix updates produce technical debt.

**Corrective.** Configuration-driven model selection per region. Code is region-agnostic.

### 9.3 The cross-region CDN that caches PII

**Pattern.** API responses cached at global CDN edge. PII in response served from any edge worldwide; effective residency is "everywhere."

**Corrective.** CDN configured region-scoped; PII responses are cache-bypass; verify with cache audit.

### 9.4 The DR replication across residency boundary

**Pattern.** Disaster recovery replicates data to a different region for resilience. Replication target is outside the residency boundary.

**Corrective.** DR target within the residency boundary. Cross-boundary DR requires explicit customer agreement; usually means accepting longer RPO/RTO.

### 9.5 The provider region-deprecation surprise

**Pattern.** Provider announces deprecation of a region you depend on. Customer has 30 days; you have weeks.

**Corrective.** Subscribe to provider release notes; alert on region/model deprecations; maintain fallback model availability per region.

### 9.6 The "we'll add EU support when a customer asks" deferral

**Pattern.** No EU deployment. First EU customer arrives; sales promises a delivery timeline; engineering discovers it's 6 months of work.

**Corrective.** Build region-pinning capability into the architecture before the first regulated customer. Adding regions is a configuration + deployment operation; the foundation must exist.

### 9.7 The shared control plane carrying PII

**Pattern.** Control plane is global; it carries customer identity, account info, sometimes user metadata. Some of that metadata is PII in some jurisdictions.

**Corrective.** Audit what's in the control plane. PII either filtered out or moved to regional data planes. Cross-border control plane data minimized.

### 9.8 The audit log that aggregates cross-region

**Pattern.** Datadog (or equivalent) collects logs from all regions to a US-based account. Customer's logs cross border without their knowledge.

**Corrective.** Regional log destinations; central observability fed by sanitized aggregates; raw logs stay in region.

---

## 10. Findings (sprint-assignable)

**ARCH-DRP-001 (P0). Residency is single-dimension (data-at-rest only).** Model serving, retrieval, audit cross the boundary. Audit all seven dimensions (§2); region-pin each; document any gaps with legal counsel. Owner: AI platform + security.

**ARCH-DRP-002 (P0). No per-region model-availability matrix maintained.** Region planning happens without binding constraint awareness. Build matrix for your providers and models; verify with providers; update quarterly. Owner: AI platform + product.

**ARCH-DRP-003 (P0). Hardcoded model-region pairs.** Matrix changes produce technical debt; deprecations are emergencies. Configuration-driven model selection per region; code is region-agnostic. Owner: AI platform.

**ARCH-DRP-004 (P0). Cross-region data replication enabled by default.** S3 / RDS / vector-store auto-replication crosses residency boundaries. Audit replication; explicitly configure within-region or disable cross-region. Owner: platform + security.

**ARCH-DRP-005 (P1). No CDN configuration for residency-sensitive responses.** PII cached at global edges. Configure CDN region-scoped or bypass for PII; audit cache content. Owner: platform.

**ARCH-DRP-006 (P1). No subscription to provider region-deprecation announcements.** Deprecations are surprises. Subscribe to provider release notes; alert on region/model deprecations affecting your deployments. Owner: AI platform.

**ARCH-DRP-007 (P1). No fallback model defined per region.** Provider incident in region means workload outage. Define per-region fallback (acceptable alternative model); validate via eval suite. Owner: AI platform.

**ARCH-DRP-008 (P1). Centralized observability tools aggregate cross-region without PII filtering.** Logs containing PII land in cross-region account. Regional log destinations; central tier receives sanitized aggregates only. Owner: SRE + security.

**ARCH-DRP-009 (P1). DR replication crosses residency boundary.** DR plan violates residency. Move DR target within boundary; accept longer RPO/RTO if necessary; document. Owner: SRE + security.

**ARCH-DRP-010 (P1). Customer's region not tagged in routing layer.** Routing depends on inferred region. Customer's residency region tagged at onboarding; routing layer respects tag. Owner: API team + customer success.

**ARCH-DRP-011 (P2). Self-hosting capability not maintained for residency fallback.** When matrix doesn't have model in region, self-hosting is the escape valve — but team has no GPU ops capacity. Maintain at least one self-host-ready pipeline; have GPU operations capability. Owner: AI platform.

**ARCH-DRP-012 (P2). Cross-region operational concerns (support, deployment) not designed.** Support engineer can't help cross-region customer; deployment pipeline doesn't handle regions. Design cross-region operational access (with audit); region-aware deployment. Owner: SRE + platform.

**ARCH-DRP-013 (P2). Embedding generation region not aligned with retrieval region.** Embeddings generated by cross-region embedding model; data crosses boundary for embedding. Verify embedding model is in region; use same provider's regional embedding service. Owner: AI platform.

**ARCH-DRP-014 (P2). Conversation history portability across regions undefined.** User moving regions either loses history or violates residency. Define policy: separate conversations per region (default) or explicit migration. Owner: AI platform + product.

**ARCH-DRP-015 (P2). FedRAMP / regulated cloud boundary not documented.** First federal customer is improvised. Document SSP / boundary if pursuing regulated cloud; engage compliance from day one. Owner: compliance + AI platform.

**ARCH-DRP-016 (P3). No legal review for cross-border contractual instruments.** SCC / BAA / DPA terms drafted without specific AI workflow review. Legal review of contractual instruments for AI-specific implications. Owner: legal + AI platform.

**ARCH-DRP-017 (P3). No per-customer residency assertion in customer onboarding.** Residency requirements gathered ad-hoc. Customer onboarding includes explicit residency requirement; routed to correct region from day one. Owner: customer success + AI platform.

**ARCH-DRP-018 (P3). Region-deprecation runbook not prepared.** First deprecation is improvised crisis. Document deprecation runbook: alternative model evaluation, customer communication, migration playbook. Owner: AI platform + customer success.

---

## 11. Adoption sequencing checklist

For a team adopting data residency capability, in order:

- [ ] **Build per-region model-availability matrix (§3).** Verify with each provider; document.
- [ ] **Audit current deployment against the seven dimensions of residency (§2).** Document where each dimension is region-pinned and where it isn't.
- [ ] **Refactor model selection to be configuration-driven per region.** Eliminate hardcoded model-region pairs.
- [ ] **Audit cross-region data replication (storage, DR).** Configure within-region or disable cross-region.
- [ ] **Audit CDN, edge caching, and similar global services.** Region-scope or bypass for residency-sensitive responses.
- [ ] **Audit centralized observability (logs, metrics).** Regional destinations; sanitized aggregates to central tier.
- [ ] **Design per-region deployment topology (§4).** Per-region API endpoints; routing layer; per-region service deployment.
- [ ] **Tag customer accounts with residency region.** Routing layer uses the tag.
- [ ] **For each cross-border restriction relevant to your business (§5):** legal review; architectural response; contractual instruments.
- [ ] **Subscribe to provider region-deprecation announcements.** Alert on changes affecting your deployments.
- [ ] **Define per-region fallback models.** Validate via eval suite.
- [ ] **Maintain self-hosting capability as escape valve.** At least one self-host-ready pipeline; GPU ops capacity.
- [ ] **Plan FedRAMP / regulated cloud capability if government customers are on roadmap.** Engage compliance early; document boundary.
- [ ] **Build customer onboarding flow that captures residency requirement.** Routes to correct region from day one.
- [ ] **Pre-production test:** verify all seven dimensions are region-pinned for the test deployment.
- [ ] **Annual residency review.** Matrix update; deployment audit; customer attestations.

---

## 12. References

**In this folder.**
- [isolation-models.md](./isolation-models.md) — the three isolation models; data residency is the regulatory dimension of isolation choice.
- [per-tenant-prompt-and-context.md](./per-tenant-prompt-and-context.md) — per-tenant prompt patterns that compose with per-region deployment.
- [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) — vector store patterns; per-region vector store deployment.
- [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md) — broader leakage controls.
- [noisy-neighbor-mitigation.md](./noisy-neighbor-mitigation.md) — multi-tenant fairness on shared regional infrastructure.
- [per-tenant-fine-tuning.md](./per-tenant-fine-tuning.md) — per-tenant fine-tunes have per-region implications.

**Elsewhere in this repo.**
- [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](../model-strategy/frontier-vs-open-weights-vs-fine-tune.md) — model selection framework; residency narrows the available set.
- [model-strategy/model-routing-and-tiering.md](../model-strategy/model-routing-and-tiering.md) — routing patterns including per-region routing.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — self-hosting as residency escape valve.
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue tracks per-region availability.
- [integration-architecture/sync-vs-async-vs-streaming.md](../integration-architecture/sync-vs-async-vs-streaming.md) — integration patterns may have cross-region implications.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose Canadian expansion is described here.

**Sibling repos.**
- [ai-engineering-reference-architecture / model-lifecycle / model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md) — engineering-side registry tracking per-region availability.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — security and compliance crosswalk for regulated cloud.

**External.**
- AWS region documentation for model availability per region (Bedrock, SageMaker).
- Azure region documentation for Azure OpenAI availability.
- GCP region documentation for Vertex AI availability.
- Anthropic, OpenAI, Cohere region documentation.
- EU GDPR + Schrems II + DPF resources.
- Canadian PIPEDA + provincial laws (PHIPA, HIA, etc.).
- China PIPL / DSL / CSL.
- FedRAMP marketplace and authorization documentation.
- Sovereign cloud documentation (Bleu, Delos, etc.).
