# Context Window Budgeting

> **Audience.** Architects whose context window is shared across system, retrieval, history, tools, and answer — and they don't know what fraction each consumes. Tech leads whose "1M context model" is somehow still hitting context limits. Anyone whose retrieval keeps pulling "one more chunk for safety" until 16k of retrieval is in every prompt. **Scope.** The *architectural* discipline of treating the context window as a budgeted resource: per-allocation budgets (system, instructions, retrieval, history, tools, answer); overflow handling (priority truncation, summarization, tiered-context routing); context-budget-as-SLO; patterns for unbounded-context workloads (long agents, repository-aware coding). Not the system prompt design (see [system-prompt-architecture.md](./system-prompt-architecture.md), companion). Not the long-context-vs-RAG decision (see [long-context-vs-rag.md](./long-context-vs-rag.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The context window is a shared, finite, expensive resource that every part of an AI feature competes for. In 2026 with 200k-1M context models, "finite" is less constrained than 2023, but the resource is still bounded — and "expensive" still applies (every token in context costs money and latency).

The typical context distribution for a mature AI feature:

```
Total prompt: ~12,000 tokens
  System (assembled): 1,500 tokens (12%)
  Retrieved context: 6,000 tokens (50%)
  Conversation history: 2,500 tokens (21%)
  Tool definitions: 1,000 tokens (8%)
  Current user message: 100 tokens (1%)
  Reserved for output: 1,000 tokens (8% of total)
```

Each allocation has competing pressure:

- Retrieval team: "more chunks = better recall."
- Conversation team: "more history = better memory."
- Tools team: "more tools = more capability."
- System prompt team: "more instructions = better behavior."
- Output team: "more max_tokens = more complete answers."

Without budgets, every team optimizes its allocation; the total grows; cost and latency suffer; quality often regresses (more context = more noise, sometimes).

The architectural alternative: treat the context window as a budget; allocate explicitly per use; have a policy for overflow; monitor consumption.

This document covers the budgeting framework, the overflow handling strategies, and the discipline that makes the budget operationally enforceable.

This document is opinionated about four things:

1. **The context window is a budget, not "as much as fits."** Per-allocation explicit budgets; overflow handled deliberately.
2. **Overflow handling is per-allocation, not platform-wide.** Retrieval overflows differently than history overflows differently than tool definitions overflow.
3. **Bigger context isn't free.** 1M-context model exists; using all of it is expensive (per token cost + latency). Budget at need, not capacity.
4. **Context budget is an SLO.** Track consumption; alert on growth; review periodically.

Structure: (2) the six context allocations; (3) per-allocation budget design; (4) overflow handling strategies; (5) context-budget-as-SLO; (6) the unbounded-context case; (7) tiered context routing; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The six context allocations

What goes in the context window.

### 2.1 System (the assembled system prompt)

Per [system-prompt-architecture.md](./system-prompt-architecture.md):

- Platform base.
- Feature overlay.
- Tenant overlay (if any).

Typical: 800-2500 tokens.

### 2.2 Instructions (current request's specific guidance)

Beyond the system prompt, per-request instructions:

- "Respond in JSON format."
- "Focus on the patient's recent labs."
- "Limit response to 200 words."

Typical: 0-500 tokens.

### 2.3 Retrieved context

Documents pulled from RAG:

- Document chunks.
- Knowledge base entries.
- Reference materials.

Typical: 1000-10000 tokens (highly variable; depends on k and chunk size).

### 2.4 Conversation history

Prior turns in the conversation:

- Verbatim history.
- Or summarized.
- Or selective.

Typical: 0-15000 tokens (grows unbounded if not managed).

### 2.5 Tool definitions

For agent workloads:

- Available tools with descriptions.
- Tool parameters.
- Examples.

Typical: 500-3000 tokens (depends on tool count).

### 2.6 Current user message

The actual ask.

Typical: 10-2000 tokens.

### 2.7 Reserved for output (max_tokens)

The model's response space:

- max_tokens parameter.
- Bounded; counts toward "context window" for some providers.

Typical: 500-4000 tokens.

### 2.8 The allocation summary

