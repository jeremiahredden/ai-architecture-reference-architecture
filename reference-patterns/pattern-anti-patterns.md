# Pattern Anti-Patterns

> **Audience.** Tech leads inheriting an AI system. Architects auditing a roadmap of AI features. Anyone whose AI architecture has a pattern that doesn't quite fit the problem and isn't sure if it's the pattern or the implementation. **Scope.** The six pattern-selection anti-patterns observed most often — what the pattern looks like in production; why teams choose it; what it causes; the corrective pattern. A consolidated catalogue across the reference-patterns folder; references the per-topic depths. **Worked client.** Meridian Health.

---

## 1. Why this document exists

Every pattern doc in this folder has an "anti-patterns" section covering the wrong applications of that pattern. This document is the consolidated catalogue of the six pattern-selection mistakes that recur across teams — patterns chosen when a simpler one would have served better, or chosen because the pattern was on a vendor's slide deck rather than because the problem demanded it.

The mistakes share a common shape: complexity adopted ahead of need. AI architecture's complexity compounds nonlinearly. Adding a second component (a router, a graph store, a fine-tuned model, an agent loop) more than doubles operational overhead — each new component adds its own failure modes, eval surfaces, observability requirements, and cost lines. Patterns that earn their complexity (production data supports their value) are worth their cost; patterns adopted on speculation or vendor pitch usually aren't.

The catalogue is operational, not theoretical. Each anti-pattern has been observed in production at multiple companies; each has caused at least one notable incident class or quality plateau; each has a corrective pattern that simpler-and-effective beats complex-and-fashionable.

Two things to keep in mind while reading:

First, these anti-patterns rarely appear alone. The team that adopted "agent for everything" often also adopted "graph RAG" and "fine-tune as first move" — the same underlying condition (complexity is taken as sophistication) produces multiple at once. Audits should look for clusters.

Second, the corrective isn't "never use the complex pattern." It's "evaluate the simpler pattern first; graduate to the complex one only when production data justifies." Some workloads genuinely need agents, graphs, fine-tuning, routing. The discipline is choosing deliberately rather than by default.

Structure: (2) "agent for everything"; (3) "graph RAG because the data is graph-shaped"; (4) "fine-tune as first move"; (5) "router as first move"; (6) "every step is an LLM call"; (7) "monolithic prompt as architecture"; (8) worked Meridian example (audit and remediation); (9) findings; (10) adoption checklist (the architecture audit); (11) references.

---

## 2. Anti-pattern #1: "Agent for everything"

### 2.1 What it looks like

Every AI feature in the team's roadmap is an agent. Single-call solutions that would have worked are wrapped in agent loops. Workflow-shaped problems are forced into agent shape. The team's "AI feature" mental model has collapsed into "agent."

### 2.2 Why it happens

- Framework demos are agent demos; the first feature followed the demo pattern; subsequent features default to it.
- "Agent" sounds more advanced than "workflow"; perceived sophistication drives the choice.
- Vendor marketing positions agentic capability as the leading edge; teams adopt the framing.
- The team hasn't built the discipline of explicit shape selection per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md) in the engineering sibling.

### 2.3 What it causes

Per [agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) anti-pattern #1:

- 3-10× the operational cost of equivalent workflow.
- Latency 3-5× workflow.
- Debugging surface requiring trajectory reading for simple cases.
- Eval harder and more expensive than necessary.
- On-call burden disproportionate to feature importance.

### 2.4 The corrective

Per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md): the shape decision tree. Default to workflow; agent only when criteria justify.

Refactor candidates: features whose production traces show the agent following the same path on 80%+ of requests — the recurring pattern is workflow-shaped.

### 2.5 The case study

A retail company built a customer-support agent. Production traffic was 90% "where's my order" + 7% "return process" + 3% other. The agent's trajectory was nearly identical across the 97% common cases.

Refactor: workflow with classify → branch (order status / return / other). The agent step is only for the 3% genuinely-flexible cases. Cost dropped 70%; latency cut in half; quality stable.

The refactor was 2 sprints; the savings compound.

---

## 3. Anti-pattern #2: "Graph RAG because the data is graph-shaped"

### 3.1 What it looks like

