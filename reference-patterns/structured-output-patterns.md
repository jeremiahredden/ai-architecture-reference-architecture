# Structured Output Patterns

> **Audience.** Architects deciding how the model's output flows into downstream code. Tech leads whose "JSON" outputs sometimes aren't actually JSON. Anyone whose parser needs a try/except to handle the LLM's creative interpretations of the schema. **Scope.** The *architectural* patterns: JSON Schema with tool-calling, constrained decoding, output validation + repair, failure modes, and the strict-schema-vs-freeform-with-extraction decision. Not the engineering of the validator (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s [structured-output-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/structured-output-engineering.md)). Not the schema-as-contract discipline (see [prompt-engineering/prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) in the engineering sibling). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Production AI systems almost universally consume model outputs programmatically: a parser extracts fields; a UI renders structured content; a downstream service receives a typed object. The contract between the model and the downstream code is the schema — what fields, what types, what constraints, what shape.

Without architectural attention, this contract is brittle:

- Model outputs free-form text that "looks like JSON" but isn't (missing comma; smart quotes; trailing notes).
- Schema drift: the prompt's expected schema and what the model produces diverge subtly over time.
- Hallucinated fields: the model adds fields the schema didn't ask for.
- Truncated outputs: the response token limit cuts off mid-JSON.
- Edge cases the prompt didn't anticipate: empty arrays, null values, escaped strings.

The architectural decision is how the team enforces and verifies the contract. The 2026 toolkit has matured significantly:

- Provider-native structured output (Anthropic's tool use, OpenAI's structured outputs / function calling, JSON mode).
- Constrained decoding libraries (Outlines, Instructor, Guidance, vendor SDK features).
- Output validation libraries (Pydantic, Zod, ajv).
- Repair-loop patterns for when validation fails.

The decision: which mechanism for which workload; how strict to be; what to do when validation fails; how to evolve schemas without breaking consumers.

This document is opinionated about four things:

1. **Structured output should be enforced, not encouraged.** Asking the model nicely is unreliable; constrained decoding or strict validation + retry is reliable.
2. **The schema is the contract.** Versioned, reviewed, eval'd, evolved with discipline. Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md).
3. **Provider-native is the default.** Built-in tool-calling and structured-output features beat custom approaches in 2026; reach for libraries only when the provider's offering falls short.
4. **Repair loops are bounded.** Validation failure shouldn't trigger unbounded retries; bounded with clear fail-mode.

Structure: (2) the structured-output mechanisms; (3) the strict vs freeform decision; (4) common failure modes; (5) validation and repair patterns; (6) schema evolution; (7) the architectural placement; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The structured-output mechanisms

The 2026 toolkit.

### 2.1 Provider-native tool calling

Anthropic's tool use, OpenAI's function calling. The model produces a structured "tool call" that the SDK presents as a parsed object:

```python
response = client.messages.create(
    tools=[{
        "name": "extract_patient_summary",
        "description": "...",
        "input_schema": {...JSON Schema...}
    }],
    # ...
)
# response.content[0] is a typed tool_use with parsed arguments
```

**Pros.**
- Native to the provider; tightly integrated.
- Schema enforced server-side; validated before return.
- Well-tested; mature.

**Cons.**
- Some providers' schema support is partial (not all JSON Schema features).
- Cross-provider portability requires abstraction.

**When right.** Default for most use cases in 2026.

### 2.2 Provider-native structured output (JSON mode)

OpenAI's `response_format: {"type": "json_schema", "json_schema": {...}}` (or the simpler JSON mode); some Anthropic equivalents.

**Pros.**
- Schema enforcement at generation time.
- No need to frame as "tool call" semantically.

**Cons.**
- Provider-specific.
- May have schema-feature limitations.

**When right.** When the output is a single structured response (not a tool-call abstraction).

### 2.3 Constrained decoding (Outlines / Guidance / library-level)

Libraries that constrain the model's generation to a grammar / schema at decode time:

```python
from outlines import generate
result = generate.json(model, schema=PatientSummary)(...)
```

**Pros.**
- Provider-agnostic.
- Works with open-source models.
- Can enforce complex grammars (regex, BNF) beyond JSON Schema.

