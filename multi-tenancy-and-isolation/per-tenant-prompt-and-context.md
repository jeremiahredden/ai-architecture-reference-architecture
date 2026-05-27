# Per-Tenant Prompt and Context

> **Audience.** Architects of multi-tenant AI features whose prompts are assembled at request time from shared and tenant-specific pieces. Tech leads writing the prompt-assembly layer for a system that will host customers B, C, and D as well as customer A. Anyone whose retrieval pipeline returns chunks that may include another tenant's data if the filter is missing or wrong. **Scope.** The *architectural* decisions: tenant identity as a first-class context field; the layered prompt-assembly architecture (shared base + tenant overlay + per-request context); tenant-aware retrieval at the prompt boundary; per-tenant tool authorization at the prompt-assembly layer; output validation against tenant scope; per-tenant conversation state and memory. Not the vector-store-side namespacing (see [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md), which is the data layer). Not the broader leakage controls (see [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md), which catalogs the leakage paths comprehensively). Not the fairness mechanics (see [noisy-neighbor-mitigation.md](./noisy-neighbor-mitigation.md)). Not the prompt-engineering disciplines themselves (see sibling [prompt-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/prompt-engineering)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

In multi-tenant AI, the most damaging leakage path is not the vector store and not the database — both have well-understood isolation patterns. The most damaging leakage path is the *prompt assembly*: the moment when a system stitches together a system prompt, a tenant-specific overlay, retrieved context, conversation history, and the current user input into a single string that gets sent to the model.

The model doesn't care whose data is in the prompt. The model will dutifully reason over whatever you put in front of it and produce an output. If tenant A's payment history is in tenant B's prompt because a retrieval filter was off-by-one or a context cache was tenant-blind, the model will format tenant A's payment history into tenant B's response. From the model's perspective, it has done its job perfectly.

The architectural failure is upstream of the model. It is in how the prompt was assembled.

The reasons prompt-assembly is the high-risk surface:

- **The integration points are many.** A typical assembled prompt draws from a system-prompt store, a tenant configuration store, a retrieval pipeline, a conversation history store, a tool-result store, a memory store, and the current user message. Each is its own source; each requires its own tenant filter; missing or misconfiguring any one produces a leak.
- **The prompt is plaintext.** Unlike a database query that goes through a schema layer with validation, a prompt is just a string. There is no schema enforcement on its content. If your prompt assembler accidentally concatenates the wrong chunk, no system layer below it will catch the mistake.
- **The state lives in opaque places.** Per-tenant prompt overlays live in config; per-tenant context lives in cache; per-tenant memory lives in a separate store; per-tenant tool authorization lives in policy. Walking the data flow requires touching multiple systems, each owned by a different team.
- **The bug is invisible until the customer sees it.** A retrieval filter that returns 11 documents instead of 10 (one belonging to the wrong tenant) doesn't fail any test that doesn't specifically check tenancy. The eleventh document just shows up in the response, sometimes verbatim, sometimes summarized.
- **The detection lag is high.** A tenant who sees another tenant's data in their response may not notice (if the data is generic), may not report (if it's mildly embarrassing), or may report through an unexpected channel weeks later.

This document covers the architectural patterns that make per-tenant prompt assembly robust by construction: tenant identity propagation, layered prompt structure, tenant-aware retrieval at the assembly boundary, tool authorization at the prompt layer, output validation, and per-tenant conversation state. These are companions to (not replacements for) the data-layer isolation patterns in the rest of this folder.

This document is opinionated about four things:

1. **Tenant identity is a first-class context field, not a global variable or a thread-local.** Every prompt-assembly function must take tenant_id as an explicit parameter. Implicit tenant context — "the current request's tenant" — produces bugs that survive testing and surface in production.
2. **Retrieval filters are mandatory, not advisory.** Deny-by-default: no retrieval can happen without an explicit tenant filter; the filter is enforced at the retrieval layer, not at the prompt-assembly layer. "Trust the retrieval layer to do the right thing" is how leaks happen.
3. **Per-tenant overlays are layered on a shared base, not full prompt templates per tenant.** Full prompt-per-tenant produces unmaintainable drift; shared base + tenant overlay keeps the surface manageable while still allowing per-tenant customization.
4. **Output must be validated against tenant scope before being returned.** Even with perfect prompt assembly and perfect retrieval, the model can hallucinate; output that references entities outside the tenant's scope must be flagged or rejected. This is the last line of defense.

Structure: (2) the tenant-identity-as-context pattern; (3) the layered prompt-assembly architecture; (4) tenant-aware retrieval at the prompt boundary; (5) per-tenant tool authorization at the assembly layer; (6) output validation against tenant scope; (7) per-tenant conversation state and memory; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The tenant-identity-as-context pattern

The foundational discipline. Every component that touches a prompt or its inputs must know which tenant it is operating for, and the answer must be explicit, not inferred.

### 2.1 What "first-class" means

A first-class tenant identity has four properties:

- **Explicit at every boundary.** Function signatures take `tenant_id` as a parameter; HTTP requests carry it in a header or path; queue messages carry it in the payload; cache keys include it.
- **Validated at entry.** When a request enters the system, the tenant is established (via API key, JWT claim, or session) once, in one place, and validated against the authentication / authorization system.
- **Propagated, never re-derived.** Downstream code does not re-derive the tenant from "the current user" or "the current document"; it receives the established tenant from upstream. Re-deriving is a vector for subtle mismatches.
- **Logged at every step.** Every log line, span, and metric carries the tenant ID. Debugging requires being able to filter by tenant; auditing requires being able to enumerate by tenant.

