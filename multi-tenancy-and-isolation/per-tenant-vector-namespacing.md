# Per-Tenant Vector Namespacing

> **Audience.** Architects designing multi-tenant AI systems where retrieval surfaces are shared across tenants. Engineers reviewing whether an existing system's tenant isolation will hold under audit. **Scope.** The architectural pattern decision for vector-store tenant isolation: per-tenant namespace vs per-tenant index vs shared-index-with-mandatory-filter. Not the data architecture choice between vector store products (that lives in [data-architecture-for-ai/vector-store-architecture.md](../data-architecture-for-ai/) when written). **Worked client.** Meridian Health, the fictional regulated healthcare SaaS. **Companion docs.** [isolation-models.md](./) for the overall multi-tenancy framework (coming); [cross-tenant-leakage-prevention.md](./) for the layered defense across non-vector surfaces (coming).

---

## 1. Why this document exists

The Care Coordinator GA blocker `ARCH-CARE-001` ("tenant filter in the retrieval client is enforced by application code rather than as a wrapper-mandatory constraint; a code path can bypass it") is the most common multi-tenant-AI failure mode in 2026. A single missing filter — a forgotten `WHERE tenant_id = ?` clause, a code path that bypasses the retrieval wrapper, an admin endpoint that does not inject tenant context — and tenant A's clinical documents surface in tenant B's retrieval results.

The traditional defense ("be careful in code") does not survive contact with a real engineering team. New endpoints get added. Refactors move retrieval calls. The pattern that *works in production* is to push tenant isolation into the *retrieval architecture itself*, so that the only way to query is the way that enforces tenancy. Application code does not "remember" to filter; it cannot do otherwise.

This document is opinionated about three things:

1. **The retrieval client is the chokepoint.** All vector queries flow through a single wrapper; tenant context is a required parameter; the wrapper injects it, verifies it post-retrieval, and logs it.
2. **The isolation model is a deliberate architectural commitment**, not a default. Per-tenant namespace, per-tenant index, and shared-with-filter have different cost, throughput, and blast-radius profiles. Pick on signal, not on guess.
3. **The testing discipline is non-optional.** A test that exercises every code path against a planted "tenant B data should not appear in tenant A query" canary is the only reliable defense against missing-filter regressions.

The structure: (2) the question framework; (3) the three isolation models; (4) the retrieval-client wrapper pattern; (5) defense in depth; (6) testing patterns; (7) worked Meridian example; (8) anti-patterns; (9) sprint-assignable findings; (10) adoption checklist.

---

## 2. The questions to answer before picking an isolation model

Four questions. Answer in writing before reading the catalogue.

