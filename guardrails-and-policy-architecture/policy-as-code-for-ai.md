# Policy as Code for AI

> **Audience.** Architects designing the policy layer for AI systems — which models are approved, which tools are allowed, which features can process which data classifications. Tech leads whose AI policy lives as a Confluence document that nobody enforces. Compliance teams whose audit asks "show me where this policy is enforced" and the team can't answer. **Scope.** The *architectural* practice of policy as code — bundle and evaluation patterns, OPA / Cedar / custom engines, deployment integration, per-tenant overrides. Not the threat-model policies (sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)). Not the engineering of policy enforcement (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI policy is what determines which choices the system makes: which model to use for what; which tools an agent can call for which features; which content classes require human review; which tenants can access which capabilities. Without an architectural pattern for policy, these decisions live as code conditionals scattered across the system. A change to "which model is approved for PHI workloads" requires hunting through six services to find the conditionals.

The corrective pattern, mature in adjacent fields (authorization with OPA / Cedar / similar), is *policy as code*: policies are written as versioned, deployable artefacts; the policy engine evaluates at decision points; application code asks "is this allowed?" rather than implementing the policy logic itself.

In AI systems, policy-as-code becomes architecturally important because:

- **AI policies change frequently.** Model versions deprecate; new tools launch; new data classifications emerge; regulatory requirements shift.
- **AI policies span teams.** Security defines data classification rules; platform defines model approvals; product defines feature access; legal defines refusal patterns. Without a shared policy mechanism, the teams' decisions don't compose.
- **AI policies must be auditable.** Compliance audits ask "prove this policy is enforced." Code conditionals don't audit well; centralised policy evaluation does.
- **AI policies have tenant variations.** Premium tenants get different model access; regulated tenants get stricter content controls. Per-tenant policy is a first-class need.

The architectural decision is: where the policies live, what engine evaluates them, how they integrate with the AI gateway and other choke points, how they version and deploy.

This document is opinionated about four things:

1. **Policies are code artefacts, not documents.** Written, versioned, reviewed, deployed, observed.
2. **Policy evaluation is centralised at a small number of points.** Typically the AI gateway, the tool registry, and the agent runner. Application code asks; doesn't decide.
3. **The policy engine matters less than the policy practice.** OPA, Cedar, custom — pick what fits; the discipline (versioning, eval, deployment, observation) is what matters.
4. **Per-tenant overrides are first-class.** AI policy varies by tenant tier, regulatory context, contractual commitment. The architecture supports overrides; doesn't require code branches.

Structure: (2) what AI policies cover; (3) policy as code pattern; (4) the policy engine choice; (5) the policy bundle and deployment; (6) per-tenant overrides; (7) integration with the choke points; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. What AI policies cover

The scope of policy-as-code in AI systems.

### 2.1 Model approval policies

Which models are approved for which uses:

- "Model X is approved for processing PHI."
- "Model Y is approved for processing financial data."
- "Model Z is approved for general internal use only; not for customer-facing."
- "Model fine-tuned variant ABC is approved only for the patient-summary feature."

These policies derive from:

- Vendor terms (which models cover what data classifications under BAA / DPA).
- Data classification (which data can flow to which model).
- Approval workflows (which models the security team has approved).
- Cost / quality decisions.

### 2.2 Tool access policies

Which agents / features can call which tools:

- "The care-coordinator agent can call patient-data tools; the analytics-copilot cannot."
- "Tool X requires elevated authorization; only premium-tier users can trigger calls that use it."
- "Tool Y is rate-limited to N calls per hour per tenant."

These policies derive from:

- Feature scope (each feature has a defined tool catalogue).
- Authorization (who can trigger which actions).
- Cost control (expensive tools have stricter policies).

### 2.3 Data classification policies

Which data classifications can flow where:

- "PHI cannot leave the tenant's region."
- "Confidential data cannot be sent to non-BAA model providers."
- "Public data has no restrictions."
- "PII cannot be logged unredacted."

These derive from:

- Data classification taxonomy (the organisation's data labels).
- Regulatory requirements (HIPAA, GDPR, etc.).
- Contractual commitments.

### 2.4 Content policies

What content the system can produce or refuse:

- "Medical-advice responses must include a 'consult your physician' note."
- "Responses about competitor products are refused."
- "Responses that mention specific patient names must cite a source."
- "Output containing PHI is redacted before logging."

These derive from:

- Brand and product positioning.
- Regulatory requirements.
- Safety considerations.
- Legal review.

### 2.5 Feature access policies

Which users / tenants can use which AI features:

- "Care-coordinator is available to clinical staff only."
- "Analytics-copilot is available to analyst-role users only."
- "Beta features are available to opt-in tenants only."

These derive from:

- Product entitlements.
- Subscription tier.
- Role-based access control.

### 2.6 Review-required policies

Which outputs / actions require human review:

- "Care plan changes always require physician approval."
- "Outputs with confidence < 0.7 are flagged for review."
- "Actions affecting more than 100 records require approval."

These derive from:

- Risk tolerance.
- Regulatory requirements.
- Quality investment decisions.

### 2.7 The policy taxonomy

| Category | Examples | Owner |
| --- | --- | --- |
| Model approval | Model X for PHI; Model Y for general | Security + Platform |
| Tool access | Feature A can call tool X; B can't | Security + Platform + Product |
| Data classification | PHI to BAA models only | Security + Legal |
| Content | Medical-advice disclaimer | Legal + Product |
| Feature access | Care-coordinator for clinical-role | Product + Platform |
| Review-required | Care-plan changes need MD approval | Product + Compliance |

Each category has a natural owner; cross-team coordination required.

---

## 3. Policy as code pattern

The mechanical layer.

### 3.1 The policy artefact

A policy is a code artefact:

```rego
# OPA Rego example
package ai.model_approval

allow {
    input.data_classification == "PHI"
    input.model in {"anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-7"}
    input.tenant_baa == true
}

allow {
    input.data_classification == "general"
    input.model in {"anthropic/claude-haiku-4-5", "anthropic/claude-sonnet-4-6", "openai/gpt-5"}
}
```

Or in Cedar:

```cedar
permit (
    principal,
    action == Action::"callModel",
    resource is Model
) when {
    resource.dataClassification == "PHI" &&
    resource.model in ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-7"] &&
    resource.tenantBaa == true
};
```

The policy is:

- **Versioned.** Source control; PRs; review.
- **Tested.** Unit tests; integration tests.
- **Deployed.** Through CI/CD with the rest of the codebase.
- **Observable.** Decisions are logged.

### 3.2 The application's role

Application code doesn't implement policy. It asks:

```python
# At decision point in the AI gateway:
decision = policy_engine.evaluate(
    policy="ai.model_approval",
    input={
        "data_classification": context.data_classification,
        "model": requested_model,
        "tenant_baa": tenant_metadata.has_baa,
    }
)

if not decision.allow:
    raise PolicyDenied(decision.reason)
```

The policy engine decides; the application acts on the decision.

### 3.3 The decision shape

A policy evaluation returns:

- **Decision.** Allow / deny.
- **Reason.** Human-readable explanation (for logging and user-facing messages).
- **Obligations.** Additional requirements the application must satisfy (e.g., "if you call this model, you must also log the decision to audit_trail X").

Rich decisions support more nuanced control than allow/deny.

### 3.4 The "policy explosion" risk

When policies multiply, they can become hard to reason about:

- Policy A says "allow X."
- Policy B says "deny X."
- Which wins?

Engines have conflict resolution rules; the team must understand and design accordingly. The rule of thumb: keep policies simple; few enough that the team can reason about interactions.

### 3.5 The "policy as comment in code" anti-pattern

Some teams write the policy as a comment near the conditional that implements it. The comment is the documentation; the code is the enforcement. Changes can desynchronise.

Corrective: the policy is the source of truth; the application code calls the policy engine; no parallel implementation.

### 3.6 The "policy in a doc" anti-pattern

Some organisations have policy as a wiki page. Nobody enforces it; reviews discover violations; the policy is treated as advisory.

Corrective: written policies become code; code enforces; the wiki may document for humans but is not the authoritative source.

### 3.7 The integration with code review

Policy changes go through PR review like code:

- Reviewer from policy owner.
- Tests required.
- CI runs policy tests.
- Deployment goes through standard pipeline.

The discipline keeps policy current and quality controlled.

---

## 4. The policy engine choice

The vendor / approach decision.

### 4.1 The options

| Engine | Language | Maturity | Typical fit |
| --- | --- | --- | --- |
| OPA (Open Policy Agent) | Rego | Mature (CNCF graduated) | General-purpose policy; widely supported |
| Cedar (AWS) | Cedar | Mature (AWS-backed; open) | AWS-centric; authorisation-flavoured |
| Custom (in-house engine) | Team's choice | Varies | When existing infrastructure or specific needs |
| Centralised authorization platform | Varies (Permify, Authzed/SpiceDB, etc.) | Mature for specific use cases | When AuthZ is the primary need; policy a side-aspect |

Each has trade-offs.

### 4.2 OPA

The de facto standard for policy-as-code in cloud-native systems.

**Pros.**
- Mature ecosystem; many integrations.
- Rego is powerful; expresses complex policies.
- Bundle pattern for distribution.
- Multiple deployment modes (sidecar, library, central service).

**Cons.**
- Rego has a learning curve.
- Performance at very high QPS may need tuning.

**When right.** Default for most teams without existing investment.

### 4.3 Cedar

AWS's policy language; positioned as more readable than Rego.

**Pros.**
- More approachable syntax for non-policy specialists.
- AWS-native integration.
- Strong type system.

**Cons.**
- Younger ecosystem.
- AWS-centric (though open-source).

**When right.** AWS-centric teams; teams that value readability over Rego's expressiveness.

### 4.4 Custom

Some teams build their own policy engine.

**Pros.**
- Tailored to specific needs.
- No external dependency.

**Cons.**
- Reinvention; ongoing maintenance.
- Missing ecosystem.
- Usually less robust than mature options.

**When right.** Rare. Specific needs that mature engines can't address.

### 4.5 The decision

For most teams: OPA. For AWS-centric teams considering readability: Cedar. Custom only when justified.

The choice is less important than the discipline (policy as code, central evaluation, version control, eval, observability).

### 4.6 The "we'll use what the team already knows" alignment

If the organisation has OPA for Kubernetes admission control, extending to AI policy is natural. If it has Cedar for IAM, same. Reuse existing investment.

### 4.7 The deployment topology

- **Sidecar.** Policy engine runs alongside the application; library makes calls. Low latency; deployment complexity.
- **Centralised service.** Policy engine runs as a service; applications call over network. Easier deployment; higher latency.
- **Library.** Policy engine is a library in the application process. Lowest latency; less flexibility.

The choice depends on QPS and team's deployment patterns.

---

## 5. The policy bundle and deployment

The artefact and pipeline.

### 5.1 The policy bundle

A bundle is the deployable unit of policies:

```
policies/
  ai/
    model_approval.rego
    tool_access.rego
    data_classification.rego
    content_policies.rego
    feature_access.rego
    review_required.rego
  data/
    approved_models.json
    tool_catalogue.json
    tenant_metadata.json
  tests/
    model_approval_test.rego
    tool_access_test.rego
    ...
```

The bundle contains:

- **Policy files.** The Rego / Cedar code.
- **Data files.** Tables the policies reference (model lists, tenant attributes).
- **Tests.** Verify policies behave as expected.

### 5.2 Bundle versioning

The bundle has a version (typically a git commit hash or a semver tag). The policy engine pulls bundles by version; pin-and-deploy pattern.

### 5.3 The build pipeline

```
PR with policy change →
  CI: run policy tests →
  CI: lint Rego/Cedar →
  CI: integration tests (policy + application stub) →
  Bundle build →
  Bundle published (S3, OCI, etc.) →
  Policy engine pulls new bundle →
  Production traffic uses new policies
```

The pipeline is standard CI/CD adapted for policies.

### 5.4 The rollout pattern

Policy changes can be staged:

- **Canary policy deployment.** New policy version applies to a fraction of traffic; metrics monitored.
- **Per-tenant rollout.** Specific tenants get new policy first.
- **Shadow mode.** New policy runs alongside old; results compared without enforcement.

Shadow mode is especially useful for policy changes that might unexpectedly deny — discover impact before enforcement.

### 5.5 The rollback path

Policy issues need fast rollback:

- Bundle versioning supports pin-rollback.
- The policy engine picks up the rolled-back bundle within seconds.
- No application restart needed.

The agility is one of policy-as-code's biggest benefits — bugs in policy don't require code rollback.

### 5.6 The "policy hot reload" capability

The policy engine watches for bundle updates and reloads without restart. This makes policy iteration fast:

- New bundle published → policy engine picks it up → next decision uses new policy.

The capability matters for production agility.

### 5.7 The policy testing

Tests are part of the bundle:

```rego
test_phi_to_baa_model_allowed {
    allow with input as {
        "data_classification": "PHI",
        "model": "anthropic/claude-sonnet-4-6",
        "tenant_baa": true,
    }
}

test_phi_to_non_baa_denied {
    not allow with input as {
        "data_classification": "PHI",
        "model": "openai/gpt-5",
        "tenant_baa": false,
    }
}
```

Tests are run in CI; coverage tracked.

### 5.8 The policy linting

Beyond tests, linting catches stylistic issues and likely bugs:

- Unused rules.
- Always-true / always-false conditions (likely bugs).
- Missing default decisions.
- Naming inconsistency.

OPA has built-in linters; Cedar has its own.

---

## 6. Per-tenant overrides

AI policy varies by tenant. The architecture supports this without code branches.

### 6.1 The override mechanism

The policy engine looks up per-tenant configuration:

```rego
allow {
    input.data_classification == "PHI"
    tenant_config := data.tenant_configs[input.tenant_id]
    input.model in tenant_config.approved_models_for_phi
}
```

The `tenant_configs` data is part of the bundle (or pulled from a database the policy engine consults).

### 6.2 The tenant configuration data

```yaml
tenant_configs:
  uuid-tenant-a:
    name: "Hospital Group A"
    has_baa: true
    approved_models_for_phi:
      - "anthropic/claude-sonnet-4-6"
      - "anthropic/claude-opus-4-7"
    rate_limits:
      per_minute: 200
    review_required_for:
      - care_plan_changes
      - prescription_modifications
  uuid-tenant-b:
    name: "Premium Customer B"
    has_baa: true
    approved_models_for_phi:
      - "anthropic/claude-opus-4-7"  # premium tier — opus only
    rate_limits:
      per_minute: 1000  # higher quota
    review_required_for:
      - care_plan_changes
```

### 6.3 The per-tenant policy update

When a tenant's contract changes (new tier, new compliance), the tenant config updates:

- Config change → PR review → CI tests → bundle build → deployed.
- Per-tenant change in minutes, not days.

### 6.4 The cross-tenant default

Most policies have a default that applies to all tenants; per-tenant overrides apply where present:

```rego
default_config := {
    "approved_models_for_phi": ["anthropic/claude-sonnet-4-6"],
    "rate_limits": {"per_minute": 100},
}

tenant_config[tenant_id] := config {
    config := data.tenant_configs[tenant_id]
} else := default_config
```

Tenants without explicit config get defaults.

### 6.5 The override audit

Per-tenant overrides should be auditable:

- Why does tenant X have a higher rate limit? (Premium contract.)
- Why does tenant Y not require review for care-plan changes? (Doesn't have an MD-approval workflow; accepted risk.)

Each override has a rationale documented somewhere (the tenant's contract, the security review).

### 6.6 The "every tenant has special rules" failure

Some teams create tenant-specific overrides at the drop of a hat. The override table grows to thousands of entries; nobody knows which apply; maintenance becomes impossible.

Mitigation:

- Default-first design: policies cover most tenants with defaults.
- Override approval: each override requires justification.
- Periodic override audit: review and consolidate.

### 6.7 The per-tenant deployment cadence

Some tenant changes are time-sensitive (a contract starts Monday); the deployment must support this:

- Policy engine pulls latest bundle frequently (every few minutes).
- Per-tenant config can be deployed via a separate fast path (data update without bundle rebuild).

---

## 7. Integration with the choke points

Policy is evaluated at the choke points from [guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md).

### 7.1 At the AI gateway

The gateway evaluates policy at:

- Inbound (can this user / tenant call this feature with this input?).
- Pre-call (is this model approved for this data classification?).
- Post-call (does the output require review per content policy?).

The gateway's policy hooks are part of its request pipeline.

### 7.2 At the tool registry

Per [tool-call-authorization.md](./tool-call-authorization.md). The registry evaluates policy at:

- Tool dispatch (is this caller authorised for this tool?).
- Argument validation (are these arguments within policy?).

### 7.3 At the retrieval client

Per [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md). The retrieval client evaluates:

- Source allow-list (is this source authorised for this caller?).
- Scope (is this query within authorised scope?).

### 7.4 At the agent runner

Per the engineering sibling's [agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md). The runner evaluates:

- Budget (is this within authorised cost / time / turn limits?).
- Action approval (does this action require human approval per policy?).
- Escalation (per policy, when does the agent escalate vs proceed?).

### 7.5 The "policy decision context"

Every policy evaluation needs context:

- Identity: who's calling (user, agent, system).
- Resource: what's being accessed (model, tool, data, feature).
- Action: what's being done (call, modify, log, etc.).
- Environment: tenant, region, time, version.

The gateway and registry are responsible for assembling this context and passing to the policy engine.

### 7.6 The observability

Per evaluation:

- Decision (allow / deny).
- Reason.
- Policy name and version.
- Context (sanitised; no PII).
- Latency.

Aggregations:

- Deny rate per policy (high deny rate may indicate a UX problem or policy misalignment).
- Most-evaluated policies.
- Policy evaluation latency p99.

The observability supports debugging "why was this denied" and tuning.

### 7.7 The fail-mode

Policy engine unreachable:

- **Fail closed.** Default for security-critical policies; refuse the request.
- **Fail open.** Acceptable for non-critical; proceed with default behaviour.

The choice per policy. Document; review.

---

## 8. Worked Meridian example

Meridian's policy-as-code practice.

### 8.1 The stack

- **Engine.** OPA (Open Policy Agent), deployed as a service in the Kubernetes cluster.
- **Bundles.** Built from `meridian-policy-bundle` Git repository; published to S3; OPA pulls every 30 seconds.
- **Application integration.** The AI gateway (`meridian-llm-gateway`) and tool registry (`meridian-tools-registry`) call OPA via REST.
- **Observability.** Per-decision spans; per-policy metrics in Grafana.

### 8.2 The policy bundle structure

```
meridian-policy-bundle/
  policies/
    ai/
      model_approval.rego        # which models for which data classifications
      tool_access.rego           # which agents can call which tools
      data_classification.rego   # PHI / PII / public flow rules
      content_policies.rego      # required disclaimers, refusal patterns
      feature_access.rego        # which roles can use which features
      review_required.rego       # what triggers human review
  data/
    approved_models.json         # current model approvals
    tool_catalogue.json          # tool definitions
    tenant_configs.json          # per-tenant overrides
  tests/
    model_approval_test.rego
    tool_access_test.rego
    ...
  build/
    bundle.tar.gz                # built bundle for OPA consumption
```

### 8.3 Sample policies

**Model approval for PHI:**

```rego
package ai.model_approval

default allow := false

allow {
    input.data_classification == "PHI"
    tenant := data.tenant_configs[input.tenant_id]
    input.model in tenant.approved_models_for_phi
}

allow {
    input.data_classification == "general"
    input.model in data.approved_models.general
}

allow {
    input.data_classification == "public"
    # Any approved model
    input.model in data.approved_models.all
}
```

**Content disclaimer for medical advice:**

```rego
package ai.content_policies

requires_medical_disclaimer {
    input.feature == "care-coordinator"
    input.output_classification == "medical_advice"
}

# In application: if requires_medical_disclaimer, append the disclaimer
```

**Review required for care-plan changes:**

```rego
package ai.review_required

review_required {
    input.action == "modify_care_plan"
    input.feature == "care-coordinator"
}

review_required {
    input.action.confidence < 0.7
    input.action.has_side_effect == true
}
```

### 8.4 Tenant configuration

```yaml
tenant_configs:
  hospital-network-a:
    name: "Hospital Network A"
    has_baa: true
    approved_models_for_phi:
      - "anthropic/claude-sonnet-4-6"
      - "anthropic/claude-opus-4-7"
    review_required_for:
      - modify_care_plan
      - prescription_change
    rate_limits:
      per_minute: 200
      per_day: 50000
  premium-hospital-b:
    name: "Premium Hospital B"
    has_baa: true
    approved_models_for_phi:
      - "anthropic/claude-opus-4-7"  # frontier only
    review_required_for:
      - modify_care_plan
      - prescription_change
      - schedule_appointment  # extra strict
    rate_limits:
      per_minute: 1000
      per_day: 200000
```

### 8.5 The deployment cadence

- Policy PRs: ~5-10 per month.
- Bundle builds: per PR merge.
- Bundle deployment: automatic to staging; manual promotion to production after smoke tests.
- Per-tenant config: same pipeline; can be deployed via a separate "config-only" PR path for time-sensitive changes.

Average time from PR merge to production: 30 minutes.

### 8.6 The observability

Per-policy dashboard:

- Decisions per minute (allow / deny breakdown).
- Top denial reasons.
- Per-tenant deny rate.
- Policy evaluation latency.

Quarterly review identifies:

- Policies that deny frequently (UX or alignment issue).
- Policies never evaluated (dead code).
- Per-tenant patterns.

### 8.7 The incidents

**Q2-25.** A new tenant's contract went live with PHI-approved model list including a model the security team hadn't approved. The mistake was caught by CI tests (the test for "PHI requires approved models" failed); the PR was reverted; the tenant onboarding was delayed by a day.

The CI catch prevented a production incident that would have been a HIPAA exposure.

**Q3-25.** A policy update tightened the review-required logic; unintended consequence: a non-modifying agent action (just informational) was flagged as requiring review. Care managers were paged for non-actionable items.

Mitigation: rollback the policy (took 5 minutes via bundle pin-rollback); investigate; fix (the policy condition was too broad); redeploy.

**Q4-25.** Per-tenant config explosion: 18 months in, the tenant_configs.json file had grown to ~80 tenants with ~30 having custom overrides. Audit found 12 overrides were no longer justified (contracts had renewed without the special terms). Consolidation removed 12; the remaining 18 were re-validated.

### 8.8 What worked

- **Centralised evaluation.** All AI decisions consult OPA; no scattered conditionals.
- **CI tests.** Policy changes caught before production.
- **Hot reload.** Policy updates deploy in minutes, not days.
- **Per-tenant overrides.** Contractual variations supported without code changes.

### 8.9 What didn't work initially

- **Policy logic in code.** Early implementations had AI decisions as if-else in the gateway; policies were comments. Migration to OPA took 2 quarters but eliminated drift.
- **Override explosion.** Per-tenant overrides were too easy to add; consolidation now happens quarterly.
- **Fail-open default.** Initial OPA outage caused failures; the team's first instinct was fail-open (proceed without policy). Reverted after the security team flagged the silent-risk implication.

---

## 9. Anti-patterns

### 9.1 "Policy as wiki page"

The policy is documented; not enforced. Reviewers catch violations; production drifts.

**Corrective.** Policy as code per section 3; enforced at choke points.

### 9.2 "Policy in code conditionals"

Each application has its own if-else implementing the policy. Drift between applications.

**Corrective.** Central policy engine per section 4; applications call.

### 9.3 "No policy tests"

Policies ship without verification; bugs go to production.

**Corrective.** Policy tests in CI per section 5.7.

### 9.4 "Fail-open default"

Policy engine down → proceed without enforcement. Silent risk.

**Corrective.** Fail-closed default per section 7.7.

### 9.5 "Per-tenant override explosion"

Every tenant has custom rules; nobody can reason about the policy.

**Corrective.** Default-first design per section 6.6; periodic audit.

### 9.6 "Policy decisions not logged"

Decisions happen; no audit trail; compliance can't verify.

**Corrective.** Per-decision logging per section 7.6.

### 9.7 "Policy bundles don't version"

Bundle updates are unversioned; rollback unclear.

**Corrective.** Versioning per section 5.2; pin-and-deploy.

### 9.8 "Policy evaluation latency unmonitored"

Policy adds latency; users notice; nobody knows.

**Corrective.** Latency monitoring per section 7.6.

---

## 10. Findings (sprint-assignable)

### ARCH-PAC-001 — Severity: Critical
**Finding.** AI policies live as wiki / docs; not enforced.
**Recommendation.** Policy as code per section 3; central engine; choke point integration.
**Owner.** architecture + security, sprint N+1.

### ARCH-PAC-002 — Severity: Critical
**Finding.** Policy decisions implemented in application code conditionals; drift across services.
**Recommendation.** Central policy engine per section 4.
**Owner.** ai-platform-eng + security, sprint N+1.

### ARCH-PAC-003 — Severity: Critical
**Finding.** Policy decisions not logged; compliance audit cannot verify enforcement.
**Recommendation.** Per-decision logging per section 7.6.
**Owner.** ai-platform-eng + compliance, sprint N+1.

### ARCH-PAC-004 — Severity: High
**Finding.** Policy engine fail-open default; silent risk during outages.
**Recommendation.** Fail-closed for security-critical per section 7.7.
**Owner.** ai-platform-eng + security, sprint N+2.

### ARCH-PAC-005 — Severity: High
**Finding.** Policy tests absent; bugs reach production.
**Recommendation.** Policy tests in CI per section 5.7.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-PAC-006 — Severity: High
**Finding.** Per-tenant overrides without governance; explosion looms.
**Recommendation.** Default-first design per section 6.6; override approval.
**Owner.** architecture + product, sprint N+2.

### ARCH-PAC-007 — Severity: High
**Finding.** Policy ownership unclear; cross-team policies stall.
**Recommendation.** Per-category owners per section 2.7.
**Owner.** architecture + leadership, sprint N+2.

### ARCH-PAC-008 — Severity: Medium
**Finding.** Bundle versioning informal; rollback unclear.
**Recommendation.** Versioned bundles per section 5.2; rollback runbook.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-PAC-009 — Severity: Medium
**Finding.** Policy evaluation latency not measured; user-perceptible delays unattributed.
**Recommendation.** Latency monitoring per section 7.6.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-PAC-010 — Severity: Medium
**Finding.** Policy bundle deployment manual; takes hours; agility lost.
**Recommendation.** Automated pipeline per section 5.3; hot reload.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-PAC-011 — Severity: Medium
**Finding.** Policy engine choice ad-hoc; team would benefit from deliberate decision.
**Recommendation.** Engine evaluation per section 4; document.
**Owner.** architecture, sprint N+3.

### ARCH-PAC-012 — Severity: Medium
**Finding.** Tenant config audit not scheduled; overrides drift from justification.
**Recommendation.** Quarterly audit per section 6.5 / 6.6.
**Owner.** architecture + product, sprint N+3.

### ARCH-PAC-013 — Severity: Medium
**Finding.** Policy decisions returned only allow/deny; rich obligations unsupported.
**Recommendation.** Rich decision shape per section 3.3.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-PAC-014 — Severity: Medium
**Finding.** Shadow mode not used for policy changes; impact discovered in production.
**Recommendation.** Shadow rollout per section 5.4 for risky policies.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-PAC-015 — Severity: Low
**Finding.** Policy linting not configured; stylistic and likely-bug issues miss.
**Recommendation.** Linting per section 5.8.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-PAC-016 — Severity: Low
**Finding.** Cross-team policy coordination ad-hoc; new policies fall through gaps.
**Recommendation.** Cross-team policy guild; cadence.
**Owner.** architecture, sprint N+5.

### ARCH-PAC-017 — Severity: Low
**Finding.** Policy engine deployment topology not optimised; latency higher than needed.
**Recommendation.** Topology choice per section 4.7.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-PAC-018 — Severity: Low
**Finding.** Policy documentation thin; new engineers don't know what exists.
**Recommendation.** Policy catalog; per-policy documentation.
**Owner.** ai-platform-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building policy as code:

- [ ] **Sprint 0 — policy inventory.** What policies exist today (in wikis, code, heads)?
- [ ] **Sprint 0 — owner identification.** Per category per section 2.7.
- [ ] **Sprint 0 — engine choice.** OPA, Cedar, custom per section 4.
- [ ] **Sprint 1 — engine deployment.** Per chosen topology.
- [ ] **Sprint 1 — first policies.** Migrate the highest-stakes policies from code/wiki.
- [ ] **Sprint 1 — choke point integration.** Gateway, tool registry call the engine.
- [ ] **Sprint 2 — tests.** Per policy.
- [ ] **Sprint 2 — observability.** Per-decision logging; per-policy metrics.
- [ ] **Sprint 3 — bundle pipeline.** CI/CD for policies.
- [ ] **Sprint 3 — hot reload.** Engine picks up updates within minutes.
- [ ] **Sprint 4 — per-tenant overrides.** Configuration as data.
- [ ] **Sprint 4 — shadow mode.** For risky policy changes.
- [ ] **Sprint 5 — audit.** Decisions queryable for compliance.
- [ ] **Ongoing — quarterly review.** Per-policy quality; tenant override audit; bundle health.

For a team retrofitting:

- [ ] **Sprint 0 — audit.** Where do policies live today?
- [ ] **Sprint 1 — engine adoption.** Pick one; deploy.
- [ ] **Sprint 2 — migrate highest-stakes first.** Demonstrate value.
- [ ] **Sprint 3 — gradual migration.** Other policies one at a time.
- [ ] **Sprint 4 — sunset old paths.** Application code stops implementing policy.

A team that completes the sequence has policy that's enforceable, versioned, observable, and tenant-aware. A team that doesn't has policy as advice with predictable drift.

---

## 12. References

- [guardrail-placement-decision-framework.md](./guardrail-placement-decision-framework.md) — choke points where policy is evaluated.
- [ai-gateway-pattern.md](./ai-gateway-pattern.md) — gateway's role in policy evaluation.
- [tool-call-authorization.md](./tool-call-authorization.md) — tool registry's role.
- [retrieval-scope-enforcement.md](./retrieval-scope-enforcement.md) — retrieval client's role.
- [content-moderation-architecture.md](./content-moderation-architecture.md) — content policies overlap.
- [refusal-and-escalation-design.md](./refusal-and-escalation-design.md) — refusal patterns from policy.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant considerations.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-loop-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-loop-design.md) — agent runner as policy evaluation point.
- Sibling repo: [ai-engineering-reference-architecture/cicd-and-eval-gates/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cicd-and-eval-gates/) — eval gates that consume policy decisions.
- Sibling repo: ai-security-reference-architecture — threat-driven policies.
- OPA (Open Policy Agent), CNCF graduated project documentation.
- AWS Cedar policy language documentation.
- Permify, SpiceDB, Authzed — alternative authorization platforms.
- "Policy as Code" (Pulumi, HashiCorp, others) — broader industry practice.