### 2.2 The propagation pattern

```python
@dataclass
class RequestContext:
    tenant_id: str
    user_id: str
    request_id: str
    auth_scopes: list[str]
    # ... other request-scoped fields

# Established once at the request boundary:
context = RequestContext(
    tenant_id=verified_tenant_from_jwt(request.headers["Authorization"]),
    user_id=verified_user_from_jwt(request.headers["Authorization"]),
    request_id=generate_request_id(),
    auth_scopes=verified_scopes(),
)

# Passed explicitly down the call chain:
response = handle_query(context, user_query)

# Every function takes context as a parameter:
def handle_query(context: RequestContext, query: str) -> Response:
    retrieved = retrieve_context(context, query)
    prompt = assemble_prompt(context, query, retrieved)
    output = call_model(context, prompt)
    return validate_and_format(context, output)
```

Frameworks that hide tenant context in thread-locals, request-scoped containers, or implicit globals make this harder. They aren't fatal, but they make per-tenant violations harder to spot in code review.

### 2.3 Where re-derivation goes wrong

The pattern that produces leaks:

```python
# WRONG: re-derives tenant from the document
def retrieve_more_like_this(document_id: str) -> list[Document]:
    document = fetch_document(document_id)  # whose tenant? whoever owns document_id
    tenant = document.tenant_id              # implicit; depends on document being correct
    similar = vector_search(document.embedding, filter={"tenant": tenant})
    return similar
```

The bug: if `document_id` somehow points to a wrong-tenant document (passed in error, attacker-controlled, copy-pasted from another context), the retrieval is scoped to the wrong tenant. The function had no opportunity to verify.

The fix:

```python
# RIGHT: tenant is an explicit parameter
def retrieve_more_like_this(context: RequestContext, document_id: str) -> list[Document]:
    document = fetch_document(context, document_id)  # verifies tenant ownership
    similar = vector_search(document.embedding, filter={"tenant": context.tenant_id})
    return similar
```

Now `fetch_document` can verify the requester is allowed to access `document_id`; the retrieval filter is scoped to the requester's tenant; cross-tenant access requires bypassing both.

### 2.4 The tenant-tag-on-every-object discipline

Every domain object in the system has a `tenant_id` field. Patients have it. Documents have it. Conversations have it. Memory entries have it. Cache entries have it.

The field is non-nullable; objects without a tenant are not allowed.

The field is enforced at write time; the persistence layer rejects writes missing or mismatched.

The field is checked at read time; queries that don't include a tenant filter raise an error in development and log a warning in production (or, in mature systems, fail closed).

### 2.5 Auditing the propagation

The discipline can be audited:

- **Static analysis.** Lint rules that flag any function call passing data between tenants. Difficult to implement universally; possible for high-risk APIs (retrieval, model call, cache write).
- **Logging analysis.** Every log line includes tenant; aggregating by tenant_id reveals queries that touched multiple tenants in a single request (suspicious).
- **Runtime contracts.** Decorators or middleware that verify `tenant_id` is in scope at every key boundary; production-grade systems fail-closed if it's missing.

The auditing posture matures over time. Early systems rely on code review and convention; mature systems automate.

---

## 3. The layered prompt-assembly architecture

Per-tenant prompts are an architecture, not a template. The pattern that scales is shared base + tenant overlay + per-request context, assembled at request time.

### 3.1 The three layers

**Layer 1: Shared base prompt.** The platform's system prompt. Defines model role, response style, safety constraints, output format. Owned by the platform team; same for all tenants.

**Layer 2: Per-tenant overlay.** Tenant-specific customizations. Tenant's preferred terminology, tenant-specific tool descriptions, tenant-specific compliance constraints. Owned by the tenant's account team; small (typically < 1k tokens); curated.

**Layer 3: Per-request context.** Retrieved documents, conversation history, current user message, tool results. Assembled at request time; specific to this request.

The final prompt is the concatenation (with templating discipline):

```
{shared_base_prompt}

{tenant_overlay}

## Retrieved context
{retrieved_chunks_filtered_by_tenant}

## Conversation history
{conversation_history_filtered_by_tenant_and_session}

## Current request
{user_message}
```

### 3.2 Why this structure and not full prompt-per-tenant

**Full prompt-per-tenant** would mean each tenant has their own system prompt template that diverges arbitrarily from others. Initially attractive (maximum customization); over time produces drift:

- Tenant A's prompt has a useful safety constraint; tenant B's doesn't; tenant B incident.
- Prompt improvement requires updating N tenant prompts; updates lag; quality drifts.
- New platform-level feature requires modifying N tenant prompts.
- Eval coverage is N times the work.

**Shared base + tenant overlay** keeps the surface area manageable. Platform changes apply to all tenants by updating the base. Tenant customization happens in the overlay, which is bounded and reviewable.

### 3.3 Overlay scope

The overlay must be small and bounded. Things the overlay can do:

- Add tenant-specific terminology ("call it 'membership' not 'subscription'").
- Add tenant-specific tool descriptions (the same underlying tool may have tenant-specific framing).
- Add tenant-specific compliance notes ("this tenant requires citations on all clinical claims").
- Add tenant-specific persona ("respond as a Meridian assistant; respond as a Northstar Health assistant").
- Add tenant-specific output format preferences (markdown vs plaintext; bullet vs paragraph).

