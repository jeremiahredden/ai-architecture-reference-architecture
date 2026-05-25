# Agent Topologies

> **Audience.** Architects choosing the topology for an AI system that involves more than a single LLM call. Tech leads reviewing whether an existing system's topology is the right one. **Scope.** The *architectural* shape decision — single agent vs supervisor/worker vs pipeline vs hierarchical vs swarm vs workflow-with-LLM-steps — not the engineering practice for operating any of them (the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `agent-engineering/` folder owns that). **Worked client.** Meridian Health, the fictional regulated healthcare SaaS used across the sibling reference repos. **Companion doc.** [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) for the retrieval-pattern decision, which often interacts with the topology decision.

---

## 1. Why this document exists

The single most consequential decision in any non-trivial AI system in 2026 is the topology. It governs cost, latency, debuggability, observability, eval surface, blast radius, and on-call experience. Get it right and the system is operable on the same rotation as the rest of the platform; get it wrong and the system is a 24/7 specialty operation forever.

Most teams pick the topology by intuition — "this feels agent-shaped" — and discover the consequences six months later when costs blow out, the trace becomes unreadable, or the eval suite cannot keep up with the trajectory variance. The corrective discipline is the same as in [rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md): *pick the simplest topology that meets the requirements; graduate to more complex ones only on measured signal*. The signals are the failure modes the simpler topology cannot address, demonstrated on the actual workload by evaluation, not anticipated by the architecture diagram.

So this document is organized in roughly-increasing order of operational complexity:

1. **No agent at all** (a deterministic workflow with LLM steps embedded).
2. **Single-agent loop** (ReAct / tool-use loop).
3. **Plan-then-execute** (planning separated from execution).
4. **Reflection / self-critique** (single-agent with a self-review pass).
5. **Supervisor / worker** (one supervisor dispatches to specialized workers).
6. **Pipeline** (sequence of specialized agents, output of one is input to the next).
7. **Map-reduce** (parallel agents over partitions, aggregator at the end).
8. **Hierarchical** (supervisor / sub-supervisor / workers).
9. **Tree-of-thought / search** (parallel exploration of solution paths).
10. **Swarm / free-form multi-agent** (often the wrong answer, included for completeness).

Each entry includes the topology shape, what kind of workload it fits, the failure modes that justify graduating to the next step, and the non-functional profile (cost, latency, observability, eval shape). The catalogue is followed by an explicit *graduation-signals* table, three worked Meridian topologies (Care Coordinator = supervisor/worker; patient-API assist = workflow-with-LLM-steps; analytics copilot = workflow with a contained single-agent loop), an anti-pattern checklist, and twenty sprint-assignable findings.

---

## 2. The questions to answer before picking a topology

Five questions. Answer them in writing before reading the catalogue; the right topology usually becomes obvious.

1. **Is the plan known up front, or does it emerge from intermediate results?** If the plan is known (e.g., "classify → retrieve → answer → format"), this is workflow-shaped. If the plan emerges as the system works ("retrieve, see what came back, decide whether to retrieve again or answer or escalate"), this is agent-shaped.
2. **How variable is the step count across runs?** Workflows have bounded step counts. Agents have variable step counts. If the variance is wide (one run takes 1 step, another takes 12), the topology needs to handle that variance explicitly.
3. **What is the latency budget?** A sub-3-second budget rules out multi-step agent loops. A 30-second budget allows most topologies. A 5-minute async budget allows almost anything.
4. **What is the cost budget per interaction?** A 5-cent budget rules out hierarchical, swarm, and most agent loops. A 50-cent budget allows most patterns. Above $1/interaction, exotic patterns become economic.
5. **How parallel is the work?** Strictly sequential work argues for single-agent or pipeline. Embarrassingly parallel work over partitions argues for map-reduce. Mixed-shape work where some sub-tasks parallelize and others do not argues for supervisor/worker.

The answers usually point to two or three candidate topologies, and the catalogue below lets you compare them on the dimensions that actually differ.

---

## 3. The topology catalogue

### 3.1 No agent at all (workflow with LLM steps)

**Shape.** A deterministic workflow (state machine, DAG, or imperative orchestration) where some steps invoke an LLM as a sub-routine. The LLM's job is the local sub-task; the orchestration is the team's code.

```
┌──────────────┐    ┌────────────┐    ┌──────────────┐    ┌─────────┐
│ Classify     │ →  │ Retrieve   │ →  │ Answer (LLM) │ →  │ Format  │
│ (LLM or rule)│    │ (system)   │    │              │    │ (rule)  │
└──────────────┘    └────────────┘    └──────────────┘    └─────────┘
```

**Sweet-spot workload.** The set of sub-tasks is known and stable. The order of sub-tasks is fixed or follows a small enumerable set of branches. Each sub-task has well-defined input and output. The system needs predictable cost and latency. The system needs to be reproducible from a release artifact for audit purposes.