**Cons.**
- Requires direct model access (logit-level intervention); not all hosted providers support.
- Library maintenance overhead.

**When right.** Self-hosted models; specialised grammars; providers without native structured-output.

### 2.4 Prompt + post-hoc validation (the legacy pattern)

The prompt asks for JSON; the application parses; validation catches errors:

```python
response = client.messages.create(prompt="Output JSON with fields...")
try:
    data = json.loads(response.content)
    validated = PatientSummary.parse_obj(data)
except (json.JSONDecodeError, ValidationError):
    # retry or fail
```

**Pros.**
- Works with any provider.
- Simple.

**Cons.**
- Parsing failures common (the model produces "JSON-ish" not JSON).
- Repair loops add cost and latency.

**When right.** Limited use cases in 2026; mostly when provider features and constrained decoding aren't options.

### 2.5 The mechanism comparison

| Mechanism | Reliability | Provider lock-in | Latency overhead | Use case |
| --- | --- | --- | --- | --- |
| Provider tool-calling | High | Medium | Low | Default; tool-call semantics fit |
| Provider structured output | High | High | Low | Single structured response |
| Constrained decoding | High | Low | Low-medium | Self-hosted; complex grammars |
| Post-hoc validation | Low-medium | None | High (with repair) | Legacy; last resort |

In 2026, default to provider-native; fall back to constrained decoding for self-hosted; avoid post-hoc validation for new builds.

### 2.6 The "structured output isn't always JSON" perspective

Structured output can be:

- JSON (most common).
- XML (some workflows).
- Code (SQL, scripts, configuration).
- Tabular (CSV, table).
- Restricted natural language (e.g., with required sections).

The mechanisms apply per format; some formats have specialised approaches (e.g., constrained SQL generation).

---

## 3. The strict vs freeform decision

The architectural choice for each output point.

### 3.1 The strict-schema approach

The output is rigidly schematised:

```json
{
  "patient_id": "uuid-...",
  "summary": "...",
  "key_concerns": ["concern1", "concern2"],
  "recommendations": [
    {"action": "...", "priority": "high"}
  ],
  "confidence": 0.85,
  "citations": [...]
}
```

Every field's type and constraints defined; downstream code consumes deterministically.

**Pros.**
- Downstream code is reliable; no parsing surprises.
- Easy to validate and eval.
- Schema as contract per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md).

**Cons.**
- Model's expressiveness constrained.
- New "shape" needs schema change (versioning overhead).
- Some content doesn't fit rigid structure naturally.

**When right.** Most production use cases where output is consumed programmatically.

### 3.2 The freeform-with-extraction approach

The model produces freeform output; a second step (parser, classifier, LLM extractor) extracts structured data:

```
Model produces: "The patient's main concern is X. Their priority recommendation is Y..."

Extractor (downstream LLM or regex): 
  → {"key_concern": "X", "priority_recommendation": "Y"}
```

**Pros.**
- Model's expressiveness preserved.
- New extraction patterns added without prompt changes.
- Some content benefits (narrative responses, complex reasoning chains).

**Cons.**
- Extraction itself can fail.
- Two-step process; more cost and latency.
- Harder to ensure schema compliance.

**When right.** When the output is primarily human-readable (and a parsed view is secondary); when the model's narrative quality matters.

### 3.3 The hybrid approach

Most production responses have both:

- Structured fields (the parsed view; downstream code consumes).
- A free-form response text (the human-readable view; user sees).

```json
{
  "structured_fields": {
    "patient_id": "...",
    "summary_type": "encounter_summary",
    "key_concerns": [...]
  },
  "human_response": "Here's the summary of the patient's recent visit..."
}
```

Strict on the fields; free-form on the narrative; both validated.

This is the production-default for most user-facing AI features.

### 3.4 The decision criteria

| Criterion | Strict | Freeform | Hybrid |
| --- | --- | --- | --- |
| Downstream consumes structured? | Yes | Yes (via extraction) | Yes |
| User sees output directly? | Schema-rendered | Yes | Yes |
| Schema rigidity acceptable? | Yes | No | Acceptable for fields |
| New patterns frequent? | Adds versioning friction | Easier | Mixed |
| Quality on subjective dimensions | Bound by schema | Higher | Best of both |

