# Data Contracts for Retrieval

> **Audience.** Architects designing the data side of a RAG system that consumes content from upstream sources owned by other teams. Engineers debugging "the model's quality went down last Tuesday and nobody knows why." **Scope.** The *architectural* pattern for treating retrieval corpora as contract-bound published datasets, with schema, freshness, content-type, and change-protocol guarantees enforced at the boundary. Not the engineering practice of building ingestion pipelines (the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `rag-engineering/ingestion-pipeline-engineering.md` owns that) or the dataset-versioning practice (sibling's `data-engineering-for-ai/dataset-versioning.md`). **Worked client.** Meridian Health. **Companion docs.** [lineage-and-provenance.md](./) for the per-document attribution pattern (coming); [freshness-architecture.md](./) for the staleness-vs-realtime architectural choices (coming); [vector-store-architecture.md](./) for the storage decisions (coming).

---

## 1. Why this document exists

The single most common diagnosis for an unexplained RAG quality regression in 2026 is: *the upstream content source changed and the AI team did not know about it*. The CMS team updated the HTML structure of clinical guidelines; the chunker started producing chunks split at the wrong boundaries; retrieval recall on the eval suite dropped 9 points overnight. The wiki team migrated from one platform to another; the document IDs changed; the production system's deep links to "guideline section 3.2.1" now 404. The product team added a new content type that the ingestion pipeline does not handle; the new content shows up in retrieval but cannot be reliably formatted into context.

In every case, the root cause is the same: the retrieval corpus has no *contract* with its upstream sources. The AI team consumes whatever the upstream produces; when upstream changes, the AI team finds out by observing a quality regression in production. The diagnostic process is "production is broken → grep through audit logs → eventually correlate to an upstream change."

The data-engineering canon solved this problem a decade ago for data warehouses. Data contracts — schema, freshness, content-type, owner, change protocol — turn upstream sources into committed suppliers rather than ambient inputs. The AI corpus is a downstream consumer; the same discipline applies; almost no one applies it.

This document is opinionated about three things:

1. **The retrieval corpus is a published dataset.** It has an owner, a schema, a freshness SLO, a versioning model, a documented refresh process, and a deprecation lifecycle. Treating it as anything less is choosing not to know about upstream changes.
2. **Contracts are enforced at the ingestion boundary, not after retrieval failure.** When the upstream produces content that violates the contract, the ingestion pipeline refuses to ingest it (or quarantines it) and alerts the contract owner. The contract is the line of defense; production quality is what remains after the contract has held.
3. **Every retrieval corpus has a humanly-named owning team.** The team that publishes the upstream content owns the contract; the AI team is the contract consumer. The ownership is enforced architecturally (who can change the contract, who is paged when the contract is violated) and contractually (a documented data-supplier agreement between the teams).

Structure: (2) the contract elements; (3) per-source vs unified contracts; (4) contract enforcement at ingestion; (5) contract violation detection and response; (6) the lifecycle of a contract; (7) worked Meridian example across three corpus sources; (8) anti-patterns; (9) findings; (10) adoption checklist.

---

## 2. The contract elements

A complete retrieval-corpus data contract has nine elements. Each is enforced or surfaced architecturally; each has an owner; each is versioned.

### 2.1 Identity

- **Corpus name.** A canonical name (`clinical-guidelines`, `tenant-protocols`, `drug-interactions`, `patient-education`). This is the identifier used in code, in dashboards, in incident channels.
- **Owner team.** The team responsible for the upstream content (not the AI team). For Meridian's `clinical-guidelines`: the medical-content-licensing team. For `tenant-protocols`: the hospital's own clinical-informatics team (per tenant, so the contract is multi-party). For `drug-interactions`: the platform pharmacy-informatics team.
- **Consumer team.** The AI team that consumes from this corpus. Often more than one.
- **Contract version.** Semantically versioned. Major bumps are breaking changes; minor bumps are additive; patches are clarifications.

### 2.2 Source location and access

- **Upstream system.** Where the canonical content lives — the publishing CMS, the FDA SPL feed, the tenant CMS, the wiki. Named explicitly so an incident responder knows where to look.
- **Access mechanism.** Pull API endpoint, S3 bucket, webhook subscription, database replica, file drop. Whatever the AI ingestion pipeline reads from.
- **Authentication.** What credentials the AI team uses to read; whose credentials they are; what rotation cadence applies.
- **Availability SLO.** What uptime guarantee the upstream offers, what fallback the AI ingestion uses if upstream is unavailable.

### 2.3 Schema

- **Document schema.** What fields every document has, what types, what is required, what is optional, what valid values are for enums. For Meridian's `clinical-guidelines`: title, version, publication date, society, topic, body (HTML), citations, structured-metadata (medications-discussed, conditions-discussed, CPT codes mentioned).
- **Body content type.** HTML, Markdown, plain text, PDF, XML, mixed. Mixed is allowed but enumerated — "HTML or Markdown" rather than "whatever the upstream sends."
- **Versioning model.** Per-document version-string, last-modified timestamp, content hash. The AI pipeline uses this to detect updates without redoing unchanged work.

### 2.4 Freshness

- **Update cadence.** How often the upstream publishes (continuously, daily, weekly, quarterly, on-event). For each cadence, the SLO the AI ingestion will hold itself to.
- **Freshness SLO.** Maximum acceptable age of content in the retrieval corpus, per source. For `clinical-guidelines`: 7 days from upstream publication. For `tenant-protocols`: 24 hours from upstream change. For `drug-interactions`: 30 days from FDA SPL release.
- **Freshness alerting.** When the SLO is at risk (corpus is approaching freshness limit), what alert fires and who is paged.

### 2.5 Volume and growth

- **Expected size.** Document count and aggregate size (tokens, bytes). Used for capacity planning and to alert on unexpected growth (10x size suddenly is a signal).
- **Growth rate.** Per-period new-document rate. Used for capacity planning and chunking-throughput planning.
- **Per-document size distribution.** P50, P95, P99 document sizes. Used for chunk-strategy validation.

### 2.6 Quality

- **Content quality contract.** What the upstream guarantees about content quality — editorial review, peer review, automated lint, no guarantee. Drives the trust the AI consumer can place in the content.
- **Known issues.** Documented patterns that the upstream knows about but does not fix — for example, "tables in clinical guidelines occasionally lose formatting; AI ingestion should fall back to text extraction." Better to document than discover.

### 2.7 Multi-tenancy

- **Tenant scope.** Which tenants this corpus applies to. `clinical-guidelines` applies to all tenants (global content). `tenant-protocols` applies to one tenant per instance (per-tenant contract). `drug-interactions` is global.
- **Per-tenant variability.** If different tenants get different content from the same logical corpus, the variation rules are documented.

### 2.8 Change protocol

- **Breaking change definition.** What constitutes a breaking change in the corpus — schema field removed, enum value retired, content-type changed, freshness SLO loosened.
- **Notice period.** Minimum lead time for a breaking change (Meridian: 30 days for major changes; 7 days for minor).
- **Notification mechanism.** How the upstream notifies the AI team of a planned change — Slack channel, email distribution, ticket in the change-management system.
- **Deprecation lifecycle.** How a content type or field is retired without breaking the consumer.

### 2.9 Lineage and provenance

- **Source attribution.** Every chunk retrieved by the AI system can be attributed back to its source document. The contract specifies what identifier the upstream uses and the AI side preserves.
- **Provenance metadata.** What metadata travels with the document through ingestion → chunking → embedding → retrieval → answer. Drives the audit story.

---

## 3. Per-source vs unified contracts

Two extreme patterns, both wrong; the productive answer is in between.

### 3.1 The unified-contract anti-pattern

The team writes one "retrieval corpus contract" that all upstream sources are expected to conform to. The contract is generic; it does not capture per-source specifics; it cannot enforce per-source SLOs. Upstream teams sign but the contract has no teeth — there is no concrete violation that triggers a defined response.

### 3.2 The per-document-contract anti-pattern

Every individual document is treated as its own contract. The discipline cost is enormous and no team can maintain it. The contract becomes paperwork; the actual enforcement degrades.

### 3.3 The productive middle: per-source contract

One contract per logical *source* — meaning, one contract per upstream system + per tenant for per-tenant sources. For Meridian, the contract set is:

- `clinical-guidelines`: 1 contract, global.
- `drug-interactions`: 1 contract, global.
- `patient-education`: 1 contract, global.
- `tenant-protocols`: 240 contracts at scale, one per hospital tenant — but generated from one template so the contract authoring is templated, not bespoke.

Per-source means the contract can capture the source's specifics (the publishing cadence, the schema quirks, the known issues). It also means the responsibility is locatable — when `tenant-protocols` for hospital Mercy Cleveland is stale, it is Mercy Cleveland's clinical-informatics team that is responsible, not "the protocols team."

The Meridian default: per-source contracts, with a master template per *kind* of source. New hospitals are onboarded by instantiating the `tenant-protocols` template with the hospital-specific parameters; the template's contract structure is consistent across hospitals.

---

## 4. Contract enforcement at ingestion

The contract is only useful if it enforces something concrete at ingestion time. The pattern is a *contract gate* in the ingestion pipeline: every document pulled from the upstream is validated against the contract; non-conforming documents are quarantined (not silently ingested) and alerts fire.

### 4.1 The contract-gate pipeline

```
upstream system
      │
      ▼ pull (API, S3, webhook)
┌──────────────────┐
│  Receive doc     │
└─────────┬────────┘
          │
          ▼
┌──────────────────┐
│  Contract gate   │  validate against the contract:
│                  │  - required fields present?
│                  │  - field types match?
│                  │  - enum values valid?
│                  │  - body content-type accepted?
│                  │  - size within bounds?
└─────────┬────────┘
          │
       ┌──┴──┐
       │     │
   conform  violate
       │     │
       │     └──→ quarantine + alert contract owner
       ▼
┌──────────────────┐
│  Chunk + embed   │
└─────────┬────────┘
          │
          ▼
┌──────────────────┐
│  Index           │
└──────────────────┘
```

The contract gate is the single most underused pattern in production RAG. Most ingestion pipelines accept whatever the upstream sends; the contract validation happens in the consumer's head, not in code.

### 4.2 Quarantine, not reject

When a document fails contract validation, the right response is *quarantine, alert, do not block production*. The pattern:

- The non-conforming document is staged in a quarantine area (separate index, separate storage).
- The previous version of the document (if any) remains in the production retrieval corpus.
- An alert fires to the contract owner team: "Document `doc-X` failed contract validation on field `body_content_type`; quarantined; please investigate."
- A human-readable diagnostic is attached to the quarantine record so the contract owner can fix the upstream.
- The AI team's runbook for contract-violation incidents is to triage with the owner team, not to "fix" by relaxing the validation.

Blocking ingestion entirely on contract violation would freeze the corpus when any single document fails; serving the bad document silently would propagate the failure into production quality. Quarantine + alert is the productive middle.

### 4.3 What the contract gate enforces

| Contract element | Gate check |
|---|---|
| Schema — required fields | Fail if any required field is missing |
| Schema — field types | Fail if a type does not match (e.g., integer where string expected) |
| Schema — enum values | Fail if a value is outside the declared enum |
| Body content type | Fail if the content type is not one of the declared accepted types |
| Volume — per-document size | Fail if a document is wildly outside expected size distribution |
| Volume — aggregate growth | Alert (not fail) if total corpus size grows >50% in a 24-hour window |
| Freshness — staleness | Alert if the corpus has not received an update from this source in > freshness SLO |
| Quality — body sanity | Optional content-quality checks (no empty bodies, no obviously garbled extraction) |

The freshness check is interesting: it fires from the *absence* of input, not from the presence of bad input. The contract owner team is alerted when their content has not flowed.

### 4.4 The two-team workflow on a contract violation

1. **Upstream team makes a change** (publishes a doc with a new field, retires an enum value, changes content type).
2. **AI ingestion pulls the new content.** Contract gate validates. Validation fails. Document is quarantined.
3. **Alert fires** to upstream team (the contract owner) and AI team (consumer, for awareness).
4. **Upstream team triages.** Either: (a) the change was unintentional, the upstream rolls back; (b) the change was intentional but the AI team was not notified — upstream apologizes and now files a proper contract-amendment request; (c) the change was intentional and was supposed to go through change-protocol — both teams trace the failure in the change-management process.
5. **Resolution.** Contract is amended (with the proper notice period) OR upstream content is reverted OR the AI team accepts the change ad-hoc (rare).
6. **The quarantined document is re-evaluated** against the (possibly amended) contract; either ingests or stays quarantined pending resolution.

This is the workflow that gives contracts teeth. Without it, "we have a contract" is paperwork.

---

## 5. Contract-violation detection and response

Three classes of contract violation, each with a different detection mechanism and response.

### 5.1 Hard schema violation (caught at ingestion)

The doc fails the contract gate because a required field is missing or a field type is wrong. Detection: at ingestion. Response: quarantine + alert + two-team workflow.

This is the easiest class to handle because the failure is loud and immediate. The discipline is: don't ignore the alerts.

### 5.2 Silent semantic drift (caught by quality regression)

The doc passes the schema-level contract gate but the *content* has changed in a way that degrades downstream quality. Example: the upstream changed clinical guideline formatting from one paragraph per recommendation to one long block; the chunker handles the new format but produces lower-quality chunks; retrieval recall drops without anyone noticing for two weeks.

Detection: by eval quality regression in production. The diagnostic playbook (sibling's `rag-engineering/rag-failure-modes-and-debugging.md`, coming):

1. Production quality regression alert fires (judge-pass-rate dropped > 8 points).
2. The eval breakdown by case class identifies which case class regressed.
3. The trace for failing cases is inspected; retrieved doc IDs are examined.
4. The retrieved docs are compared against their versions in the regression window; the upstream change is identified.

Response: file a contract-amendment request; the contract may need to be tightened to include a structural-content check (e.g., "guidelines must have at least one section header per 1000 tokens") so the gate catches this class next time.

### 5.3 Freshness lapse (caught by absence)

The upstream stopped publishing. The corpus is now stale. Detection: freshness SLO timer fires. Response: alert the upstream owner; in parallel, the AI consumer team decides whether to serve the stale content with a freshness warning, switch to a fallback source, or degrade the feature.

This is where pre-defined runbooks matter. A team without a runbook spends the incident debating tradeoffs; a team with one executes.

---

## 6. The lifecycle of a contract

Contracts have a lifecycle. Treating them as static documents is a way to ensure they decay.

### 6.1 Initial drafting

Triggered by either: a new AI feature that needs a new source, or an existing source being formalized as a contracted supplier (the implicit pattern becomes explicit). The drafting involves the upstream team and the AI team; security and compliance review for regulated content.

The first version of a contract is usually conservative — narrower than the actual upstream behavior — because that is what can be enforced. The contract loosens (with deliberate amendments) as the AI team gains confidence.

### 6.2 Amendment (minor version)

Additive changes — a new field appears, a new accepted content type is added, the freshness SLO is improved. Minor bump; backward-compatible; no notice period beyond ordinary review.

### 6.3 Amendment (major version)

Breaking changes — a field is removed, an enum value is retired, content type changes, freshness SLO is loosened. Major bump; explicit notice period (Meridian default: 30 days); coordinated rollout; the AI team's ingestion pipeline is updated before the major change lands.

The major-version notice period is the load-bearing element. Without it, upstream "breaking" changes break production. With it, the AI team has time to adapt the ingestion (or to negotiate a different change).

### 6.4 Retirement

When a corpus is no longer needed (a feature is decommissioned, an upstream is end-of-life), the contract is formally retired. The retirement process:

- AI team announces planned removal of the source from production.
- Migration plan if any consumers still depend on it.
- After grace period, the ingestion is shut down; the corpus data is archived per the retention SLO; the contract is marked retired.

### 6.5 Audit / re-review

Quarterly: every active contract is reviewed. Are the freshness SLOs still right? Have the schemas drifted? Are quality issues accumulating? The re-review is the equivalent of dependency health checks for code dependencies — keeps drift from compounding.

---

## 7. Worked Meridian Health example

Three Meridian corpora, three contracts, three operational shapes.

### 7.1 Corpus: `clinical-guidelines`

**Contract summary:**

- **Owner:** medical-content-licensing team (Meridian platform team responsible for licensing and curating professional-society clinical guidelines from AHA, ACC, IDSA, etc.).
- **Consumer:** Care Coordinator's clinical-knowledge worker; analytics-warehouse copilot uses a subset.
- **Source:** Internal CMS that ingests from licensed publishing-society feeds, normalizes, and re-publishes.
- **Schema:** Title, version, society, publication date, last-modified date, topic taxonomy (controlled vocabulary), body (HTML with constrained set of allowed tags), citations (structured), structured metadata (medications-discussed, conditions-discussed, CPT codes, ICD-10 codes).
- **Body content type:** HTML, restricted tag set.
- **Update cadence:** Quarterly per society; the internal CMS aggregates and republishes monthly.
- **Freshness SLO:** 7 days from internal CMS publication.
- **Tenant scope:** Global (all hospitals see the same guidelines).
- **Quality:** Licensed and editorially-reviewed; the publishing societies' editorial process is the quality guarantee.
- **Change protocol:** Major changes (schema field removal, content-type change) require 30-day notice + AI team sign-off. Minor changes (new field, new society added) require 7-day notice. Per-guideline content updates are not contract changes (they are normal updates).
- **Lineage:** Every chunk carries source document ID, society, version, publication date. Cited back to the source in user-facing answers.

**Contract gate enforcement at ingestion:**
- Required fields validated.
- HTML tag set validated; documents with un-whitelisted tags are quarantined (most often a publishing-society style change introduced an unexpected tag).
- Per-document size sanity check (a 200-token "guideline" is almost certainly a malformed document; a 200,000-token "guideline" is almost certainly a multi-document concatenation).
- Freshness timer: if no new content for > 7 days, alert.

**Incidents observed:**
- 2026-03-14: The AHA published a Heart Failure guideline update with embedded SVG figures (new tag). Contract gate quarantined the doc. Medical-content-licensing team filed a minor amendment (allow `<svg>` tag); the AI team's chunker was updated to handle SVG figures; the amendment landed within a week.
- 2026-04-22: The internal CMS deploy introduced a serialization bug that produced empty `body` fields for ~6% of documents. Contract gate quarantined them. Incident was caught in ~12 minutes; CMS deploy was rolled back; quarantined docs were re-ingested.

### 7.2 Corpus: `tenant-protocols`

**Contract summary (per-hospital instance):**

- **Owner:** the hospital's clinical-informatics team (e.g., Mercy Cleveland's clinical-informatics team for the `mercy-cleveland` instance).
- **Consumer:** Care Coordinator's clinical-knowledge worker, scoped to this hospital tenant.
- **Source:** Per-hospital CMS or document store; uploaded via the Meridian tenant-content-management interface.
- **Schema:** Per-hospital protocol with title, version, applicable-roles (which clinical roles this protocol governs), body (Markdown), effective date, expiration date, owner-clinician.
- **Body content type:** Markdown.
- **Update cadence:** Continuous (whenever the hospital's clinical-informatics team makes a change).
- **Freshness SLO:** 24 hours from upstream change.
- **Tenant scope:** This one hospital.
- **Quality:** Each hospital's clinical-informatics team is responsible; the Meridian platform validates structure but not clinical content.
- **Change protocol:** Per-protocol updates are normal; structural changes (the protocol-document schema itself) are managed centrally with notice to all hospitals.
- **Lineage:** Every chunk carries source protocol ID, version, effective date, owner-clinician. Citation in answers includes the hospital's protocol name.

**Notable considerations.**
- 240 instances of this contract (one per hospital). Template-driven authoring means the contract structure is consistent; per-hospital parameters (the hospital's CMS connector, the hospital's clinical-informatics team contact) vary.
- The freshness SLO is per-hospital — if Mercy Cleveland updates a protocol and the change has not appeared in retrieval within 24 hours, Mercy Cleveland's clinical-informatics team is alerted (not Meridian's medical-content-licensing team).
- Hospital onboarding includes signing the contract; the contract is part of the customer onboarding documentation.

**Incidents observed.**
- 2026-02-11: Mercy Cleveland migrated their CMS to a new platform that changed protocol IDs. Without coordination, the AI system's existing references to "Mercy Cleveland Anticoagulation Protocol HF-22" no longer matched any source. Contract gate flagged the mass-rename as a structural anomaly (the hospital's prior 47 protocols all disappeared and 47 new ones appeared); incident was caught within 6 hours; the Mercy Cleveland team migrated IDs via a backwards-compatible alias map maintained in the contract metadata.

