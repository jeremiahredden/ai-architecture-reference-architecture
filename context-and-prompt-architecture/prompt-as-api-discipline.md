# Prompt-as-API Discipline

> **Audience.** Architects whose prompt changes have broken downstream code (extractors, chains, evals) without warning. Tech leads whose "we updated the prompt; sorry about the inconvenience" Slack message produces multiple urgent fires. Anyone whose prompt versioning is currently git history. **Scope.** The *architectural* discipline of treating prompts as versioned APIs: pinning versions in releases; deprecation lifecycles; breaking-change communication; prompt-changelog patterns; backwards-compatible evolution; integration with the engineering sibling. Not the prompt-engineering content (see sibling [prompt-engineering/prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md)). Not the system prompt design (see [system-prompt-architecture.md](./system-prompt-architecture.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Prompts have consumers. The model is one. Often forgotten: downstream parsers, structured-output extractors, eval suites, integration tests, customer-facing UIs that depend on response shape.

A prompt change can break any of these. The classic failure mode:

- Engineer updates the system prompt to "respond with more detail."
- Downstream extractor expected a specific JSON shape.
- New prompt produces different JSON; extractor breaks.
- Production silently fails for hours until a customer notices.

The problem isn't the prompt change. The problem is that the prompt is treated as a string that can be edited at will, when its consumers depend on its behavior as a contract.

Treating prompts as APIs:

- Version explicitly.
- Pin versions per release.
- Deprecate with lifecycle.
- Communicate breaking changes.

Same disciplines that apply to REST APIs apply here. This document covers the architectural framing.

This document is opinionated about four things:

1. **A prompt is a versioned API, not a string.** Consumers depend on its behavior; that behavior must be stable or evolve with notice.
2. **Breaking changes get deprecation cycles.** Don't ship a breaking change without giving consumers time to adapt.
3. **The prompt's API surface is its output behavior, not its input format.** Internal prompt text can change; output behavior shouldn't change unexpectedly.
4. **Eval is the contract test.** Eval cases test prompt behavior; eval pass is the gate for prompt changes.

Structure: (2) what a prompt's API surface actually is; (3) versioning patterns; (4) pinning per release; (5) deprecation lifecycle; (6) breaking-change communication; (7) backwards-compatible evolution; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. What a prompt's API surface actually is

Before versioning, understand what's being versioned.

### 2.1 The behavioral contract

The prompt's API surface is the *behavior* it produces:

- **Output format.** JSON shape; markdown structure; prose style.
- **Output content.** Topics covered; level of detail; refusal patterns.
- **Tool-use behavior.** Which tools chosen; how called.
- **Latency profile.** How long responses take (related to prompt length / complexity).

When any of these changes meaningfully, the API has changed.

### 2.2 The non-API internals

What's NOT part of the API:

- Exact wording of system prompt (can be paraphrased).
- Internal comments / structure.
- Component-internal organization.

These can change freely as long as behavior is preserved.

### 2.3 The "behavior change is API change" rule

If a prompt update changes:

- The output JSON structure → API change.
- The list of topics covered → API change.
- The refusal patterns → API change.
- The citation format → API change.

These are consumer-facing; consumers depend on them.

### 2.4 The "internal change is not API change" rule

If a prompt update:

- Reorganizes the system prompt sections.
- Rewords instructions for clarity.
- Adds comments / explanation.
- Adjusts the wording of refusal policy without changing what's refused.

Same behavior; not API-visible.

### 2.5 The eval-as-contract

Eval cases test behavior:

- Pass: behavior consistent with contract.
- Fail: behavior changed.

If eval passes, the change is internal-only.
If eval fails, the change is behavioral.

Cross-link to [system-prompt-architecture.md §6](./system-prompt-architecture.md).

### 2.6 The consumer catalog

Who depends on the prompt's behavior:

- Downstream parsers (structured output extractors).
- Eval suites.
- Integration tests.
- UI components that display specific fields.
- Customer applications that consume the response.

List the consumers; understand what they depend on.

### 2.7 The "we have no consumers; we can change freely" claim

Even without external consumers, internal consumers (eval, UI) exist.

Even minimal prompts have eval suites that depend on behavior.

The API discipline applies everywhere.

---

## 3. Versioning patterns

How to version.

### 3.1 The semver mapping

Borrow from semver:

- **Major version.** Breaking change to behavior or output.
- **Minor version.** Backwards-compatible addition (new optional fields; new behaviors that consumers can ignore).
- **Patch version.** Internal change (refactor, clarity improvement); behavior preserved.

```
v1.0.0: initial prompt
v1.0.1: rewording for clarity (patch)
v1.1.0: added optional citation format (minor)
v2.0.0: changed JSON shape (major; breaking)
```

### 3.2 The version-in-catalogue

Cross-link to [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md):

Each prompt has a catalogue entry with version:

```yaml
prompt: care-coordinator-v2.3.0
status: in_production
model_compatibility: [claude-sonnet-4-6, claude-sonnet-4-7]
breaking_change_log:
  - v2.0.0: 2026-03-15: changed JSON shape from {summary, actions} to {summary, key_findings, suggested_actions, citations}
```

Catalogue-driven; auditable.

### 3.3 The component-versioning vs assembled-versioning

For decomposed prompts (cross-link to [system-prompt-architecture.md §4.3](./system-prompt-architecture.md)):

- Each component versioned independently.
- Assembled prompt is a function of component versions.

```yaml
care-coordinator-prompt-v2.3.0:
  components:
    platform_base@v12
    feature_overlay/care-coordinator@v8
    refusal_policy@v5
    safety_constraints@v7
```

If a component changes, the assembled prompt's minor/patch version reflects.

### 3.4 The breaking-change determination

Is this change breaking?

- Run eval against new prompt.
- Compare to eval against old prompt.
- If pass-rate dropped > X%: probably breaking.
- If output structure changed: definitely breaking.
- If new optional fields added: minor (backwards-compatible).

### 3.5 The "we just patched the prompt" reality

Most prompt updates are patch:

- Reword for clarity.
- Add example for a specific case.
- Fix typos.

Internal-only; no API impact.

### 3.6 The version-pinning per consumer

Each consumer pins:

```yaml
care-coordinator-feature:
  pinned_prompt_version: care-coordinator-v2.3.0
  
analytics-extractor:
  expects_prompt_version: care-coordinator-v2.x
```

Pinning means the consumer knows what they're getting.

### 3.7 The version-evolution discipline

Patch versions: no announcement needed.
Minor versions: announcement to consumers; opt-in.
Major versions: deprecation cycle.

Each version's scope informs the workflow.

---

## 4. Pinning per release

How versions are deployed.

### 4.1 The release model

A release of the AI feature pins:

- Model version.
- Prompt version.
- Eval suite version.
- Configuration.

Pinned together; deployed atomically.

### 4.2 The "feature_X uses prompt_Y" manifest

```yaml
care-coordinator-release:
  version: 14.2.1
  model: anthropic:claude-sonnet:4-6
  prompt: care-coordinator-v2.3.0
  eval_suite: care-coordinator-eval-v1.8.0
  deployed_at: 2026-05-10
```

Manifest reviewed at release.

### 4.3 The "we rolled forward" discipline

Releases don't downgrade without explicit intent:

- New release pins to newer prompt version.
- Rollback (if needed) is to an explicit previous release.

### 4.4 The "the prompt was updated but the feature wasn't redeployed" trap

If prompt is updated independently of the release:

- Behavior changes without the release.
- Eval may not have run for this combination.

**Corrective.** Prompt changes shipped via release; not via direct config edit.

### 4.5 The canary release

Deploying a new prompt:

- Canary 5% of traffic; new prompt version.
- 95% on previous.
- Monitor quality / cost / latency.
- Gradual ramp.

Cross-link to [model-strategy/model-migration-playbook.md §7](../model-strategy/model-migration-playbook.md).

### 4.6 The rollback

Rollback = revert to previous release's manifest:

- Model, prompt, eval all snap back together.
- Verified to work (it was working before).

Don't roll back just the prompt; roll back the release.

### 4.7 The "we have a config flag for prompt version" pattern

Some platforms use feature flags:

- Flag value determines which prompt version.
- Per-user / per-tenant flag.
- Allows A/B testing.

Cross-link to [ai-engineering-reference-architecture / prompt-engineering / prompt-ab-testing.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-ab-testing.md).

---

## 5. Deprecation lifecycle

How a breaking change ships.

### 5.1 The five phases

```
Phase 1: Announcement (deprecation announced; new version not yet shipped).
Phase 2: Coexistence (both versions in production; consumers migrate).
Phase 3: Default switch (new version is default; old still available).
Phase 4: Removal warning (announce removal date).
Phase 5: Removed (old version no longer available).
```

Each phase has a duration; total typically 1-3 months.

### 5.2 Phase 1: Announcement

Notify consumers:

- Internal teams.
- Customer-facing integrations.
- Documentation update.

Provide migration guide.

### 5.3 Phase 2: Coexistence

Both versions accessible:

```
care-coordinator-v1.5.0 (old; deprecated)
care-coordinator-v2.0.0 (new; recommended)
```

Consumers can pin either. Migrate at their pace.

### 5.4 Phase 3: Default switch

Default changes to new version:

- New consumers get new version by default.
- Existing pinned to old continue.
- Documentation reflects.

### 5.5 Phase 4: Removal warning

Announce removal date (e.g., 30 days).

Final reminder to consumers still on old version.

### 5.6 Phase 5: Removed

Old version no longer available:

- Catalogue entry marked "removed."
- Consumer attempts to use → error.

### 5.7 The "we accidentally broke compatibility" recovery

Sometimes a "patch" turns out to be breaking:

- Production behavior changes.
- Consumer issues surface.

Recovery:

- Roll back to previous version.
- Investigate.
- Reclassify as major version (with proper deprecation).
- Re-ship with discipline.

The fix isn't the prompt; it's the discipline.

### 5.8 The minimum deprecation period

Per organization:

- 30 days for low-impact changes.
- 60-90 days for high-impact.

Document the policy.

### 5.9 The "we have no choice; provider deprecated model" exception

Sometimes external forces require breaking changes faster:

- Provider deprecates model.
- Workflow forced to migrate.

Accelerated cycle; less ideal but unavoidable.

Cross-link to [model-strategy/model-migration-playbook.md](../model-strategy/model-migration-playbook.md).

---

## 6. Breaking-change communication

How to communicate.

### 6.1 The prompt-changelog

```markdown
# Care Coordinator Prompt Changelog

## v2.0.0 - 2026-04-15 [BREAKING]
- Changed JSON output shape from {summary, actions} to {summary, key_findings, suggested_actions, citations}.
- New required field: citations.
- Migration guide: [link]

## v1.5.0 - 2026-02-10
- Added optional field "confidence" in output.

## v1.4.2 - 2026-01-20
- Internal: clarified refusal policy wording. Behavior preserved.

## v1.4.0 - 2026-01-05
- Added new behavior for handling ambiguous patient context (improved disambiguation).
```

Maintained per prompt; visible to consumers.

### 6.2 The migration guide

For breaking changes:

```markdown
# Migration from Care Coordinator v1.x to v2.0.0

## What's Changing
- Output JSON shape...

## How to Migrate
1. Update your consumer to expect new shape...
2. ...

## Timeline
- 2026-04-15: v2.0.0 released; v1.x deprecated
- 2026-05-15: v2.0.0 becomes default
- 2026-07-15: v1.x removed

## Help
- Slack #ai-platform
- Email: ai-platform@meridian.example.com
```

Per breaking change; published.

### 6.3 The customer communication

For changes affecting customer-facing API:

- 30+ days advance notice.
- Customer success / account managers involved.
- Email + dashboard banner + release notes.

### 6.4 The internal communication

For internal-only changes:

- #ai-platform Slack announcement.
- Engineering all-hands or weekly digest.
- Wiki update.

### 6.5 The post-change support

After ship:

- Monitor for consumer issues.
- Quick-response team for migration help.
- Office-hours for consumers.

### 6.6 The "we communicated but nobody noticed" anti-pattern

Communication that's ignored:

- Posted in a channel nobody reads.
- Email going to spam.
- Wiki update nobody knows about.

Mitigation:

- Multi-channel.
- Acknowledgment required for major changes.
- Follow-up before deprecation.

### 6.7 The customer-facing API consideration

If customers use the API directly (rare but real):

- SLA terms determine deprecation period.
- Customer success involved.
- Versioned API endpoints (some platforms).

---

## 7. Backwards-compatible evolution

Patterns for evolution without breaking.

### 7.1 Additive-only changes

Add fields; don't remove:

```yaml
# v1.0.0
output: {summary, actions}

# v1.1.0 (backwards-compatible)
output: {summary, actions, confidence}  # added confidence

# v1.2.0 (backwards-compatible)
output: {summary, actions, confidence, citations}  # added citations
```

Existing consumers ignore new fields; new consumers use them.

### 7.2 Optional fields

New fields are optional:

- Model may include or not.
- Consumer code: `confidence = response.get("confidence", default)`.

Backwards-compatible.

### 7.3 Deprecation marker for fields

When a field will be removed:

```yaml
output:
  summary: ...
  actions: ...
  confidence: ...  # deprecated; use new_confidence instead. Removal: 2026-06-15.
  new_confidence: ...  # new field; preferred
```

Coexist briefly; sunset old.

### 7.4 The "version in response" pattern

Output includes version:

```json
{
  "_prompt_version": "v2.3.0",
  "summary": "...",
  "actions": [...]
}
```

Consumer can adapt based on version.

### 7.5 The renaming-without-breaking

If a field needs renaming:

- Phase 1: include both `old_name` and `new_name`.
- Phase 2: prefer `new_name` in documentation.
- Phase 3: remove `old_name`.

Same as semver for libraries.

### 7.6 The behavior-change-without-shape-change

Sometimes content changes but shape doesn't:

- The `summary` field still exists.
- But what's in it differs (more detailed; different phrasing).

This is behaviorally-breaking even though structurally-compatible.

**Corrective.** Treat as breaking; deprecation cycle.

### 7.7 The eval-pinning

Eval suite pins to prompt version:

- Eval suite is part of the prompt's contract.
- Changes to the eval should track prompt versioning.

### 7.8 The "we made a small change but tests passed" reassurance

Tests passed means: contract preserved.

If tests fail: contract changed; manage as breaking.

### 7.9 The breakdown-of-fields evolution

For complex output shapes:

```yaml
v1.0.0: {summary, actions: ["..."]}
v1.1.0: {summary, actions: ["..."], detailed_actions: [{action, rationale, urgency}]}
v2.0.0: {summary, detailed_actions: [...]}  # removed plain actions; breaking
```

Two minor versions establish the new field; one major version removes the old.

---

## 8. Worked Meridian example

Meridian's prompt-as-API discipline.

### 8.1 The version catalog

```yaml
care-coordinator-prompt:
  current: v2.3.0
  status: in_production
  components:
    platform_base@v12
    feature_overlay/care-coordinator@v8
    refusal_policy@v5
    safety_constraints@v7
  output_shape:
    summary: string
    key_findings: list[string]
    suggested_actions: list[string]
    citations: list[citation]
  consumers:
    - care-coordinator-ui
    - care-coordinator-extractor-service
    - care-coordinator-eval-suite
    - audit-log-pipeline

patient-api-chat-prompt:
  current: v1.5.0
  status: in_production
  ...
```

Tracked; consumers visible.

### 8.2 The Q1 2026 prompt-bloat incident as a discipline failure

(Cross-link to [cost-incident-runbook.md §9](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md).)

The prompt-bloat incident was a patch-classified-as-major issue:

- Engineer thought: minor / patch.
- Reality: behavior change (more verbose; cost-per-call doubled).
- No deprecation cycle.
- Production impact.

After incident: prompt-change classification process tightened. Any change requires:

- Eval pass.
- Classification (patch / minor / major).
- Owner approval.

### 8.3 The Q2 2026 breaking change to Care Coordinator

Decision: improve Care Coordinator output structure (add citations).

Process:

1. **Announcement (week 1):** new version v2.0.0 announced; v1.x deprecated.
2. **Coexistence (weeks 2-5):** both versions available; consumers migrate.
3. **Default switch (week 5):** new version is default; v1.x still pinnable.
4. **Removal warning (week 7):** v1.x to be removed in 2 weeks.
5. **Removed (week 9):** v1.x removed from catalogue.

Total: 9 weeks from announcement to removal.

### 8.4 The consumer migration

Consumers migrated:

- care-coordinator-ui: updated to expect new shape; deployed week 3.
- care-coordinator-extractor-service: updated; deployed week 4.
- care-coordinator-eval-suite: updated to test new shape; deployed week 5.
- audit-log-pipeline: backwards-compatible; updated to use new field; deployed week 6.

All consumers migrated by week 7. No consumer was caught unprepared.

### 8.5 The communication

Internal:
- #ai-platform Slack post on day 0.
- Engineering weekly digest on weeks 0, 4, 7.
- Doc updated.

External (customer-facing for premium tenants with API access):
- Email 60 days in advance.
- Dashboard banner.
- Migration guide provided.

Premium tenants migrated on their own timeline; all done by week 7.

### 8.6 The avoided incident

If the breaking change had shipped without the discipline:

- Estimated 5-10 consumer breakages.
- Multiple 30-min to 2-hour incidents.
- Customer trust damage.

The discipline prevented all of these.

### 8.7 The cost of discipline

- Per breaking change: ~3 hours overhead (announcement, doc, communication).
- Per minor / patch: minimal overhead.
- Annual: ~2-4 breaking changes; ~12 hours overhead.

Justified by avoided incidents.

### 8.8 The "we caught a silent breaking change" example

A prompt update added a new optional field. Engineer:

- Tested locally; passed.
- Submitted PR.
- Code review: pointed out the new field changes some response shapes.
- Caught as minor (backwards-compatible additional field).

Without the prompt-as-API discipline, the engineer might have shipped as patch; consumer might have been confused later.

### 8.9 The cross-team norm

Prompt-as-API is documented in Meridian's engineering wiki:

- All teams aware.
- New engineers onboarded.
- Annual refresh.

Cultural norm; not just process.

### 8.10 The lessons

- Treating prompts as APIs is engineering discipline.
- Communication is the load-bearing part; not the versioning system.
- Coexistence period is essential for graceful migration.
- Eval as contract test is the gate.

---

## 9. Anti-patterns

### 9.1 The "patch is whatever we want it to be" laziness

**Pattern.** Engineer classifies all changes as patch (no impact analysis). Breaking changes ship as patch; consumers break.

**Corrective.** Classification per §3.4; eval evaluates.

### 9.2 The undocumented prompt change

**Pattern.** Engineer edits prompt; pushes; no documentation; no announcement.

**Corrective.** Prompt-changelog per §6.1.

### 9.3 The "tests passed; behavior didn't change" assumption

**Pattern.** Eval passed; engineer ships. But the eval coverage was insufficient; production breaks.

**Corrective.** Eval suite covers contract dimensions per §2.5.

### 9.4 The unilateral breaking change

**Pattern.** Engineer makes breaking change; ships; consumers find out by breakage.

**Corrective.** Deprecation lifecycle per §5.

### 9.5 The no-coexistence-period

**Pattern.** Old version removed when new ships. No migration window.

**Corrective.** Phase 2 (coexistence) per §5.3.

### 9.6 The version-soup-in-production

**Pattern.** Many versions in production simultaneously; nobody knows which is correct.

**Corrective.** Versioned catalog per §3.2; lifecycle states (current / deprecated / removed) tracked.

### 9.7 The "we change the prompt frequently" wisdom

**Pattern.** Prompt updated weekly; consumers can't keep up; instability.

**Corrective.** Versioned releases per §4; cadence aligned with consumer adaptability.

### 9.8 The communication-in-Slack-only

**Pattern.** Announcement in #ai-platform; nobody reads. Breaking change surprise.

**Corrective.** Multi-channel per §6.6.

### 9.9 The customer-facing prompt without API contract

**Pattern.** Customer API exposed; behavior changes without notice; customer integration breaks.

**Corrective.** API discipline applies; customers consulted before breaking changes.

### 9.10 The "we'll formalize versioning later" deferral

**Pattern.** Move fast; "versioning is overhead." First major break is the trigger.

**Corrective.** Versioning from day one; minimal overhead with discipline.

---

## 10. Findings (sprint-assignable)

**ARCH-PAD-001 (P0). Prompts not versioned.** Changes ship as strings; consumers break. Adopt semver per §3. Owner: AI platform.

**ARCH-PAD-002 (P0). Breaking changes ship without deprecation cycle.** Consumer surprise. Per §5. Owner: AI platform + engineering management.

**ARCH-PAD-003 (P0). No prompt-changelog.** Consumer can't track changes. Per §6.1. Owner: AI platform.

**ARCH-PAD-004 (P1). Consumer catalog not maintained.** Unknown who depends. Per §2.6. Owner: AI platform.

**ARCH-PAD-005 (P1). Version-pinning per release absent.** Releases mix old / new. Per §4. Owner: AI platform.

**ARCH-PAD-006 (P1). Migration guides not produced for breaking changes.** Consumers improvise migration. Per §6.2. Owner: AI platform.

**ARCH-PAD-007 (P1). Communication channels for prompt changes single.** Notifications missed. Per §6.6. Owner: AI platform + engineering management.

**ARCH-PAD-008 (P1). Eval suite not pinned per prompt version.** Eval test fails for valid reasons. Per §7.7. Owner: AI platform + eval.

**ARCH-PAD-009 (P2). Customer-facing breaking changes without customer-success involvement.** Customer churn risk. Per §6.3. Owner: customer success + AI platform.

**ARCH-PAD-010 (P2). Coexistence period too short.** Consumers can't migrate. Per §5.8. Owner: engineering management.

**ARCH-PAD-011 (P2). Version-in-response not used.** Consumer can't adapt to version. Per §7.4. Owner: AI platform.

**ARCH-PAD-012 (P2). Component-level versioning absent.** Composed prompts harder to track. Per §3.3. Owner: AI platform.

**ARCH-PAD-013 (P2). Behavior-change-without-shape-change classified as patch.** Silent break. Per §7.6. Owner: AI platform.

**ARCH-PAD-014 (P2). No post-change support window.** Consumers struggle without help. Per §6.5. Owner: AI platform + customer success.

**ARCH-PAD-015 (P3). Annual prompt API audit absent.** Drift accumulates. Owner: AI platform.

**ARCH-PAD-016 (P3). Per-tenant prompt overrides not versioned.** Hard to roll back per tenant. Per [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md). Owner: AI platform.

**ARCH-PAD-017 (P3). Prompt-changelog not in source-control.** Changelog drifts. Per §6.1. Owner: AI platform.

**ARCH-PAD-018 (P3). Cross-team education on prompt-as-API absent.** New engineers unfamiliar. Per §8.9. Owner: engineering management.

---

## 11. Adoption sequencing checklist

- [ ] **Adopt semver for prompts (§3).**
- [ ] **Build prompt catalog (§3.2).**
- [ ] **Maintain consumer catalog per prompt (§2.6).**
- [ ] **Implement release-pinning (§4).**
- [ ] **Define deprecation lifecycle (§5).**
- [ ] **Prompt-changelog per prompt (§6.1).**
- [ ] **Migration guide template (§6.2).**
- [ ] **Multi-channel communication (§6.6).**
- [ ] **Eval suite pinned per prompt version (§7.7).**
- [ ] **Annual cross-team education on prompt-as-API.**

---

## 12. References

**In this folder.**
- [system-prompt-architecture.md](./system-prompt-architecture.md) — system prompt versioning.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — assembly is part of the contract.
- [context-window-budgeting.md](./context-window-budgeting.md) — context budget version.
- [chat-history-architecture.md](./chat-history-architecture.md) — history affects prompt contract.
- [few-shot-vs-fine-tune-vs-system-prompt.md](./few-shot-vs-fine-tune-vs-system-prompt.md) — alternative approaches with similar discipline.

**Elsewhere in this repo.**
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue tracks prompt versions.
- [model-strategy/model-migration-playbook.md](../model-strategy/model-migration-playbook.md) — model migrations as releases.

**Sibling repos.**
- [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) — engineering-side versioning.
- [ai-engineering-reference-architecture / prompt-engineering / prompts-as-code-discipline.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompts-as-code-discipline.md) — engineering discipline.
- [ai-engineering-reference-architecture / prompt-engineering / prompt-as-api-contract.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-as-api-contract.md) — engineering API contract.
- [ai-engineering-reference-architecture / cicd-and-eval-gates / prompt-version-pinning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cicd-and-eval-gates/prompt-version-pinning.md) — release-time pinning.

**External.**
- semver.org — versioning specification.
- Stripe API versioning documentation — model for prompt API versioning.
- GitHub REST API versioning — similar reference.
