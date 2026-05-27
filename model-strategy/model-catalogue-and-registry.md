# Model Catalogue and Registry

> **Audience.** Architects building the platform discipline around model usage. Tech leads tired of discovering which model is in production by reading a YAML file that nobody updated. Platform teams asked to govern AI adoption without slowing it. Anyone whose answer to "which models are in production right now" is "let me check with each team." **Scope.** The *architectural* decisions: the catalogue schema (what fields the registry tracks); the catalogue's role in adoption (new model evaluation, approval workflow); the catalogue's role in operational monitoring (per-model telemetry, deprecation tracking, usage analytics); the catalogue as source of truth for deployment configuration; governance (ownership, decision flow); tenant-aware and region-aware extensions. Not the engineering-side registry implementation (see sibling [model-lifecycle / model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md), which covers the operational mechanics). Not the broader model strategy (see [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md) for the foundational choice). Not the migration playbook (see [model-migration-playbook.md](./model-migration-playbook.md), planned). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The default state of model usage in growing organizations is sprawl. Each team picks a model when they ship a feature. The team's choice is recorded in their own config, sometimes in code, sometimes in a deployment YAML. Different teams pick different versions. Some teams pin versions; some use "latest." Some teams use a fine-tuned variant; some use stock. Some teams discovered a model that was deprecated six months ago and is still running in production because their tests pass.

By the time leadership asks "which models are we using and how much do they cost?", the answer requires interviewing every team, reading every config, and reconciling with the provider's invoice. The answer is usually wrong by the time it's finished.

The model catalogue (or registry) is the platform discipline that makes the answer correct and always available. Models are first-class dependencies, tracked with the same rigor as production databases, message brokers, and third-party services. Each model in production has an entry; each entry has owners, version pins, allowed contexts, compliance status, deprecation dates, and usage telemetry. Adding a new model goes through a review process; removing a model is a tracked decommissioning.

The catalogue isn't a list; it's an architecture. Done well, it:

- Lets platform say "yes" to model adoption requests without losing control of the production surface.
- Catches deprecation issues before they become outages.
- Surfaces cost outliers (the model that turned out to be 5x more expensive than expected).
- Provides the source of truth for security and compliance teams ("which models touch PHI? Which have BAA coverage?").
- Underpins the model migration playbook ("when we move from Sonnet 4.5 to 4.6, which features are affected?").

Done poorly (or not at all), the absence of a catalogue produces the standard set of incidents: deprecated models running silently, surprise bills, compliance violations, untracked fine-tuned variants, and no way to coordinate migrations.

This document covers the architectural decisions: what the catalogue tracks, how it integrates with adoption and operations, how it serves as the source of truth for deployment, who governs it, and how it accommodates tenant and region complexity.

This document is opinionated about four things:

1. **The catalogue is a first-class platform artifact, not a wiki page or a shared spreadsheet.** It must be queryable, versioned, integrated with deployment, and authoritative. The "shared doc" approach drifts within months.
2. **Every model in production must have a catalogue entry.** No exceptions for "small features" or "experimental code." If it makes an LLM call in production, it's in the catalogue. The catalogue's completeness is the foundation for every claim built on it.
3. **The catalogue is the source of truth for deployment configuration.** Code references the catalogue (by model ID + version); the catalogue resolves to the current deployment endpoint, model parameters, and access controls. This eliminates the "code says one thing, deploy says another" drift.
4. **Governance is lightweight by default; heavyweight where regulation requires.** A startup adopting their first frontier model needs a catalogue but not a Change Advisory Board. A regulated enterprise needs both. The catalogue's governance scales with the org; the catalogue itself doesn't.

Structure: (2) the catalogue schema; (3) the catalogue's role in adoption decisions; (4) the catalogue's role in operational monitoring; (5) integration with deployment; (6) governance; (7) tenant-aware and region-aware extensions; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The catalogue schema

What fields the registry tracks. The right schema is small enough to maintain and rich enough to answer the questions that matter.

### 2.1 Identity fields

- **model_id.** Unique within the catalogue. Convention: `<provider>:<model>:<version>` (e.g., `anthropic:claude-sonnet:4-5`). Stable across catalogue updates.
- **provider.** Anthropic, OpenAI, AWS Bedrock, GCP Vertex, self-hosted, etc.
- **model_family.** Claude Sonnet, GPT-4o, Llama 3, etc. Used for grouping across versions.
- **version.** The specific version (e.g., `4-5`, `4-6`). Matches provider's versioning.
- **alias.** Optional friendly name (e.g., `production-chat-default`). Useful for "the default model for chat" stable across version changes.

### 2.2 Deployment fields

