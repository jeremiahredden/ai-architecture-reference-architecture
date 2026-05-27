# Capability vs Cost vs Latency Tradeoffs

> **Audience.** Architects choosing a model for a new workload. Tech leads asked "why are we using Sonnet for that?" — and who want a defensible answer that isn't "because it's good." Anyone whose model selection has been ad-hoc and is now looking for a framework. **Scope.** The *architectural* framework for comparing models across capability, cost, and latency dimensions: the non-functional comparison matrix; the workload-driven prioritization; the calibration discipline for 2026 frontier and open-weight options; how the dimensions interact. Not the foundational frontier-vs-open-weights decision (see [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md)). Not the routing patterns (see [model-routing-and-tiering.md](./model-routing-and-tiering.md)). Not the migration playbook (see [model-migration-playbook.md](./model-migration-playbook.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Model selection happens often: a new feature ships; a new model is announced; a cost review prompts review; a latency target requires reconsideration. Each time, the team faces the same question: "which model fits this workload?"

Without a framework:

- Choice is by enthusiasm ("Sonnet is the best, use Sonnet").
- Choice is by inertia ("we used Sonnet last time").
- Choice is by simplification ("frontier is always right").

With a framework:

- Choice is by structured comparison: capability fit + cost fit + latency fit per workload.
- Trade-offs are explicit.
- Decisions are defensible.

The dimensions:

- **Capability:** does the model produce sufficiently-good output for this workload?
- **Cost:** is the per-call cost acceptable at expected volume?
- **Latency:** does the model meet the workload's latency target?

These three rarely align (the best capability is also the slowest and the most expensive). The architecture's job is to make the trade-off explicit and choose deliberately.

This document covers the framework: how to compare across the three dimensions; how workload priorities determine the right answer; the calibration data for 2026 era; and how the dimensions interact in production.

This document is opinionated about four things:

1. **No single model is universally best.** Capability, cost, latency form a 3-way trade-off; the right answer depends on the workload.
2. **Workload prioritization is the input.** "Cost-sensitive vs latency-sensitive vs quality-sensitive" determines the choice; the model is the output.
3. **The dimensions have non-linear interaction.** A 50% cost increase for 20% capability gain may or may not be worth it; the relationship is per-workload.
4. **Calibration data ages quickly.** Specific numbers in this document are illustrative; the framework is the durable contribution.

Structure: (2) the three dimensions defined; (3) the capability dimension; (4) the cost dimension; (5) the latency dimension; (6) the workload-driven prioritization; (7) the comparison matrix; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three dimensions defined

The framework's primitives.

### 2.1 Capability

What the model can do; how well it does it.

Sub-dimensions:

- **Reasoning ability.** Multi-step logic; chain-of-thought; tool use.
- **Domain knowledge.** Specific subject expertise (clinical, legal, code).
- **Context handling.** How well long context is utilized; recall of distant context.
- **Structured-output reliability.** JSON / schema adherence rate.
- **Multilingual coverage.** Language support and quality.
- **Tool-use reliability.** Correct tool selection and argument formatting.
- **Refusal calibration.** Appropriate refusal vs over-refusal.
- **Hallucination rate.** Fabrication of non-existent entities.

Each measurable; each varies by model.

### 2.2 Cost

What the model costs to use.

Sub-dimensions:

- **Per-input-token cost.**
- **Per-output-token cost.**
- **Per-call cost** (= input cost + output cost; depends on prompt + response shape).
- **Volume-based discounts** (commit tiers, batch APIs, prompt caching).
- **Indirect cost** (operational, lifecycle).

Total cost = function of all of these.

### 2.3 Latency

How fast the model responds.

Sub-dimensions:

- **Time to first token (TTFT).** For streaming; how long until output begins.
- **Tokens per second (TPS).** Throughput once generating.
- **Per-call P50, P95, P99.** Distribution of total latency.
- **Long-context latency.** How TTFT scales with input length.
- **Latency variance.** Spread of the distribution.

Workload latency depends on the call shape, not just the model.

### 2.4 The dimensions interact

A choice on one affects others:

- More capable model → more expensive per call (usually).
- More capable model → slower (usually; not always).
- Smaller / cheaper model → faster (usually).
- Bigger context → slower TTFT.