```
Total context window: 200k tokens (for Sonnet-class in 2026)

Reserved for output:       ~2,000 (1%)
Available for input:      ~198,000 (99%)

Typical input allocation:
  System (assembled):       ~1,500 (1%)
  Per-request instructions:   ~200 (0.1%)
  Retrieved context:        ~6,000 (3%)
  Conversation history:     ~3,000 (1.5%)
  Tool definitions:         ~1,000 (0.5%)
  Current user message:       ~100 (<0.1%)
  ─────────────────────────────────
  Used: ~11,800 (6% of available)
  Headroom: ~186,000 (94%)
```

Headroom is large; but cost and latency apply per used token, not per headroom.

### 2.9 The "we have 1M context; use it" anti-instinct

1M-context model exists. Using all of it:

- Costs more per call (per-token cost × all tokens).
- Slower TTFT (longer input takes longer to process).
- "Lost in the middle" — quality on long context degrades for some workloads.

Budget at need.

---

## 3. Per-allocation budget design

Designing the budgets.

### 3.1 The system budget

Target: 1500-2500 tokens.

Per [system-prompt-architecture.md §3.8](./system-prompt-architecture.md):

- Platform base: ~800 tokens.
- Feature overlay: ~500 tokens.
- Tenant overlay: ~150 tokens (where applicable).

Above 3000 tokens: refactor.

### 3.2 The retrieval budget

Target: 2000-8000 tokens.

Considerations:

- k (number of chunks): typically 5-15.
- Chunk size: typically 200-500 tokens.
- Total: k × chunk_size.

Above 8000 tokens: investigate (retrieval bloat).

Below 2000: investigate (might be under-retrieving).

### 3.3 The conversation history budget

Target: 1500-5000 tokens for sustained conversations.

Two regimes:

- Short conversations (<5 turns): minimal history; <1000 tokens.
- Long conversations: history dominates; budget the most.

Above budget: summarize old turns (cross-link to [chat-history-architecture.md](./chat-history-architecture.md)).

### 3.4 The tool definitions budget

Target: 500-2000 tokens.

Bounded by number of tools × per-tool description size.

Above 2000: consider tool consolidation or per-feature tool subsets.

### 3.5 The instructions budget

Target: 100-500 tokens.

Per-request guidance; not the system prompt; not the user's full ask.

Above 500: probably belongs in the system prompt (feature overlay).

### 3.6 The user-message budget

Variable; user-driven.

- Chat: typically 50-200 tokens.
- Document upload: could be 100k+.

For very long inputs, may need pre-processing (chunking, summarization).

### 3.7 The output budget

Target: 500-4000 tokens.

max_tokens setting. Variable per workload:

- Chat: ~500.
- Document drafting: ~4000.
- Agent step: ~1500.

### 3.8 The total budget

Sum of allocations:

```
System:         1500
Instructions:    200
Retrieval:      4000
History:        2000
Tools:          1500
User:            300
Output reserve: 1500
─────────────────────
Total:         11000 tokens budget

(Headroom in 200k context: ~189k; in 1M context: ~989k. But cost / latency at 11k.)
```

The budget is per-workload; per-feature.

### 3.9 The "no budget" alternative

Without per-allocation budgets:

- Each allocation grows.
- Total grows.
- Cost and latency suffer.
- Hard to diagnose what's growing.

With budgets: each allocation has a known shape; growth is detected.

---

## 4. Overflow handling strategies

What to do when an allocation exceeds budget.

### 4.1 The five strategies

**Strategy A: Hard cap (reject overflow).** Reject the request; return error.

**Strategy B: Truncate.** Drop the tail of the overflowing content.

**Strategy C: Prioritize and drop.** Keep highest-priority content; drop lower-priority.

**Strategy D: Summarize.** Replace verbose content with a summary.

**Strategy E: Tier up (route to bigger-context model).** Move to a model with more context.

Each appropriate for different allocations.

### 4.2 The system-prompt overflow

If the assembled system prompt is too large:

- Strategy: refactor (cross-link to [system-prompt-architecture.md §7](./system-prompt-architecture.md)).
- Not a per-request decision; a design-time decision.

### 4.3 The retrieval overflow

If retrieval returns too many tokens:

- Strategy C: keep top-K most relevant chunks.
- Strategy B: truncate each chunk to a smaller size.
- Strategy D: summarize multiple chunks into one.

Cross-link to [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md).

### 4.4 The history overflow

If conversation history grows too large:

- Strategy D: summarize old turns; keep recent verbatim.
- Strategy C: drop turns marked as "less relevant."

Cross-link to [chat-history-architecture.md](./chat-history-architecture.md).