- **endpoint(s).** Per-region endpoint URLs or service identifiers.
- **regions.** Where the model is deployed and available.
- **deployment_type.** Hosted (provider-managed), self-hosted (your inference cluster), managed (Bedrock / Vertex / Azure OpenAI).
- **inference_config.** Default temperature, max_tokens, system_prompt overrides if any. Per-feature override is allowed but documented.

### 2.3 Ownership fields

- **platform_owner.** The platform team responsible for the catalogue entry.
- **business_owner.** The product or engineering team that drove the adoption.
- **first_adopted_at.** Date of first production use.
- **on_call_rotation.** Who's paged if this model has an incident.

### 2.4 Compliance and governance fields

- **baa_coverage.** Yes / No / Not applicable. Provider BAA status.
- **dpa_coverage.** Yes / No. Data Processing Agreement status.
- **fedramp_status.** Authorized at Moderate / High / IL5 / Not authorized.
- **gdpr_lawful_basis.** Documented lawful basis for processing EU personal data.
- **data_classifications_allowed.** Which sensitivity classes can be processed (PHI / PII / Public / Confidential / Restricted).
- **prohibited_uses.** Explicit list of use cases not allowed (e.g., "no autonomous medical diagnosis").
- **review_status.** Pending / Approved / Restricted / Deprecated.
- **last_reviewed_at.** Date of last governance review.

### 2.5 Lifecycle fields

- **status.** Available, in-trial, in-production, deprecated, decommissioned.
- **deprecation_announced_at.** Date deprecation was announced (by provider or internally).
- **deprecation_effective_at.** Date the model is no longer usable.
- **replacement_model_id.** What model replaces this when deprecated.
- **migration_status.** Not started, in progress, completed.

### 2.6 Operational fields

- **typical_use_cases.** Free-text description of what this model is used for.
- **known_limitations.** Documented quirks, edge cases, output style notes.
- **cost_per_input_token.** Current rate.
- **cost_per_output_token.** Current rate.
- **typical_latency_ms.** P50 / P95 / P99 from production telemetry.
- **rate_limits.** RPM and TPM at the provider account level.
- **monthly_spend.** Aggregated from cost dashboards.
- **monthly_call_count.** Aggregated from usage telemetry.

### 2.7 Optional fine-tune fields

For fine-tuned variants:

- **base_model_id.** What the fine-tune is based on.
- **fine_tune_owner.** Team that owns the fine-tune.
- **tenant_id.** If per-tenant.
- **training_data_reference.** Pointer to the training data (with access controls).
- **eval_suite_reference.** Pointer to the eval suite for this variant.

### 2.8 The schema as code

```yaml
model_id: anthropic:claude-sonnet:4-5
provider: anthropic
model_family: claude-sonnet
version: 4-5
alias: production-chat-default

deployment:
  type: managed
  endpoints:
    us-east-1: arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-5
    eu-west-1: arn:aws:bedrock:eu-west-1::foundation-model/anthropic.claude-sonnet-4-5
  inference_config:
    temperature_default: 0.7
    max_tokens_default: 4096

ownership:
  platform_owner: ai-platform-team
  business_owner: clinical-ai-team
  first_adopted_at: 2025-08-15
  on_call_rotation: ai-platform-oncall

compliance:
  baa_coverage: yes
  dpa_coverage: yes
  fedramp_status: not_authorized
  gdpr_lawful_basis: legitimate_interest_documented
  data_classifications_allowed: [PHI, PII, Confidential, Public]
  prohibited_uses:
    - autonomous_medical_diagnosis_without_clinician_review
    - automated_eligibility_determination_without_human_review
  review_status: approved
  last_reviewed_at: 2026-04-01

lifecycle:
  status: in_production
  deprecation_announced_at: null
  deprecation_effective_at: null
  replacement_model_id: null
  migration_status: not_applicable

operational:
  typical_use_cases:
    - Care Coordinator agent (clinical reasoning)
    - Patient API AI-Assist (chat)
    - Analytics Warehouse Copilot (SQL generation)
  known_limitations:
    - Tends to over-explain in patient-facing contexts; tune via prompt
  cost_per_input_token: 0.000003
  cost_per_output_token: 0.000015
  typical_latency_ms:
    p50: 1200
    p95: 3400
    p99: 8200
  rate_limits:
    rpm: 5000
    tpm: 2000000
  monthly_spend: 67000
  monthly_call_count: 2800000
```

### 2.9 Schema discipline

- Fields are required or optional; the catalogue's CI validates schema.
- Required fields cannot be empty for any in-production entry.
- Schema changes are versioned; backward-compatible additions are easy; field removals are deprecation cycles.

---

## 3. The catalogue's role in adoption decisions

When a team wants to use a new model, the catalogue is the gateway.

### 3.1 The adoption workflow