Per-workload navigation through the trade-off space.

### 2.5 The Pareto-optimal frontier

Plotted on a 3-axis chart, models form a Pareto frontier:

- Some models dominate others (better on all dimensions).
- Some are Pareto-optimal: trading one for another.

For each workload, pick the Pareto-optimal point that matches priorities.

### 2.6 The "we want best on all dimensions" reality check

No model is best on all. Settle for "good enough" on the dimensions that aren't the primary priority.

---

## 3. The capability dimension

How to evaluate model capability.

### 3.1 The benchmark trap

Published benchmarks (MMLU, GPQA, HumanEval, MTPB):

- Useful for high-level comparison.
- Poor proxy for workload-specific quality.
- Models train on benchmarks; performance can overstate generalization.

Treat as input, not as decision.

### 3.2 The workload eval

The right capability measure is your eval suite (cross-link to [ai-engineering-reference-architecture / eval-engineering / regression-eval-suites.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/regression-eval-suites.md)):

- Pass rate on the workload's specific test cases.
- Quality scores per workload-relevant dimension.
- Performance on edge cases.

This is the capability that matters.

### 3.3 The capability matrix per model

```
                 Sonnet 4.6   Haiku 4.5   Opus 4.7   GPT-4o   Llama 3 70B
Clinical reasoning  ✓✓✓        ✓          ✓✓✓✓       ✓✓✓     ✓✓
Structured output   ✓✓✓        ✓✓✓        ✓✓✓        ✓✓✓     ✓✓
Tool use            ✓✓✓        ✓✓         ✓✓✓✓       ✓✓✓     ✓✓
Multilingual        ✓✓✓        ✓✓         ✓✓✓✓       ✓✓✓     ✓✓
Long context        ✓✓✓        ✓✓         ✓✓✓✓       ✓✓✓     ✓✓
Hallucination rate  Low        Low        Very-low   Low      Medium
Cost (per 1M out)   $15        $5         $75        $15      ~$1 (self-host)
TTFT (median)       ~1.2s      ~0.5s      ~2.0s      ~1.0s    ~0.6s (self-host)
```

(Illustrative for 2026 era; verify for actual workload.)

### 3.4 The "good enough" threshold per workload

For each workload, define:

- Minimum acceptable capability.
- Stretch target.
- Specific dimensions that matter most.

If multiple models meet "good enough," pick on cost / latency.

### 3.5 The capability degradation tolerance

How much capability loss is acceptable for cost / latency gains?

- Clinical workflows: low tolerance (-3% capability is significant).
- Internal copilot: high tolerance (-15% is acceptable for 5x cost reduction).
- Customer chat: medium tolerance.

Per workload.

### 3.6 The capability calibration cadence

Models improve; recalibrate:

- New model release: re-eval.
- Quarterly: re-eval current models against current eval suite.
- Annual: full capability matrix refresh.

### 3.7 The "we used to need Opus; Sonnet is now enough" trajectory

Models improve over time:

- Workload required Opus 4 in 2024.
- Sonnet 4.6 in 2026 meets the bar.
- Migrate to Sonnet (cross-link to [model-migration-playbook.md](./model-migration-playbook.md)).

Annual review catches this.

---

## 4. The cost dimension

How to evaluate cost.

### 4.1 The per-call cost formula

```
cost_per_call = (input_tokens × input_rate) + (output_tokens × output_rate)
```

Per model, per call shape:

```
                 Sonnet 4.6   Haiku 4.5   Opus 4.7
Input rate       $3/M         $1/M        $15/M
Output rate      $15/M        $5/M        $75/M

For 1k input + 200 output:
  Sonnet:        $0.003 + $0.003 = $0.006
  Haiku:         $0.001 + $0.001 = $0.002
  Opus:          $0.015 + $0.015 = $0.030

5x difference from cheapest to most expensive.
```

(Illustrative; verify current rates.)

### 4.2 The volume scaling

At low volume: per-call cost dominates total.

At high volume: total cost = per-call × volume; small per-call differences become large total differences.

For 100k calls/day:
- Sonnet: $600/day → $18k/month.
- Haiku: $200/day → $6k/month.
- Opus: $3000/day → $90k/month.

The volume scale changes which option is feasible.

### 4.3 The prompt-caching discount