### 7.3 Corpus: `drug-interactions`

**Contract summary:**

- **Owner:** platform pharmacy-informatics team (consumes the FDA Structured Product Labels feed and maintains the curated drug-interaction graph).
- **Consumer:** Care Coordinator's clinical-knowledge worker (for interaction-shaped queries); also exposed as a tool (`look_up_drug_interactions`).
- **Source:** FDA SPL feed (upstream-to-Meridian), with internal curation overlay.
- **Schema:** Graph structure — nodes (medications with normalized IDs, alternative names, classes), edges (interactions with severity, mechanism, clinical recommendation, source citation).
- **Update cadence:** FDA SPL feed updates continuously; Meridian's curated graph is rebuilt monthly with FDA changes + Meridian-curated content.
- **Freshness SLO:** 30 days from FDA SPL release.
- **Tenant scope:** Global.
- **Quality:** FDA SPL is authoritative for the structured product label content; the platform pharmacy-informatics team is responsible for the curation overlay (alternative-name normalization, severity categorization, clinical-recommendation guidance).
- **Change protocol:** FDA SPL schema changes are out of Meridian's control; the platform team commits to absorbing them within 14 days. Curation-overlay changes (new severity criteria, new alternative-name mapping) go through internal change management.
- **Lineage:** Every interaction returned from the graph carries the FDA SPL source document ID and revision date. Answers cite "FDA SPL for Drug X (revised 2026-04-12)".