```
Team proposes a new model
    ↓
Catalogue entry created in "in-trial" status
    ↓
Trial deployment in staging only
    ↓
Eval runs; cost analysis; security / compliance review
    ↓
Review board (or async approval for low-risk) decides
    ↓
Catalogue entry promoted to "approved" or rejected
    ↓
Production deployment unlocked
```

The catalogue entry is the artifact tracked through the workflow. The team proposes; reviewers comment; the entry is updated; the status field reflects the current state.

### 3.2 What the review evaluates

For a new model:

- **Technical fit.** Does the model meet the workload's quality bar (eval suite results)?
- **Cost.** What's the expected monthly spend? Compare to alternatives.
- **Latency.** Does the model meet the workload's latency requirements?
- **Compliance.** BAA / DPA coverage? Data classifications allowed? Regulatory posture?
- **Security.** Provider's security posture, data handling, model isolation.
- **Operational.** Provider reliability history, support quality, rate-limit suitability.
- **Strategic.** Does this add a new vendor dependency? Is the model unique or substitutable?

Some of these are bullet points the team submits; others are reviewed by specialists (security review, compliance review, platform review).

### 3.3 The lightweight vs heavyweight workflow

**Lightweight.** For low-risk model additions (new version of an existing approved model, alternative for an already-approved use case), async approval. Team updates the catalogue entry; platform reviews; approval within days.

**Heavyweight.** For new vendor additions, new high-risk use cases, regulated workloads. Synchronous review (review board meeting); legal review; compliance attestation. Approval within weeks.

The workflow's weight should match the risk. Forcing all model adoptions through the heavyweight path slows the team unnecessarily; making everything lightweight fails to catch the high-risk cases.

### 3.4 The "I want to use a different model just this once" case

A team has an approved primary model. For one specific workload, they want a different model (maybe a cheaper tier for low-stakes calls, maybe a faster model for latency-sensitive calls).

**Pattern.** The catalogue supports multiple approved models. Teams choose from the approved set; usage is tracked.

**Anti-pattern.** Team uses a model not in the catalogue "just this once." Production has an untracked model. Compliance review can't enumerate the production surface accurately.

### 3.5 The trial-period discipline

Models in "in-trial" status are restricted:

- Staging only.
- Limited usage cap (e.g., 1000 calls per day).
- Eval suite runs daily; results tracked.
- Trial period is bounded (e.g., 30 days); at end, decide promote or remove.

This discipline prevents "we'll just try it in production for a week to see" creep.

### 3.6 The catalogue as a model marketplace internally

A mature catalogue functions like an internal marketplace. Teams can browse the approved set; understand what each model is good for; pick the right one without inventing their own selection process.

```
Catalogue browse:
  Looking for: chat / medium-quality / low-latency / low-cost
  Approved options:
    - claude-haiku-4-5: $0.001/$0.005 per 1k tokens, 400ms P50
    - gpt-4o-mini: $0.001/$0.004 per 1k tokens, 350ms P50
    - llama-3-8b-self-hosted: $0.0001/$0.0005 per 1k tokens, 250ms P50
  Filter: must have BAA coverage
    - claude-haiku-4-5 ✓
    - gpt-4o-mini ✓
    - llama-3-8b-self-hosted ✓ (self-hosted in HIPAA-compliant infra)
```

The marketplace view reduces "what model should I use" research from days to minutes.

---

## 4. The catalogue's role in operational monitoring

Beyond adoption, the catalogue is the operational hub for production model usage.

### 4.1 Per-model usage telemetry

The catalogue is the join key for usage analytics:

- Monthly spend per model (cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md)).
- Monthly call count per model.
- Latency distribution per model (P50, P95, P99 from production traces).
- Error rate per model.
- Per-feature usage breakdown per model.

The telemetry feeds the catalogue's operational fields, which the catalogue exposes via API and dashboard.

### 4.2 Deprecation tracking

The catalogue is the source of truth for deprecations:

- Provider announces deprecation; platform team updates the entry's `deprecation_announced_at` and `deprecation_effective_at`.
- The catalogue's deprecation dashboard surfaces models nearing deprecation; teams using those models are notified.
- Migration to the `replacement_model_id` is tracked; the catalogue's `migration_status` field updates.

Without catalogue-tracked deprecation, deprecation surprises are normal. With it, they're prevented.

### 4.3 Cost outlier detection

Per-model spend is in the catalogue. Anomaly detection on the time series surfaces:

- Sudden spend increases (potential runaway feature or abuse).
- Sustained increases (organic growth that warrants budget update).
- Models with low usage but high per-call cost (consolidation candidates).

### 4.4 Latency degradation detection

The catalogue's latency fields update from production telemetry. If P99 drifts upward without a corresponding traffic increase, something's wrong:

- Provider-side degradation (escalate to provider).
- Prompt/context growth (audit the feature).
- Tool-call overhead increase (audit the agent).

### 4.5 Compliance audit support