The team's data has entities and relationships (customer, product, order; patient, medication, encounter; user, document, tag). The team adopts a knowledge graph because the data "is graph-shaped." Constructed the graph; integrated with retrieval; quality didn't materially improve over vector retrieval; maintenance burden is significant.

### 3.2 Why it happens

- Recent attention on GraphRAG (Microsoft's 2024 paper popularised it).
- "Our data is naturally a graph" sounds like obvious justification.
- Vendor pitches for graph databases.
- The team doesn't ask "which queries actually need traversal?"

### 3.3 What it causes

Per [knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md):

- Construction cost (extraction is expensive at scale).
- Maintenance cost (graph schema evolves; entity disambiguation is ongoing work).
- Operational complexity (graph store alongside vector store).
- Quality often comparable to better vector retrieval + reranking.

### 3.4 The corrective

Per [knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md) section 2.3: diagnostic questions before adopting.

- Can you write 5 specific queries the KG should answer?
- For each, can you describe the traversal?
- Where would the entity data come from?
- Who maintains it ongoing?

Honest answers reduce KG adoption to the cases where it actually helps. Most workloads: hybrid retrieval + reranking covers the value.

### 3.5 The case study

A B2B SaaS company built a knowledge graph from their customer relationships, product catalog, and support tickets. Spent 6 months on construction + ongoing 0.5 FTE maintenance.

Production: the graph queries were used in 2% of support-assistant interactions. The other 98% were served by vector retrieval (which worked).

Retirement: the team retired the KG after a quarterly review showed the cost wasn't justified. The 2% of queries that used graph traversal were re-routed to a simpler "fetch related tickets" tool.

The 6-month investment was largely sunk; the ongoing 0.5 FTE recovered.

---

## 4. Anti-pattern #3: "Fine-tune as first move"

### 4.1 What it looks like

The team starts a new AI feature with "let's fine-tune." A custom dataset is curated; fine-tune runs; the result is deployed. Production quality isn't materially better than a well-engineered prompt would have produced; cost / latency are similar; the fine-tune is now operational baggage.

### 4.2 Why it happens

- "Fine-tuning" sounds like the serious ML approach.
- The team has ML background; fine-tuning is familiar.
- Vendor sales pitches for fine-tuning services.
- Misunderstanding of when fine-tune adds value (per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) section 6).

### 4.3 What it causes

- Months of engineering effort that could have been weeks.
- Fine-tune locks the team to a base model; upgrades require re-fine-tune.
- Quality often comparable to prompt + few-shot + good retrieval.
- Operational baggage (fine-tune pipeline, version management, eval).

### 4.4 The corrective

Per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) section 6: fine-tune crossover criteria.

- Few-shot example count exceeds 20.
- Use case is stable (frequent changes invalidate the fine-tune).
- Quality gap remains after prompt + few-shot optimization.
- Volume is high (cost savings justify the investment).

If three of four don't apply, fine-tune is premature.

Start with: prompt + retrieval + few-shot. Iterate. If quality plateau is reached and the four criteria hold, then fine-tune.

### 4.5 The case study

A media company wanted to fine-tune a model for "their brand voice." Curated 500 training samples; ran the fine-tune; deployed.

Eval against the same prompt + 5 few-shot examples (no fine-tune): the few-shot approach scored within 2% on brand voice, at no fine-tune investment.

The fine-tune was abandoned. The 6 weeks of engineering effort was recovered for prompt engineering, which produced larger quality gains.

---

## 5. Anti-pattern #4: "Router as first move"

### 5.1 What it looks like

The team's first AI feature has a router (rules / classifier / LLM-as-router) deciding which model handles each request. Multiple models in production from day one. Router complexity equals or exceeds the feature complexity.

### 5.2 Why it happens

- Cost-conscious teams hear "tier routing saves money" and adopt before they have a cost problem.
- Anticipating future complexity ("we'll need to route eventually").
- Architectural elegance pull ("the right architecture has a router").

### 5.3 What it causes

- Engineering effort spent on routing instead of feature quality.
- Router behaviour debugging adds to incident triage.
- Multiple models in production means multiple eval surfaces.
- The original feature ships later than it should have.

### 5.4 The corrective

Per [multi-model-orchestration.md](./multi-model-orchestration.md) section 3.8: the "always frontier" reset.

