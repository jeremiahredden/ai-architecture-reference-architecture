# Chat History Architecture

> **Audience.** Architects whose chat feature is starting to feel "forgetful" — long conversations lose context. Tech leads whose conversations exceed 30 turns and the costs are climbing. Anyone whose "we keep verbatim chat history forever" architecture is hitting context-window walls. **Scope.** The *architectural* decisions about chat history: short-term vs long-term memory; verbatim history vs running summary vs tool-and-fact extraction vs hybrid; retention policy as an architectural choice; per-user vs per-session vs cross-session memory. Not the per-tenant memory isolation (see [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md §7](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)). Not the agent-memory engineering (see sibling [agent-engineering/memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Chat history is the part of context that grows. Every other allocation (system prompt, tools, retrieval) has stable size; history grows turn by turn. Without architecture, it grows until it fills the context window, costs explode, and quality drifts.

The pattern: a chat feature ships with "keep all history verbatim." Works at 1-5 turns; concerning at 10-20 turns; broken at 30+ turns:

- Cost per turn climbs.
- Latency rises (longer input = slower TTFT).
- Model behavior degrades ("lost in the middle"; conflicting older context).
- Eventually: hits context window; conversation ends abruptly.

The architectural question: what's the right memory model for this workload?

Options:

- **Verbatim.** Keep everything as said.
- **Running summary.** Continuously summarize older turns.
- **Tool / fact extraction.** Extract structured facts; discard verbiage.
- **Hybrid.** Recent verbatim; older summarized; structured facts persisted.

Each has trade-offs across cost, latency, quality, and use-case fit.

This document covers the architecture. It is opinionated about four things:

1. **Verbatim-forever is the wrong default at any non-trivial conversation length.** Plan for summarization or extraction from day one.
2. **The right memory model is per-workload, not per-platform.** Customer-support chat is different from clinical agent is different from coding copilot.
3. **Cross-session memory is a separate architectural decision.** "What we remember about a user across sessions" is different from "what we remember within a session."
4. **Memory has compliance implications.** PII in memory; retention; right-to-forget. Architect for these from the start.

Structure: (2) the memory models; (3) within-session memory architecture; (4) cross-session memory architecture; (5) summarization patterns; (6) fact extraction patterns; (7) retention and compliance; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The memory models

The architectural choices.

### 2.1 Verbatim history

Every turn kept exactly as said:

```
[Turn 1]
User: My patient has diabetes; what should I know about insulin?
Assistant: For type 2 diabetes management with insulin, consider...

[Turn 2]
User: What's the typical starting dose?
Assistant: Starting dose typically ranges from 0.1-0.2 units/kg...

[Turn 3]
User: What about for elderly patients?
Assistant: For elderly patients, considerations include...

... etc, all turns preserved ...
```

**Properties.**
- Faithful; no information loss.
- Grows linearly with conversation length.
- Eventually hits context window.

**When right.** Short conversations (< 10 turns); high-fidelity domains (legal, medical).

### 2.2 Running summary

Continuous summarization of older turns:

```
[Conversation summary so far]:
The conversation has covered insulin management for type 2 diabetes,
including starting doses (0.1-0.2 units/kg), elderly considerations,
and adjustments for kidney function.

[Recent turn]
User: What about diet considerations?
Assistant: ...
```

**Properties.**
- Bounded size (summary).
- Loses verbatim detail.
- Recency bias possible.

**When right.** Long conversations; cost-sensitive; detail not critical.

### 2.3 Tool / fact extraction

Extract structured facts; discard the verbiage:

```
[Conversation facts]:
- Patient context: T2DM
- Insulin discussed: type, dose, elderly, diet, kidney
- User profile: clinician
- Last interaction: 2026-05-27
```

**Properties.**
- Smallest size.
- Loses voice / tone.
- Structured; queryable.

**When right.** Workflow systems; structured tasks; analytics.

### 2.4 Hybrid

Combination:

- Recent N turns: verbatim.
- Older: summarized.
- Persistent facts: structured.

```
[Persistent facts about user]:
- Role: physician, internal medicine
- Specialty: diabetes management

[Conversation summary (turns 1-25)]:
This session has covered insulin management for type 2 diabetes...

[Recent turns (turns 26-30 verbatim)]:
[Turn 26]
User: ...
Assistant: ...
[Turn 27]
...
```

**Properties.**
- Recent detail preserved.
- Older content compressed.
- Long-term facts queryable.

**When right.** Most production chat; the typical right answer.

### 2.5 The trade-off matrix

```
Model              Size growth  Cost      Quality   When right
─────────────────────────────────────────────────────────────────
Verbatim           Linear        Linear    Highest   Short conversations
Running summary    Bounded       Bounded   Medium    Long; cost-sensitive
Fact extraction    Smallest      Lowest    Lowest    Structured workflows
Hybrid             Sub-linear    Sub-linear High     Default for most chat
```

### 2.6 The recent-verbatim window

For hybrid: how many turns kept verbatim?

- 3-5 turns: Patient API chat (UX-focused; recent context dominant).
- 5-10 turns: Care Coordinator (clinical detail mattering).
- 10+ turns: code/contract review (longer context needed).

Per-workload tuning.

### 2.7 The summarization frequency

For running summary:

- Re-summarize every N turns (e.g., every 5).
- Per-N-turns the summarizer runs.
- New summary replaces old.

Compute cost: small.

### 2.8 The fact-extraction granularity

For fact extraction:

- Per-turn extraction: extract facts after each turn.
- Per-session extraction: extract facts at session end.

Per-turn enables real-time fact use; per-session is cheaper.

### 2.9 The "we use all four in different parts" combination

For mature platforms:

- Within-session: hybrid (recent verbatim + summary).
- Cross-session: fact extraction (persistent facts about user).

Different memory models for different timescales.

---

## 3. Within-session memory architecture

How the system remembers during a conversation.

### 3.1 The session model

A session = a conversation:

- Starts when the user starts.
- Ends when the user stops (timeout, explicit end, app close).
- Has a session_id.

Within a session: conversation state grows.

### 3.2 The history storage

Per session, history is stored:

```
Session storage:
  session_id: ...
  user_id: ...
  tenant_id: ...
  turns: [
    {turn_n: 1, user: "...", assistant: "...", timestamp: ...},
    {turn_n: 2, user: "...", assistant: "...", timestamp: ...},
    ...
  ]
  summary_state: {...}  # for summarization
  fact_state: {...}     # for fact extraction
```

DB-backed; sometimes Redis for active sessions.

### 3.3 The session-fetch pattern

At each new turn:

1. Receive new user message.
2. Fetch session state.
3. Assemble prompt with history.
4. Call model.
5. Update session state with new turn.

Session state grows per turn.

### 3.4 The history-formatting

When fetching history for the prompt:

```python
def format_history_for_prompt(session):
    if session.turns_count <= 5:
        # Short conversation; verbatim
        return format_verbatim(session.turns)
    else:
        # Hybrid
        recent = session.turns[-5:]
        older = session.turns[:-5]
        summary = session.summary_state.summary
        return f"[Conversation summary]\n{summary}\n\n[Recent turns]\n{format_verbatim(recent)}"
```

Per-workload logic; consistent.

### 3.5 The summarization trigger

When to summarize:

- Every N turns.
- When token-budget is approached.
- On session end (for archival).

Per-workload trigger.

### 3.6 The summarization call

Summarization is itself a model call:

```python
def summarize(session):
    older_turns = session.turns[:-5]
    summary_so_far = session.summary_state.summary or ""
    
    new_summary = call_model(
        prompt=f"Summarize this conversation segment, focusing on facts and decisions.\nPrevious summary: {summary_so_far}\nNew turns to incorporate:\n{format(older_turns)}",
        model="sonnet"  # or cheaper model for summarization
    )
    
    session.summary_state.summary = new_summary
    session.turns = session.turns[-5:]  # only keep recent in detail
    session.save()
```

A summary-update call per N turns; small cost.

### 3.7 The "summary lost something important" risk

Summarization is lossy:

- Important detail may be dropped.
- Tone / voice lost.

Mitigation:

- Recent turns verbatim (recent detail preserved).
- Fact extraction for must-not-lose items.

### 3.8 The session-end archival

When a session ends:

- Persist final state.
- Optionally extract long-term facts (cross-link to §4).
- Move to cold storage if needed.

Session storage lifecycle defined.

---

## 4. Cross-session memory architecture

What we remember across sessions.

### 4.1 The cross-session use cases

- Customer support: remember the customer's previous issues.
- Personal assistant: remember user's preferences.
- Clinical: remember the patient's care plan.
- Code copilot: remember the codebase / project context.

Each has different memory requirements.

### 4.2 The per-user memory model

```
User memory:
  user_id: ...
  facts: [
    {fact: "Prefers concise responses", confidence: 0.95, source: session_xyz},
    {fact: "Role: physician", confidence: 1.0, source: explicit},
    {fact: "Specialty: cardiology", confidence: 0.9, source: session_abc},
    ...
  ]
  preferences: {...}
  history_summary: "..."  # aggregated across sessions
```

Per-user; persists across sessions.

### 4.3 The fact-extraction at session end

Periodically (e.g., session end):

```python
def extract_facts_from_session(session, user):
    facts_called_out = call_model(
        prompt=f"Extract factual claims about the user from this conversation:\n{session_text}",
        format="json"
    )
    for fact in facts_called_out:
        user.facts.add_with_dedup(fact)
    user.save()
```

Facts accumulate.

### 4.4 The fact-deduplication

When a fact is encountered:

- Compare to existing facts (semantic similarity).
- If close match: increase confidence; refine.
- If new: add.
- If conflicting: prefer most recent or highest-confidence.

Don't pile up redundant facts.

### 4.5 The cross-session fact retrieval

When a new session starts:

- Fetch relevant user facts.
- Include in prompt:

```
[User memory]
Role: physician
Specialty: cardiology
Preferences: prefers concise responses with citations.
Recent topic: insulin management for elderly patients with T2DM.
```

Lighter than full conversation history; preserves key context.

### 4.6 The "remember everything across all conversations" pattern

For very-long-running relationships (months / years):

- All extracted facts persisted.
- Searchable.
- May exceed in-context budget.

For such cases:

- Fact-retrieval (RAG on facts).
- Top-K relevant facts included.

Cross-link to [ai-engineering-reference-architecture / agent-engineering / memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md).

### 4.7 The cross-tenant boundary

Per-user memory is per-tenant scoped (cross-link to [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md §7.3](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)):

- Memory key: `tenant:T1:user:U1:facts:F1`.
- Cross-tenant access prohibited.

### 4.8 The "user has multiple sessions in parallel" complication

When a user is in multiple sessions (web + mobile):

- Sessions independent; share user memory.
- New facts from one session appear in others.
- Eventual consistency acceptable.

Design for parallel sessions.

### 4.9 The "user is in multiple tenants" case

Some users belong to multiple tenants:

- Memory per-tenant per-user.
- Each tenant has its own view of the user.

This is the right default for B2B; the inverse causes leakage.

---

## 5. Summarization patterns

The mechanics of running summary.

### 5.1 The single-pass summary

Take all older turns; summarize:

```python
def summarize_all(older_turns):
    prompt = f"Summarize: {format(older_turns)}"
    return call_model(prompt)
```

Simple; expensive for long conversations.

### 5.2 The incremental summary

Take existing summary + new turns; update:

```python
def update_summary(existing_summary, new_turns):
    prompt = f"""
    Existing summary: {existing_summary}
    
    New conversation turns since last summary:
    {format(new_turns)}
    
    Update the summary to incorporate the new turns.
    """
    return call_model(prompt)
```

Sub-linear cost; preferred.

### 5.3 The structured summary

Summary in structured form:

```yaml
summary:
  topics_discussed: [insulin management, elderly considerations, kidney function]
  decisions_made: [start with 10 units, monitor weekly]
  facts_established:
    - patient is 78 years old
    - patient has T2DM for 15 years
    - patient has eGFR 45
  user_role: physician
  current_task: medication adjustment
```

Easier to parse; preserves key dimensions.

### 5.4 The summary-cap-size

Bound summary size:

- Target: 500-1500 tokens.
- If approaching: condense (summarize the summary).

Prevents unbounded summary growth.

### 5.5 The summary-validity

Summary should be accurate:

- Periodically: validate against actual turns.
- Adjust prompt if summarizer is missing things.

Validation in production via sampling.

### 5.6 The summarization-prompt tuning

The summarization prompt itself is a prompt:

- "Focus on facts and decisions."
- "Include clinical specifics."
- "Preserve user's stated preferences."

Per-workload prompt for the summarizer.

### 5.7 The "summarizer hallucinated" risk

Summarizers can introduce errors:

- Fabricate facts not in source.
- Distort meaning.

Mitigation:

- Conservative prompt ("only what was explicitly said").
- Validation against turns.
- Recent verbatim provides ground truth.

### 5.8 The "fact emerged 20 turns ago" recovery

If a fact in the summary is wrong:

- Recent verbatim may have correction.
- Model can reconcile.

Within-session, model has access to detail (in recent turns).

---

## 6. Fact extraction patterns

The mechanics of fact extraction.

### 6.1 The fact-extraction prompt

```
You are extracting facts about the user from a conversation.
Output JSON: {facts: [{statement, confidence, evidence_quote}, ...]}

Conversation:
{conversation text}
```

Structured extraction.

### 6.2 The fact format

```yaml
fact:
  statement: "User prefers concise responses"
  confidence: 0.9
  evidence_quote: "Please keep your responses short and to the point"
  category: preference
  added_at: 2026-05-27
  source_session: sess_xyz
```

Structured; queryable.

### 6.3 The fact categories

Pre-defined categories:

- profile (role, name, location).
- preference (style, format).
- context (current task, ongoing topic).
- relationship (other people / things mentioned).
- knowledge (what user knows / has expressed).

Per category, the extraction handling differs.

### 6.4 The semantic deduplication

```python
def add_fact(user, new_fact):
    existing = find_similar(user.facts, new_fact)
    if existing:
        # Update confidence; possibly refine statement
        existing.confidence = max(existing.confidence, new_fact.confidence)
        if better_quote(new_fact, existing):
            existing.evidence_quote = new_fact.evidence_quote
    else:
        user.facts.append(new_fact)
```

Avoids fact accumulation.

### 6.5 The fact-decay

Facts may become stale:

- Time-decay confidence (older facts less confident).
- Explicit expiry (preference might change after 6 months).

Or recompute periodically.

### 6.6 The fact-conflict

Two facts conflict:

- "User is a physician" vs "User is a nurse."
- Investigation: clarify; or use most-recent / highest-confidence.

### 6.7 The fact-retrieval for prompts

When assembling a new prompt:

```python
def get_relevant_facts(user, current_query):
    # Filter to relevant facts (semantic similarity to query).
    relevant = semantic_search(user.facts, current_query, top_k=10)
    return format_facts(relevant)
```

Inject into prompt as "user memory" section.

### 6.8 The fact-correction by user

User says: "Actually, I'm a nurse, not a physician."

- Detect correction.
- Update / replace fact.
- Confidence: high (user explicitly stated).

### 6.9 The fact-extraction cost

Per session-end:

- One LLM call to extract.
- ~$0.05-0.20 per session (depending on length and model).

For high-volume: budget.

---

## 7. Retention and compliance

The legal / compliance dimension.

### 7.1 The retention policy

Per workload:

- Short retention (24h): chat where retention is operational.
- Medium retention (30 days): chat that may need replay.
- Long retention (1 year): regulated workflows.
- Indefinite retention: only if explicit (rare; mostly for compliance).

Per-workload; documented.

### 7.2 The right-to-delete

Compliance (GDPR, CCPA):

- User can request deletion.
- All their session + memory data deleted within N days.
- Audit log of deletion.

### 7.3 The PII in memory

Memory may contain PII:

- User's name.
- User's preferences (sensitive if reveals identity).
- User's questions (may include health data).

For PII-bearing memory:

- Encrypted at rest.
- Access-controlled.
- Audit log of access.
- Retention-bounded.

### 7.4 The cross-region considerations

For multi-region deployments (cross-link to [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md)):

- Memory stays in region.
- Cross-region memory access restricted.

### 7.5 The "we keep memory forever" risk

Without retention policy:

- Memory grows unbounded.
- Storage cost rises.
- Compliance exposure (right-to-delete becomes painful).

Define retention from day one.

### 7.6 The session-vs-cross-session retention

May differ:

- Within-session: short retention (operational; while user is engaged).
- Cross-session: longer retention (the value is persistence).

Policy per memory class.

### 7.7 The customer-controlled retention

For enterprise customers:

- Tenant-level retention policy.
- Tenant can request specific retention (longer for compliance; shorter for privacy).

Tenant overlay for retention.

### 7.8 The decryption-on-access

When memory is accessed:

- Decrypt.
- Audit (who accessed; when).

For high-sensitivity memory.

### 7.9 The retention-vs-quality trade-off

Longer retention → more cross-session memory → better personalization.

Shorter retention → less risk → less personalization.

Per-workload trade-off.

---

## 8. Worked Meridian example

Meridian's memory architecture per feature.

### 8.1 The Care Coordinator session memory

Within-session memory: hybrid.

- Recent 5 turns verbatim.
- Older turns: structured summary.
- Per-task fact extraction (patient context, clinical decisions made).

Configuration:

```yaml
care-coordinator-memory:
  type: hybrid
  recent_verbatim_turns: 5
  summarization_trigger: every 5 turns
  summary_target_size: 800 tokens
  
  fact_extraction:
    enabled: true
    categories: [patient_context, decisions, follow_ups]
    trigger: on_session_end
```

Per-session retention: 7 days (operational replay if needed).

### 8.2 The Patient API chat session memory

Within-session memory: hybrid with shorter recent window.

```yaml
patient-api-chat-memory:
  type: hybrid
  recent_verbatim_turns: 3
  summarization_trigger: every 3 turns
  summary_target_size: 500 tokens
```

Shorter conversations; lighter memory.

### 8.3 The cross-session memory for clinicians

Per-clinician memory across sessions:

```yaml
clinician-cross-session-memory:
  type: facts
  categories:
    - role (e.g., "physician")
    - specialty (e.g., "internal medicine")
    - preferences (e.g., "concise responses with citations")
    - recent_topics (last 30 days of topics)
  retention: 90 days
  per_tenant_scoped: yes
```

Per-clinician; ~50-200 facts after several months.

### 8.4 The cross-session memory for patients

Per-patient memory:

- Clinical context: maintained in EHR (not chat memory).
- Conversational context: light (patients may have multiple sessions; chat is mostly stateless).

```yaml
patient-cross-session-memory:
  type: facts
  categories: [communication_preferences]
  retention: 30 days
```

Minimal; PHI stays in EHR.

### 8.5 The "the Care Coordinator remembers across sessions" implementation

Sample flow:

Day 1, Session 1:

- Clinician: "Patient John Smith, recent ER admission..."
- Care Coordinator: provides summary of approach.
- Session ends.
- Fact extraction: "Recently engaged with patient John Smith's ER follow-up."

Day 3, Session 2:

- Same clinician.
- Fetch cross-session memory.
- Memory included: "Recently engaged with John Smith's follow-up..."
- Clinician: "What was that follow-up plan I had for John Smith?"
- Care Coordinator: with the cross-session context, can recall the previous session.

The continuity is there; not naively storing every conversation; structured facts.

### 8.6 The Q1 2026 PHI-in-memory audit

Compliance audit:

- Asked: where is PHI stored?
- Answer: EHR (primary); session storage (operational; 7-day retention); user-level facts (clinician-only, 90-day).
- Patient identifiers in chat: encrypted; tenant-scoped.
- Right-to-delete: tested; verified.

Passed.

### 8.7 The Q2 2026 summarization quality issue

A clinician reported: "The AI summarized our last session incorrectly; said I prescribed X but I asked about it."

Investigation:

- Summary prompt was too aggressive on inferring decisions.
- "Discussed prescribing X" became "prescribed X."

Fix: summarizer prompt tightened to distinguish "discussed" / "considered" / "decided."

Quality check: verified on samples; deployed.

### 8.8 The session-storage cost

Per session: ~5KB compressed + 1KB facts.

For 10k sessions/day: ~60MB/day; 18GB over the 7-day retention.

Database cost: trivial (<$100/month).

Memory store: Redis for active; PostgreSQL for completed.

### 8.9 The summarization cost

Per session, ~3 summarization calls (every 5 turns; ~15-turn average):

- Per call: ~1000 input + 300 output tokens.
- Per call cost: ~$0.008 (Haiku for summarization).
- Per session: ~$0.025.

For 10k sessions/day: ~$250/day → $7,500/month.

Manageable; included in budget.

### 8.10 The lessons

- Hybrid memory is the right default.
- Per-workload tuning matters.
- Compliance considerations from day one.
- Summarization quality matters; eval-driven.

---

## 9. Anti-patterns

### 9.1 The verbatim-forever

**Pattern.** Keep every turn verbatim; hit context limit; conversation ends abruptly.

**Corrective.** Hybrid per §2.4.

### 9.2 The no-cross-session memory

**Pattern.** Each session starts fresh; user reintroduces themselves repeatedly.

**Corrective.** Cross-session memory per §4.

### 9.3 The full-conversation-in-prompt

**Pattern.** No summarization; every prompt includes full history.

**Corrective.** Summarization or truncation per §3.

### 9.4 The summary-without-validation

**Pattern.** Summarizer can hallucinate; nobody validates.

**Corrective.** Sampling-based validation per §5.5.

### 9.5 The unbounded fact accumulation

**Pattern.** Facts pile up forever; no deduplication.

**Corrective.** Semantic dedup per §6.4.

### 9.6 The PII-without-encryption

**Pattern.** Memory contains PII; not encrypted.

**Corrective.** Encryption + access control per §7.3.

### 9.7 The no-retention-policy

**Pattern.** Memory kept forever; storage grows; compliance exposure.

**Corrective.** Retention policy per §7.1.

### 9.8 The cross-tenant memory leak

**Pattern.** User memory not tenant-scoped; one tenant's memory leaks to another.

**Corrective.** Per-tenant scoping per [per-tenant-prompt-and-context.md §7.3](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md).

### 9.9 The "remember everything; will think about retention later" deferral

**Pattern.** Build memory; never define retention; can't comply with right-to-delete.

**Corrective.** Retention from day one per §7.1.

### 9.10 The conversation-fact-conflict-unhandled

**Pattern.** New session contradicts old facts; system uses old without flagging.

**Corrective.** Conflict resolution per §6.6.

---

## 10. Findings (sprint-assignable)

**ARCH-CHA-001 (P0). Verbatim-forever conversation history.** Hits context limit; UX failure. Hybrid per §2.4. Owner: AI platform.

**ARCH-CHA-002 (P0). No cross-session memory.** User experience disconnected. Per §4. Owner: AI platform + product.

**ARCH-CHA-003 (P0). No retention policy.** Compliance exposure. Per §7.1. Owner: AI platform + security.

**ARCH-CHA-004 (P0). Memory not tenant-scoped.** Cross-tenant leakage risk. Per [per-tenant-prompt-and-context.md §7.3](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md). Owner: AI platform + security.

**ARCH-CHA-005 (P1). Summarizer hallucination risk.** Distortion of conversation. Per §5.7. Owner: AI platform.

**ARCH-CHA-006 (P1). No fact deduplication.** Facts accumulate; quality degrades. Per §6.4. Owner: AI platform.

**ARCH-CHA-007 (P1). PII in memory not encrypted at rest.** Compliance gap. Per §7.3. Owner: security + AI platform.

**ARCH-CHA-008 (P1). Right-to-delete not implemented.** Compliance failure. Per §7.2. Owner: AI platform + security + product.

**ARCH-CHA-009 (P1). No per-workload memory configuration.** One-size-fits-all is wrong. Per §8. Owner: AI platform + feature teams.

**ARCH-CHA-010 (P2). Summarizer not eval-validated.** Quality unknown. Per §5.5. Owner: AI platform + eval.

**ARCH-CHA-011 (P2). Fact-conflict unhandled.** Stale or wrong facts persist. Per §6.6. Owner: AI platform.

**ARCH-CHA-012 (P2). Memory access not audit-logged.** Forensic / compliance gap. Per §7.8. Owner: security.

**ARCH-CHA-013 (P2). Cross-region memory not pinned.** Residency violation. Per §7.4. Owner: AI platform + security.

**ARCH-CHA-014 (P2). Tenant-controlled retention not offered.** Customer compliance posture limited. Per §7.7. Owner: product + AI platform.

**ARCH-CHA-015 (P3). Summary growth unbounded.** Long sessions hit context window despite summarization. Per §5.4. Owner: AI platform.

**ARCH-CHA-016 (P3). Per-fact-category granularity not used.** All facts treated alike. Per §6.3. Owner: AI platform.

**ARCH-CHA-017 (P3). Fact-decay not implemented.** Stale facts persist indefinitely. Per §6.5. Owner: AI platform.

**ARCH-CHA-018 (P3). Memory cost not tracked per workload.** Cost dimension invisible. Per §8.8. Owner: FinOps + AI platform.

---

## 11. Adoption sequencing checklist

- [ ] **Define memory model per workload (§2).** Hybrid for most.
- [ ] **Implement session storage with retention policy (§3.2, §7.1).**
- [ ] **Implement summarization (§5).** Per-workload tuned.
- [ ] **Implement fact extraction at session end (§6.3).**
- [ ] **Define cross-session memory architecture (§4).**
- [ ] **Implement per-tenant scoping for memory (§4.7).**
- [ ] **Implement PII encryption at rest (§7.3).**
- [ ] **Implement right-to-delete (§7.2).**
- [ ] **Build memory dashboards (cost, retention compliance).**
- [ ] **Periodic summarizer eval (§5.5).**
- [ ] **Per-workload memory configuration (§8).**

---

## 12. References

**In this folder.**
- [system-prompt-architecture.md](./system-prompt-architecture.md) — system prompt is the other major context component.
- [context-window-budgeting.md](./context-window-budgeting.md) — history is one allocation in the budget.
- [prompt-assembly-patterns.md](./prompt-assembly-patterns.md) — history is assembled.
- [long-context-vs-rag.md](./long-context-vs-rag.md) — context decisions.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](../multi-tenancy-and-isolation/per-tenant-prompt-and-context.md) — tenant-scoped memory.
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — region pinning for memory.
- [data-architecture-for-ai/lineage-and-provenance.md](../data-architecture-for-ai/lineage-and-provenance.md) — fact provenance.
- [guardrails-and-policy-architecture/policy-as-code-for-ai.md](../guardrails-and-policy-architecture/policy-as-code-for-ai.md) — memory access policy.

**Sibling repos.**
- [ai-engineering-reference-architecture / agent-engineering / memory-engineering.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/memory-engineering.md) — engineering-side memory.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — memory cost tracking.

**External.**
- LangChain ConversationSummaryMemory documentation.
- LlamaIndex chat memory patterns.
- GDPR right-to-erasure requirements.
- HIPAA retention requirements for clinical data.