When a compliance audit asks "which models process PHI?", the catalogue answers immediately:

```
Query: data_classifications_allowed contains "PHI" and status = "in_production"
Result:
  - anthropic:claude-sonnet:4-5
  - anthropic:claude-haiku:4-5
  - aws:llama-3-70b:self-hosted-hipaa-cluster
  - cohere:command-r-plus:ca-central-1
```

The auditor's question that takes a day to answer without the catalogue takes a minute with it.

### 4.6 Per-feature usage breakdown

The catalogue is the join key from per-call telemetry to per-model attribution:

```
Feature: Care Coordinator
  Calls: 800k/month
  Cost: $32k/month
  Model breakdown:
    - claude-sonnet-4-5: 700k calls, $28k (primary)
    - claude-haiku-4-5: 100k calls, $4k (fallback)
```

This enables product / engineering / FinOps conversations about feature-level cost and model usage.

### 4.7 The catalogue as a documentation hub

Each catalogue entry is documentation. Engineers reading the entry learn:

- What the model is good for (typical_use_cases).
- What to watch out for (known_limitations).
- Where to go if it breaks (on_call_rotation).
- What the constraints are (compliance fields).

The catalogue becomes the model's living documentation.

---

## 5. Integration with deployment

The catalogue's most load-bearing role is as the source of truth for deployment configuration.

### 5.1 Code references the catalogue

```python
# Instead of:
MODEL = "claude-sonnet-4-5-us-east-1"  # hardcoded
client = AnthropicClient(model=MODEL)

# Reference the catalogue:
model_entry = catalogue.get("anthropic:claude-sonnet:4-5")
client = get_client_for(model_entry, region=request.region)
```

The catalogue resolves to the appropriate client, endpoint, configuration, and access controls. Code doesn't hardcode endpoints, doesn't manage credentials, doesn't pick the wrong version.

### 5.2 The catalogue as the deployment manifest

Every production deployment references catalogue entries. The deployment manifest lists:

- Which catalogue entries this deployment uses.
- For each entry, what feature uses it.
- For each entry, what configuration overrides apply.

The CI pipeline validates that all referenced entries are in "approved" or "in-production" status. Deployments referencing non-approved entries fail to deploy.

### 5.3 The catalogue's API

```
GET /catalogue/models?status=in_production
GET /catalogue/models/{model_id}
GET /catalogue/models/{model_id}/endpoints?region={region}
POST /catalogue/models (with review workflow)
PATCH /catalogue/models/{model_id} (with audit log)
GET /catalogue/models/{model_id}/usage?period={period}
```

The API is what production code and CI use; the UI is what humans use.

### 5.4 The catalogue's UI

A web UI for browsing, querying, and managing entries. Used by:

- Engineers picking a model for a new feature.
- Platform team reviewing adoption requests.
- Compliance team auditing.
- FinOps reviewing cost.
- Leadership browsing the model portfolio.

### 5.5 Drift detection

A periodic job verifies:

- Every model referenced in production has a catalogue entry.
- Every catalogue entry in "in-production" status has actual usage.
- No deployment references models in "deprecated" status (after grace period).
- No deployment uses models in "rejected" status.

Drift triggers alerts. The catalogue's authority is enforced by detection + correction, not just by convention.

### 5.6 The catalogue and feature flags

For gradual rollout (e.g., A/B testing a new model version), feature flags reference catalogue entries:

```python
model_id = feature_flag(
    "primary_chat_model",
    default="anthropic:claude-sonnet:4-5",
    variants=["anthropic:claude-sonnet:4-6"]
)
client = get_client_for(catalogue.get(model_id))
```

Both candidate models are in the catalogue (one as "in-trial", one as "in-production"). The flag controls which is used; the catalogue ensures both have been reviewed.

---

## 6. Governance

The catalogue needs human governance. Without it, the catalogue is a database with no decisions; with too much of it, the catalogue becomes a bottleneck.

### 6.1 The roles

- **Catalogue admin.** Platform team member responsible for catalogue integrity. Reviews entries; approves schema changes; investigates drift.
- **Model owner (business).** Team that drove adoption. Maintains the entry's business fields; signs off on changes affecting their use case.
- **Model owner (platform).** Platform engineer responsible for operational concerns; on-call for incidents involving this model.
- **Reviewers.** Security, compliance, FinOps, architecture. Approve adoption; participate in reviews.

### 6.2 The decision flow

For new adoption:

- Team proposes (creates catalogue entry in "proposed" status).
- Catalogue admin checks completeness (required fields populated).
- Reviewers comment; approve or request changes.
- Review board (synchronous, weekly or biweekly) approves heavyweight cases.
- Lightweight cases approved async.

For modification:

- Owner proposes change.
- Catalogue admin checks completeness.
- Reviewers approve if material change (e.g., changing data classifications allowed).
- Updates logged in audit trail.