### 4.5 The tools overflow

If tool definitions are too large:

- Strategy C: limit tools to those relevant to the current task (per-task tool subset).
- Strategy D: shorter tool descriptions.

### 4.6 The user-message overflow

If the user's input is very large:

- Strategy A: reject (with friendly error).
- Strategy B: truncate (warn user).
- Strategy E: route to large-context-capable model.

For pre-known large inputs (document analysis): plan for it; route to appropriate model.

### 4.7 The tier-up strategy

When total context approaches the standard model's limit:

- Route to a larger-context model.
- Cost: higher per token.
- Capability: handles the case.

Per-workload decision.

### 4.8 The overflow detection

Pre-call check:

```python
def check_context_budget(assembled_prompt, max_tokens):
    total_input = count_tokens(assembled_prompt)
    total_required = total_input + max_tokens + buffer
    
    if total_required > model_context_limit:
        return OverflowDecision(strategy="...", ...)
    if total_input > expected_budget:
        return OverflowDecision(strategy="truncate_low_priority", ...)
    return ContextOK()
```

Don't wait for the model to reject; check pre-call.

### 4.9 The user-visible overflow

If overflow forces degradation:

- User-visible indicator (cross-link to [reliability-engineering/degraded-mode-design.md §5](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md)).
- Transparency: "Conversation has grown long; older context summarized."

---

## 5. Context-budget-as-SLO

Treating budget as a reliability dimension.

### 5.1 The SLO

Per feature:

```yaml
care-coordinator-context-budget-slo:
  target_p99_total_tokens: 12000
  target_p99_retrieval_tokens: 5000
  target_p99_history_tokens: 3000
  
  alert_when_growth_exceeds: 20% over rolling baseline
```

Like latency / cost SLO; context has targets.

### 5.2 The metrics

Per call:

- Total input tokens.
- Per-allocation tokens.
- Output tokens.

Per feature, aggregate:

- P99 of each.
- Trend over time.

### 5.3 The growth alerts

When an allocation grows:

- Retrieval allocation up 30%: investigate (chunk size change? K increased? content change?).
- History allocation up: longer conversations or summarization broken.
- System allocation up: prompt modification.

Per-allocation alerts surface specific growth.

### 5.4 The budget-burn

Like cost burn:

```
context_burn_rate = current_p99 / target_p99
```

If > 1.2: notify; investigate.
If > 1.5: alert.

### 5.5 The "the prompt grew silently" detection

Without monitoring: prompts grow; cost rises; nobody knows why.

With per-allocation monitoring: growth is detected per allocation; investigation can pinpoint.

### 5.6 The quarterly context review

Per feature:

- Review per-allocation P99 trends.
- Identify which is growing.
- Decide: refactor or accept.

Annual: full review of context architecture.

### 5.7 The "we discovered we'd been over budget for months" learning

Without monitoring: discovery is when a cost or latency incident occurs.

With monitoring: discovery is in the routine review; cheap to address.

---

## 6. The unbounded-context case

When context truly grows without bound.

### 6.1 The use cases

- Long-running conversations (days of chat).
- Repository-aware coding agents (entire codebase as context).
- Multi-document research agents.
- Workflow agents over weeks.

### 6.2 The "fits in 1M tokens" reality check

1M context models can hold:

- ~750k words (3-5 books).
- ~25k lines of code (small repo).
- ~50 long PDFs.

For most "unbounded" cases, 1M context is enough — if you fit cleverly.

### 6.3 The pagination strategy

For very large content:

- Break into sections.
- Process section-by-section.
- Aggregate.

Each call is bounded; total work is paginated.

### 6.4 The summarization strategy

Recursive summarization:

- Summarize each section.
- Summarize summaries.
- Process top-level summary.

Loses detail; reduces context size.

### 6.5 The RAG strategy

Even for "long-context" use cases, RAG often wins:

- Retrieve only what's relevant.
- 200k-1M context not needed.
- Cheaper.

Cross-link to [long-context-vs-rag.md](./long-context-vs-rag.md).

### 6.6 The persistent-memory pattern

For long-running conversations:

- Conversation history grows.
- Periodically: condense to facts + summary.
- Keep recent verbatim + condensed older.

Cross-link to [chat-history-architecture.md](./chat-history-architecture.md).

### 6.7 The agent-state truncation

For long-running agents:

