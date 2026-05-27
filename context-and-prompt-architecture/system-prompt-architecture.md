# System Prompt Architecture

> **Audience.** Architects whose system prompt has grown to 4000+ tokens of accumulated edge-case patches and they can no longer reason about its behavior. Tech leads whose multi-feature platform has 5 different system prompts that mostly say the same thing in slightly different ways. Anyone whose "system prompt" was originally one paragraph and is now an undocumented monolith. **Scope.** The *architectural* design of the system prompt: its function as identity / behavior / refusal-policy / formatting / tool-use contract; the decomposition pattern (platform base + feature overlay + tenant overlay); prompts-as-components model; testing patterns; the migration path from monolithic to composed. Not the prompt-engineering tradecraft itself (see sibling [prompt-engineering/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/prompt-engineering)). Not the multi-tenant prompt mechanics (see [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The system prompt is the load-bearing primitive of every AI feature. It defines the model's identity, its behavior, its refusal policy, its output format, and its tool-use rules. It's the first thing every call sees. It's also, in most production systems, an undocumented monolith that's grown for two years.

The pattern: the first version is 200 tokens, written by the engineer who shipped the feature. Six months later, a customer complaint produces 300 more tokens patching a specific case. A year later, another patch. Two years later, it's 4500 tokens of overlapping instructions, contradictory edge cases, dead instructions for features that no longer exist, and a "do not do X" section that's six paragraphs long.

The cost of the monolith:

- **Cognitive cost.** Engineering can't reason about what the prompt does without reading it cover to cover.
- **Token cost.** Every call pays for the full prompt; longer prompts are more expensive (cross-link to [cost-and-performance-architecture/token-economics.md](../cost-and-performance-architecture/token-economics.md)).
- **Quality cost.** Conflicting or redundant instructions degrade model behavior.
- **Maintainability cost.** Edits are scary; nobody knows what depends on which line.
- **Evolution cost.** Adding a new feature requires modifying the global prompt; small changes affect everything.

The architectural alternative: treat the system prompt as a structured artifact with components, owners, and discipline. The same way an organization treats a shared codebase.

This document is opinionated about four things:

1. **The system prompt is architecture, not a string.** It has structure (identity, behavior, format, tools); it has components; it has owners. Treat it that way.
2. **Decompose to platform-base + feature-overlay + tenant-overlay.** Three layers; clear ownership of each; bounded changes.
3. **Test the system prompt as you would code.** Regression tests; eval coverage per overlay; change review.
4. **The system prompt is read more than it's written; optimize for readability.** Structure for human comprehension; comments where the model doesn't need them; sections with clear titles.

Structure: (2) the system prompt's six functions; (3) the decomposition pattern; (4) prompts-as-components; (5) per-tenant overlays; (6) testing and review; (7) the migration from monolith; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The system prompt's six functions

What the system prompt actually does.

### 2.1 Identity

Who the model is acting as:

```
You are the Meridian Care Coordinator, a clinical AI assistant
helping clinicians manage patient care plans.
```

Function: orients the model's response style and content.

### 2.2 Behavior

How it behaves:

```
You respond with clinical precision. You ask clarifying questions
when patient data is ambiguous. You defer to the clinician's
judgment for diagnostic decisions.
```

Function: shapes the response posture; informs how the model interprets requests.

### 2.3 Refusal policy

What it won't do:

```
You do not provide:
- Diagnostic determinations (defer to clinician)
- Drug-interaction checks beyond formulary references
- Treatment recommendations outside of established protocols
```

Function: explicit refusal boundary; safety.

### 2.4 Formatting contract

How output is structured:

```
Respond in this format:
- Summary (2-3 sentences)
- Key clinical considerations (bulleted)
- Suggested next actions (numbered)
- Citations [source: ...]
```

Function: structured output; downstream parseable.

### 2.5 Tool-use policy

When and how to call tools:

```
Available tools:
- fetch_patient_record: retrieves current patient data
- search_clinical_notes: searches historical notes
- check_formulary: verifies medication availability

Use fetch_patient_record before referencing specific patient data.
Use search_clinical_notes when historical context is needed.
Cite specific notes via search_clinical_notes results.
```

Function: directs tool use; influences agent behavior.

### 2.6 Reasoning style

How the model thinks:

```
For complex care decisions, structure your reasoning:
1. State the clinical question.
2. List relevant patient context.
3. Evaluate options.
4. Recommend.
5. Note caveats.
```

Function: shapes chain-of-thought; improves consistency.

### 2.7 The six functions composed

A complete system prompt has all six (some implicit):

```
[Identity] You are the Meridian Care Coordinator...

[Behavior] You respond with clinical precision...

[Refusal] You do not provide diagnostic determinations...

[Format] Respond in this format: Summary; Key considerations; Actions; Citations.

[Tools] Available tools: fetch_patient_record; search_clinical_notes; ...

[Reasoning] For complex decisions, structure your reasoning...
```

Each section is a clear function; each can be evolved independently.

### 2.8 The "no section per function" anti-pattern

Some prompts blend the six:

```
You are the care coordinator. Use the tools when they help.
Format as bullet points usually. Be precise but not too verbose.
Don't make diagnoses. If a tool isn't there don't call it.
```

Compressed; unclear; hard to modify any specific function.

Decompose to sections per function. Clarity.

### 2.9 The "section header" practice

Sections marked with clear titles:

```markdown
## Identity
...

## Behavior
...

## Refusal Policy
...
```

Models respect structure (most provide best results with structured prompts). Engineers can scan.

Trade-off: more tokens in the prompt. Net benefit: usually worth it.

---

## 3. The decomposition pattern

The architecture that scales.

### 3.1 The three layers

```
Platform base prompt (shared across all features)
                ↓
Feature overlay (specific to the feature)
                ↓
Tenant overlay (specific to the customer; optional)
                ↓
[Per-request context: retrieved docs, conversation history, user message]
```

Each layer adds; total is the assembled prompt.

### 3.2 The platform base

Shared across all features:

- Platform identity ("You are an AI assistant for Meridian Health").
- Common refusal policy (clinical disclaimers, no PII storage, no off-topic).
- Platform-wide formatting conventions.
- Safety constraints.

~500-1500 tokens; stable; rarely changes.

### 3.3 The feature overlay

Per feature:

- Feature identity ("As the Care Coordinator").
- Feature-specific behavior.
- Feature-specific tools.
- Feature-specific format.

~200-1000 tokens; medium-stable; changes occasionally.

### 3.4 The tenant overlay

Per tenant (optional):

- Tenant-specific terminology.
- Tenant-specific persona.
- Tenant-specific compliance constraints.
- Tenant-specific tool restrictions.

~50-300 tokens; tenant-specific.

Cross-link to [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md §3](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md).

### 3.5 The composition order

Order matters:

```
Platform base (most general; first)
  ↓
Feature overlay (specific; refines base)
  ↓
Tenant overlay (most specific; refines feature)
```

Each layer can override the previous; later layers win on conflict.

### 3.6 The "platform vs feature" boundary

What goes in platform vs feature:

- Platform: applies to every feature.
- Feature: applies only to this feature.

If a "platform" instruction is actually used by only one feature, it belongs in that feature's overlay.

### 3.7 The "is this base or override" decision

When adding a new instruction:

- Is it true for every feature? → platform base.
- Is it true for this feature only? → feature overlay.
- Is it specific to one tenant? → tenant overlay.

Categorization discipline keeps the architecture clean.

### 3.8 The size budget per layer

```
Platform base:    ~800 tokens (target)
Feature overlay:  ~500 tokens (target)
Tenant overlay:   ~150 tokens (target; many tenants have zero)
─────────────────────────────────
Total system:    ~1450 tokens (median; 3000 tokens cap)
```

Per layer, a size budget; exceeding triggers refactor.

### 3.9 The "we have too many overlays" warning

If overlays multiply (50+ tenant overlays):

- Operational overhead grows.
- Drift inevitable.
- Tenant overlays should be exception, not norm.

Most tenants use just platform + feature; overlays for the minority who need them.

---

## 4. Prompts-as-components

The components model.

### 4.1 The component pattern

Each layer is composed of named components:

```yaml
platform_base:
  identity: "You are an AI assistant for Meridian Health..."
  refusal_policy: "..."
  format_conventions: "..."
  safety_constraints: "..."

feature_overlay (care-coordinator):
  feature_identity: "As the Care Coordinator..."
  feature_behavior: "..."
  feature_tools: "Available tools: ..."
  feature_format: "..."

tenant_overlay (meridian-southwest):
  tenant_terminology: "Refer to clinicians as 'providers'..."
  tenant_persona: "..."
```

Named components; each is its own unit.

### 4.2 The assembly

At request time:

```python
def assemble_system_prompt(context):
    parts = [
        platform_base.identity,
        platform_base.refusal_policy,
        platform_base.format_conventions,
        platform_base.safety_constraints,
        feature_overlay(context.feature).feature_identity,
        feature_overlay(context.feature).feature_behavior,
        feature_overlay(context.feature).feature_tools,
        feature_overlay(context.feature).feature_format,
    ]
    if context.tenant.has_overlay:
        parts.extend([
            tenant_overlay(context.tenant).tenant_terminology,
            tenant_overlay(context.tenant).tenant_persona,
        ])
    return "\n\n".join(parts)
```

Assembly is deterministic; cacheable; testable.

### 4.3 The component-versioning

Each component has a version:

```yaml
platform_base:
  version: 12
  identity:
    version: 3
    text: "You are an AI assistant for Meridian Health..."
  refusal_policy:
    version: 7
    text: "..."
```

When a component changes, its version bumps; the assembly bumps too.

### 4.4 The component-ownership

Each component has an owner team:

- Platform base: platform team.
- Feature overlay (care-coordinator): clinical-AI team.
- Tenant overlay (meridian-southwest): account team.

Changes to a component need owner approval.

### 4.5 The component-as-code

Components in version control:

```
meridian-platform/prompts/
  platform-base/
    identity.md (v3)
    refusal_policy.md (v7)
    format_conventions.md (v4)
    safety_constraints.md (v5)
  feature-overlays/
    care-coordinator/
      feature_identity.md (v2)
      feature_behavior.md (v5)
      feature_tools.md (v8)
      feature_format.md (v3)
  tenant-overlays/
    meridian-southwest/
      tenant_terminology.md (v3)
      tenant_persona.md (v2)
```

Git-tracked; reviewable; revertable.

### 4.6 The component-testing

Tests per component:

```python
def test_care_coordinator_feature_identity():
    overlay = load_overlay("care-coordinator/feature_identity.md")
    assert "Care Coordinator" in overlay
    assert "clinical" in overlay.lower()
    # Eval test: model trained on assembled prompt produces correct identity
```

Components have their own tests; assembly has integration tests.

### 4.7 The component-evolution

When a component needs to change:

- Edit the component.
- Bump version.
- Run tests.
- Run eval suite.
- Deploy.

Same as code; familiar workflow.

### 4.8 The component-discovery

A component catalog:

```
$ list_components
Platform base: 4 components (v3, v7, v4, v5)
Feature overlays: 8 (5 features)
Tenant overlays: 3 (12 tenants; 9 use defaults only)
```

Surfaces the surface area; informs maintenance.

---

## 5. Per-tenant overlays

Multi-tenant customization at the prompt layer.

### 5.1 The minimal tenant overlay

For most tenants:

- No overlay; uses platform base + feature default.

This is the desired state for the majority. Custom overlays are exceptions.

### 5.2 The scoped overlay

When a tenant needs customization, the overlay is bounded:

- Terminology (call clinicians "providers" not "doctors").
- Persona (respond as "Meridian Southwest assistant").
- Compliance addendum (PHI handling for their compliance posture).

NOT in the overlay:

- Behavior modifications (that would degrade safety).
- New tools (those should be platform-wide if needed).
- Custom refusal policy beyond restriction (overlay can add restrictions; not remove platform-wide ones).

Cross-link to [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md §3.3](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md).

### 5.3 The overlay approval workflow

Tenant overlay changes:

1. Tenant proposes (account team).
2. Platform team reviews (does it conflict with platform; does it degrade safety).
3. Eval run against the tenant's eval suite.
4. Approved by platform + tenant.

Not "tenant edits their YAML and pushes."

### 5.4 The overlay test suite

Each tenant overlay has its own eval suite:

- Verifies the overlay's instructions are respected.
- Verifies the overlay doesn't break platform behavior.
- Runs on every change to platform base or feature overlay.

### 5.5 The "we don't allow per-tenant overlays" pattern

Some platforms decline:

- Operational overhead.
- Risk of drift.

For their target customer set, platform-only is sufficient.

For Meridian's case (some enterprise customers required overlays), the overlay capability is justified.

### 5.6 The "tenant overlay grew" indicator

When a tenant overlay grows beyond ~300 tokens:

- It's becoming a separate feature.
- Consider promoting to a feature overlay (if multi-tenant relevant).
- Consider whether the customizations are still necessary.

Annual review.

### 5.7 The overlay-vs-feature decision

Sometimes a "tenant overlay" should be a "feature overlay":

- Overlay applies to one tenant; feature overlay applies to a tenant tier or feature use.
- If the same overlay would be applied to multiple tenants, promote to feature overlay.

---

## 6. Testing and review

The discipline that keeps the system prompt healthy.

### 6.1 The eval coverage per component

Each component (or component group) has eval cases:

- Identity component: cases that test the model adopts the identity.
- Refusal policy: cases that test refusals fire correctly.
- Format contract: cases that test format adherence.
- Tools: cases that test tool selection.

Cross-link to [ai-engineering-reference-architecture / eval-engineering / golden-set-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/golden-set-design.md).

### 6.2 The change review

Every change to the system prompt:

- PR review (engineering eyes).
- Eval suite passes.
- Component owner approval.

For platform base changes: extra scrutiny (affects everything).

### 6.3 The pre-deploy eval

Before promoting a prompt change to production:

- Full eval suite (golden cases + regression cases).
- Assembly tests (does the composed prompt look right).
- Manual review of a few sample outputs.

### 6.4 The post-deploy monitoring

After deploying:

- Live-judge quality (cross-link to [reliability-engineering/fault-budgets-for-ai.md §3](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fault-budgets-for-ai.md)).
- Cost-per-call (cross-link to [cost-and-performance-architecture/token-economics.md §6](../cost-and-performance-architecture/token-economics.md)).
- User feedback signals.

Detect regressions early.

### 6.5 The "we A/B test the new prompt" pattern

For impactful changes:

- 5% of traffic to new prompt.
- 95% to existing.
- Compare quality metrics for 1-2 weeks.
- Decide.

Cross-link to [ai-engineering-reference-architecture / prompt-engineering / prompt-ab-testing.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-ab-testing.md).

### 6.6 The "we rolled back the prompt" capability

Prompts must be reversible:

- Version control.
- Roll back is a version-pin change.
- Verified in pre-production.

### 6.7 The audit trail

Every prompt change logged:

- Who.
- What changed.
- When.
- Why (PR description).

Compliance / debugging requirement.

### 6.8 The prompt-versioning in catalogue

Each model + prompt combination tracked:

- Catalogue entry (cross-link to [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md)).
- Prompt version pinned per feature.

Migration of prompt across model versions handled via catalogue.

---

## 7. The migration from monolith

How to decompose an existing monolith.

### 7.1 The diagnostic

Sample of the monolith:

- Total tokens: ?
- Sections by function (identity / behavior / refusal / format / tools / reasoning): how distributed?
- Redundancy: how much repetition?
- Dead instructions: how many for retired features?

Diagnose before decomposing.

### 7.2 The decomposition strategy

1. **Map the monolith to functions.** Categorize each section.
2. **Identify platform-wide vs feature-specific.** Which sections apply to all features?
3. **Identify dead content.** Which sections refer to retired features?
4. **Plan the target structure.** Platform base + feature overlay per feature + tenant overlays.

### 7.3 The decomposition steps

For each feature:

1. Extract platform-wide content to platform base.
2. Extract feature-specific content to feature overlay.
3. Remove dead content.
4. Reduce redundancy.
5. Assemble and test against the monolith's eval suite.
6. Compare outputs side-by-side.
7. Iterate until parity.

### 7.4 The "we have N existing features" challenge

For N features, do them one at a time:

- Feature 1: decompose; verify; deploy.
- Feature 2: build on feature 1's decomposition.
- ...

Avoid "all features at once" big-bang.

### 7.5 The size reduction goal

Typical reduction: 30-50% in total tokens after decomposition.

- Removed redundancy across features.
- Removed dead content.
- Tightened wording.

### 7.6 The quality validation

After each feature's decomposition:

- Eval suite passes (parity with monolith).
- Cost-per-call lower.
- Engineering can reason about the prompt.

### 7.7 The "we accept temporary parallel" pattern

During migration:

- Old monolith and new decomposed system coexist.
- Feature flag controls which is used per feature.
- Migrate features one by one.
- Decommission monolith after last feature.

### 7.8 The lessons-learned process

After decomposition:

- Document what was learned.
- Update the platform's guidelines.
- Prevent future monolith growth.

---

## 8. Worked Meridian example

Meridian's prompt architecture evolved from monolith to composed.

### 8.1 The pre-decomposition state (early 2025)

Each AI feature had its own monolithic system prompt:

- Care Coordinator: 4200 tokens.
- Patient API chat: 3800 tokens.
- Clinical decision support: 5100 tokens.
- Analytics warehouse copilot: 3200 tokens.

Total platform "prompt surface": ~16k tokens.

Redundancy: significant. Each prompt had similar refusal policies, safety constraints, format conventions, but written slightly differently.

### 8.2 The decomposition (Q1 2026)

Platform team led the decomposition effort.

**Platform base components extracted:**

- `identity.md` (~150 tokens): "You are an AI assistant for Meridian Health..."
- `refusal_policy.md` (~400 tokens): clinical disclaimers, PII handling, off-topic.
- `format_conventions.md` (~200 tokens): markdown structure, citation format.
- `safety_constraints.md` (~300 tokens): explicit boundaries.

Total platform base: ~1050 tokens.

**Feature overlays:**

- Care Coordinator: ~600 tokens (clinical-specific identity, behavior, tools).
- Patient API chat: ~400 tokens (chat-specific).
- Clinical decision support: ~700 tokens (decision-specific behavior, tool catalog).
- Analytics warehouse copilot: ~500 tokens (analytics-specific).

**Tenant overlays:**

- 3 tenants have overlays (most use defaults).
- Each overlay ~150 tokens.

### 8.3 The total post-decomposition

Per Care Coordinator call (typical):

- Platform base: 1050 tokens.
- Feature overlay: 600 tokens.
- Tenant overlay (if any): 0-150 tokens.

Total system prompt: ~1650-1800 tokens (vs 4200 monolithic).

Reduction: ~55%.

### 8.4 The decomposition timeline

```
Week 1-2: Audit existing monoliths; categorize sections.
Week 3-4: Extract platform base; verify it's consistent.
Week 5-6: Decompose Care Coordinator; pre-production eval; deploy.
Week 7-8: Decompose Patient API chat.
Week 9-10: Decompose Clinical decision support.
Week 11-12: Decompose Analytics warehouse copilot.
Week 13: Final sweep; remove monolith remnants.
```

13 weeks; ~3 engineering FTE during peak.

### 8.5 The infrastructure cost

- Prompt assembly layer: ~2 weeks engineering.
- Component storage / versioning: integrated with existing config management.
- Eval suite expansion: ~2 weeks (more cases per component).

### 8.6 The benefits realized

- Cost: -55% on system-prompt tokens; ~$15k/month saved across platform.
- Latency: TTFT slightly faster (smaller prompts process faster).
- Maintainability: engineers can confidently modify components.
- New feature ramp-up: writing a feature overlay is 1-2 hours (vs 1-2 days for monolithic).

### 8.7 The Q2 2026 platform-base update

Platform team added a new safety constraint to the platform base:

- Single edit to `safety_constraints.md`.
- All features automatically benefit on next deploy.
- Without decomposition: would have needed to edit 4 monolithic prompts.

### 8.8 The Q3 2026 tenant overlay scenario

A new enterprise tenant required terminology customization:

- Created tenant overlay with their terminology mapping.
- Approved through workflow.
- Deployed.

Took ~3 hours (vs days for monolithic approach).

### 8.9 The eval expansion

Per-component eval cases:

- Platform refusal policy: 30 test cases.
- Platform format conventions: 25 cases.
- Care Coordinator feature behavior: 50 cases.
- Per-tenant overlay: 5-15 cases each.

Total eval cases: ~250 across all components.

Cross-component tests: 100 cases for assembled prompts.

Total prompt test coverage: ~350 cases; sufficient.

### 8.10 The post-decomposition discipline

Quarterly:

- Review each component's size.
- Identify candidates for tightening.
- Identify dead content.

Component growth is monitored; refactor before monolith reforms.

### 8.11 The cross-team learning

After Meridian's decomposition, the discipline spread:

- Internal documentation.
- Tech-talk to other AI teams.
- New AI features start with decomposed structure (no greenfield monolith).

### 8.12 The lessons

- Decomposition is a one-time investment with ongoing returns.
- 55% token reduction is typical; cost savings substantial.
- Component testing is the discipline that prevents regression.
- Quarterly review prevents drift.

---

## 9. Anti-patterns

### 9.1 The runaway monolith

**Pattern.** System prompt grows uncontrolled. Each customer complaint adds a paragraph. Two years in: 4000+ tokens.

**Corrective.** Decompose per §7; institute size budgets per §3.8.

### 9.2 The "we have 5 system prompts that say the same thing" duplication

**Pattern.** Each feature has its own monolithic prompt; substantial redundancy across.

**Corrective.** Platform base extracts the common content per §3.2.

### 9.3 The opaque prompt

**Pattern.** System prompt is one long paragraph; no structure; impossible to scan.

**Corrective.** Section headers per §2.9.

### 9.4 The undocumented edge-case patch

**Pattern.** Engineer adds a line to fix one customer's case; doesn't document why; future engineer can't tell if it's still needed.

**Corrective.** Component-level changes go through PR review with rationale.

### 9.5 The "tenants edit their own overlay" governance gap

**Pattern.** Tenants edit YAML; push; safety regressions ship.

**Corrective.** Approval workflow per §5.3.

### 9.6 The prompt without eval coverage

**Pattern.** Prompt changes ship; no eval; quality regresses; nobody catches for weeks.

**Corrective.** Per-component eval per §6.1.

### 9.7 The "we'll add structure when we have time" deferral

**Pattern.** Structure deferred indefinitely. Cost of decomposition grows.

**Corrective.** Decompose when prompt exceeds 2000 tokens; never let it grow further.

### 9.8 The dead content accumulation

**Pattern.** Instructions for retired features remain. Bloat.

**Corrective.** Quarterly audit per §8.10.

### 9.9 The "tenant overlay grew to 1000 tokens" smell

**Pattern.** Tenant overlay becomes large; effectively a separate feature.

**Corrective.** Promote to feature overlay per §5.7.

### 9.10 The version-untracked prompt

**Pattern.** Prompt edits not versioned. Rollback impossible. Audit trail absent.

**Corrective.** Component-as-code per §4.5.

---

## 10. Findings (sprint-assignable)

**ARCH-SP-001 (P0). System prompt is a monolith.** Cost, maintainability, quality all suffer. Decompose per §7. Owner: AI platform.

**ARCH-SP-002 (P0). Per-prompt size budget undefined.** Prompts grow uncontrolled. Per §3.8. Owner: AI platform.

**ARCH-SP-003 (P0). No platform-base extraction across features.** Redundancy across feature prompts. Per §3.2. Owner: AI platform.

**ARCH-SP-004 (P0). Prompt changes ship without eval.** Quality regressions. Per §6.1 and §6.3. Owner: AI platform + eval.

**ARCH-SP-005 (P1). Per-tenant overlays edited without governance.** Safety regressions risk. Approval workflow per §5.3. Owner: AI platform + product.

**ARCH-SP-006 (P1). Components not versioned.** Rollback impossible; audit gap. Per §4.3. Owner: AI platform.

**ARCH-SP-007 (P1). Prompts not in version control.** Component-as-code per §4.5. Owner: AI platform.

**ARCH-SP-008 (P1). No structured section headers in prompts.** Hard to scan and maintain. Per §2.9. Owner: feature teams.

**ARCH-SP-009 (P1). Dead content accumulates in prompts.** Bloat. Quarterly audit per §8.10. Owner: AI platform + feature teams.

**ARCH-SP-010 (P1). Component ownership not assigned.** Changes happen without approval. Per §4.4. Owner: AI platform + product.

**ARCH-SP-011 (P2). No live-judge for post-deploy quality.** Regressions undetected. Per §6.4 and [fault-budgets-for-ai.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fault-budgets-for-ai.md). Owner: AI platform + observability.

**ARCH-SP-012 (P2). A/B testing for prompt changes absent.** Impact validation post-deploy only. Per §6.5. Owner: AI platform.

**ARCH-SP-013 (P2). Component eval coverage uneven.** Some components untested. Per §6.1. Owner: eval + AI platform.

**ARCH-SP-014 (P2). Tenant overlay growth not monitored.** Some overlays approach feature-size. Per §5.6. Owner: AI platform.

**ARCH-SP-015 (P2). Migration from monolith deferred.** Costs accumulate. Plan per §7. Owner: engineering management.

**ARCH-SP-016 (P3). No audit trail for prompt changes.** Compliance / debugging gap. Per §6.7. Owner: AI platform.

**ARCH-SP-017 (P3). Prompt-version not in model catalogue.** Per-model prompt-version pinning absent. Per §6.8. Owner: AI platform.

**ARCH-SP-018 (P3). Cross-team prompt learnings not shared.** Patterns rediscovered. Per §8.11. Owner: AI platform + engineering management.

---

## 11. Adoption sequencing checklist

- [ ] **Diagnose existing prompts (§7.1).**
- [ ] **Plan target structure (§7.2).**
- [ ] **Extract platform base (§3.2).**
- [ ] **Build prompt assembly layer (§4.2).**
- [ ] **Move prompts to version control (§4.5).**
- [ ] **Add component versioning (§4.3).**
- [ ] **Assign component ownership (§4.4).**
- [ ] **Define size budgets per layer (§3.8).**
- [ ] **Build per-component eval (§6.1).**
- [ ] **Add change review workflow (§6.2).**
- [ ] **Establish per-tenant overlay governance (§5.3).**
- [ ] **Quarterly component audit (§8.10).**
- [ ] **A/B testing for major prompt changes (§6.5).**

---

## 12. References

**In this folder.**
- [context-window-budgeting.md](./context-window-budgeting.md) — system prompt is one allocation in the budget.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — assembly mechanics.
- [chat-history-architecture.md](./chat-history-architecture.md) — chat history is another context allocation.
- [long-context-vs-rag.md](./long-context-vs-rag.md) — context-window decision framework.
- [prompt-as-api-discipline.md](./prompt-as-api-discipline.md) *(coming)* — versioning prompts.
- [few-shot-vs-fine-tune-vs-system-prompt.md](./few-shot-vs-fine-tune-vs-system-prompt.md) *(coming)* — when each approach.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md) — per-tenant mechanics.
- [cost-and-performance-architecture/token-economics.md](../cost-and-performance-architecture/token-economics.md) — prompt size and cost.
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue + prompt versioning.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — worked example.

**Sibling repos.**
- [ai-engineering-reference-architecture / prompt-engineering / prompts-as-code-discipline.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompts-as-code-discipline.md) — engineering-side discipline.
- [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) — engineering-side versioning.
- [ai-engineering-reference-architecture / prompt-engineering / prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md) — engineering-side library patterns.
- [ai-engineering-reference-architecture / eval-engineering / golden-set-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/golden-set-design.md) — eval design.

**External.**
- Anthropic / OpenAI / Google prompt-engineering guides.
- Promptfoo, LangSmith documentation for prompt management.
- Component-oriented design literature.