Things the overlay should not do:

- Override the platform's safety constraints.
- Redefine the model's role or fundamental capabilities.
- Inject custom tool authorization (that lives in policy; see §5).
- Inject retrieved context (that lives in layer 3).
- Become a vehicle for tenants to expand the agent's capability beyond what the platform allows.

### 3.4 Overlay storage and lifecycle

```yaml
tenant: meridian-southwest
overlay_version: 4
overlay_content: |
  This system serves Meridian Health Southwest Region. When referring to
  clinicians, use "provider" rather than "doctor." For medication-related
  responses, include relevant Meridian formulary notes when available.
  All responses about clinical decisions must include a citation.
deployed_at: 2026-04-02T14:30:00Z
deployed_by: meridian-platform-team
approval_status: approved
approver: platform-prompt-review-board
```

Overlays are versioned and stored in a config store. Changes go through the same review process as code changes; emergency overlay changes are possible but logged. Overlays are tested in pre-production against the eval suite before deployment.

Cross-link to [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) for the engineering-side versioning discipline.

### 3.5 The assembled-prompt audit

Every assembled prompt should be loggable (with PII redaction). The audit log shows:

- Which shared base version was used.
- Which tenant overlay version was used.
- Which retrieved chunks were included (by ID; not full content if sensitive).
- Which conversation history was included (by session ID).
- The current user message.

A leak investigation starts here: did the assembled prompt for tenant B's request contain anything tenant A's?

### 3.6 The prompt-assembly function as bottleneck

All assembled prompts flow through a single function (or a small number of functions per use case). This is intentional: a chokepoint is auditable; a chokepoint is testable; a chokepoint is the place to enforce invariants.

Sprawl is the enemy. If twenty different code paths assemble prompts in twenty different ways, isolation cannot be enforced consistently. Consolidating to one assembler per use case is a one-time investment that pays off in every subsequent isolation review.

---

## 4. Tenant-aware retrieval at the prompt boundary

Retrieval is where most multi-tenant prompt leaks happen. The pattern: a retrieval call returns chunks; those chunks are formatted into the prompt; the chunks include a chunk that belongs to a different tenant.

The defense is layered: the retrieval call must be scoped (handled by the data layer; cross-link to [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md)); the chunks returned must be re-verified at the prompt-assembly layer; the assembled prompt is audited.

### 4.1 The retrieval call's tenant scope

The retrieval call always passes a tenant filter:

```python
def retrieve_for_query(context: RequestContext, query: str, k: int = 10) -> list[Chunk]:
    query_embedding = embed(query)

    results = vector_search(
        query_embedding,
        k=k,
        filter={"tenant_id": context.tenant_id}  # mandatory, deny-by-default
    )

    # Verify, don't trust:
    for chunk in results:
        if chunk.tenant_id != context.tenant_id:
            log.error(
                "tenant_filter_violation",
                expected=context.tenant_id,
                actual=chunk.tenant_id,
                chunk_id=chunk.id
            )
            raise TenantIsolationError(f"chunk {chunk.id} belongs to wrong tenant")

    return results
```

Two enforcement layers:

- The filter parameter to vector_search. The vector store enforces it.
- The verification loop. Even if the vector store had a bug (or someone removed the filter), the chunks are re-verified before being passed to the prompt assembler.

### 4.2 The chunk's tenant tag

Every chunk in the vector store has a `tenant_id` metadata field. Cross-link to [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) for the namespacing patterns; the relevant fact for prompt assembly is: every returned chunk has the field, and the field is checkable.

### 4.3 The "deny by default" enforcement

In production, retrieval without a tenant filter should be impossible. Mechanisms:

- **Static analysis.** Lint rules that flag `vector_search` calls without a filter parameter (where filter is non-optional).
- **Runtime check.** The vector_search function rejects calls without a filter; logs the violation; raises an error.
- **Wrapper enforcement.** A wrapper around the vector store SDK that requires the filter; the raw SDK is not used directly.

The "convention says you should always include a filter" approach is insufficient; conventions drift. Architectural enforcement is required.

### 4.4 The cross-tenant retrieval exception

Some workloads legitimately need cross-tenant retrieval — analytics dashboards, platform-wide search for internal tools, anti-abuse scanning. These are rare and must be:

- Explicitly authorized (a specific scope or role that grants cross-tenant access).
- Logged differently (cross-tenant access is a security event).
- Audited (periodic review of who is doing cross-tenant retrieval).
- Walled off from tenant-facing surfaces (cross-tenant code paths cannot be invoked by tenant requests).

The default for any tenant-facing code is single-tenant retrieval; cross-tenant is the exception with its own controls.

### 4.5 The retrieval result as prompt input

After verification, retrieved chunks become part of the prompt. The formatting includes:

- Source attribution (where the chunk came from).
- Tenant attribution (implicit because all chunks are from the same tenant after the filter, but explicitly noted in audit logs).
- Citation format (so the model can refer to specific chunks in its response).

The chunks are passed to the prompt assembler as a typed list, not as concatenated strings. Type safety helps catch the "accidentally passed the wrong list" bug.

---

## 5. Per-tenant tool authorization at the assembly layer

