# Token Economics

> **Audience.** Architects sizing a new AI feature whose CFO will ask "what will this cost?" Tech leads whose AI bill is one number but they need to explain the components. Anyone whose cost surprises arrive because the per-call structure wasn't modeled. **Scope.** The *architectural* model of AI cost: per-call cost breakdown (input, output, cached, tool, embedding, reranker); model-tier cost curves; cost-per-conversation, per-user, per-tenant, per-feature accounting; cost-budget-as-SLO. Not the engineering attribution mechanics (see [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md)). Not the broader strategy (see [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI cost is engineering's problem. Architects who treat it as a finance concern after launch find that the cost shape is fixed by the architecture they shipped. Architects who model the token economics at design time find that they have substantial cost-control headroom they can deploy as needed.

The per-call cost shape is the architectural artifact: how many tokens flow in; how many flow out; how many are cached; how many tool calls happen; what model serves; what discount applies. Each is a lever.

This document covers the model:

- Per-call cost breakdown (the formula).
- Model-tier cost curves (what each tier costs).
- Per-conversation, per-user, per-tenant, per-feature aggregation.
- Cost-budget-as-SLO (treating cost as a reliability dimension).

The engineering-side mechanics (telemetry, attribution, reconciliation) live in the sibling [cost-and-finops/cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md). This document is the architectural framing.

This document is opinionated about four things:

1. **Cost is part of the architecture, not added later.** The per-call cost shape is set by design decisions; making the right decisions requires modeling them at design time.
2. **The aggregation dimensions all matter.** Per-feature, per-tenant, per-user, per-conversation — each surfaces different patterns; design for all four.
3. **Cost should be treated as an SLO.** Like latency and availability, cost has targets, budgets, burn rates, and triggers. Make it operational.
4. **The model-tier cost curves are non-linear.** A 10x cost-difference between tiers compounds at volume; the right tier per workload is a 5-10x architectural lever.

Structure: (2) the per-call cost breakdown; (3) the model-tier cost curves; (4) per-conversation cost; (5) per-user, per-tenant, per-feature aggregation; (6) cost-budget-as-SLO; (7) cost sensitivity to architectural decisions; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The per-call cost breakdown

The formula and its components.

### 2.1 The base formula

```
per_call_cost = (input_tokens × input_rate) + (output_tokens × output_rate)
              + (tool_call_costs)
              + (cache_write_cost) - (cache_hit_discount)
              + (other components, if applicable)
```

The dominant terms are input and output tokens at the model's rates.

### 2.2 The input-tokens component

Input tokens come from:

- The system prompt.
- Retrieved context (RAG documents).
- Conversation history.
- The current user message.
- Tool definitions.
- Few-shot examples.

Each contributes; total is the sum.

```
typical care-coordinator call:
  system_prompt:        500 tokens
  tool_definitions:     1500 tokens
  retrieved_context:    3000 tokens (5 documents)
  conversation_history: 500 tokens
  current_message:      100 tokens
  ─────────────────────────────
  Total input:          5600 tokens

At Sonnet rate ($3/M):
  Input cost:           5600 × $3/1,000,000 = $0.0168
```

### 2.3 The output-tokens component

Output is whatever the model produces:

- The response text.
- Tool calls (in agent workloads).

For typical care-coordinator response:
```
typical care-coordinator output:
  Response text:        400 tokens
  Tool calls (3):       300 tokens
  ─────────────────────────────
  Total output:         700 tokens

At Sonnet rate ($15/M output):
  Output cost:          700 × $15/1,000,000 = $0.0105
```

### 2.4 The cache-hit discount

For prompts with stable prefixes, cache hits reduce cost:

```
With prompt-prefix caching:
  Cache write (first call): 5000 prefix tokens × $3.75/M = $0.01875
  Cache hit (subsequent): 5000 × $0.30/M = $0.0015
  Variable suffix: 600 × $3/M = $0.0018
  
  First call cost: $0.01875 + $0.0018 + output = $0.0306
  Subsequent: $0.0015 + $0.0018 + output = $0.0138
  Savings: 55% per cached call
```

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).

### 2.5 The tool-call cost (agent workloads)

Each tool call is its own LLM cost:

```
Agent task with 5 LLM calls:
  Call 1 (initial reasoning): 4000 input, 200 output = $0.015
  Call 2 (after tool 1): 4500 input, 250 output = $0.0173
  Call 3 (after tool 2): 5200 input, 200 output = $0.0186
  Call 4 (after tool 3): 5800 input, 300 output = $0.022
  Call 5 (final response): 6300 input, 400 output = $0.025
  ──────────────────────────────────────────
  Total: $0.098 per agent task
```

Cross-link to [ai-engineering-reference-architecture / agent-engineering / agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md).

### 2.6 The embedding component

For RAG workloads:

```
Embedding cost:
  Documents in corpus: 100,000 docs × 500 tokens avg = 50M tokens
  Embedding rate (Anthropic embed-large, hypothetical): $0.10/M
  One-time corpus embedding: $5
  
  Ongoing: 40k new docs/day × 500 tokens × $0.10/M = $2/day
  Query embedding: 1k queries/day × 30 tokens × $0.10/M = $0.003/day
```

Embedding cost is typically small compared to LLM cost.

### 2.7 The reranker component

For workloads with reranking:

```
Reranker cost:
  Retrieved candidates: 50 documents
  Reranker score per pair: $0.001 per 1k tokens
  Per query: ~$0.005
```

Variable; depends on reranker choice.

### 2.8 The "all-in" per-call cost

For a complete picture, sum all components:

```
Care Coordinator full per-task:
  5 LLM calls × avg $0.02 = $0.10
  Vector retrieval (4 queries): $0.02
  Reranker (if used): $0.005
  Embedding (per turn): $0.0001
  Tool call infrastructure (negligible): ~$0
  ─────────────────────────────────────
  Total per task: ~$0.13
```

The "all-in" is what determines aggregate cost.

---

## 3. The model-tier cost curves

How cost varies across model tiers.

### 3.1 The frontier hosted tier

Sonnet-class / GPT-4o-class:

- Input: ~$3-5 / 1M tokens.
- Output: ~$15-20 / 1M tokens.
- Typical per-call cost: $0.005-0.05 (depending on prompt size).

### 3.2 The cheap-hosted tier

Haiku-class / GPT-4o-mini class:

- Input: ~$1 / 1M tokens.
- Output: ~$5 / 1M tokens.
- Typical per-call cost: $0.001-0.02.

5-10x cheaper than frontier; quality lower but often sufficient.

### 3.3 The premium tier

Opus-class / GPT-4.5 class:

- Input: ~$15 / 1M tokens.
- Output: ~$75 / 1M tokens.
- Typical per-call cost: $0.03-0.30.

3-5x more expensive than frontier; for highest-quality workloads.

### 3.4 The reasoning tier

Models with extended thinking (Opus extended thinking, o1):

- Output may be 10x normal (lots of thinking tokens).
- Per-call cost: $0.10-1.00.
- Justified only for hard reasoning tasks.

### 3.5 The open-weight self-hosted tier

Llama-class self-hosted:

- Direct cost: near-zero (after infrastructure).
- Effective per-call: ~$0.0001-0.001 depending on infrastructure utilization.
- Plus operational cost amortized.

### 3.6 The white-label inference tier

Together / Fireworks / Replicate hosting open weights:

- Cost: between hosted-direct and self-hosted.
- Typical: $0.0005-0.005 per call.

### 3.7 The tier cost ratios

```
Tier comparison (per-call, typical):
  Haiku-class:      $0.005
  Sonnet-class:     $0.025      (5x Haiku)
  Opus-class:       $0.075      (15x Haiku)
  Self-hosted Llama: $0.0005    (1/10 Haiku; 1/150 Opus)
```

5-150x cost range across tiers for similar workloads.

### 3.8 The cost-tier-vs-capability curve

Each step up in tier:

- Cost increases (5-15x per step).
- Capability increases (3-15% per step at typical workloads).

The marginal capability gain is small; the cost gain is large. Per-workload, the right tier depends on whether the capability gain justifies the cost.

---

## 4. Per-conversation cost

How cost aggregates within a conversation.

### 4.1 The conversation shape

For a multi-turn conversation:

- Turn 1: full system prompt + history (empty) + user message = ~1000 input + 200 output.
- Turn 2: full system + history (1 turn) + new message = ~1300 input + 200 output.
- Turn 3: full system + history (2 turns) + new message = ~1700 input + 200 output.
- ...
- Turn N: full system + history (N-1 turns) + new message.