For deprecation:

- Trigger: provider announces, internal decision, security finding.
- Catalogue admin updates deprecation fields.
- Notification to affected teams via the catalogue's subscriber system.
- Migration tracking begins.

### 6.3 The review board

For organizations large enough to need one:

- Membership: platform lead, security lead, compliance lead, FinOps lead, sometimes legal.
- Cadence: weekly or biweekly.
- Agenda: review heavyweight adoption requests, deprecation announcements, exceptions.
- Decisions documented in the catalogue's audit trail.

Smaller organizations don't need a formal board; the catalogue admin coordinates with reviewers async.

### 6.4 The "can we just" pressure

Teams sometimes pressure for "can we just use this model without going through the process?" The answer is no — the catalogue's value depends on its completeness. Exceptions undermine the catalogue.

The fix isn't more flexibility; it's a faster lightweight path. Most adoption requests should be lightweight and approved within days. The heavyweight path is for genuinely heavy decisions.

### 6.5 Catalogue versioning

The catalogue itself is versioned. Changes are tracked; rollback is possible. The catalogue is treated like infrastructure-as-code: changes go through PR review, CI validation, deployment.

Some implementations store the catalogue in Git (YAML files per entry); some use a database with audit log + API. Either works; the discipline of versioning is the requirement.

### 6.6 The "central catalogue vs federated catalogues" question

Large organizations sometimes have multiple AI platforms (per business unit, per region, per regulated boundary). Options:

- **Central catalogue with scopes.** One catalogue; entries have scope (which business units / regions can use them). One source of truth.
- **Federated catalogues.** Multiple catalogues, each per BU / region. Models in multiple may have multiple entries. Reconciliation required.

**Recommendation.** Central catalogue with scopes is simpler and produces better cross-org visibility. Federation is justified when business units are genuinely independent (different regulatory environments, different procurement, different platforms).

---

## 7. Tenant-aware and region-aware extensions

The base catalogue tracks models. Multi-tenant and multi-region platforms need extensions.

### 7.1 Per-tenant catalogue scopes

Some catalogue entries are tenant-specific (per-tenant fine-tunes, dedicated provider accounts for specific tenants). The schema extends:

- **tenant_scope.** Specific tenant IDs or "all" if the entry is universal.
- **per_tenant_overrides.** Tenant-specific inference config, rate limits, cost attribution.

A tenant's effective catalogue is the union of universal entries plus tenant-specific entries. Cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md).

### 7.2 Per-region availability tracking

The catalogue tracks which regions each model is available in (the per-region model-availability matrix from [data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md)).

- **regions.** List of regions where the model is deployed and approved.
- **per_region_endpoints.** Endpoint per region.
- **per_region_compliance.** Compliance posture per region (e.g., model in `us-east-1` is BAA-covered; same model in `eu-west-1` is GDPR-eligible).

Teams selecting a model for a region query the catalogue with both criteria; the catalogue returns the eligible options.

### 7.3 Per-environment status

Different statuses per environment:

- Dev: in-trial.
- Staging: in-trial.
- Production: in-production.

Promotion through environments is tracked. The deployment manifest checks the environment-specific status.

### 7.4 The migration tracking extension

When a model is being migrated to a replacement, the catalogue tracks:

- Replacement model ID.
- Migration milestones (shadow start, canary start, full rollout, decommission old).
- Per-feature migration status (some features migrated; some not yet).
- Migration owner and timeline.

The catalogue's migration dashboard shows in-flight migrations and their state.

### 7.5 The audit log

Every change to the catalogue is logged:

- Who made the change.
- What changed.
- When.
- Why (link to approval, ticket, incident).

The audit log is queryable; compliance reviews use it; post-incident reviews use it.

---

## 8. Worked Meridian example

Meridian's model catalogue is the platform discipline that scaled from 3 models in 2024 to 18 models in 2026.

### 8.1 The starting state (2024)

Meridian had three models in informal use:

- GPT-4o for the Patient API AI-Assist chat.
- A small fine-tuned BERT for document classification.
- An OpenAI embedding model.

No catalogue. Configuration was per-team. When the security team asked "what models process PHI?", the answer required interviewing each team. The answer was: probably all of them, possibly some that nobody could enumerate.

### 8.2 The catalogue's introduction (2025)

Meridian's platform team introduced the catalogue. Initial scope:

- One YAML file per model entry, in Git, under `meridian-platform/catalogue/models/`.
- CI validation of schema.
- Required: model_id, provider, owner, BAA coverage, data classifications allowed.
- Optional initially (added later): cost, latency, deprecation, replacement.

First-pass populated 5 entries (the 3 existing plus 2 candidates being evaluated).

Within 3 months: 9 entries. Within 12 months: 15 entries.

### 8.3 The current state (mid-2026)