1. **How many tenants?** A handful (under 10) of high-value tenants is a different problem from hundreds (Meridian's 240 hospitals) is a different problem from millions (consumer SaaS). The economics flip across these ranges.
2. **What is the per-tenant data volume?** Tenants with similar volumes are easier to isolate uniformly. Tenants with wildly different volumes (one is 1000x larger than the median) need either dedicated infrastructure for the outlier or a sharding plan.
3. **What is the regulatory / contractual posture?** HIPAA / financial / public-sector workloads have explicit tenant-isolation obligations. Some customer contracts require dedicated infrastructure as a procurement-level commitment, independent of the technical isolation model.
4. **What is the operational team's appetite?** Per-tenant index is cleaner architecturally and more operationally expensive (more indexes to manage, more capacity planning, more failure modes). Shared-with-filter is cheaper and has a larger blast radius on misconfiguration. The right answer depends on what the team can operate.

Most workloads fall into one of three modal answers:
- Few high-value tenants + regulated → **per-tenant index**.
- Many tenants + regulated + moderate volume → **per-tenant namespace** (the Meridian default).
- Many tenants + lower regulatory bar → **shared-index-with-mandatory-filter**.

---

## 3. The three isolation models

### 3.1 Model A: Per-tenant index

**Shape.** Every tenant has its own physical index (separate Pinecone index, separate Postgres database with its own pgvector instance, separate Weaviate collection). Queries route by tenant ID to the right index. There is no shared data path that could leak across tenants.

```
┌──────────────┐   tenant=A    ┌─────────────────┐
│  query       │ ───────────→  │  Index A        │
│  (tenant=A)  │               │  (only A data)  │
└──────────────┘               └─────────────────┘

┌──────────────┐   tenant=B    ┌─────────────────┐
│  query       │ ───────────→  │  Index B        │
│  (tenant=B)  │               │  (only B data)  │
└──────────────┘               └─────────────────┘
```

**Sweet-spot workload.** Few high-value tenants (typically <50). Strict regulatory or contractual isolation requirements. Tenants with very different scale profiles. The Meridian "dedicated-infrastructure" premium customer is on this model.

**What it gives you.**
- **Strongest isolation.** Cross-tenant leak is physically impossible at the storage layer. A missing filter in application code does not produce cross-tenant retrieval, because the wrong index was never connected.
- **Per-tenant capacity isolation.** One tenant's traffic spike cannot starve another's queries.
- **Per-tenant operational levers.** Re-index this tenant without touching others. Tune index parameters per-tenant. Roll out vector-store version changes per-tenant.
- **Simplest audit story.** "This tenant's data is in this physical index" is easier to attest than "this tenant's data is logically isolated by metadata."

**What it costs.**
- **Operational overhead per tenant.** Each tenant is an index to provision, monitor, capacity-plan, patch, back up. At 240 tenants this becomes a full-time platform job.
- **Cost overhead.** Each tenant pays a base operational cost for its index whether it has 100 documents or 100,000. Many vector store products have per-index minimums.
- **Connection management.** The application connects to one of N indexes per request based on tenant. Connection pooling and routing become non-trivial.
- **Cross-tenant operations are awkward.** Platform-level work (an upgrade to the embedding model that requires reindex) becomes "do this 240 times" rather than "do this once."

**Cost shape.** Highest per-tenant cost floor. Best per-tenant cost ceiling (small tenants pay only their own usage).

### 3.2 Model B: Per-tenant namespace inside a shared index

**Shape.** All tenants share a single physical index, but the index supports a tenant-namespace concept (Pinecone namespaces, Weaviate multi-tenancy, Qdrant collections-as-namespaces, pgvector with a tenant column + partition). Queries specify the namespace; the index returns results only from that namespace.

```
┌──────────────────────────────────────────────────────────┐
│                  Single physical index                    │
│                                                           │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│   │ namespace  │ │ namespace  │ │ namespace  │   . . .   │
│   │ tenant=A   │ │ tenant=B   │ │ tenant=C   │           │
│   └────────────┘ └────────────┘ └────────────┘           │
└──────────────────────────────────────────────────────────┘
        ▲              ▲              ▲
        │              │              │
   query (A)      query (B)      query (C)
```

**Sweet-spot workload.** Many tenants (tens to thousands). Moderate per-tenant volume. Regulated but not requiring physical isolation. The Meridian Care Coordinator default for the 240 standard-tier hospital tenants.

**What it gives you.**
- **Strong logical isolation.** The vector store enforces namespace-scoping at the query level. A query against namespace A cannot return vectors from namespace B even if the metadata filter were missing — the namespace is part of the query, not just a filter.
- **Shared operational footprint.** One index to operate. Platform-level upgrades happen once. Backups, monitoring, capacity planning are unified.
- **Per-tenant operational levers (limited).** Most products allow per-namespace metrics; some allow per-namespace policy (retention, replication). Granularity is less than per-tenant-index.
- **Better cost economics.** Small tenants pay their share of the shared infrastructure rather than a per-index minimum.

**What it costs.**
- **Shared-index failure modes affect everyone.** A bug in the index, an outage, a rate-limit-exceeded event affects all tenants simultaneously.
- **Per-tenant capacity is fuzzier.** Most products do not provide hard per-namespace capacity isolation. Noisy-neighbor mitigation is a separate engineering investment.
- **Slightly weaker audit story.** "Tenant data is logically isolated by namespace" requires explaining what "namespace" means; "tenant data is in its own physical index" does not.

**Cost shape.** Best total cost across many tenants. Per-tenant cost scales with usage rather than starting from a base floor.

### 3.3 Model C: Shared index with mandatory metadata filter

**Shape.** A single shared index. Every vector is annotated with `tenant_id` (and optionally other partition keys). Every query must include a `WHERE tenant_id = ?` clause. There is no namespace primitive; the filter is the only isolation.

```
┌──────────────────────────────────────────────────────────┐
│                  Single physical index                    │
│   ┌─────────────────────────────────────────────────┐    │
│   │  vector + metadata{tenant_id, doc_id, ...}      │    │
│   │  vector + metadata{tenant_id, doc_id, ...}      │    │
│   │  vector + metadata{tenant_id, doc_id, ...}      │    │
│   │  ...                                             │    │
│   └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
                          ▲
                          │
              query + FILTER tenant_id = X
              (must include filter or sees everything)
```

**Sweet-spot workload.** Workloads where the regulatory bar is lower (consumer SaaS without per-user data-residency requirements, internal-tools shared across teams without isolation contracts), or workloads where tenants are conceptual partitions rather than legally-distinct entities.

**What it gives you.**
- **Cheapest.** No per-tenant operational floor. No namespace primitive overhead. Simplest infrastructure footprint.
- **Most flexible.** Cross-tenant queries are possible when needed (admin tools, platform analytics) because the filter is at the query level.

**What it costs.**
- **Highest blast radius on misconfiguration.** A query that forgets the filter sees everything. Code review and testing have to catch every such case; one missed case is one cross-tenant leak.
- **Weakest audit story.** Isolation is by query convention, not by storage architecture. Auditors and customer compliance teams find this hard to attest.
- **No per-tenant operational levers.** Everything is shared.

**Cost shape.** Lowest total cost. Highest risk cost (incident cost when a leak happens is meaningful).

### 3.4 The decision

| Tenant count | Regulatory bar | Per-tenant volume variance | Recommended model |
|---|---|---|---|
| < 10 | Strict | Any | Per-tenant index |
| 10–100 | Strict | Low | Per-tenant namespace |
| 10–100 | Strict | High (one tenant >> others) | Per-tenant namespace for most + per-tenant index for outliers |
| 100–1000 | Regulated (HIPAA, SOC 2, financial) | Low | Per-tenant namespace |
| 100–1000 | Lower | Low | Shared with mandatory filter |
| > 1000 | Any | Any | Per-tenant namespace, with sharding plan |

The Meridian Care Coordinator sits in the "100–1000 / regulated / low variance" cell → per-tenant namespace. The premium-customer dedicated-infrastructure variant moves that one customer into "per-tenant index" (which is the contracted commitment the customer paid for).

---

## 4. The retrieval-client wrapper pattern

Whichever isolation model is chosen, the load-bearing engineering pattern is the same: **all vector queries go through a single retrieval-client wrapper that takes tenant context as a non-optional parameter, injects it into the underlying query, verifies it post-retrieval, and logs the call.**

This is the surface that turns "the developer remembered to filter" into "the developer cannot query without filtering."

### 4.1 Wrapper interface

The wrapper exposes a single retrieve function. Tenant context is a required argument. There is no overload, no default, no convenience method that omits it.

```python
class RetrievalClient:
    def __init__(self, vector_store, index_name):
        self._store = vector_store
        self._index = index_name

    def retrieve(
        self,
        query: str,
        tenant_id: str,                # REQUIRED — no default
        top_k: int = 10,
        additional_filters: dict | None = None,
        trace_id: str | None = None,
    ) -> list[RetrievalResult]:
        if not tenant_id:
            raise ValueError("tenant_id is required")

        # Embed query
        query_vector = self._embed(query)

        # Per-tenant-namespace model: route to the right namespace
        # Per-tenant-index model: route to the right index
        # Shared-with-filter model: inject the filter
        results = self._store.query(
            index=self._resolve_index(tenant_id),
            namespace=self._resolve_namespace(tenant_id),
            vector=query_vector,
            top_k=top_k,
            filter={**self._tenant_filter(tenant_id), **(additional_filters or {})},
        )

        # Post-retrieval verification (defense in depth)
        for r in results:
            if r.metadata.get("tenant_id") != tenant_id:
                self._log_tenant_violation(tenant_id, r, trace_id)
                raise TenantIsolationViolation(
                    f"Retrieved result has tenant_id={r.metadata.get('tenant_id')} "
                    f"but request was for tenant_id={tenant_id}"
                )

        self._log_call(tenant_id, query, results, trace_id)
        return results
```

Three things to notice:

- **`tenant_id` is required and has no default.** It is impossible to call this function and forget the tenant.
- **Post-retrieval verification.** Every returned result is checked against the requested tenant. If any result has a different tenant, the function raises — and the violation is logged separately as a high-severity signal.
- **Per-call logging.** Tenant, query, retrieved doc IDs, trace ID. The trail that supports incident investigation.

### 4.2 The platform discipline

The wrapper is only effective if it is the *only* path to the vector store. Two disciplines support this:

1. **The underlying vector-store client is private to the wrapper.** No application code imports the vector-store SDK directly. Code review rejects any PR that does.
2. **A lint rule (custom AST check, or a simple grep in CI) flags any direct import of the vector-store SDK in application code.** The lint rule is gated; new violations cannot land.

This is the "make the right path the only path" pattern. Application code that wants to retrieve calls the wrapper; the wrapper enforces tenant isolation; there is no alternative.

### 4.3 What the wrapper should NOT do

- **Do not have a "skip tenant filter" parameter.** Even for "admin tools." Admin tools that need cross-tenant access should use a separate, audited, explicitly-authorized admin-retrieval client — not a flag on the normal client.
- **Do not have a "default tenant" fallback.** Even for "system-wide content." System-wide content is a separate tenant ID (the Meridian Care Coordinator uses `tenant_id='global-guidelines'` for the clinical guidelines that all hospitals share); the wrapper still enforces tenant context, just with that ID.
- **Do not infer tenant from a global context.** Some frameworks make a "current tenant" available globally. Using that global to infer tenant inside the wrapper means a code path that runs outside the framework (a batch job, a CLI tool) does not have the global set and may default to no-filter. Pass tenant explicitly.

---

## 5. Defense in depth

The wrapper is necessary but not sufficient. Three additional layers prevent isolation failures that bypass the wrapper, occur inside the wrapper, or are introduced as the system evolves.

### 5.1 Layer 1: Storage-layer guarantees

Wherever the vector store supports it:

- **Per-tenant-index model.** The application connects to a specific index per request, and the index physically contains only that tenant's data. A wrong connection is a wrong index, not a wrong filter.
- **Per-tenant-namespace model.** The vector store's namespace primitive is the storage-layer guarantee. The query specifies the namespace; the store will not return cross-namespace data even if the metadata filter were missing.
- **Shared-index model.** No storage-layer guarantee is available; the application-layer filter is the only line of defense, which is why this model has the highest blast radius.

### 5.2 Layer 2: Retrieval-client wrapper

Section 4. Tenant ID required parameter; underlying store routing keyed on it; post-retrieval verification raises on violation.

### 5.3 Layer 3: Tenant-violation testing

Section 6. A test suite that plants canary data in tenant B and queries from tenant A, asserting that the canary is never returned. Runs in CI, runs nightly on staging, runs on every wrapper change.

### 5.4 Layer 4: Production audit logging

Every retrieval call logs: tenant ID, query (or query hash), retrieved doc IDs, retrieval scores, trace ID. A separate alert fires on any tenant-violation event raised by the wrapper's post-retrieval verification.

The audit log supports two operations:
- **Incident investigation.** If a cross-tenant leak is suspected, the audit log is the evidence.
- **Compliance attestation.** Customers' compliance teams can be shown: every query against your data was tagged with your tenant ID; no query returned data from another tenant ID.

### 5.5 The layered model

Each layer addresses a different failure mode:

| Failure mode | Defended by |
|---|---|
| Application code skips the wrapper | Lint rule (section 4.2) + code review |
| Wrapper has a bug that forgets to inject tenant | Post-retrieval verification (section 4.1) |
| Vector-store SDK bug returns cross-namespace data | Per-tenant-index or per-tenant-namespace (section 5.1) |
| Wrapper is modified in a way that drops the filter | Tenant-violation tests (section 6) catch regression |
| Cross-tenant leak occurs despite all defenses | Audit log (section 5.4) supports investigation and reporting |

No single layer is sufficient. Together they form the defense pattern that actually holds under engineering team turnover and system evolution.

---

## 6. Testing patterns

The tenant-violation test is the most important test in a multi-tenant AI system. It is also the test most teams skip because it requires planting canary data and running with production-like configuration.

### 6.1 The canary pattern

In a test environment:

1. **Plant.** For each of two test tenants (A and B), insert a small set of vectors with known content. Tenant B's content is the canary — clearly identifiable, never expected to appear in tenant A queries.
2. **Query.** From tenant A, run a battery of queries that, *if the filter were missing*, would return tenant B's canary content (semantically similar queries to the canary content).
3. **Assert.** Every query result must have `tenant_id == 'A'`. The canary content from tenant B must never appear in any result.

The test runs against the same retrieval-client wrapper code path that production uses. Any code path that bypasses the wrapper or breaks the filter fails this test.

### 6.2 The test taxonomy

Run the canary pattern across every code path that touches retrieval:

- **Normal user query path.** The most-used code path.
- **Admin retrieval paths.** Tools that the platform team uses for debugging or content management. These are the most likely to be added without tenant scoping.
- **Background job paths.** Eval runs, batch processing, scheduled jobs. These may not have a "current tenant" context.
- **Agent tool-call paths.** When an agent calls a retrieval tool, the tool's invocation must propagate tenant context. A canary test against the agent path catches the case where the tool implementation drops the tenant.
- **API-gateway retrieval paths.** If retrieval is exposed via API to external consumers, the gateway is the surface where tenant context must be set; canary test the gateway-to-wrapper path.

### 6.3 CI integration

The canary suite is part of CI for any change touching the retrieval-client wrapper, the vector store interaction, or the multi-tenancy layer. It is also part of the release-candidate gate (run the full canary suite before promotion to production).

The suite is small (typically <50 cases) and fast (<2 minutes) because the canary content is small. Cost is negligible.

### 6.4 Production canary verification

A weekly automated check runs a single canary query against production and asserts isolation. The cost is one query per week per tenant pair (or per representative tenant pair); the value is detecting drift in production configuration that the test environment does not reflect.

### 6.5 Synthetic tenant for ongoing verification

A reserved synthetic-tenant ID (`tenant_id='__canary__`) is populated with content that should never match any real query. Production queries from real tenants are asserted not to return canary content. This is a passive monitor; if the canary content appears in real-tenant retrieval, alerts fire immediately.

---

## 7. Worked Meridian Health example

### 7.1 The decision

Meridian's Care Coordinator answers the four questions from section 2 as:

1. **Tenant count.** 240 hospital tenants, growing to ~400 over the next year as customer onboarding continues.
2. **Per-tenant data volume.** Median ~150 protocol documents per tenant (small), with global shared content (clinical guidelines from professional societies) at ~50,000 chunks accessible to all tenants. Volume variance is moderate — the largest customer has ~3x the median tenant's volume.
3. **Regulatory posture.** HIPAA business-associate; SOC 2 Type II; HITRUST; FedRAMP track in parallel. Per-tenant isolation is a contractual commitment to every hospital customer; cross-tenant data appearance would be a HIPAA breach.
4. **Operational team appetite.** The ai-platform-eng team is 6 people; per-tenant-index at 240+ tenants would require dedicated platform staff that the team does not have.

→ **Per-tenant namespace** for the 240 standard-tier hospital tenants. **Per-tenant index** for the one premium customer that contracted dedicated infrastructure.

### 7.2 The storage layout

```
Aurora Postgres (us-east-1)
├── meridian_vectors (database)
│   └── chunks (table, pgvector index)
│       ├── partition by tenant_id (Postgres declarative partitioning)
│       │   ├── chunks_p_mercy_cleveland
│       │   ├── chunks_p_st_marys_boston
│       │   ├── chunks_p_global_guidelines      (the global tenant)
│       │   ├── ... (240 partitions)
│       │   └── chunks_p_default                (catch-all, should always be empty)
│       └── HNSW index per partition
│
Aurora Postgres (us-east-1) -- separate cluster for premium customer
├── meridian_vectors_premium_customer (database)
│   └── chunks (table, pgvector index)
│       └── (single-tenant, all data is the premium customer's)
```

Per-partition HNSW indexes mean queries against `tenant_id='mercy-cleveland'` use only the mercy-cleveland partition's index. Postgres partition pruning is the storage-layer guarantee.

### 7.3 The retrieval-client wrapper

```python
class MeridianRetrievalClient:
    def __init__(self, store_config: VectorStoreConfig):
        self._store = _build_store(store_config)

    def retrieve(
        self,
        query: str,
        tenant_id: str,
        retrieval_surfaces: list[str],
        top_k: int = 10,
        trace_id: str | None = None,
    ) -> RetrievalBundle:
        if not tenant_id:
            raise ValueError("tenant_id is required for all retrieval")

        # Resolve tenant scope. For most tenants, that's their hospital
        # namespace + the global guidelines. For the premium customer,
        # it's their dedicated index + the global guidelines.
        scope = self._resolve_scope(tenant_id)

        # Run BM25 + vector retrieval in parallel
        bm25_results, vector_results = self._parallel_retrieve(
            query, scope, top_k * 5
        )

        # Merge (RRF) and rerank
        merged = self._reciprocal_rank_fusion(bm25_results, vector_results)
        reranked = self._rerank(query, merged[:50])[:top_k]

        # POST-RETRIEVAL VERIFICATION
        for result in reranked:
            result_tenant = result.metadata.get("tenant_id")
            if result_tenant not in {tenant_id, "global-guidelines"}:
                self._alert_tenant_violation(
                    requested=tenant_id,
                    returned=result_tenant,
                    trace_id=trace_id,
                )
                raise TenantIsolationViolation(
                    f"Requested tenant={tenant_id}, "
                    f"result has tenant={result_tenant}"
                )

        self._audit_log(tenant_id, query, scope, reranked, trace_id)
        return RetrievalBundle(results=reranked, scope=scope)
```

The wrapper enforces:
- `tenant_id` required.
- Tenant scope resolved deterministically (the only "exception" is the global-guidelines tenant, which is explicitly allowed in the verification).
- Post-retrieval check raises on any cross-tenant result.
- Every call audit-logged.

### 7.4 Defense layers in production

- **Storage layer.** Postgres partition pruning means a query scoped to `tenant_id='mercy-cleveland' OR tenant_id='global-guidelines'` only scans those two partitions. Even with a wrapper bug, no other tenant's data is accessed.
- **Wrapper layer.** Mandatory `tenant_id` parameter; post-retrieval verification.
- **Lint layer.** A CI check rejects any application-code import of `psycopg2` or the pgvector client directly outside the `meridian_retrieval` package.
- **Test layer.** A 38-case canary test suite runs on every PR touching `meridian_retrieval/*` and on every release candidate.
- **Audit layer.** Every retrieval call logs `tenant_id`, query hash, returned doc IDs, scores, trace ID; the audit log is HIPAA-attestation-grade.

### 7.5 The premium-customer variant

For the one premium customer on per-tenant-index:

- Separate Aurora cluster, separate VPC, separate KMS key.
- The retrieval client's `_resolve_scope` returns a connection to that cluster when called with the premium customer's tenant ID.
- Every other element of the architecture is otherwise identical — the premium customer benefits from the *same* wrapper, the *same* post-retrieval verification, the *same* audit log. The dedicated infrastructure is an additional layer, not a replacement for the others.

This is important. Customers paying for dedicated infrastructure should not lose the wrapper-level protections; they should gain a storage-layer additional protection on top. The principle: **isolation layers compose; they do not replace.**

### 7.6 The canary suite

The Meridian canary test suite has 38 cases organized as:

- **Basic canary (8 cases).** Plant canary content in tenant B; query from tenant A with semantically similar queries; assert no canary content returned.
- **Admin-path canary (4 cases).** Use the platform admin retrieval tool; assert that tenant scoping is enforced even from admin context.
- **Background-job canary (6 cases).** Run a sample eval job, a sample re-indexing job, a sample data-quality job; assert that none of them retrieve cross-tenant.
- **Agent-tool canary (8 cases).** Invoke the agent retrieval tool from a test agent context; assert tenant propagation.
- **API-gateway canary (4 cases).** Hit the retrieval endpoint with a tenant=A token; assert no tenant=B content returned.
- **Global-tenant canary (4 cases).** Assert that global-guidelines content IS retrievable (positive test for the explicit exception).
- **Default-partition canary (2 cases).** Assert the `chunks_p_default` partition is empty; assert that queries with an invalid tenant ID fail closed.
- **Re-index canary (2 cases).** During a partition re-index, assert that cross-tenant retrieval is still prevented.

The full suite runs in ~90 seconds and gates every PR to the retrieval package.

### 7.7 Production verification

- Every Monday at 02:00 UTC, an automated test queries one canary-pair (tenant A queries for tenant B canary content) against production. Result is logged; cross-tenant return pages the on-call.
- A synthetic `tenant_id='__canary__'` is populated with content that should never match real queries. A weekly job scans the production audit log for any real-tenant query that returned `__canary__` content.
- HIPAA-attestation-grade quarterly: the security team pulls a sample of audit-log entries and confirms tenant-attribution for each.

---

## 8. Anti-patterns

### 8.1 "We pass tenant_id everywhere, that's enough"

The team has `tenant_id` in every function signature but no central enforcement. New code paths forget to pass it; old code paths use defaults; the tenant_id-in-every-signature discipline degrades over a year.

**Corrective.** Centralize in the wrapper. Make `tenant_id` impossible to omit because the only function that retrieves *requires* it. Add the lint rule that prevents direct vector-store access.

### 8.2 "The framework's middleware sets tenant from request context"

The web framework's middleware sets `current_tenant` based on the JWT; application code reads from `current_tenant` when needed. Works for HTTP-shaped traffic; fails for batch jobs, scheduled jobs, CLI tools, eval runs — all of which run outside the middleware's scope and default to no tenant or to a stale one.

**Corrective.** Pass `tenant_id` explicitly through every code path, including non-HTTP ones. The middleware can populate it at the HTTP boundary, but downstream code must accept it as a parameter, not read it from a global.

### 8.3 "Admin tools have their own retrieval client without tenant filtering"

To support legitimate admin operations (look at all the data, debug cross-tenant patterns), the team built a separate admin retrieval client without tenant scoping. It is now also used for non-admin operations because it is convenient.

**Corrective.** Admin tools that need cross-tenant access call a separate, audited, explicitly-authorized `AdminRetrievalClient`. Every call is logged with the admin user's identity and the business reason. Application code cannot import the admin client; only platform-eng has access.

### 8.4 "Per-tenant index without operational investment"

The team picked per-tenant-index for the strongest isolation, then discovered they have 240 indexes to operate and no platform-eng capacity to operate them. Patches are delayed; capacity is unbalanced; backups are unreliable; eventually one index fails and the team cannot recover it.

**Corrective.** Per-tenant index needs a per-tenant operational platform: automated provisioning, automated capacity management, automated backup verification, automated patch rollout, per-tenant monitoring. If the team cannot stand up that platform, choose per-tenant-namespace instead.

### 8.5 "Shared-index for regulated workloads because it is cheaper"

The team chose shared-with-filter because vector-store cost was the loudest line in the budget at design time. After a single missed filter incident, the team is migrating to per-tenant namespace under audit pressure, and the cost of migration plus the cost of incident response far exceeds the cost saved.

**Corrective.** Match the isolation model to the regulatory bar from the start. For HIPAA / financial / public-sector workloads, the cost premium of per-tenant namespace over shared-with-filter is small; the cost of a breach is enormous.

### 8.6 "Canary test was written once and is no longer run"

The canary suite exists but has not run in 8 months because of a flaky test infrastructure issue. The team assumes the canary is still protecting them.

**Corrective.** Canary suite as a required PR check for the retrieval package. Suite failure blocks merge. Flaky tests are debugged, not skipped.

### 8.7 "Tenant verification is done before the wrapper, not in it"

Tenant-authorization checks live in an outer layer (the API gateway, the middleware). The wrapper trusts the tenant_id it receives. A bug in the outer layer (or a code path that bypasses it) becomes a tenant leak.

**Corrective.** Verification in the wrapper is independent of any outer-layer check. Both layers should be enforcing; the wrapper's verification is the defense-in-depth backstop.

---

## 9. Findings (sprint-assignable)

### ARCH-TENANT-001 — Severity: Critical
**Finding.** Retrieval-client wrapper does not enforce `tenant_id` as a required parameter; code paths can call retrieval without scoping.
**Recommendation.** Make `tenant_id` required; add lint rule preventing direct vector-store imports outside the wrapper package.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TENANT-002 — Severity: Critical
**Finding.** Post-retrieval verification (results must have matching `tenant_id`) is not implemented; wrapper trusts the storage layer to enforce scope.
**Recommendation.** Add post-retrieval check per section 4.1; raise on mismatch; alert as Sev-1.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TENANT-003 — Severity: Critical
**Finding.** Canary test suite does not exist; no automated test asserts that tenant A queries cannot return tenant B data.
**Recommendation.** Build the canary suite per section 6; gate the retrieval-package CI on it.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TENANT-004 — Severity: High
**Finding.** Admin retrieval is performed via the same client as user retrieval, with a `skip_tenant_filter=True` flag.
**Recommendation.** Split into `RetrievalClient` (no skip) and `AdminRetrievalClient` (separate, audited, explicit authorization); remove the flag.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-TENANT-005 — Severity: High
**Finding.** Background jobs (eval, batch, re-index) do not propagate tenant context; some default to a "default tenant" that may bypass scoping.
**Recommendation.** Every job entrypoint accepts tenant context as a required parameter; remove default-tenant fallbacks.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TENANT-006 — Severity: High
**Finding.** Tenant context is inferred from a thread-local set by HTTP middleware; non-HTTP code paths do not set it.
**Recommendation.** Pass `tenant_id` explicitly through call paths; HTTP middleware can populate the parameter but downstream code does not read from globals.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TENANT-007 — Severity: High
**Finding.** Retrieval audit log does not include `tenant_id`; cross-tenant incident investigation requires reconstruction from request logs.
**Recommendation.** Add `tenant_id` (and query hash, returned doc IDs, scores, trace ID) to the retrieval audit log; verify HIPAA-attestation completeness.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-TENANT-008 — Severity: High
**Finding.** Isolation model in production is shared-with-filter for a regulated workload; one missed filter is one breach.
**Recommendation.** Migrate to per-tenant namespace; canary test the migration; backfill audit log to demonstrate scope from migration date forward.
**Owner.** ai-platform-eng, sprint N+3 through N+5 (multi-sprint migration).

### ARCH-TENANT-009 — Severity: Medium
**Finding.** Per-tenant-index model in use without automated provisioning / patching / backup verification; operational load is borne ad-hoc.
**Recommendation.** Either build the per-tenant operational platform or migrate to per-tenant namespace.
**Owner.** ai-platform-eng team lead, sprint N+3 (decision); N+4+ (execution).

### ARCH-TENANT-010 — Severity: Medium
**Finding.** Canary test suite exists but is flaky and frequently skipped.
**Recommendation.** Debug and stabilize the suite; make passage a required, non-skippable PR check; allocate engineering capacity to maintain it.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TENANT-011 — Severity: Medium
**Finding.** Global / shared content (clinical guidelines, common policies) is stored with `tenant_id=NULL` rather than as an explicit global tenant.
**Recommendation.** Use an explicit `tenant_id='global'` (or similar); the wrapper's post-retrieval check whitelists exactly that ID; no NULL.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TENANT-012 — Severity: Medium
**Finding.** Per-namespace product features (per-namespace metrics, per-namespace policy) are not in use; cross-tenant operational signal is lost.
**Recommendation.** Enable per-namespace observability; surface per-tenant retrieval-latency and retrieval-error-rate dashboards.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TENANT-013 — Severity: Medium
**Finding.** Re-index operations do not preserve isolation guarantees during the operation; transient cross-tenant exposure has been observed.
**Recommendation.** Re-index per-partition with per-partition canary verification; never re-index across tenants in one operation.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TENANT-014 — Severity: Medium
**Finding.** Production canary verification (weekly automated check) is not scheduled.
**Recommendation.** Schedule the production canary; route failures to on-call paging.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-TENANT-015 — Severity: Medium
**Finding.** Wrapper exists but is bypassable — the underlying SDK can be imported directly from application code.
**Recommendation.** Add a CI lint rule that rejects direct SDK imports outside the wrapper package; add the rule to required PR checks.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TENANT-016 — Severity: Medium
**Finding.** Cross-tenant queries are performed for platform analytics without an audited authorization path.
**Recommendation.** Build the `AdminRetrievalClient` with explicit authorization and audit; route platform analytics through it.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### ARCH-TENANT-017 — Severity: Low
**Finding.** Per-tenant retrieval latency varies meaningfully; noisy-neighbor mitigation is not in place.
**Recommendation.** Per-tenant query budgets and rate limiting at the wrapper layer.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TENANT-018 — Severity: Low
**Finding.** Tenant-isolation architecture is undocumented; new engineers cannot reason about the guarantees.
**Recommendation.** Document the isolation model, the wrapper contract, the test suite, and the audit log; commit alongside the wrapper code.
**Owner.** ai-platform-eng, sprint N+4.

---

## 10. Adoption sequencing checklist

For a team about to land a multi-tenant retrieval system:

- [ ] **Sprint 0 — decide.** Answer the four questions from section 2. Pick the isolation model. Document the decision with rationale (volume, regulatory bar, team capacity).
- [ ] **Sprint 1 — wrapper.** Build the retrieval-client wrapper per section 4. Tenant required. Post-retrieval verification. Audit logging. Lint rule against direct SDK imports.
- [ ] **Sprint 1 — canary suite.** Build the canary test suite per section 6. Gate retrieval-package CI on it.
- [ ] **Sprint 2 — storage layer.** Implement the chosen isolation model in the storage layer (namespace, dedicated index, or partition with filter). For per-tenant namespace, enable per-namespace metrics.
- [ ] **Sprint 2 — observability.** Per-tenant audit log; production canary on schedule; alerting on tenant-violation events.
- [ ] **Pre-launch — security review.** Walk through the layered defenses (section 5). Confirm each layer is in place. Document for the customer compliance package.
- [ ] **Post-launch — monthly verification.** Sample the audit log; assert tenant-attribution is complete. Quarterly: customer-facing compliance attestation.

A team that follows this sequence has the isolation pattern that holds under team turnover, code refactors, and audit scrutiny. A team that defers any of these steps is taking on tail-risk that surfaces at the worst possible moment — typically during a customer compliance review.

---

## 11. References

- Pinecone, Weaviate, Qdrant, pgvector documentation — vendor-specific namespace and multi-tenancy primitives.
- AWS Aurora Postgres declarative partitioning documentation — the storage-layer guarantee used in the Meridian example.
- HIPAA Security Rule §164.312 (technical safeguards) — the regulatory frame for the Meridian audit-log requirements.
- SOC 2 Common Criteria 6.1 (logical access controls) — the framework used for customer compliance attestation.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) for the production architecture context.
- This repo: [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) ARCH-RAG-008 finding, which references this document.
- This repo: [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/) (coming) for the broader guardrails view.
- This repo: [multi-tenancy-and-isolation/isolation-models.md](./) (coming) for the full multi-tenancy framework that this document is one chapter of.
- Sibling repo: [ai-security-reference-architecture/llm-application-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) for the threat model — including prompt-injection exfiltration paths that this pattern alone does not address.
- Sibling repo: [cloud-security-reference-architecture/data-security/](https://github.com/jeremiahredden/cloud-security-reference-architecture) (coming) for the cloud-layer storage isolation patterns that complement this AI-layer pattern.
