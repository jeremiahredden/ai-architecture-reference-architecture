# Model Migration Playbook

> **Audience.** Architects whose primary model has been deprecated, whose better model has shipped, or whose vendor's pricing has shifted enough to make migration worth the work. Tech leads who have done one model migration and want to make the next one cheaper. Anyone whose "we'll just swap the model" plan discovered that prompts, evals, and downstream consumers all assumed the old model. **Scope.** The *architectural* playbook for migrating from model A to model B without a quality regression: parallel shadow traffic, eval-suite cross-check, prompt-port discipline, the fallback configuration that supports rollback, the rollout sequence, the rollback criteria. Not the build-vs-buy decision (see [build-vs-buy-decision.md](./build-vs-buy-decision.md)). Not the model selection framework (see [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md)). Not the per-tenant fine-tune migration mechanics (see [multi-tenancy-and-isolation/per-tenant-fine-tuning.md §7](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Model migrations happen often: Anthropic ships Sonnet 4.6, then 4.7; OpenAI ships GPT-4.5, then 5; a workload that was right for one model is suddenly behind a newer one. The migration question is "how do we move without breaking things?" — and the answer is the same playbook applied repeatedly, not a one-off improvisation.

The teams that get migration right share specific habits:

- **They treat the migration as a structured project**, not an inline swap. There's a plan, a timeline, and stop conditions.
- **They run the new model in parallel with the old** before committing. Shadow traffic produces real comparison data.
- **They re-run the eval suite** against the new model and verify acceptable quality.
- **They port the prompt deliberately**, not literally. The new model may need adjustments.
- **They have a rollback plan**, and they've tested it.

The teams that get migration wrong share predictable patterns:

- **They swap the model in a deploy and watch dashboards.** When quality regresses, the rollback is improvised.
- **They assume the new model is "the same or better" without eval.** It usually isn't on every workload.
- **They don't port the prompt.** Subtle behavior changes accumulate.
- **They don't have the old model accessible after switch.** Rollback means rebuilding.

The migrations that ship cleanly cost a few weeks of engineering work. The migrations that ship messily cost a few months of follow-up incidents.

This document covers the architectural playbook. It is opinionated about four things:

1. **Migration is a structured project, not an inline change.** Time-bound; staged; reversible. Engineering should resist the "just swap it" pressure.
2. **Parallel shadow traffic is the load-bearing validation step.** Static eval suites are necessary but not sufficient. Real production traffic reveals issues static evals miss.
3. **The prompt must be ported, not copied.** Each model has quirks; the prompt that was tuned for the old model is rarely optimal for the new one.
4. **Rollback must be tested before launch, not after a problem.** The first time rollback is attempted should not be during an incident.

Structure: (2) when to migrate (and when not to); (3) the migration phases; (4) parallel shadow traffic; (5) eval-suite cross-check; (6) prompt-port discipline; (7) the rollout sequence; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. When to migrate (and when not to)

Not every new model release warrants migration. The decision.

### 2.1 The reasons to migrate

**Deprecation by provider.** Provider announces end-of-life for current model; migration is forced.

**Capability improvement.** New model is meaningfully better on the workload's eval suite (e.g., +5% pass rate, +10% structured-output reliability).

**Cost reduction.** New model is significantly cheaper at comparable quality (e.g., 30%+ cost reduction with eval pass).

**Latency improvement.** New model is faster at comparable quality (e.g., 30%+ faster P99).

**Strategic alignment.** Provider's roadmap aligns with workload's future needs.

### 2.2 The reasons not to migrate

**"It's new and shiny."** The new model is available but doesn't improve the current workload's metrics.

**"Everyone else is migrating."** Peer pressure rather than evidence.

**Marginal improvements.** 1-2% eval improvement that doesn't justify the migration cost.

**Active production stability.** Current model is working; risk of migration > benefit.

**Other priorities.** Migration is real engineering work; other work may be higher-value.

### 2.3 The migration decision matrix

| Criterion | Score (1-5) | Notes |
| --- | --- | --- |
| Eval improvement | _ | Per workload's specific eval |
| Cost reduction | _ | Per current spend |
| Latency improvement | _ | Per current SLO |
| Deprecation pressure | _ | 5 = forced; 1 = no pressure |
| Engineering capacity | _ | 5 = readily available; 1 = stretched |
| Risk tolerance | _ | 5 = comfortable; 1 = risk-averse |