18 entries:

```
Anthropic:
  anthropic:claude-opus:4-7      (in-trial; premium tenant evaluation)
  anthropic:claude-sonnet:4-6    (in-production; default)
  anthropic:claude-sonnet:4-5    (deprecated; migration in progress)
  anthropic:claude-haiku:4-5     (in-production; fallback)

OpenAI:
  openai:gpt-4o:current          (in-production; legacy Patient API)
  openai:gpt-4o-mini:current     (in-production; cost-tier)
  openai:text-embedding-3-large  (in-production; embeddings)

Cohere:
  cohere:command-r-plus:ca-central-1 (in-production; Canadian residency)
  cohere:embed-multilingual       (in-trial; Canadian residency embedding)

Self-hosted:
  selfhosted:llama-3-70b:hipaa-cluster (in-production; high-volume classification)
  selfhosted:llama-3-8b:hipaa-cluster  (in-production; structured-output for billing-code workflow with the per-tenant fine-tune)
  selfhosted:bge-large:hipaa-cluster   (in-production; embeddings for sensitive data)

Per-tenant fine-tunes:
  selfhosted:llama-3-8b:tenant-specialty-practice-billing-code (in-production; one tenant, see per-tenant-fine-tuning example)

Provisional / evaluation:
  google:gemini-2.5-pro          (in-trial; alternative for analytics)
  selfhosted:qwen-3-72b          (in-trial; multilingual evaluation)
  anthropic:claude-haiku:4-6     (in-trial; haiku upgrade evaluation)

Embedding-specific:
  voyage:voyage-large-2          (in-trial; embedding quality evaluation)

Deprecated (kept for reference):
  selfhosted:bert-classification-v1 (decommissioned 2025-09; replaced by Llama 3 8B)
```

### 8.4 What the catalogue caught

**The "is this still in use?" question.** A platform engineer ran a query: "which catalogue entries have monthly_call_count < 100?" Result: bert-classification-v1 had been decommissioned but a config file in one feature still referenced it; a fallback path occasionally still called it. Caught and removed.

**The deprecation surprise prevented.** Anthropic announced Sonnet 4.5 deprecation effective Q3 2026. The catalogue's deprecation alert fired; affected features were identified; migration to Sonnet 4.6 was planned with months of lead time.

**The cost outlier surfaced.** The Care Coordinator's Sonnet 4.6 spend grew 40% in one month. The catalogue's spend trend flagged it; investigation revealed a feature that started using longer prompts; prompt trimming brought it back to baseline.

**The compliance audit answered.** During a HIPAA audit, the question "which models process PHI?" was answered in minutes via catalogue query. The auditor was satisfied with the documentation.

**The regional deployment for Canada.** When the Canadian acquisition required `ca-central-1` deployment, the catalogue was extended with per-region availability tracking. Cohere Command R+ was added with `ca-central-1` region. Per-region compliance fields were populated for each Canadian-eligible model.

### 8.5 The governance in practice

Meridian's review process:

- Lightweight adoption: new version of approved model, or new variant of approved family. Approved async by platform team within 3 business days.
- Heavyweight adoption: new vendor, new use case, new compliance posture. Reviewed at biweekly AI Architecture Review Board. Decisions documented.
- Deprecation: announced via catalogue notification; teams using the model are notified; 30-day comment period before deprecation status changes.
- Modifications: changes to compliance fields require security + compliance approval; operational fields update automatically from telemetry.

Time from adoption proposal to production deployment:

- Lightweight: 3-7 days.
- Heavyweight: 4-6 weeks.

### 8.6 The catalogue's integration

- Code references the catalogue (model_id, region) and gets back endpoint + config.
- Deployment manifests reference catalogue entries; CI fails if entries are not in approved status.
- Cost dashboards join with catalogue for per-model attribution.
- Latency monitoring updates the catalogue's per-model latency fields.
- Compliance dashboards show catalogue-based posture.

### 8.7 What the catalogue costs

- Initial setup: ~3 weeks of platform team's time.
- Ongoing: ~5% of platform team's time (catalogue admin role split across team).
- Infrastructure: small (Git repo + small DB for telemetry aggregations).
- Governance: ~2 hours / month for the review board.

### 8.8 What the catalogue avoids

- Untracked models in production (zero in last 12 months).
- Deprecation surprises (zero in last 12 months).
- Compliance audit improvisation (audits now routine, not panicked).
- "Which model are we using for X?" emails (answered by browsing catalogue).
- Cost surprises (per-model trends visible; outliers caught early).

Estimated avoided cost: 1-2 FTE / year in coordination, audit support, and incident response.

---

## 9. Anti-patterns

### 9.1 The shared spreadsheet catalogue

**Pattern.** Catalogue is a Google Sheet. Anyone can edit; nobody owns; it drifts; the last update was 8 months ago.