**Notable considerations.**
- This is a graph contract rather than a document contract; the schema fields are graph-shaped (node types, edge types, properties).
- The FDA SPL feed is itself an external dependency outside Meridian's control. The contract documents this and includes the fallback (the previous month's graph remains available if the SPL ingest fails).
- The monthly rebuild is a structured operation: the new graph is built in shadow; eval suite runs against the new graph; if eval passes, the new graph promotes; if not, the previous graph remains in service. The eval gate is the contract enforcement for content quality.

**Incidents observed.**
- 2026-01-20: FDA SPL feed changed a metadata field schema. The Meridian pharmacy-informatics team's automated ingest failed on the schema change. Contract gate alerted; the team updated the ingest within 3 days; the production graph remained on the previous month's version during the gap. No downstream impact.

### 7.4 Cross-corpus dashboard

The Meridian platform team maintains a single dashboard listing every corpus, contract version, owner, last-update timestamp, freshness SLO, and current health:

```
Corpus                         Contract    Owner team              Last update     Freshness    Health
─────────────────────────────────────────────────────────────────────────────────────────────────────
clinical-guidelines            v2.1        med-content-licensing   2026-05-22      7d / OK      OK
drug-interactions              v1.4        pharmacy-informatics    2026-05-01      30d / OK     OK
patient-education              v3.0        patient-content         2026-05-24      7d / OK      OK
tenant-protocols/mercy-cleveland  v1.0     mercy-cleveland-ci      2026-05-25      24h / OK     OK
tenant-protocols/st-marys-bos     v1.0     st-marys-bos-ci         2026-05-23      24h / WARN   stale 2d
... (240 tenant rows)
```