Most providers offer prompt-prefix caching (cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md §2](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md)):

- Cached tokens at 10% of rate (Anthropic).
- Cached tokens at 50% (OpenAI).
- Significant savings for stable-prefix workloads.

Include in cost estimation.

### 4.4 The batch-API discount

For batch-eligible workloads, ~50% discount (cross-link to [ai-engineering-reference-architecture / cost-and-finops / batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md)).

Affects total cost; affects feasibility.

### 4.5 The self-hosted economics

Self-hosted open-weight models:

- Per-call: near-zero (after infrastructure cost).
- Infrastructure: per-month for GPU + ops.
- Break-even at high volume.

Cross-link to [build-vs-buy-decision.md §4](./build-vs-buy-decision.md).

### 4.6 The "we picked the cheapest model" pitfall

Sometimes the cheapest model has quality cost:

- Net cost = direct cost + cost of bad outputs (rework, customer support, lost revenue).
- A "cheap" model that produces 20% bad outputs may have higher net cost than an expensive model with 2% bad.

Cost analysis must include quality impact.

### 4.7 The cost forecasting

For a new workload:

- Estimated volume.
- Per-call cost.
- Monthly cost = volume × per-call × 30.

Compare to budget; informs decision.

### 4.8 The per-tier portfolio

A platform may use multiple tiers:

- Haiku for high-volume routine.
- Sonnet for primary workloads.
- Opus for high-stakes complex tasks.

Per-workload selection; routing layer dispatches.

---

## 5. The latency dimension

How to evaluate latency.

### 5.1 The latency profile per model

```
                 Sonnet 4.6   Haiku 4.5   Opus 4.7
TTFT (median)    ~1.2s        ~0.5s       ~2.0s
TTFT (P99)       ~3.5s        ~1.5s       ~6.0s
TPS (median)     ~75          ~120        ~50
Per-call median  ~3.0s (250 out)  ~1.5s   ~7.0s

(Illustrative; verify for current provider performance.)
```

Smaller models faster; larger slower.

### 5.2 The workload latency budget

Per workload, define:

- Real-time interactive: P99 < 5s.
- Streaming chat: TTFT < 2s; total < 30s.
- Agent task: TTFT lenient; per-step < 15s.
- Batch: any reasonable.

The budget constrains model choice.

### 5.3 The long-context latency penalty

TTFT scales with input length:

- 1k input: ~1s TTFT.
- 8k input: ~3s TTFT.
- 32k input: ~7s TTFT.

For long-context workloads, latency is harder.

### 5.4 The streaming optimization

For interactive workloads, streaming reduces perceived latency:

- User sees first token in < 2s.
- Total response in 5-10s acceptable (user reads as it generates).

Without streaming: full response wait; latency feels longer.

### 5.5 The reasoning-model latency

Models with extended thinking (Claude Opus extended thinking, OpenAI o-series):

- TTFT during thinking: variable (10-60s).
- Total latency: long (30s+).
- Capability: highest.

Trade extreme: very capable, very slow.

### 5.6 The per-model latency variance

Some models have tight latency distribution; some have long tails. P99 / P50 ratio:

- Tight: ratio 3-5x.
- Wide: ratio 8-12x.

Predictability matters as much as median.

### 5.7 The "we need it faster than any model" case

Some workloads' latency budgets are below any frontier model:

- Self-host with smaller model (Llama 3 8B).
- Specialized inference providers (Groq, Cerebras) for ultra-low latency.

Cross-link to [build-vs-buy-decision.md](./build-vs-buy-decision.md).

---

## 6. The workload-driven prioritization

Each workload weights the three dimensions differently.

### 6.1 The workload-type taxonomy

**Real-time interactive (chat, copilot):**
- Latency: critical (must be fast).
- Capability: moderate.
- Cost: moderate.

**Agent task (multi-step, long-running):**
- Latency: tolerant (user expects wait).
- Capability: critical (errors compound).
- Cost: significant (multi-call).

**Batch processing (overnight, async):**
- Latency: irrelevant.
- Capability: workload-specific.
- Cost: critical (high volume).

**Safety-critical (clinical, financial, legal):**
- Capability: paramount.
- Cost: secondary.
- Latency: workload-specific.

