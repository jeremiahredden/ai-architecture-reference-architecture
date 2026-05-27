# Event-Driven AI Integration

> **Audience.** Architects landing AI features on an event bus. Platform teams whose ingestion pipeline now has an LLM step. Anyone whose AI feature has to fan out to multiple downstream consumers or whose upstream is a high-volume document/event stream. **Scope.** The *architectural* decision: when event-driven beats synchronous; the three AI roles on an event bus; the four event-driven AI patterns; ordering, idempotency, retry, and DLQ disciplines AI workloads need; schema discipline at event boundaries; how backpressure propagates in event-driven AI systems. Not the synchronous-vs-streaming-vs-async shape decision (see [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md)). Not the queue-fairness mechanics (see [backpressure-and-queueing.md](./backpressure-and-queueing.md)). Not the message-broker tutorial (SNS/SQS/EventBridge/Kafka are referenced as examples, not introduced). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Event-driven integration is not the default mental model for AI features. The default mental model is request/response: a user clicks, an LLM responds, the response renders. That model works fine for chat, copilots, and most user-facing assistants. It breaks for a different class of AI work:

- **High-volume ingestion.** Tens of thousands of documents per day, each needing classification, embedding, or summarization. No human is waiting; the work must be durable, resumable, and parallel.
- **Multi-consumer fan-out.** An LLM result that should land in the warehouse, the search index, the audit log, the customer notification stream, and the analytics dashboard — five consumers, one event.
- **Decoupled producers and consumers.** A document arrives in the storage tier; some weeks later, a new analytics team wants to classify all documents in scope. Event replay turns "weeks later" into "tomorrow" rather than "we backfill manually."
- **Long-running agent work.** An agent runs for forty seconds; the originating HTTP request times out at thirty. The agent's *completion* is the event; the originating context subscribes for it (or doesn't, and someone else does).

For these classes of work, request/response is the wrong shape. Force-fitting it produces the same well-known anti-patterns conventional systems produced before event-driven architecture matured: the fragile cron job, the lossy in-memory queue, the synchronous call that occasionally takes ten seconds and blocks the calling thread, the manual backfill script that nobody dares re-run.

But event-driven AI is not "regular event-driven plus a model call." AI workloads bring properties that the conventional event-driven playbook does not directly address:

- **A single AI event is expensive.** A retry isn't free — it costs cents to dollars per attempt. Aggressive retry policies suitable for stateless HTTP services produce shocking bills when applied to LLM-handling consumers.
- **A single AI event is slow.** Throughput per consumer is bound not by the broker but by the model provider's TPM and the consumer's concurrency. Tuning consumer parallelism in the absence of provider rate awareness produces upstream-induced 429 storms.
- **A single AI event is non-deterministic.** Same input, different output. Idempotency must be defined at the *side-effect* layer, not the *response* layer — because the response is never exactly the same twice.
- **A single AI event has a long blast radius.** A poisoned event (one that loops or one whose handler hallucinates a destructive tool call) can do real damage before a human notices. Dead-letter routing and circuit-breaking must be wired in from day one.

This document covers the architectural decisions that make event-driven AI integration durable: when to choose this shape, which AI role you're playing on the bus, which of the four common patterns you're building, and the ordering / idempotency / retry / schema disciplines the AI workload requires beyond what the broker provides.

This document is opinionated about four things:

1. **Event-driven is the right shape when no human is waiting *and* throughput or fan-out matters.** It is the wrong shape when latency is a UX constraint or when the workflow is single-shot and short. Defaulting to event-driven for chat is as wrong as defaulting to request/response for ingestion.
2. **Idempotency is not optional for AI consumers.** Retries happen. Brokers redeliver. Replays are useful. A consumer that does not declare an idempotency key and dedup window is a consumer that will eventually charge twice, embed twice, write the same row twice, or page someone twice.
3. **The broker is not the durability story.** A broker can deliver an event. It cannot tell you whether the LLM call inside the consumer happened, completed, or produced the right side effect. Durability is the consumer's responsibility — an outbox or status table next to the side-effect store, not the broker's redelivery semantics.
4. **Schema discipline is the integration contract.** Event-driven AI systems accumulate consumers. The event schema is the API. Versioning, additive-only changes, and the "what's in this event and what's a pointer to it" decision determine whether the system stays maintainable past three consumers or collapses into a tangle of brittle producer-consumer pairs.

Structure: (2) the three AI roles on an event bus; (3) the four event-driven AI patterns; (4) ordering, idempotency, and exactly-once semantics; (5) retry, DLQ, and poison-pill handling; (6) schema discipline at event boundaries; (7) how backpressure propagates; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three AI roles on an event bus

An AI feature on an event bus is playing one of three roles. The role determines what the integration must guarantee.

### 2.1 AI-as-consumer

The AI feature subscribes to upstream events. Each event triggers an LLM call (classify, summarize, embed, extract). The AI feature's output is either stored, published as a new event, or both.

**Examples.**

- Document uploaded → classify document type → write classification to metadata.
- EHR clinical note saved → extract structured findings → write to clinical-data warehouse.
- Customer support ticket created → summarize, route, suggest reply → update ticket.

**What the integration must guarantee.**

- **Per-event idempotency.** The consumer will receive the same event more than once (broker redelivery, replay, retry). The side effect must be safe under at-least-once delivery.
- **Backpressure to the broker.** When the LLM provider is slow or rate-limited, the consumer cannot grow its in-flight queue without bound. The broker's flow-control or the consumer's visibility-timeout discipline must absorb the slowness.
- **Poison-pill quarantine.** Some events will trigger LLM failures (oversized input, content-policy refusal, hallucinated tool call). After N retries, the event goes to a DLQ; the consumer does not loop on it forever.

**Where it shows up.** Ingestion pipelines. Document-classification pipelines. Compliance scanners. Any "every X gets processed by an AI step" workflow.

### 2.2 AI-as-producer

The AI feature *emits* events that downstream consumers process. The AI's completion (or its intermediate state) is the event payload.

**Examples.**