The dashboard is the operational surface; alerts route to the appropriate owner team per row.

---

## 8. Anti-patterns

### 8.1 "We ingest whatever the upstream sends"

No contract; no validation; no defined consequence when the upstream changes. The team discovers upstream changes by observing production quality regressions. Diagnosis-to-fix latency is days to weeks.

**Corrective.** Per-source contracts (section 2). Contract gate at ingestion (section 4). Quarantine + alert (section 4.2).

### 8.2 "Validation lives in chunking code"

The team validates content shape inside the chunker — "if the document is missing a title, skip it." Validation is invisible from the outside; failures are silent; upstream teams have no signal that something they did broke downstream consumption.

**Corrective.** Validation is a named pipeline stage (the contract gate); failures emit alerts; the upstream team is notified, not just the AI team.

### 8.3 "The contract is a wiki page"

A "data contract" exists as documentation but is not enforced. The wiki page is updated when someone remembers; the production ingestion pipeline does not consult it.

**Corrective.** Contract is code (a schema definition, validation rules, freshness SLO declarations). The wiki documents intent; the code enforces.

### 8.4 "Breaking changes ship without notice"

The upstream team changes the schema; the AI team's ingestion breaks the next day. The upstream team is surprised; the AI team is paged; trust between teams degrades over time.