Aggregate score informs the decision. No automatic threshold; subjective weighting.

### 2.4 The "we'll migrate when v5 comes out" deferral

Sometimes the right answer is wait:

- Provider releases v4.6; you're on v4.5; v4.7 is announced for next month.
- Wait for v4.7 to avoid double-migrating.

The decision is a strategic one; consider release cadence.

### 2.5 The portfolio view

A platform has many features; each may have its own model. Migration isn't all-at-once:

- Care Coordinator migrates first (high value; resources allocated).
- Patient API chat migrates next quarter.
- Document classification stays on current (no benefit).

Per-workload decisions; the platform doesn't move uniformly.

### 2.6 The "we have an old model we should retire" check

Some workloads are on outdated models simply because nobody revisited:

- Annual review surfaces these.
- Decide: migrate, retire, or accept.

Avoid drift to ancient versions.

---

## 3. The migration phases

A structured migration has 5 phases.

### 3.1 Phase 1: Evaluation

Before committing to migration:

- Run the workload's eval suite against the new model.
- Document quality delta per dimension.
- Document cost delta.
- Document latency delta.
- Identify any regressions.

**Output.** Decision: migrate, don't migrate, or investigate further.

**Duration.** 1-2 weeks (eval runs + analysis).

### 3.2 Phase 2: Planning

If migrating:

- Define the migration scope (which features; which tenants).
- Define the rollout schedule.
- Define rollback criteria.
- Identify stakeholders.

**Output.** Migration plan document.

**Duration.** 1 week.

### 3.3 Phase 3: Implementation

Engineering work:

- Update the model reference (catalogue entry; cross-link to [model-catalogue-and-registry.md](./model-catalogue-and-registry.md)).
- Port the prompt (§6).
- Update tests.
- Stage in pre-production.

**Output.** Migration-ready code.

**Duration.** 1-3 weeks.

### 3.4 Phase 4: Validation

The deciding phase:

- Shadow traffic against new model (§4).
- Compare outputs.
- Eval against full eval suite.
- Internal user testing.

**Output.** Go / no-go decision for production rollout.

**Duration.** 1-3 weeks.

### 3.5 Phase 5: Rollout

The cutover:

- Canary rollout (1-10% of traffic).
- Monitor metrics.
- Progressive ramp-up.
- Full rollout.
- Decommission old model.

**Output.** Migrated.

**Duration.** 1-4 weeks (depends on traffic volume and risk).

### 3.6 The total timeline

For a typical AI feature migration: 6-12 weeks.

Faster: forced deprecation (less validation; more risk).
Slower: high-risk feature (more validation; safer).

### 3.7 The "we're moving faster than this" reality check

Some teams attempt 1-2 week migrations:

- May work for low-risk features.
- High risk of post-migration incidents.
- Engineering management should question if it's a good trade.

The playbook's value is in catching issues before production; compressing the playbook compresses validation.

---

## 4. Parallel shadow traffic

The load-bearing validation step.

### 4.1 The shadow pattern

A copy of production traffic is sent to both old and new models:

```
User request → Primary path (old model) → Response to user
            → Shadow path (new model) → Response logged, not returned
```

The user gets the old model's response (no impact). The new model's response is logged for comparison.

### 4.2 The sampling rate

Not all traffic needs to be shadowed:

- 1-10%: lightweight; data after a week.
- 50%: heavier; data faster.
- 100%: expensive (doubles inference cost); only briefly.

Start at 5%; increase as confidence grows.

### 4.3 The comparison metrics

Per shadowed call, compare:

- Output similarity (semantic similarity score).
- Output structure (schema validation).
- Quality scores (judge model).
- Latency.
- Cost.
- Tool-call accuracy (if applicable).

### 4.4 The aggregate analysis

Over the shadow period:

- Per-metric distribution comparison.
- Outlier analysis (calls where new is much worse than old).
- Cohort analysis (which input classes show regression).

### 4.5 The "we found a regression" handling

If shadow reveals regression:

- Categorize the regression (specific input class? specific tool?).
- Decide: prompt fix; model variant; or abort migration.
- Iterate.

Shadow is the place to find regressions; before production.

### 4.6 The "shadow looks good; do we go?" decision

Acceptance criteria for migration:

- Quality score parity or improvement.
- Cost improvement (or acceptable trade).
- Latency improvement (or acceptable trade).
- No regressions in critical workflows.

Document the criteria upfront; decide objectively at decision time.

### 4.7 The non-shadowable workflows