Input grows with each turn; output stays comparable.

### 4.2 The cumulative cost

```
Per-turn cost grows ~linearly:
  Turn 1: $0.005
  Turn 5: $0.012
  Turn 10: $0.025
  Turn 20: $0.045

For a 10-turn conversation: ~$0.15 total.
```

### 4.3 The history-truncation strategy

To control conversation cost:

- Truncate history to last N turns.
- Summarize older history.
- Drop irrelevant turns.

Each is a cost-vs-quality trade-off.

### 4.4 The prompt-caching benefit

With prompt caching:
- System prompt is cached (90% discount on hits).
- History up to last turn can be cached.
- Only the new turn is uncached.

```
With caching:
  Turn 5: 90% of 1300 cached + new turn = effectively ~$0.003
  Cumulative through Turn 10: ~$0.06 (vs $0.15 without)
```

60% savings.

### 4.5 The "long conversation" cost concern

For 50-turn conversations:

- Without caching: ~$1+ per conversation.
- With caching: ~$0.30.

Long conversations get expensive; design with caching in mind.

### 4.6 The per-conversation budget

Set a budget:

- Free tier: $0.10 per conversation.
- Premium tier: $1.00 per conversation.

When conversation cost approaches the budget:
- Truncate history.
- Switch to cheaper model.
- Surface "this conversation has been long; summarizing for efficiency" UX.

---

## 5. Per-user, per-tenant, per-feature aggregation

The dimensions for cost analysis.

### 5.1 The aggregation hierarchy

Per-call → per-conversation → per-user → per-tenant → per-feature → aggregate.

Each level surfaces different patterns.

### 5.2 Per-user cost

Distribution analysis:

- Median user: $0.50/month.
- P90 user: $5/month.
- P99 user: $50/month.
- Outlier (1 in 10k): $500/month (probably misconfigured or abuse).

The distribution shape matters; tail catches outliers.

### 5.3 Per-tenant cost

For multi-tenant platforms:

- Per-tenant aggregate over the period.
- Compared to budget.
- Used for chargeback (cross-link to [ai-engineering-reference-architecture / cost-and-finops / per-tenant-cost-control.md §7](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/per-tenant-cost-control.md)).

### 5.4 Per-feature cost

Each feature's monthly aggregate:

- Top 3 features typically 60-80% of total spend.
- Bottom features rounding error.
- Per-feature budget set; tracked.

### 5.5 The aggregate dashboard

```
Aggregate spend (last 30 days): $42,000

Per-feature breakdown:
  Care Coordinator:        45%  ($18,900)
  Patient API chat:        25%  ($10,500)
  Document ingestion:      15%  ($6,300)
  Analytics copilot:       8%   ($3,360)
  Embedding:               5%   ($2,100)
  Internal copilot:        2%   ($840)

Per-tenant breakdown:
  Enterprise tenant 1:     30%  ($12,600)
  Premium tenants (2):     22%  ($9,240)
  Standard tenants (5):    35%  ($14,700)
  Free trial tenants (4):  10%  ($4,200)
  Internal:                3%   ($1,260)

Per-model breakdown:
  Sonnet 4.6:              68%  ($28,560)
  Haiku 4.5:               10%  ($4,200)
  Self-hosted Llama:       12%  ($5,040)
  Cohere Command R+:       5%   ($2,100)
  Embedding (self-host):   5%   ($2,100)
```

Multiple views; each surfaces different patterns.

### 5.6 The cost-anomaly detection

Per-dimension:

- Per-feature spike: a feature's spend doubled vs baseline.
- Per-tenant spike: a tenant's spend up 5x.
- Per-user spike: an outlier user.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-dashboards-and-alerts.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-dashboards-and-alerts.md).

### 5.7 The forecast across dimensions

```
30-day forecast:
  Care Coordinator: ~$20k (5% growth from current trajectory).
  Patient API chat: ~$12k (15% growth, new feature launching).
  Document ingestion: stable ~$6k.
  
  Total forecast: ~$48k (vs current $42k; +14%)
```

Per-feature forecast aggregates to total.

---

## 6. Cost-budget-as-SLO

Treating cost as a reliability dimension.

### 6.1 The cost SLO