- Recent steps in detail.
- Older steps as summary.
- State accumulates structured (not in prompt).

The "prompt" doesn't grow; the agent's external state does.

### 6.8 The "we genuinely need all of it" rare case

Some workloads do need everything:

- Legal contract analysis (every clause matters).
- Code refactoring across the codebase.
- Long-form document Q&A.

For these:

- Use large-context model.
- Accept the cost.
- Optimize: caching, batch processing.

---

## 7. Tiered context routing

Multiple models with different context capacities.

### 7.1 The tier pattern

```
Small-context tier:  Haiku, 200k context, low cost
Mid-context tier:    Sonnet, 200k context, medium cost
Large-context tier:  Sonnet with 1M context, higher cost
Reasoning tier:      Opus, 200k context, premium cost
```

Routing per request based on context need.

### 7.2 The routing decision

```python
def select_tier(estimated_context_size):
    if estimated_context_size < 5000:
        return "small_context"  # Haiku
    elif estimated_context_size < 50000:
        return "mid_context"    # Sonnet
    else:
        return "large_context"  # Sonnet 1M
```

Per-call routing.

### 7.3 The "we always use mid-context tier" simplification

Many platforms use one tier:

- Sonnet for everything.
- Simpler routing.
- Wastes capability or cost depending on workload.

For volume justifying the operational overhead, tiered.

### 7.4 The fallback tier

When estimated context overflows:

- Tier up.
- More cost; more capability.

Cross-link to [reliability-engineering/degraded-mode-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md).

### 7.5 The per-feature tier mapping

```
Feature                Primary tier    Overflow tier
─────────────────────────────────────────────────────
Patient API chat       mid (Sonnet)    -
Care Coordinator       mid (Sonnet)    large (Sonnet 1M)
Document analysis      large (Sonnet 1M)  -
Internal copilot       small (Haiku)   mid (Sonnet)
```

Per-feature; predictable.

### 7.6 The "we'll route at runtime" complexity

Runtime tier selection adds:

- Latency (deciding which tier).
- Cost-tracking complexity.
- Per-tier eval suites.

For most platforms, per-feature default tier is sufficient; runtime tier-up reserved for overflow cases.

---

## 8. Worked Meridian example

Meridian's context budget per feature.

### 8.1 The Care Coordinator context budget

```yaml
care-coordinator-context-budget:
  total_target_p99: 12000 tokens
  
  allocations:
    system:        1500
    instructions:   200  (per-task specific)
    retrieval:     4500  (5-7 documents, ~700 tokens each)
    history:       2000  (last 5 turns, with summarization of older)
    tools:         1500  (12 tools with descriptions)
    user_message:   300  (clinician's task description)
    output_reserve: 1500  (max_tokens=1500)

  overflow_handling:
    system: not applicable (refactor)
    retrieval: top-K = 7; if growing, reduce to 5
    history: summarize beyond 5 turns
    tools: per-task subset if needed
```

Actual P99 today: 11,000 tokens (under budget).

### 8.2 The Patient API chat budget

```yaml
patient-api-chat-budget:
  total_target_p99: 6000 tokens
  
  allocations:
    system:        1200  (smaller than Care Coordinator)
    retrieval:     2000  (3 documents, smaller)
    history:       1500  (last 3 turns)
    user_message:   200
    output_reserve: 1000
```

Smaller budget; faster TTFT; lower cost per turn.

### 8.3 The clinical decision support budget

```yaml
clinical-decision-support-budget:
  total_target_p99: 15000 tokens
  
  allocations:
    system:        2000  (extensive safety + format)
    retrieval:     7000  (10 documents; comprehensive)
    history:       0    (stateless; no chat)
    tools:         2000  (specific decision-support tools)
    user_message:   500
    output_reserve: 1500
```

Larger; needed for comprehensive clinical context.

### 8.4 The monitoring

Per-feature dashboard:

```
Care Coordinator (last 30 days):
  Total P99: 10,800 / 12,000 (within budget)
  Retrieval P99: 4,200 (within target)
  History P99: 2,100 (within)
  Tools: 1,500 (constant)
  Trend: stable

Patient API chat:
  Total P99: 5,800 / 6,000 (close to budget)
  ⚠️ retrieval growing 5% per month
  Investigation triggered
```

Patient API's retrieval allocation growth caught early.

### 8.5 The Q1 2026 prompt-bloat incident (related)