**Background enrichment (embeddings, classification):**
- Cost: critical (high volume).
- Capability: workload-specific.
- Latency: irrelevant.

### 6.2 The priority matrix

```
Workload              Capability  Cost   Latency
─────────────────────────────────────────────────
Patient chat          M           M      H
Care Coordinator      H           M      M
Clinical decision     VH          L      M
Document ingestion    M           H      L
Embedding             M           VH     L
Internal copilot      L           L      L
```

Where: VH=Very High, H=High, M=Medium, L=Low.

Priorities drive choice.

### 6.3 The priority-to-model mapping

For each priority pattern, recommended model:

```
Patient chat (M-M-H):           Sonnet (capability + latency balance)
Care Coordinator (H-M-M):       Sonnet or Opus (capability matters)
Clinical decision (VH-L-M):     Opus (capability paramount)
Document ingestion (M-H-L):     Haiku or self-host (cost matters)
Embedding (M-VH-L):              Self-host (cost critical; capability adequate)
Internal copilot (L-L-L):        Haiku (any sufficient model)
```

### 6.4 The customer-tier overlay

Premium tenants may justify different models:

- Standard tenant: Sonnet.
- Premium tenant: Opus.

Tier-based routing (cross-link to [model-routing-and-tiering.md](./model-routing-and-tiering.md)).

### 6.5 The workload-priority evolution

Workload priorities change:

- Initial: cost-sensitive (startup).
- Maturing: capability-sensitive (paying customers).
- Mature: latency-sensitive (UX-critical).

Re-evaluate priorities periodically.

### 6.6 The "we don't know our priorities" diagnostic

When choosing for a new workload:

- What would the user accept as worst case for each dimension?
- Which dimension is the customer-facing differentiator?
- Which dimension hits engineering's budget hardest?

Answer these to derive priorities.

### 6.7 The trade-off worksheet

```
Workload: [name]
Volume: [calls/day]

Capability requirement: [description]
Cost target: $[per call] × volume = $[per month]
Latency target: P99 < [seconds]

Acceptable trade-offs:
  Capability: -[N]% acceptable for [N]x cost reduction.
  Cost: +[N]x acceptable for +[N]% capability.
  Latency: +[N]s acceptable for [reason].

Recommended model: [choice]
Reasoning: [why]
```

Document per workload; review annually.

---

## 7. The comparison matrix

For 2026, the calibrated matrix.

### 7.1 The hosted-frontier models

```
                  Sonnet 4.6   Haiku 4.5   Opus 4.7   GPT-4o   Gemini 2.5 Pro
─────────────────────────────────────────────────────────────────────────────
Capability        High         Medium      Very High  High     High
Cost (in/out $/M) $3/$15       $1/$5       $15/$75    $5/$15   $4/$12
TTFT median       1.2s         0.5s        2.0s       1.0s     1.1s
TPS               75           120         50         85       90
Long-context      200k         200k        200k       128k     2M
Structured output Excellent    Good        Excellent  Excellent Excellent
Tool use          Excellent    Good        Excellent  Excellent Good
Multilingual      Excellent    Good        Excellent  Excellent Excellent
Pricing-tier disc Yes (vol)    Yes         Yes        Yes      Yes
Prompt caching    90% disc     90% disc    90% disc   50% disc  Various
Batch API         50% disc     50% disc    50% disc   50% disc  Various
```

(Illustrative; verify with current data.)

### 7.2 The open-weight self-hosted

```
                  Llama 3 70B   Llama 3 8B   Mistral Large   Qwen 2.5 72B
──────────────────────────────────────────────────────────────────────────
Capability        Medium-High   Medium       Medium-High     High
Cost              Self-host     Self-host    Self-host       Self-host
GPU req           1x H100       1x A10       1x H100        1x H100
TPS (continuous)  ~25/req       ~80/req      ~30/req         ~25/req
Long-context      128k          128k         128k            128k
Structured output Good          Good         Good            Good
Tool use          Good          Moderate     Good            Good
Multilingual      Good          Moderate     Good            Excellent
Inference cost    ~$0.0001/call ~$0.00001    ~$0.0001        ~$0.0001
                  + GPU op cost + GPU op     + GPU op        + GPU op
```

(Illustrative; verify.)

### 7.3 The white-label inference