**Corrective.** Catalogue as code (Git) or as platform service. Schema-validated. Owned. Updated as part of the workflow.

### 9.2 The "we'll add it to the catalogue when we have time" exception

**Pattern.** New model added to production "in a hurry." Catalogue entry never created. Six months later, nobody remembers the model is in production.

**Corrective.** Catalogue entry is part of deployment; CI fails if model used in production isn't in catalogue. No exceptions for "small features" or "experiments."

### 9.3 The hardcoded model in code

**Pattern.** Feature code says `model = "claude-sonnet-4-5"`. Catalogue exists but is bypassed. Version changes require code changes; compliance can't verify.

**Corrective.** Code references catalogue by ID; catalogue resolves to current version / endpoint / config.

### 9.4 The catalogue with no telemetry

**Pattern.** Catalogue has compliance fields and cost fields filled in once at adoption; never updated; current cost / usage unknown.

**Corrective.** Operational fields update from telemetry. Anomalies (drift, deprecation, cost spike) detected.

### 9.5 The "deprecated but kept running" entry

**Pattern.** Model status changed to "deprecated" in catalogue. Production still uses it. Grace period passes; production still uses it. Eventually provider removes the model; outage.

**Corrective.** Deprecation enforcement: drift detection alerts on production usage of deprecated models; eventually CI fails deployment.

### 9.6 The lightweight path that's too heavyweight

**Pattern.** Every adoption requires synchronous review board approval. Teams batch adoption requests; new models take 3 months to deploy; teams work around the catalogue.

**Corrective.** Lightweight path for low-risk additions (same vendor, new version; alternative for approved use case). Heavyweight only for genuinely heavy decisions.

### 9.7 The catalogue that doesn't include self-hosted

**Pattern.** Catalogue tracks hosted models; self-hosted models considered "internal" and excluded. When compliance asks about models processing PHI, self-hosted models are missed.

**Corrective.** Every model in production is in the catalogue, regardless of hosting. Self-hosted has different fields (cluster reference, GPU SKU) but is catalogued.

### 9.8 The single catalogue across regulated and commercial

**Pattern.** Catalogue has both FedRAMP-authorized and commercial entries; queries don't easily distinguish. FedRAMP audits get confused.

**Corrective.** Scope field per entry; FedRAMP scope is distinct from commercial; queries respect scope; audit views are scope-filtered.

---

## 10. Findings (sprint-assignable)

**ARCH-MCR-001 (P0). No model catalogue exists.** Production models are not enumerable. Build catalogue (Git-as-code or platform service); populate with all current production models; enforce schema. Owner: AI platform.

**ARCH-MCR-002 (P0). Catalogue exists but doesn't include all production models.** Some features use models not in catalogue. Audit production; add all to catalogue; enforce via CI on deployment. Owner: AI platform + feature teams.

**ARCH-MCR-003 (P0). Code hardcodes model IDs / endpoints.** Catalogue is bypassed; drift inevitable. Refactor to catalogue references; client SDK resolves catalogue to endpoint. Owner: platform.

**ARCH-MCR-004 (P0). No CI check that deployed models are in catalogue.** Deployment with non-catalogued models succeeds. CI fails deployment if any referenced model isn't in "approved" or "in-production" status. Owner: platform.

**ARCH-MCR-005 (P1). Catalogue's operational fields (cost, latency) not updated from telemetry.** Catalogue says one thing; reality is another. Build telemetry pipeline that updates operational fields daily; alert on drift. Owner: observability + platform.

**ARCH-MCR-006 (P1). No deprecation tracking in catalogue.** Provider deprecations are surprises. Add deprecation fields; subscribe to provider announcements; populate proactively. Owner: AI platform.

**ARCH-MCR-007 (P1). No per-region availability tracking.** Region migration requires manual matrix lookup. Add regions and per_region_endpoints; populate from provider docs. Owner: AI platform.

**ARCH-MCR-008 (P1). No compliance field discipline.** BAA / DPA / FedRAMP / data_classifications_allowed not consistently populated. Define required compliance fields; audit existing entries; populate. Owner: compliance + AI platform.

**ARCH-MCR-009 (P1). Adoption workflow undefined or heavyweight-by-default.** Teams work around the catalogue. Define lightweight and heavyweight paths; commit to SLAs (3-7 days lightweight, 4-6 weeks heavyweight). Owner: AI platform.

**ARCH-MCR-010 (P1). No catalogue UI.** Browsing requires reading YAML or querying API. Build simple UI (read-mostly, with workflow integration). Owner: AI platform.

**ARCH-MCR-011 (P2). No drift detection.** Drift accumulates; catalogue's authority degrades. Periodic job verifies referenced models are catalogued, in-production entries have actual usage, deprecated entries aren't used. Owner: AI platform.