The prompt-bloat incident (cross-link to [cost-incident-runbook.md §9](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-incident-runbook.md)) was a system-allocation budget breach:

- System allocation jumped from 1500 to 2700 tokens.
- Caught by the cost-per-call alert; also visible in context-budget dashboard.
- Reverted within 1h 37m.

The context-budget-as-SLO dashboard would have caught it on the system-budget dimension; complementary to cost monitoring.

### 8.6 The retrieval growth investigation

Q2 2026: Patient API chat's retrieval allocation grew from 1800 to 2300 tokens average.

Investigation:

- Chunk size hadn't changed.
- K hadn't changed.
- New content in the corpus had larger average chunks (longer documents).

Action: chunking re-tuned to keep chunk size in 500-token range.

Retrieval allocation returned to 1900 average.

### 8.7 The tier routing for large queries

Patient API chat occasionally has queries that need large context (medical history review):

- Detect estimated context > 50k.
- Tier up to Sonnet-large-context (1M).
- Process; return result.
- Cost: 5x normal; user-visible "thorough review" indicator.

Few queries (<1% of traffic); cost manageable.

### 8.8 The unbounded-context case

Care Coordinator agent tasks can grow:

- 5-step task: ~12k context.
- 12-step task (rare): ~25k context.

Strategy: history summarization at step 5; older steps condensed.

For very long tasks: tier up to large-context model.

### 8.9 The infrastructure cost

- Context-budget monitoring: ~1 week engineering setup + integration with existing observability.
- Per-allocation alerts: configured per feature.
- Quarterly review: ~1 hour per feature.

Total ongoing cost: minimal.

### 8.10 The value

