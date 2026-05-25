# AI Gateway Pattern

> **Audience.** Architects designing or refactoring the platform layer that all AI invocations flow through. Tech leads weighing build-vs-buy for an AI gateway. **Scope.** The *architectural* pattern of a thin gateway as the chokepoint for every AI call — where guardrails, routing, cost accounting, rate limiting, and audit attach. Not the engineering of individual components (the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `observability-and-telemetry/llm-call-instrumentation.md` owns the per-call wrapper; `cost-and-finops/cost-budget-circuit-breaker.md` owns the circuit pattern). **Worked client.** Meridian Health. **Companion docs.** [tool-call-authorization.md](./tool-call-authorization.md) — the parallel pattern on the tool side; [retrieval-scope-enforcement.md](./) (coming) — the parallel pattern on the retrieval side; [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the architecture this gateway serves.

---

## 1. Why this document exists

Most production AI systems in 2026 grow guardrails, cost telemetry, observability, and rate limiting *piecemeal*. The chat feature has a moderation API call wrapped around its LLM invocation. The clinical-knowledge-worker has its own cost-logging. The batch-classification job has its own rate limiter. Each component is reasonable; the union is inconsistent, hard to operate, and impossible to audit uniformly.

The corrective architectural pattern is the *AI gateway*: a thin platform component that every model call passes through. The gateway is not a deep framework; it is a small piece of architecture whose value comes from being the single chokepoint. By making every call traverse the gateway, the team gets uniform: authentication, per-tenant routing, rate limiting, cost accounting, input filtering, output filtering, logging, observability, and circuit-breaker enforcement.

The Care Coordinator's ADR-CARE-007 (guardrail placement) commits to the AI gateway as one of the three architectural chokepoints (the others being the tool registry and the retrieval-client wrapper). This document is the depth on the gateway pattern specifically — what it is, what it does, how to build it (or buy it), what it must not become.

This document is opinionated about three things:

1. **The gateway is thin.** It is a small piece of code (or a small managed service) whose job is to enforce uniformity and observability — not to do business logic, not to do prompt engineering, not to do orchestration. A thick gateway becomes its own maintenance burden.
2. **All AI invocations go through it.** No special paths, no exceptions, no "this batch job needs to skip the gateway because it's bulky." Exceptions accumulate and the gateway's value evaporates.
3. **The gateway's contract is stable across providers and across time.** Providers change; models deprecate; new providers launch. The gateway absorbs the variability; consumers see a stable interface.

Structure: (2) what the gateway is and is not; (3) the responsibilities; (4) the internal API contract; (5) build-vs-buy decision; (6) integration with adjacent platform components; (7) the deployment shape; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. What the gateway is and is not

### 2.1 What it is

The AI gateway is a platform-level service (or library, depending on deployment shape) that sits between application code and model providers. It is the single API surface for AI calls within the platform.

- **A chokepoint.** Every model call flows through it. Lint rules and code review ensure no calling code bypasses it.
- **A uniformity layer.** Consumers see a single interface (the gateway's API), not many provider-specific SDKs.
- **An enforcement point.** Authentication, authorization, rate limiting, cost circuit breakers, input/output filtering all attach here.
- **An observability point.** Every call is instrumented uniformly with traces, spans, and cost telemetry.
- **A routing layer.** The gateway dispatches to the right provider, model, and deployment based on the request.

### 2.2 What it is not

- **An LLM provider.** It does not generate completions; it dispatches to providers that do.
- **An agent framework.** It does not run agent loops; it runs single calls. The agent loop runner (covered in the engineering sibling's `agent-engineering/`) calls the gateway repeatedly.
- **A prompt engineering tool.** It does not assemble prompts; consumers pass complete messages.
- **A guardrail provider.** Some guardrails (input filtering, output filtering) attach at the gateway, but the guardrail logic itself is owned elsewhere (vendor moderation, custom classifiers).
- **A workflow engine.** It does not orchestrate multi-step workflows; consumers (workflows, agents) call it for individual steps.

The temptation is to grow the gateway into all of these. Resist. The thin gateway is the maintainable gateway.

### 2.3 The line between gateway and consumer

The gateway provides:
- A `call_llm` (or equivalent) method.
- Per-call attribute capture.
- Routing to the dispatched provider.
- Enforcement of platform-level controls.

The consumer provides:
- The messages to send (prompt assembly is the consumer's job).
- The model and parameters to use (model selection is the consumer's job).
- The tools to make available (tool definitions come from the tool registry, but the consumer specifies which ones are exposed for this call).
- The context (tenant, user, feature, session, trace) so the gateway can attribute and enforce.

This boundary is the design discipline. Crossing it produces a fat gateway that consumers depend on for things they should own themselves.

---

## 3. The responsibilities

The gateway has eight concrete responsibilities. Each is uniform across the platform because the gateway is the chokepoint.

### 3.1 Authentication and identity propagation

The gateway authenticates the calling code (often via the platform's existing identity model — Okta-backed JWTs in the Meridian case) and propagates the identity to downstream concerns: cost attribution, audit logging, authorization decisions.

Authentication failures are rejected at the gateway with a structured error; downstream services never see un-authenticated calls.

### 3.2 Per-tenant routing

The gateway resolves the tenant from the call context and routes accordingly. For most tenants on shared infrastructure, this is a logical-routing decision (the right provider deployment, the right model deployment, the right BAA configuration). For dedicated-infrastructure customers, this is a physical-routing decision (a different model endpoint, a different provider account, a different region).

The routing logic is configuration, not code. New tenant onboarding does not require gateway code changes.

### 3.3 Rate limiting

Per-tenant, per-feature, per-user rate limits enforced at the gateway. Limits are configured; the gateway checks every call; over-limit calls are rejected with a structured rate-limited error.

The rate limits are layered with the cost circuit breakers (see [cost-and-finops/cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md)) — rate limits address request volume; circuit breakers address cost. Both are needed.

### 3.4 Cost accounting

Every call's cost is computed at the gateway (the gateway has the token counts, the model, the pricing table). Cost is emitted to the cost-aggregation pipeline; the gateway also enforces the per-interaction / per-session / per-tenant / per-feature cost circuits.

The four-layer cost stack is implemented at the gateway (see [cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md)). The gateway is the natural enforcement surface because it sees every call before it happens.

### 3.5 Input filtering

The gateway runs input filters on the assembled prompt before dispatching:

- **PHI / PII screening.** Detect sensitive data in the input; route to the redacted pipeline; emit telemetry.
- **Prompt-injection screening.** When the input includes content from outside-trust sources (a patient-supplied message quoted into a clinician's query), screen for injection patterns.
- **Content-policy screening.** Vendor moderation (Anthropic, OpenAI Moderation, Azure Content Safety, AWS Bedrock Guardrails) runs here.
- **Schema and structure validation.** The messages must conform to expected shape; tools must be from the registry; parameters must be within bounds.

Filter failures are rejected with structured errors. The consumer's prompt assembly is informed by what failed; the agent's prompt knows to handle the rejection class.

### 3.6 Output filtering

After the model responds, the gateway runs output filters before returning to the consumer:

- **PHI / PII re-screening.** Detect any PHI / PII the model produced (often hallucinated or surfaced from retrieval); redact or reject.
- **Schema validation.** When structured output is expected (JSON, tool calls), validate against the schema; route to repair-loop if invalid.
- **Content-policy re-screening.** Vendor moderation on outputs.
- **Citation validation.** When the output includes citations (Care Coordinator clinical answers), validate that citations point to actually-retrieved sources.

Output filter failures route to repair (a re-call with the failure context), to fallback (degraded response), or to rejection (the call returns a graceful failure).

### 3.7 Observability

Every call produces a trace span with the universal attributes (tenant, user, feature, session, interaction) and the LLM-call attributes (provider, model, version, tokens, cost, latency). The gateway is the natural place to populate these — it has all the information at the moment of dispatch.

The trace structure is detailed in [observability-and-telemetry/trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md). The gateway is the trace's primary span source for LLM calls.

### 3.8 Audit logging

For regulated workloads, the gateway emits audit-log entries: who called, what tenant, what model, what prompt version, what was the result classification, what cost was incurred. The audit log is separate from the routine trace; it has stricter access controls and longer retention.

For Meridian: every Care Coordinator gateway call writes to the clinical-audit sink with PHI-redacted content. The audit supports HIPAA attestation and clinical-decision-support reviews.

---

## 4. The internal API contract

The gateway's API is the consumer-facing surface. Its stability is what insulates consumers from provider churn.

### 4.1 The contract shape

```python
gateway.call(
    provider: str,           # "anthropic" | "openai" | etc.
    model: str,              # model name
    messages: list[Message], # already-assembled messages
    *,
    prompt_version: str,
    tools: list[Tool] | None = None,
    parameters: ModelParameters,
    context: CallContext,
    stream: bool = False,
) -> Response | StreamingResponse
```

The shape mirrors the LLM-call wrapper described in [llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md) — they are often the same component or directly composed.

### 4.2 What the contract guarantees

- The set of providers and models is enumerable (the gateway publishes a catalog).
- The message structure is uniform (the consumer's prompt assembly does not need provider-specific accommodations).
- The response is uniform (the consumer parses one response shape regardless of provider).
- Errors are classified into the same enum across providers.
- The trace attributes are populated identically across providers.

### 4.3 What the contract does NOT guarantee

- Specific model is always available. Models deprecate; the gateway raises a clear error when an unavailable model is requested.
- Specific cost. Prices change; the gateway computes from the current pricing table.
- Specific latency. Provider performance varies; the gateway exposes the actual latency but does not promise SLAs.
- Specific output content. The model is non-deterministic; the gateway returns what the model produced.

### 4.4 The contract evolution discipline

When a new feature is added to the gateway (a new attribute, a new filter, a new routing rule):
- It is additive by default.
- Existing consumers are not required to change.
- The feature is gated behind opt-in for the first sprint, then becomes default.

When something must change in a breaking way (rare):
- Major version bump.
- Deprecation lifecycle: old API and new API in parallel; old API marked deprecated; consumers migrate; old API removed.

This is the same contract discipline that good API design uses elsewhere. The gateway is an API.

---

## 5. Build-vs-buy decision

Two extremes are usually wrong; the productive answer is in between.

### 5.1 The pure-buy extreme

Vendor solutions exist for parts of the gateway pattern: Portkey, Helicone, LangSmith Proxy, AWS Bedrock with Guardrails, Azure OpenAI with Content Safety. Each provides a subset of the responsibilities — Portkey is strong on routing and rate limiting; Helicone on cost / observability; Bedrock Guardrails on content filtering.

Buying a vendor for the entire gateway pattern is rarely complete. The vendor's per-tenant routing model does not match the team's tenancy. The vendor's cost-attribution dimensions do not match the team's chargeback model. The vendor's audit log does not align with the team's compliance requirements.

### 5.2 The pure-build extreme

Building the entire gateway from scratch — including custom moderation models, custom audit infrastructure, custom routing layer — is expensive and rarely justified. The team ends up maintaining infrastructure that is not differentiated.

### 5.3 The productive middle: build the chokepoint, buy the components

The gateway itself (the chokepoint, the contract, the routing logic, the platform-specific integrations) is built. The components plugged into the gateway (moderation, observability, cost tracking) are bought where vendor offerings fit.

For Meridian:
- The gateway is custom (Python service, deployed on AWS).
- Content moderation uses Anthropic's built-in content policies + AWS Bedrock Guardrails for the supplementary PHI screening.
- Observability uses OpenTelemetry exporting to Datadog (general APM) and LangSmith (AI-specific debugging).
- Cost tracking is custom (pricing table, Redis-backed circuit breakers, custom dashboards).
- Routing is custom (per-tenant configuration, BAA-aware model selection).

This pattern means the team owns the parts that are differentiated (the integration with their tenancy, compliance, audit) and consumes vendor offerings for the parts that are commodity (the moderation API, the trace backend).

### 5.4 The build-vs-buy decision framework

For each gateway responsibility, ask:

| Responsibility | Buy when | Build when |
|---|---|---|
| Authentication | Existing identity infrastructure (Okta, Azure AD) → consume | Custom identity model |
| Per-tenant routing | N/A — always build (the routing logic is your tenancy model) | Always |
| Rate limiting | API gateway product (Kong, Apigee) for HTTP-shaped rate limiting | Custom for per-tenant per-feature cost-based limits |
| Cost accounting | Vendor (Helicone, Portkey) for general cost dashboards | Custom for per-tenant chargeback aligned with your business model |
| Input/output filtering | Vendor moderation (Anthropic, OpenAI, Bedrock Guardrails, Azure Content Safety) | Custom classifiers for domain-specific patterns (PHI for healthcare) |
| Observability | OpenTelemetry + your existing APM | Custom collectors for AI-specific attributes |
| Audit logging | Cloud-native audit (AWS CloudTrail, Azure Activity Logs) for infrastructure events | Custom for AI-specific audit (model calls, prompt versions, content) |

### 5.5 The vendor-lock-in consideration

Any vendor in the gateway path becomes a critical dependency. The gateway's stability requires that vendor swaps are possible without consumer changes.

The mitigation pattern: the gateway's API to vendors is its own internal abstraction, not the vendor SDK directly. When the vendor swaps, the gateway's vendor adapter changes; the gateway's consumer-facing API does not.

For Meridian: the moderation step accepts a list of moderation backends (Anthropic native, AWS Bedrock Guardrails, OpenAI Moderation); each is a pluggable adapter; routing across them is configuration.

---

## 6. Integration with adjacent platform components

The gateway is one of three platform-level chokepoints (per ADR-CARE-007). The others are the tool registry and the retrieval-client wrapper. The three components interact:

### 6.1 Gateway + tool registry

When the gateway dispatches a call that includes tools, the tools come from the tool registry (the tools' definitions and authorization metadata). When the model's response includes tool calls, the consumer dispatches them through the tool registry (not directly to tool implementations).

The boundary: the gateway gives the model the *available* tool surface (which tools the consumer chose to expose); the tool registry handles *invocation* of those tools when the model selects them.

Cross-link: [tool-call-authorization.md](./tool-call-authorization.md).

### 6.2 Gateway + retrieval-client wrapper

When a tool call invokes retrieval (a `look_up_clinical_guideline` tool that does RAG), the retrieval flows through the retrieval-client wrapper (which enforces tenant scoping and emits its own spans). The gateway sees the LLM call; the wrapper sees the retrieval; both attach to the same trace.

Cross-link: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md).

### 6.3 Gateway + cost circuit breaker

The cost circuit breaker is implemented at the gateway. The gateway has all four pieces needed: the cost estimate (pre-call), the actual cost (post-call), the counter aggregations (Redis), and the rejection mechanism (returning a structured error).

Cross-link: [cost-and-finops/cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md).

### 6.4 Gateway + observability

The gateway emits the LLM-call span. The agent-loop runner emits the per-turn span (which wraps the gateway calls within the turn). The retrieval wrapper emits the retrieval span. The trace ties them together.

Cross-link: [trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md).

---

## 7. Deployment shape

The gateway can be deployed three ways. The right answer depends on the team's existing infrastructure pattern.

### 7.1 As a library (in-process)

The gateway is a library that application code imports. No separate service. Every application instance has its own gateway instance; circuit-breaker counters are shared via Redis.

- **Pros.** Simple deployment. No network hop. Latency overhead is negligible.
- **Cons.** Multi-language platforms must implement the library multiple times. Configuration sync requires coordination across application deployments.
- **When to use.** Single-language platform (Python-only, Node-only). Latency-sensitive workloads where the network hop matters.

### 7.2 As a sidecar

The gateway runs as a sidecar process alongside each application instance. Application code calls the local sidecar; the sidecar handles routing, enforcement, observability, then dispatches to providers.

- **Pros.** Language-agnostic (any application that can make HTTP calls can use the gateway). Configuration sync is the sidecar's job, not the application's.
- **Cons.** Operational complexity of the sidecar deployment. Network hop adds latency (typically sub-1ms but non-zero).
- **When to use.** Multi-language platforms. Teams comfortable with sidecar patterns (Istio / Envoy / Linkerd users).

### 7.3 As a centralized service

The gateway is a centralized service with its own deployment. Application code calls the gateway over the network (HTTP or gRPC).

- **Pros.** Single deployment to operate. Configuration is centralized. Multi-language out of the box.
- **Cons.** Centralized failure mode (gateway outage takes down all AI features). Network hop and serialization overhead are higher.
- **When to use.** Large platform with many AI feature teams who do not want to operate per-team gateways. Strong centralized platform team.

### 7.4 Meridian's choice

The Meridian Care Coordinator uses a Python library deployment (in-process). The platform is Python-primary; latency budget is tight (sub-1.5s TTFT); the team operates the gateway as part of the existing service deployment pipeline.

A centralized service was considered for the cross-feature platform layer; the trade-off (additional latency + operational complexity) was assessed as not justified at current scale. Revisit threshold: when 4+ feature teams are independently operating the gateway library, consolidate to a sidecar or service.

---

## 8. Worked Meridian Health example

### 8.1 The gateway component

`meridian.ai_gateway` is a Python package. The library exposes a `Gateway` class with the `call_llm` method described in section 4.1. Consumer code imports the package; no other path to LLM providers exists.

### 8.2 Configuration

The gateway is configured via a YAML file (deployed alongside the application):

```yaml
gateway:
  providers:
    anthropic:
      endpoint: https://api.anthropic.com
      auth: ${ANTHROPIC_API_KEY}
      baa_covered: true
    openai:
      endpoint: https://api.openai.com
      auth: ${OPENAI_API_KEY}
      baa_covered: true
    cohere:
      endpoint: https://api.cohere.ai
      auth: ${COHERE_API_KEY}
      baa_covered: true

  models:
    claude-opus-4-7:
      provider: anthropic
      version_pin: "2026-04-12"
      baa_covered: true
      max_context_tokens: 200000
    claude-sonnet-4-6:
      provider: anthropic
      version_pin: "2025-08-15"
      baa_covered: true
      max_context_tokens: 200000
    # ...

  pricing_table: "./pricing/2026-05.yaml"

  rate_limits:
    per_tenant_default:
      requests_per_minute: 100
      tokens_per_minute: 200000
    per_feature:
      care-coordinator:
        requests_per_minute_total: 500

  cost_circuit_breakers:
    per_interaction_usd: 0.50
    per_session_usd: 1.50
    per_tenant_per_day_usd_default: 50
    per_tenant_per_day_usd_premium: 200
    per_feature_per_day_usd:
      care-coordinator: 1500
      analytics-copilot: 400

  input_filters:
    - phi_detection_v2
    - prompt_injection_screener_v1
  output_filters:
    - phi_redaction_v2
    - citation_validation_v1

  observability:
    trace_exporter: otlp
    trace_endpoint: ${OTEL_ENDPOINT}

  audit:
    sink: clinical_audit_s3
    retention_years: 7
```

### 8.3 The call flow

A typical Care Coordinator supervisor call traverses the gateway:

1. **Authentication.** The caller (Care Coordinator service) authenticates via service identity; the platform-level JWT carries tenant and feature context.
2. **Routing.** The gateway resolves `claude-opus-4-7` to the Anthropic provider, the BAA-covered endpoint, the version pin.
3. **Rate limit check.** Per-tenant and per-feature rate limits checked; pass.
4. **Cost circuit-breaker check.** Pre-call cost estimate; all four budgets checked; pass.
5. **Input filtering.** PHI detector scans the input messages; flag set if PHI present (for audit, not rejection). Prompt-injection screener scans patient-supplied content; pass.
6. **Dispatch.** Anthropic API call.
7. **Response receipt.** Streaming response.
8. **Output filtering.** PHI re-screen on the response; citation validation when citations are produced.
9. **Cost accounting.** Actual cost computed; emitted to cost-aggregation pipeline; counters updated.
10. **Trace emission.** Span closed with all attributes (universal + LLM-specific + cost + latency).
11. **Audit emission.** Clinical-audit entry written.
12. **Response return.** The streaming response is wrapped and returned to the consumer.

The consumer (the agent loop runner, in this case) sees a uniform interface. The gateway has done all the enforcement, attribution, observability, and audit.

### 8.4 What the gateway does NOT do

- The agent loop is the agent-loop-runner's job, not the gateway's.
- The prompt assembly is the consumer's job (the supervisor's prompt-building code).
- The tool dispatch (when the model returns tool calls) is the tool registry's job.
- The retrieval (when a tool calls retrieval) is the retrieval-client wrapper's job.

Each chokepoint handles its slice; together they form the platform.

### 8.5 Vendor adapters

Pluggable adapters for each provider. Adding a new provider is a new adapter + a configuration entry, not a gateway rewrite. The adapter handles provider-specific quirks (message-format differences, streaming-event formats, error-classification differences) and presents the uniform internal interface.

### 8.6 The platform discipline

- `meridian.ai_gateway` is the only path to LLM providers. Lint rule rejects direct provider SDK imports outside the gateway package.
- Configuration changes go through PR review with ai-platform-eng + security-eng (for any provider / model / BAA changes).
- Quarterly review of routing rules, cost budgets, filter configurations.
- Annual security review of the gateway's threat model (input filtering is a critical security boundary).

---

## 9. Anti-patterns

### 9.1 "Gateway becomes a framework"

The gateway accumulates features: it does prompt assembly, it runs agent loops, it has its own prompt library, it includes orchestration. The thin gateway has become a fat platform.

**Corrective.** Maintain the thin discipline. Move accumulated features out — prompt assembly to the prompt layer, orchestration to the agent loop runner. The gateway's value is in being thin.

### 9.2 "Direct provider SDK imports in application code"

Application code imports the Anthropic SDK directly because "we just need to make this one call quickly." The gateway's coverage is partial; the platform's uniformity claim is false.

**Corrective.** Lint rule against direct SDK imports outside the gateway package. Code review enforces. New contributors learn the discipline at onboarding.

### 9.3 "Vendor SDK is the gateway"

The team configures the vendor's gateway product (Portkey, Helicone, LangSmith Proxy) as their gateway, with no abstraction in between. Vendor changes propagate; vendor outages take down all AI features; vendor swap requires application-wide refactoring.

**Corrective.** Build the thin chokepoint; consume the vendor as an adapter inside the chokepoint. The vendor swap is an adapter change, not an application change.

### 9.4 "Gateway is consumer-aware"

The gateway has special cases for specific consumers ("if feature=care-coordinator, skip the moderation filter because clinical use is different"). The exceptions accumulate; the gateway becomes consumer-specific code.

**Corrective.** The gateway's behavior is configuration-driven, not consumer-name-aware. If care-coordinator legitimately needs different filtering, model that in the configuration (a filter profile per feature), not as code branches.

### 9.5 "Gateway and circuit breakers in different code paths"

The cost circuit breaker is implemented separately from the gateway and runs in a different code path. Inconsistencies emerge; some calls hit the circuit, some bypass.

**Corrective.** The circuit breaker is implemented inside the gateway. Every call through the gateway is checked; every check is recorded; there is no other path.

### 9.6 "Synchronous-only gateway"

The gateway supports synchronous calls only; streaming requires a separate path. Streaming-specific instrumentation lives in the consumer's code; the gateway's observability claims do not cover streaming.

**Corrective.** Streaming is first-class in the gateway. TTFT, idle-gap, stream-error classification are gateway attributes.

### 9.7 "Gateway upgrades require consumer downtime"

A gateway upgrade is a breaking change for consumers. Releases coordinate; downtime windows are negotiated; consumers fear gateway changes.

**Corrective.** Backward-compatible contract evolution per section 4.4. Gateway upgrades are routine; deprecation lifecycle handles breaking changes.

### 9.8 "No emergency override path"

In a vendor outage, the team has no way to route around the failing provider quickly. Recovery requires a code change.

**Corrective.** Routing configuration supports runtime updates (per-provider kill switch, per-tenant fallback routing). The override is configuration, not code.

---

## 10. Findings (sprint-assignable)

### ARCH-GATEWAY-001 — Severity: Critical
**Finding.** Application code imports LLM provider SDKs directly; gateway coverage is partial.
**Recommendation.** Lint rule against direct SDK imports; migrate all LLM calls to the gateway; refuse code that bypasses.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-GATEWAY-002 — Severity: Critical
**Finding.** Cost circuit breakers are separate from the gateway; some calls bypass cost enforcement.
**Recommendation.** Circuit breakers implemented inside the gateway; every call goes through; no separate paths.
**Owner.** ai-platform-eng + finops, sprint N+1.

### ARCH-GATEWAY-003 — Severity: Critical
**Finding.** Per-tenant routing is implemented in application code; tenant misconfiguration can route to wrong provider / BAA boundary.
**Recommendation.** Routing logic in the gateway; configuration-driven; tenant onboarding does not require code changes.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-GATEWAY-004 — Severity: High
**Finding.** Input / output filtering (PHI, prompt-injection, moderation) is per-consumer; consistency is partial.
**Recommendation.** Filtering at the gateway, profile-per-feature configuration; uniform application across all calls.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-GATEWAY-005 — Severity: High
**Finding.** Model versions are referenced by alias in gateway configuration; drift is invisible.
**Recommendation.** Pin to full versions per [llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md) OBS-LLM-004; gateway refuses unpinned references.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-GATEWAY-006 — Severity: High
**Finding.** Audit log writes happen post-gateway in application code; some calls produce no audit entry.
**Recommendation.** Audit emission at the gateway; uniform for every call; configurable sink per feature.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+2.

### ARCH-GATEWAY-007 — Severity: High
**Finding.** Streaming calls bypass some gateway responsibilities (cost accounting, output filtering); streaming-specific instrumentation is inconsistent.
**Recommendation.** Streaming is first-class; all gateway responsibilities apply; instrumentation covers TTFT, idle-gap, stream errors.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-GATEWAY-008 — Severity: High
**Finding.** Provider SDKs are imported by the gateway and exposed indirectly to consumers (e.g., response objects are vendor-specific); consumers depend on vendor types.
**Recommendation.** Gateway exposes uniform response types; provider-specific types are internal to vendor adapters.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-GATEWAY-009 — Severity: High
**Finding.** New models can be invoked through the gateway without prior registration (missing from pricing table, missing from BAA configuration).
**Recommendation.** Gateway refuses unregistered models; the registration process is the deliberate enablement decision.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GATEWAY-010 — Severity: High
**Finding.** Gateway has no kill-switch / emergency-override mechanism; provider outages require code deploy to mitigate.
**Recommendation.** Per-provider kill switch as runtime configuration; per-tenant fallback routing for graceful degradation.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-GATEWAY-011 — Severity: Medium
**Finding.** Gateway upgrades are breaking; consumer migrations are coordinated as outages.
**Recommendation.** Backward-compatible contract evolution; additive features by default; deprecation lifecycle for breaking changes.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GATEWAY-012 — Severity: Medium
**Finding.** Vendor adapters are tightly coupled to vendor SDKs; vendor swap is a deep refactor.
**Recommendation.** Vendor adapters expose the gateway's internal contract; SDK details internal to each adapter; vendor swap is an adapter change.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GATEWAY-013 — Severity: Medium
**Finding.** Gateway runs as a centralized service with no fallback path; gateway outage halts all AI features.
**Recommendation.** Either move to in-process library, or add fallback path (cached vendor SDK with degraded enforcement) for gateway-outage scenarios.
**Owner.** ai-platform-eng + sre, sprint N+4.

### ARCH-GATEWAY-014 — Severity: Medium
**Finding.** Configuration is code-deployed; runtime tuning of rate limits / circuit thresholds requires a release.
**Recommendation.** Move tunable configuration to runtime-updated store; deploy code only for new logic.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GATEWAY-015 — Severity: Medium
**Finding.** Provider failover (Anthropic → OpenAI for the same workload) is undocumented and untested.
**Recommendation.** Document the failover routing; eval-test the failover path; calibrate quality expectations on the fallback model.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-GATEWAY-016 — Severity: Medium
**Finding.** Gateway accumulates feature-specific logic over time; the thin discipline is eroding.
**Recommendation.** Quarterly review of gateway code; refactor accumulated features back to consumers (prompt assembly, orchestration) where they belong.
**Owner.** ai-platform-eng team lead, sprint N+4.

### ARCH-GATEWAY-017 — Severity: Low
**Finding.** Gateway observability does not include per-feature, per-tenant request volume dashboards.
**Recommendation.** Add per-feature × per-tenant volume dashboards from the gateway's emitted spans.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ARCH-GATEWAY-018 — Severity: Low
**Finding.** Gateway documentation is internal-wiki-only; new contributors discover behavior by reading source code.
**Recommendation.** Generate documentation from the gateway's configuration schema and adapter contract; commit alongside code.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team without an AI gateway:

- [ ] **Sprint 0 — decide.** Deployment shape (library vs sidecar vs service). Build-vs-buy per component. Document.
- [ ] **Sprint 1 — chokepoint.** Build the thin gateway (or stand up the vendor solution + thin wrapper). Single API method. Universal attributes captured.
- [ ] **Sprint 1 — primary path.** Migrate the main production path to the gateway. Verify uniformity.
- [ ] **Sprint 2 — enforcement.** Wire authentication, per-tenant routing, rate limiting, cost accounting through the gateway.
- [ ] **Sprint 2 — observability.** Trace emission from the gateway; per-call attributes per [llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md).
- [ ] **Sprint 3 — filtering.** Input and output filtering profiles; integrate vendor moderation; build custom filters where needed.
- [ ] **Sprint 3 — circuit breakers.** Four-layer cost circuit breakers per [cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md).
- [ ] **Sprint 4 — audit.** Audit-log emission to compliance sink; retention aligned with regulatory requirement.
- [ ] **Sprint 4 — enforcement.** Lint rule against direct SDK imports; code review enforces; team trained.
- [ ] **Sprint 5 — secondary paths.** Migrate eval runners, batch jobs, background tasks, MCP tool invocations.
- [ ] **Sprint 6 — kill-switches and failover.** Runtime configuration for emergency routing; per-provider kill switches; failover paths tested.
- [ ] **Ongoing — discipline.** Quarterly review against the thin-gateway principle. Refactor accumulated features.

A team that completes this sequence has the chokepoint pattern that makes guardrails, cost control, observability, and audit uniform across the platform. A team that defers this accumulates inconsistent patterns and pays the audit / compliance / operational tax forever.

---

## 12. References

- API gateway pattern from microservices literature — the conceptual parent of this pattern adapted to AI.
- AWS API Gateway, Kong, Apigee documentation — general gateway products that AI gateways often coexist with (the API gateway handles HTTP-shaped concerns; the AI gateway handles AI-specific concerns).
- Portkey, Helicone, LangSmith Proxy, AWS Bedrock with Guardrails, Azure OpenAI with Content Safety — vendor offerings for parts of the gateway pattern; build-vs-buy candidates.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — ADR-CARE-007 commits to this pattern.
- This repo: [guardrails-and-policy-architecture/tool-call-authorization.md](./tool-call-authorization.md) — the parallel chokepoint pattern on the tool side.
- This repo: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — the parallel chokepoint pattern on the retrieval side.
- This repo: [guardrails-and-policy-architecture/content-moderation-architecture.md](./) (coming) — the depth on input / output filtering choices.
- Sibling repo: [ai-engineering-reference-architecture/observability-and-telemetry/llm-call-instrumentation.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/llm-call-instrumentation.md) — the per-call wrapper that often sits inside the gateway.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md) — the circuit-breaker pattern implemented at the gateway.
- Sibling repo: [ai-security-reference-architecture/llm-application-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) — the threat-model context for the input / output filtering responsibilities.