```
care-coordinator-cost-slo:
  monthly_budget_usd: 20000
  daily_warning_threshold: 80% of pace
  daily_hard_threshold: 100% of pace
  alert_burn_rate_threshold: 2.0x
```

Like latency / availability SLO; cost has targets.

### 6.2 The burn-rate alert

```
burn_rate = (spend_so_far / period_elapsed) / (budget / period_total)
```

When > 2.0, alert. Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-dashboards-and-alerts.md §3](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-dashboards-and-alerts.md).

### 6.3 The cost-budget-as-circuit-breaker

When budget is exceeded:

- Circuit-breaker fires.
- Feature serves degraded mode (cross-link to [ai-engineering-reference-architecture / reliability-engineering / degraded-mode-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md)).
- New requests refused or rate-limited.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / cost-budget-circuit-breaker.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-budget-circuit-breaker.md).

### 6.4 The stop-shipping discipline

When cost budget is exhausted:

- New cost-increasing features paused.
- Reliability work prioritized.

Cross-link to [ai-engineering-reference-architecture / reliability-engineering / fault-budgets-for-ai.md §8](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fault-budgets-for-ai.md).

### 6.5 The SLO review

Monthly:

- Was the SLO met?
- If not, why? Was the budget wrong; or did spend exceed expectations?
- Adjust budget or address spend.

### 6.6 The "tight cost SLO" trap

Too-tight cost SLO produces frequent fires; team disregards.

**Corrective.** Forecast-driven budget; reasonable headroom; periodic recalibration.

### 6.7 The "no cost SLO" gap

Without it, cost drifts; surprises arrive monthly.

**Corrective.** Per-feature cost SLO from launch.

---

## 7. Cost sensitivity to architectural decisions

The decisions that move the cost line.

### 7.1 Model-tier decision

Per workload:

- Frontier (Sonnet) vs cheap (Haiku) vs premium (Opus).
- 5-15x cost difference.
- Per-workload eval validates whether the tier is right.

Cross-link to [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md).

### 7.2 Prompt-size decision

Per workload, the prompt is a budget:

- Smaller prompt: cheaper per call.
- Smaller prompt: faster.
- Smaller prompt: may lose context.

Trade-off; optimize per workload.

### 7.3 Retrieval-depth decision

Per workload:

- Retrieve 3 documents: smaller context; cheaper; lower retrieval recall.
- Retrieve 10 documents: larger context; more expensive; better recall.

Per-workload tuning; eval-driven.

### 7.4 Tool-call vs single-call decision

Some workflows can be done as:

- Agent (multi-step; tool calls): more capable but expensive (5-10 calls).
- Single LLM call (with full context): cheaper (1 call).

The cost ratio: 5-10x.

For some workflows, the single-call approach is sufficient; for others, agent is required. Choose deliberately.

### 7.5 Caching-enabled vs caching-disabled

For repeat-content workloads:

- With caching: 50%+ cost reduction.
- Without: full cost per call.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md).

### 7.6 Batch vs real-time

For batch-eligible workloads:

- Batch API: 50% discount.
- Real-time: full cost.

Cross-link to [ai-engineering-reference-architecture / cost-and-finops / batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md).

### 7.7 Self-hosted vs hosted

For high-volume workloads:

- Hosted: per-token cost dominates.
- Self-hosted: infrastructure cost dominates; per-call near-zero.

Break-even depends on volume; cross-link to [model-strategy/build-vs-buy-decision.md §4](../model-strategy/build-vs-buy-decision.md).

### 7.8 The architectural cost levers summary

```
Lever                    Typical impact
Model-tier               5-15x
Prompt-size              2-5x
Retrieval-depth          1.5-3x
Agent vs single-call     5-10x
Caching                  1.5-2x (often 50%+)
Batch vs real-time       2x
Self-hosted vs hosted    3-50x at scale
```

Combined: 100-1000x potential cost variation per workload depending on architecture.

### 7.9 The "we want all the savings" temptation

Stacking all cost-reduction levers degrades quality:

- Cheapest model + smallest prompt + minimal retrieval + no agent + aggressive caching → too aggressive; quality suffers.

Per-workload tuning; eval-driven.

### 7.10 The "we picked the cheap option but quality regressed" recovery

If cost optimization went too far:

- Eval validates the regression.
- Walk back the most aggressive cuts.
- Re-eval until quality acceptable + cost lower than start.

Iterative; not all-at-once.

---

## 8. Worked Meridian example

Meridian's token economics per workload.

### 8.1 Care Coordinator per-task cost

```
Avg task:
  System prompt + tool defs:    2000 tokens
  Patient context retrieved:    4000 tokens
  Conversation history:         1000 tokens
  Cumulative input across 5 calls: ~7000 input per call average (with growth)
  Cumulative output across 5 calls: ~700 output per call average
  
Per call:
  Input: ~7000 × $3/M = $0.021
  Output: ~700 × $15/M = $0.0105
  Per-call: $0.0315 (no caching) → $0.012 (with caching)

Per task (5 calls):
  Without caching: $0.16
  With caching:    $0.06 (62% savings via prefix caching)

Volume: 5k tasks/day = $300/day → $9k/month
Adjusted budget allows for: 7k tasks/day peak = $12k/month
```

Architecture decisions that affect this:
- Sonnet vs Haiku: 5x cost; Sonnet justified by quality eval.
- 5-call avg: vs 8-call earlier; prompt engineering reduced steps.
- Caching enabled: 62% savings.

### 8.2 Patient API chat per-conversation cost

```
Avg conversation:
  System prompt: 800 tokens
  Tenant overlay: 200 tokens
  Retrieval: 1500 tokens (3 documents)
  
First turn:
  Input: 2500 + new message (200) = 2700
  Output: 400 tokens
  Cost: $0.014

Subsequent turns (with caching):
  Input: 90% cached + new turn = effective 270 tokens
  Output: 400 tokens
  Cost per turn: $0.007

Typical 5-turn conversation:
  Turn 1: $0.014
  Turns 2-5: $0.007 × 4 = $0.028
  Total: $0.042

Volume: 12k turns/day = ~2400 conversations
Cost/day: ~$100 → $3k/month
```

### 8.3 Document ingestion per-document cost

```
Per document:
  Input: 3000 tokens (document text + extraction prompt)
  Output: 800 tokens (structured classification + extracted fields)
  
With Llama 3 70B self-hosted: ~$0.0001 per call (infrastructure amortized)
  
Vs Sonnet (alternative): $0.021 per call
  
Volume: 40k docs/day
  Self-hosted: $4/day → $120/month + infrastructure $5k/month = ~$5.1k/month
  Sonnet alternative: $840/day → $25k/month
  
Savings via self-host: ~$20k/month
```

### 8.4 Embedding cost

```
Per embedding:
  Input: 500 tokens (typical document)
  Self-hosted BGE: ~$0.000005 (infrastructure amortized)
  
Volume: 40k docs/day
  Self-hosted: $0.20/day → $6/month + infrastructure $500/month = ~$506/month
  Hosted alternative: $0.10/M × 40k × 500 = $2/day → $60/month
  
Hosted is cheaper for embedding!
  But self-hosted is on HIPAA infrastructure; data isolation justifies.
```

### 8.5 The aggregate

```
Workload                    Volume          Monthly Cost
Care Coordinator (US)       5k tasks/day    $9k
Patient API chat (US)       12k turns/day   $3k
Patient API chat (CA)       2k turns/day    $1k
Document ingestion          40k docs/day    $5k (mostly infra)
Document embedding          40k embeds/day  $500 (mostly infra)
Clinical decision support   500 queries/day $2k
Analytics warehouse copilot 2k queries/day  $2k
Internal copilot            100/day         $0.10
────────────────────────────────────────────────────
Total                                       ~$22k/month
```

### 8.6 The cost SLOs

Per-feature SLOs:

```
Care Coordinator:           $10k monthly budget; alert at 80%.
Patient API chat (US):      $4k monthly budget.
Patient API chat (CA):      $1.5k monthly budget.
Document ingestion:         $6k monthly budget.
Document embedding:         $700 monthly budget.
Clinical decision support:  $2.5k monthly budget.
Analytics warehouse copilot: $2.5k monthly budget.
Internal copilot:           $50 monthly budget.
─────────────────────────────────────────────
Total platform budget:      $27k monthly (with 23% headroom over current $22k spend)
```

### 8.7 The architectural cost levers (Meridian's)