Most teams' first version should be a single tier (often Sonnet or equivalent). After production data reveals which requests are easy vs hard, introduce tier routing. The data drives the decision; not speculation.

The order:

1. Ship the feature on one model.
2. Gather production data.
3. Identify which requests would handle on cheaper / simpler tier.
4. Introduce routing.
5. Verify cost / quality balance.

Most teams shave 60-80% of cost by introducing routing *after* the feature is mature. Starting with routing on speculation usually produces worse outcomes.

### 5.5 The case study

A startup's first AI feature shipped with a 3-tier router (Haiku → Sonnet → Opus by classifier). Took 6 weeks vs estimated 2 weeks (router engineering + eval + monitoring).

Production data revealed the classifier was sending 60% to Sonnet (its calibration was off); the cost was nearly the same as Sonnet-only.

The team's next AI feature shipped without a router (Sonnet only). After 3 months of production data, they added routing — calibrated to actual production patterns. Cost dropped 50% with minimal quality impact. The pattern worked the second time because the data informed it.

---

## 6. Anti-pattern #5: "Every step is an LLM call"

### 6.1 What it looks like

The team's AI workflow has 8 steps. All 8 are LLM calls. Some of them are doing things that deterministic code could handle (parsing, formatting, simple lookups, validation).

### 6.2 Why it happens

- "LLMs are flexible; let them handle everything."
- Composing prompts is faster than writing code (in the short term).
- The team doesn't think to ask "does this step need a model?"

### 6.3 What it causes

- Cost: each step pays LLM cost when deterministic would be free.
- Latency: each step adds 200-1000ms of model latency.
- Failure modes: each step is a potential hallucination / failure point.
- Debugging surface: 8 LLM-call traces to read per failure.

### 6.4 The corrective

For each step, ask: "Could a deterministic implementation handle this reliably?"

- Parsing structured data: deterministic.
- Formatting output: deterministic (templates).
- Calculations: deterministic.
- Looking up known values: deterministic (DB / cache).
- Generating natural-language content: LLM.
- Making decisions on unstructured input: LLM.
- Classifying with stable categories: small classifier (could be LLM-as-classifier or actual classifier).

The right pattern is "LLM for the parts that need flexibility; code for the parts that don't."

### 6.5 The case study

A data processing pipeline had 6 LLM steps: extract → validate → format → store → notify → log. The middle 4 (validate, format, store, notify) didn't need LLM judgment.

Refactored to 2 LLM steps (extract, log summary) + 4 deterministic steps. Cost dropped 75%; latency cut in half; reliability up (the deterministic steps don't hallucinate).

---

## 7. Anti-pattern #6: "Monolithic prompt as architecture"

### 7.1 What it looks like

One giant prompt (4,000-8,000 tokens) contains the entire feature's logic: instructions, tools list, examples, formatting requirements, refusal patterns, behaviour rules, scope constraints. Changes mean editing the monolith; debugging means reading it; every interaction pays for all the tokens.

### 7.2 Why it happens

- The first version was small; grew over time.
- Refactoring feels like a bigger investment than adding another paragraph.
- The team hasn't separated concerns (system prompt vs tools vs schema vs few-shot vs format).

### 7.3 What it causes

Per [prompt-engineering/prompt-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-anti-patterns.md) anti-pattern #3 ("kitchen-sink monolith prompt"):

- Cost: every call pays for the entire monolith even when parts are irrelevant.
- Quality: model's attention divided; quality degrades.
- Change risk: any edit can have unintended effects on other parts.
- Debugging: hard to localize behaviour to specific parts.

### 7.4 The corrective

Decompose:

- System prompt (stable, broad): one component.
- Tool descriptions (per tool): separate.
- Schema (output structure): separate.
- Few-shot examples (when needed): separate; per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md).
- Per-feature behaviour: separate.

Per [prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md): components compose. Each has its own owner, version, eval.

For agent-shaped features, much of the monolith should be in tools / schemas / workflow steps rather than in one prompt.

### 7.5 The case study

A customer-service AI had a 6,500-token system prompt. Every call paid for all 6,500 tokens regardless of the customer's question. Quality issues traced to prompt-section conflicts (different sections instructing differently).

