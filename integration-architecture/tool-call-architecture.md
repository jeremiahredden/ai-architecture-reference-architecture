# Tool Call Architecture

> **Audience.** Architects designing where the agent's tools live. Tech leads weighing in-process functions vs HTTP services vs MCP servers. Anyone whose tool-call latency is high and isn't sure if the answer is moving tools closer or further. **Scope.** The *architectural* decision: where tools live; trade-offs across latency, security boundary, language coupling, deployment independence; tool-discovery pattern; tool-schema-as-contract discipline; MCP-adoption decision tree. Not the tool-design engineering (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md)). Not the authorization architecture (see [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Tools are the agent's hands. Where those hands live — in the same process as the agent's runner, behind an HTTP service, behind an MCP server, or some mixture — is an architectural decision with material consequences for latency, security, deployment cadence, language flexibility, and operational complexity.

The choice often defaults rather than deliberates:

- "We're using LangChain; tools are functions in the LangChain process." (In-process by accident.)
- "We have an existing service that does X; just call it from the tool." (HTTP service by inheritance.)
- "We heard MCP is the way; let's MCP everything." (MCP by fashion.)

Each default has cases where it's right and cases where it's wrong. The deliberate decision examines the specific tool, the specific risk profile, the specific deployment topology — and picks per tool rather than per platform.

The architectural framing matters because tool decisions accumulate:

- A team that defaults to in-process discovers they can't reuse tools across language boundaries (Python agent + Go service that needs the same capability).
- A team that defaults to HTTP discovers latency adds up across many tool calls.
- A team that defaults to MCP discovers operational complexity (every tool is its own server to maintain).

This document covers the architectural decision; per-tool considerations that compose into a coherent system.

This document is opinionated about four things:

1. **The decision is per-tool, not platform-wide.** Each tool's right home depends on its specific characteristics; a one-size-fits-all default produces wrong choices.
2. **In-process is the default for the team's own tools.** Lowest latency, simplest deployment, language-coupled but the agent and tool are usually the same language anyway.
3. **MCP is for cross-language reuse, isolation, and ecosystem.** Adopt where its specific benefits apply; not because it's the trendy pattern.
4. **The tool's schema is the contract.** Per [structured-output-patterns.md](../reference-patterns/structured-output-patterns.md) and [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md), the schema is the integration surface.

Structure: (2) the four tool-location options; (3) the trade-off matrix; (4) the tool-discovery pattern; (5) the schema-as-contract discipline; (6) the MCP-adoption decision tree; (7) hybrid catalogues (mixing patterns); (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four tool-location options

The architectural choices.

### 2.1 In-process function

The tool is a function in the agent's process:

```python
@tool
def fetch_patient(patient_id: str, context: CallContext) -> PatientDemographics:
    # Implementation
    return client.get_patient(patient_id)
```

The agent's runner imports the tool; calls it directly.

**Characteristics.**
- Latency: function call (microseconds).
- Deployment: bundled with the agent.
- Language: same as the agent.
- Security: in-process; same memory; no boundary.
- Operational cost: none beyond agent's own.

**When right.**
- Tool is owned by the same team as the agent.
- Tool is in the same language.
- Low-risk capability (no need for security boundary).
- Tool's deployment cadence aligns with the agent.

### 2.2 HTTP service

The tool is exposed by a separate HTTP service:

```
Agent → HTTP(tool API) → Service → Response → Agent
```

The agent's tool implementation is an HTTP client wrapper.

**Characteristics.**
- Latency: network round-trip (typically 10-100ms).
- Deployment: independent service.
- Language: tool can be any language.
- Security: process boundary; potentially network boundary.
- Operational cost: separate service to deploy, monitor, scale.

**When right.**
- Tool exists as a service for other reasons (legacy API, microservice in the team's architecture).
- Tool's team owns it independently from the agent's team.
- Tool's deployment cadence differs from the agent's.

### 2.3 MCP server

The tool is exposed via Model Context Protocol (MCP):

```
Agent → MCP client → MCP server (separate process) → Tool → Response → Agent
```

MCP is a standardised protocol for tool exposure; multiple agent runtimes can consume the same MCP server.

**Characteristics.**
- Latency: process boundary, sometimes network (typically 5-50ms locally, 10-100ms remote).
- Deployment: separate process/server; standard MCP interface.
- Language: tool can be any language.
- Security: process boundary; often used for sandboxing.
- Operational cost: MCP server's lifecycle.
- Ecosystem: many third-party MCP servers available.

**When right.**
- Tool is reused across multiple agents (potentially different teams / languages).
- Tool needs process isolation (sandboxed code execution; capability-restricted).
- Third-party MCP server provides the capability.
- Tool's deployment is independent from agents.

### 2.4 Hybrid (in-process front; service backend)

A common pattern: in-process tool wrapper that calls a service:

```python
@tool
def fetch_patient(patient_id: str, context: CallContext) -> PatientDemographics:
    # In-process wrapper
    return patient_service.client.get_patient(patient_id)
```

The agent sees a clean in-process tool; the service is an implementation detail.

**Characteristics.**
- Latency: function call + service call (same as HTTP service).
- Deployment: agent and service deploy independently; agent's tool wrapper is small.
- Language: agent is one; service is whatever.
- Operational: service runs on its own.

**When right.**
- The team wants in-process tool ergonomics for the agent.
- The actual implementation needs to be in a service for other reasons.

Most production "in-process" tools are actually this hybrid — the function is in-process; the work it does is in a service.

### 2.5 The decision matrix

| Criterion | In-process | HTTP service | MCP server |
| --- | --- | --- | --- |
| Latency overhead | None | 10-100ms | 5-100ms |
| Operational complexity | Minimal | Service to maintain | MCP server to maintain |
| Language flexibility | Same as agent | Any | Any |
| Cross-agent reuse | Limited (same language / process) | Yes | Yes (standardised) |
| Security isolation | None | Process | Process / sandbox |
| Deployment independence | No (bundled with agent) | Yes | Yes |
| Ecosystem | Team's own | Team's services | Third-party MCP servers available |

The matrix informs the per-tool decision.

### 2.6 The "all tools in one place" temptation

Some teams pick one pattern and force all tools through it. The result:

- All in-process: tool reuse across languages becomes impossible; isolated capabilities can't be sandboxed.
- All HTTP service: latency adds up; deployment overhead per tool; over-engineering for simple tools.
- All MCP: operational complexity per tool; many MCP servers to maintain.

The right pattern: per-tool decision, hybrid catalogue.

### 2.7 The "tools are functions; everything is in-process" framing

Some agent frameworks (LangChain, CrewAI, Vercel AI SDK) present tools as functions; the implicit assumption is in-process.

The framing is fine for simple cases; production teams typically need the hybrid (in-process wrapper calling a service) or explicit MCP for some tools.

---

## 3. The trade-off matrix

The dimensions that drive the per-tool decision.

### 3.1 Latency

Per tool call:

- In-process: < 1ms (function call).
- HTTP local: 5-20ms.
- HTTP cross-region: 50-200ms.
- MCP local: 5-30ms.
- MCP remote: 20-100ms.

For features with many tool calls per request, the latency adds up. Care Coordinator has 5-15 tool calls per request; even at 20ms per HTTP call, 100-300ms total tool-call overhead.

If latency is the constraint, in-process wins; if other factors matter more, the latency is acceptable.

### 3.2 Security boundary

In-process: no boundary; tool's code runs in agent's memory space.

HTTP service: process boundary; tool's failures don't crash the agent; tool's vulnerabilities don't directly affect the agent.

MCP server: same as HTTP plus often added isolation (sandboxing, capability restriction).

For tools with security-sensitive operations (code execution, filesystem access, secret handling), the boundary matters. For tools with read-only data access, less so.

### 3.3 Language coupling

In-process: agent and tool same language.

HTTP / MCP: tool can be any language.

If the team's existing capability is in a different language (e.g., agent in Python; specialised analyzer in Rust), HTTP / MCP is the natural fit.

### 3.4 Deployment independence

In-process: tool deploys with agent; changes require agent redeploy.

HTTP / MCP: tool deploys independently; tool team controls cadence.

For tools owned by different teams or with different cadences, independence matters.

### 3.5 Cross-agent reuse

In-process: tool can be shared across agents in the same language, but each agent has its own instance.

HTTP service: one service serves multiple agents.

MCP server: one MCP server serves multiple agents (potentially across languages).

For shared capabilities (a "search documents" tool used by multiple agent features), HTTP or MCP is the natural fit.

### 3.6 Operational complexity

In-process: minimal beyond the agent's complexity.

HTTP service: full service overhead (deployment, scaling, monitoring, on-call).

MCP server: similar to service plus the MCP protocol layer.

Per-tool decision: the operational cost is per tool the team takes on.

### 3.7 Cost

In-process: minimal incremental cost.

HTTP / MCP: infrastructure cost for the service / MCP server (compute, storage, network).

For low-volume tools, the difference is negligible; for high-volume, the service cost can be material.

### 3.8 Failure isolation

In-process: tool bug crashes the agent.

HTTP / MCP: tool failure returns error; agent's failure handling kicks in.

For tools that might fail (external dependencies, complex logic), HTTP / MCP isolates.

### 3.9 The per-tool decision

For each tool, the team asks:

- Latency budget acceptable for HTTP / MCP?
- Security boundary needed?
- Tool in same language as agent?
- Reused across agents?
- Owned by different team?
- Failure isolation needed?

The answers point to the right location. Most tools end up in-process (low-latency, same language, same team); a minority warrant HTTP or MCP.

---

## 4. The tool-discovery pattern

How agents know what tools exist.

### 4.1 Static catalogue

The agent's tool list is hardcoded at deploy time:

```python
agent = Agent(
    tools=[
        fetch_patient,
        search_clinical_notes,
        propose_followup_appointment,
        escalate_to_care_manager,
    ]
)
```

**Pros.**
- Predictable; the agent's capabilities are known.
- Simple; no discovery infrastructure.

**Cons.**
- Adding a tool requires agent deploy.
- No dynamic capability extension.

**When right.** Most production agents. The agent's capabilities are deliberately scoped; static suffices.

### 4.2 Dynamic discovery (MCP-style)

The agent discovers tools at runtime by asking servers:

```
Agent startup → MCP server.list_tools() → Tools available → Agent's catalogue
```

**Pros.**
- New tools available without agent code change.
- Per-environment / per-tenant tool catalogues.

**Cons.**
- Less predictable; the agent's capabilities depend on what's discovered.
- Server availability affects agent startup.
- Security: the agent trusts whatever the server says is a tool.

**When right.** Dynamic environments where tool catalogues genuinely vary; less common in production agent features.

### 4.3 The "static + per-tenant overlay" pattern

The base tool catalogue is static; per-tenant additions are config-driven:

```python
agent = Agent(
    tools=base_tools + tenant_specific_tools(tenant_id),
)
```

**Pros.**
- Base behaviour predictable.
- Tenant customization where needed.
- No dynamic discovery complexity.

**When right.** Multi-tenant agents where tenants have different tool subsets.

### 4.4 The schema-driven catalogue

Even static catalogues can be expressed declaratively:

```yaml
tools:
  - name: fetch_patient
    schema: schemas/fetch_patient_v3.json
    implementation: meridian.tools.patient.fetch_patient
  - name: search_clinical_notes
    schema: schemas/search_clinical_notes_v2.json
    implementation: meridian.tools.search.notes_search
```

The catalogue is data; the agent loads it at startup.

### 4.5 The catalogue ownership

Per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) and [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md): the tool registry is platform; tools have owners; the catalogue is versioned.

Discovery / static is the runtime mechanism; the catalogue's *governance* is the same regardless.

### 4.6 The "agent discovers from the internet" anti-pattern

Some experimental setups have agents discover tools from arbitrary MCP servers at runtime. The security implications: the agent trusts any discovered tool's description.

For production: the catalogue is curated. Discovered tools (from third-party MCP) are reviewed before added.

### 4.7 The catalogue as documentation

The tool catalogue is also documentation of the agent's capabilities. For:

- Engineers maintaining the agent.
- Security review.
- Audit / compliance.
- New team members.

A discoverable, structured catalogue serves multiple purposes.

---

## 5. The schema-as-contract discipline

The tool's schema is the integration surface.

### 5.1 What's in the schema

Per the engineering sibling's [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md):

- Tool name.
- Description (model-facing).
- Input schema (arguments, types, constraints).
- Output schema (success and error envelopes).
- Side-effect classification.

The schema is the contract between the agent and the tool, regardless of where the tool lives.

### 5.2 Schema versioning

Tools evolve; schemas version. Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md):

- Major bump for breaking changes.
- Minor for additive.
- Patch for fixes.

Agents pin to specific schema versions; updates are coordinated.

### 5.3 The cross-language schema

For HTTP / MCP tools, the schema is the cross-language contract:

- Python agent + Go tool.
- TypeScript agent + Rust tool.

The schema in JSON Schema (or equivalent) is language-neutral; bindings are generated.

### 5.4 The schema validation

At the boundary:

- Agent validates the tool's output against schema.
- Tool validates the agent's input against schema.

Validation failures are structured errors; per [error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md).

### 5.5 The schema as authorization input

Per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md): the schema describes the tool's side effects; authorization policies reference the schema.

### 5.6 The schema as documentation

The schema documents:

- What the tool does (description).
- How to call it (input schema).
- What it returns (output schema).
- Failure modes (error envelope).

For tool authors and consumers; for the model selecting the tool; for the security reviewer.

### 5.7 The "schema drift" failure

A common bug: tool's actual behaviour drifts from its schema:

- Code returns an additional field; schema doesn't list it.
- Code changes an input requirement; schema doesn't reflect.

Mitigations:

- Schema generation from code (where feasible).
- Schema verification in CI (the tool's tests verify it matches the schema).
- Drift detection in production (validator catches outputs that don't match).

### 5.8 The MCP schema convention

MCP standardises the schema format (a flavor of JSON Schema). MCP servers expose schemas; agents validate.

For non-MCP HTTP services, the team picks a schema convention (OpenAPI, JSON Schema). Consistency matters more than format choice.

---

## 6. The MCP-adoption decision tree

When MCP is the right pattern.

### 6.1 MCP's specific value

MCP (Model Context Protocol) is a standardised protocol for exposing tools to AI agents. Its specific value:

- **Standardisation.** One protocol; many agents can consume.
- **Ecosystem.** Many third-party MCP servers (filesystem, git, slack, etc.) available.
- **Sandboxing.** Process boundary supports capability restriction.
- **Discovery.** Built-in tool discovery (per section 4.2).

### 6.2 When MCP is right

- **Cross-language reuse.** Tool serves Python, TypeScript, and Go agents; MCP's standardisation pays off.
- **Sandboxing.** Code-execution or filesystem-access tools that warrant capability restriction.
- **Third-party tool.** Existing MCP server for the capability; integration is faster than building.
- **Independent deployment cadence.** Tool team ships independently; MCP is the contract.

### 6.3 When MCP isn't justified

- **Single agent + single language.** In-process or HTTP suffices; MCP's standardisation adds no value.
- **Latency-critical.** MCP adds process boundary; latency may not be acceptable.
- **Team has no MCP operational experience.** The MCP server is its own runtime; team takes on operations.
- **Simple tool.** In-process function is simpler.

### 6.4 The decision tree

```
Tool needs cross-language reuse?
  Yes → MCP (or HTTP if standard isn't critical)
  No → continue

Tool needs sandboxing / capability restriction?
  Yes → MCP (or sandboxed service)
  No → continue

Third-party MCP server exists for the capability?
  Yes → MCP (use existing)
  No → continue

Tool owned by different team with different cadence?
  Yes → HTTP or MCP (independence)
  No → continue

Tool same language, same team, simple capability?
  Yes → In-process
  No → HTTP (if latency tolerable) or MCP (if security boundary needed)
```

The tree pushes toward in-process by default; MCP only when its specific benefits apply.

### 6.5 The "MCP for everything" anti-pattern

Teams that have adopted MCP enthusiastically:

- Every tool is its own MCP server.
- Each MCP server has its own deployment, monitoring, on-call.
- Operational overhead scales with tool count.

For most agents, this is over-engineering. MCP for specific tools; in-process for the rest.

### 6.6 The third-party MCP server review

Adopting a third-party MCP server:

- **Capability scope.** What can it do? Are there risks?
- **Authentication.** What credentials does it need? Where?
- **Trust model.** The MCP server can run any code in its process; trust accordingly.
- **Maintenance.** Who patches it? On what cadence?
- **Audit.** What does it log?

Per the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)'s `agent-security/mcp-server-hardening.md` (planned).

### 6.7 The MCP server's own operational discipline

A MCP server in production is a service with all the operational concerns:

- Health checks.
- Restart on crash.
- Versioning.
- Observability.
- Scaling (if high-volume).

The team owning the MCP server (whether internal or third-party-integrated) takes on these.

### 6.8 The "MCP as security boundary" framing

For risky capabilities, MCP provides a process boundary that in-process doesn't:

- The agent's process can't access the MCP server's memory.
- The MCP server can run with lower privileges.
- Capability restrictions enforced at the OS / container level.

For tools whose capability is risky (code execution, filesystem, network), the boundary matters more than the operational cost.

---

## 7. Hybrid catalogues (mixing patterns)

The real-world architecture.

### 7.1 Most production catalogues are hybrid

For a typical AI feature's tool catalogue (10-15 tools):

- 70-80% in-process (or in-process wrapper around team's services): patient lookups, internal data access, application-domain operations.
- 10-20% HTTP service: tools that call external APIs or specialised services.
- 0-10% MCP: tools needing process isolation or third-party MCP integration.

The mix reflects per-tool decisions.

### 7.2 The catalogue's coherent surface

Even with mixed locations, the catalogue presents one coherent interface to the agent:

```python
agent = Agent(
    tools=[
        fetch_patient,                  # in-process
        search_clinical_notes,          # in-process wrapper around RAG service
        check_drug_interaction,         # HTTP service (external pharmacy API)
        execute_code_snippet,           # MCP (sandboxed)
        escalate_to_care_manager,       # in-process
    ]
)
```

The agent doesn't know (or care) where each tool lives; it sees the schema-defined interface.

### 7.3 The observability across patterns

Per-tool observability per [agent-step-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/agent-step-instrumentation.md):

- Span per tool call (regardless of location).
- Latency attributed per tool.
- Failure category per tool.

The trace is uniform; the location is metadata.

### 7.4 The tool registry as unification

Per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md): the tool registry mediates all dispatches:

- Authorization checked.
- Schema validated.
- Dispatched to the actual implementation (wherever it lives).
- Result returned; observed.

The registry presents one interface; per-tool location is implementation detail.

### 7.5 The migration between patterns

A tool may migrate over time:

- Starts in-process (simple).
- Moves to HTTP service (when reused across agents).
- Becomes MCP (when reused across languages or needs sandboxing).

The schema stays consistent; the location changes. With the registry pattern, the agent's code doesn't need to change.

### 7.6 The cost analysis per catalogue

For a hybrid catalogue:

- In-process tools: minimal infrastructure cost.
- HTTP services: per-service infrastructure.
- MCP servers: per-server infrastructure.

A 15-tool catalogue with 3 HTTP + 1 MCP: ~$200-500/month additional infrastructure.

Per-tool cost worth tracking; sometimes a service can be retired or merged.

### 7.7 The team-coordination per pattern

- In-process tools: team's own.
- HTTP services: team or another team's; coordinate via API contracts.
- MCP servers: same as HTTP, plus protocol coordination.

Cross-team tools have coordination overhead; the architecture acknowledges it.

---

## 8. Worked Meridian example

Meridian's tool architecture.

### 8.1 The Care Coordinator's tool catalogue

12 tools:

| Tool | Location | Why |
| --- | --- | --- |
| fetch_patient | In-process (wrapper around patient-service) | Same team; same language; latency matters |
| search_clinical_notes | In-process (wrapper around RAG service) | Same team |
| search_clinical_guidelines | In-process (wrapper around RAG service) | Same team |
| check_drug_interaction | HTTP service (external pharmacy API integration) | External dependency; failure isolation |
| propose_followup_appointment | In-process | Same team |
| send_patient_message | In-process | Same team |
| schedule_appointment | HTTP service (scheduling system is older Java service) | Different team's service |
| log_action_in_ehr | HTTP service (EHR integration team) | Different team |
| escalate_to_care_manager | In-process | Same team |
| read_log_file | MCP (sandboxed) | Capability restriction; potential filesystem access |
| fetch_file_metadata | MCP (sandboxed) | Same MCP server as read_log_file |
| reference_lookup (RxNorm/SNOMED) | HTTP service | External knowledge base |

Mostly in-process; some HTTP for cross-team services; MCP only for the two filesystem-access tools.

### 8.2 The MCP decision for the filesystem tools

The team considered exposing filesystem access as in-process tools. Risk assessment:

- Filesystem access from the agent's process could touch sensitive paths.
- A bug in the agent could read files it shouldn't.
- Compromise of the agent process could exfiltrate.

The MCP server runs with restricted filesystem access (specific directories only); the agent process has none.

The MCP overhead (server deployment, monitoring) is justified by the security boundary.

### 8.3 The HTTP service decisions

The 3 HTTP tools (scheduling, EHR logging, knowledge-base lookups) each have specific reasons:

- **Scheduling:** the scheduling team owns the service; their cadence differs; their language is Java.
- **EHR logging:** EHR integration team owns; FedRAMP boundary requires separate process.
- **Knowledge-base lookups:** RxNorm/SNOMED are external; specialised library wraps them in a service.

In-process would have required different teams to embed code in the agent's process; not viable.

### 8.4 The catalogue management

- **Static catalogue.** The 12 tools are loaded from a YAML config at agent startup.
- **Per-tenant overlay.** Some tenants have additional tools (e.g., one hospital's specialised lab-result tool).
- **Schema repository.** Each tool's schema in `meridian-tool-schemas` repository; versioned.
- **Tool registry.** `meridian-tools-registry` mediates dispatch; authorization; observability.

### 8.5 The latency per pattern

Per Care Coordinator request (5-10 tool calls):

- In-process tools: ~5ms per call.
- HTTP service tools: ~50ms per call (local network).
- MCP tools (~rare): ~30ms per call.

Total tool overhead: ~100-300ms per request. Acceptable within the 6s latency budget.

### 8.6 The observability

Per tool call: span with attributes:

- tool_name.
- tool_location (in-process / http / mcp).
- latency.
- success/failure.
- cost (if applicable; e.g., external API charges).
- tenant_id (for cost attribution).

Aggregations per tool, per location pattern.

### 8.7 The migration history

In the last 2 years:

- **fetch_patient** started as HTTP service; migrated to in-process wrapper when the patient-service team's API stabilised.
- **read_log_file** started as in-process; moved to MCP after a near-miss with in-process filesystem access.
- **schedule_appointment** stayed HTTP throughout (cross-team).

Per-tool migration as conditions changed; schema kept consistent.

### 8.8 What worked

- **Per-tool decisions.** No platform-wide default; per tool's characteristics.
- **In-process default.** Most tools are in-process; latency low.
- **MCP for sandboxing.** Justified by security boundary; not by fashion.
- **Tool registry as choke point.** Uniform observability; uniform authorization.

### 8.9 What didn't work initially

- **All-MCP initial design.** Early design proposed MCP for all tools; rejected after eval (latency, operational cost too high for the benefit).
- **In-process filesystem tools.** Initial implementation; corrected after security review.
- **Schema drift.** First tool deployments had schema and code drift; CI tests added to verify.

---

## 9. Anti-patterns

### 9.1 "MCP for everything"

Every tool is its own MCP server; operational overhead scales with tool count.

**Corrective.** Per-tool decision per section 6.4.

### 9.2 "In-process for security-sensitive tools"

Filesystem access, code execution, credential handling in-process. Bug or compromise exposes more than necessary.

**Corrective.** MCP or sandboxed service for these tools per section 6.2.

### 9.3 "HTTP service for simple tools"

Tool that's a 10-line function becomes an HTTP service. Latency, deployment, monitoring overhead for no benefit.

**Corrective.** In-process for simple tools per section 2.5.

### 9.4 "No tool registry"

Tools dispatched directly by agent code; no centralised authorization or observability.

**Corrective.** Tool registry per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md).

### 9.5 "Schema drift"

Tool's actual behaviour drifts from documented schema; agent's expectations break.

**Corrective.** Schema verification per section 5.7.

### 9.6 "Dynamic discovery in production"

Agent discovers tools from arbitrary servers at runtime; trust model unclear.

**Corrective.** Static + per-tenant overlay per section 4.3.

### 9.7 "Third-party MCP without review"

Adopted MCP server without security / operational review.

**Corrective.** Per section 6.6.

### 9.8 "Tool location dictated by framework default"

Framework's default location pattern used for all tools; per-tool consideration skipped.

**Corrective.** Per-tool decision per section 2.6.

---

## 10. Findings (sprint-assignable)

### ARCH-TCA-001 — Severity: Critical
**Finding.** Security-sensitive tools (filesystem, code execution) in-process; no sandboxing.
**Recommendation.** MCP or sandboxed service per section 6.2.
**Owner.** architecture + security, sprint N+1.

### ARCH-TCA-002 — Severity: Critical
**Finding.** No tool registry; tools dispatched directly; no central authorization or observability.
**Recommendation.** Tool registry per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md).
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TCA-003 — Severity: High
**Finding.** All-MCP architecture; operational overhead high for non-security-critical tools.
**Recommendation.** Per-tool decision per section 6.4; in-process for simple tools.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-TCA-004 — Severity: High
**Finding.** Schema drift; tool behaviour doesn't match documented schema.
**Recommendation.** Schema verification in CI per section 5.7.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TCA-005 — Severity: High
**Finding.** Tool location decisions ad-hoc; no documented per-tool reasoning.
**Recommendation.** Per-tool decision documentation per section 3.9; design review.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-TCA-006 — Severity: High
**Finding.** Third-party MCP servers adopted without security review.
**Recommendation.** Per section 6.6.
**Owner.** ai-platform-eng + security, sprint N+2.

### ARCH-TCA-007 — Severity: Medium
**Finding.** HTTP service tool latency not budgeted; cumulative tool-call latency exceeds feature SLO.
**Recommendation.** Latency budgets per tool per section 3.1.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TCA-008 — Severity: Medium
**Finding.** Tool observability inconsistent across patterns; in-process vs HTTP vs MCP have different metrics.
**Recommendation.** Uniform observability via registry per section 7.3.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-TCA-009 — Severity: Medium
**Finding.** Tool schema not versioned; agents pin to "current" with no protection from changes.
**Recommendation.** Schema versioning per section 5.2.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TCA-010 — Severity: Medium
**Finding.** Tool failure isolation absent for in-process tools; tool exceptions can crash the agent.
**Recommendation.** Exception handling in tool wrappers; structured errors per [error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md).
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TCA-011 — Severity: Medium
**Finding.** Cross-team tool coordination informal; tool team's changes break agent.
**Recommendation.** Schema-as-contract discipline per section 5 / [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md).
**Owner.** ai-platform-eng + tool-owning teams, sprint N+3.

### ARCH-TCA-012 — Severity: Medium
**Finding.** MCP server operational monitoring absent; outage not detected.
**Recommendation.** Standard service monitoring per section 6.7.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-TCA-013 — Severity: Medium
**Finding.** Tool catalogue not documented; team's understanding ad-hoc.
**Recommendation.** Catalogue as documentation per section 4.7.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TCA-014 — Severity: Medium
**Finding.** Per-tool cost not tracked; some HTTP / MCP tools more expensive than realised.
**Recommendation.** Per-tool cost in observability per section 8.6.
**Owner.** ai-platform-eng + finance, sprint N+4.

### ARCH-TCA-015 — Severity: Low
**Finding.** Dynamic discovery used in production; trust model unclear.
**Recommendation.** Static + per-tenant overlay per section 4.3.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TCA-016 — Severity: Low
**Finding.** Cross-language tool reuse not designed; tools rebuilt per language.
**Recommendation.** MCP or HTTP for cross-language tools per section 6.2.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-TCA-017 — Severity: Low
**Finding.** Tool migration path (in-process → HTTP → MCP) not designed; conditions change but tools don't migrate.
**Recommendation.** Per section 7.5; migration plan when conditions warrant.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-TCA-018 — Severity: Low
**Finding.** Tool sunset process informal; deprecated tools linger in catalogue.
**Recommendation.** Per-tool deprecation lifecycle.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team designing a tool catalogue:

- [ ] **Sprint 0 — capability inventory.** What does the agent need to do?
- [ ] **Sprint 0 — per-tool location decision.** Per section 6.4 for each.
- [ ] **Sprint 1 — schema definitions.** Per [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md).
- [ ] **Sprint 1 — registry implementation.** Per [tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md).
- [ ] **Sprint 2 — in-process tools.** Implement; integrate with registry.
- [ ] **Sprint 2 — HTTP / MCP tools.** As needed per decisions.
- [ ] **Sprint 3 — observability.** Per-tool spans; cost attribution.
- [ ] **Sprint 3 — schema versioning.** Pinning; CI verification.
- [ ] **Sprint 4 — sandboxing.** For tools requiring it.
- [ ] **Sprint 4 — cross-team contracts.** For HTTP / MCP cross-team tools.
- [ ] **Ongoing — quarterly review.** Tool usage, cost, latency; consider migrations.

For a team retrofitting:

- [ ] **Sprint 0 — audit.** What tools; where do they live; what's the rationale.
- [ ] **Sprint 1 — fix worst gaps.** Often security-sensitive tools in-process.
- [ ] **Sprint 2 — registry.** If absent.
- [ ] **Sprint 3 — schema discipline.** If informal.

A team that completes the sequence has a tool architecture that's right-sized per tool. A team that doesn't has either over- or under-engineered tools.

---

## 12. References

- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — tool calls follow integration shape decisions.
- [event-driven-ai-integration.md](./event-driven-ai-integration.md) — event-driven tool patterns.
- [human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md) — HITL tools.
- [backpressure-and-queueing.md](./backpressure-and-queueing.md) — tool capacity.
- [integration-failure-patterns.md](./integration-failure-patterns.md) — tool failure handling.
- [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md) — async tool results.
- [reference-patterns/agent-topologies.md](../reference-patterns/agent-topologies.md) — agent topologies that affect tool catalogue design.
- [reference-patterns/multi-model-orchestration.md](../reference-patterns/multi-model-orchestration.md) — orchestration may involve tools.
- [guardrails-and-policy-architecture/tool-call-authorization.md](../guardrails-and-policy-architecture/tool-call-authorization.md) — authorization layer.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — policies that govern tools.
- [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md) — schema-as-contract pattern.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md) — engineering depth on tool design.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — tool failure handling.
- Sibling repo: ai-security-reference-architecture `agent-security/mcp-server-hardening.md` (planned) — MCP security.
- Model Context Protocol (MCP) — Anthropic specification.
- OpenAI function calling, Anthropic tool use — provider-native tool interfaces.