Most workloads: hybrid. Pure strict for backend API endpoints; pure freeform for purely-narrative consumer features.

### 3.5 The "schema as cage" anti-pattern

Some teams use schemas that are too restrictive:

- Enum with 3 values when 10 are needed.
- Max-length too short for content.
- Required fields that don't fit some cases.

The model's outputs hit the schema's edges; quality drops. The schema is meant to enable, not cage.

Iterate on the schema as production reveals gaps.

### 3.6 The "no schema" failure

The opposite: no structure; downstream code parses ad-hoc; behaviour is fragile.

For any output that's parsed programmatically, structure it. The cost of structure is much less than the cost of parsing surprises.

---

## 4. Common failure modes

What goes wrong in production.

### 4.1 Truncation

The response token limit is hit mid-output. The JSON is incomplete:

```json
{
  "summary": "Long content that fills the response budget and then is cut off mid-sent
```

Downstream parsing fails.

**Corrective.** Set response budget generously; estimate from content length; warn if hit consistently (means the schema demands more than budget allows).

### 4.2 Schema drift

The model's outputs subtly differ from the schema over time:

- Field that was always present is now sometimes absent.
- Type inferred as int now sometimes returned as string.
- Enum values include new values the schema didn't allow.

**Corrective.** Provider-native enforcement catches at generation; for legacy post-hoc validation, eval flags drift.

### 4.3 Hallucinated fields

The model adds fields the schema didn't request:

```json
{
  "summary": "...",
  "key_concerns": [...],
  "my_thoughts": "I also wanted to mention..."  // not in schema
}
```

**Corrective.** Strict schemas reject extra fields; downstream code ignores them; either works depending on enforcement.

### 4.4 Escape character issues

JSON within strings can have escape problems:

```json
{
  "code_example": "if (x == "value") { ... }"  // unescaped quotes break parsing
}
```

**Corrective.** Provider-native handling typically escapes correctly; library validators catch issues.

### 4.5 Encoding issues

Smart quotes, Unicode edge cases, emoji:

```json
{
  "patient_name": "Smith—J"  // em-dash that some parsers handle, some don't
}
```

**Corrective.** Standard UTF-8 throughout; validate before downstream consumption.

### 4.6 Wrong data type within valid schema

Schema allows array; model returns single value (or vice versa). Validation catches:

```json
{
  "citations": "DOC-123"  // expected array of strings
}
```

**Corrective.** Schema enforcement plus repair loop (with bounded retries).

### 4.7 Missing required field

The model omits a required field:

```json
{
  "summary": "..."
  // missing required "patient_id"
}
```

**Corrective.** Provider-native enforcement; or post-hoc validation + repair.

### 4.8 The "garbage in valid format" failure

The output is valid JSON matching the schema but content is wrong:

```json
{
  "patient_id": "actual_uuid",
  "summary": "Lorem ipsum dolor sit amet..."  // hallucinated content
}
```

Schema validation passes; content is wrong. Different failure mode; requires content-level eval per the engineering sibling.

### 4.9 The "schema enforcement reduces quality" tension

Strict enforcement sometimes reduces quality:

- The model's natural response would have been better; the schema's constraints distorted it.
- The schema's enums don't capture all valid values.

The fix: iterate on the schema; ensure constraints match the natural response shape.

---

## 5. Validation and repair patterns

What to do when validation fails.

### 5.1 The bounded repair loop

```python
def call_with_repair(prompt, schema, max_attempts=2):
    for attempt in range(max_attempts):
        response = model.call(prompt)
        try:
            return schema.parse(response)
        except ValidationError as e:
            prompt = update_prompt_with_error(prompt, response, e)
    raise ValidationFailed("Exceeded repair attempts")
```

**Pros.**
- Catches recoverable errors.
- Bounded cost / latency.

**Cons.**
- Adds latency per attempt.
- Some errors won't recover (the model's understanding is wrong, not its formatting).

### 5.2 The "fail fast" alternative