**Why it should be the default.**
- Bounded cost and latency. The workflow has at most N LLM calls per invocation, where N is enumerable from the workflow definition.
- Bounded blast radius. A workflow cannot loop, cannot decide to call a tool 40 times, cannot escalate to a more expensive model unbidden.
- Debuggable. Each step is a known location with known inputs and outputs. Failures are localizable.
- Observable with conventional tools. Same patterns as any other workflow system.
- Reproducible. The same input produces the same trajectory (modulo LLM nondeterminism within a step).

**What it cannot do.**
- Open-ended exploration. A workflow cannot decide it needs a second retrieval after seeing the first; that would be an agent step.
- Plan emergence. A workflow's plan is its definition; the runtime cannot generate a new plan.

**Cost / latency / eval shape.** Cheapest topology. Each invocation costs the sum of its LLM steps; latency is the sum of step latencies (or max, for parallel branches). Eval surface is per-step — evaluate each LLM step's quality independently, plus end-to-end outcome eval.

**Meridian example.** The patient-API AI assist is workflow-shaped: classify the question, retrieve from patient-education content, generate the answer with citations, validate output for PHI re-screen, return. No agent loop, no plan emergence, predictable cost per interaction.

### 3.2 Single-agent loop (ReAct)

**Shape.** An LLM in a loop: think → decide on an action (typically a tool call) → observe the result → loop. The loop terminates when the model returns a final answer (no more tool calls) or when a budget is breached.

```
                    ┌────────────────────────────────────┐
                    │                                    ▼
   user input → LLM thinks → decides tool call → tool runs → observation
                    │                                                 │
                    │              (loop)                              │
                    └─────────────────────────────────────────────────┘
                                       │
                                       ▼
                                final answer
```

**Sweet-spot workload.** Multi-step problems where the next action depends on the previous observation. Question answering with retrieval where a second retrieval may be needed after seeing the first. Bounded-domain tool use (a small, well-curated tool set). Conversational systems where the agent needs to fetch information from multiple tools as the conversation develops.

**What it adds over a workflow.**
- The plan emerges from the work. The model decides what to do next based on what it has seen, not based on a pre-built state machine.
- Adaptive recovery. If a tool returned an unexpected result, the model can change strategy without intervention.
- Single architectural surface. One loop, one set of tools, one prompt. Simpler than supervisor/worker for problems that do not need specialization.

**What it cannot do well.**
- Specialization. The single LLM is responsible for everything — planning, reasoning, formatting, tool selection, refusal decisions. Quality on each sub-task is capped by the single model's all-around capability.
- Predictable cost. Without explicit budgets, a confused agent can loop for many turns; even with budgets, cost varies meaningfully across runs.
- Predictable latency. Same as cost — the loop length is data-dependent.
- Cheap workers. Every step uses the same model, even simple ones that a Haiku-class model would handle equally well.