- Agent finished task → emit `agent.task.completed` event → downstream EHR, audit, analytics, and customer-notification consumers each subscribe.
- LLM scored a document as high-risk → emit `document.risk-flagged` event → compliance review queue subscribes.
- Embedding job completed → emit `document.embedded` event → search-index updater subscribes.

**What the integration must guarantee.**

- **Schema stability.** Consumers depend on the event schema. Breaking changes break consumers; the producer must version additively or coordinate with consumers explicitly.
- **At-least-once delivery semantics.** Producers cannot guarantee exactly-once to consumers; consumers must be idempotent. The producer's job is to ensure the event is published at least once after the side effect commits (outbox pattern, transactional publish).
- **Causal ordering, where it matters.** "Agent started" before "agent completed" before "agent's output indexed." Per-aggregate ordering (per-patient, per-document, per-tenant) is usually achievable with partition keys; global ordering is rarely worth the cost.

**Where it shows up.** Agent-completion events. Long-running classification or embedding jobs. AI-as-source-of-truth scenarios (rare; usually AI is upstream-of or downstream-of source-of-truth, not the source itself).

### 2.3 AI-as-step-in-saga

The AI feature is one step in a multi-step business process. Each step is its own service; the LLM step participates as a peer. The saga (orchestrated or choreographed) coordinates step ordering, retries, and compensation.

**Examples.**

- Insurance claim received → validate → AI-extract codes → human review → AI-summarize for adjudicator → adjudication step → notify customer. The AI steps are two of seven.
- Loan application received → KYC → AI-summarize history → underwriter decision → AI-generate disclosure → record.
- Patient referral received → eligibility check → AI-extract referral context → care-coordinator agent runs → human approve → EHR write → notify referring provider.

**What the integration must guarantee.**

- **Compensation semantics for AI steps.** If a later step fails, the AI step's side effects must be reversible (or, more honestly, the AI step must avoid irreversible side effects until the saga commits). "Sent the email" is not reversible; "drafted the email and stored it pending send" is.
- **Cost-aware retry.** A failing downstream step may trigger saga retry. If the AI step re-runs on every saga retry, costs balloon. Caching the AI step's output keyed on input hash is the standard mitigation.
- **Observability across steps.** Trace ID propagation through saga steps lets you debug "the agent did the right thing but step 5 lost the result" vs "the agent's output was wrong from the start."

**Where it shows up.** Regulated industries (insurance, banking, healthcare). Anywhere AI is a participant in a workflow that humans and other services also participate in. Less common than the other two roles but the highest-value when correctly designed.

### 2.4 Choosing the role

The role isn't optional — every event-driven AI feature is one of these three. But the choice has real consequences:

| Role | Latency budget | Idempotency burden | Schema burden | Best fit |
| --- | --- | --- | --- | --- |
| AI-as-consumer | Seconds-to-minutes | Consumer-side | Consumes existing event schema | Ingestion pipelines |
| AI-as-producer | Sub-second to seconds | Downstream consumers | Producer defines schema; must version carefully | Completion fan-out |
| AI-as-step-in-saga | Per-step; saga-bounded | Saga-aware compensation | Step contract per step | Regulated workflows |

A common architectural mistake is conflating roles — building "AI-as-consumer" infrastructure and then asking it to play AI-as-producer for downstream fan-out. The two roles need different operational tooling; one consumer + one DLQ is not the same as one producer + N consumer subscriptions.

---

## 3. The four event-driven AI patterns

Four patterns cover ~90% of event-driven AI integration in production systems.

### 3.1 Ingestion-triggered AI