**ARCH-MCR-012 (P2). Catalogue audit log is incomplete or absent.** Compliance audits can't trace decisions. Every change logged: who, what, when, why. Audit log queryable. Owner: platform + compliance.

**ARCH-MCR-013 (P2). Self-hosted models not in catalogue.** Compliance enumeration misses them. Self-hosted entries with cluster reference, GPU SKU, ops owner. Owner: AI platform.

**ARCH-MCR-014 (P2). Per-tenant fine-tunes not catalogued.** Tenant-specific models invisible at platform level. Per-tenant entries with tenant_scope and tenant_id. Owner: AI platform.

**ARCH-MCR-015 (P2). No alias support.** Code references specific version; version upgrade requires code change. Add alias field; code references alias; catalogue resolves alias to current version. Owner: platform.

**ARCH-MCR-016 (P3). Review board doesn't meet on schedule.** Heavyweight adoption requests pile up. Commit to cadence; document decisions; track decision lead time. Owner: AI platform.

**ARCH-MCR-017 (P3). No catalogue notification system.** Affected teams not informed of deprecations, compliance changes. Subscribe / notification system on entry changes. Owner: platform.

**ARCH-MCR-018 (P3). Federated catalogues without reconciliation.** Multiple BUs have their own catalogues; cross-BU visibility absent. Either consolidate to central with scopes or build reconciliation. Owner: enterprise architecture.

---

## 11. Adoption sequencing checklist

For a team adopting a model catalogue, in order:

- [ ] **Decide on storage:** Git-as-code (YAML per entry) or platform service (DB + API). Either works.
- [ ] **Define minimum schema (§2).** Required fields: identity, ownership, compliance, lifecycle. Optional fields added later.
- [ ] **Populate with all current production models.** Audit production; create entry per model.
- [ ] **Implement schema validation in CI.** Required fields must be populated; status field is constrained enum.
- [ ] **Refactor code to reference catalogue (§5.1).** Eliminate hardcoded model IDs / endpoints.
- [ ] **Add CI check (§5.5):** deployment with non-catalogued or non-approved models fails.
- [ ] **Build catalogue API (§5.3).** Query, get, get-endpoints, post, patch.
- [ ] **Build telemetry pipeline (§4.1).** Production usage updates catalogue's operational fields.
- [ ] **Define adoption workflow (§3.1).** Lightweight and heavyweight paths; SLAs.
- [ ] **Set up notification system.** Affected teams notified of entry changes.
- [ ] **Add deprecation tracking (§4.2).** Subscribe to provider announcements; populate proactively.
- [ ] **Add per-region availability tracking (§7.2).** Populate from per-region model-availability matrix.
- [ ] **Add per-tenant entries support (§7.1).** Tenant scope; per-tenant overrides.
- [ ] **Build catalogue UI (read-mostly initially).** Browsing, search, basic editing.
- [ ] **Implement drift detection (§11 finding).** Periodic job; alerts on drift.
- [ ] **Set up governance (§6).** Roles defined; review board if needed; audit log.
- [ ] **Run quarterly catalogue health review.** Schema additions; drift; outdated entries.

---

## 12. References

**In this folder.**
- [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md) — foundational decision; what models go in the catalogue.
- [model-routing-and-tiering.md](./model-routing-and-tiering.md) — routing patterns that reference catalogue entries.
- [build-vs-buy-decision.md](./build-vs-buy-decision.md) — companion; the build decisions land in the catalogue.
- [model-migration-playbook.md](./model-migration-playbook.md) *(coming)* — migration tracking uses catalogue lifecycle fields.
- [capability-vs-cost-vs-latency-tradeoffs.md](./capability-vs-cost-vs-latency-tradeoffs.md) *(coming)* — comparison framework; catalogue captures the data.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — per-tenant fine-tunes land in the catalogue with tenant scope.
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — per-region availability tracking integrates with catalogue.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose catalogue is described here.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — policy that references catalogue.

**Sibling repos.**
- [ai-engineering-reference-architecture / model-lifecycle / model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md) — engineering-side registry implementation; this doc is the architectural view.
- [ai-engineering-reference-architecture / cicd-and-eval-gates / model-version-pinning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cicd-and-eval-gates/model-version-pinning.md) — version-pinning discipline that catalogue enforces.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-model cost attribution joins to catalogue.
- [ai-engineering-reference-architecture / observability-and-telemetry / cost-dashboards.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/cost-dashboards.md) — dashboards join to catalogue.

**External.**
- Backstage (Spotify) — service catalogue concepts; similar architectural pattern for software services.
- Anthropic / OpenAI / Google model documentation — source for catalogue entry fields per model.
- AWS Bedrock model marketplace — internal-marketplace pattern analog.
- MLflow Model Registry — ML-side registry; similar concepts.