**Corrective.** Change protocol with explicit notice periods (section 2.8). Breaking changes are coordinated. The contract is the meeting ground.

### 8.5 "Freshness is measured post-hoc"

The team notices stale content in production when the model serves an out-of-date answer. Freshness has not been a tracked SLO; the gap was not visible until users noticed.

**Corrective.** Freshness SLO per corpus, alerting on SLO breach (section 4.3). The team knows the corpus is stale before users do.

### 8.6 "Per-tenant contracts are bespoke"

The team writes one-off contracts for each tenant. At 240 tenants, the contract-maintenance burden is unsustainable; contracts drift; each tenant gets slightly different treatment.

**Corrective.** Template-driven contracts per source kind. Per-tenant parameters fill the template; the structure is shared. New tenants are onboarded by template instantiation, not by drafting from scratch.

### 8.7 "Contract violations get logged and ignored"

The contract gate fires alerts; the alerts go to a channel nobody monitors. Violations accumulate; eventually one becomes a production incident.

**Corrective.** Alerts route to the contract owner team (not just the AI team); SLO on alert response time; track and report the violation rate to leadership.

### 8.8 "Contracts are written once and never reviewed"

Contracts created during a feature launch are still in service three years later, unchanged, against an upstream that has evolved. The contract no longer reflects reality; enforcement is theater.