**Cost / latency / eval shape.** 3x–10x the cost and latency of a workflow with embedded LLM steps, depending on average loop length. Eval surface is trajectory + outcome (the single-call eval is insufficient because the loop's intermediate steps matter). Cost-as-circuit-breaker is mandatory.

**Meridian example.** Not used in production for any Care Coordinator path; considered and rejected because the supervisor/worker pattern's specialization measurably lifted quality on clinical reasoning. Used in early prototyping of the analytics copilot, then graduated to workflow-with-contained-agent-loop for the production architecture.

### 3.3 Plan-then-execute

**Shape.** A planning pass (often using a more capable model) produces an explicit plan as a structured artifact. An execution pass (often using a cheaper model) executes the plan. The planner may be invoked again if execution surfaces a need to replan.

```
            ┌──────────────────┐
user input →│ Planner (Opus)   │── plan (structured) ──┐
            └──────────────────┘                       ▼
                                            ┌──────────────────┐
                                            │ Executor (Sonnet)│── result
                                            │ runs plan steps  │
                                            └──────────────────┘
                                                     │
                                            (optional replan loop)
                                                     │
                                                     ▼
                                              final answer
```

**Sweet-spot workload.** Problems where planning requires significantly more capability than execution. Research-shaped tasks where the plan is itself a deliverable. Workloads where the plan can be reviewed (by a human or by validation logic) before execution begins.

**What it adds.**
- Planning quality scales with the planner model; execution cost scales with the executor model. The cost-quality trade-off is tunable per axis.
- Plan as an inspectable artifact. The plan can be validated, audited, modified, or rolled back before any action is taken.
- Naturally supports HITL: the human approves the plan before execution.

**What it cannot do well.**
- Highly emergent plans. If the plan changes dramatically with each new observation, the replanning overhead may exceed the planning benefit.
- Tight-loop iterative work. The plan/execute split adds latency that pure ReAct does not have.

**Cost / latency / eval shape.** Similar to ReAct overall, but the cost is distributed differently — more cost in the planner, less in the executor. Eval surface is plan-quality + execution-quality + outcome. The plan-as-artifact makes trajectory eval easier than in pure ReAct.

**Meridian example.** Considered for the async coordination tasks; not adopted because the workflow-with-fan-out pattern was operationally simpler at the achieved quality level. Could be revisited if the coordination tasks grow into genuinely-multi-step research-shaped work.

### 3.4 Reflection / self-critique

**Shape.** A single-agent loop with an additional step: after producing an output, the model critiques its own output against a rubric or against the original requirements, and revises if the critique surfaces a problem.

```
                    ┌─ critique pass ←──── output ──┐
                    │       │                       │
                    ▼       │ (if fail, revise)     │
   user input → LLM ─── output ──→ if pass → final answer
                    ▲                                
                    │ (revise based on critique)     
                    └──────────                     
```

**Sweet-spot workload.** Quality-sensitive workloads where a second pass can catch errors the first pass made. Reasoning-heavy problems where the model benefits from "show your work then check it." Workloads where a self-review pass is cheaper than catching the same errors downstream.

**What it adds.**
- Quality lift on workloads where the model can recognize its own errors when re-reading. Most current frontier models can do this for many error classes.
- A natural surface for structured criteria (factuality check, completeness check, format check) to be evaluated.

**What it cannot do well.**
- The model that produced the output critiques the output. The critique inherits the model's blind spots. A different-model critique (cross-critique) is often higher quality, at proportional additional cost.
- Adds 50%–100% to per-interaction cost and latency. Earns it on quality-sensitive workloads, does not on others.

**Cost / latency / eval shape.** 1.5x–2x baseline. Eval surface is outcome — but with a separate measurement of "did the critique pass improve quality" so the cost can be justified or removed.

**Meridian example.** Not used in production. Considered for the clinical-knowledge worker; rejected because the LLM-as-judge eval pattern already provides the critique signal at lower cost than a runtime self-critique. The pattern is on the watch-list for high-stakes question classes where production critique could pay back.

### 3.5 Supervisor / worker

**Shape.** A supervisor agent receives the user request, plans the sub-tasks, and dispatches each sub-task to a specialized worker. Workers are typically single-call LLMs with task-specific system prompts, often on a cheaper model tier than the supervisor. The supervisor consolidates worker outputs into the final response.

```
                                    user input
                                        │
                                        ▼
                        ┌──────────────────────────────┐
                        │   Supervisor (Opus-class)    │
                        │   plans + consolidates       │
                        └────┬────────┬──────────┬─────┘
                             │        │          │
              ┌──────────────▼┐  ┌────▼──────┐  ┌▼─────────────┐
              │ Worker A      │  │ Worker B  │  │ Worker C     │
              │ (Opus-class)  │  │ (Sonnet)  │  │ (Haiku)      │
              │ clinical      │  │ drafting  │  │ classifier   │
              │ reasoning     │  │           │  │              │
              └───────────────┘  └───────────┘  └──────────────┘
                             │        │          │
                             └────────▼──────────┘
                                      │
                                Supervisor consolidates
                                      │
                                      ▼
                                final answer
```

**Sweet-spot workload.** Multi-faceted problems where different sub-tasks benefit from different model tiers, different system prompts, different specializations. Workloads where the parallelism is real (multiple workers can run concurrently). Systems large enough that specialization pays back the coordination cost (typically: enough question-class variety to justify multiple workers).

**What it adds over single-agent.**
- Specialization. The clinical-knowledge worker gets a system prompt that focuses entirely on clinical reasoning; the drafting worker gets a system prompt that focuses on patient-appropriate composition; the classifier gets a tiny prompt that focuses on one decision.
- Cost optimization through tier routing. Cheap models handle the cheap tasks (classification, formatting); expensive models handle the expensive tasks (clinical reasoning). The aggregate cost is meaningfully lower than putting everything on the most-capable tier.
- Cleaner debuggability per sub-task. Each worker's trace span is its own scoped concern.

**What it cannot do well.**
- More moving parts. Three workers + a supervisor = four prompts to version, four model versions to pin, four observability streams.
- The supervisor is a single point of coordination. A confused supervisor produces confused workers.
- Coordination cost. The supervisor's dispatch + consolidation overhead is non-trivial.

**Cost / latency / eval shape.** Cost is supervisor + sum of worker calls (~2–4x a single-agent baseline, often much less than single-agent-on-best-tier). Latency is supervisor + parallel-max of workers + supervisor consolidation. Eval surface is per-worker + supervisor + outcome.

**Meridian example.** The Care Coordinator. Three worker specializations (clinical-knowledge on Opus, drafting on Sonnet, classifier on Haiku) plus a query rewriter (Haiku, conditional). The pattern was chosen because (a) clinical reasoning earns the more capable model and the other sub-tasks do not, and (b) the per-worker trace spans give the debugging surface needed for clinical-grade audit.

### 3.6 Pipeline

**Shape.** A → B → C in sequence. Each agent's output is the next agent's input. Each agent is specialized for its stage. No backward communication (an agent later in the pipeline does not feed back to an earlier one).

```
user input → Agent A → Agent B → Agent C → final answer
            (extract)  (analyze)  (compose)
```

**Sweet-spot workload.** Staged transformations where each stage is genuinely different work and where the stages are sequentially dependent. Document processing pipelines (extract → classify → enrich → summarize). Data preparation pipelines for downstream tasks.

**What it adds.**
- Clear separation of concerns; each stage is its own prompt, its own eval surface, its own optimization target.
- Stages can be developed and deployed independently.
- Failure modes are stage-localized; the cause of a bad final output is usually attributable to a specific stage.

**What it cannot do well.**
- Inflexible to plan changes. If stage B realizes that stage A needs to redo its work with different parameters, the pipeline shape does not natively support that.
- Sequential latency. Each stage adds its full latency to the total.
- Error compounding. An error in stage A propagates through B and C; downstream stages do not know that A was wrong.

**Cost / latency / eval shape.** Cost is sum of stages. Latency is sum of stages. Eval surface is per-stage + end-to-end.

**Meridian example.** Used in the document-ingestion pipeline that feeds the retrieval corpus (parse → chunk → classify → embed → index), but that is a data pipeline rather than an agent system in the conversational sense. Not used in any of the user-facing agent paths.

### 3.7 Map-reduce (parallel fan-out)

**Shape.** A single input is partitioned (the "map") and each partition is processed by an independent agent. Results are aggregated by a final agent (the "reduce"). The pattern can be one-level (single map + single reduce) or recursive (map of maps).

```
                            user input + partitioning
                                       │
                  ┌────────────────────┼────────────────────┐
                  ▼                    ▼                    ▼
            Agent (part 1)      Agent (part 2)        Agent (part N)
                  │                    │                    │
                  └────────────────────┼────────────────────┘
                                       ▼
                                Reducer agent
                                       │
                                       ▼
                                  final answer
```

**Sweet-spot workload.** Embarrassingly parallel work over partitions: summarize each of these 100 documents, classify each of these 1,000 messages, analyze each patient in a cohort, generate per-customer drafts. The fan-out factor is high enough that parallel execution is faster than sequential.

**What it adds.**
- Linear speedup with fan-out (up to provider rate limits). A 100-patient batch completes in roughly the time of one patient.
- Per-partition isolation: one partition's failure does not affect others.
- Naturally maps to durable-workflow infrastructure (Step Functions, Temporal).

**What it cannot do well.**
- Inter-partition dependencies. If processing partition 2 requires the result of processing partition 1, map-reduce is the wrong shape.
- Fan-out cost. 100 parallel agents at $0.20 each cost $20 per invocation. Cost-budget discipline is non-negotiable.
- Result aggregation can become its own bottleneck if the reducer needs to fit all partition results in context.

**Cost / latency / eval shape.** Cost is N × per-partition + reducer. Latency is per-partition (in parallel) + reducer. Eval surface is per-partition + aggregation correctness + end-to-end.

**Meridian example.** The async coordination tasks. The care coordinator initiates "for every CHF patient discharged in the last 7 days, draft an outreach message and a suggested appointment slot." The task is map-reduce: one per-patient agent per patient (run in parallel up to a concurrency cap), then a summary reducer that returns the batch result to the coordinator. Each per-patient agent is itself a workflow with embedded LLM steps; the map-reduce is the outer shape.

### 3.8 Hierarchical (use sparingly)

**Shape.** A top-level supervisor dispatches to sub-supervisors, each of which dispatches to its own set of workers. Two or more levels of nesting.

```
                            top supervisor
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                                   ▼
        sub-supervisor                     sub-supervisor
        ┌───┴───┐                             ┌───┴───┐
        ▼       ▼                             ▼       ▼
     worker  worker                        worker  worker
```

**Sweet-spot workload.** Genuinely-hierarchical problem decomposition where the second level of decomposition is meaningfully different from the first. Very large systems where the cognitive load of one supervisor managing many workers is too high.

**What it adds.**
- Scales beyond what one supervisor can coordinate.
- Per-sub-system optimization: each sub-supervisor can be tuned for its slice of the problem.

**What it cannot do well.**
- Coordination overhead at two levels. Each level adds dispatch + consolidation latency and cost.
- Debuggability degrades with nesting depth. A trace that spans three levels of agent hierarchy is hard to read.
- Most problems are not genuinely hierarchical. The two-level decomposition often turns out to be artificial, and the system would work as well or better with a flat supervisor / worker.

**Cost / latency / eval shape.** Cost and latency scale with the nesting; eval surface needs per-level coverage which is expensive.

**Meridian example.** Not used and not currently planned. Revisit only if the Care Coordinator's supervisor / worker pattern ceases to scale with question-class growth — for example, if specialized sub-domains (cardiology vs oncology vs neurology) developed sufficiently different patterns that splitting the clinical-knowledge worker into per-domain sub-workers became justified.

### 3.9 Tree-of-thought / search

**Shape.** The agent explores multiple solution paths in parallel. Each path is independently developed; a scoring function (often itself an LLM) selects the best one or merges across paths.

```
                       initial state
                       /     │     \
                      /      │      \
               branch A   branch B   branch C
                  │          │          │
                  ▼          ▼          ▼
              develop     develop    develop
                  │          │          │
                  ▼          ▼          ▼
                       score and select
                              │
                              ▼
                         final answer
```

**Sweet-spot workload.** Research-shaped problems where the search space is large enough that single-path reasoning is likely to miss the best answer. Mathematical / logical problems where multiple solution strategies exist and the best one is not obvious upfront. Code-generation problems where multiple candidate implementations are worth evaluating.

**What it adds.**
- Higher solution quality on hard reasoning problems by avoiding early commitment to a single path.
- The scoring function makes the search bias explicit and tunable.

**What it cannot do well.**
- Expensive. N parallel branches cost N times a single-path baseline.
- Latency is bounded by the slowest branch (or the scoring pass), not by the average branch.
- Search-space pruning is non-trivial; without it, the branching factor explodes.

**Cost / latency / eval shape.** Cost is N × baseline + scoring. Latency is max-branch + scoring. Eval surface is per-branch + final selection.

**Meridian example.** Not used. Considered as an academic exercise; the workloads in scope do not have the multi-strategy character that tree-of-thought specifically addresses.

### 3.10 Swarm / free-form multi-agent (almost always wrong)

**Shape.** Multiple agents communicate freely with each other, negotiating, debating, dividing work emergently. No fixed supervisor.

```
                       Agent A ──── Agent B
                          \   /     /
                           \ /     /
                          Agent C
                           / \
                          /   \
                       Agent D ─── Agent E
                       (all communicate freely)
```

**When it is the right shape.** Genuinely simulation-shaped problems where the emergent behavior of multiple agents is itself the deliverable (multi-agent reinforcement learning research, social-simulation studies, certain game-theoretic problems).

**Why it is almost always wrong for product workloads.**
- The same work can almost always be done by a single agent with multiple tool sources, or by a supervisor/worker pattern with deterministic dispatch.
- Coordination overhead is unbounded. Two agents negotiating produce more tokens than one agent reasoning, with comparable or worse quality.
- Debuggability is poor. The decision that led to the final output is distributed across multiple agents' reasoning, and the trace is hard to reconstruct.
- Cost is unbounded; latency is unbounded; eval surface is unbounded.

**Meridian example.** Considered briefly as a curiosity during early exploration; immediately rejected. Pattern is included in this document only so practitioners can name it when they encounter the temptation, and so they have a reference for why to choose differently.

---

## 4. The graduation signals

The discipline is "what is the smallest topology that meets requirements, and what signals trigger graduating to a more complex one?" The table names them.

| Currently using | Graduate to | Trigger signal (observed in eval / production, not anticipated in design) |
|---|---|---|
| **Workflow with LLM steps** | **Single-agent loop** | The eval surfaces cases where a second retrieval or a second reasoning pass is required and the workflow cannot accommodate it without re-architecture. |
| **Single-agent loop** | **Plan-then-execute** | Planning quality is the bottleneck (frontier-tier model on planning lifts quality measurably); execution is well-bounded and cheaper with a smaller model. |
| **Single-agent loop** | **Reflection / self-critique** | Production reveals a class of errors the model can recognize when re-reading its own output; the cost of a critique pass earns its quality lift on those errors. |
| **Single-agent loop** | **Supervisor / worker** | Eval shows quality is capped by the single model trying to be all things; specializing sub-tasks (clinical-reasoning vs drafting vs classification) lifts quality on each. |
| **Supervisor / worker** | **Pipeline (per-stage)** | Sub-tasks are sequentially dependent in a stable pattern that does not need plan emergence; a pipeline is simpler and faster than supervisor-dispatch for the stable sequence. |
| **Single-agent loop** | **Map-reduce** | Workload involves embarrassingly-parallel work over partitions (per-patient, per-document, per-customer); single-agent sequential execution is unnecessarily slow. |
| **Supervisor / worker** | **Hierarchical** | Question-class variety exceeded what one supervisor can coordinate; second level of specialization is genuinely different work, not artificial nesting. |
| **Single-agent loop** | **Tree-of-thought / search** | Workload is genuinely multi-strategy research-shaped (math, code generation, complex planning); single-path reasoning has a quality ceiling that branch-then-select breaks through. |
| **Anything** | **Swarm** | Almost never. If you find yourself here, the answer is probably a supervisor / worker or a single-agent with more tools. |

Two patterns to *avoid graduating to* on intuition alone:

- **Supervisor / worker before measuring.** Specialization is appealing in theory. Build the single-agent first. Measure the quality ceiling. Graduate only if the eval shows that ceiling matters.
- **Map-reduce when fan-out is artificial.** If the "partitions" are not genuinely independent (they share state, share retrieval, share user-facing output), map-reduce will surface coordination problems that a workflow would not have.

---

## 5. Worked Meridian Health topologies

Three Meridian workloads use three different topologies. The differences are not arbitrary — they fall out of the answers to the five questions in section 2.

### 5.1 Patient-API AI assist → Workflow with LLM steps

**Workload shape.** High-volume (~50K interactions/day), low-risk (no clinical decision support), single-turn lookup-shaped questions from patients. Latency budget: sub-3-second chat. Cost budget: ~$0.01 per interaction.

**Topology decision (the five questions).**
1. Plan known up front? Yes — classify → retrieve → answer → format → output-validate.
2. Step count variable? No — same step count every invocation (5 steps).
3. Latency budget? Sub-3-second, which rules out multi-step agent loops.
4. Cost budget? $0.01, which rules out anything past a couple of LLM calls.
5. Parallel? No — strictly sequential, single-shot.

**Architecture.** Workflow with two LLM steps embedded (classification on Haiku-tier, answer generation on Haiku-tier). The rest of the steps are deterministic code.

**Why not an agent.** An agent loop here would add 5–10x cost and latency for no quality benefit. The workload is not plan-emergent; the workflow can describe it completely.

### 5.2 Care Coordinator → Supervisor / worker

**Workload shape.** Lower volume (~3K interactions/day across 240 hospitals), higher risk (clinical decision support, side-effect operations), conversational with diverse question classes (lookups, multi-step coordination, drug-interaction analysis, drafting). Latency budget: 6 seconds for chat, 12 seconds for multi-step. Cost budget: ~$0.18 per interaction.

**Topology decision (the five questions).**
1. Plan known up front? Partially — the high-level decomposition (classify → retrieve → reason → optionally draft → consolidate) is known, but which workers run depends on the question class and conversational state.
2. Step count variable? Yes — a simple lookup completes in 3 worker calls; a multi-step coordination task uses 5–7.
3. Latency budget? 6 seconds, which accommodates supervisor-dispatched parallel worker calls.
4. Cost budget? $0.18, which accommodates a supervisor (Opus) + 2–4 workers across mixed tiers.
5. Parallel? Yes for the worker dispatches; the classifier and the conversational-context detection can run before the clinical-knowledge worker, and the drafting worker can run in parallel with consolidation.

**Architecture.** Supervisor (Opus) dispatches to four specialized workers (classifier on Haiku, clinical-knowledge on Opus, drafting on Sonnet, query rewriter on Haiku conditional). The drug-interaction graph is queried in parallel with the main retrieval when the classifier indicates interaction-class.

**Why not single-agent.** Eval showed the clinical-knowledge worker on Opus-tier with a focused prompt scored 12 percentage points higher than the same model handling the full agent loop. The specialization paid for itself.

**Why not hierarchical.** The four workers are flat — no second-level decomposition is needed. Adding a layer would add coordination cost without quality benefit.

**Why not pipeline.** The dispatch is not strictly sequential — the classifier informs the parallel dispatch of clinical-knowledge and (conditionally) the graph query, then the drafting worker runs in parallel with supervisor consolidation. A linear pipeline would serialize work that has natural parallelism.

**Cross-link.** Full architecture in [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md).

### 5.3 Analytics-warehouse copilot → Workflow with a contained single-agent loop

**Workload shape.** Lower volume (~400 interactions/day), variable risk (no PHI but re-identification risk on some joins), interactive (15–60s latency). Cost budget: ~$0.30 per query.

**Topology decision (the five questions).**
1. Plan known up front? The high-level workflow (retrieve schema → propose SQL → validate → execute → format) is known, but inside the schema-retrieval / SQL-revision steps the plan emerges based on what the schema looks like.
2. Step count variable? The workflow's outer step count is fixed; inside the schema-retrieval step the agent loop varies (1–4 retrieval iterations typically).
3. Latency budget? Permissive (15–60s), which allows agent-loop work inside the workflow.
4. Cost budget? $0.30, which allows several model calls per query.
5. Parallel? Mostly sequential; some retrieval parallelism inside the agent loop.

**Architecture.** A workflow whose skeleton is fixed (retrieve schema → propose SQL → validate → execute in sandbox → format), but where the schema-retrieval and SQL-revision steps are wrapped in a contained single-agent loop. The agent inside that step decides what to retrieve next based on what it has already seen; it cannot escape the bounded step of the outer workflow.

**Why this hybrid.** Pure workflow could not handle the cases where the first schema retrieval surfaces a need for additional schema. Pure agent loop would lose the audit and bounded-cost properties that the workflow provides. The hybrid keeps the strong properties of the workflow at the outer level (predictable structure, audit, retry, partial-result handling) while accommodating the genuine plan-emergence at the inner level.

**Cross-link.** Reference architecture (coming): `reference-systems/analytics-warehouse-copilot.md`.

---

## 6. Anti-patterns

The seven topology mistakes I see most often, with the corrective pattern.

### Anti-pattern 1: "Agent for everything"

The team picks an agent loop for every AI feature because the agent is the most general topology. The simple FAQ workflow that would cost $0.005 with two LLM steps costs $0.15 with a 5-turn agent loop, takes 8 seconds instead of 2, and has worse quality on the simple cases because the agent over-thinks them.

**Corrective.** The agent-vs-workflow decision in section 3.1 and the engineering sibling's `agent-engineering/agent-vs-workflow-decision.md`. Workflow is the default; agent is the exception that has to justify itself.

### Anti-pattern 2: "Supervisor / worker for a 2-step problem"

The team picks supervisor / worker because "supervisor / worker is the modern pattern." The two-step problem now has four moving parts (supervisor prompt, worker prompt, dispatch logic, consolidation logic) when a single-agent loop or a simple workflow would have had two.

**Corrective.** Pick supervisor / worker only when specialization is real — different sub-tasks benefit from different prompts, different model tiers, or different observability. Two-step problems are single-agent or workflow.

### Anti-pattern 3: "Free-form multi-agent debate"

The team builds a system where multiple agents debate each other to converge on an answer. Cost is unbounded; latency is unbounded; debugging is impossible because the answer was reached by emergent argument; quality is no better than a single agent with critical prompting.

**Corrective.** Supervisor / worker with deterministic dispatch, or single-agent with a critic-tool that surfaces objections without the back-and-forth.

### Anti-pattern 4: "Hierarchical for nesting's sake"

Two-level decomposition was added because the team imagined the problem was hierarchical. In practice, the two levels are artificial — the sub-supervisor is just routing to workers that the top supervisor could route to directly.

**Corrective.** Flatten. Hierarchical is correct only when the second level of decomposition is genuinely different work that benefits from its own coordination context.

### Anti-pattern 5: "Tree-of-thought without scoring discipline"

The team built tree-of-thought because it sounds powerful, but the scoring function is "ask a model which branch is best" without calibration. The system spends N times the baseline cost producing N branches and then selects one indistinguishable from random.

**Corrective.** Tree-of-thought needs an eval-validated scoring function or it is wasting parallel cost. If the scoring is not strong, drop back to single-path with the best model.

### Anti-pattern 6: "Map-reduce when partitions are coupled"

The team uses map-reduce over partitions that share state. The "parallel" agents discover they need information from each other partway through; coordination problems surface that a sequential or supervisor-coordinated design would not have.

**Corrective.** Either re-partition so the partitions are genuinely independent, or use a different topology (sequential workflow, supervisor / worker) that handles the coupling explicitly.

### Anti-pattern 7: "Topology selected by framework default"

The team adopted a framework (LangGraph, CrewAI, AutoGen) and built whatever topology the framework's tutorial example used. The chosen topology bears no specific relationship to the workload's actual requirements.

**Corrective.** Start from the workload — the five questions in section 2. Pick the topology. Then choose the framework that fits, not the other way around.

---

## 7. Findings (sprint-assignable)

The canonical agent-topology findings. Each has an ID (`ARCH-TOP-NNN`), severity, finding, recommendation, sprint owner template.

### ARCH-TOP-001 — Severity: High
**Finding.** An agent topology was chosen without measurable signal that a workflow would not have worked.
**Recommendation.** Build a workflow prototype; measure quality; graduate to agent only on signal.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TOP-002 — Severity: High
**Finding.** Supervisor / worker topology in use for a workload where the workers do not benefit from specialization (same model, same prompt-shape).
**Recommendation.** Collapse to single-agent loop or simple workflow; re-evaluate supervisor / worker if eval surfaces a quality cap that specialization would lift.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TOP-003 — Severity: High
**Finding.** Agent loop has no turn budget; runaway loops have been observed in production.
**Recommendation.** Add explicit turn / cost / time / tool-call budgets per the engineering sibling's `agent-engineering/agent-engineering-playbook.md` section 3.2.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-TOP-004 — Severity: High
**Finding.** Free-form multi-agent debate pattern in use; coordination cost is meaningful slice of total cost.
**Recommendation.** Refactor to supervisor / worker with deterministic dispatch; collapse "debate" to "supervisor consults specialist worker."
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-005 — Severity: High
**Finding.** Hierarchical agent topology in use with no clear specialization between supervisor levels.
**Recommendation.** Flatten to single supervisor + workers; document the criteria that would justify reintroducing the hierarchy.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-006 — Severity: High
**Finding.** Pipeline topology in use where backward communication is needed (a later stage needs to revise an earlier stage's output).
**Recommendation.** Replace with supervisor / worker or single-agent with revision tool; pipeline is the wrong shape for backward-dependency work.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-007 — Severity: High
**Finding.** Map-reduce over partitions that have shared state; coordination failures observed.
**Recommendation.** Re-partition or change topology to one that handles coupling (sequential workflow, supervisor / worker).
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TOP-008 — Severity: Medium
**Finding.** Reflection / self-critique pass in use without measurement that it lifts quality.
**Recommendation.** Add per-pass eval comparison (with vs without critique); keep the critique only if measured lift justifies the cost.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-009 — Severity: Medium
**Finding.** Tree-of-thought topology in use with an uncalibrated scoring function.
**Recommendation.** Either calibrate the scoring function against human-rated branches or drop back to single-path with the most capable model.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TOP-010 — Severity: Medium
**Finding.** Topology decision was driven by framework example rather than by workload requirements.
**Recommendation.** Walk through the five questions in section 2; choose topology from the catalogue; re-evaluate framework choice.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TOP-011 — Severity: Medium
**Finding.** Per-worker trace observability is not implemented in a supervisor / worker system; the trace shows only the supervisor's view.
**Recommendation.** Per-worker spans with the attributes from the engineering sibling's `observability-and-telemetry/agent-step-instrumentation.md`.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-TOP-012 — Severity: Medium
**Finding.** Cost is attributed at the supervisor level only; per-worker cost is not separately tracked.
**Recommendation.** Per-worker cost attribution; diagnose which workers drive cost trends.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-013 — Severity: Medium
**Finding.** Workflow with embedded LLM steps treats each LLM call as a side effect; rollback semantics are not defined.
**Recommendation.** Define which LLM calls are idempotent and which are not; for non-idempotent ones (side-effect tools), engineer compensation per `agent-engineering-playbook.md` section 6.4.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-014 — Severity: Medium
**Finding.** Plan-then-execute topology in use; planner output is not inspectable / auditable.
**Recommendation.** Persist the plan as a structured artifact in the trace; surface in audit log.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TOP-015 — Severity: Medium
**Finding.** Hybrid workflow-with-contained-agent-loop in use; the contained loop has no internal budget.
**Recommendation.** Add a per-contained-loop budget (turn / cost) that fails into the outer workflow's normal error path.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-016 — Severity: Medium
**Finding.** Eval suite does not distinguish trajectory failures (right answer, wrong path) from outcome failures (wrong answer).
**Recommendation.** Trajectory eval per the engineering sibling's `eval-engineering-playbook.md` section 11 / `eval-of-agents.md`.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-TOP-017 — Severity: Low
**Finding.** Topology change is being considered without an A/B comparison or shadow-traffic plan.
**Recommendation.** Run the candidate topology in shadow against current production traffic; compare on quality, cost, latency before any cutover.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-TOP-018 — Severity: Low
**Finding.** Topology decision is undocumented; the team cannot articulate why this topology was chosen.
**Recommendation.** Write a one-page ADR per section 2's five questions; commit alongside the system.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-TOP-019 — Severity: Low
**Finding.** Map-reduce reducer requires the full set of partition results in context; aggregation is hitting context-window limits.
**Recommendation.** Hierarchical reduce (intermediate aggregators) or per-partition pre-summarization before the final reducer.
**Owner.** ai-platform-eng, sprint N+5.

### ARCH-TOP-020 — Severity: Low
**Finding.** Multiple AI features in the platform use ad-hoc topologies; no platform-level guidance exists.
**Recommendation.** Document the platform's topology-decision template (this guide); make it a required checkpoint in the AI-feature design-review process.
**Owner.** ai-platform-eng team lead, sprint N+4.

---

## 8. Adoption sequencing checklist

For a team about to start an AI feature:

- [ ] **Sprint 0 — define.** Answer the five questions in section 2. Pick the candidate topology. Write a one-page ADR with the reasoning. Circulate for agreement before any code.
- [ ] **Sprint 1 — simplest first.** Build the simplest topology that the five questions point at. Workflow if workflow; single-agent if single-agent. Resist supervisor / worker on day one unless the specialization is obviously real.
- [ ] **Sprint 1 — eval suite.** Per the engineering sibling's `eval-engineering-playbook.md`. The eval is what tells you whether your topology hit its quality ceiling.
- [ ] **Sprint 2 — observability.** Per-step / per-worker spans with the attributes from `observability-and-telemetry/agent-step-instrumentation.md`. Topology choices are only assessable from trace data; instrument early.
- [ ] **Sprint 2 — budgets.** Cost / turn / time / tool-call budgets wired before launch (even for workflows — runaway costs are an agent-specific risk but cost monitoring is universal).
- [ ] **Sprint 3 — measure and reconsider.** With real production traffic, re-examine the topology against the graduation-signals table (section 4). If you're hitting a quality ceiling, graduate one step. If you're over-engineered, simplify.
- [ ] **Ongoing — discipline.** Every topology change is a project, not a refactor. Eval-validate before, canary or shadow during, monitor after.

A team that follows this sequencing arrives at the simplest topology that meets the requirements, with the observability and budget primitives in place to graduate the topology later if signal justifies it. A team that starts with the "modern" topology because it sounds right pays the operational tax for the lifetime of the system.

---

## 9. References

- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models* (2022) — foundational single-agent loop reference.
- Wang et al., *Plan-and-Solve Prompting* (2023) — plan-then-execute reference.
- Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning* (2023) — reflection / self-critique reference.
- Yao et al., *Tree of Thoughts: Deliberate Problem Solving with Large Language Models* (2023) — tree-of-thought reference.
- Wu et al., *AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation* (2023) — multi-agent framework reference.
- LangGraph, CrewAI, AutoGen, Vercel AI SDK, Anthropic SDK documentation — framework-level patterns for the topologies described here.
- This repo: [reference-patterns/rag-architecture-decision-guide.md](./rag-architecture-decision-guide.md) for the retrieval-pattern decision that often interacts with topology choice.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) for the supervisor / worker worked example end-to-end.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-engineering-playbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-engineering-playbook.md) for the engineering practice that operates whichever topology is chosen.
- Sibling repo: [ai-security-reference-architecture/agent-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) for the threat model and adversarial considerations across topologies.