```
                  Together AI   Fireworks   Anyscale   DeepInfra
─────────────────────────────────────────────────────────────────
Available models  Many          Many        Many       Many
Cost              Per-vendor    Per-vendor  Per-vendor Per-vendor
Latency           Variable      Variable    Variable   Variable
Compliance        Per-vendor    Per-vendor  Per-vendor Per-vendor
SLA               Per-vendor    Per-vendor  Per-vendor Per-vendor
```

Variable; evaluate each.

### 7.4 The matrix interpretation

- No single model dominates.
- Per-workload, the right cell is different.
- Annual recalibration as new models ship.

### 7.5 The decision-tree

```
What's the workload's primary priority?
  Capability (highest quality required):
    Frontier? Opus, Sonnet, GPT-4.5+, Gemini Ultra.
    Open-weight viable? Llama 3 70B, Mistral Large.
  Latency (sub-second required):
    Smaller hosted model: Haiku, GPT-4o-mini.
    Self-host small model: Llama 3 8B.
    Specialized providers: Groq, Cerebras.
  Cost (volume-driven):
    Open-weight self-host: Llama, Mistral.
    White-label: Together, Fireworks.
    Batch API: 50% discount.
  Balanced (most production workloads):
    Hosted mid-tier: Sonnet, GPT-4o, Gemini Pro.
```

Use as starting point; refine with eval.

### 7.6 The "we tested and the best is X" recommendation pattern

For each Meridian feature:

- Document the eval results per candidate.
- Document the cost analysis.
- Document the latency analysis.
- Document the chosen model.

Decision evidence; defensible to leadership.

---

## 8. Worked Meridian example

Meridian's per-workload model selection demonstrates the framework.

### 8.1 The Care Coordinator workload

Priorities: High capability, medium cost, medium latency.

Eval matrix (Q1 2026):

```
                  Eval pass   Cost / 1k calls  P99 latency  Decision factor
Sonnet 4.6        96%         $25              7.0s         Best balance
Haiku 4.5         88%         $4               4.0s         Quality gap too large
Opus 4.7          98%         $200             14.0s        Cost too high
GPT-4o            93%         $30              6.5s         Acceptable; vendor diversification not justified
Llama 3 70B (S/H) 91%         $3 + ops         8.0s         Quality gap; ops complexity
```

Decision: Sonnet 4.6.

Reasoning: best capability that fits cost / latency budget; quality gap to Haiku is significant for clinical workflow.

### 8.2 The Patient API chat (US)

Priorities: Medium capability, medium cost, high latency-sensitivity.

Decision: Sonnet 4.6 for primary; Haiku for fallback.

Reasoning: streaming UX needs <2s TTFT; Sonnet meets it; Haiku is fallback when primary degrades (cross-link to [reliability-engineering/fallback-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fallback-patterns.md)).

### 8.3 The Patient API chat (CA)

Priorities: Same as US, but Anthropic not in `ca-central-1`.

Decision: Cohere Command R+.

Eval matrix:
```
                       Eval pass   Cost   P99 latency   Decision
Cohere Command R+ (CA) 92%         $20    5.0s          Best CA option
Self-hosted Llama (CA) 87%         $3 + ops  6.0s       Quality below threshold
```

Choice: Cohere; trade -4% capability for residency.

### 8.4 The document ingestion

Priorities: Low latency-sensitivity, medium capability, high cost-sensitivity (high volume).

Decision: Self-hosted Llama 3 70B.

Volume: 40k docs/day; 1.2M tokens/day.

Cost analysis:
- Hosted Sonnet: $30k/month.
- Self-hosted Llama 3 70B: $5k/month (infrastructure) + $4k (ops).
- Net savings: $21k/month.

Quality: Llama 3 70B at 91% pass rate; meets threshold (>85%).

Decision: self-host.

### 8.5 The clinical decision support

Priorities: Very high capability (safety-critical), low cost-sensitivity, medium latency.

Decision: Sonnet 4.6 with conservative quality SLO (97%).

Considered Opus 4.7 (more capable); decided not because:
- Sonnet 4.6 achieves 97% on the workload eval.
- Latency would worsen with Opus.
- Cost would increase ~5x.
- Quality improvement marginal.

If a future case demonstrates Opus is needed, re-evaluate.

### 8.6 The internal IT copilot