Care Coordinator:
- Prompt-prefix caching: 62% savings on input cost.
- Sonnet (not Opus): 5x savings vs premium.
- Multi-step agent: needed for capability; not optimizable.

Document ingestion:
- Self-hosted Llama: 80% savings vs hosted Sonnet.
- Batch processing: not applicable (already self-hosted with batching).

Patient API chat:
- Caching: 60% savings on input.
- Sonnet primary, Haiku fallback: balanced.
- Streaming: latency optimization, not cost.

### 8.8 The cost-budget-as-SLO

When Care Coordinator's burn rate exceeded 2.0x (Q1 2026 prompt-bloat incident), the cost SLO fired:

- Engineering team alerted.
- Investigation per cost-incident runbook.
- Revert applied within 30 minutes.

Cost protection in action.

### 8.9 What the modeling enables

- Per-feature cost forecast within 15% accuracy.
- Annual budget cycle informed by per-workload model.
- New features have cost projection before launch.
- Cost anomalies caught early via burn rate.

### 8.10 The lessons

- Per-workload cost modeling produces predictable spend.
- Architectural decisions are the dominant cost lever.
- Caching, tier-routing, batch are 50-80% savings each; combined are 2-3x reduction.
- Self-hosting for high-volume specific workloads is justified.

---

## 9. Anti-patterns

### 9.1 The single-number cost view

**Pattern.** "AI cost is $X/month." No breakdown. Cost decisions made without diagnostic data.

**Corrective.** Per-feature, per-tenant, per-user breakdown per §5.

### 9.2 The "we'll size it later" deferral

**Pattern.** Feature launches; cost discovered after the bill.

**Corrective.** Per-call cost modeled before launch per §2.

### 9.3 The "Sonnet for everything" cost waste

**Pattern.** Standard model for all workloads; cheap workloads paying premium.

**Corrective.** Per-workload tier per §3.

### 9.4 The hidden cost of agent loops

**Pattern.** Agent task cost calculated as one LLM call; reality is 5-10.

**Corrective.** Per-task cost includes all calls per §2.5.

### 9.5 The forgotten cache benefit

**Pattern.** No prompt-prefix caching enabled; full cost on every call.

**Corrective.** Enable caching; ~50% savings per §2.4.

### 9.6 The "we should be cheaper" without analysis

**Pattern.** Cost target set arbitrarily; engineering can't meet it.

**Corrective.** Forecast-based budget per §8.6.

### 9.7 The cost-budget-as-SLO absent

**Pattern.** Cost target exists; no SLO; no enforcement.

**Corrective.** Cost-budget-as-SLO per §6.

### 9.8 The cost-anomaly invisible

**Pattern.** Per-user / per-tenant outliers invisible in aggregate; first signal is invoice surprise.

**Corrective.** Per-dimension dashboards per §5.5.

### 9.9 The premium tier without cost differential

**Pattern.** Premium tenants get the same model; same cost; no value-cost separation.

**Corrective.** Per-tier cost model per §5.3.

### 9.10 The "we optimized too far" quality regression

**Pattern.** Cost cuts aggressive; quality dropped.

**Corrective.** Per §7.10; eval-validated cost reductions only.

---

## 10. Findings (sprint-assignable)

**ARCH-TOK-001 (P0). No per-feature cost model.** Cost decisions made without diagnostic data. Per-feature modeling per §2. Owner: AI platform + FinOps.

**ARCH-TOK-002 (P0). No per-user / per-tenant cost aggregation.** Outliers invisible. Per-dimension per §5. Owner: AI platform + observability.

**ARCH-TOK-003 (P0). Cost-budget-as-SLO absent.** Cost drifts without protection. Per §6. Owner: AI platform + engineering management.

**ARCH-TOK-004 (P1). Per-call cost not modeled before feature launch.** Cost surprises post-launch. Per §2 and launch-readiness gate (cross-link to [finops-process.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/finops-process.md)). Owner: AI platform + product.

**ARCH-TOK-005 (P1). Prompt-prefix caching not enabled.** 50%+ cost savings left on table. Per §2.4. Owner: AI platform.

**ARCH-TOK-006 (P1). Agent task cost not multi-call modeled.** Cost understated. Per §2.5. Owner: AI platform.

**ARCH-TOK-007 (P1). Per-tier cost differential not surfaced.** Premium-tier value unclear. Per §5.3. Owner: AI platform + product.