Agents have tools. Each tenant's agent may have a different set of allowed tools, different tool argument constraints, different tool-result handling. The authorization happens at the prompt-assembly layer (which tools to expose in the prompt's tool list) and at the tool-call dispatch layer (whether to actually execute when the model invokes).

### 5.1 The tool catalog per tenant

Each tenant has an effective tool catalog:

```yaml
tenant: meridian-southwest
allowed_tools:
  - name: fetch_patient_summary
    constraints:
      max_patients_per_call: 1
      auth_required: true
  - name: search_clinical_notes
    constraints:
      max_results: 20
      time_window_days: 365
  - name: draft_referral
    constraints:
      requires_approval: true
disallowed_tools:
  - send_email_to_patient  # disabled for this tenant
  - update_billing  # disabled for this tenant
```

The catalog is per-tenant and configurable. New tools can be added platform-wide and enabled per-tenant; tools can be disabled per-tenant for compliance reasons.

### 5.2 The prompt-layer enforcement

When the prompt is assembled, the tool list passed to the model is filtered by tenant catalog:

```python
def assemble_prompt(context: RequestContext, ...) -> Prompt:
    tenant_tools = get_tool_catalog(context.tenant_id)
    available_tools = [t for t in tenant_tools if t.is_enabled]

    return Prompt(
        system=base + tenant_overlay,
        tools=[t.to_model_format() for t in available_tools],
        ...
    )
```

The model literally cannot call a tool that isn't in the tool list it was given. This is the first enforcement layer.

### 5.3 The dispatch-layer enforcement