**Corrective.** Quarterly contract review (section 6.5). Active contracts are kept current; stale ones are retired.

---

## 9. Findings (sprint-assignable)

### ARCH-CORPUS-001 — Severity: Critical
**Finding.** No data contracts exist for production retrieval corpora; upstream changes are discovered via production quality regression.
**Recommendation.** Draft per-source contracts for every active corpus (section 2). Identify owner teams. Start with the highest-impact corpus.
**Owner.** ai-platform-eng + corpus-owning teams, sprint N+1 (drafting), N+2+ (rollout).

### ARCH-CORPUS-002 — Severity: Critical
**Finding.** Ingestion pipeline has no contract gate; non-conforming upstream content is silently ingested.
**Recommendation.** Add the contract-gate pipeline stage (section 4); quarantine non-conforming documents; alert the contract owner team.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-CORPUS-003 — Severity: High
**Finding.** Freshness SLOs are not declared per corpus; staleness is invisible until users observe outdated answers.
**Recommendation.** Define per-corpus freshness SLOs; instrument freshness timer; alert on SLO breach.
**Owner.** ai-platform-eng + corpus-owning teams, sprint N+2.

### ARCH-CORPUS-004 — Severity: High
**Finding.** Change protocol is undefined; upstream teams ship breaking changes without notice.
**Recommendation.** Establish change protocol per corpus (section 2.8). Document notice periods; build the notification channel.
**Owner.** ai-platform-eng + corpus-owning teams, sprint N+3.