No repair; on first validation failure, return error to the caller:

**Pros.**
- Fastest path.
- Caller handles error (which they need to anyway).

**Cons.**
- Loses repairable cases.
- Lower overall success rate.

**When right.** Latency-critical; or workloads where format errors are rare.

### 5.3 The provider-side enforcement

Provider-native tool-calling typically retries server-side; the client gets a valid result or a failure indication. No client-side repair needed.

This is the default in 2026; client-side repair is for cases the provider's enforcement misses.

### 5.4 The repair attempt cost

Each repair attempt is a full model call. For a 5,000-token prompt + 1,000-token response: ~$0.018 (Sonnet). Two repair attempts: ~$0.054. Compared to no repair (single $0.018 call): 3× the cost.

Worth it on cases that actually recover; wasted on cases that don't. Eval the recovery rate.

### 5.5 The repair prompt

The retry prompt should explain the failure:

```
Your previous response had these issues:
- Missing required field 'patient_id'
- 'confidence' value 'high' is not a valid number

Please correct and try again. Original task: [task]
```

The explanation helps the model recover; vague "try again" rarely does.

### 5.6 The "schema enforces but model strains" pattern

When the model keeps trying to add fields the schema doesn't allow, or keeps using enum values not in the schema:

- The schema may be wrong (constraints too tight).
- The prompt may be wrong (asking for things the schema can't represent).

Eval surfaces this; iterate the schema or prompt.

### 5.7 The fail-mode for unrecoverable cases

When repair exhausts:

- Return error to caller; let calling code handle.
- The error includes the model's last (invalid) output and the validation error.
- Logged for engineering investigation.

Don't silently fall back to a partial / hallucinated response.

### 5.8 The eval surface for repair

- Repair-attempt rate (how often does the model fail validation?).
- Recovery rate (when validation fails, how often does repair succeed?).
- Per-failure-type distribution (what kinds of errors are common?).

Aggregations inform schema / prompt / provider tuning.

---

## 6. Schema evolution

Schemas change; consumers depend on them. Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) in the engineering sibling.

### 6.1 The version dimension

Every schema has a version. Consumers pin a version. The model's output is generated against the pinned version.

### 6.2 Backward-compatible changes (minor)

- Adding optional fields.
- Widening enum values (in some cases).
- Loosening constraints (less restrictive).

Consumers can adopt at their own pace.

### 6.3 Breaking changes (major)

- Removing fields.
- Renaming fields.
- Changing types.
- Tightening constraints.

Consumers must migrate before the new schema is used.

### 6.4 The schema migration pattern

Old and new schemas coexist for a window:

- Old consumers on old schema.
- New consumers on new schema.
- Old schema deprecated; consumers migrate.
- Old schema removed after deprecation window.

Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) section 5.

### 6.5 The schema test

Schemas have tests:

- Valid example outputs pass validation.
- Invalid example outputs fail validation appropriately.
- Backward compatibility tests (old-format outputs still parse).

Per-schema in CI.

### 6.6 The schema impact analysis

Before changing a schema:

- Which features use it (consumer registry).
- What's the impact of the change per consumer.
- What migration is needed.

Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md).

### 6.7 The schema's natural granularity

Some teams use one big schema for "the response"; others use multiple per-feature schemas.

- Bigger schemas: fewer; easier to reason about; harder to evolve.
- Smaller per-feature schemas: more; per-feature evolution; more consumer coordination overhead.

Most production: per-feature schemas with shared components (for common fields like citations).

---

## 7. The architectural placement

Where structured output sits in the architecture.

### 7.1 The model boundary

The structured output happens at the model call. The provider's tool-calling or structured-output API returns the parsed object directly.

### 7.2 The gateway role

The AI gateway (per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md)) handles:

- Schema versioning per request.
- Validation on the returned object.
- Repair loop (if configured).
- Error to caller on unrecoverable.

The gateway is the choke point; calling code asks for "the parsed response," not for "the raw JSON."

### 7.3 The downstream consumer

Consumers receive typed objects:

```python
result: PatientSummary = await ai_gateway.call(
    feature="patient-summary",
    input=patient_data,
    output_schema=PatientSummary,
)
# result is parsed; no JSON manipulation in caller code
```

The gateway's interface is typed; the caller writes type-safe code.

### 7.4 The schema repository

Schemas live in a versioned repository (per [prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md)):

- Per-feature schemas.
- Versioned.
- Tests.
- Documentation.

Consumers and producers (the model calls) reference schemas by id + version.

### 7.5 The eval surface

Structured-output eval includes:

- Schema validity rate (how often does validation pass?).
- Per-field accuracy (does each field have the right content?).
- Schema-drift detection.

### 7.6 The observability

Per call:

- Schema id + version.
- Validation result (pass / fail / repaired).
- Repair attempts count.
- Per-field nulls / missing.

Aggregations:

- Per-schema validity rate.
- Per-schema repair rate.
- Schema-version usage over time.

---

## 8. Worked Meridian example

Meridian's structured-output architecture.

### 8.1 The schemas

| Feature | Schema | Version | Usage |
| --- | --- | --- | --- |
| Care Coordinator | CareCoordinatorResponse | v23 | All interactions |
| Patient API AI Assist | PatientApiResponse | v8 | All interactions |
| Patient Summary | PatientSummary (SOAP format) | v5 | All summaries |
| Analytics Copilot | SQLGenerationResponse | v4 | All SQL generation |
| Various tools | Per-tool input/output schemas | per-tool | Tool calls |

All schemas defined in `meridian-schemas` repository; versioned; PR-reviewed.