Some workflows can't easily be shadowed:

- Side-effect tools (agent that sends emails).
- Stateful conversations (history is per-model).
- Workflows with downstream commits.

For these:
- Test in pre-production with synthetic data.
- Pilot with internal users.
- Be more cautious in rollout.

---

## 5. Eval-suite cross-check

The structured quality validation.

### 5.1 The eval suite for migration

The eval suite (cross-link to [ai-engineering-reference-architecture / eval-engineering / regression-eval-suites.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/regression-eval-suites.md)) is the structured test for quality.

For migration:
- Run eval against current model (baseline).
- Run eval against new model.
- Compare.

### 5.2 The eval pass criteria

For each eval case:

- Pass / fail (binary).
- Score (continuous; 0-1).
- Tags (which workload class).

Per-case comparison reveals patterns.

### 5.3 The "eval pass rate dropped" analysis

If new model has lower pass rate:

- Which cases failed on new but passed on old?
- Is the pattern consistent (specific input class)?
- Can the prompt be adjusted?

Sometimes a small prompt adjustment closes the gap.

### 5.4 The "eval pass rate is the same; production may differ" caveat

Eval suites are samples; production may behave differently. Shadow traffic complements eval:

- Eval validates known patterns.
- Shadow validates the full production distribution.

Both needed.

### 5.5 The new-model-specific eval cases

The new model may have new capabilities:

- New eval cases that exercise them.
- Validate the workload can take advantage.

Migration is also an opportunity to expand the eval suite.

### 5.6 The eval-as-input-to-decision

The migration decision uses eval data:

- Quality delta from eval.
- Cost delta (per eval run).
- Latency delta.

Triangulate with shadow data and internal testing.

### 5.7 The eval re-run on the new model's updates

After migration, the eval continues:

- Re-run regularly.
- Catch drift.
- Inform future decisions (next migration).

---

## 6. Prompt-port discipline

The prompt is rarely portable as-is.

### 6.1 The "swap and hope" anti-pattern

Take the old prompt; use it with new model:

- May work; may not.
- Subtle behavior changes.
- Quality drift.

Don't.

### 6.2 The deliberate port

Each prompt element reviewed:

- System prompt: does the new model interpret it the same?
- Few-shot examples: still effective?
- Format instructions: needed differently?
- Tool descriptions: same format?

Port deliberately.

### 6.3 The model-specific tuning

Each model has quirks:

- Some respond better to declarative instructions; some to examples.
- Some need explicit format guidance; others infer.
- Some are more verbose; others terse.

Port for the new model's quirks.

### 6.4 The eval-driven tuning

Tune the prompt against the eval:

- Initial port: best-guess.
- Run eval; identify failures.
- Adjust prompt; re-run.
- Iterate until eval matches or exceeds baseline.

Engineering effort; pays off in production quality.

### 6.5 The "we tuned the prompt for the new model; old model now fails" trap

After tuning, the prompt may work less well with the old model:

- Affects rollback (if we need to revert).
- Solution: maintain prompt versions per model.

Cross-link to [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md).

### 6.6 The system-prompt overlay management

For multi-tenant (cross-link to [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)):

- Tenant overlays may need adjustment per model.
- Per-tenant prompt testing.
- Migration is per-tenant scoped.

### 6.7 The tool-description port

For agent workloads (tool descriptions in prompt):

- Tool descriptions may need adjustment.
- Format may differ between providers.
- Cross-link to [integration-architecture/tool-call-architecture.md](../integration-architecture/tool-call-architecture.md).

### 6.8 The structured-output port

Structured output format may differ:

- Anthropic: tool calling with input_schema.
- OpenAI: function calling with parameters.
- Cohere: tool use with different format.

If migrating across providers, the structured-output integration may need rewriting.

---

## 7. The rollout sequence

The actual cutover.

### 7.1 The canary rollout

Initial rollout:

- 1-5% of traffic to new model.
- Other 95-99% to old.
- Monitor for issues.
- Duration: 1-3 days.

### 7.2 The ramp-up

Progressive increase:

- Day 1: 5%.
- Day 3: 10%.
- Day 7: 25%.
- Day 10: 50%.
- Day 14: 100%.

Each step: monitor; verify health.

### 7.3 The rollout gates

At each step, verify:

- Quality SLO compliance.
- Latency SLO compliance.
- Cost SLO compliance.
- No customer complaints.

If any gate fails: pause; investigate; decide.

