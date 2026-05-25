# Retrieval Scope Enforcement

> **Audience.** Architects designing the platform layer where retrieval-side guardrails attach. Tech leads worried about which controls live in retrieval code vs which live in the platform. **Scope.** The *architectural* pattern of the retrieval-client wrapper as the chokepoint for tenant scope, per-record scope, content-type scope, sensitivity scope, time scope, and audit logging. Pair with [tool-call-authorization.md](./tool-call-authorization.md) (the parallel pattern on the tool side) and [ai-gateway-pattern.md](./ai-gateway-pattern.md) (the parallel pattern on the LLM-call side). The per-tenant-vector-namespacing implementation detail lives in [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A retrieval-augmented AI system has three architectural chokepoints — the LLM-call layer (the AI gateway per [ai-gateway-pattern.md](./ai-gateway-pattern.md)), the tool-call layer (the tool registry per [tool-call-authorization.md](./tool-call-authorization.md)), and the retrieval layer (the retrieval-client wrapper, the subject of this document). Each chokepoint is the architectural surface where multiple guardrails attach. Without the chokepoint, the guardrails bolt onto calling code; with it, the guardrails are platform components that apply uniformly.

Retrieval-side guardrails are particularly important because retrieval is the surface where data leaves storage and enters the model's context. A misconfigured retrieval can:
- Return another tenant's documents (cross-tenant leak).
- Return documents the requesting user is not authorized to see (per-record scope failure).
- Return stale or expired content beyond its useful lifetime.
- Return sensitive content that should have been redacted.
- Return content from a corpus the requester is not subscribed to.

Each is an architectural concern, not a per-feature concern. Placing the enforcement in the platform's retrieval-client wrapper means every retrieval — chat-driven, tool-driven, background-job-driven, eval-driven — passes through the same controls.

The Care Coordinator's GA-blocker `ARCH-CARE-001` (tenant filter enforcement) is the most-cited example of why this pattern matters. This document is the architectural depth on the broader pattern.

This document is opinionated about three things:

1. **The retrieval-client wrapper is a platform component, not a library convenience.** It is the only path to retrieval, and it enforces every retrieval-side scope concern, not just tenancy.
2. **Scope dimensions compose.** Tenant, per-record, time, content-type, sensitivity — each is a dimension of the retrieval request. The wrapper enforces each; calling code does not.
3. **Audit attaches to the wrapper.** Every retrieval is logged uniformly. Audit-grade attribution is platform-provided, not per-feature-implemented.

Structure: (2) the chokepoint discipline; (3) the scope dimensions; (4) the wrapper interface; (5) integration with the tool registry; (6) audit logging; (7) failure modes and defense; (8) the wrapper's deployment shape; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The chokepoint discipline

The retrieval-client wrapper is the chokepoint. All retrieval flows through it. No exceptions.

### 2.1 What "chokepoint" means

- Application code does not import vector-store SDKs (pgvector connection, Pinecone client, Weaviate client, etc.) directly.
- Application code does not query retrieval-backing systems (Postgres FTS, OpenSearch, Elasticsearch) directly for retrieval purposes.
- The wrapper is the only public retrieval API.
- Lint rules and code review enforce the discipline.

### 2.2 Why one chokepoint

The alternative — guardrails per calling site — has predictable failure modes:

- Some calling sites have full guardrails (tenant filter, audit log, scope check).
- Some have partial guardrails (tenant filter but no audit).
- Some have no guardrails (the engineer who wrote the code forgot or did not know the requirement).
- New calling sites added later inherit no guardrails by default.

The chokepoint pattern produces uniform enforcement that survives engineering team turnover.

### 2.3 The relationship to the LLM gateway

The retrieval wrapper and the LLM gateway are parallel patterns. Both are chokepoints; both enforce; both audit. They are separate because:

- They guard different surfaces (data egress from storage vs model inference).
- They have different scope concerns (retrieval has per-record and content-type; LLM has model-version and tool-availability).
- They have different operational concerns (retrieval has freshness and source-attribution; LLM has cost and rate-limiting).

Some implementations combine them (one platform component handles both). This is fine architecturally; the conceptual separation matters because the enforcement of each is distinct.

### 2.4 The relationship to the tool registry

When retrieval is exposed to an agent as a tool, the tool registry (per [tool-call-authorization.md](./tool-call-authorization.md)) is the authorization surface for the tool call. The retrieval wrapper is the enforcement surface for the retrieval that the tool's implementation performs. Both fire:

- Tool registry: "is this caller authorized to call the `retrieve_clinical_guideline` tool?"
- Retrieval wrapper: "given that the tool was called, is the retrieval scoped correctly?"

Both layers are needed. The tool registry handles "can the agent ask?"; the retrieval wrapper handles "given the ask, what data is returned?"

---

## 3. The scope dimensions

The retrieval-client wrapper enforces multiple scope dimensions. Each is a check on the retrieval request.

### 3.1 Tenant scope

The retrieval can only see documents from the requesting tenant (or from explicitly-allowed cross-tenant scopes like global content).

Implementation per [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md). The wrapper:
- Resolves tenant from the call context.
- Injects mandatory tenant filter into the underlying query.
- Verifies post-retrieval that every returned chunk has the expected tenant attribution.
- Raises on tenant violation.

### 3.2 Per-record / per-resource scope

Some retrievals require the caller to have a relationship to the specific record. Examples:

- Clinical retrieval: the clinician must be on the care team for the patient whose records are being retrieved.
- Customer-data retrieval: the customer must own the record being retrieved.
- Project retrieval: the user must be a member of the project the document belongs to.

Implementation: the wrapper consults an authorization service (the existing identity / authorization model) to determine the caller's per-record permissions. The retrieval query is filtered to records the caller is authorized for.

This is finer-grained than tenant scope. Tenant scope says "this hospital's data only"; per-record scope says "this hospital's data and only patients on this clinician's care team."

### 3.3 Content-type / corpus scope

Different callers have access to different corpora. Examples:

- A clinical-decision tool can retrieve from the clinical guidelines corpus but not from the patient-message-templates corpus.
- A patient-facing assistant can retrieve from the patient-education corpus but not from the clinician-protocols corpus.
- An internal-tools agent can retrieve from internal documentation but not from any customer-tenant corpus.

Implementation: the wrapper accepts a `corpus` parameter (one or more); the call's caller-context determines which corpora that caller is authorized for; out-of-scope corpus requests are refused.

### 3.4 Time / freshness scope

Some retrievals require currency constraints. Examples:

- Only return documents updated within the last 30 days (for time-sensitive workloads).
- Only return documents published before a specific date (for historical research).
- Refuse retrieval if the corpus is staler than its freshness SLO.

Implementation: the wrapper accepts time-window parameters; corpus-freshness check verifies the corpus has been updated within its SLO; refuses on staleness.

### 3.5 Sensitivity scope

Some retrievals need to exclude sensitive content. Examples:

- Patient-facing retrieval excludes documents flagged as clinician-only.
- Public-facing retrieval excludes documents flagged as internal or confidential.
- Cross-tenant analytics retrieval excludes documents flagged as patient-identifiable.

Implementation: documents carry sensitivity classifications; the wrapper applies the caller's sensitivity authorization to filter.

### 3.6 The scope composition

A single retrieval request has all five dimensions. The wrapper:

1. Resolves tenant scope from call context.
2. Looks up per-record scope from the authorization service.
3. Verifies content-type scope.
4. Applies time / freshness scope.
5. Applies sensitivity scope.
6. Constructs the underlying retrieval query with all scopes applied.
7. Post-retrieval, verifies each returned document satisfies all scopes (defense-in-depth).

The composition is mandatory. The wrapper does not allow "scope-free" retrieval as a convenience; every call has fully-resolved scope.

---

## 4. The wrapper interface

The wrapper exposes a single public API. The shape:

### 4.1 The signature

```python
class RetrievalClient:
    def retrieve(
        self,
        query: str,
        *,
        # Required scope dimensions
        corpus: str,                              # which corpus to query
        context: CallContext,                     # tenant, user, feature, session, trace
        # Optional scope dimensions
        time_window: TimeWindow | None = None,    # freshness / historical constraints
        sensitivity_level: int | None = None,     # ceiling on sensitivity
        # Retrieval parameters (not scope)
        top_k: int = 10,
        retriever_type: str = "hybrid",
        rerank: bool = True,
    ) -> RetrievalBundle:
        ...
```

The required parameters are non-optional. The wrapper enforces every scope dimension before performing the retrieval.

### 4.2 The wrapper's responsibilities

- Resolve all scope dimensions from the call context.
- Construct the scope-filtered query.
- Dispatch to the underlying retriever (BM25, vector, hybrid, graph).
- Post-retrieval verification.
- Apply reranking (if requested).
- Lineage / provenance attribution (per [lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) coming).
- Audit logging.
- Span emission.

What the wrapper does NOT do:

- Embed the query (the LLM-call wrapper handles embedding calls).
- Choose the retrieval pattern (the calling code chooses naive vs hybrid vs agentic; the wrapper executes).
- Format retrieved content into prompts (the prompt-assembly layer does that).
- Cache retrieval results (the caching layer does, transparently to the wrapper).

### 4.3 The "only path" enforcement

- The underlying vector-store SDK is private to the wrapper package.
- Application code that imports the SDK directly is rejected at code review.
- A lint rule (custom AST check) detects direct SDK imports outside the wrapper package and flags them.
- The wrapper does not have a "skip scope" parameter; legitimate cross-scope retrievals use a separate `CrossScopeRetrievalClient` with explicit audit.

### 4.4 The response shape

```python
class RetrievalBundle:
    results: list[RetrievalResult]
    # Per-result attribution
    # Each result has: chunk content, doc ID, scores, source metadata, scope tags

    metadata: RetrievalMetadata
    # Aggregate: corpus version, retrieval latency, scope checks passed, lineage tokens
```

Consumers see a uniform response shape regardless of underlying retriever. The metadata supports downstream lineage attribution (the model's answer can cite source documents).

---

## 5. Integration with the tool registry

When the agent exposes retrieval as a tool, the tool's implementation calls the retrieval client wrapper. Both layers fire.

### 5.1 The tool-implementation pattern

```python
@registry.register(
    name="retrieve_clinical_guideline",
    class_="read-only",
    arguments=ClinicalRetrievalArguments,
    authorization=AuthorizationSpec(
        required_roles=["rn", "ma", "care_coordinator", "physician"],
        tenant_scope="per_request",
    ),
)
def retrieve_clinical_guideline(args, ctx):
    bundle = retrieval_client.retrieve(
        query=args.query,
        corpus="clinical-guidelines",
        context=ctx.caller_call_context,
        top_k=args.top_k or 10,
    )
    return bundle
```

The tool registry handles "can the agent call this tool?"; the retrieval wrapper handles "given the call, what is the scoped result?"

The two-layer pattern has belt-and-suspenders properties:
- A misconfigured tool authorization (the tool was registered with permissive roles) is caught at the retrieval-scope layer if the requesting context lacks per-record scope.
- A misconfigured retrieval scope (the wrapper has a bug) is caught at the tool authorization layer if the agent should not have been allowed to call the tool at all.

### 5.2 Cross-tool retrieval consistency

Multiple tools may call the same underlying retrieval (e.g., `retrieve_clinical_guideline` and `lookup_drug_interaction` both query clinical-knowledge sources). The wrapper provides consistency:
- Both tools' retrievals use the same scope-enforcement logic.
- Both produce comparable audit records.
- The wrapper's response shape lets the consuming tools format consistently.

---

## 6. Audit logging

Every retrieval produces an audit entry. The entry supports incident investigation, compliance attestation, and quality analysis.

### 6.1 What gets logged

Per retrieval:

- Trace ID + parent context (so retrieval can be tied to the broader interaction).
- Caller identity (user, role).
- Tenant.
- Feature initiating the call.
- Query (or query hash for sensitive workloads).
- Corpus.
- Scope dimensions applied (tenant, per-record, time, sensitivity).
- Retriever type and parameters.
- Returned doc IDs and scores.
- Lineage tokens.
- Latency.
- Scope-verification result (all passed; or specific failure).

### 6.2 Audit log scope

- **Routine retrievals** (read-only, no PHI) go to the standard observability stack.
- **Sensitive retrievals** (PHI-containing corpus, regulated content) go to a dedicated audit sink with stricter retention.
- **Scope-violation events** (a returned chunk has wrong tenant tag) are high-priority alerts.

### 6.3 Lineage integration

The audit log integrates with the lineage / provenance pattern per [lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) (coming). When the model's answer cites a source document, the lineage trail goes: answer → retrieval audit entry → source document. This is what supports the user-facing citation feature and the regulatory audit story.

### 6.4 Cross-tenant operations

Cross-tenant operations (admin analytics, platform-wide queries) require a separate audit posture:

- A `CrossScopeRetrievalClient` (distinct from the regular client) is used.
- Every call is explicitly authorized by an admin role.
- The audit entry includes the business justification.
- The cross-scope audit is reviewed periodically.

---

## 7. Failure modes and defense

The retrieval-scope-enforcement pattern is layered defense. Each layer catches different failure modes.

### 7.1 The defense layers

| Layer | Catches |
|---|---|
| Application code review | Code paths that bypass the wrapper |
| Lint rule | Direct SDK imports outside the wrapper package |
| Wrapper interface | Missing scope parameters (call refused) |
| Wrapper logic | Scope construction bugs |
| Post-retrieval verification | Underlying-store bugs that return out-of-scope data |
| Storage layer | Per-tenant namespacing as defense in depth |
| Audit log | Forensic evidence after-the-fact |
| Canary tests | Drift in scope enforcement detected in CI |

Each layer is independent. A failure in one is caught by another.

### 7.2 The canary tests

A canary test suite (per [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) section 6) plants known-distinct content in two tenants' scope and verifies the cross-scope retrievals never return the planted content. The tests run:

- On every PR to the wrapper or to retrieval-touching code.
- Nightly against staging.
- Quarterly against production (with a controlled synthetic canary tenant).

### 7.3 The drift-detection pattern

Production audit logs are scanned periodically for anomalies:
- Any retrieval where post-retrieval verification raised but was suppressed.
- Any retrieval that returned chunks with unexpected tenant tags.
- Any retrieval where the scope dimensions differed from the expected pattern for the caller class.

Anomalies trigger investigation. The pattern detects drift before incidents.

---

## 8. The wrapper's deployment shape

Like the AI gateway, the retrieval wrapper can be deployed as a library, sidecar, or centralized service.

### 8.1 Library (in-process)

The wrapper is a library that application code imports. No separate service. Most common shape; lowest latency.

For Meridian: this is the deployment shape.

### 8.2 Sidecar

The wrapper runs as a sidecar process. Application code calls the local sidecar over HTTP / gRPC.

Use when: multi-language platform; team is comfortable with sidecar patterns.

### 8.3 Centralized service

The wrapper is its own service. Application code calls it remotely.

Use when: large platform with many AI feature teams; strong centralized platform team.

The choice is the same one made for the AI gateway. Often the same shape is right for both.

---

## 9. Worked Meridian Health example

### 9.1 The wrapper component

`meridian.retrieval` is a Python package. The `RetrievalClient` class is the only public interface. The package owns connections to pgvector, Postgres FTS, and the drug-interaction graph.

### 9.2 A typical call

The Care Coordinator's clinical-knowledge worker invokes retrieval:

```python
bundle = retrieval_client.retrieve(
    query="What is the post-discharge follow-up protocol for a CHF patient on the new pathway?",
    corpus="clinical-guidelines-and-protocols",  # corpus + tenant-protocols merged view
    context=CallContext(
        tenant_id="mercy-cleveland",
        user_id="u-mercy-cleveland-rn-001234",
        user_role="rn",
        feature_id="care-coordinator",
        session_id="session-2026-05-25-a7b8-3",
        interaction_id="interaction-2026-05-25-a7b8",
    ),
    top_k=10,
    retriever_type="hybrid",
    rerank=True,
)
```

The wrapper:
1. Resolves tenant=mercy-cleveland. Adds tenant filter (`tenant_id IN ('mercy-cleveland', 'global-guidelines')`).
2. Per-record scope: clinical retrieval is corpus-level, no per-patient scoping needed for this query. (Patient-context retrievals would scope by care-team membership.)
3. Content-type: confirms `care-coordinator` feature is authorized for `clinical-guidelines-and-protocols` corpus.
4. Time scope: no constraint specified; uses default freshness window.
5. Sensitivity: clinical context allows clinician-level content.
6. Constructs the hybrid query (BM25 + vector + filter). Runs.
7. Reranks via Cohere.
8. Post-retrieval: verifies every returned chunk has tenant_id in the allowed set.
9. Constructs the response bundle with per-result lineage tokens.
10. Audit-logs the call.
11. Returns the bundle.

### 9.3 The tool-registry integration

The clinical-knowledge worker accesses retrieval via a tool (`retrieve_clinical_guideline`). The tool registry authorizes; the tool implementation calls the wrapper.

A non-authorized caller (e.g., a tool call from a feature that shouldn't have access to clinical content) would be refused at the tool-registry layer. A scope misconfiguration that slipped past the tool registry (e.g., a tool registered too permissively) would be caught at the wrapper's per-record-scope check.

### 9.4 The canary suite in production

The Meridian canary suite (per [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) section 7.6) runs against the wrapper. The 38-case suite has been in CI for 8 quarters; has caught 4 cross-tenant regressions in pre-production code reviews (each was a developer attempting a code path that bypassed the wrapper).

### 9.5 Audit log examples

A routine retrieval audit entry:

```yaml
event_id: rt-2026-05-25-x9a2
trace_id: t-2026-05-25-3a4b
caller: u-mercy-cleveland-rn-001234
caller_role: rn
tenant: mercy-cleveland
feature: care-coordinator
query_hash: 9c2a1f8b...
corpus: clinical-guidelines-and-protocols
scope_dimensions:
  tenant: enforced
  per_record: not_applicable
  content_type: passed
  time_window: passed
  sensitivity: passed
retriever: hybrid_with_rerank
returned_count: 10
returned_doc_ids: [doc-c1, doc-c2, doc-p1, ...]
lineage_tokens: [...]
latency_ms: 187
scope_verification: all_passed
```

When a scope violation fires (post-retrieval verification catches an unexpected tenant), the audit entry has `scope_verification: tenant_violation` and the entry is routed to high-priority alerting.

### 9.6 Cross-scope retrieval

The platform analytics team occasionally needs cross-tenant aggregate queries. The `CrossScopeRetrievalClient` is the path:

- Used only by platform-eng + data-eng with explicit authorization.
- Every call requires a business justification recorded in the audit entry.
- The audit entries are reviewed quarterly by security-eng + compliance.
- Application code cannot import the `CrossScopeRetrievalClient`; access is limited to specific package imports under platform-team control.

### 9.7 The platform discipline

- `meridian.retrieval` is the only path. Lint rule rejects direct vector-store SDK imports outside the package.
- Canary suite gates every PR touching `meridian.retrieval/*`.
- Audit log retention is 7 years (HIPAA-aligned).
- Quarterly: scope-policy review, cross-scope audit review, anomaly review.

---

## 10. Anti-patterns

### 10.1 "Application code does its own scope enforcement"

Every calling site implements its own tenant filter, its own per-record check. The pattern is convention; the convention degrades over team turnover.

**Corrective.** Wrapper enforces. Calling code passes context; the wrapper does the work.

### 10.2 "Skip scope flag"

The wrapper has a `skip_scope=True` parameter for "admin operations." Convenience for engineering; isolation hazard.

**Corrective.** No skip flag. Cross-scope operations use a separate authorized client with explicit audit.

### 10.3 "Scope at wrapper but not at storage"

The wrapper enforces; the storage layer (vector store) has no namespacing. A wrapper bug exposes all tenants' data.

**Corrective.** Defense in depth — storage-layer namespacing (per-tenant namespace, per-tenant index, or per-tenant partition).

### 10.4 "Post-retrieval verification is optional"

The wrapper trusts the storage layer to enforce; no post-retrieval check. A storage-layer bug returns cross-scope data without detection.

**Corrective.** Mandatory post-retrieval verification. Returned chunks are checked against the requested scope; mismatch raises.

### 10.5 "Sensitivity scope not enforced"

The corpus has sensitivity classifications, but the wrapper does not consult them. A caller authorized for a corpus retrieves any document in it regardless of sensitivity flag.

**Corrective.** Sensitivity is a scope dimension; the wrapper filters accordingly; per-caller sensitivity authorization is in the platform's identity model.

### 10.6 "Audit logs are routine logs"

Retrieval audit goes to the standard application log. Retention is the log infrastructure's default. Access controls are operational, not compliance-grade.

**Corrective.** Dedicated audit sink for sensitive workloads. Retention aligned with regulatory requirement. Access controls aligned with the data sensitivity.

### 10.7 "Cross-scope retrieval is the regular client with a flag"

The same client serves both per-tenant retrievals and cross-tenant analytics. The latter requires the operator to remember to set the flag; mistakes happen.

**Corrective.** Separate `CrossScopeRetrievalClient` with explicit authorization, separate audit, explicit business-justification capture.

### 10.8 "Canary tests are absent"

No automated tests verify that cross-scope retrievals are blocked. The team trusts the wrapper's logic without verification.

**Corrective.** Canary suite per the per-tenant-vector-namespacing pattern. Gates PR merge and pre-release.

---

## 11. Findings (sprint-assignable)

### ARCH-SCOPE-001 — Severity: Critical
**Finding.** Retrieval-client wrapper does not exist; application code accesses vector store directly.
**Recommendation.** Build the wrapper per section 4; lint rule against direct SDK imports.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-SCOPE-002 — Severity: Critical
**Finding.** Tenant scope enforcement is in application code, not in a wrapper; one missing filter is one breach.
**Recommendation.** Move tenant enforcement to the wrapper per section 3.1; post-retrieval verification per section 3.6 step 7.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-SCOPE-003 — Severity: Critical
**Finding.** Post-retrieval scope verification is not implemented; underlying-store bugs can return out-of-scope data without detection.
**Recommendation.** Post-retrieval verification as a wrapper step; raise on mismatch; alert as Sev-1.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-SCOPE-004 — Severity: Critical
**Finding.** No canary test suite verifies cross-scope retrieval is blocked.
**Recommendation.** Canary suite per [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) section 6; gate PR merge.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-SCOPE-005 — Severity: High
**Finding.** Per-record scope enforcement is not implemented; users authorized for a tenant can retrieve any record in that tenant regardless of relationship.
**Recommendation.** Per-record scope per section 3.2; integration with the authorization service.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-SCOPE-006 — Severity: High
**Finding.** Content-type / corpus scope enforcement is implicit; features can retrieve from corpora they should not have access to.
**Recommendation.** Corpus authorization per section 3.3; wrapper refuses out-of-scope corpus requests.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-SCOPE-007 — Severity: High
**Finding.** Cross-scope retrieval is performed through the regular client with a `skip_scope` parameter; abuse and accidents are possible.
**Recommendation.** Separate `CrossScopeRetrievalClient` per section 6.4; explicit authorization; separate audit.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-SCOPE-008 — Severity: High
**Finding.** Sensitivity scope enforcement is not implemented; sensitive documents are retrievable by all authorized callers.
**Recommendation.** Sensitivity scope per section 3.5; per-caller sensitivity authorization.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### ARCH-SCOPE-009 — Severity: High
**Finding.** Audit log does not include scope-dimension verification results; investigation requires inferring from indirect signals.
**Recommendation.** Explicit scope-verification field in audit entry per section 9.5.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-SCOPE-010 — Severity: High
**Finding.** Time / freshness scope is not enforced; stale content is served past corpus freshness SLO.
**Recommendation.** Time scope per section 3.4; corpus-freshness check integrated with the data-contracts pattern.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SCOPE-011 — Severity: Medium
**Finding.** Tool implementations that call retrieval bypass the wrapper (call SDK directly).
**Recommendation.** Tool implementations call the wrapper; lint rule against direct SDK imports in tool code.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SCOPE-012 — Severity: Medium
**Finding.** Background jobs (eval, batch processing, re-indexing) do not propagate caller context; default scope is applied incorrectly.
**Recommendation.** Job entrypoints accept context as required parameter; no default-scope fallbacks.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SCOPE-013 — Severity: Medium
**Finding.** Audit log retention is 30 days; regulatory requirement is 7 years.
**Recommendation.** Dedicated audit sink with retention aligned to regulation.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+3.

### ARCH-SCOPE-014 — Severity: Medium
**Finding.** Lineage tokens are not produced; cited documents cannot be traced back through the retrieval chain.
**Recommendation.** Lineage tokens per [lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md); attribute on every result.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SCOPE-015 — Severity: Medium
**Finding.** Drift-detection scans of production audit logs are not scheduled.
**Recommendation.** Periodic anomaly scans per section 7.3; integrate with monitoring.
**Owner.** ai-platform-eng + sre, sprint N+4.

### ARCH-SCOPE-016 — Severity: Low
**Finding.** Wrapper deployment shape is undocumented; new engineers cannot reason about latency / failure modes.
**Recommendation.** Document the deployment shape (library, sidecar, service) per section 8.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-SCOPE-017 — Severity: Low
**Finding.** Cross-scope audit review is not scheduled.
**Recommendation.** Quarterly cross-scope audit review with security-eng + compliance.
**Owner.** ai-platform-eng team lead, sprint N+4.

### ARCH-SCOPE-018 — Severity: Low
**Finding.** Wrapper does not expose retrieval observability uniformly; some retrievals emit rich spans, others minimal.
**Recommendation.** Wrapper-emitted retrieval span has the full attribute set per [trace-and-span-design.md](../../ai-engineering-reference-architecture/observability-and-telemetry/trace-and-span-design.md).
**Owner.** ai-platform-eng, sprint N+4.

---

## 12. Adoption sequencing checklist

For a team without retrieval scope enforcement:

- [ ] **Sprint 0 — design.** Decide the wrapper's interface (section 4). Inventory existing retrieval paths.
- [ ] **Sprint 1 — wrapper.** Build the chokepoint with tenant scope enforcement and post-retrieval verification.
- [ ] **Sprint 1 — canary suite.** Build the cross-scope canary tests; gate CI.
- [ ] **Sprint 2 — per-record and content-type scope.** Add these scope dimensions; integrate with the authorization service.
- [ ] **Sprint 2 — audit log.** Dedicated audit sink for sensitive workloads; structured entries.
- [ ] **Sprint 3 — migrate calling sites.** All retrieval calls go through the wrapper; lint rule against bypass.
- [ ] **Sprint 3 — cross-scope client.** Separate `CrossScopeRetrievalClient` for legitimate cross-scope operations; security-eng review.
- [ ] **Sprint 4 — time / sensitivity scope.** Add these dimensions where the workload requires.
- [ ] **Sprint 4 — drift detection.** Periodic audit log scans; alerting on anomalies.
- [ ] **Sprint 5 — observability.** Uniform retrieval spans with full attribute set.
- [ ] **Ongoing — discipline.** Quarterly scope-policy review; quarterly cross-scope audit review.

A team that completes this sequence has the architectural pattern that turns retrieval-side enforcement from convention into platform property. A team that defers carries the per-site-enforcement tax forever.

---

## 13. References

- HIPAA Security Rule §164.312 — the technical-safeguards requirements that this pattern's audit and access-control support.
- SOC 2 Common Criteria CC6 — the access-control framework.
- This repo: [guardrails-and-policy-architecture/ai-gateway-pattern.md](./ai-gateway-pattern.md) — the parallel pattern for LLM calls.
- This repo: [guardrails-and-policy-architecture/tool-call-authorization.md](./tool-call-authorization.md) — the parallel pattern for tool calls.
- This repo: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — the storage-layer implementation pattern.
- This repo: [multi-tenancy-and-isolation/isolation-models.md](../multi-tenancy-and-isolation/isolation-models.md) — the isolation-model framework this enforcement supports.
- This repo: [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) — the lineage pattern the wrapper produces.
- This repo: [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md) — the corpus-as-contract pattern that complements scope enforcement.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — ARCH-CARE-001 is the cross-link finding this document closes architecturally.
- Sibling repo: [ai-engineering-reference-architecture/rag-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/rag-engineering) — the engineering practice for the retrieval pipelines this wrapper sits in front of.
- Sibling repo: [ai-security-reference-architecture/llm-application-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) — the broader threat model for retrieval-side attacks.