**ARCH-TOK-008 (P1). Cost forecast across dimensions absent.** Forward-looking planning lacks data. Per §5.7. Owner: FinOps + AI platform.

**ARCH-TOK-009 (P2). Per-conversation cost growth (with history) not modeled.** Long conversations get expensive unexpectedly. Per §4.2. Owner: AI platform.

**ARCH-TOK-010 (P2). Cost-anomaly detection absent.** Surprises arrive late. Per §5.6. Owner: observability-eng.

**ARCH-TOK-011 (P2). Architectural cost levers not documented per workload.** Future cost-reduction work lacks roadmap. Per §7. Owner: AI platform.

**ARCH-TOK-012 (P2). Self-hosted vs hosted economics not calculated.** Cost-saving opportunities missed for high-volume workloads. Per §7.7 and [build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md). Owner: AI platform + FinOps.

**ARCH-TOK-013 (P2). Cost SLOs not tuned to workload volume.** SLOs may be too tight or too loose. Per §6.5. Owner: AI platform.

**ARCH-TOK-014 (P3). Per-call retention of cost metadata absent.** Audit trail incomplete. Per cost-attribution sibling document. Owner: AI platform.

**ARCH-TOK-015 (P3). Cost dashboards not aligned with finance categorization.** Reconciliation harder than needed. Per FinOps process sibling. Owner: FinOps.

**ARCH-TOK-016 (P3). Per-workload cost-vs-volume curve not maintained.** Hard to see scaling behavior. Per §4.2. Owner: AI platform.

**ARCH-TOK-017 (P3). Cost-impact of new features not estimated.** Surprise spend post-launch. Per §2.8 and launch readiness. Owner: feature teams.

**ARCH-TOK-018 (P3). Reconciliation between modeled cost and vendor invoice not monthly.** Drift in modeling assumptions. Per [cost-attribution.md §7](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md). Owner: FinOps.

---

## 11. Adoption sequencing checklist

- [ ] **Build per-call cost model per workload (§2).**
- [ ] **Implement per-feature, per-tenant, per-user aggregation (§5).**
- [ ] **Enable prompt-prefix caching (§2.4).**
- [ ] **Set cost-budget-as-SLO per feature (§6).**
- [ ] **Implement cost-burn-rate alerts (§6.2).**
- [ ] **Build per-call cost dashboard (§5.5).**
- [ ] **Forecast per workload (§5.7).**
- [ ] **Annual cost-model review.**
- [ ] **Per-workload architectural cost-lever assessment (§7).**
- [ ] **Pre-launch cost projection for new features.**
- [ ] **Monthly reconciliation against vendor invoices.**

---

## 12. References

**In this folder.**
- [caching-tiers.md](./caching-tiers.md) — caching as cost lever.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) *(coming; partial overlap with engineering sibling)* — model tiering.
- [gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md) — self-host economics.
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — latency dimension; companion.
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — concurrency dimension; companion.
- [cost-incident-playbook.md](./cost-incident-playbook.md) — incident response.

**Elsewhere in this repo.**
- [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md) — comparison framework.
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — self-hosted economics.
- [model-strategy/model-catalogue-and-registry.md](../model-strategy/model-catalogue-and-registry.md) — catalogue tracks rates.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant cost isolation.

**Sibling repos.**
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — engineering attribution mechanics.
- [ai-engineering-reference-architecture / cost-and-finops / cost-dashboards-and-alerts.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-dashboards-and-alerts.md) — dashboards on this data.
- [ai-engineering-reference-architecture / cost-and-finops / finops-process.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/finops-process.md) — process that consumes cost data.
- [ai-engineering-reference-architecture / cost-and-finops / caching-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/caching-for-cost.md) — caching mechanics.
- [ai-engineering-reference-architecture / cost-and-finops / batch-vs-realtime-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/batch-vs-realtime-cost.md) — batch discount.
- [ai-engineering-reference-architecture / cost-and-finops / per-tenant-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/per-tenant-cost-control.md) — per-tenant accounting.
- [ai-engineering-reference-architecture / agent-engineering / agent-cost-control.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-cost-control.md) — agent-specific.

**External.**
- Provider pricing pages.
- FinOps Foundation token-economics literature.
- Anthropic / OpenAI / Google cost-modeling documentation.