### 7.4 The cohort-aware rollout

For multi-tenant: roll out per tenant tier:

- Free tier first (lowest stakes).
- Standard next.
- Premium / enterprise last (highest stakes).

Detects per-tier issues before they hit critical customers.

### 7.5 The geographic rollout

For multi-region:

- Single region first.
- Other regions after.

Geographic isolation of any issues.

### 7.6 The "we got to 100%; do we decommission the old model" decision

At 100%:

- Old model still configured.
- Decommission after a stable period (e.g., 2 weeks at 100%).
- Decommissioning means: remove from catalogue, remove from infrastructure, stop testing.

Premature decommission removes the rollback option.

### 7.7 The rollback criteria

Pre-defined; objective:

- Quality SLO violation > N% over M hours.
- Latency SLO violation > N% over M hours.
- Customer complaints > threshold.
- Cost SLO violation > N%.
- Any unexpected behavior in critical workflow.

If criteria met: roll back to old model.

### 7.8 The rollback mechanics

- Flip the catalogue entry alias.
- Or flip the feature flag.
- Wait for in-flight requests to complete.
- Verify rollback completed.
- Investigate the issue that triggered rollback.

Rollback should be fast (minutes); pre-tested in pre-production.

### 7.9 The "rollback worked; what now" path

After rollback:

- Investigate.
- Iterate on prompt / config.
- Re-run shadow / eval.
- Plan re-attempt.

A rolled-back migration is not failed; it's paused.

### 7.10 The decommission step

After confidence period:

- Remove old model from catalogue.
- Remove infrastructure (if self-hosted).
- Update documentation.
- Archive eval cases specific to old model.

Decommission frees resources.

---

## 8. Worked Meridian example

Meridian migrated Care Coordinator from Claude Sonnet 4.5 to 4.6 in Q1 2026. The playbook applied.

### 8.1 The decision

Anthropic announced Sonnet 4.6 in February 2026. Anthropic's published benchmarks showed improvement; pricing was unchanged; deprecation of 4.5 was not announced but expected within months.

Meridian's decision:

- Run eval against 4.6 for Care Coordinator workload.
- Result: 96% pass rate (vs 4.5's 95%); marginal improvement.
- Cost: unchanged.
- Latency: 4.6's P99 was 6.8s (vs 4.5's 7.4s); 8% improvement.

Migration approved; planned for March 2026.

### 8.2 Phase 1: Evaluation

Eval suite (cross-link to [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md)):

- 250 eval cases covering clinical workflows.
- Run against 4.5 (baseline): 96% pass rate.
- Run against 4.6 (candidate): 97% pass rate.
- 6 cases passed on 4.5 failed on 4.6 (specific edge cases: rare medication names).
- 11 cases passed on 4.6 failed on 4.5.

Net: improvement, with specific concerns about rare-medication cases.

### 8.3 Phase 2: Planning

Plan documented:

- Scope: Care Coordinator agent only; Patient API chat separately migrated later.
- Schedule: 6-week total (eval → shadow → canary → full).
- Rollback criteria: quality SLO < 94% for > 2 hours; or > 5 customer complaints in 24 hours.
- Stakeholders: AI platform team (lead); clinical informatics (validation); customer success (rollback authority).

### 8.4 Phase 3: Implementation

- Catalogue entry updated to support both 4.5 and 4.6 references.
- Prompt port: 4.6's slightly different behavior on tool calling required adjustment to the system prompt section.
- Eval cases for rare-medication edge cases tagged for special monitoring.
- Pre-production deployment validated.

### 8.5 Phase 4: Shadow traffic

Shadow setup:

- 10% of Care Coordinator traffic shadowed to 4.6.
- 90% on 4.5 (production path).
- 2-week shadow window.

Results:

- 4.6 quality scores: 96.5% (vs 4.5's 95.8% on same shadow window).
- 4.6 latency P99: 6.7s (vs 4.5's 7.5s on same period); ~10% improvement.
- 4.6 cost: same per call.
- Outlier analysis: 4 calls where 4.6 produced lower-quality output (3 were prompt-port issues; fixed).

Decision: proceed with rollout.

### 8.6 Phase 5: Rollout

Canary at 5%:

- Day 1-2: 5%.
- Quality SLO: 96.7% (above 95% threshold).
- Latency SLO: P99 6.8s (within 8s target).
- No customer complaints.
- Proceeded to next step.

Ramp:

- Day 3-4: 20%.
- Day 5-7: 50%.
- Day 8-10: 100%.

At each step, verified all SLOs; proceeded.

### 8.7 The week-after monitoring

After 100%:

- 7 days of monitoring with all traffic on 4.6.
- Quality and latency held.
- Customer feedback: 2 informal mentions of "the AI seems to remember context better" (4.6 has subtle improvements).

### 8.8 The decommission

After 14 days at 100% stable:

- 4.5 removed from catalogue's active section (kept in deprecated for reference).
- 4.5 routing removed from feature flag.
- Documentation updated.

Decommission complete.

### 8.9 The Patient API chat migration

Following Care Coordinator's success, Patient API chat migrated in Q2:

- Same playbook applied.
- Compressed timeline (1 month) given lower risk.
- Smooth migration.

### 8.10 What the playbook produced

- Zero customer-visible issues during either migration.
- Documented validation evidence.
- Eval suite expanded with rare-medication cases identified in shadow.
- Team confidence: next migration will be faster.

### 8.11 The total cost

- Engineering time: ~3 engineers × 6 weeks = 18 engineer-weeks for first migration.
- Shadow infrastructure: ~$2k extra cost for 2-week shadow period (10% additional inference).
- Total: ~$60k loaded.

### 8.12 The lessons

- Shadow is the most informative phase; ate 1/3 of the timeline.
- Prompt-port took longer than expected (4 days vs estimated 1).
- Rollback criteria were objective; nobody had to make a judgment call during stress.
- Documentation enabled the second migration to be faster.

---

## 9. Anti-patterns

### 9.1 The "just swap it" migration

**Pattern.** Engineer updates the model reference in one PR; deploys. Quality regresses; rollback is improvised.

**Corrective.** Structured playbook per §3.

### 9.2 The migration without shadow

**Pattern.** Static eval looks good; team skips shadow. Production reveals regressions that eval missed.

**Corrective.** Shadow traffic per §4.

### 9.3 The prompt that was copy-pasted

**Pattern.** Old prompt used directly with new model. Behavior shifts.

**Corrective.** Deliberate port per §6.

### 9.4 The "we never tested rollback" surprise

**Pattern.** Rollback configured but never exercised. First real rollback reveals broken implementation.

**Corrective.** Pre-launch rollback test.

### 9.5 The "migration is one PR" assumption

**Pattern.** Migration scoped as one PR; takes 2 hours; ships. Realistic migration is weeks of work.

**Corrective.** Honest scoping per §3.6.

### 9.6 The all-tenants-at-once flip

**Pattern.** Migration touches all tenants simultaneously. Per-tenant issues affect everyone at once.

**Corrective.** Per-tenant or per-tier rollout per §7.4.

### 9.7 The decommission-too-fast pattern

**Pattern.** New model rolled out; old model decommissioned immediately. Rollback option lost.

**Corrective.** Wait per §7.6.

### 9.8 The "migration was a success" without verification

**Pattern.** Team declares migration done; quality drift goes undetected; weeks later, issues surface.

**Corrective.** Post-migration monitoring per §8.7.

### 9.9 The "new model has new features but we kept using old prompt" miss

**Pattern.** New model has new capability (e.g., extended thinking, prompt caching); team doesn't use it; benefits left on the table.

**Corrective.** Eval-driven tuning to exploit new capabilities per §5.5.

### 9.10 The migration during a feature-launch window

**Pattern.** Major migration during a high-stakes business window. Risk concentration.

**Corrective.** Schedule migrations for low-risk windows.

---

## 10. Findings (sprint-assignable)

**ARCH-MMP-001 (P0). No documented migration playbook.** First migration is improvised. Adopt playbook per this document. Owner: AI platform + engineering management.

**ARCH-MMP-002 (P0). Migrations done via inline change rather than structured project.** Risk concentration. Define migration as a project per §3. Owner: engineering management + AI platform.

**ARCH-MMP-003 (P0). Shadow traffic not used in migrations.** Production reveals issues eval missed. Implement shadow per §4. Owner: AI platform.

**ARCH-MMP-004 (P1). Eval suite doesn't run against candidate models.** Migration decision lacks evidence. Per §5; eval suite as input to decision. Owner: AI platform + eval.

**ARCH-MMP-005 (P1). Prompt-port discipline absent; prompts copy-pasted across models.** Quality drift. Deliberate port per §6. Owner: AI platform + feature teams.

**ARCH-MMP-006 (P1). Rollback not tested before migration.** First rollback is improvised. Pre-launch test per §9.4. Owner: AI platform + SRE.

**ARCH-MMP-007 (P1). Per-tenant or per-tier rollout absent.** All-tenants-at-once risk. Cohort rollout per §7.4. Owner: AI platform.

**ARCH-MMP-008 (P1). Rollback criteria not pre-defined.** Subjective decision during stress. Per §7.7. Owner: AI platform + engineering management.

**ARCH-MMP-009 (P1). Old model decommissioned too early.** Rollback option lost. Wait per §7.6 (2 weeks at 100%). Owner: AI platform.

**ARCH-MMP-010 (P2). Per-model prompt versions not maintained.** Rollback prompt incompatible with old model. Per §6.5. Owner: AI platform.

**ARCH-MMP-011 (P2). Migration decision lacks structured framework.** Decisions are subjective. Decision matrix per §2.3. Owner: AI platform + product.

**ARCH-MMP-012 (P2). Annual review of out-of-date models absent.** Drift to ancient versions. Per §2.6. Owner: AI platform.

**ARCH-MMP-013 (P2). Tool-description port not considered.** Agent workload behavior changes. Per §6.7. Owner: agent platform.

**ARCH-MMP-014 (P2). Structured-output port not considered.** Format may change across providers. Per §6.8. Owner: AI platform.

**ARCH-MMP-015 (P2). Geographic rollout absent for multi-region workloads.** Cross-region issues. Per §7.5. Owner: AI platform.

**ARCH-MMP-016 (P3). Migration during business-critical window.** Risk concentration. Schedule for low-risk per §9.10. Owner: engineering management.

**ARCH-MMP-017 (P3). Post-migration monitoring absent.** Issues surface late. Per §8.7. Owner: SRE + AI platform.

**ARCH-MMP-018 (P3). Migration cost not tracked.** No data for next migration's planning. Per §8.11. Owner: FinOps + engineering management.

---

## 11. Adoption sequencing checklist

- [ ] **Adopt structured playbook (§3).** Define phases; gate progression.
- [ ] **Build eval suite for each major workload (§5).** Cross-link to engineering eval-engineering.
- [ ] **Build shadow infrastructure (§4).** Per-feature.
- [ ] **Document rollback criteria (§7.7).** Objective; pre-defined.
- [ ] **Pre-launch rollback test (§9.4).**
- [ ] **Per-tier rollout (§7.4) for multi-tenant.**
- [ ] **Per-model prompt versions (§6.5).**
- [ ] **Decision matrix for migrate / don't migrate (§2.3).**
- [ ] **Annual review of model portfolio (§2.6).**
- [ ] **Post-migration monitoring period (§7.6).**
- [ ] **Documentation of migration lessons.**
- [ ] **Migration cost tracking (§8.11).**

---

## 12. References

**In this folder.**
- [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md) — model selection that informs migration.
- [model-routing-and-tiering.md](./model-routing-and-tiering.md) — routing layer that supports gradual rollout.
- [model-catalogue-and-registry.md](./model-catalogue-and-registry.md) — catalogue tracks migration state.
- [build-vs-buy-decision.md](./build-vs-buy-decision.md) — broader model strategy.
- [capability-vs-cost-vs-latency-tradeoffs.md](./capability-vs-cost-vs-latency-tradeoffs.md) — non-functional comparison framework for migration decision.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — per-tenant fine-tune migration.
- [integration-architecture/tool-call-architecture.md](../integration-architecture/tool-call-architecture.md) — tool-description port considerations.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — worked example.

**Sibling repos.**
- [ai-engineering-reference-architecture / eval-engineering / regression-eval-suites.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/regression-eval-suites.md) — eval-suite engineering for migration validation.
- [ai-engineering-reference-architecture / prompt-engineering / prompt-versioning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-versioning.md) — per-model prompt versioning.
- [ai-engineering-reference-architecture / model-lifecycle](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle) — engineering-side migration mechanics (canary, shadow, rollback).
- [ai-engineering-reference-architecture / reliability-engineering / incident-response-for-ai.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/incident-response-for-ai.md) — incident response for migration issues.

**External.**
- Anthropic, OpenAI, Google model release notes and migration guides.
- Continuous delivery literature (Humble, Farley) for the rollout sequence patterns.
- Feature-flag platforms (LaunchDarkly, Statsig) for gradual rollout.