Refactored: 1,500-token base system prompt + per-intent specialised prompts (loaded based on classifier) + tool-specific instructions (loaded based on tool selection). Average tokens per call dropped to 2,800; quality up; debugging vastly easier.

---

## 8. Worked Meridian example

Meridian's pattern-anti-pattern audit cadence and findings.

### 8.1 The audit cadence

Quarterly pattern audit. The team walks through each anti-pattern; checks whether any AI feature exhibits it.

### 8.2 Findings over 18 months

- **#1 ("agent for everything").** Never triggered for new features (per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md)); audit confirms shape choices remain right.

- **#2 ("graph RAG because data is graph-shaped").** Triggered Q2-25 prototype evaluation; rejected per [knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md) section 8.1. Two small graphs adopted where genuine traversal value existed.

- **#3 ("fine-tune as first move").** Never triggered as "first move." One fine-tune adopted Q2-25 after few-shot example count exceeded 20 + use case proven stable + cost justification met.

- **#4 ("router as first move").** *Triggered once*; corrected. See section 8.3.

- **#5 ("every step is an LLM call").** Never triggered for major features. One audit found a workflow with unnecessary LLM call for date formatting; replaced with deterministic.

- **#6 ("monolithic prompt").** *Triggered once*; corrected. See section 8.4.

### 8.3 Anti-pattern #4 trigger and correction

Q3-25 audit identified that the analytics-warehouse copilot launched with a 4-tier router (Haiku → Sonnet → Opus + specialty paths). The router was sending too many to Opus; cost was 2× projected.

Investigation: the original team had over-engineered the router on speculation; production traffic showed simpler patterns.

Corrective:

1. Disabled the 4-tier router; switched to Sonnet-only.
2. Collected 6 weeks of production data.
3. Re-introduced 2-tier routing (Haiku for orchestration / classification; Sonnet for SQL generation and synthesis) based on the data.

Outcome: cost dropped to 40% of original (60% reduction); quality stable.

### 8.4 Anti-pattern #6 trigger and correction

Q4-25 audit identified that an early prototype care-coordinator iteration had a 5,200-token system prompt. Production rollout was blocked by per-request cost above target.

Investigation: the prompt had accumulated 9 months of additions; many sections weren't relevant to most requests.

Corrective per [prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md):

1. Extracted shared components (HIPAA disclaimer, escalation patterns, citation format).
2. Per-intent specialised prompts (loaded based on classifier).
3. Tool descriptions moved to tool registry rather than embedded in prompt.

Outcome: average prompt size 2,400 tokens (54% reduction); per-request cost within target; quality up (less attention division).

### 8.5 The audit's value

Two corrective actions in 18 months. Both caught before becoming notable incidents. Each saved months of operational pain that would have followed unchecked.

### 8.6 The new-feature design review

For each new AI feature, design review uses the six as a checklist. Anti-pattern signals trigger discussion; some are caught at design time and never reach production.

### 8.7 The "respect the pattern" framing

Each anti-pattern has a corresponding *pattern* (the corrective is itself a pattern). The framing helps:

- #1's corrective: workflow / hybrid / simpler shape.
- #2's corrective: hybrid retrieval + reranking.
- #3's corrective: prompt + retrieval + few-shot first.
- #4's corrective: ship single-tier; route after data.
- #5's corrective: deterministic where deterministic suffices.
- #6's corrective: composed prompts per [prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md).

The team's vocabulary: when an anti-pattern is identified, the corrective pattern is named, scoped, scheduled.

---

## 9. Findings (sprint-assignable)

These findings are cross-cutting. Each maps to one or more anti-patterns; the recommended actions reference the per-topic docs.

### ARCH-PAP-001 — Severity: Critical
**Finding.** Production AI feature is an agent loop where production traces show 80%+ identical paths (anti-pattern #1).
**Recommendation.** Refactor to workflow / hybrid per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md); section 2.4.
**Owner.** architecture + feature team, sprint N+1.