The model occasionally hallucinates tool calls (tools it wasn't told about). The dispatch layer enforces:

```python
def dispatch_tool_call(context: RequestContext, tool_call: ToolCall) -> ToolResult:
    tenant_tools = get_tool_catalog(context.tenant_id)
    tool = find_tool(tenant_tools, tool_call.name)

    if tool is None:
        log.warn("hallucinated_tool_call", name=tool_call.name, tenant=context.tenant_id)
        return ToolResult(error="tool_not_available")

    if not tool.is_enabled_for(context):
        log.warn("unauthorized_tool_call", name=tool_call.name, tenant=context.tenant_id)
        return ToolResult(error="not_authorized")

    validate_tool_args(tool, tool_call.args)
    return tool.execute(context, tool_call.args)
```

Defense in depth: the prompt layer didn't list the tool; the dispatch layer rejects if the model invents one. Cross-link to [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) for the deeper authorization patterns.

### 5.4 The tool argument scope

Even allowed tools must have tenant-scoped arguments. A tool like `fetch_patient_summary` takes a patient_id; the patient must belong to the tenant.

```python
def fetch_patient_summary(context: RequestContext, patient_id: str) -> Summary:
    patient = fetch_patient(patient_id)
    if patient.tenant_id != context.tenant_id:
        log.error("tenant_violation", patient=patient_id, tenant=context.tenant_id)
        raise NotAuthorized("patient does not belong to your tenant")
    return summarize(patient)
```

The argument-validation step is per-tool. The tool's implementation is responsible; the dispatch layer can't enforce it generically (it doesn't know what each tool's arguments mean).

### 5.5 The tool result as prompt input

Tool results, like retrieved chunks, become part of the prompt. They must also be tenant-scoped:

- Tool results are tagged with the tenant they pertain to.
- Tool results that reference entities outside the tenant's scope are flagged (rare; usually a bug).
- The prompt assembler verifies tool results' tenant tag matches the context.

---

## 6. Output validation against tenant scope

Even with perfect prompt assembly and perfect retrieval, the model can produce output that references things outside the tenant's scope. This is the last line of defense.

### 6.1 Why model output can violate tenant scope

- **Hallucination.** The model invents a patient name or document title that doesn't exist in the tenant's data.
- **Confused context.** The model recalls something from training that happens to look like another tenant's data.
- **Prompt-injection success.** A document or tool result contained an instruction to "include data about tenant Z"; the model follows.
- **Cross-conversation contamination.** The model's response includes content from a previous conversation that wasn't fully cleared from context.

### 6.2 The output validation pattern

Output validation parses the response, identifies referenced entities, and verifies each entity is within the tenant's scope.

```python
def validate_output_scope(context: RequestContext, output: str) -> ValidationResult:
    referenced_entities = extract_entity_references(output)
    # e.g., patient IDs, document IDs, account IDs

    for entity in referenced_entities:
        entity_record = fetch_entity(entity)
        if entity_record is None:
            return ValidationResult(violation="hallucinated_entity", entity=entity)
        if entity_record.tenant_id != context.tenant_id:
            return ValidationResult(violation="cross_tenant_reference", entity=entity)

    return ValidationResult(ok=True)
```

The entity-extraction step is domain-specific. For Meridian, entities are patient IDs, MRN numbers, document IDs, encounter IDs. For a SaaS analytics tool, entities are customer IDs, report IDs, dashboard IDs.

### 6.3 The action on violation

When validation finds a violation:

- **Block the response.** Don't return the response to the user; return a structured error indicating the system encountered an inconsistency.
- **Log the violation.** Tenant ID, request ID, entity, output excerpt (PII-redacted). Investigation follows.
- **Alert.** Tenant-scope violations are security events; the operator is paged.
- **Retry with stricter prompt.** Sometimes the model produces a clean response on retry. A bounded retry (1-2 attempts) is reasonable.

### 6.4 The cost of validation

Output validation adds latency. For each response, the entity-extraction + verification adds 50-200ms (depending on entity count and the verification path).

For high-stakes workloads (clinical, financial), the latency is justified. For low-stakes workloads (casual chat), sampling-based validation (validate 10% of responses) is a reasonable trade-off.

### 6.5 The validation rules per tenant

Some tenants have stricter validation rules:

- Regulated tenants: full validation on every response.
- Premium tenants: full validation as part of the SLA.
- Free tier: sampling-based validation.

Per-tenant validation policy lives in the same config as the tool catalog and overlay.

### 6.6 The "the model said something true but out of scope" edge case

A subtle case: the model produces output that's technically true but references data the tenant shouldn't see. For example, a general medical fact that happens to apply to a patient the tenant doesn't have visibility into.

The validation pattern handles this by checking *referenced entities*, not raw content. If the response says "patients with condition X often need Y" without naming a specific patient, no entity reference exists; validation passes. If the response says "patient P12345 with condition X needs Y" and P12345 doesn't belong to the tenant, the entity check fails.

---

## 7. Per-tenant conversation state and memory

Multi-tenant AI features with conversation history or persistent memory have additional isolation requirements at the conversational-state layer.

### 7.1 Conversation history isolation

A conversation history is a list of (user message, assistant response) pairs. Each conversation belongs to a tenant; the history is fetched as part of prompt assembly.

```python
def fetch_conversation_history(context: RequestContext, conversation_id: str) -> list[Turn]:
    conversation = fetch_conversation(conversation_id)
    if conversation.tenant_id != context.tenant_id:
        raise NotAuthorized("conversation does not belong to your tenant")
    return conversation.turns
```

The conversation is keyed on conversation_id; the tenant check is mandatory. A conversation belongs to exactly one tenant.

### 7.2 The cross-conversation cache trap

A common pattern: caching responses for performance. A naive cache keyed only on (query, retrieved_context) returns the same response across tenants.

The cache key must include tenant_id:

```python
cache_key = hash(f"{tenant_id}:{query}:{retrieved_context_hash}")
```

Even better: separate cache namespaces per tenant.

```python
cache_key = f"tenant:{tenant_id}:query:{hash(query)}"
```

A tenant's cache hits affect only their own subsequent requests; never another tenant's.

### 7.3 Per-tenant memory

Agents with persistent memory (recall from prior conversations, learned facts about the user) store the memory per tenant + per user:

```
memory_key = f"tenant:{tenant_id}:user:{user_id}:fact:{fact_id}"
```

Memory queries are tenant-scoped:

```python
def recall_memory(context: RequestContext, query: str) -> list[Memory]:
    return memory_store.search(
        query_embedding=embed(query),
        filter={"tenant_id": context.tenant_id, "user_id": context.user_id}
    )
```

A memory created by user A in tenant T is not recallable by user B in tenant T; it is not recallable by anyone in tenant S.

Cross-link to [ai-engineering-reference-architecture / agent-engineering / memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) for the engineering-side memory patterns.

### 7.4 The conversation forking problem

A user copies a conversation from one tenant to another (e.g., they're a consultant who works with multiple Meridian regions). The conversation has tenant A's data; if it's restored in tenant B's context, leakage.

Mitigations:

- Conversations are not portable across tenants by default.
- A user with cross-tenant access (rare) sees separate conversations per tenant.
- Conversation export / re-import requires explicit user action and is logged.

### 7.5 The "shared conversation between tenants" anti-pattern

Some collaboration features want multi-tenant conversations (e.g., an inter-tenant referral discussion). These are advanced patterns and should be designed carefully:

- The conversation has multiple tenant tags; access is governed by membership.
- Each tenant sees only the portions relevant to them.
- The prompt assembled for the model should not mix multiple tenants' contexts; the model sees one tenant's view at a time.
- The audit trail captures which tenant saw what.

The simpler default: no shared conversations. Users in multiple tenants have separate conversation histories per tenant. The collaboration case is an explicit feature with its own design.

---

## 8. Worked Meridian example

Meridian's Patient API AI-Assist serves 12 tenants on shared infrastructure. Per-tenant prompt and context isolation is the architecture's load-bearing surface for the multi-tenant capability.

### 8.1 The prompt-assembly architecture

```
┌─────────────────────────────────────────────────┐
│  RequestContext (tenant_id, user_id, ...)       │
│  established at request entry, propagated down   │
└───────────────────────┬─────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
┌──────────────┐ ┌─────────────┐ ┌──────────────┐
│ Shared base  │ │ Tenant      │ │ Per-request   │
│ prompt       │ │ overlay     │ │ context       │
│ (v17)        │ │ (tenant-    │ │ (retrieved +  │
│              │ │  southwest, │ │  conversation │
│              │ │  v4)        │ │  + tools)     │
└──────┬───────┘ └─────┬───────┘ └──────┬───────┘
       │               │                 │
       └───────────────┼─────────────────┘
                       ↓
           ┌─────────────────────────┐
           │ Prompt assembler        │
           │  - injects tenant_id    │
           │  - verifies retrieved   │
           │    chunks belong to     │
           │    tenant               │
           │  - filters tool list    │
           │    to tenant catalog    │
           │  - emits assembled      │
           │    prompt audit log    │
           └────────────┬────────────┘
                        ↓
           ┌─────────────────────────┐
           │ Model call              │
           └────────────┬────────────┘
                        ↓
           ┌─────────────────────────┐
           │ Output validator        │
           │  - extracts entities    │
           │  - verifies in scope    │
           │  - blocks on violation  │
           └────────────┬────────────┘
                        ↓
                  Response to user
```

### 8.2 The tenant overlay structure

Meridian Health Southwest's overlay (overlay v4):

```yaml
tenant: meridian-southwest
overlay_version: 4
overlay_content: |
  This system serves Meridian Health Southwest. Use the following
  conventions:

  - Refer to clinicians as "providers" (Meridian Southwest convention).
  - When discussing medications, prefer generic names; include brand
    names in parentheses for clarity (Southwest formulary convention).
  - All clinical claims must include a citation to the source document.
  - For pediatric patients, defer to the Southwest pediatric guidelines
    (cross-reference the SWPG handbook in retrieval).
  - The "regional medical director" role is named Dr. Sarah Chen.

  Do not:
  - Reference Meridian Coastal Region's protocols (separate tenant).
  - Recommend therapies that are not in the Southwest formulary
    without flagging them as "off-formulary; consult before prescribing."
```

The overlay is small (~200 tokens); it's curated by Meridian Southwest's clinical informatics team; it's reviewed before deployment.

### 8.3 The retrieval enforcement

Retrieval uses Pinecone with per-tenant namespacing (cross-link to [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md)):

```python
def retrieve_for_clinical_query(context: RequestContext, query: str) -> list[Chunk]:
    embedding = embed_clinical_query(query)

    # Namespace ensures cross-tenant queries are impossible at the store level:
    results = pinecone_index.query(
        namespace=context.tenant_id,
        vector=embedding,
        top_k=12,
        filter={"document_type": ["clinical_note", "formulary", "guideline"]}
    )

    # Defense in depth: verify each chunk's tenant tag:
    for match in results.matches:
        assert match.metadata["tenant_id"] == context.tenant_id, \
            f"Cross-tenant chunk {match.id}: expected {context.tenant_id}, got {match.metadata['tenant_id']}"

    return [Chunk(...) for match in results.matches]
```

Three layers of enforcement:
- Pinecone namespace (vector store layer).
- Metadata filter on document_type (additional scoping).
- Application-layer assertion (verification).

### 8.4 The tool authorization

Each Meridian tenant has its own tool catalog:

```yaml
tenant: meridian-southwest
allowed_tools:
  - fetch_patient_summary
  - search_clinical_notes
  - draft_referral
  - check_formulary
  - schedule_appointment (requires_approval: true)
disallowed_tools:
  - billing_lookup       # Southwest uses external billing system
  - eligibility_check    # Southwest uses different vendor
```

When the Care Coordinator agent runs for a Southwest patient, the model's tool list excludes `billing_lookup` and `eligibility_check`. If the model hallucinates calls to those, the dispatch layer rejects.

### 8.5 The output validation

Every response is parsed for entity references:

- Patient IDs (matching the Meridian MRN format).
- Document IDs (matching the EHR document ID format).
- Encounter IDs (matching the encounter ID format).
- Provider IDs (matching the Meridian provider directory format).

Each referenced entity is verified to belong to the tenant. If not, the response is blocked and the operator is paged.

In the last 12 months: 3 violations caught. All were hallucinations (the model invented a plausible-looking ID that didn't exist or belonged to another tenant). No real cross-tenant data exposure.

### 8.6 The near-miss

In Q3 2025, a developer modified the retrieval function to support a new "search across tenants for analytics" workflow. The developer added a parameter `include_cross_tenant: bool = False` and modified the retrieval logic to skip the tenant filter when `include_cross_tenant=True`.

The change shipped to production. The analytics workflow used it correctly. But three days later, an unrelated PR refactored the function signature; in the refactor, the `include_cross_tenant` parameter's default was preserved but a logic bug caused the parameter to be set to `True` in one specific code path (a fallback retrieval after a primary retrieval miss).

For 4 hours, the fallback path returned cross-tenant chunks. Caught by:

- The application-layer assertion in the retrieval wrapper (the assertion in §8.3). It raised an exception when the assertion failed.
- The error logged with tenant_id mismatch detail.
- The alert fired on "tenant_filter_violation" log pattern.
- On-call engineer rolled back within 15 minutes.

No customer-visible leak (the application-layer assertion fired before any chunk reached the prompt assembler). Post-incident:

- Removed the `include_cross_tenant` parameter from the general retrieval function.
- Created a separate `analytics_cross_tenant_retrieval` function for the analytics use case, with explicit cross-tenant authorization and dedicated audit logging.
- Made the application-layer assertion non-bypassable (raises in all environments; not just dev).
- Added a CI check that flags any PR touching retrieval code for additional security review.

The lesson: a single parameter with the wrong default can undermine a multi-tenant architecture. The defense-in-depth assertion was the only thing that prevented a real incident.

### 8.7 What this architecture costs

- Engineering effort: ~3 weeks initial build (one engineer); ongoing maintenance ~5% of platform team's time.
- Latency overhead: ~40ms per request for the assembly + validation + audit logging.
- Storage: per-tenant overlay storage (~10 KB per tenant); audit log storage (~2 KB per request).
- Cost: negligible incremental LLM cost; the prompt is the same size regardless of tenant.

### 8.8 What this architecture prevents

- Zero documented cross-tenant leaks via prompt assembly in 18 months across 12 tenants.
- The Q3 2025 near-miss caught before any chunk reached the model.
- Hallucinated entity references in output caught and blocked.
- Compliance auditor's review (Q1 2026) found the controls satisfactory; no remediation required.

---

## 9. Anti-patterns

### 9.1 The implicit tenant from thread-local

**Pattern.** Tenant context lives in a thread-local or request-scoped global. Functions don't take it as a parameter; they read it from the global. A bug in setting the global (mismatched request handler, async context boundary, forgotten override) produces wrong-tenant operations.

**Corrective.** Explicit tenant parameter on every function that touches tenant data. Static analysis to catch implicit-tenant patterns.

### 9.2 The full prompt template per tenant

**Pattern.** Each tenant has their own complete system prompt. Initially attractive; drifts over time; eval coverage is N times the work; platform updates lag.

**Corrective.** Shared base + bounded tenant overlay. Overlay reviewed and tested; base updates apply universally.

### 9.3 The retrieval call with optional tenant filter

**Pattern.** Retrieval function signature has `tenant_filter` as optional. Callers usually pass it; sometimes forget. The forget-case produces cross-tenant retrieval.

**Corrective.** Tenant filter is mandatory (non-optional parameter). Retrieval cannot be called without it.

### 9.4 The cache keyed without tenant

**Pattern.** Cache key is `hash(query) + hash(retrieved_context)`. Tenant A's cached response is returned for tenant B's identical query.

**Corrective.** Cache key includes tenant_id; separate cache namespaces per tenant.

### 9.5 The hallucinated entity ignored

**Pattern.** Model produces output referencing an entity that doesn't exist in the tenant. Output is returned to user; user sees gibberish or, worse, plausible-but-wrong data; eventually correlates to wrong tenant.

**Corrective.** Output validation; entity references checked against tenant scope; violations blocked.

### 9.6 The tool dispatch trusting the model

**Pattern.** Model invokes a tool; dispatch executes without verifying the tool is in the tenant's catalog. Hallucinated tool calls succeed.

**Corrective.** Dispatch layer checks tenant catalog before execution; unknown / disallowed tools rejected.

### 9.7 The conversation copied across tenants

**Pattern.** User who works in multiple tenants copies a conversation from tenant A to tenant B as a context shortcut. Tenant A's data lands in tenant B's prompt.

**Corrective.** Conversations are tenant-scoped; cross-tenant copy requires explicit user action and is audited.

### 9.8 The "we'll add output validation later" deferral

**Pattern.** Output validation is on the backlog. "We've never seen a hallucination cause a leak; it's a low priority." Six months later, a model upgrade increases hallucination rate; an output validation gap becomes a customer-reported incident.

**Corrective.** Output validation in v1 for tenant-sensitive workloads. Sampling-based validation if full validation is cost-prohibitive.

---

## 10. Findings (sprint-assignable)

**ARCH-PTC-001 (P0). Tenant context is implicit (thread-local or request-scoped global).** Subtle bugs produce wrong-tenant operations. Refactor to explicit tenant parameter on all tenant-touching functions; add static analysis. Owner: platform.

**ARCH-PTC-002 (P0). Retrieval filter is optional.** Forget-case produces cross-tenant retrieval. Make filter mandatory; runtime enforcement that rejects unfiltered calls. Owner: AI platform.

**ARCH-PTC-003 (P0). No application-layer verification of retrieved chunk tenant tag.** Single point of failure if vector-store-side filter has a bug. Add assertion: every returned chunk's tenant_id matches request context; raise on mismatch. Owner: AI platform.

**ARCH-PTC-004 (P0). Cache keys don't include tenant_id.** Tenant A's cached response served to tenant B. Add tenant_id to all cache keys; verify with cross-tenant cache test. Owner: AI platform.

**ARCH-PTC-005 (P0). Tool dispatch doesn't verify tool is in tenant catalog.** Hallucinated tool calls succeed. Add catalog check before dispatch; reject unknown or disallowed tools. Owner: agent platform.

**ARCH-PTC-006 (P1). No per-tenant tool authorization model.** All tenants get the same tools. Implement per-tenant tool catalog; disable per-tenant where required (compliance, contract). Owner: AI platform + product.

**ARCH-PTC-007 (P1). No output validation against tenant scope.** Model hallucinations referencing wrong-tenant entities reach the user. Implement output validator; extract entity references; verify each in tenant scope. Owner: AI platform.

**ARCH-PTC-008 (P1). Per-tenant overlay storage is ad-hoc (env vars, hardcoded).** Changes are risky; no versioning. Move to versioned config store; review process for overlay changes; pre-deploy eval. Owner: AI platform + tenant ops.

**ARCH-PTC-009 (P1). Multiple prompt-assembly code paths.** Isolation enforcement inconsistent across paths. Consolidate to one assembler per use case; chokepoint enables consistent enforcement and audit. Owner: AI platform.

**ARCH-PTC-010 (P1). No prompt-assembly audit log.** Cross-tenant leak investigations have no source-of-truth. Log every assembled prompt's components (base version, overlay version, chunk IDs, conversation ID, user message); PII-redact full content. Owner: AI platform + security.

**ARCH-PTC-011 (P2). Conversation history fetched without tenant verification.** Conversation lookup by ID succeeds across tenants. Add tenant check in conversation fetch; reject mismatch. Owner: AI platform.

**ARCH-PTC-012 (P2). Per-tenant memory not scoped at storage layer.** Memory recall could span tenants. Memory store key includes tenant_id + user_id; queries filtered. Owner: AI platform.

**ARCH-PTC-013 (P2). No cross-conversation cache key isolation.** Conversation-A response cached, conversation-B serves cached. Cache key includes conversation_id (or tenant_id if conversation-agnostic). Owner: AI platform.

**ARCH-PTC-014 (P2). Tool argument validation doesn't check tenant scope.** Allowed tool called with wrong-tenant argument (e.g., wrong patient ID) succeeds. Per-tool argument validation includes tenant ownership check. Owner: tool implementers.

**ARCH-PTC-015 (P2). Cross-tenant retrieval not isolated to specific authorized code path.** Future bug could route cross-tenant retrieval to tenant-facing path. Cross-tenant retrieval lives in a separate function; auth required; tenant-facing code can't import. Owner: AI platform + security.

**ARCH-PTC-016 (P3). No CI check on retrieval code changes.** Subtle changes to retrieval logic ship without security review. CI flags any PR touching retrieval; mandatory security review. Owner: platform + security.

**ARCH-PTC-017 (P3). No periodic audit of cross-tenant access patterns.** Long-running cross-tenant access goes unnoticed. Weekly review of cross-tenant retrieval logs; per-user / per-service review. Owner: security.

**ARCH-PTC-018 (P3). Output validation latency unmeasured.** Don't know cost of validation; can't decide on sampling. Measure validation P99 latency; document the cost; decide policy (full vs sampled) per tenant tier. Owner: AI platform + observability.

---

## 11. Adoption sequencing checklist

For a team adopting per-tenant prompt and context discipline, in order:

- [ ] **Establish RequestContext at the request boundary (§2.2).** Tenant_id, user_id, request_id, auth_scopes; validated once.
- [ ] **Refactor functions to take RequestContext as explicit parameter.** Lint rules to enforce.
- [ ] **Add tenant_id to all log lines, spans, metrics, and cache keys.** Auditing depends on it.
- [ ] **Implement layered prompt-assembly (§3).** Shared base + tenant overlay + per-request context.
- [ ] **Store tenant overlays in versioned config; review process for changes.**
- [ ] **Make retrieval filter mandatory (§4.3).** Runtime enforcement; lint rules.
- [ ] **Add application-layer chunk verification (§4.1).** Every chunk's tenant_id verified before reaching prompt assembler.
- [ ] **Implement per-tenant tool catalog (§5.1).** Prompt assembler filters tool list; dispatch layer verifies.
- [ ] **Per-tool argument validation (§5.4).** Tool arguments verified against tenant scope.
- [ ] **Implement output validation (§6).** Entity extraction; tenant-scope verification; block on violation.
- [ ] **Tenant-scope conversation fetches and memory queries (§7).** Tenant_id in keys; verified on read.
- [ ] **Build prompt-assembly audit log.** Components logged for every assembled prompt.
- [ ] **Consolidate prompt-assembly to chokepoints.** One assembler per use case.
- [ ] **CI flag for retrieval-code changes.** Security review required.
- [ ] **Pre-production test:** synthetic cross-tenant queries; verify all isolation layers block.
- [ ] **Periodic audit:** cross-tenant access logs reviewed; assembly audit logs spot-checked.
- [ ] **Document the architecture** for new team members; multi-tenant onboarding includes this doc.

---

## 12. References

**In this folder.**
- [isolation-models.md](./isolation-models.md) — the three isolation models; this doc applies most directly to shared-model-isolated-data.
- [per-tenant-vector-namespacing.md](./per-tenant-vector-namespacing.md) — data-layer namespacing that this doc composes with at the retrieval boundary.
- [cross-tenant-leakage-prevention.md](./cross-tenant-leakage-prevention.md) — the broader leakage-path catalog; this doc covers the prompt-assembly subset in depth.
- [noisy-neighbor-mitigation.md](./noisy-neighbor-mitigation.md) — performance isolation that complements this doc's data isolation.

**Elsewhere in this repo.**
- [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) — deeper tool authorization patterns that this doc references at §5.
- [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — deeper retrieval scoping patterns.
- [integration-architecture/tool-call-architecture.md](../integration-architecture/tool-call-architecture.md) — where tools live; the dispatch layer is part of that architecture.
- [reference-systems/patient-api-ai-assist.md](../reference-systems/patient-api-ai-assist.md) — the worked multi-tenant SaaS reference system.

**Sibling repos.**
- [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) — engineering-side prompt versioning that this doc references at §3.4.
- [ai-engineering-reference-architecture / prompt-engineering / prompts-as-code-discipline.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompts-as-code-discipline.md) — broader prompts-as-code that informs overlay management.
- [ai-engineering-reference-architecture / agent-engineering / memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) — engineering-side memory patterns; this doc covers the multi-tenant overlay.
- [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — engineering-side retrieval patterns; this doc covers the tenant-aware overlay.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — prompt-injection attacks that try to make the model violate tenant scope.

**External.**
- OWASP LLM Top 10 — particularly LLM02 (Insecure Output Handling) and LLM06 (Sensitive Information Disclosure).
- Anthropic / OpenAI documentation on system-prompt design and tool specification.
- Multi-tenant SaaS security literature — the AI-specific patterns here build on conventional multi-tenant patterns.
