# Frontier vs Open Weights vs Fine-Tune

> **Audience.** Architects and engineering leadership setting model strategy for an AI portfolio. Tech leads deciding per-feature which model approach fits. Anyone whose first answer to "what model should we use?" is the model the team used last time. **Scope.** The three-way decision framework — frontier hosted, open-weight self-hosted, fine-tuned — and when to do none of these (yet). Cost, latency, capability, regulatory, and operational trade-offs. Not the per-provider model selection (a tactical detail). Not fine-tune engineering depth (sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `model-lifecycle/` folder). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Model selection is the most consequential and most-defaulted decision in AI architecture. Teams adopt their team-mate's recommendation, the vendor with the best sales rep, or "what the prototype used." Production reveals: the choice has 3-10× cost implications, material capability implications, regulatory implications (some models can process certain data; others can't), and operational implications (self-hosting vs managed are different worlds).

The three primary approaches in 2026:

- **Frontier hosted.** Anthropic Claude, OpenAI GPT, Google Gemini. Best general capability; managed; per-call cost; vendor risk.
- **Open weights self-hosted.** Llama, Mistral, Qwen, DeepSeek, et al. Lower per-call cost at scale; full control; operational burden; capability typically a generation behind frontier.
- **Fine-tuned.** A specific base model fine-tuned on the team's data. Captures domain knowledge; cheaper per-call than frontier on the same task; operational complexity; locked to base model.

A fourth, often-correct answer:

- **None of these (yet).** Use a frontier model with great prompting + retrieval + few-shot. Many teams skip this and over-engineer toward fine-tune or self-hosting prematurely.

The decision is per-workload, not platform-wide. Different features rightly use different approaches. The team's strategy is the portfolio of decisions; the platform supports multiple.

This document is opinionated about four things:

1. **The default is frontier hosted.** Best capability, fastest to ship, no operational burden. Deviate only when a specific reason justifies.
2. **Open weights for cost / data residency / regulatory.** When per-call cost compounds at scale, when data can't leave the team's environment, when regulation precludes hosted. Not for "we should have our own model."
3. **Fine-tune is a late-stage optimization.** Per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md): only when 20+ few-shot examples + stable use case + high volume + remaining quality gap. Premature fine-tune is operational baggage.
4. **The portfolio mixes approaches per workload.** Care Coordinator: frontier (clinical capability + BAA). Bulk classification: open-weight self-hosted (volume cost). Document type detection: fine-tuned small model (stable + high-volume). No single approach fits all.

Structure: (2) the three approaches characterised; (3) the "none of these yet" default; (4) the decision criteria per dimension; (5) the decision tree; (6) the per-workload mix (portfolio strategy); (7) migration between approaches; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three approaches characterised

The properties of each.

### 2.1 Frontier hosted

The provider runs the model; the team consumes via API. Models: Claude Sonnet / Opus, GPT-5, Gemini Ultra, etc.

**Capability.** Best available; provider invests in continuous improvement.

**Cost.** Per-call (input + output tokens). At low-medium volume: cheap. At high volume: scales linearly; can become expensive.

**Latency.** Network round-trip + provider's processing. 500ms-5s typical.

**Operational.** Zero infrastructure burden; vendor SLA; vendor support.

**Regulatory.** Depends on provider's terms (BAA for HIPAA, etc.). Some providers' terms are limiting for some workloads.

**Vendor risk.** Provider outage; pricing changes; model deprecation; terms changes.

**Best for.** Workloads where capability matters; volume is bounded; vendor terms are acceptable.

### 2.2 Open weights self-hosted

Models with public weights run on the team's infrastructure. Models: Llama-3 / 4, Mistral, Qwen, DeepSeek, Gemma, etc.

**Capability.** Typically a generation behind frontier; gap narrowing in 2026; some specialised tasks open-weights are competitive.

**Cost.** Infrastructure (GPUs, serving) + electricity. Per-call cost at high volume: 5-20× cheaper than frontier. At low volume: more expensive (idle infrastructure).

**Latency.** Co-located inference can be very fast (5-50ms per token); same as or faster than hosted.

**Operational.** Significant. GPU operations (provisioning, scheduling, scaling), model serving (vLLM, TGI, TensorRT-LLM), version management, monitoring.

**Regulatory.** Data stays in the team's environment; no provider data flow concerns.

**Vendor risk.** Low (open-source); but base model deprecation by upstream is possible.

**Best for.** High-volume workloads where per-call cost dominates; strict data-residency; teams with ML platform investment.