Priorities: Low everything (internal; low volume).

Decision: Haiku 4.5.

Reasoning: any sufficient model works; cheapest is fine.

### 8.7 The portfolio

```
Workload                    Model              Volume         Cost/mo
─────────────────────────────────────────────────────────────────────
Care Coordinator (US)       Sonnet 4.6         140k tasks    $42k
Patient API chat (US)       Sonnet 4.6         380k turns    $14k
Patient API chat (CA)       Cohere Command R+  85k turns      $4k
Document ingestion          Llama 3 70B (S/H)  40k docs/day   $9k (infra+ops)
Document embedding          BGE-large (S/H)    40k docs/day   $1k
Clinical decision support   Sonnet 4.6         8k queries     $3k
Analytics warehouse copilot Sonnet 4.6         12k queries    $8k
Internal IT copilot         Haiku 4.5          400 queries    $0.50
─────────────────────────────────────────────────────────────────────
Total                                                          $82k/month
```

If all on Sonnet 4.6: ~$130k/month. Hybrid saves ~$48k/month while meeting quality.

### 8.8 The annual review

Q1 2027: review the choices.

For each workload:

- Has the priority changed? (e.g., Care Coordinator's volume grew; cost more sensitive now)
- Has a better model emerged? (e.g., Sonnet 4.7 or 5.0)
- Has cost shifted?

Per workload, decide: keep, migrate, re-evaluate.

### 8.9 The decision documentation

Each workload's model selection is documented:

- Rationale.
- Eval data.
- Cost analysis.
- Date of last review.

The documentation is the institutional knowledge.

### 8.10 The lessons

- Per-workload choice produces a portfolio that aggregates to substantial savings.
- Annual review prevents drift to outdated choices.
- Eval data is the key input; subjective preferences are not.
- The hybrid mix is the right answer for most platforms.

---

## 9. Anti-patterns

### 9.1 The "Sonnet for everything" default

**Pattern.** Team uses Sonnet for all workloads; never considers cheaper / faster alternatives. Net: wasted spend.

**Corrective.** Per-workload selection per §6.

### 9.2 The "we picked the cheapest" rush

**Pattern.** Cost-only optimization; capability ignored; bad outputs cost more in support / rework.

**Corrective.** Net-cost analysis per §4.6.

### 9.3 The benchmark-driven choice

**Pattern.** Choose model based on published benchmark; production performance differs.

**Corrective.** Workload-specific eval per §3.2.

### 9.4 The "frontier is always right" reflex

**Pattern.** Always pick the most capable model; latency and cost ignored.

**Corrective.** Per-workload prioritization per §6.

### 9.5 The "this model is good in 2024" inertia

**Pattern.** Old choice never revisited; better options unavailable.

**Corrective.** Annual review per §6.5.

### 9.6 The "we'll pick at the end" deferral

**Pattern.** Build the system; pick the model at the end. Architecture is implicitly tied to the picked model; can't easily switch.

**Corrective.** Pick model with eval early; architect for that choice.

### 9.7 The mismatched priorities

**Pattern.** Workload is latency-sensitive; model choice optimizes for capability. Latency suffers.

**Corrective.** Match model to priorities per §6.

### 9.8 The "all workloads same model" simplification

**Pattern.** Platform standardizes on one model. Per-workload optimization left on the table.

**Corrective.** Per-workload selection per §6.

### 9.9 The forgotten cost-saving

**Pattern.** Workload could use Haiku; team uses Sonnet for safety; cost waste.

**Corrective.** Eval-driven downgrade; verify capability is sufficient.

### 9.10 The "no documentation of choice" gap

**Pattern.** Choice was made; rationale lost; future reviews can't decide.

**Corrective.** Documentation per §8.9.

---

## 10. Findings (sprint-assignable)

**ARCH-CVL-001 (P0). Model selection ad-hoc, not framework-driven.** Sub-optimal choices. Adopt framework per this document; per-workload analysis. Owner: AI platform + engineering management.

**ARCH-CVL-002 (P0). All workloads on one model.** Wasted spend or wasted capability. Per-workload selection per §6. Owner: AI platform.

**ARCH-CVL-003 (P0). No workload-specific eval data.** Model choice lacks evidence. Per §3.2; build eval per workload. Owner: AI platform + eval.

**ARCH-CVL-004 (P1). Cost analysis missing the prompt-caching and batch discounts.** Cost overestimated for cacheable workloads. Per §4.3, §4.4. Owner: AI platform + FinOps.

**ARCH-CVL-005 (P1). Per-tenant tier-based model selection absent.** Premium tier no different from standard. Per §6.4. Owner: AI platform + product.

**ARCH-CVL-006 (P1). Annual model-portfolio review not scheduled.** Drift to outdated choices. Per §6.5. Owner: AI platform + engineering management.

**ARCH-CVL-007 (P1). Decision documentation absent.** Rationale lost. Per §8.9. Owner: AI platform.

**ARCH-CVL-008 (P1). Capability matrix not maintained.** Comparison data stale. Per §7. Owner: AI platform.

**ARCH-CVL-009 (P2). Latency budget not aligned with model choice.** Sub-optimal latency. Per §5.2. Owner: AI platform + feature teams.

**ARCH-CVL-010 (P2). Reasoning-model latency not accounted for.** Workload suffers unexpectedly. Per §5.5. Owner: AI platform.

**ARCH-CVL-011 (P2). White-label inference options not evaluated.** Cost-savings opportunity missed. Per §7.3. Owner: AI platform.

**ARCH-CVL-012 (P2). Per-workload priority not explicit.** Subjective choices. Per §6.7. Owner: AI platform.

**ARCH-CVL-013 (P2). Custom-fine-tune evaluated only against same provider's models.** Misses self-hosted alternatives. Per §4.5 and [per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md). Owner: AI platform.

**ARCH-CVL-014 (P2). Long-context latency penalty not in calibration.** Long-context workloads have surprising slowness. Per §5.3. Owner: AI platform.

**ARCH-CVL-015 (P3). Streaming optimization not considered for latency-sensitive workloads.** UX latency higher than necessary. Per §5.4. Owner: AI platform + product.

**ARCH-CVL-016 (P3). Capability-degradation tolerance not documented per workload.** Future migrations lack guidance. Per §3.5. Owner: AI platform.

**ARCH-CVL-017 (P3). Volume scaling not in cost analysis.** Cost surprises at scale. Per §4.2. Owner: AI platform + FinOps.

**ARCH-CVL-018 (P3). Per-tier portfolio mix not visualized.** Hard to see the trade-offs. Per §4.8 and §8.7. Owner: AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Define workload priority matrix (§6).** Capability / cost / latency per workload.
- [ ] **Build workload-specific eval suites per §3.2.**
- [ ] **Run eval against candidate models for each workload.**
- [ ] **Cost analysis with volume scaling (§4.2).**
- [ ] **Latency analysis with workload's call shape (§5).**
- [ ] **Document per-workload choice (§8.9).**
- [ ] **Annual portfolio review (§6.5).**
- [ ] **Comparison matrix maintenance (§7).**
- [ ] **Per-tier overlay for premium / enterprise (§6.4).**
- [ ] **Decision template per workload (§6.7).**

---

## 12. References

**In this folder.**
- [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md) — foundational decision; capability vs cost vs latency is the operational framework.
- [model-routing-and-tiering.md](./model-routing-and-tiering.md) — routing implements per-workload choice.
- [model-catalogue-and-registry.md](./model-catalogue-and-registry.md) — catalogue tracks per-workload model selection.
- [build-vs-buy-decision.md](./build-vs-buy-decision.md) — broader option set including self-hosted.
- [model-migration-playbook.md](./model-migration-playbook.md) — companion; migrating to a new optimal choice.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — fine-tune as a capability option.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — worked example with multi-model portfolio.

**Sibling repos.**
- [ai-engineering-reference-architecture / eval-engineering / regression-eval-suites.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/regression-eval-suites.md) — eval suite construction.
- [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md) — caching effects on cost.
- [ai-engineering-reference-architecture / cost-and-finops / batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md) — batch discount.

**External.**
- Provider pricing pages (Anthropic, OpenAI, Google, Cohere) for current rates.
- Provider model documentation for capability claims.
- Vendor-published benchmarks (MMLU, GPQA, HumanEval, MTPB) for high-level comparison.
- LMArena, Chatbot Arena for crowd-sourced comparison.
