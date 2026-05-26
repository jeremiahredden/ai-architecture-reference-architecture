# Guardrail Placement Decision Framework

> **Audience.** Architects deciding where each guardrail lives in an AI system. Tech leads whose AI feature has the right controls in the wrong places — caught by a security review, a compliance audit, or an incident. Anyone whose answer to "where does the PII filter live?" is "I think in the chat handler... or maybe the gateway... let me check." **Scope.** The *architectural* map of guardrail placement — input, output, intermediate-step, retrieval — and the layered defense pattern that combines them. Not the per-control engineering depth (see [ai-gateway-pattern.md](./ai-gateway-pattern.md), [tool-call-authorization.md](./tool-call-authorization.md), [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md)). Not the threat model and detection content (sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Most AI guardrail failures in production aren't because the control was wrong; they're because the control was in the wrong place. The PII filter wraps the chat handler — and is silently bypassed by the new batch-classification handler that ships three months later. The tool-call authorization is implemented in the care-coordinator's tool-dispatch code — and is missing in the patient-summary service's similar dispatch code. The retrieval-scope filter is in the prompt assembly — and is forgotten in the new "summarise documents" feature.

The pattern: controls placed in calling code drift. New code paths are added without re-implementing the controls; existing code paths are refactored without preserving the controls; nobody notices until the audit, the incident, or both.

The architectural solution is to place each guardrail at the smallest set of choke points that all AI invocations must go through. The AI gateway is one such choke point for inputs and outputs. The tool registry is one for tool calls. The retrieval client is one for retrieval scope. With controls at these layers, application code can't bypass them; new features inherit the controls by virtue of using the platform infrastructure.

This document is the architectural map: which guardrails belong at which choke point, why, and how the layered defense pattern combines them. It's the design conversation that prevents the misplaced-control failure.

This document is opinionated about four things:

1. **Guardrail placement is an architectural decision, not a per-feature implementation choice.** The placement of each control is a one-time design decision; all features inherit it through the platform layer.
2. **The platform layer is the right home for most guardrails.** Application-level controls drift; platform-level controls don't.
3. **Layered defense is the default; single-layer is fragile.** Most production guardrails fail eventually; redundancy at multiple layers prevents single-failure-cascades.
4. **Each guardrail's placement has a "fail-loud" requirement.** A guardrail that fails silently (the filter is bypassed without anyone noticing) is worse than no guardrail; the placement must surface failures.

Structure: (2) the four guardrail categories; (3) the architectural choke points; (4) the placement decision framework; (5) the layered defense pattern; (6) fail-loud requirements per category; (7) the relationship to the sibling security and engineering content; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four guardrail categories

A taxonomy that organises the rest of this document.

### 2.1 Input guardrails

Controls applied to the input before it reaches the model.

- **PII / PHI detection.** Identify sensitive data in user input; redact, refuse, or route differently.
- **Prompt injection screening.** Detect known injection patterns; reject or flag.
- **Abuse / off-topic detection.** Detect requests for prohibited content or out-of-scope topics.
- **Input length / shape validation.** Reject inputs exceeding limits or malformed.
- **Authorization checks.** Confirm the user / tenant is authorized for this feature.
- **Rate limiting.** Per-user / per-tenant request rate caps.

### 2.2 Output guardrails

Controls applied to the model's output before it reaches the user / downstream consumer.

- **PII / PHI leak prevention.** Scan output for sensitive data not appropriate to expose.
- **Format validation.** Output matches schema / structural expectations.
- **Claim verification.** Output's factual claims are checked (against retrieval, against known facts).
- **Harmful content detection.** Output doesn't contain prohibited content.
- **Hallucination detection.** Output's claims are grounded in provided context.
- **Citation / source requirements.** Output includes required citations.

### 2.3 Intermediate-step guardrails

Controls applied during the agent's processing.

- **Tool-call authorization.** Each tool call is authorized.
- **Agent action approval.** High-stakes actions require explicit approval.
- **Budget enforcement.** Per-request cost / time / turn budgets.
- **Tool result validation.** Tool outputs are validated before agent consumes.
- **Loop / iteration limits.** Bounded agent loops.
- **Decision logging.** Every meaningful decision is logged.

### 2.4 Retrieval guardrails

Controls applied to retrieval (the data the model sees as context).

- **Scope enforcement.** Retrieval respects tenant / user / role scope.
- **Freshness checks.** Retrieved content meets freshness requirements.
- **Source allow-listing.** Only authorized sources are queried.
- **Result filtering.** Sensitive or stale results filtered post-retrieval.
- **Provenance tracking.** Retrieved content carries lineage.
- **Volume caps.** Bounded number of retrieved chunks per query.

### 2.5 Why this categorisation matters

Each category has natural placement: input at the input boundary, output at the output boundary, intermediate-step within the agent loop, retrieval within the retrieval client. The categorisation maps directly to architectural choke points.

The categorisation also clarifies the layered defense: the same threat (e.g., data leakage) is addressed by multiple categories (input filter that blocks injection; retrieval filter that prevents over-scoped access; output filter that catches what leaks past).

### 2.6 The four-pillar framing

The four categories are the four pillars of an AI system's guardrail surface:

```
        Input → Retrieval → Agent → Output
          ↓        ↓          ↓        ↓
       Input    Retrieval  Inter-     Output
       guards   guards     mediate    guards
                           guards
```

Every AI feature has the four pillars; every guardrail decision is "which pillar(s)."

---

## 3. The architectural choke points

The system-wide locations where guardrails are placed.

### 3.1 The AI gateway

A platform-level service (or library) that all model invocations go through. Per [ai-gateway-pattern.md](./ai-gateway-pattern.md).

**Responsibilities relevant to guardrails:**

- Input filtering (PII, injection, abuse, length).
- Output filtering (PII leak, format validation, content moderation).
- Authorization enforcement (tenant / user / feature authorization).
- Rate limiting and budget enforcement.
- Cost accounting (per [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) in the engineering sibling).
- Logging and observability for all invocations.

**Why this is the right home for input and output guardrails:**

- Every invocation goes through it; no bypass possible.
- Centralised; updates apply everywhere.
- Observable; the gateway sees all traffic.
- Manageable: one team owns the gateway; can enforce policy uniformly.

### 3.2 The tool registry

A platform-level service that all tool calls go through. Per [tool-call-authorization.md](./tool-call-authorization.md) and the engineering sibling's [tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md).

**Responsibilities relevant to guardrails:**

- Tool-call authorization (caller, target, arguments).
- Tool argument validation (schema).
- Tool result validation (schema, content).
- Rate limiting per tool.
- Per-tool budget enforcement.
- Logging.

**Why this is the right home for intermediate-step guardrails:**

- All tool calls go through; agent can't bypass.
- Per-tool policy is centralised.
- Observable; complete tool-call trace.

### 3.3 The retrieval client

A platform-level abstraction over the vector store (and other retrieval sources). Per [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md).

**Responsibilities relevant to guardrails:**

- Tenant / scope enforcement.
- Source allow-listing.
- Result filtering.
- Freshness validation.
- Query rewriting (to inject scope filters).
- Audit trail.

**Why this is the right home for retrieval guardrails:**

- All retrieval goes through; prompt assembly can't query directly.
- Scope is enforced at the data boundary, not the prompt boundary.
- Observable; per-tenant access audited.

### 3.4 The agent runner

For agent-shaped features, the runner is itself a choke point. Per the engineering sibling's [agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md).

**Responsibilities relevant to guardrails:**

- Loop / iteration limits.
- Cost / time budget enforcement.
- Decision logging.
- State checkpointing.
- Escalation gateways.

The runner is platform code; the agent's behaviour is bounded by it.

### 3.5 The four-choke-point map

| Guardrail category | Primary choke point | Secondary choke point |
| --- | --- | --- |
| Input guardrails | AI gateway | (Agent prompt assembly, for agent-specific) |
| Output guardrails | AI gateway | (Application formatter, for context-specific format) |
| Intermediate-step guardrails | Tool registry + Agent runner | — |
| Retrieval guardrails | Retrieval client | (Per-query filters at query-construction time) |

Most guardrails have one primary location and possibly a secondary defense.

### 3.6 The "we don't have a platform" excuse

Some teams say "we don't have a platform team to build these choke points." The corrective: you don't need a "platform team" — you need *some* code that all features go through. Even a single shared library (`from team.ai_client import call_model`) creates the choke point if all features import it.

The choke point is architectural, not organisational. Start with a library; mature to a service if scale demands.

### 3.7 The "controls in calling code" anti-pattern

Some teams place controls in calling code:

- The chat handler does PII filtering before calling the model.
- The batch handler does (or doesn't) PII filtering before calling the model.
- The analytics handler does (or doesn't) PII filtering...

Each handler's controls drift independently. The fix is moving the controls to the gateway; calling code no longer carries the responsibility.

---

## 4. The placement decision framework

For each guardrail, ask:

### 4.1 Question 1 — Which category?

Input / output / intermediate / retrieval. Often obvious from the guardrail's purpose; sometimes a control fits multiple categories (e.g., PII can be input-filtered, retrieval-filtered, and output-filtered — typically all three for sensitive workloads).

### 4.2 Question 2 — At what choke point?

For each category, the primary choke point per section 3.5. Place there unless there's a specific reason not to.

### 4.3 Question 3 — Layered defense?

For high-stakes controls, layer:

- Input filter at the gateway (first defense).
- Retrieval filter at the retrieval client (second defense, for data-leak prevention).
- Output filter at the gateway (third defense, for what leaks past).

For low-stakes controls, single-layer is fine. The team decides per-control based on consequence-of-failure.

### 4.4 Question 4 — Per-feature override?

Most guardrails are platform-wide. Some need per-feature configuration:

- A medical workflow may have stricter PII rules than a non-medical one.
- An internal tool may need different authorization than an external API.

The choke point supports per-feature configuration (passed in by the caller); the calling feature can't disable controls, but can request stricter ones.

### 4.5 Question 5 — Per-tenant override?

Some controls vary per tenant:

- A regulated tenant may need stricter content moderation.
- A premium tenant may have higher rate limits.

Per-tenant configuration is the gateway's responsibility; passed in via the tenant context.

### 4.6 Question 6 — What's the failure mode?

The placement must surface failures (per section 6):

- If the guardrail itself fails (the moderation API is down), the system shouldn't silently proceed.
- If the guardrail rejects, the rejection must be observable and the user-facing experience must be defined.

### 4.7 Question 7 — Cost and latency?

Each guardrail adds cost and latency:

- An LLM-based moderator adds 200-500ms and a per-call cost.
- A structured-output validator adds milliseconds.
- A retrieval-scope filter is essentially free (it's a SQL clause).

Calibrate per-feature: latency-critical features may tolerate fewer or faster guardrails; cost-sensitive features may use cheaper alternatives.

### 4.8 The completion check

For each AI feature, the team should be able to enumerate:

- Which input guards apply, where they're placed, how they fail.
- Same for output guards.
- Same for intermediate-step (if agent).
- Same for retrieval guards.

If any category lacks a documented answer, the feature has a guardrail gap.

---

## 5. The layered defense pattern

When single-layer isn't sufficient.

### 5.1 The threat-multiple-defense principle

For each threat, the team identifies multiple defenses across categories. Example: data leakage prevention.

| Defense | Where | What it catches |
| --- | --- | --- |
| Input scope check | Gateway | Inappropriate access requests |
| Retrieval scope filter | Retrieval client | Over-scoped queries reaching the data layer |
| Output PII scan | Gateway | Sensitive data in the output |
| Cross-reference with retrieval scope | Output validator | Output references data not in scope |

Each layer catches some failures of the previous. Defence-in-depth.

### 5.2 The trade-off

Layered defense costs more (each layer is processing and latency); team must accept the overhead. For high-stakes controls, the overhead is worth it.

### 5.3 The "we already have it at the gateway" objection

A common pushback: "the gateway catches it; we don't need a second layer."

The counter: gateway-only is single-point-of-failure. A bug in the gateway, a misconfiguration, a new code path that bypasses the gateway — any of these breaks the single defense. Layered defense survives single-point failures.

For high-stakes controls (PHI in healthcare, financial data, security-sensitive content), pay the layered defense overhead.

### 5.4 The "layered defense is over-engineering" objection

For low-stakes controls, this is correct. The framework's question 3 (per section 4.3) is the deliberate choice: layered for high-stakes, single-layer for low-stakes.

### 5.5 Layered defense doesn't replace fixing the root cause

If a layer fails, the team must fix it. Layered defense buys time; it doesn't make the failure acceptable.

When the second layer catches what the first should have caught, the incident review investigates why the first failed and fixes it.

### 5.6 The relationship to threat modelling

Per the sibling security repo, threat modelling identifies the threats. The layered defense maps each threat to multiple categories.

Without threat modelling, the team can't tell which threats need layered defense. The security repo provides the threat catalog; this document provides the architectural placement.

### 5.7 The "third line" — detection and response

Beyond preventive controls (the categories above), the system needs detective controls (log analysis, anomaly detection, alerting) and response controls (incident response, rollback). Per the sibling security repo.

The architecture this document covers is the preventive layer. Detection and response live alongside but are different surfaces.

---

## 6. Fail-loud requirements per category

A guardrail that fails silently is worse than no guardrail. The placement must guarantee surface visibility on failure.

### 6.1 What "fail loud" means

When a guardrail is bypassed, disabled, or itself fails:

- The fact is logged with appropriate severity.
- An alert fires for security-critical guardrails.
- The user-facing behaviour is defined (typically refuse the request rather than proceed without protection).
- The incident is investigable from the logs.

### 6.2 Per-category fail-loud requirements

**Input guardrails.**

- If the PII filter fails to run (the service is down), the gateway refuses the request rather than proceeding without filtering.
- If the input filter rejects, the rejection is logged with reason and is observable in metrics.
- New code paths that bypass the gateway are detected (per section 6.4).

**Output guardrails.**

- If the output filter fails to run, the gateway returns an error rather than passing the unfiltered output.
- If the output filter modifies the output (e.g., redacts), the modification is logged.
- Output validation failures (schema mismatch) are logged for engineering investigation.

**Intermediate-step guardrails.**

- Tool-call authorization failures are logged with full context (caller, tool, arguments).
- Budget breaches are logged and alerted per [agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md) in the engineering sibling.
- Agent loop limit breaches are logged.

**Retrieval guardrails.**

- Scope filter failures (a query reaches the data layer without scope) are logged with high severity.
- Filter modification (the query was rewritten to add scope) is logged.
- Cross-scope attempts (the application requested data outside its scope) are logged and alerted.

### 6.3 The "guardrail bypass" detection

Specific to layered architectures: detection that some code path bypassed the gateway.

Approaches:

- **Audit log compare.** The gateway logs every invocation; the LLM provider's logs show invocations; comparison surfaces missing-gateway invocations.
- **Network policy.** Only the gateway has network access to the LLM provider; bypass is blocked at the network layer.
- **IAM policy.** Only the gateway's identity can call the LLM provider's API; bypass is blocked at the IAM layer.

The network or IAM approach is preferred; comparison is the fallback.

### 6.4 The "new code path" detection

When a new feature ships, the team needs assurance that it uses the gateway:

- CI checks for direct LLM provider SDK imports outside the gateway library.
- PR review templates ask "does this use the AI gateway?"
- Periodic audit of all LLM-calling code paths.

### 6.5 The "guardrail itself failed" handling

When the guardrail service is unavailable:

- **Fail closed (default for security-critical).** Reject the request; surface a clear error.
- **Fail open (acceptable for non-critical).** Proceed without the guardrail; log loudly.

Default to fail closed; require justification for fail open.

### 6.6 The "guardrail produced wrong result" detection

The guardrail's own quality matters. If a PII filter has 80% recall, 20% of PII leaks. Detection:

- Periodic eval of the guardrail's accuracy.
- Sample-based human review of guardrail decisions.
- Trends in false-positive / false-negative rates.

The guardrail itself is a system that needs eval, not "set and forget."

---

## 7. The relationship to sibling security and engineering content

This document is one of three perspectives on guardrails.

### 7.1 The security sibling

[ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) addresses:

- The threat model (what can go wrong).
- The detection content (how to identify attacks in progress).
- The response runbooks (what to do when something happens).

The security repo says *what threats exist*. This document says *where the architectural defenses go*.

### 7.2 The engineering sibling

[ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture) addresses:

- The engineering practice of building the gateway, tool registry, retrieval client, agent runner.
- The observability of these systems.
- The eval discipline for guardrails.
- The operational on-call patterns.

The engineering repo says *how to build and operate the choke points*. This document says *what controls go in them*.

### 7.3 This document's scope

The architectural decisions:

- The choke point pattern.
- Per-category placement.
- Layered defense.
- Fail-loud requirements.

Combined, the three perspectives cover the practice end to end.

### 7.4 The integration discipline

The three perspectives are designed to integrate:

- This document's placement decisions reference the engineering sibling's implementations.
- The engineering sibling's implementations are built to support the security sibling's detections.
- The security sibling's detections inform this document's placement decisions (where the threats are determines where defenses go).

A team using all three has comprehensive coverage; a team using only one has gaps.

### 7.5 The reading order

For a team starting from zero:

1. Security repo's threat model (what to defend against).
2. This document's placement framework (where to place defenses).
3. Engineering repo's implementation patterns (how to build).

For a team retrofitting:

1. Audit existing controls against this document's framework.
2. Identify gaps.
3. Reference engineering sibling for implementation.
4. Reference security sibling for detection and response.

---

## 8. Worked Meridian example

Meridian's guardrail placement.

### 8.1 The architectural map

```
                    User input
                       ↓
              [AI Gateway (meridian-llm-gateway)]
              - Input: PII detection, injection screening, length, auth
              - Cost / budget / rate limit
                       ↓
              [Care-coordinator agent runner]
              - Loop limits, budgets, escalation gateway
                       ↓
              [Tool registry (meridian-tools-registry)]
              - Per-tool authorization, schema, idempotency
                       ↓
              [Retrieval client (meridian-retrieval)]
              - Tenant scope, source allow-list, freshness
                       ↓
              [LLM call returns]
                       ↓
              [AI Gateway: Output filter]
              - PII leak scan, format validation, citation check
                       ↓
                   User receives
```

Every invocation goes through this path. Application code cannot bypass.

### 8.2 Per-category placement

**Input guardrails (at the gateway):**

- PII / PHI detection: regex + Presidio + LLM classifier; layered.
- Prompt injection screening: pattern-based filter + content classifier.
- Authorization: tenant + user + feature.
- Length: per-feature limits.
- Rate limit: per-tenant per-minute, per-day caps.

**Output guardrails (at the gateway):**

- PII / PHI leak prevention: same pipeline as input.
- Format validation: schema check.
- Hallucination detection: claim grounding check (LLM judge against retrieval context); layered with citation requirements.
- Content moderation: Anthropic Constitutional + medical-safety prompt validation.

**Intermediate-step guardrails:**

- Tool registry: per-tool authorization (per [tool-call-authorization.md](./tool-call-authorization.md)).
- Agent runner: loop limits (12 turns), cost budget ($0.50), time budget (45s).
- Approval gates for high-stakes actions (per the engineering sibling's tool architecture).

**Retrieval guardrails (at the retrieval client):**

- Tenant scope: enforced at query level (every query has WHERE tenant_id = X).
- Source allow-list: per-agent per-tenant.
- Freshness: per-source SLO check (per [freshness-architecture.md](../data-architecture-for-ai/freshness-architecture.md)).
- Result filtering: stale results downgraded; sensitive metadata filtered.

### 8.3 Layered defense examples

**PHI leak prevention:**

- Layer 1: input PII filter (catches user-provided PII not meant to be in the prompt).
- Layer 2: retrieval scope (catches over-scoped queries that would return other tenants' PHI).
- Layer 3: output PII scan (catches PHI that leaked from retrieved content or hallucinated).
- Layer 4: citation requirement (PHI in output must be backed by an authorized source).

Four layers. Each caught failures in incidents over the last 18 months.

**Tool authorization:**

- Layer 1: gateway-level feature authorization (user can use the care-coordinator at all).
- Layer 2: tool-registry per-tool authorization (user can call this specific tool).
- Layer 3: per-call argument validation (the arguments are within authorized scope).
- Layer 4: idempotency keys (no duplicate side effects).

### 8.4 Fail-loud examples

**Gateway bypass attempt:** The team configured AWS IAM such that only the gateway's service role can call Anthropic / OpenAI APIs from the production VPC. An attempted direct call from any other service is blocked at the network layer; logged.

In 18 months, two such attempts were detected (both from a debug script run by an engineer; both quickly resolved).

**PII filter failure:** When the Presidio service was unavailable for 3 hours in Q2-25, the gateway failed closed (returned errors); the care-coordinator was unavailable for that time; on-call alerted.

The fail-closed posture was the right choice; serving without PII filtering would have been worse than the downtime.

### 8.5 The "new code path" detection

CI rule: any code path calling Anthropic / OpenAI / Voyage / etc. SDKs directly (not via `meridian-llm-gateway`) fails the build.

In 14 months, the rule has fired ~12 times (mostly engineers prototyping; one production attempt that was caught at PR review).

### 8.6 The eval surface for guardrails

Per-guardrail eval:

- PII filter recall / precision: measured monthly on a labeled set; current 96% recall / 91% precision.
- Injection screening: measured against known injection patterns; current 89% recall.
- Hallucination detection: LLM-judge on a sample; current 87% recall.

Quality is monitored; degradations are investigated.

### 8.7 What worked

- **Choke-point architecture.** No bypass possible; controls are platform-wide.
- **Fail-closed default.** Outages were inconvenient; would have been incidents if served unfiltered.
- **Layered defense for PHI.** Caught multiple subtle failures; no major PHI incident in 18 months.
- **Per-guardrail eval.** Quality is measured, not assumed.

### 8.8 What didn't work initially

- **Application-level controls.** Initial deployment had PII filtering in the care-coordinator handler only; the patient-summary service shipped without it. Detected in Q1-25 audit; moved to the gateway.
- **Fail-open default.** Initial Presidio integration failed open if the service was slow; a bug caused 4 hours of unfiltered traffic. Switched to fail-closed after.
- **No bypass detection.** Pre-IAM-policy, a debug script bypassed the gateway without anyone noticing for a week. IAM policy added.

---

## 9. Anti-patterns

### 9.1 "Guardrails in calling code"

PII filter, authorization, etc. live in each calling feature's code. Drift between features; new features ship without controls.

**Corrective.** Move to the gateway per section 3.1 / 3.7.

### 9.2 "Single-layer defense for high-stakes"

PII protection only at the input filter; if the filter has gaps, leakage occurs.

**Corrective.** Layered defense per section 5; multiple categories cover the same threat.

### 9.3 "Fail-open default"

Guardrail unavailable → proceed without protection. Silent risk.

**Corrective.** Fail-closed default per section 6.5; fail-open requires justification.

### 9.4 "No bypass detection"

The team assumes the gateway is used; no detection if it isn't.

**Corrective.** Network / IAM policy + audit log compare per section 6.3.

### 9.5 "Guardrails not eval'd"

The team assumes guardrails are working; never measures.

**Corrective.** Per-guardrail eval per section 6.6.

### 9.6 "New code path skipped"

A new AI feature ships without the architectural review for guardrail placement.

**Corrective.** Design review per section 4.8; checklist per category.

### 9.7 "Output guardrails missing"

Input controls are placed; output controls are forgotten. The output can leak even though the input was clean.

**Corrective.** Per-category coverage per section 4.

### 9.8 "Guardrail decisions not observable"

Filter modifications, rejections, authorization denials happen but aren't logged. Investigation is impossible.

**Corrective.** Fail-loud requirements per section 6; logging mandatory.

---

## 10. Findings (sprint-assignable)

### ARCH-GR-001 — Severity: Critical
**Finding.** Guardrails placed in calling code; new features have ad-hoc coverage.
**Recommendation.** Move to platform choke points per section 3; deprecate in-feature controls.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-GR-002 — Severity: Critical
**Finding.** No bypass detection; direct LLM API calls possible from any code path.
**Recommendation.** Network / IAM policy per section 6.3; CI rule per section 6.4.
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-GR-003 — Severity: Critical
**Finding.** Fail-open default for security-critical guardrails; silent risk during outages.
**Recommendation.** Fail-closed default per section 6.5; justify any fail-open.
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-GR-004 — Severity: Critical
**Finding.** High-stakes data (PHI, financial) protected by single-layer guardrails only.
**Recommendation.** Layered defense per section 5 / 8.3; multiple categories.
**Owner.** architecture + ai-platform-eng, sprint N+1.

### ARCH-GR-005 — Severity: High
**Finding.** Per-category coverage gaps; some categories missing for some features.
**Recommendation.** Per-feature guardrail audit per section 4.8.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-GR-006 — Severity: High
**Finding.** Guardrail decisions not observable; investigation difficult.
**Recommendation.** Fail-loud requirements per section 6.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-GR-007 — Severity: High
**Finding.** Guardrail quality not measured; recall / precision unknown.
**Recommendation.** Per-guardrail eval per section 6.6.
**Owner.** ai-platform-eng + security, sprint N+2.

### ARCH-GR-008 — Severity: High
**Finding.** New AI features ship without guardrail-placement review.
**Recommendation.** Design review per section 4.8; checklist.
**Owner.** architecture + tech leads, sprint N+2.

### ARCH-GR-009 — Severity: Medium
**Finding.** Retrieval scope enforcement at prompt-assembly layer (drifts) rather than retrieval-client layer.
**Recommendation.** Retrieval client per section 3.3.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GR-010 — Severity: Medium
**Finding.** Tool authorization scattered across tools; per-tool implementations diverge.
**Recommendation.** Tool registry per section 3.2.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GR-011 — Severity: Medium
**Finding.** Per-tenant guardrail configuration not supported; some tenants need stricter controls.
**Recommendation.** Per-tenant overrides per section 4.5.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-GR-012 — Severity: Medium
**Finding.** Layered defense applied uniformly; cost overhead for low-stakes features.
**Recommendation.** Per-control decision per section 5.4; layered for high-stakes only.
**Owner.** architecture, sprint N+3.

### ARCH-GR-013 — Severity: Medium
**Finding.** Guardrail latency not budgeted; some features exceed latency SLO due to guardrail overhead.
**Recommendation.** Latency budget per section 4.7; cheaper alternatives where appropriate.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-GR-014 — Severity: Medium
**Finding.** Sibling security repo's threats not mapped to this document's placements; coverage gaps possible.
**Recommendation.** Per-threat mapping per section 5.6; review quarterly.
**Owner.** architecture + security, sprint N+4.

### ARCH-GR-015 — Severity: Low
**Finding.** Guardrail-bypass detection only via comparison; no preventive layer.
**Recommendation.** Network / IAM enforcement per section 6.3.
**Owner.** ai-platform-eng + ops, sprint N+4.

### ARCH-GR-016 — Severity: Low
**Finding.** Per-guardrail dashboards absent; quality trends invisible.
**Recommendation.** Per-control monitoring per section 6.6.
**Owner.** ai-platform-eng + ops, sprint N+5.

### ARCH-GR-017 — Severity: Low
**Finding.** Guardrail-decision template not in PR / design review; controls discussed ad-hoc.
**Recommendation.** Standard checklist per section 4.
**Owner.** architecture, sprint N+5.

### ARCH-GR-018 — Severity: Low
**Finding.** Cross-team guardrail standards not communicated; teams reinvent.
**Recommendation.** Shared documentation; security guild / similar.
**Owner.** ai-platform-eng + security, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building guardrail architecture:

- [ ] **Sprint 0 — threat model.** Per sibling security repo; identify the threats.
- [ ] **Sprint 0 — choke point review.** AI gateway, tool registry, retrieval client, agent runner present?
- [ ] **Sprint 1 — gateway implementation.** Per [ai-gateway-pattern.md](./ai-gateway-pattern.md); deploy.
- [ ] **Sprint 1 — input guardrails at gateway.** PII, injection, authorization, length, rate limit.
- [ ] **Sprint 1 — output guardrails at gateway.** PII leak, format, moderation.
- [ ] **Sprint 2 — tool registry.** Per [tool-call-authorization.md](./tool-call-authorization.md).
- [ ] **Sprint 2 — retrieval client scope.** Per [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md).
- [ ] **Sprint 3 — layered defense.** For high-stakes controls (PHI, financial, etc.).
- [ ] **Sprint 3 — fail-loud.** Per section 6; alerting; logging.
- [ ] **Sprint 4 — bypass detection.** Network / IAM policy.
- [ ] **Sprint 4 — guardrail eval.** Per-control quality monitoring.
- [ ] **Sprint 5 — per-feature audit.** Each AI feature against the framework.
- [ ] **Ongoing — design review.** New features include guardrail-placement review.

For a team retrofitting:

- [ ] **Sprint 0 — audit current placements.** Per category, where do controls live?
- [ ] **Sprint 1 — move gateway-eligible controls.** From calling code to gateway.
- [ ] **Sprint 2 — fix fail-open defaults.** Switch to fail-closed.
- [ ] **Sprint 3 — add bypass detection.** Identify gaps before incidents.
- [ ] **Sprint 4 — add layered defense.** For the highest-stakes controls.

A team that completes the sequence has guardrails that survive system evolution. A team that doesn't has controls that drift and fail at the wrong moments.

---

## 12. References

- [ai-gateway-pattern.md](./ai-gateway-pattern.md) — the primary choke point for input and output guardrails.
- [tool-call-authorization.md](./tool-call-authorization.md) — tool-registry's role in intermediate-step guardrails.
- [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md) — retrieval client's role.
- [policy-as-code-for-ai.md](./policy-as-code-for-ai.md) — policy as the framework for what guardrails enforce.
- [content-moderation-architecture.md](./content-moderation-architecture.md) — moderation as one specific guardrail category.
- [refusal-and-escalation-design.md](./refusal-and-escalation-design.md) — what the system does when a guardrail rejects.
- [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — tenant isolation at the data layer.
- [data-architecture-for-ai/freshness-architecture.md](../data-architecture-for-ai/freshness-architecture.md) — freshness as a retrieval guardrail.
- [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) — provenance enables guardrail decisions.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md) — agent runner as choke point.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/tool-architecture.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/tool-architecture.md) — tool engineering depth.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — fail-loud at the engineering layer.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — cost guardrails.
- Sibling repo: ai-security-reference-architecture — threat model, detection content, response runbooks.
- Defense-in-depth literature (NIST, NSA cybersecurity frameworks) — broader pattern that informs the layered defense approach here.