### ARCH-CORPUS-005 — Severity: High
**Finding.** Contract violations are logged but not surfaced to upstream owner teams; alerts route only to AI team.
**Recommendation.** Route contract-violation alerts to the contract owner team; AI team gets CC for awareness.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ARCH-CORPUS-006 — Severity: High
**Finding.** Corpus ownership is unclear — no single team is accountable for upstream behavior.
**Recommendation.** Assign explicit owner team per corpus; document in the corpus catalog; surface owner contact in alerts.
**Owner.** ai-platform-eng team lead + corpus-owning teams, sprint N+1.

### ARCH-CORPUS-007 — Severity: High
**Finding.** Per-tenant content corpora (e.g., tenant-protocols) lack tenant-specific contracts; tenants do not know their freshness obligations.
**Recommendation.** Template-driven per-tenant contracts; include in customer onboarding; surface in the tenant admin UI.
**Owner.** ai-platform-eng + customer-success, sprint N+3.

### ARCH-CORPUS-008 — Severity: High
**Finding.** Lineage from source document → chunk → embedding → retrieval → answer is not preserved; cannot attribute user-facing claims back to source.
**Recommendation.** Implement provenance metadata propagation through the ingestion-and-retrieval pipeline; surface in user-facing citations and in audit logs.
**Owner.** ai-platform-eng, sprint N+2.
**Cross-link.** This repo: `lineage-and-provenance.md` (coming).

### ARCH-CORPUS-009 — Severity: Medium
**Finding.** Quarantine workflow does not exist; non-conforming documents either block the pipeline or are silently ingested.
**Recommendation.** Build quarantine staging; document the two-team triage workflow; runbook for contract-violation incidents.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-CORPUS-010 — Severity: Medium
**Finding.** Major contract changes have no documented notice period; AI team is surprised by upstream changes.
**Recommendation.** Establish 30-day notice for major changes, 7-day for minor; document in the change protocol; require notice in the change-management system.
**Owner.** ai-platform-eng + change-management, sprint N+3.

### ARCH-CORPUS-011 — Severity: Medium
**Finding.** Cross-corpus dashboard does not exist; operational state of corpora is not visible at a glance.
**Recommendation.** Build the dashboard (section 7.4) showing every corpus, owner, freshness, health.
**Owner.** ai-platform-eng + observability, sprint N+3.

### ARCH-CORPUS-012 — Severity: Medium
**Finding.** Contracts are documented but not versioned; changes are not traceable.
**Recommendation.** Contracts as code, semantically versioned; commit history is the change log.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-CORPUS-013 — Severity: Medium
**Finding.** Quarterly contract review is not scheduled; contracts drift from upstream reality.
**Recommendation.** Schedule quarterly review per active contract; assign reviewer pairs (AI + owner team); track review completion.
**Owner.** ai-platform-eng team lead, sprint N+4.