### 8.2 The Care Coordinator schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "meridian/care-coordinator/v23",
  "type": "object",
  "required": ["response", "escalate"],
  "additionalProperties": false,
  "properties": {
    "response": {
      "type": "string",
      "maxLength": 1500,
      "description": "User-facing response, clinical tone, max 1500 chars."
    },
    "escalate": {
      "type": "boolean",
      "description": "Should this be escalated to a human."
    },
    "escalation_reason": {
      "type": "string",
      "description": "If escalate=true, the reason."
    },
    "tools_called": {
      "type": "array",
      "items": {"type": "string"},
      "description": "List of tools called during this turn."
    },
    "citations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["doc_id"],
        "properties": {
          "doc_id": {"type": "string"},
          "section": {"type": "string"},
          "relevance": {"type": "number"}
        }
      }
    }
  }
}
```

Strict; required fields enforced; bounded sizes.

### 8.3 The enforcement

- Anthropic tool-calling enforces the schema server-side.
- The gateway validates the parsed response (defense in depth).
- Invalid responses trigger repair loop (max 1 attempt; bounded cost).
- Unrecoverable failures raise error to caller.

### 8.4 The repair-loop metrics

Production data:

- Validation failure rate: ~0.8% of responses.
- Repair success rate: ~85% (most failures are recoverable).
- Final failure rate: ~0.12% of responses (these become errors to the caller).

The repair loop adds latency to ~0.8% of requests (acceptable); cost overhead negligible.

### 8.5 The schema evolution

Care Coordinator schema has evolved v1 → v23 over 18 months:

- v23 (current): added `confidence_score` field (minor, additive).
- v22 → v23: minor bump; backward-compatible.
- v15 → v16 → v17: a series of minor refinements (additional optional fields).
- v10 → v11: major bump (removed `tools_called` and added `relevance` to citations).

Most evolution is additive; the one major bump was coordinated with consumers.

### 8.6 Schema testing

CI tests per schema:

- Valid examples pass.
- Invalid examples fail with appropriate errors.
- Backward compat: prior schema outputs parse under new schema (for minor bumps).
- Round-trip: parse + serialise + re-parse produces same.

### 8.7 The hybrid pattern in practice

The Care Coordinator response is hybrid:

- `response` field: human-readable; user sees this.
- Other fields: structured; consumed by the application code (escalation routing, citation rendering, analytics).

Both validated; both relied on.

### 8.8 The provider-native default

All Meridian features use Anthropic tool-calling for structured output (Anthropic is the primary provider). The team would have alternatives (Outlines for self-hosted; OpenAI structured outputs for failover) but the production default is provider-native.

### 8.9 What worked

- **Provider-native enforcement.** Reliable; minimal client-side code.
- **Strict schemas.** Downstream code consumes typed objects without surprises.
- **Schema versioning.** Coordinated evolution; no consumer breakage in 18 months.
- **Bounded repair.** Failures recovered without unbounded cost.

### 8.10 What didn't work initially

- **Post-hoc validation (early v1-v3).** Used before provider-native tool calling was reliable; failure rate ~5%; repair loops were expensive. Switched to provider-native; failure rate dropped to <1%.
- **Schema-as-code-comment.** First schemas were comments in the prompt; consumers ad-hoc parsed. Centralised schema repository eliminated this.
- **Field name inconsistency.** Schemas for different features used inconsistent field naming (snake_case vs camelCase). Standardised on snake_case org-wide; migrated.

---

## 9. Anti-patterns

### 9.1 "Ask nicely for JSON"

Prompt asks for JSON; no enforcement; outputs are JSON-ish at best.

**Corrective.** Provider-native or constrained decoding per section 2.

### 9.2 "Unbounded repair loop"

Validation fails → retry → fails → retry → cost explodes.

**Corrective.** Bounded repair per section 5.1.

### 9.3 "No schema versioning"

Schema changes; consumers break silently.

**Corrective.** Version per section 6.

### 9.4 "Schema as code comment"

Schema lives only in the prompt or docstring; no enforcement; no central reference.

**Corrective.** Schema repository per section 7.4.

### 9.5 "Too-strict schema cages model"

Schema constraints prevent natural responses; quality drops.

**Corrective.** Iterate schema per section 3.5.

### 9.6 "No structured output where downstream needs it"

Free-form output; downstream parses ad-hoc; fragile.

**Corrective.** Structured output per section 3.

### 9.7 "Validation not measured"

Validation failure rate unknown; can't tell if production has issues.

**Corrective.** Per section 5.8 / 7.6.

### 9.8 "Provider-native ignored"

Using post-hoc validation when provider-native is available; missing free reliability.

**Corrective.** Provider-native per section 2.1.

---

## 10. Findings (sprint-assignable)

### ARCH-SOP-001 — Severity: Critical
**Finding.** Downstream parsing relies on "JSON-ish" output; failures common; repair loops unbounded.
**Recommendation.** Provider-native enforcement per section 2.1; bounded repair per section 5.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-SOP-002 — Severity: Critical
**Finding.** Schemas not versioned; changes break consumers.
**Recommendation.** Versioning per section 6; consumer migration patterns.
**Owner.** ai-platform-eng + feature teams, sprint N+1.

### ARCH-SOP-003 — Severity: High
**Finding.** Validation failure rate unmeasured; production quality blind.
**Recommendation.** Observability per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+2.

### ARCH-SOP-004 — Severity: High
**Finding.** Schemas live in prompts only; no central repository.
**Recommendation.** Schema repository per section 7.4.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-SOP-005 — Severity: High
**Finding.** Schema-eval cases absent; valid/invalid samples not tested.
**Recommendation.** Schema tests per section 6.5.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-SOP-006 — Severity: High
**Finding.** Post-hoc validation used where provider-native is available.
**Recommendation.** Provider-native default per section 2.1.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-SOP-007 — Severity: Medium
**Finding.** Schema impact analysis not done before changes; consumers surprised.
**Recommendation.** Per section 6.6.
**Owner.** architecture + ai-platform-eng, sprint N+3.

### ARCH-SOP-008 — Severity: Medium
**Finding.** Field naming inconsistent across schemas; cross-feature integration brittle.
**Recommendation.** Naming convention per section 8.10.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-SOP-009 — Severity: Medium
**Finding.** Repair-loop attempts unbounded; cost can spike.
**Recommendation.** Bounded per section 5.1; per [agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md).
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-SOP-010 — Severity: Medium
**Finding.** Schemas too strict; production reveals quality drop on edge cases.
**Recommendation.** Per section 3.5; iterate schemas on eval feedback.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-SOP-011 — Severity: Medium
**Finding.** Hybrid pattern (structured + free-form text) not used; user-facing features lose narrative quality.
**Recommendation.** Per section 3.3.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-SOP-012 — Severity: Medium
**Finding.** Response truncation not handled; mid-output cutoffs produce invalid JSON.
**Recommendation.** Per section 4.1; budget generously; alert on truncation rate.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-SOP-013 — Severity: Medium
**Finding.** Schema deprecation lifecycle informal; old versions linger.
**Recommendation.** Per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) section 5.
**Owner.** ai-platform-eng + consumer teams, sprint N+4.

### ARCH-SOP-014 — Severity: Medium
**Finding.** Cross-provider portability of structured output not planned; provider switch would break.
**Recommendation.** Abstraction layer per section 2.5; fallback support.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-SOP-015 — Severity: Low
**Finding.** Schema documentation thin; new engineers don't know what's available.
**Recommendation.** Per-schema documentation in repository.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-SOP-016 — Severity: Low
**Finding.** Per-field accuracy not eval'd; some fields may have systematic quality issues.
**Recommendation.** Field-level eval per section 7.5.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-SOP-017 — Severity: Low
**Finding.** Constrained decoding alternatives not evaluated; team locked to provider-native even when alternatives might work better.
**Recommendation.** Quarterly review of constrained-decoding options.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-SOP-018 — Severity: Low
**Finding.** Schema-to-typed-object generation manual; downstream code has type drift.
**Recommendation.** Generate typed bindings from schemas (Pydantic, Zod, etc.); CI-enforced.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building structured output:

- [ ] **Sprint 0 — schema design.** Per feature; strict / hybrid / freeform decision.
- [ ] **Sprint 0 — schema repository.** Versioned; central.
- [ ] **Sprint 1 — provider-native integration.** Tool-calling or structured output via SDK.
- [ ] **Sprint 1 — validation.** Schema enforcement (provider + client-side).
- [ ] **Sprint 2 — repair loops.** Bounded; with appropriate retries.
- [ ] **Sprint 2 — observability.** Validation rate; repair rate; per-schema metrics.
- [ ] **Sprint 3 — schema tests.** Valid/invalid; backward compat; round-trip.
- [ ] **Sprint 3 — typed bindings.** Pydantic/Zod from schemas.
- [ ] **Sprint 4 — schema versioning discipline.** Per-version pins; consumer migration.
- [ ] **Sprint 4 — failure mode.** Errors propagated to caller cleanly.
- [ ] **Ongoing — schema evolution review.** Quarterly per [prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md).

For a team retrofitting:

- [ ] **Sprint 0 — audit.** Per feature: what's the current schema; how is it enforced; what's the failure rate.
- [ ] **Sprint 1 — provider-native migration.** Move from post-hoc to native where possible.
- [ ] **Sprint 2 — repository.** Centralise schemas.
- [ ] **Sprint 3 — versioning.** Apply discipline.

A team that completes the sequence has structured output that's reliable, evolvable, and observable. A team that doesn't has parsing surprises and schema drift.

---

## 12. References

- [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) — structured output enables reliable RAG responses.
- [agent-topologies.md](./agent-topologies.md) — agent tool calls use structured output.
- [multi-model-orchestration.md](./multi-model-orchestration.md) — orchestrators consume structured output.
- [hybrid-retrieval-patterns.md](./hybrid-retrieval-patterns.md) *(coming)* — structured filter inputs use the same patterns.
- [pattern-anti-patterns.md](./pattern-anti-patterns.md) *(coming)* — monolithic-prompt-as-architecture anti-pattern related.
- [guardrails-and-policy-architecture/ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md) — gateway as the home for schema enforcement.
- [guardrails-and-policy-architecture/content-moderation-architecture.md](../guardrails-and-policy-architecture/content-moderation-architecture.md) — inline moderation via constrained decoding.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/structured-output-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/structured-output-engineering.md) — engineering depth.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) — schema as contract.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) — schema versioning.
- Anthropic tool use, OpenAI structured outputs / function calling, JSON mode documentation.
- Outlines, Instructor, Guidance — constrained-decoding libraries.
- Pydantic, Zod, ajv — validation libraries.