### ARCH-PAP-002 — Severity: Critical
**Finding.** Knowledge graph in production with unclear value; maintenance burden ongoing (anti-pattern #2).
**Recommendation.** Re-evaluate per [knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md); retire if not justified.
**Owner.** architecture + feature team, sprint N+1.

### ARCH-PAP-003 — Severity: Critical
**Finding.** New AI feature roadmap calls for fine-tune as first step; corrective pattern not evaluated (anti-pattern #3).
**Recommendation.** Prompt + retrieval + few-shot first per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md); fine-tune only when criteria met.
**Owner.** architecture + feature team, sprint N+1.

### ARCH-PAP-004 — Severity: High
**Finding.** Router engineered before production data; complexity ahead of need (anti-pattern #4).
**Recommendation.** Single-tier first; route after data per [multi-model-orchestration.md](./multi-model-orchestration.md) section 3.8.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-PAP-005 — Severity: High
**Finding.** AI workflow has steps doing deterministic work (parsing, formatting, lookups); unnecessary LLM cost (anti-pattern #5).
**Recommendation.** Per-step audit per section 6.4; replace with deterministic where appropriate.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### ARCH-PAP-006 — Severity: High
**Finding.** Monolithic system prompt > 4000 tokens; quality and cost issues traceable to it (anti-pattern #6).
**Recommendation.** Decompose per [prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md).
**Owner.** ai-platform-eng + feature team, sprint N+2.

### ARCH-PAP-007 — Severity: High
**Finding.** Pattern audit not performed; quality / cost issues compound undetected.
**Recommendation.** Quarterly audit per section 8.1.
**Owner.** architecture + ai-platform-eng, sprint N+2.

### ARCH-PAP-008 — Severity: Medium
**Finding.** New AI features ship without pattern-anti-pattern review at design time.
**Recommendation.** Design review checklist per section 8.6.
**Owner.** architecture + tech leads, sprint N+3.

### ARCH-PAP-009 — Severity: Medium
**Finding.** "Agent" used colloquially for features that are workflows; vocabulary imprecise.
**Recommendation.** Per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md); precise shape naming.
**Owner.** architecture + tech leads, sprint N+3.

### ARCH-PAP-010 — Severity: Medium
**Finding.** Vendor pitch drove pattern adoption; team's actual workload not validated against the pattern.
**Recommendation.** Per-workload eval before pattern adoption.
**Owner.** architecture, sprint N+3.

### ARCH-PAP-011 — Severity: Medium
**Finding.** Fine-tune in production but base-model upgrades not tracked; fine-tune may be on a deprecated base.
**Recommendation.** Deprecation watch per section 4.4; re-fine-tune cadence.
**Owner.** ml-eng + ai-platform-eng, sprint N+3.

### ARCH-PAP-012 — Severity: Medium
**Finding.** Router complexity exceeds feature complexity; debugging is router-debugging.
**Recommendation.** Per [multi-model-orchestration.md](./multi-model-orchestration.md); simplify or retire router.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### ARCH-PAP-013 — Severity: Medium
**Finding.** Knowledge graph data ingestion broken; stale data in production but undetected.
**Recommendation.** Per [knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md) section 5.4.
**Owner.** ai-platform-eng + data team, sprint N+3.

### ARCH-PAP-014 — Severity: Medium
**Finding.** Prompts grown over time without periodic cleanup; cumulative cruft.
**Recommendation.** Prompt audit per [few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) section 5.4; periodic cleanup.
**Owner.** ai-platform-eng + feature teams, sprint N+4.

### ARCH-PAP-015 — Severity: Medium
**Finding.** Workflow has redundant LLM-call steps; one classifier doing same work as another's first step.
**Recommendation.** Per section 6.4; consolidate redundant steps.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-PAP-016 — Severity: Low
**Finding.** Team's vocabulary for shape patterns is imprecise; "AI feature" used for all shapes.
**Recommendation.** Shape vocabulary in design discussions per [agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md).
**Owner.** architecture + tech leads, sprint N+4.

### ARCH-PAP-017 — Severity: Low
**Finding.** Anti-pattern catalog not maintained; latest patterns not incorporated.
**Recommendation.** Annual review of the catalog; update with new patterns observed.
**Owner.** architecture, sprint N+5.

### ARCH-PAP-018 — Severity: Low
**Finding.** Cross-team anti-pattern findings not shared; same mistakes repeated across teams.
**Recommendation.** Quarterly cross-team audit summary; learnings shared.
**Owner.** architecture + tech leads, sprint N+5.

---

## 10. Adoption sequencing checklist (the architecture audit)

The audit checklist for an existing AI architecture. Walk through each of the six; record whether the system exhibits the pattern; for each "yes," follow the corrective.

### 10.1 The audit walkthrough

For each of the six anti-patterns:

- [ ] **#1 "Agent for everything."** For each AI feature, what's the shape? Production traces analyse: how varied are the trajectories? Outcome: features whose shapes are wrong identified.
- [ ] **#2 "Graph RAG because data is graph-shaped."** Any knowledge graphs in production? Are they earning their maintenance cost? Outcome: KG audit; retirement candidates identified.
- [ ] **#3 "Fine-tune as first move."** Roadmap has fine-tunes planned? Have alternatives been evaluated? Outcome: fine-tune decisions justified or deferred.
- [ ] **#4 "Router as first move."** Routers in production: were they justified by data or speculation? Outcome: routers ahead of data identified; simplification scoped.
- [ ] **#5 "Every step is an LLM call."** Workflows audited per step: which steps need a model? Outcome: deterministic candidates identified.
- [ ] **#6 "Monolithic prompt."** Prompt sizes per feature: any > 3000 tokens? Outcome: decomposition candidates.

The walkthrough takes ~2-4 hours per AI feature; quarterly cadence.

### 10.2 The remediation sequencing

For each "yes," remediation per the corrective. Typical sequencing:

- Sprint N+1: critical-severity findings (anti-patterns #1-3 are usually critical when present).
- Sprint N+2: high-severity (anti-patterns #4-6).
- Sprint N+3: medium-severity (process / discipline gaps).
- Sprint N+4: low-severity (documentation, vocabulary).

A team starting from a baseline (no audit ever) can reach a healthy state in 2-3 quarters.

### 10.3 The new-feature checklist

For each new AI feature, design review uses the six as a checklist. The reviewer signs off only when none of the patterns are present at design time.

### 10.4 The portfolio view

| Feature | #1 | #2 | #3 | #4 | #5 | #6 |
| --- | --- | --- | --- | --- | --- | --- |
| care-coordinator | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (after Q4 fix) |
| patient-summary | n/a | ✓ | ✓ | ✓ | ✓ | ✓ |
| analytics-copilot | ✓ | ✓ | ✓ | ✓ (after Q3 fix) | ✓ | ✓ |
| patient-API copilot | n/a | ✓ | ✓ | ✓ | ✓ | ✓ |

The portfolio surfaces cross-feature trends; platform team can prioritise gaps across multiple features.

---

## 11. References

- [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) — RAG decision; anti-pattern #2 is a RAG-shape anti-pattern.
- [agent-topologies.md](./agent-topologies.md) — agent topology choices; anti-pattern #1 is agent-shape anti-pattern.
- [multi-model-orchestration.md](./multi-model-orchestration.md) — orchestration; anti-pattern #4 is router-shape anti-pattern.
- [structured-output-patterns.md](./structured-output-patterns.md) — schema discipline.
- [hybrid-retrieval-patterns.md](./hybrid-retrieval-patterns.md) — hybrid retrieval; corrective for some anti-pattern #2 cases.
- [data-architecture-for-ai/knowledge-graph-augmentation.md](../data-architecture-for-ai/knowledge-graph-augmentation.md) — KG-specific corrective per anti-pattern #2.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-vs-workflow-decision.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-vs-workflow-decision.md) — corrective for anti-pattern #1.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-anti-patterns.md) — agent-specific anti-patterns; overlaps with #1.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/few-shot-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/few-shot-engineering.md) — corrective for anti-pattern #3.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/prompt-libraries-and-components.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-libraries-and-components.md) — corrective for anti-pattern #6.
- Sibling repo: [ai-engineering-reference-architecture/prompt-engineering/prompt-anti-patterns.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/prompt-engineering/prompt-anti-patterns.md) — prompt-specific anti-patterns; overlaps with #6.
- Sibling repo: [ai-engineering-reference-architecture/model-lifecycle/](https://github.com/jeremiahredden/ai-engineering-reference-architecture/tree/main/model-lifecycle) — fine-tune operational depth.
- "An anti-pattern catalogue is a teaching tool" — same precedent as agent-anti-patterns.md and prompt-anti-patterns.md.