### ARCH-CORPUS-014 — Severity: Medium
**Finding.** Volume anomalies (sudden 10x growth, sudden 10x shrinkage) are not alerted.
**Recommendation.** Add volume-anomaly detection to the contract gate; alert on per-corpus growth-rate violations.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-CORPUS-015 — Severity: Medium
**Finding.** Body-content sanity checks (no empty bodies, no obviously garbled extraction) are not in place.
**Recommendation.** Add content-sanity checks to the contract gate; calibrate thresholds against historical content.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-CORPUS-016 — Severity: Medium
**Finding.** Corpus retirement process is undefined; deprecated sources accumulate.
**Recommendation.** Define the retirement lifecycle (section 6.4); maintain a retired-corpora list with archive locations.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-CORPUS-017 — Severity: Low
**Finding.** Per-corpus quality metrics (retrieval recall on the corpus, citation accuracy when citing it) are not separately reported.
**Recommendation.** Surface per-corpus quality dimensions in the eval dashboards; track trends.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-CORPUS-018 — Severity: Low
**Finding.** Upstream-side credential rotation is uncoordinated; ingest occasionally fails on expired credentials.
**Recommendation.** Document credential rotation cadence per corpus; align with upstream rotation; alert on impending expiration.
**Owner.** ai-platform-eng + security-eng, sprint N+5.

### ARCH-CORPUS-019 — Severity: Low
**Finding.** Known-issues documentation per corpus is absent; new ingest engineers re-discover the same upstream quirks.
**Recommendation.** Maintain known-issues list per corpus contract; update on incident; surface in the dashboard.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-CORPUS-020 — Severity: Low
**Finding.** Multi-tenant per-source contracts are authored bespoke per tenant rather than template-driven; maintenance overhead scales with tenant count.
**Recommendation.** Move per-tenant contracts to template-driven authoring; consolidate; tenant onboarding instantiates the template.
**Owner.** ai-platform-eng, sprint N+4.

---

## 10. Adoption sequencing checklist

For a team starting from "we have RAG sources but no contracts":

- [ ] **Sprint 0 — inventory.** List every active retrieval corpus. For each: source system, upstream owner team, current usage, current pain points. Surfaces what you're working with.
- [ ] **Sprint 0 — prioritize.** Rank corpora by impact (which one breaks would cause the worst production incident?). Tackle highest-impact first.
- [ ] **Sprint 1 — first contract.** Draft the contract for the highest-impact corpus. All nine elements (section 2). Get sign-off from the upstream owner team.
- [ ] **Sprint 2 — contract gate.** Implement the contract-gate stage in the ingestion pipeline for that one corpus. Quarantine + alert. Validate against real upstream traffic.
- [ ] **Sprint 2 — dashboard skeleton.** Surface the one corpus on a dashboard with health status. Even a single-row dashboard is useful.
- [ ] **Sprint 3 — second and third contracts.** Repeat the pattern for two more corpora. Refine the contract template based on what you learned.
- [ ] **Sprint 4 — change protocol.** Document and socialize the change protocol; integrate with the change-management system; agree the notification channel with upstream teams.
- [ ] **Sprint 5 — backfill remaining corpora.** Now that the pattern is proven, write contracts for the remaining corpora.
- [ ] **Sprint 6 — multi-tenant template.** For per-tenant corpora, build the contract template; onboard new tenants through it.
- [ ] **Ongoing — quarterly review.** Every quarter, review every active contract. Stale contracts get retired or updated; new corpora get contracted.

A team that completes this sequence has the corpus-as-product discipline that makes "upstream changes break production" a once-a-year incident instead of a once-a-week incident.

---

## 11. References

- Atlassian / Data Mesh community: data contracts as the central pattern.
- Chad Sanderson, *Data Contracts* (book and talks, 2024-2025) — canonical reference for the data-contracts pattern as applied to data warehousing.
- Andrew Jones, *Driving Data Quality with Data Contracts* — practical guide to contract authoring and enforcement.
- ASC's OpenLineage spec — for the lineage portion of data contracts.
- This repo: [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) — finding `ARCH-RAG-012` (corpus has no documented refresh SLO or owner team), which this document closes.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked architecture that depends on these contracts.
- This repo: [data-architecture-for-ai/lineage-and-provenance.md](./) (coming) for the source-attribution detail.
- This repo: [data-architecture-for-ai/freshness-architecture.md](./) (coming) for the freshness-vs-staleness architectural choices.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/ingestion-pipeline-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/) (coming) for the engineering practice of the ingestion pipeline that the contract gate sits in.
- Sibling repo: [ai-engineering-reference-architecture/data-engineering-for-ai/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/data-engineering-for-ai) for the broader data-engineering discipline.