- Caught the retrieval growth before it became cost incident.
- Caught the system-prompt bloat as a complementary signal.
- Engineering can plan capacity (model selection per workload's budget).
- Cost predictable.

### 8.11 The lessons

- Per-allocation budget is the primitive; total budget alone doesn't surface what's growing.
- Quarterly review prevents drift.
- Tier routing for occasional overflow is cleaner than oversizing all calls.
- 1M-context is rarely needed; budget at need.

---

## 9. Anti-patterns

### 9.1 The "as much as fits" allocation

**Pattern.** No budget; allocations grow; total grows; cost and latency suffer.

**Corrective.** Per-allocation budgets per §3.

### 9.2 The single-budget-only

**Pattern.** "Total prompt < 10k tokens" but allocation-level invisible.

**Corrective.** Per-allocation per §2.8.

### 9.3 The retrieval bloat

**Pattern.** Retrieval increases k or chunk size "to be safe"; retrieval allocation grows; quality may not improve.

**Corrective.** Budget retrieval; eval-driven k tuning.

### 9.4 The conversation-history-grows-forever

**Pattern.** Verbatim history kept indefinitely; allocation grows; latency rises; cost balloons on long conversations.

**Corrective.** Summarization per [chat-history-architecture.md](./chat-history-architecture.md).

### 9.5 The "we have 1M context; let's use it"

**Pattern.** Large-context model used for every call. Cost per call 5-10x what's needed.

**Corrective.** Budget at need per §2.9.

### 9.6 The overflow with no strategy

**Pattern.** Hit context limit; first time is in production; nobody knows what to do.

**Corrective.** Pre-defined per-allocation overflow strategy per §4.

### 9.7 The hard-cap-only

**Pattern.** Overflow → 400 error to user. Bad UX.

**Corrective.** Graceful degradation (truncation, summarization, tier-up).

### 9.8 The opaque growth

**Pattern.** Per-call context grows; total prompt rises; cost rises; nobody can attribute the growth.

**Corrective.** Per-allocation metrics + alerts per §5.

### 9.9 The "we'll add budgets later" deferral

**Pattern.** Build feature; ship; never add budgets. Growth uncontrolled.

**Corrective.** Budgets from launch.

### 9.10 The "tier up for everything" reflex

**Pattern.** Every overflow routes to large-context model. Cost balloons.

**Corrective.** Per-overflow strategy; truncation / summarization preferred for most.

---

## 10. Findings (sprint-assignable)

**ARCH-CWB-001 (P0). No per-allocation context budget defined.** Total grows unmanaged. Per §3. Owner: AI platform.

**ARCH-CWB-002 (P0). No per-allocation observability.** Growth invisible. Per §5.2. Owner: observability-eng.

**ARCH-CWB-003 (P0). Overflow strategy undefined.** First overflow is improvised. Per §4. Owner: AI platform.

**ARCH-CWB-004 (P1). No alerts on allocation growth.** Drift undetected. Per §5.3. Owner: SRE + observability.

**ARCH-CWB-005 (P1). Retrieval allocation not bounded.** Bloat. Per §3.2. Owner: AI platform.

**ARCH-CWB-006 (P1). History grows unbounded.** Long conversations expensive. Per §3.3 and [chat-history-architecture.md](./chat-history-architecture.md). Owner: AI platform.

**ARCH-CWB-007 (P1). Context-budget-as-SLO absent.** Reliability dimension missed. Per §5. Owner: AI platform + SRE.

**ARCH-CWB-008 (P1). Tier-routing for overflow not implemented.** Either degraded or expensive default. Per §7. Owner: AI platform.

**ARCH-CWB-009 (P2). Per-feature budgets not differentiated.** One-size-fits-all is wrong. Per §8 examples. Owner: AI platform + feature teams.

**ARCH-CWB-010 (P2). 1M-context model used by default.** Cost waste. Per §2.9 and §7. Owner: AI platform.

**ARCH-CWB-011 (P2). Quarterly context review absent.** Drift accumulates. Per §5.6. Owner: AI platform.

**ARCH-CWB-012 (P2). User-message overflow not handled.** Long user inputs fail. Per §4.6. Owner: AI platform.

**ARCH-CWB-013 (P2). Tool-definition allocation not bounded.** Many tools = bloat. Per §3.4. Owner: agent platform.

**ARCH-CWB-014 (P2). Pre-call budget check absent.** Overflows discovered at provider rejection. Per §4.8. Owner: AI platform.

**ARCH-CWB-015 (P3). User-visible overflow indicator absent.** UX surprise on degradation. Per §4.9. Owner: product + AI platform.

**ARCH-CWB-016 (P3). Repository-/document-aware workloads use naive context.** No pagination / summarization. Per §6. Owner: AI platform.

**ARCH-CWB-017 (P3). Agent state grows in prompt rather than external store.** Prompt grows; should grow externally. Per §6.7. Owner: agent platform.

**ARCH-CWB-018 (P3). 1M-context-only workloads not identified.** Some workloads forced to small; need large. Per §6.8. Owner: AI platform + product.

---

## 11. Adoption sequencing checklist

- [ ] **Define per-allocation budgets per feature (§3).**
- [ ] **Build per-allocation observability (§5.2).**
- [ ] **Implement per-call token counting per allocation.**
- [ ] **Define overflow handling per allocation (§4).**
- [ ] **Add pre-call budget check (§4.8).**
- [ ] **Set context-budget-as-SLO (§5).**
- [ ] **Implement tier-routing for overflow (§7).**
- [ ] **Quarterly context review (§5.6).**
- [ ] **Per-feature budgets in feature design docs.**
- [ ] **Annual context architecture review.**

---

## 12. References

**In this folder.**
- [system-prompt-architecture.md](./system-prompt-architecture.md) — system allocation; companion.
- [chat-history-architecture.md](./chat-history-architecture.md) — history allocation strategies.
- [long-context-vs-rag.md](./long-context-vs-rag.md) — retrieval vs context decision.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — assembly mechanics.
- [prompt-as-api-discipline.md](./prompt-as-api-discipline.md) *(coming)* — versioning across budget changes.

**Elsewhere in this repo.**
- [cost-and-performance-architecture/token-economics.md](../cost-and-performance-architecture/token-economics.md) — cost implications.
- [cost-and-performance-architecture/latency-budgets-and-streaming.md](../cost-and-performance-architecture/latency-budgets-and-streaming.md) — latency implications.
- [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md) — model choice with context implications.

**Sibling repos.**
- [ai-engineering-reference-architecture / rag-engineering / retrieval-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/rag-engineering/retrieval-engineering.md) — retrieval allocation engineering.
- [ai-engineering-reference-architecture / agent-engineering / memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) — agent memory at scale.
- [ai-engineering-reference-architecture / reliability-engineering / degraded-mode-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/degraded-mode-design.md) — degraded mode for overflow.

**External.**
- Provider context-window documentation (Anthropic 200k/1M, OpenAI 128k+, Google 1M+).
- "Lost in the Middle" research on long-context quality.
- Token-counting libraries (tiktoken, etc.).