### 2.3 Fine-tuned

A specific base model (open-weight or hosted) fine-tuned on the team's data. The fine-tune captures domain patterns.

**Capability.** On the fine-tuned task: better than base. On other tasks: typically same as or worse than base.

**Cost.** Per-call cost typically lower than base model (specialised; can be smaller). Training cost: $1k-$50k+ per fine-tune.

**Latency.** Often faster than base (smaller model possible for narrow task).

**Operational.** Fine-tune lifecycle: data curation, training, eval, deployment, retraining as base model upgrades.

**Regulatory.** Depends on how fine-tuned (hosted provider's fine-tune; or self-hosted with open weights).

**Best for.** Stable, high-volume, narrow tasks where the fine-tune's specialization pays back the operational cost.

### 2.4 The three together

| Dimension | Frontier hosted | Open weights | Fine-tuned |
| --- | --- | --- | --- |
| Capability | Best general | Generation behind | Best on narrow task |
| Per-call cost (high volume) | High | Lower | Lower |
| Latency | Higher | Lower | Lower |
| Operational | Minimal | Significant | Moderate (lifecycle) |
| Time to ship | Days | Weeks-Months | Months |
| Vendor risk | High | Low | Mixed |
| Regulatory flexibility | Limited (provider terms) | High | Mixed |
| Volume threshold | < 1M / day usually | > 5M / day usually | Highly variable |

The matrix informs the decision; each row is a per-workload question.

### 2.5 The 2026 capability gap

Open-weight models in 2026 are materially better than 2024; the gap to frontier has narrowed:

- Top open-weight models on standard benchmarks: ~85-95% of frontier capability.
- On specialised tasks (code, domain-specific): often comparable.
- On novel reasoning / multi-step / agent shapes: frontier still leads.

The gap means the open-weight option is increasingly viable; the team's eval against their workload decides.

### 2.6 The "fine-tune outperforms base on the task" claim

Common pitch: "fine-tuning gives task-specific quality lift."

Often true on the narrow task; often false in production:

- Quality lift on benchmark: 5-15%.
- Quality lift in production: often 0-5% (the benchmark didn't reflect real workload).
- Production reveals new patterns the fine-tune doesn't cover; retraining cycle never catches up.

Per-workload eval before committing.

### 2.7 The hidden cost of self-hosting

Self-hosting open-weights costs more than the GPU bill:

- ML platform engineering (1-3 FTE for serious deployment).
- Model serving infrastructure (vLLM, TGI, or commercial).
- Capacity planning and scaling.
- Version management.
- Monitoring and on-call.
- Cost of slow model upgrades (the team's pace, not the open-source community's).

For a small team, the total cost may exceed frontier hosted at low-medium volume.

---

## 3. The "none of these yet" default

Often the right first answer.

### 3.1 What it means

The team's first version of an AI feature uses a frontier hosted model with good prompting + retrieval + few-shot. No self-hosting; no fine-tuning. Iterate the prompt and retrieval until quality plateaus.

### 3.2 Why this is the default

- Fastest to ship (days, not weeks).
- Best capability available.
- No operational burden.
- Iteration is fast (prompt changes don't require retraining).
- Reveals what the workload actually needs (only after production do the team's actual constraints surface).

Most AI features should start here; subsequent decisions (fine-tune, self-host) made when production data justifies.

### 3.3 The premature-optimization pattern

Teams that skip the default:

- "Let's fine-tune from day one." Months of fine-tune engineering before knowing the workload.
- "Let's self-host so we control costs." Operational burden before having the volume to justify.
- "Let's use multiple models." Routing complexity before the workload is understood.

These are all premature.

### 3.4 The graduation criteria

Graduate from "none of these yet" to fine-tune / self-host when:

- Volume is high enough that per-call cost matters.
- Quality plateau reached with prompt + retrieval + few-shot.
- Specific regulatory / data-residency requirement that hosted doesn't meet.
- Specific capability not in any hosted model.

Without one of these, stay on hosted.

### 3.5 The team's experience

Teams that started with "none of these yet" and iterated:

- Saved months of premature optimization.
- Discovered actual constraints (not imagined ones).
- Made better fine-tune / self-host decisions when they came.

Teams that went straight to fine-tune / self-host:

- Often discovered the chosen approach wasn't right.
- Rebuilt; lost the initial investment.

### 3.6 The patience discipline

The default requires patience. "We could fine-tune; let's see if we need to" feels less ambitious than "let's fine-tune now." But the patience produces better outcomes.

Engineering leadership models the patience: "we ship on hosted first; we revisit at production milestones."

### 3.7 The "hosted is just the prototype" anti-pattern

Some teams treat hosted as "the prototype" and have an immediate plan to migrate. The plan is premature; production reveals what the real plan should be.

Frame hosted as "the first production version"; subsequent decisions made with production data.

---

## 4. The decision criteria per dimension

The dimensions that drive the choice.

### 4.1 Capability

What the model can actually do for the workload.

- **Frontier hosted.** Best general capability; safest bet for novel / complex / multi-step.
- **Open-weight.** Comparable on benchmarks; per-workload eval may reveal gaps.
- **Fine-tuned.** Better on the narrow task; possibly worse on adjacent tasks.

For the workload's quality bar: which approach meets it?

### 4.2 Cost at projected volume

```
hosted_cost = volume × per_call_cost
self_hosted_cost = infrastructure + operational + (volume × marginal_cost)
fine_tuned_cost = training_cost + (volume × fine_tuned_per_call_cost)
```

Per-workload projection: at expected volume, what's each approach's monthly cost?

Crossover examples:
- Below 100k calls/day: hosted typically cheapest (avoids infrastructure overhead).
- Above 5M calls/day: self-hosted typically cheapest (per-call savings dominate).
- Between: depends on specifics.

### 4.3 Latency

Per-call latency:

- **Hosted.** 500ms-3s typical; depends on provider's load.
- **Self-hosted co-located.** 100ms-1s; controllable.
- **Fine-tuned (smaller model).** Often fastest; specialised model can be small.

For latency-critical features (sub-1s), self-hosted or fine-tuned may be needed.

### 4.4 Regulatory

- **Hosted.** Vendor's terms matter; BAA for HIPAA, DPA for GDPR, etc.
- **Self-hosted.** Data stays in team's environment; full regulatory control.
- **Fine-tuned.** Depends on where the fine-tune training and serving happen.

For some workloads (specific data classifications, specific geographies), regulatory forces the choice.

### 4.5 Vendor risk

- **Hosted.** Provider risks (outage, pricing, deprecation, terms).
- **Self-hosted.** Open-source community risk (less acute); team's own operational risk.
- **Fine-tuned (on hosted).** Vendor risk plus model-deprecation risk.

For some workloads, vendor risk is acceptable; for others, mitigation (multi-provider, self-hosted backup) is required.

### 4.6 Time to ship

- **Hosted.** Days. Sign up; integrate; ship.
- **Self-hosted.** Weeks to months. Infrastructure, model deployment, monitoring.
- **Fine-tuned.** Months. Data curation, training, eval, deployment.

For urgent features, hosted's speed often dictates the choice.

### 4.7 Operational capacity

- **Hosted.** Minimal team capacity needed.
- **Self-hosted.** 1-3 FTE ML platform engineering.
- **Fine-tuned.** 1-2 FTE ML / data engineering for ongoing.

Smaller teams: hosted by default; self-host / fine-tune when capacity is in place.

### 4.8 The team's strategic position

- Early-stage company: hosted (speed; capital efficiency).
- Mid-stage with ML team: hosted for most; self-host for specific cases.
- Large enterprise with ML platform: portfolio of all three.

The team's stage matters; the right answer evolves.

---

## 5. The decision tree

Walk through each criterion in order.

### 5.1 The tree

```
Is the workload at significant volume (> 1M calls/day projected)?
  No → frontier hosted (default)
  Yes → continue

Is per-call cost a material concern at projected volume?
  No → frontier hosted
  Yes → continue

Are there regulatory / data-residency requirements that hosted doesn't meet?
  Yes → self-hosted (open weights) is the answer
  No → continue

Is the workload narrow and stable (same task type repeatedly)?
  No → self-hosted (open weights, general-purpose)
  Yes → continue

Has the team done eval showing fine-tune meaningfully outperforms hosted?
  No → hosted or self-hosted; defer fine-tune
  Yes → continue

Is the team's ML capacity sufficient for fine-tune lifecycle?
  No → hosted; build capacity first
  Yes → fine-tuned

Default at any uncertain branch: frontier hosted with good prompting.
```

### 5.2 The "always hosted for the first version" rule

Regardless of where the tree goes, the first version should be hosted. The tree's destination is for "after we ship and learn."

### 5.3 The annual re-evaluation

The decision evolves:

- Open-weight capability improves (the threshold for self-hosting moves down).
- Vendor pricing changes (the cost crossover shifts).
- Workload's volume changes (more or less reason for per-call savings).
- Team's capacity grows (more options available).

Annual review per workload; revise.

### 5.4 The "different workloads, different answers" portfolio

A team typically has 3-5 AI workloads, each with its own answer. The portfolio:

- Care Coordinator: frontier hosted (clinical capability + BAA).
- Patient API AI Assist: frontier hosted (volume manageable; capability needed).
- Analytics Copilot: frontier hosted for SQL gen; self-hosted small classifier for intent.
- Patient Summary: fine-tuned (high volume; narrow task).
- Bulk document classification: self-hosted (volume + cost).

No single answer; the portfolio reflects per-workload analysis.

### 5.5 The "we standardised on X" failure

Some teams pick one approach platform-wide ("we're a self-hosted shop"; "everything goes through OpenAI"). The standardisation simplifies operations but produces wrong answers for some workloads.

Better: a recommended default + room for justified per-workload deviation.

### 5.6 The "let's pilot all three" anti-pattern

Some teams try all three approaches in parallel for the same workload. The result is three half-built systems instead of one good one.

Pick one based on the tree; iterate; revisit later if needed.

---

## 6. The per-workload mix (portfolio strategy)

The platform decisions that support multiple approaches.

### 6.1 The platform supports all three

The AI gateway (per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md)) abstracts model location:

- Hosted models via provider SDKs.
- Self-hosted via internal API (consistent with hosted's interface).
- Fine-tuned via deployment infrastructure.

The agent feature code doesn't care where the model runs; the gateway routes.

### 6.2 The model catalogue per workload

Per workload, a documented model choice:

```yaml
features:
  care-coordinator:
    model_primary: anthropic/claude-sonnet-4-6
    rationale: "Frontier capability + BAA; volume manageable"
    revisit: 2026-Q4
  patient-summary:
    model_primary: internal/patient-summary-finetune-v3
    rationale: "Stable high-volume; fine-tune saved 60% cost vs hosted"
    revisit: 2026-Q3
  bulk-classification:
    model_primary: internal/llama-3-70b-instruct
    rationale: "Volume cost; data residency"
    revisit: 2027-Q1
```

The catalogue is the strategy made concrete.

### 6.3 The per-tenant model differentiation

Some workloads have per-tenant differentiation:

- Premium tenants: frontier always.
- Standard tenants: tier-routed (frontier for hard; self-hosted for easy).
- Tenants with strict residency: self-hosted only.

The platform supports per-tenant policy (per [policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md)).

### 6.4 The capability portfolio

Beyond core text models:

- Embeddings: typically separate decision (often a hosted embedding provider like OpenAI or Voyage).
- Reranking: typically hosted (Cohere or Voyage).
- Specialised models (code, multilingual, medical): often hosted specialists or open-weight specialists per workload.

The portfolio includes these; not just the primary text model.

### 6.5 The migration paths

Workloads can migrate:

- Hosted → Self-hosted when volume warrants.
- Hosted → Fine-tuned when stability + volume + plateau warrant.
- Self-hosted → Hosted if open-weight quality plateaus and hosted catches up.
- Fine-tuned → Hosted if hosted's capability lift over the fine-tune.

The platform supports migrations; per [model-migration-playbook.md](./model-migration-playbook.md).

### 6.6 The eval that drives the portfolio

Per workload + per approach, eval:

- Quality on the team's golden set.
- Latency.
- Cost projection at expected volume.

The eval data informs the catalogue; updated quarterly.

### 6.7 The "platform team" model

For a multi-workload portfolio, a platform team is essential:

- Hosted provider relationships (contracts, BAAs, monitoring).
- Self-hosted infrastructure (GPU operations, serving).
- Fine-tune pipeline (training, deployment).
- The gateway that abstracts all three.

Without a platform team, each feature team rebuilds; coherence suffers.

### 6.8 The cost reporting per approach

Per [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md):

- Per-feature cost.
- Per-approach cost (hosted vs self-hosted vs fine-tune).
- Trends over time.

The reporting informs portfolio decisions.

---

## 7. Migration between approaches

When the answer changes.

### 7.1 The migration triggers

- **Volume growth.** Per-call hosted cost passes the crossover; self-hosted becomes cheaper.
- **Capability lift on alternative.** Open-weight capability catches up; reason to self-host.
- **Stability proven.** Workload's task is stable; fine-tune justified.
- **Regulatory change.** New requirement that hosted doesn't meet.
- **Cost shock.** Vendor pricing change.
- **Capacity available.** Team's ML platform team built; can take on self-hosting.

### 7.2 The migration steps (hosted → self-hosted)

1. **Eval baseline.** Hosted's quality, latency, cost on team's eval set.
2. **Candidate eval.** Self-hosted candidates (Llama-3, Mistral, etc.) on the same eval.
3. **Infrastructure prep.** GPU capacity; serving infrastructure (vLLM); monitoring.
4. **Parallel deployment.** Both serve traffic (canary); compare.
5. **Cutover.** Once parity is verified.
6. **Hosted as backup.** Keep hosted available for failover.

Multi-quarter effort.

### 7.3 The migration steps (hosted → fine-tuned)

1. **Eval baseline.**
2. **Data curation.** Training data (per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) and sibling [fine-tuning-operations.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/fine-tuning-operations.md)).
3. **Base model selection.** Which model to fine-tune.
4. **Training.** Run the fine-tune.
5. **Eval.** Quality, latency, cost vs hosted.
6. **Deployment.** Hosted vs self-hosted serving.
7. **Cutover.** When parity / lift is verified.

### 7.4 The "migration that wasn't" outcome

Sometimes the migration eval shows: hosted is still the right answer. Hosted's capability gain in the interim, or the open-weight's quality didn't reach parity, or the fine-tune's complexity wasn't worth the cost.

The eval was the investment; the migration is paused; revisit in 6-12 months.

### 7.5 The reverse migration

Sometimes the team migrates back:

- Self-hosted → Hosted: open-weight model is deprecated upstream; hosted's pricing dropped.
- Fine-tuned → Hosted: hosted's general capability now exceeds the fine-tune's specialised capability.

The platform supports reverse migrations too; per [model-migration-playbook.md](./model-migration-playbook.md).

### 7.6 The migration's risk

Migrations are operational risk:

- Quality regression in production.
- Cost surprise (the projection was wrong).
- Latency change.

Mitigations:

- Canary traffic.
- Shadow deployment (both serve; compare without enforcement).
- Tenant-by-tenant rollout.
- Rollback ready.

### 7.7 The "annual portfolio review" cadence

Once a year, the team reviews each workload's approach against current alternatives:

- Has the eval changed? (New models; new approaches.)
- Has the workload changed? (Volume; requirements.)
- Should we migrate?

Most workloads: no change. Some: migration triggered.

---

## 8. Worked Meridian example

Meridian's model strategy.

### 8.1 The portfolio

| Feature | Approach | Model | Rationale |
| --- | --- | --- | --- |
| Care Coordinator (clinical) | Frontier hosted | Claude Sonnet 4.6 / Opus 4.7 | Best capability; BAA; manageable volume |
| Patient API AI Assist | Frontier hosted (tier-routed) | Haiku / Sonnet | Volume bounded; capability needed |
| Analytics Copilot | Frontier hosted | Claude Sonnet (SQL gen) + Haiku (intake) | SQL quality matters; volume manageable |
| Patient API copilot | Frontier hosted | Sonnet + Voyage Code embeddings | Specialised embeddings only |
| Patient Summary (clinical history) | Fine-tuned | Internal/patient-summary-ft-v3 (base: Claude Haiku) | High volume + stable task; 60% cost saving |
| Bulk document classification | Self-hosted | Internal/llama-3-70b-instruct | Volume cost; on-prem GPUs |
| Email subject extraction (high volume) | Self-hosted (small classifier) | Internal/distilbert-finetune | Very high volume; cheap |
| Embeddings (across features) | Hosted | OpenAI text-embedding-3-large | Single decision; mature |
| Reranking | Hosted | Cohere rerank-3 | Specialised; mature |

Five approaches across the portfolio; each justified per-workload.

### 8.2 The default

"Frontier hosted (Anthropic Claude)" is the default for new features. Deviations require justification.

### 8.3 The annual review (Q4-25)

The team reviewed each workload:

- **Care Coordinator:** Stay frontier. Quality lift any switch would offer doesn't justify migration; BAA reliability matters.
- **Patient API AI Assist:** Stay frontier. Volume hasn't grown enough to warrant self-hosting; tier routing covers cost.
- **Patient Summary:** Stay fine-tuned. The Q2-25 fine-tune is delivering 60% cost saving; quality stable; retraining schedule for Q3-26 when new base model lands.
- **Bulk classification:** Stay self-hosted. Llama-3-70b is meeting quality; alternatives (Mistral) considered; no compelling switch.
- **Email subject:** Stay self-hosted small classifier. Volume justifies.

No migrations triggered; quarterly check-ins.

### 8.4 The Q2-25 fine-tune decision (patient summary)

Q1-25: Patient summary feature on hosted; cost $14k/month.
- Volume: 80k summaries/day; stable.
- Per-summary cost: $0.04 hosted.
- Projected: 18-month total: $250k.

Q1-25 fine-tune decision:
- Train Haiku-base fine-tune on curated 800 example summaries.
- Estimated training: $3k; ongoing retraining $1k/quarter.
- Projected per-summary cost: $0.016.
- Projected 18-month total: $100k + $4k training = $104k.
- Saving: ~$146k over 18 months.

Decision: fine-tune. Implemented Q2-25; production by end of Q2.

Outcome:
- Actual quality: +2% (slightly better than hosted; the fine-tune captured medical-summary conventions).
- Actual cost: $0.014 per summary (slightly better than projected).
- Payback: ~5 months.
- Retraining cadence: quarterly with new clinical-content batches.

### 8.5 The self-hosting bulk classification

Q3-24: bulk document classification at 30M docs/day.
- Per-classification cost (hosted Haiku): $0.0005.
- Daily: $15k.
- Monthly: $450k.
- Projection a year: $5.4M.

Q3-24 self-host decision:
- Llama-3-70b on internal GPU cluster.
- Infrastructure cost: ~$45k/month.
- Per-classification cost: ~$0.00007 (after fixed cost).
- Monthly total: ~$60k (vs $450k hosted).
- Saving: ~$390k/month; ~$4.7M/year.

Decision: self-host. Implemented Q4-24; production Q1-25.

Outcome:
- Quality: parity with hosted Haiku (on this narrow task).
- Cost: as projected.
- Operational: 1 FTE platform engineering ongoing; significant operational maturation.

### 8.6 The portfolio cost summary

Across all features:

- Hosted spend: ~$50k/month.
- Self-hosted infrastructure: ~$55k/month.
- Fine-tune training: ~$15k/year (amortised ~$1.3k/month).
- Total: ~$110k/month all-in.

Compared to "hosted everything" projection: ~$580k/month. The portfolio saves ~$470k/month through per-workload optimization.

### 8.7 The "frontier capability lift" reaction

Q1-26: new model release (Claude Opus 4.7) with material capability improvements.

The team's response:
- **Care Coordinator:** Eval against current Sonnet; tested Opus on hard cases. Decision: route hardest 5% to Opus; rest stay Sonnet. Small cost increase; quality lift on hardest cases.
- **Patient Summary fine-tune:** Re-eval against new Haiku. Decision: re-fine-tune scheduled Q3-26 if Haiku base improves materially.
- **Bulk classification:** No change (self-hosted; not affected by hosted updates).

The strategy responds to capability lift selectively; not "switch everything to the new model."

### 8.8 What worked

- **Default to hosted + iteration.** Saved months on each feature's first version.
- **Per-workload eval before deviation.** No premature optimization.
- **Fine-tune when criteria met.** Patient summary's payback validates the criteria.
- **Self-host when volume warrants.** Bulk classification's saving is real.
- **Annual review.** Decisions stay current.

### 8.9 What didn't work initially

- **Early self-host attempt (Q2-24).** Attempted self-hosting for Care Coordinator. Quality was 8% below hosted; operational complexity high. Reverted to hosted. Lesson: don't self-host high-stakes features prematurely.
- **Fine-tune for analytics-copilot SQL gen (Q1-25).** Proposed; eval showed marginal lift; abandoned. The hosted Sonnet + good prompt was sufficient. Lesson: fine-tune isn't always the answer.
- **Multi-provider hosted (early attempt).** Tried using OpenAI for some, Anthropic for others. Operational complexity high; standardised on Anthropic primary + OpenAI failover.

---

## 9. Anti-patterns

### 9.1 "Self-host everything"

The team decides "we want control"; self-hosts everything. Operational complexity high; team's other priorities suffer.

**Corrective.** Per-workload decision per section 5.

### 9.2 "Fine-tune from day one"

First version is fine-tuned. Months of fine-tune engineering before knowing workload. Often discovers the fine-tune wasn't needed.

**Corrective.** Hosted first per section 3; fine-tune when criteria justify.

### 9.3 "Multi-provider for diversification of one workload"

Same workload runs on both Anthropic and OpenAI in parallel. Cost doubles; operational complexity; rarely justified for a single workload.

**Corrective.** Single primary + failover per [multi-model-orchestration.md](../reference-patterns/multi-model-orchestration.md).

### 9.4 "Platform-wide standardisation"

One approach for all workloads. Wrong answer for some.

**Corrective.** Default + room for deviation per section 5.5.

### 9.5 "No portfolio review"

Decisions made once; never revisited. Conditions change but approach doesn't.

**Corrective.** Annual review per section 7.7.

### 9.6 "Cost crossover not measured"

Team self-hosts on belief; never measures actual cost vs hosted equivalent.

**Corrective.** Per-workload cost tracking per section 6.8.

### 9.7 "Frontier hosted prematurely abandoned"

Team migrates to self-host or fine-tune before hosted is exhausted as an option.

**Corrective.** Quality / cost plateau before considering migration per section 3.4.

### 9.8 "Capability gap ignored"

Open-weight chosen for cost; quality regression on production traffic ignored.

**Corrective.** Per-workload eval per section 4.1; quality bar must be met.

---

## 10. Findings (sprint-assignable)

### ARCH-FOF-001 — Severity: Critical
**Finding.** Platform-wide approach forced (all self-host or all fine-tune); wrong answer for some workloads.
**Recommendation.** Per-workload decision per section 5; default + deviations.
**Owner.** architecture + engineering leadership, sprint N+1.

### ARCH-FOF-002 — Severity: Critical
**Finding.** First AI feature being built as fine-tune; premature optimization.
**Recommendation.** Hosted first per section 3.2 / 3.6; fine-tune later if justified.
**Owner.** architecture + feature team, sprint N+1.

### ARCH-FOF-003 — Severity: High
**Finding.** Self-hosting attempted without ML platform team; operational burden overwhelming.
**Recommendation.** Per section 4.7; capacity assessment before self-host.
**Owner.** engineering leadership + architecture, sprint N+2.

### ARCH-FOF-004 — Severity: High
**Finding.** Per-workload cost not tracked by approach; can't validate decisions.
**Recommendation.** Per-approach cost reporting per section 6.8 / [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md).
**Owner.** ai-platform-eng + finance, sprint N+2.

### ARCH-FOF-005 — Severity: High
**Finding.** Annual portfolio review not performed; decisions stale.
**Recommendation.** Annual review per section 7.7.
**Owner.** architecture, sprint N+2.

### ARCH-FOF-006 — Severity: High
**Finding.** Migration plans for approach changes informal; future migrations would be risky.
**Recommendation.** Per section 7; documented migration patterns; per [model-migration-playbook.md](./model-migration-playbook.md).
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-FOF-007 — Severity: High
**Finding.** Per-workload eval missing; approach choices on speculation.
**Recommendation.** Per section 4.1; eval per workload before commitment.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-FOF-008 — Severity: Medium
**Finding.** Fine-tune in production with no re-train schedule; base model upgrades not absorbed.
**Recommendation.** Per [model-migration-playbook.md](./model-migration-playbook.md); re-train cadence; deprecation watch.
**Owner.** ml-eng + ai-platform-eng, sprint N+3.

### ARCH-FOF-009 — Severity: Medium
**Finding.** Self-hosted models not version-controlled; production might run unverified version.
**Recommendation.** Per [model-catalogue-and-registry.md](./model-catalogue-and-registry.md); version pinning.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-FOF-010 — Severity: Medium
**Finding.** Vendor risk for primary hosted provider not mitigated; outage means feature outage.
**Recommendation.** Failover per [multi-model-orchestration.md](../reference-patterns/multi-model-orchestration.md) section 5.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-FOF-011 — Severity: Medium
**Finding.** Per-tenant model differentiation not supported; premium tenants get same as standard.
**Recommendation.** Per section 6.3; per-tenant policy.
**Owner.** product + ai-platform-eng, sprint N+3.

### ARCH-FOF-012 — Severity: Medium
**Finding.** Self-hosted GPU infrastructure capacity-planned for steady-state; surges produce degradation.
**Recommendation.** Per section 7; capacity headroom for surges.
**Owner.** ai-platform-eng + ops, sprint N+3.

### ARCH-FOF-013 — Severity: Medium
**Finding.** Capability portfolio (embeddings, reranking, specialised) not deliberately chosen; defaults across workloads.
**Recommendation.** Per section 6.4; per-need decisions.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-FOF-014 — Severity: Medium
**Finding.** Fine-tune ROI not validated post-launch; assumed savings may not have materialised.
**Recommendation.** Per section 8.4 / 8.5; post-launch ROI tracking.
**Owner.** ai-platform-eng + finance, sprint N+4.

### ARCH-FOF-015 — Severity: Low
**Finding.** New AI feature design reviews don't include model strategy section.
**Recommendation.** Design review template includes model approach + rationale.
**Owner.** architecture, sprint N+4.

### ARCH-FOF-016 — Severity: Low
**Finding.** Platform team capacity for self-host / fine-tune not staffed; future migrations infeasible.
**Recommendation.** Platform team sized per section 6.7.
**Owner.** engineering leadership, sprint N+5.

### ARCH-FOF-017 — Severity: Low
**Finding.** "Hosted as backup" pattern not implemented; if self-hosted fails, no failover.
**Recommendation.** Per section 7.2; hosted retained as backup.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-FOF-018 — Severity: Low
**Finding.** Open-weight quality landscape not tracked; new model releases not evaluated.
**Recommendation.** Quarterly monitoring of open-weight releases; eval when materially better.
**Owner.** ai-platform-eng + ml-eng, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team setting model strategy:

- [ ] **Sprint 0 — workload inventory.** What AI features exist or are planned?
- [ ] **Sprint 0 — per-workload analysis.** Per section 4 / 5; documented.
- [ ] **Sprint 0 — default selection.** What's the team's default? (Most teams: frontier hosted.)
- [ ] **Sprint 1 — platform abstraction.** Gateway per [ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md); abstracts approach.
- [ ] **Sprint 1 — model catalogue.** Per section 6.2; per-workload documented.
- [ ] **Sprint 2 — cost tracking by approach.** Per [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md).
- [ ] **Sprint 2 — failover capability.** Per [multi-model-orchestration.md](../reference-patterns/multi-model-orchestration.md).
- [ ] **Sprint 3 — eval per workload.** Quality + latency + cost per approach.
- [ ] **Sprint 3 — first deviation from default.** If any workload warrants.
- [ ] **Sprint 4 — migration patterns documented.** Per section 7.
- [ ] **Sprint 4 — platform team established.** Per section 6.7 if multi-approach.
- [ ] **Annually — portfolio review.** Per section 7.7.

For a team retrofitting:

- [ ] **Sprint 0 — audit.** What's running; what's the cost; what's the rationale.
- [ ] **Sprint 1 — fix worst gap.** Often premature self-host or fine-tune.
- [ ] **Sprint 2 — eval validation.** Are current approaches justified?
- [ ] **Sprint 3 — migration if needed.** Per section 7.

A team that completes the sequence has model strategy that's deliberate, justified, and current. A team that doesn't has approaches by inertia.

---

## 12. References

- [model-routing-and-tiering.md](./model-routing-and-tiering.md) — tier routing within an approach.
- [model-catalogue-and-registry.md](./model-catalogue-and-registry.md) — catalogue mechanism.
- [build-vs-buy-decision.md](./build-vs-buy-decision.md) — broader build-vs-buy framing.
- [model-migration-playbook.md](./model-migration-playbook.md) — migration depth.
- [capability-vs-cost-vs-latency-tradeoffs.md](./capability-vs-cost-vs-latency-tradeoffs.md) — three-way tradeoff depth.
- [reference-patterns/multi-model-orchestration.md](../reference-patterns/multi-model-orchestration.md) — orchestration across approaches.
- [reference-patterns/rag-architecture-decision-guide.md](../reference-patterns/rag-architecture-decision-guide.md) — RAG decisions interact.
- [reference-patterns/pattern-anti-patterns.md](../reference-patterns/pattern-anti-patterns.md) — anti-pattern #3 (fine-tune as first move).
- [guardrails-and-policy-architecture/ai-gateway-pattern.md](../guardrails-and-policy-architecture/ai-gateway-pattern.md) — gateway abstracts approaches.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — model approval as policy.
- [multi-tenancy-and-isolation/](../multi-tenancy-and-isolation/) — per-tenant model considerations.
- Sibling repo: [ai-engineering-reference-architecture/model-lifecycle/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle) — model lifecycle engineering depth.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) — fine-tune crossover criteria.
- Sibling repo: [ai-engineering-reference-architecture/cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-approach cost attribution.
- Anthropic Claude, OpenAI GPT, Google Gemini — frontier hosted providers.
- Llama, Mistral, Qwen, DeepSeek — open-weight model families.
- vLLM, Text Generation Inference, TensorRT-LLM — open-weight serving frameworks.
- Anthropic fine-tune, OpenAI fine-tune — hosted fine-tune offerings.