**Shape.** Upstream system writes an artifact (document, record, image) to durable storage. A storage event fires. The AI consumer subscribes, processes the artifact, writes results back (to the artifact's metadata, to a sibling table, or to an analytics store).

**Common AWS implementation.** S3 PUT → S3 event notification → SNS → SQS → consumer Lambda/Fargate task → LLM call → write results.

**Common Kafka implementation.** Application writes to Kafka topic on artifact create → consumer group reads, calls LLM, writes results to sink topic or DB.

**What goes right.** Throughput scales horizontally; consumer count tuned to model provider's rate limit. Failures land in DLQ for inspection. Reprocessing is replay-based: bump consumer-group offset, replay through the AI step.

**What goes wrong.**

- **Visibility-timeout shorter than LLM latency.** If SQS visibility timeout is 30s but the LLM call routinely takes 45s, every message gets redelivered while the original handler is still working — duplicate work and duplicate spend.
- **Storage event is not the right granularity.** S3 PUT fires for every part of a multipart upload. The consumer sees N events for one file. Filter on `s3:ObjectCreated:Put` and `:CompleteMultipartUpload`, not the parts.
- **The LLM step is the bottleneck but the consumer count is what gets tuned.** Adding consumers without tuning provider concurrency produces 429 storms.

**Decision criteria.**

- Choose ingestion-triggered AI when: artifact arrival is the natural trigger; processing can be deferred (no human waits); throughput must scale; replay is valuable.
- Choose request/response instead when: the user uploading the document expects to see the result on the next screen.

### 3.2 Multi-stage AI pipeline via events

**Shape.** Several AI steps chained, each step's output becoming the next step's input via an event. Document → classify → if-classified-as-X-then-extract → if-extracted-then-embed → write.

**Why events between stages instead of one fat consumer.**

- Each stage scales independently. Embedding scales differently than extraction.
- Each stage's failure mode is isolated; a flaky embedding step doesn't take down classification.
- Each stage is replayable separately. "Re-embed everything classified-as-X" is a topic-replay.
- New stages can subscribe to upstream events without modifying the upstream pipeline.

**What goes right.** The pipeline becomes a DAG of consumers; each consumer is small, focused, and independently deployable. Re-running a single stage for backfill or improvement is a topic-replay rather than a code change.

**What goes wrong.**

- **Stage-N consumer sees stage-(N-1) before stage-(N-1) is durable.** If extraction emits its event before its DB write commits, downstream sees an extraction event for content the next consumer can't find. Use the outbox pattern.
- **Schema drift between stages.** Stage 2 starts emitting a new field; stage 3 doesn't know about it. Schema registry + consumer-driven contract tests catch this.
- **Implicit dependencies between stages that look independent.** Stage 3 quietly relies on stage 2's output having a specific shape. When stage 2 changes, stage 3 breaks. Make the dependency explicit; document the consumer's input contract.

**Decision criteria.**

- Choose multi-stage events when: the stages have different scaling characteristics, the stages benefit from independent replay, or new stages will be added over time.
- Choose monolithic consumer instead when: the stages are tightly coupled, never replayed independently, and the operational overhead of N consumers isn't worth the flexibility.

### 3.3 AI-completion fan-out

**Shape.** An AI feature (agent, batch job, or scheduled run) completes work. The completion is an event. Multiple downstream consumers subscribe.

**Common consumers.** Audit log writer, customer notification service, analytics warehouse, search-index updater, downstream business workflow.

**Why fan-out vs. direct calls.** If the AI feature directly called five downstream services, each new consumer requires modifying the AI feature. Fan-out via events makes downstream consumers additive — the AI feature doesn't know who's listening.

**What goes right.** Adding a new consumer is a subscription, not a code change in the producer. Failures in one consumer don't block others. Replays let new consumers backfill from history.

**What goes wrong.**

- **The event payload is too thin.** Consumers can't act on `{taskId: "abc"}` alone; they must re-fetch the full context. Re-fetch storms hit the source. Either fatten the event (denormalize) or accept the fetch cost — pick deliberately.
- **The event payload is too fat.** Embedding the entire LLM output (kilobytes of structured response) in the event makes the event hard to inspect, hard to evolve, and breaks broker size limits. Store the heavy payload separately; the event carries the pointer.
- **No consumer ownership.** Multiple consumers subscribe; nobody owns the event schema. When the producer changes it, three consumers break in mysterious ways.

**Decision criteria.**

- Choose AI-completion fan-out when: more than one downstream consumer needs to act on completion, or when downstream consumers will be added over time.
- Choose direct call instead when: there is exactly one downstream consumer, that consumer is high-trust, and the latency of an extra hop matters.

### 3.4 AI-driven business event recognition

**Shape.** A stream of business events flows through; the AI consumer inspects each event (or a window of them) and emits *recognized* events when it detects a pattern of interest.

**Examples.**

- Transaction stream → AI consumer detects suspicious patterns → emits `transaction.flagged-as-suspicious`.
- Support-ticket stream → AI consumer detects escalation patterns → emits `ticket.escalation-recommended`.
- Clinical-notes stream → AI consumer detects deterioration patterns → emits `patient.attention-needed`.

**Why event-driven for this.** The recognition is best done on a stream because patterns span events; the recognition is best done as a producer because the recognized event triggers downstream workflow.

**What goes right.** Real-time pattern detection without polling. Recognized events fold cleanly into existing alerting / queueing / workflow infrastructure.

**What goes wrong.**

- **Recognition latency vs. accuracy.** A model that sees the next 100 events is more accurate than one that sees the next 10. The buffer adds latency. Decide the trade-off explicitly.
- **False positives flood downstream.** A 5% false-positive rate on a 1000-event-per-day stream is 50 false alarms per day. Tune the recognition threshold to downstream tolerance.
- **The model is stateful but the consumer is restartable.** When the consumer restarts, does the model's context window reset? Usually yes — accept that recognition quality degrades briefly at restart, or maintain stateful context in a sidecar store.

**Decision criteria.**

- Choose AI-driven recognition when: the input is a continuous stream, the pattern of interest spans events, and downstream workflow benefits from a clean trigger.
- Choose batch scoring instead when: the recognition can wait, the input is naturally a batch, and the cost of streaming infrastructure isn't justified.

---

## 4. Ordering, idempotency, and exactly-once semantics

The single biggest gap between conventional event-driven thinking and event-driven AI thinking is what these properties mean when the work being done is an LLM call.

### 4.1 Ordering

**What brokers offer.** Per-partition ordering (Kafka), best-effort with FIFO option (SQS), no ordering guarantees by default (SNS).

**What AI consumers need.** Almost always *per-aggregate* ordering — per patient, per document, per tenant. Rarely *global* ordering, which is expensive and usually unnecessary.

**Pattern.** Partition by aggregate key. The aggregate key is whatever entity the AI step's output keys against — patient ID, document ID, tenant ID.

**Pitfall.** Partitioning by hash of payload (e.g., document content) produces no useful ordering and undermines retry semantics (a retry of the same payload lands in the same partition — usually fine, but only by accident). Partition explicitly by the aggregate key your consumer keys against.

**Pitfall.** Cross-partition ordering assumptions. The consumer must not assume that event from partition A arrives before event from partition B. If it does, you don't have a partitioning scheme; you have a single-partition queue with a slow consumer.

### 4.2 Idempotency

**What "idempotent" means for an AI consumer.** The *side effect* is at-most-once given the same input event. The *LLM response* may differ between attempts — temperature, model drift, retry on different model — but the side effect must not duplicate.

**The two idempotency patterns.**

**Pattern A: Idempotency key in the event; side-effect store enforces uniqueness.** The event carries a stable ID (the originating event's ID, or a hash of its content). The consumer's side-effect write uses that ID as a unique key. Duplicate deliveries fail the unique constraint and are no-ops.

**Pattern B: Processed-event log; consumer checks before processing.** The consumer maintains a "processed events" table keyed by event ID. On receipt, check the log; if present, ack and return. If absent, process, write the side effect, then write the log entry.

**Pattern A is preferred when** the side effect already has a natural unique key (database row, idempotent API). **Pattern B is preferred when** the side effect is a one-shot operation that's hard to make idempotent (sending an email, calling an external API that's not itself idempotent).

**The trap with Pattern B.** If the consumer writes the side effect, then crashes before writing the log entry, the next delivery duplicates the side effect. Either write the log entry transactionally with the side effect, or use an external idempotency layer (the API's own idempotency-key header).

**The trap with Pattern A.** The "natural unique key" is sometimes derived from the LLM's output. If the LLM produces different output on retry, the unique key changes, and the duplicate isn't detected. The unique key must be derived from the *input* event, not the LLM's output.

### 4.3 Exactly-once semantics

**Verdict.** Don't believe vendors who tell you they offer exactly-once. They offer at-least-once delivery + at-most-once side effects given consumer-side idempotency. The "exactly-once" branding is a marketing simplification of that engineering combination.

**Practical implication.** Design every AI consumer for at-least-once delivery. Make every side effect idempotent. Test the idempotency by replaying events and asserting the side-effect store is unchanged.

**Where exactly-once is genuinely available.** Within a single broker's transactional API (Kafka transactional producer + EOS consumer-side semantics) the *broker's own state* can be exactly-once. The *side effect that happens in the consumer* is still your problem.

### 4.4 The "expensive retry" trap

**Scenario.** Consumer processes an event. LLM call costs $0.40. Consumer writes the result. Consumer crashes before acking the broker. Broker redelivers. Consumer processes the event again. Second LLM call costs $0.40. Idempotency on the side-effect store catches the duplicate write — but the $0.40 is already spent.

**Mitigations.**

- **Cache the LLM call.** Use the event ID (or a hash of the input) as a cache key. On retry, the cache hit avoids the LLM call. Costs $0 on retry.
- **Defer the LLM call until after a "claim" is committed.** Write a row to a claim table keyed by event ID before calling the LLM. On retry, check the claim table; if a claim exists and is recent, the original processor is still working — back off rather than re-call.
- **Ack the broker before writing the result, with the result going through an outbox.** Riskier; if the consumer crashes between ack and write, the side effect is lost. Only use when loss is acceptable.

The right mitigation depends on the cost per LLM call and the acceptable rate of duplicate work. For $0.001-per-call workloads, ignore the problem. For $0.40-per-call workloads, the LLM-call cache or claim-table pattern is non-optional.

---

## 5. Retry, DLQ, and poison-pill handling

Conventional event-driven systems retry with exponential backoff, dead-letter after N attempts, and surface DLQ depth as an alerting metric. AI consumers need a more nuanced playbook.

### 5.1 The four classes of consumer failure

**Transient infrastructure failure.** Broker connection lost, consumer pod restarted, network blip. Retry will succeed; the LLM call was never made.

→ **Default retry.** Exponential backoff. The broker's redelivery mechanism handles this naturally.

**Transient model-provider failure.** 429 rate limit, 500 server error, network timeout to provider. Retry will likely succeed; the LLM call may or may not have completed (you might be retrying a call that already incurred cost).

→ **Retry with provider-aware backoff.** Read the provider's `Retry-After` header; don't retry faster than that. Track in-flight calls so you don't double-spend. Surface 429s as a backpressure signal (see §7).

**Persistent input-quality failure.** The input is malformed, oversized, in the wrong language, or otherwise something the LLM cannot meaningfully process. Retry will fail forever.

→ **Quick-quarantine to DLQ.** Don't burn 10 retries; one or two is enough. Tag the DLQ entry with the failure class so the operator can route to a fix workflow.

**Persistent model-output failure.** The LLM responds, but the response violates the consumer's expectations (schema fails to parse, content-policy refusal, hallucinated tool call, missing required field). Retry might succeed (non-determinism) but might also burn through the budget.

→ **Bounded retry with output-class quarantine.** Allow 2-3 retries (LLM non-determinism may resolve it). If still failing, quarantine to a separate DLQ tagged "output-quality" so the operator can distinguish from input failures.

### 5.2 DLQ topology for AI consumers

Single DLQ for everything is the easy default and the wrong long-term answer. AI consumers benefit from at least three DLQs:

- **input-quality-dlq:** the input was the problem.
- **model-output-dlq:** the model's output was the problem.
- **infrastructure-dlq:** transient failure exceeded retry budget (rare with proper retry; worth investigating).

Each DLQ has different operator action:
- input-quality → fix upstream producer or filter the bad input.
- model-output → tune the prompt or the schema; retry from DLQ after fix.
- infrastructure → investigate the platform issue; replay from DLQ once fixed.

A single DLQ collapses these classes; operators end up writing ad-hoc classification scripts to figure out what to do with the queue.

### 5.3 Poison-pill detection

A poison pill is an event that *looks* processable but reliably causes failure. The consumer accepts it, fails, retries, fails again, and either floods the DLQ or (worse) loops indefinitely if retry limits aren't enforced.

**Detection.**

- Track per-event retry count. Any event with retry count exceeding the threshold is a poison-pill candidate.
- Track failure-class distribution per consumer. A spike in one failure class often correlates with a specific input pattern.
- Maintain a known-poison-pill list (event ID hashes) that the consumer skips on receipt. Auto-populate from quarantine events that have been investigated and marked "skip permanently."

**Mitigation.**

- **Always set a retry ceiling.** Brokers default to "infinite redelivery" in some configurations; this is a footgun. Set max-receive-count or equivalent.
- **Fail fast on known bad input.** Pre-flight input validation (size, format, encoding) before the LLM call. A 2ms validation check that catches 5% of bad input saves the LLM call entirely.
- **Circuit-break on failure-rate spike.** If the consumer's failure rate exceeds X% over a window, stop polling, page the operator. Better to halt processing than to silently DLQ everything.

### 5.4 The "retry storms cost real money" reality

For LLM consumers, an aggressive retry policy can produce shocking bills.

**Worked example.** Pipeline processes 100k documents per day. 5% transient failure rate (provider 429s and 500s). Retry policy: 5 retries with exponential backoff.

- 100k × 5% = 5k failures.
- 5k × 5 = 25k retry attempts.
- At $0.04 per call (input + output), retries alone cost $1k per day.

Adding the LLM-call cache (§4.4) drops the retry cost to near-zero because retries hit the cache. The architecture choice "we have a retry policy" is cheap; the architecture choice "we have a retry policy *and* we cache the LLM call so retries don't double-spend" is the difference between $1k/day and $0/day in retry cost.

---

## 6. Schema discipline at event boundaries

Event-driven AI systems accumulate consumers over time. The event schema is the integration contract. If it drifts unpredictably, downstream consumers break.

### 6.1 Versioning posture

**Additive-only by default.** New fields are fine; renaming or removing fields breaks consumers. If a field must change semantics, add a new field with the new semantics; deprecate the old field with a stated removal date; communicate to consumers.

**Schema registry where multiple consumers exist.** Confluent Schema Registry, AWS Glue Schema Registry, or in-house equivalent. The registry rejects breaking changes at publish time. Schema drift is caught in CI, not in a downstream 3 AM page.

**Consumer-driven contract tests.** Each consumer asserts the shape it depends on (Pact, Spring Cloud Contract, or hand-rolled equivalent). Producer's CI runs the consumer contracts; a producer change that breaks a consumer fails the producer's build.

### 6.2 "Fat event" vs "thin event with pointer"

Two patterns for what goes in the event payload:

**Fat event.** The event carries the entire payload — the document text, the LLM output, the structured fields. Consumers act on the event without re-fetching.

- **Pro.** Self-contained; replayable in isolation; no fetch storms.
- **Con.** Large events stress broker (size limits), are hard to inspect, embed potentially-sensitive data in broker storage, and produce coupling between producer and broker storage costs.

**Thin event with pointer.** The event carries an ID; consumers fetch the full payload from a content store keyed by that ID.

- **Pro.** Small, easy to inspect; sensitive data stays in the content store with its own access controls; broker storage stays cheap.
- **Con.** Replay requires the content store still has the data; fetch storms hit the content store on fan-out; cold-replay (after retention expires from the content store) is impossible.

**Hybrid.** Fat event with key fields (the LLM output's structured summary, the classification result, the embedding's metadata) plus pointer for heavy payloads (raw document, full embedding vector).

**Decision criteria.**

- Choose fat when: replay value > storage cost; consumers are external (don't get content-store access).
- Choose thin when: payload is large or sensitive; content store has good lifecycle management; fan-out count is small.
- Choose hybrid when: most consumers need the summary; only some need the raw — most AI systems land here.

### 6.3 Where embeddings and large LLM outputs go

LLM outputs frequently include embedding vectors (4-12 KB) or large structured outputs (10-100 KB). Embedding these in events stresses broker limits.

**Pattern.** The event carries the structured summary (classification, key fields, confidence). The large outputs go to durable storage (S3, blob storage, vector store) and the event carries pointers. Consumers that need the heavy payload fetch on demand.

**Pitfall.** The pointer's expiration. Vector stores often have retention tied to a key TTL; large outputs in blob storage need lifecycle policies. An event that points to expired content is useless. Either align retention (event TTL ≤ content TTL) or accept that old events become un-replayable past a point.

### 6.4 PII and the event boundary

Events that flow through brokers often have broader access than the original artifact. PII in event payloads is PII in broker storage; sensitivity classifications must propagate.

**Pattern.** Filter PII out of events at the producer. If consumers need it, they re-fetch from an access-controlled store with the broker event as the pointer.

**Pattern.** Tag events with sensitivity classification; consumers' access to high-sensitivity topics is restricted at the broker level (Kafka ACLs, SNS topic policies, EventBridge bus permissions).

**Pitfall.** "We'll add PII filtering later." Once PII is in broker storage, you have a retention problem (broker may retain events for days), an audit problem (who saw them), and an exfiltration problem (anyone with broker access). Filter at the producer from day one.

---

## 7. How backpressure propagates in event-driven AI systems

Event-driven systems propagate backpressure through different mechanisms than synchronous systems. AI workloads make this more important than usual because the cost of getting it wrong is monetary as well as operational.

### 7.1 Where the bottleneck actually is

For an AI consumer, the bottleneck is almost never the broker. It's:

- The model provider's TPM or RPM limit.
- The consumer's GPU pool (for self-hosted models).
- A downstream side-effect store (vector index, DB) that can't keep up with the LLM's output rate.

The broker can scale; the LLM can't. Tuning consumer concurrency without accounting for the LLM's limit produces upstream-originated 429 storms — the consumer pulls events faster than the LLM can process them.

### 7.2 The TPM-aware consumer

The provider quotes a TPM (tokens per minute) and RPM (requests per minute). The consumer's effective throughput is:

```
effective_throughput = min(
  RPM / requests_per_event,
  TPM / tokens_per_event
)
```

Either constraint can bind. A high-token, low-RPM workload is TPM-bound; a low-token, high-RPM workload is RPM-bound. The right consumer concurrency tracks whichever is binding.

**Pattern.** The consumer maintains a token-bucket or leaky-bucket rate-limiter aware of the provider's TPM/RPM. When the bucket is empty, the consumer stops polling the broker (or sets a longer visibility-extension). The broker's backlog absorbs the burst.

**Pattern.** The consumer subscribes to provider rate-limit headers. When a 429 returns with `Retry-After: 30`, the consumer pauses polling for 30 seconds (not just the failed request). The fleet coordinates via a shared rate-limit signal (Redis-backed bucket).

### 7.3 The broker as buffer

When the consumer slows (because the LLM is slow), the broker's depth grows. This is a feature, not a bug — the broker is acting as the burst buffer the architecture intended.

But depth has a limit. Broker storage isn't free, and some brokers (SQS, EventBridge) impose hard limits or cost penalties at deep queue depths.

**Pattern.** Alert on queue depth growth rate, not absolute depth. A queue growing 1k events per minute will hit problems; a queue at 10k that's been stable for hours is fine.

**Pattern.** Shed-load when the depth grows beyond a threshold. The producer either stops emitting (best case), routes to a lower-priority topic (degraded but processing), or drops low-priority events (acceptable for some workloads, never for others).

### 7.4 Downstream-induced backpressure

When the AI's output rate exceeds the downstream consumer's ingestion rate (e.g., vector index can ingest 1k vectors/sec, AI emits 10k/sec), backpressure must propagate upstream.

**Pattern.** The downstream consumer's queue depth signals backpressure to the AI consumer. When depth grows past threshold, the AI consumer slows.

**Pattern.** The AI consumer writes to a buffer (intermediate queue or staging table) rather than directly to the downstream consumer. The downstream consumer drains at its own pace. This decouples but adds latency.

### 7.5 The end-to-end view

Backpressure in event-driven AI systems is a chain: producer → broker → AI consumer → downstream consumer → final sink. A slowdown at any link must propagate backward, or the link before it grows unboundedly.

The architecture's job is to make sure each link has a flow-control mechanism: producer can be told to slow, broker has finite capacity, AI consumer respects provider rate limits, downstream consumer signals when overloaded.

Detail on the queue topology and fairness mechanics is in [backpressure-and-queueing.md](./backpressure-and-queueing.md). This section is the event-driven view: what propagates, where, and through which mechanism.

---

## 8. Worked Meridian example

Meridian Health runs two event-driven AI pipelines in production. Both illustrate the patterns above; one shows what went right, one shows what went wrong before it was redesigned.

### 8.1 The clinical-document ingestion pipeline (went right)

**Shape.** Clinical documents (referral letters, discharge summaries, lab reports — ~40k per day) land in S3 from the EHR's nightly export and from real-time clinician uploads. Each document needs:

1. Document-type classification (referral / discharge / lab / consult / other).
2. PHI extraction and tagging (which patient, which encounter).
3. Embedding for retrieval.
4. Indexing into the care-coordinator agent's RAG store.

**Architecture.**

```
S3 PUT → EventBridge → SQS (per-stage)
                           ↓
                    [classify-consumer]
                           ↓
              EventBridge ("document.classified")
                           ↓
       ┌────────────┬─────────────┐
       ↓            ↓             ↓
  [extract-PHI]  [embed]   [route-to-warehouse]
       ↓            ↓
   EventBridge ("document.processed")
       ↓
  [index-into-rag]
```

**Patterns applied.**

- **Multi-stage AI pipeline via events (§3.2).** Each stage is its own consumer. Each has its own DLQ, rate-limit, retry policy, and deployment cadence.
- **AI-as-consumer (§2.1) at each stage.** None of these stages have a human waiting; they all process in seconds-to-minutes.
- **Hybrid event payload (§6.2).** The event carries the classification, the patient ID, and pointers to the document and embedding. The full document text and embedding vector live in their own stores.
- **TPM-aware consumer (§7.2).** Each consumer respects the model provider's TPM. The classify-consumer is RPM-bound (small inputs, lots of them); the embed-consumer is TPM-bound (medium inputs, larger contexts). Consumer concurrency tuned independently.
- **Pattern A idempotency (§4.2).** Each consumer's side-effect store uses the document ID as the unique key. Duplicate deliveries fail the unique constraint and are no-ops.
- **LLM-call cache (§4.4).** Each consumer caches the LLM call keyed on document ID + content hash. Retries hit the cache.
- **Three DLQs (§5.2):** input-quality, model-output, infrastructure. The pipeline ops dashboard shows depth per DLQ; operators have separate runbooks per DLQ class.

**Results (Q1 2026).**

- Throughput: sustained 40k documents/day; peak 110k/day; pipeline absorbs without intervention.
- LLM cost: $0.025 per document average (classification + extraction + embedding combined).
- Retry cost: <$5/day (LLM-call cache catches retries).
- DLQ depth: input-quality DLQ runs ~50 events/day (mostly malformed PDFs from one upstream system); model-output DLQ runs <5/day; infrastructure DLQ runs 0-3/day.
- MTTR for pipeline issues: 18 minutes (median); the DLQ classification accelerates triage.

### 8.2 The Care Coordinator completion-event pipeline (went wrong, then right)

**Original shape (failed).** The Care Coordinator agent completes a task (e.g., "draft a referral letter for patient X"). The agent directly calls four downstream services: write to EHR-staging, send notification to clinician portal, log to compliance audit, update analytics warehouse.

**What went wrong.**

- **Adding a fifth consumer (cost analytics) required modifying the agent's runner.** Slow PR cycle, agent deploy = blocking dependency.
- **A flaky compliance-audit service caused the agent's task to appear failed.** The agent's "completed" status hinged on all four calls succeeding; one slow service blocked the whole completion.
- **Replay was impossible.** When the analytics consumer needed to backfill, there was no historical record of completions to replay.
- **The agent retried the entire task when one downstream call failed.** $0.40 per agent task, with retry-the-whole-thing semantics, produced surprise bills during downstream incidents.

**Redesign.** Apply AI-completion fan-out (§3.3):

```
Agent completes task
       ↓
   write to agent's own outbox (transactional with task state)
       ↓
   outbox-publisher emits "agent.task.completed" to SNS
       ↓
   ┌──────────┬─────────────┬──────────┬──────────────┐
   ↓          ↓             ↓          ↓              ↓
[ehr-staging] [portal-notify] [audit-log] [analytics] [cost-attribution]
```

**Patterns applied.**

- **AI-as-producer (§2.2).** The agent's job is to complete the task and publish. Downstream consumers subscribe independently.
- **Outbox pattern (§4.1, §6.2).** The task's state and the "to be published" event are written transactionally to the agent's own DB. The outbox-publisher polls the outbox and emits to SNS.
- **At-least-once delivery + consumer-side idempotency (§4.3).** Each consumer's idempotency key is the agent's task ID.
- **Schema versioning (§6.1).** The event schema is registered. Adding the cost-attribution consumer was a subscription, not a producer change.

**Results.**

- Adding the cost-attribution consumer: 2 hours (write the consumer + subscribe). Previously: 1.5 days (PR to agent runner + deploy + verify).
- Agent task completion rate during downstream incidents: 99.8% (no longer hinges on downstream availability). Previously: 88% during the compliance-audit incident of Feb 2026.
- Cost during a downstream incident: unchanged from baseline. Previously: 3x during the Feb incident from retry-the-whole-task.
- Replay: 7 days of agent completions available for backfill in S3 (SNS → SQS → S3 archive). New consumers can backfill on first deploy.

**The lesson.** The agent's job is to complete the task; downstream consumers' job is to subscribe to completion. Coupling the two through direct calls made the agent's reliability the union of all downstream reliabilities. Decoupling through events made the agent's reliability its own.

---

## 9. Anti-patterns

These are the patterns that show up repeatedly when event-driven AI is implemented as "regular event-driven plus a model call." Each has a name; each has a corrective.

### 9.1 The infinite-redelivery loop

**Pattern.** Broker's max-receive-count not set. Consumer fails on a poison-pill event. Broker redelivers forever. Costs accumulate; DLQ never sees the event because there's no DLQ; operator notices when monthly bill arrives.

**Corrective.** Set max-receive-count on every queue. Configure DLQ for every consumer. Alert on DLQ depth.

### 9.2 The retry-the-whole-conversation pattern (agent-side)

**Pattern.** Agent runs multi-step task. Step 5 of 7 fails. Retry policy restarts the agent from step 1. Each retry re-runs steps 1-4 (expensive) before retrying step 5.

**Corrective.** Persist agent state per step. Resume from the failed step. Cross-link to [agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) in the engineering sibling.

### 9.3 The event payload that drags the LLM output

**Pattern.** Event payload includes the full LLM response (often 10-50 KB structured output). Broker storage costs balloon; events near or exceed broker size limits; debugging via broker UI becomes impossible because payloads don't render.

**Corrective.** Thin event with pointer (§6.2). The structured output goes to a payload store; the event carries the ID.

### 9.4 The retry storm during a provider degradation

**Pattern.** Provider has a degraded period (elevated 5xx rate). Consumer's retry policy fires aggressively. Each retry incurs cost (the call did reach the LLM, even if the response was bad). Provider's situation worsens because everyone's consumers are retrying simultaneously.

**Corrective.** LLM-call cache (§4.4). Provider-aware backoff (§5.1). Circuit-breaker on failure-rate spike (§5.3).

### 9.5 The "we'll add idempotency later" deferral

**Pattern.** First version of the consumer ships without idempotency. "We'll add it before production." Production happens. Idempotency keeps getting deprioritized. First incident: a broker redelivery during a normal operational event duplicates 5000 LLM calls.

**Corrective.** Idempotency on day one. The cost of adding it later (refactor consumer, backfill idempotency keys for existing data, prove correctness) is higher than the cost of designing it in from the start.

### 9.6 The DLQ that nobody reads

**Pattern.** DLQ exists, depth alerts fire occasionally, an operator silences the alert because the depth doesn't grow unboundedly. Events accumulate. Months later, someone discovers 50k DLQ events; nobody knows which are still relevant; nobody knows whether to replay or discard.

**Corrective.** Auto-classify DLQ entries by failure class. Operator runbook per class. Weekly DLQ review as a standing operational task. DLQ depth should trend to zero, not be acceptable as a non-zero baseline.

### 9.7 The single-topic event bus

**Pattern.** All AI events flow through one topic. Schema is whatever; producers and consumers are loosely coupled by naming convention. Adding a new event type means modifying the consumer to filter for it; adding a new consumer means subscribing to everything.

**Corrective.** Topic-per-event-type (or topic-per-domain at coarser granularity). Schema per topic. Consumers subscribe to specific topics. Cross-link to schema registry discipline (§6.1).

### 9.8 The synchronous wait for an async result

**Pattern.** The AI feature publishes an event; an upstream caller (HTTP request handler) blocks waiting for a result event. The blocking call has a 30-second timeout; the AI work routinely takes 45 seconds.

**Corrective.** Either make the AI work synchronous within the request budget (sync-vs-async-vs-streaming.md), or commit to async with a return path (callback-and-webhook-patterns.md). The hybrid "publish event but block synchronously" is the worst of both shapes.

---

## 10. Findings (sprint-assignable)

Eighteen findings spanning typical event-driven AI integration assessments. Each is a discrete change a team can pull into a sprint.

**ARCH-EDA-001 (P0). No max-receive-count set on AI consumer queues.** Infinite-redelivery risk. Set max-receive-count and DLQ on every queue serving an AI consumer. Owner: platform.

**ARCH-EDA-002 (P0). Consumer's visibility timeout is shorter than worst-case LLM latency.** Duplicate work, duplicate spend. Set visibility timeout to 2x P99 LLM latency; add visibility-extension heartbeat for long calls. Owner: consumer team.

**ARCH-EDA-003 (P0). No idempotency key on AI consumer side-effects.** First broker redelivery duplicates writes/charges. Identify the natural idempotency key (event ID); enforce uniqueness at the side-effect store; backfill existing data. Owner: consumer team.

**ARCH-EDA-004 (P1). No LLM-call cache on retries.** Retry storms cost real money. Implement input-keyed cache (hash of event input → LLM response) with reasonable TTL (24h for stable inputs). Owner: consumer team.

**ARCH-EDA-005 (P1). DLQ exists but is single bucket.** Operator can't distinguish input-quality from model-output from infrastructure failures. Split into three DLQs; runbook per DLQ. Owner: SRE + AI platform.

**ARCH-EDA-006 (P1). Producer doesn't use outbox pattern.** Side-effect-then-publish failures lose events. Implement transactional outbox; outbox-publisher reads outbox and publishes to broker. Owner: producer team.

**ARCH-EDA-007 (P1). Consumer concurrency not tuned to provider rate limits.** 429 storms during peak. Implement TPM/RPM-aware rate limiter; consumer pauses polling when bucket empty. Owner: consumer team.

**ARCH-EDA-008 (P1). Event schema not registered; no consumer contract tests.** Schema drift breaks downstream. Adopt schema registry; add consumer contract tests in producer CI. Owner: producer + consumer teams.

**ARCH-EDA-009 (P1). Event payload includes full LLM output (>10 KB).** Broker stress, expensive replay, PII concerns. Migrate to thin event + payload pointer; deprecate fat event after grace period. Owner: producer team.

**ARCH-EDA-010 (P2). No DLQ depth alerting.** DLQ silently grows; nobody notices. Alert on depth growth rate (events/minute > threshold) and absolute ceiling. Owner: SRE.

**ARCH-EDA-011 (P2). No partition key for per-aggregate ordering.** Out-of-order events for the same patient/document cause incorrect downstream state. Set partition key to aggregate ID (patient, document, tenant). Owner: producer team.

**ARCH-EDA-012 (P2). Synchronous wait on async event result.** Caller blocks on result event; timeouts and cancellation are messy. Migrate to true async pattern (callback or polling); see callback-and-webhook-patterns.md. Owner: feature team.

**ARCH-EDA-013 (P2). No circuit-breaker on consumer failure-rate spike.** Failure spikes silently DLQ everything. Implement failure-rate circuit-breaker; halt consumer and page operator when rate exceeds threshold. Owner: consumer team.

**ARCH-EDA-014 (P2). PII in event payloads with broad broker access.** Audit, residency, exfil risk. Filter PII at producer; consumers re-fetch from access-controlled store with broker event as pointer. Owner: producer + security.

**ARCH-EDA-015 (P2). Direct downstream calls instead of fan-out.** Adding consumers requires producer changes; producer's reliability inherits all downstream reliabilities. Migrate completion to event fan-out (§3.3); downstream consumers subscribe. Owner: producer team.

**ARCH-EDA-016 (P3). No event archive for replay.** New consumers can't backfill. Configure broker → S3 (or equivalent) archive; document replay procedure. Owner: platform.

**ARCH-EDA-017 (P3). No pre-flight input validation before LLM call.** Bad inputs incur LLM cost before being rejected. Add cheap pre-flight checks (size, format, encoding); reject before model call. Owner: consumer team.

**ARCH-EDA-018 (P3). No retry-cost reporting in cost dashboards.** Retry storms invisible to FinOps. Tag LLM calls as primary vs retry; surface retry-cost share on cost dashboards (see [cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) in the engineering sibling). Owner: FinOps + platform.

---

## 11. Adoption sequencing checklist

For a team adopting event-driven AI integration (greenfield or migration from synchronous), in order:

- [ ] **Decide the AI role on the bus (§2):** consumer, producer, or saga-step. Don't conflate.
- [ ] **Identify which of the four patterns (§3) applies.** If your shape doesn't match any of them, reconsider whether event-driven is the right shape.
- [ ] **Choose the broker explicitly.** Kafka if you need partition-keyed ordering and long retention. SQS+SNS if you need AWS-native fan-out without managing brokers. EventBridge if you need cross-account / cross-service routing. Pulsar if you need geo-replication. The broker's properties shape the consumer's required guarantees.
- [ ] **Define the event schema. Register it.** Versioning policy: additive-only. Consumer contract tests in producer CI.
- [ ] **Decide payload shape: fat, thin, hybrid (§6.2).** Document the decision and the rationale.
- [ ] **Set max-receive-count and DLQ on every queue.**
- [ ] **Define the three DLQs (§5.2):** input-quality, model-output, infrastructure. Operator runbook per DLQ.
- [ ] **Implement idempotency (§4.2). Test it by replaying events and asserting no duplicate side effects.**
- [ ] **Implement LLM-call cache (§4.4) keyed on input.**
- [ ] **Set visibility timeout to 2x P99 LLM latency. Configure heartbeat extension for long calls.**
- [ ] **Implement TPM/RPM-aware consumer rate limiter (§7.2).** Subscribe to provider's `Retry-After` header.
- [ ] **For producers, implement transactional outbox pattern. Outbox-publisher reads and publishes.**
- [ ] **Alert on DLQ depth growth rate. Alert on queue depth growth rate. Page on failure-rate spike.**
- [ ] **Tag LLM calls (primary vs retry) for cost dashboards.**
- [ ] **Filter PII at the producer. Tag events with sensitivity classification. Restrict topic ACLs accordingly.**
- [ ] **Configure broker archive for replay (S3 or equivalent). Document replay procedure.**
- [ ] **Run a chaos exercise: kill the consumer mid-LLM-call. Verify side-effect dedup. Verify cost dedup.**

---

## 12. References

**In this folder.**
- [sync-vs-async-vs-streaming.md](./sync-vs-async-vs-streaming.md) — the integration-shape decision; event-driven is one of several async shapes.
- [tool-call-architecture.md](./tool-call-architecture.md) — agent tools that themselves emit or consume events.
- [human-in-the-loop-boundaries.md](./human-in-the-loop-boundaries.md) — where humans participate in event-driven AI workflows.
- [backpressure-and-queueing.md](./backpressure-and-queueing.md) — queue topology, fairness, and rate-limit propagation (companion).
- [integration-failure-patterns.md](./integration-failure-patterns.md) — failure-mode taxonomy at AI integration boundaries (companion).
- [callback-and-webhook-patterns.md](./callback-and-webhook-patterns.md) — long-running AI work that returns via events or callbacks.

**Elsewhere in this repo.**
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — the worked example whose completion-event pipeline is described here.
- [reference-systems/analytics-warehouse-copilot.md](../reference-systems/analytics-warehouse-copilot.md) — ingestion patterns for the warehouse-side AI features.
- [multi-tenancy-and-isolation/noisy-neighbor-mitigation.md](../multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant fairness in event-driven AI consumers.
- [guardrails-and-policy-architecture/retrieval-scope-enforcement.md](../guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — scope controls when AI consumers fetch retrieval context.
- [data-architecture-for-ai/data-contracts-for-retrieval.md](../data-architecture-for-ai/data-contracts-for-retrieval.md) — schema-on-read patterns that compose with event schemas.

**Sibling repos.**
- [ai-engineering-reference-architecture / agent-engineering / error-and-partial-failure.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/error-and-partial-failure.md) — engineering perspective on agent-side failure and partial completion.
- [ai-engineering-reference-architecture / observability-and-telemetry / trace-and-span-design.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/observability-and-telemetry/trace-and-span-design.md) — tracing event-driven AI flows.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — attributing retry vs primary LLM cost in event-driven systems.
- [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — threat models for event payload exfiltration, broker access controls, and DLQ poisoning.

**External.**
- Confluent — *Designing Event-Driven Systems* (Stopford, 2018) — foundational; the AI-specific overlays in this doc sit on top of this material.
- AWS — SNS/SQS/EventBridge service docs for the AWS-native patterns referenced.
- Kafka documentation — partitioning, transactional producer, EOS consumer semantics.
- Temporal / Inngest / Step Functions — durable workflow frameworks that often own the saga (§2.3) shape.
