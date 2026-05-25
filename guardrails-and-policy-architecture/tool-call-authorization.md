# Tool-Call Authorization

> **Audience.** Architects designing agent systems where tools take real-world actions. Engineers reviewing whether an existing agent's tool surface enforces authorization architecturally rather than by convention. **Scope.** The *architectural* placement decision for tool authorization — where it lives, what it enforces, what artifacts it produces — not the engineering practice of building tools (the sibling [ai-engineering-reference-architecture](https://github.com/jeremiahredden/ai-engineering-reference-architecture)'s `agent-engineering/tool-architecture.md` owns that). **Worked client.** Meridian Health. **Companion docs.** [ai-gateway-pattern.md](./) for the per-call control surface (coming); [retrieval-scope-enforcement.md](./) for the parallel pattern on the retrieval side (coming); [reference-patterns/agent-topologies.md](../reference-patterns/agent-topologies.md) for the topology context.

---

## 1. Why this document exists

Tools are the channel by which an agent affects the world. Send a patient message, draft a clinical order, query the EHR, write to a knowledge graph, charge a card, file a ticket, call an external API. Without tools, an agent is a chat program; with tools, it is a system that takes action — sometimes consequential action, sometimes irreversible action, sometimes regulated action.

So tool authorization is the load-bearing guardrail in any agent system that does more than answer questions. Not "tool authorization" as a name for whatever validation logic exists somewhere; tool authorization as a *first-class architectural surface* — a central registry that enumerates every tool, declares each tool's authorization requirements, evaluates them on every call, logs every decision, and refuses to register a tool that does not conform.

Most production agents I see place authorization inline in tool-implementation code. The clinical-order draft tool checks "is this user a clinician?" inside the tool's function body. This pattern has three failure modes:

1. **A new tool gets added without the check.** The pattern is convention; the convention degrades over time and turnover.
2. **The check is inconsistent across tools.** One tool checks role, another checks scope, a third checks tenant; the agent's authorization story is the union of inconsistencies.
3. **The audit trail is incomplete.** A decision made inside tool code is logged at whatever level the tool's author chose; reviewers cannot reconstruct authorization decisions uniformly.

The architectural alternative: **a platform-level tool registry that is the only path to a tool, declares authorization as data, enforces it as code, and logs every decision uniformly**. The Care Coordinator's GA blocker `ARCH-CARE-002` ("side-effect tools are exposed without a uniform HITL contract; implementation enforces HITL by convention rather than registry-level requirement") is exactly the failure this pattern prevents.

This document is opinionated about three things:

1. **The registry is the chokepoint.** Every tool invocation flows through it; the registry resolves authorization, dispatches the call, and logs the decision.
2. **Authorization is declarative data on the tool**, not procedural code in the tool. The registry refuses to register tools without a complete authorization specification.
3. **Side-effect tools require a HITL contract by registration rule.** A tool that takes real-world action cannot be registered without declaring its human-approval path; the registration step is the architectural gate.

Structure: (2) the four tool classes that drive authorization shape; (3) the registry contract; (4) authorization dimensions (role, scope, tenant, time, cost); (5) the HITL contract for side-effect tools; (6) refusal-as-tool-response design; (7) audit logging; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The four tool classes

Authorization architecture starts with classification. Tools fall into four classes; each class has a different authorization surface, a different HITL contract, and a different audit posture. The class is declared at registration; the registry enforces its constraints.

### 2.1 Read-only

Tools that retrieve information without changing any state observable outside the agent loop. Read from a database, query an API for status, look up a record, retrieve from a search index.

- **Authorization surface.** Role + tenant + scope (which records this caller is permitted to see). Often the same as the application's existing read-authorization model.
- **HITL contract.** None required.
- **Audit posture.** Log the call, the parameters (or parameter hashes for sensitivity), the returned shape (not necessarily the full content), the authorization decision. Read access logs are routine; high-cardinality queries can be sampled.

Examples in Meridian: `look_up_drug_interactions`, `retrieve_clinical_guideline`, `search_patient_education`.

### 2.2 Draft-only

Tools that produce a draft of a side effect without committing it. The draft lives in a queue for human review; the tool's output is the draft, not the realized side effect.

- **Authorization surface.** Same as read-only for the draft creation. The eventual approval-and-commit is a separate operation outside the agent's scope.
- **HITL contract.** Implicit — the draft itself is the agent's output; the human commit step is the HITL.
- **Audit posture.** Log the draft creation with full draft content; log the eventual approval / discard event when it happens; preserve the lineage from draft to outcome.

Examples in Meridian: `draft_patient_message`, `draft_clinical_order`, `draft_appointment_slot`. None of these directly send / create / schedule; each produces a draft that flows to the clinician's worklist.

### 2.3 Side-effect-with-HITL

Tools where the side effect is committed only after explicit human approval, but the tool's invocation does more than just "draft" — it might reserve a resource, place a hold, or notify a downstream system that an action is pending. The HITL is a hard contract, not a convention.

- **Authorization surface.** Role + tenant + scope + an "is this caller authorized to *propose* this action" check.
- **HITL contract.** Mandatory. The tool's implementation cannot complete without the approval path being engaged. The registry refuses to register the tool without a declared approval path.
- **Audit posture.** Log the proposal, the resource reservation if any, the eventual approval / rejection, the realized side effect or compensating action. Multi-event lineage on one trace.

Examples in Meridian: `propose_inpatient_admission` (places a tentative bed hold, awaits attending approval to convert to actual admission), `propose_specialist_referral` (places a tentative referral, awaits PCP approval).

### 2.4 Side-effect-immediate

Tools where the side effect is committed at the call site, with no intervening human approval. These are the most authorization-sensitive class; in regulated workloads they should be rare.

- **Authorization surface.** Role + tenant + scope + per-call risk evaluation (does this specific invocation, with these arguments, exceed any defined threshold).
- **HITL contract.** None at call time; explicit out-of-band authorization (a contract, a policy, a one-time human authorization) is the implicit gate.
- **Audit posture.** Log every invocation with full arguments; route to a separate audit sink; alert on threshold breaches.

Examples in Meridian: very few. The agent itself does not commit clinical actions immediately. The closest is `log_clinical_decision_audit_event` — writes an audit record that the agent suggested a course of action and the clinician acknowledged it; no clinical action is taken, but the audit trail is written immediately. The pattern is reserved for cases where the side effect is itself audit-only and the audit value justifies the lack of HITL.

### 2.5 The classification discipline

Every tool, at registration time, must declare its class. The registry rejects registration of tools with no class. The classification is reviewed at registration; class changes (especially escalating from read-only to side-effect) trigger an architecture review and a security review.

The discipline matters because the failure mode is escalation by drift: a tool starts as "read-only" but a future PR adds a side effect to it; without registration-level review, the side effect ships without authorization architecture catching up. The registry's classification + the requirement to re-register on class change forces the review to happen.

---

## 3. The registry contract

The tool registry is a platform component. Every agent invocation that calls a tool goes through it. The registry is the chokepoint where authorization, dispatch, logging, and (for side-effect tools) HITL flow integration happen uniformly.

### 3.1 Registry interface — what tools declare

Every tool registration declares (in a structured manifest, code, or both):

```yaml
tool:
  name: draft_patient_message
  description: |
    Draft a message to a patient. The draft is placed in the outbound
    message queue for clinician approval. Use this tool when the user
    asks to send, write, or compose a message to a patient.
  class: draft-only
  arguments:
    patient_id: { type: string, description: "...", required: true }
    message_type: { type: enum["follow_up","appointment_reminder","education"], required: true }
    message_body: { type: string, description: "...", required: true }
  authorization:
    required_roles: ["rn", "ma", "care_coordinator", "physician"]
    tenant_scope: per_request    # caller's tenant must match patient's tenant
    patient_scope: caller_has_relationship  # caller must have a clinical relationship to the patient
    additional_checks: ["patient_active_at_tenant"]
  side_effect:
    draft_queue: outbound_messages
    eventual_action: send_message
    compensating_action: discard_draft
    approval_required: true
    approval_roles: ["rn", "physician", "care_coordinator"]
  audit:
    log_full_content: true
    audit_sink: clinical_audit
  cost:
    estimated_per_call_usd: 0.001
    rate_limit_per_user_per_hour: 60
```

The schema is enforced at registration. A tool that does not declare `class` is rejected. A `side-effect-with-HITL` class tool without `side_effect.approval_required: true` is rejected. A tool with `class: draft-only` but no `side_effect.draft_queue` is rejected.

### 3.2 Registry interface — call path

When an agent invokes a tool, the call goes to the registry, not directly to the tool:

```python
class ToolRegistry:
    def invoke(
        self,
        tool_name: str,
        arguments: dict,
        caller_context: CallerContext,   # role, tenant, identity, session
        agent_context: AgentContext,     # trace_id, supervisor_id, turn_number
    ) -> ToolInvocationResult:
        tool_spec = self._tools.get(tool_name)
        if not tool_spec:
            return self._refusal("unknown_tool", tool_name)

        # 1. Argument validation against schema
        validated = self._validate_arguments(tool_spec, arguments)
        if not validated.ok:
            return self._error_response(validated.error)

        # 2. Authorization
        authz_decision = self._evaluate_authorization(
            tool_spec, caller_context, validated.arguments
        )
        if not authz_decision.allowed:
            self._audit_log_authz_denied(tool_spec, caller_context, authz_decision)
            return self._refusal(
                "not_authorized",
                authz_decision.user_facing_message,
            )

        # 3. Cost and rate-limit checks
        budget_check = self._check_budgets(tool_spec, caller_context)
        if not budget_check.ok:
            return self._refusal("budget_exceeded", budget_check.message)

        # 4. HITL handoff for side-effect-with-HITL class
        if tool_spec.class_ == "side-effect-with-HITL":
            proposal = self._create_proposal(tool_spec, validated.arguments, caller_context)
            self._audit_log_proposal(tool_spec, caller_context, proposal)
            return ToolInvocationResult(
                success=True,
                kind="proposal_created",
                proposal_id=proposal.id,
                message_to_agent=(
                    f"Proposal {proposal.id} created and awaiting "
                    f"approval. The agent can inform the user."
                ),
            )

        # 5. Draft-only: create draft, return draft handle
        if tool_spec.class_ == "draft-only":
            draft = self._create_draft(tool_spec, validated.arguments, caller_context)
            self._audit_log_draft(tool_spec, caller_context, draft)
            return ToolInvocationResult(
                success=True,
                kind="draft_created",
                draft_id=draft.id,
                draft_summary=draft.summary,
            )

        # 6. Read-only or side-effect-immediate: dispatch to tool impl
        result = self._dispatch(tool_spec, validated.arguments, caller_context)
        self._audit_log_invocation(tool_spec, caller_context, result)
        return result
```

The pattern: the tool's *implementation* is reached only after authorization, argument validation, budget checks, and (for side-effect tools) the HITL flow is engaged. The implementation does not need to do authorization itself; the registry has already done it.

### 3.3 What the registry refuses

The registry enforces invariants at registration time and at call time:

| Registration-time refusal | What it prevents |
|---|---|
| No `class` declared | Tool with undefined authorization shape |
| `side-effect-*` without authorization rules | Tool that anyone could invoke |
| `side-effect-with-HITL` without `approval_required: true` | Tool that claims HITL but does not enforce it |
| `side-effect-immediate` without a security-review marker | Bypass of security review for immediate-effect tools |
| Tool name collision | Ambiguity in tool selection |

| Call-time refusal | What it prevents |
|---|---|
| Caller missing required role | Privilege escalation |
| Caller's tenant differs from per-call tenant scope | Cross-tenant action |
| Arguments fail schema validation | Tool invoked with bad inputs |
| Caller's per-hour rate limit exceeded | Abuse |
| Per-call cost exceeds threshold | Cost runaway |
| Tool is currently disabled (kill switch) | Operational lockout in incident |

Each refusal returns a structured, agent-readable response (see §6). The agent does not see exceptions; it sees a denial it can reason about.

---

## 4. Authorization dimensions

A complete authorization decision spans several dimensions. The registry evaluates them in sequence; any denial fails the whole call.

### 4.1 Role

The caller's role (the human user the agent is acting on behalf of, or — for autonomous agents — the agent's own role) is checked against the tool's `required_roles`. Roles are platform-defined and orthogonal to the agent.

For Meridian: `physician`, `rn` (registered nurse), `ma` (medical assistant), `care_coordinator`, `clinical_pharmacist`, `admin`, etc. Drafting a clinical order requires `physician`; drafting a patient message can be done by `rn`, `ma`, `care_coordinator`, `physician`.

### 4.2 Tenant scope

The caller's tenant must match the per-request tenant context. The tenant of the action's target (the patient's hospital, the order's hospital) must also match. Cross-tenant action is denied by default.

Cross-tenant tooling exists for the tiny set of operations that legitimately span tenants (platform-wide analytics, cross-hospital research-cohort queries) — these are routed through a separate `cross_tenant` tool class with explicit out-of-band authorization, full audit, and human approval per invocation.

### 4.3 Per-record / per-resource scope

The caller must have a relationship to the specific record they are acting on. For Meridian, this is "is this clinician on the care team for this patient?" — answered by querying the EHR's care-team service. The `caller_has_relationship` rule encapsulates the relationship check.

Per-record scope is the dimension most often forgotten. A clinician at hospital A may legitimately have the `physician` role and may legitimately be at the right tenant, but should not be able to draft a message to a patient they have no care relationship with. The per-record check catches this.

### 4.4 Time / context

Some tools are valid only at certain times or in certain contexts. Examples: `propose_specialist_referral` may only be valid during the clinician's active shift; `request_after_hours_consult` is the inverse — only valid outside shift.

Time/context scope is rare but important in regulated workloads. The registry evaluates it as an `additional_checks` entry that can include time-of-day, on-shift, on-rotation, and similar constraints.

### 4.5 Cost / rate

Per-tool, per-caller rate limits prevent abuse and runaway loops. A tool whose `rate_limit_per_user_per_hour` is 60 will refuse the 61st invocation from the same user in an hour, returning a structured refusal the agent can act on.

Per-call cost ceilings (where applicable — e.g., an LLM-backed tool with token cost) prevent a single tool call from blowing through budget. Aggregate with the four-layer cost stack in the sibling [cost-and-finops/cost-budget-circuit-breaker.md](../../ai-engineering-reference-architecture/cost-and-finops/) (coming).

### 4.6 Policy-as-code

For complex authorization where simple role/scope/tenant checks are not expressive enough, the registry evaluates a policy bundle (OPA, Cedar, or a custom evaluator). Policy bundles live as code, deploy with the platform, and are versioned alongside the tool registry.

Use sparingly. Policy-as-code introduces a new artifact to maintain; most tool authorization fits in the simpler dimensions above. The Meridian Care Coordinator uses policy-as-code only for the `propose_specialist_referral` tool, which has cross-specialty referral rules that vary per-tenant per-payer.

---

## 5. The HITL contract for side-effect tools

The single most important architectural commitment in this document: **side-effect-with-HITL is a class with mandatory contract, not a convention enforced by reviewer attention.**

### 5.1 What the contract requires

For a tool to be registered as `side-effect-with-HITL`, the registration must specify:

- **The proposal artifact.** What is created when the tool is invoked: a draft, a tentative reservation, a pending request.
- **The approval surface.** Where the human approver sees the proposal: a worklist queue, an inbox, a UI panel, a notification.
- **The approval roles.** Which roles can approve.
- **The eventual-action mechanism.** What happens on approval: send the message, commit the order, schedule the appointment.
- **The compensating action.** What happens on rejection or timeout: discard the draft, release the reservation, cancel the request.
- **The proposal lifetime.** How long the proposal remains pending before auto-expiration.

The registry refuses to register the tool without all of these.

### 5.2 The interaction with the agent

When the agent invokes a side-effect-with-HITL tool, the tool does not return the realized outcome — it returns a structured response telling the agent that the proposal has been created and is awaiting approval. The agent informs the user; the user (or an approving clinician) takes the approval action through the existing platform UI.

The agent's trace records the proposal. When the human acts (approve or reject), an event flows back to the platform; the registry's audit log links the eventual outcome to the original proposal. The trace is a complete lineage from agent invocation to realized outcome.

### 5.3 What "rubber-stamp HITL" looks like and how to prevent it

The failure mode here is that the human-approval step exists procedurally but is one-click-defeated. The clinician sees a "Click to approve" button and clicks it without reading. The HITL is theater.

Architecturally, the registry cannot prevent this. UI design and process discipline are the corrective layer. But the registry can *support* the UI design with a few moves:

- **Approval data structure includes a summary the human must engage with.** The approval surface displays the proposal in full; the approve action is gated on a viewable-content interaction (scroll-through, content-acknowledgment, or similar).
- **Approval requires the human to attest specific elements**, not just press a button. For high-stakes proposals (clinical orders), the approver must check off specific elements (drug, dose, patient identity verification).
- **The registry records the approval latency.** If the median approval-to-display time is under 2 seconds, that is a signal that approvals are not being read; surface to the team as a process-health metric.

These are UI patterns, but they integrate with the registry's audit log. The discipline is "the registry has the data; the UI uses the data; the metrics catch the degradation."

### 5.4 Worked Meridian HITL flow

For `draft_patient_message`:

1. Agent calls `draft_patient_message` with arguments. Registry validates auth, validates args, creates a draft, returns `draft_id` to the agent. No message has been sent.
2. The agent's response to the user: "I've drafted a follow-up message for [patient initials]. You can review and send it from the outbound queue, or I can show it here."
3. The draft appears in the outbound-message queue in the existing clinical platform UI. The queue is filtered to messages the clinician is authorized to send (their tenant, their patient relationships).
4. The clinician opens the draft. The full text is shown. The clinician can edit. To send, the clinician clicks "Send" — which is a separate UI action confirmed by an attestation ("I confirm this message is appropriate to send to this patient").
5. The send event flows back to the platform. The registry's audit log records the send action linked to the original draft and the original agent invocation. The trace is now complete: agent → draft → human review → send.

If the clinician discards the draft instead, the discard event is similarly logged; no message goes out.

If the draft sits in the queue past its expiration (default 7 days for messages), an auto-expiration event records the lapse.

---

## 6. Refusal-as-tool-response design

When the registry denies a call, the response is a *structured refusal* that the agent can reason about, not an exception. The agent informs the user politely; the system continues; nothing crashes.

### 6.1 The structured refusal shape

```json
{
  "success": false,
  "kind": "tool_refused",
  "refusal_class": "not_authorized",
  "user_facing_message": "You do not have permission to draft messages for patients outside your assigned care team. Try one of your assigned patients, or escalate to a care coordinator.",
  "agent_facing_message": "Caller is not on the care team for the requested patient. Authorization rule: caller_has_relationship. Caller role: rn. Patient hospital: mercy-cleveland. Caller hospital: mercy-cleveland. Suggested recovery: ask the user to confirm the patient or escalate.",
  "suggested_recovery": "verify_patient_or_escalate"
}
```

Two facing messages because they serve different audiences:

- The **user-facing message** is what the agent surfaces to the user (or uses verbatim if the agent decides to). Polite, actionable, does not leak system implementation.
- The **agent-facing message** gives the agent enough detail to reason about whether to try a different approach, ask a clarifying question, or escalate.

### 6.2 Refusal classes

The registry uses a fixed enum of refusal classes:

| Class | Meaning |
|---|---|
| `not_authorized` | The caller lacks the required role / tenant / scope |
| `bad_arguments` | The arguments do not validate against the schema |
| `unknown_tool` | The tool name does not exist |
| `tool_disabled` | The tool is currently disabled (kill switch or incident response) |
| `rate_limited` | The caller's rate limit for this tool is exceeded |
| `budget_exceeded` | A per-call or per-session cost budget is exceeded |
| `policy_denied` | A policy-as-code rule rejected the call |

The agent's prompt is built to handle each class. A `not_authorized` refusal usually leads the agent to inform the user. A `bad_arguments` refusal usually leads the agent to retry with corrected arguments (with a bounded retry count, see the engineering sibling's `agent-engineering-playbook.md` section 6.3). A `tool_disabled` refusal leads to a degraded response.

### 6.3 Refusal-as-feature, not refusal-as-bug

The refusal should feel like a tool's normal response, not like an error. The agent should not enter recovery / repair / panic mode when a tool refuses; it should *accept* the refusal and proceed. This is partly a tool-design choice (the structured response is normal-shape) and partly a prompt-engineering choice (the agent's system prompt treats refusals as expected).

The failure mode if refusals feel like errors: the agent treats refusals as transient, retries them, exhausts retry budgets, and produces confused user-facing output. The corrective is "treat refusals as terminal for that path; pick a different path."

---

## 7. Audit logging

Every authorization decision and every tool invocation produces an audit record. The audit is the trail that supports incident investigation, compliance attestation, and quality improvement.

### 7.1 What gets logged

For every call (regardless of outcome):

- Tool name + version
- Caller identity (user ID, role, session ID)
- Caller tenant
- Trace ID + parent agent identifier
- Authorization decision: allowed, denied, classed
- Authorization reason (the rule that decided)
- Arguments — full for low-sensitivity tools, hashed-with-fingerprint for sensitive ones, with sensitive fields PHI-redacted in storage
- Cost incurred (if applicable)
- Latency

For draft / side-effect tools, additionally:

- Draft ID or proposal ID
- Linked-future-event handle (so the eventual approval / discard event can attach to this record)

For audit-sensitive workloads (Meridian), the audit log is in a dedicated sink with:

- WORM / append-only storage
- Retention per regulatory requirement (Meridian: 7-year retention for clinical-decision events)
- Access controls separate from operational logs (auditors and security teams have access; engineering teams do not for production data, only for the de-identified analytics view)

### 7.2 Cross-event lineage

The audit log is structured so that one logical action (agent invocation → draft creation → human review → approval → send) is one lineage that can be queried as a whole. This is more than appending events; it is engineering the linkage so that the eventual approval event includes the original draft ID, which includes the original agent invocation, which includes the original user request.

Lineage queries support:

- "For this realized side effect, what was the agent's reasoning and what did the human approver see?"
- "For this agent invocation, what proposals were created and what was their disposition?"
- "Show all proposals that expired without action this week, by clinician."

### 7.3 What audit logs do not contain

- Full retrieval content (logged separately at the retrieval audit boundary; see [per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md))
- Full LLM call content (logged separately at the LLM audit boundary; see [observability-and-telemetry/llm-call-instrumentation.md](../../ai-engineering-reference-architecture/observability-and-telemetry/) coming)
- Production tracing payloads (those go to the observability stack)

The tool-call audit log is specifically the authorization-decision and side-effect-lineage record. Keeping it focused makes both regulatory attestation and incident investigation tractable.

---

## 8. Worked Meridian Health example

### 8.1 The tool catalog

The Care Coordinator's tool catalog at production GA:

| Tool | Class | Required roles | HITL |
|---|---|---|---|
| `look_up_drug_interactions` | read-only | any clinical role | no |
| `retrieve_clinical_guideline` | read-only | any clinical role | no |
| `search_tenant_protocols` | read-only | any clinical role on the tenant | no |
| `query_patient_context` | read-only | clinical role with care-team relationship | no |
| `search_patient_education` | read-only | any clinical role | no |
| `draft_patient_message` | draft-only | rn/ma/care_coordinator/physician | implicit (queue) |
| `draft_clinical_order` | draft-only | physician (also np/pa per state scope) | implicit (queue) |
| `draft_appointment_slot` | draft-only | care_coordinator/rn/ma/scheduler | implicit (queue) |
| `propose_specialist_referral` | side-effect-with-HITL | clinical role + policy-as-code per-tenant | mandatory (PCP approval) |
| `propose_inpatient_admission` | side-effect-with-HITL | physician + attending | mandatory (attending approval) |
| `escalate_to_human` | side-effect-immediate | any clinical role | none (the escalation IS the human handoff) |
| `log_clinical_decision_audit` | side-effect-immediate | platform-internal | none (audit write) |

The pattern: most tools are read-only or draft-only. Side-effect-with-HITL is used sparingly, only for actions that need a structured proposal-and-approval flow beyond a simple draft. Side-effect-immediate is reserved for the audit-only and escalation cases.

### 8.2 The registry implementation

A central `meridian.tools.registry` package. Tool registrations are Python decorators on tool implementations:

```python
@registry.register(
    name="draft_patient_message",
    class_="draft-only",
    arguments=PatientMessageArguments,
    authorization=AuthorizationSpec(
        required_roles=["rn", "ma", "care_coordinator", "physician"],
        tenant_scope="per_request",
        additional_checks=["caller_has_relationship_to_patient",
                          "patient_active_at_tenant"],
    ),
    side_effect=SideEffectSpec(
        draft_queue="outbound_messages",
        eventual_action="send_message",
        compensating_action="discard_draft",
        approval_required=True,
        approval_roles=["rn", "physician", "care_coordinator"],
        proposal_lifetime_days=7,
    ),
    audit=AuditSpec(log_full_content=True, audit_sink="clinical_audit"),
    cost=CostSpec(estimated_per_call_usd=0.001,
                  rate_limit_per_user_per_hour=60),
)
def draft_patient_message(args: PatientMessageArguments,
                           ctx: ToolContext) -> DraftMessage:
    # implementation creates the draft; no authorization code here
    return ctx.draft_queue.create(
        kind="patient_message",
        patient_id=args.patient_id,
        message_type=args.message_type,
        body=args.message_body,
        created_by=ctx.caller.identity,
    )
```

The implementation is short. There is no `if not is_clinician(caller): raise` — the registry already checked. The implementation does its job and returns; the registry handles the rest.

### 8.3 The audit log row, end-to-end

For a single `draft_patient_message` invocation that goes through full lineage:

```
event_id: e-2026-05-25-001
trace_id: t-2026-05-25-3a4b
agent: care-coordinator-supervisor
agent_invocation_id: ai-2026-05-25-89ab
tool: draft_patient_message
tool_version: 2.4.0
caller: u-mercy-cleveland-rn-001234
caller_role: rn
caller_tenant: mercy-cleveland
target_patient: pt-44929 (hash: 7c5f3...)
arguments_hash: 9c2a1f...
authorization_decision: allowed
authorization_rules_evaluated: [
  role_check: pass (rn in [rn,ma,care_coordinator,physician]),
  tenant_check: pass (mercy-cleveland == mercy-cleveland),
  caller_has_relationship_to_patient: pass,
  patient_active_at_tenant: pass,
  rate_limit: pass (12/60 hr),
]
draft_id: d-2026-05-25-c8d9
draft_summary: "7-day follow-up reminder, English, reading-level 7"
cost_usd: 0.0008
latency_ms: 142

  // Linked event 1: human review (4 hours later)
  event_id: e-2026-05-25-789a
  trace_id: t-2026-05-25-3a4b   (same)
  parent_event: e-2026-05-25-001 (the draft creation)
  event_kind: human_review_opened
  reviewer: u-mercy-cleveland-rn-001234
  draft_id: d-2026-05-25-c8d9

  // Linked event 2: approval (45 seconds later)
  event_id: e-2026-05-25-789b
  parent_event: e-2026-05-25-789a
  event_kind: draft_approved
  approver: u-mercy-cleveland-rn-001234
  attestation: "I confirm this message is appropriate to send"

  // Linked event 3: send (immediately after approval)
  event_id: e-2026-05-25-789c
  parent_event: e-2026-05-25-789b
  event_kind: side_effect_realized
  side_effect: send_message
  destination: patient portal channel
  message_id: m-2026-05-25-fed3
```

This is the lineage a clinical audit or a HIPAA compliance review can follow end-to-end. Every authorization decision is recorded; every action is attributable; every side effect is traceable from the original agent invocation.

### 8.4 The platform discipline

- The `meridian.tools.registry` package is the only public API for tool invocation. Lint rules prevent direct invocation of tool implementations from outside the registry.
- New tool registrations go through code review by ai-platform-eng plus security-eng for the `side-effect-*` classes.
- Quarterly: ai-platform-eng + security-eng review the full tool catalog. Each tool's class is verified against current behavior; tools that drifted (a "read-only" tool that quietly added a side effect) are re-classified, the registration is updated, and the security review is re-done.
- The registry's audit log is itself audited annually for completeness — random sample of agent invocations are walked through the audit log to verify the lineage is complete.

---

## 9. Anti-patterns

### 9.1 Inline authorization in tool code

Each tool's implementation does its own auth check. The pattern starts uniform and degrades; new tools get added with inconsistent checks; some tools forget to check at all.

**Corrective.** Authorization lives in the registry, not in tools. Tools that do their own checks should be refactored to use registry-driven specs; new tools cannot bypass the registry.

### 9.2 Side-effect tools without HITL declaration

A `send_message` tool that directly sends. The team meant for a clinician to approve out-of-band; in practice, the agent calls the tool and the message goes out.

**Corrective.** Reclassify as `side-effect-with-HITL`; add the proposal-and-approval flow; refuse the old direct-send tool registration.

### 9.3 "Authorization checked at the API gateway"

The team relies on the API gateway to authorize tool calls. The gateway authorizes the *agent's* call to the API; it does not separately authorize each tool the agent invokes inside the API. A misbehaving agent inside an authorized API call has full tool access.

**Corrective.** Gateway authorizes the agent's invocation; the registry separately authorizes each tool call within the invocation. Defense in depth, not either-or.

### 9.4 Per-call rate limit only on the LLM, not on tools

The team has rate limits on the LLM provider but not on tools. An agent in a loop calling a tool 200 times per minute is allowed by the LLM rate limit; the downstream tool is hammered.

**Corrective.** Per-tool, per-caller rate limits at the registry. Tools that interact with limited-throughput backends declare their rate limits in the registration.

### 9.5 Refusal returned as exception

The registry's authorization denial is raised as an exception. The agent sees an error, retries, sees the same error, escalates to "system unavailable" — when the actual condition was a normal authorization refusal.

**Corrective.** Return a structured refusal (section 6) that the agent's prompt knows how to handle.

### 9.6 Audit log is the regular log

Tool calls are logged at INFO level into the regular application log. When audit is required, the team greps through stdout to reconstruct events. Retention is whatever the log infrastructure happens to provide.

**Corrective.** Separate audit sink with explicit retention, access controls, and lineage structure. Audit is not a special tag on operational logs; it is a different storage with different lifecycle rules.

### 9.7 New tools added without registration review

A developer adds a tool by importing the registry decorator and writing the implementation. There is no second-pair-of-eyes review at registration; the security implications are not surfaced until something breaks.

**Corrective.** PR-level review of tool registrations is required (separate reviewer label, blocking on security-eng for `side-effect-*` classes). The registration is the security review.

### 9.8 Tool catalog grows without periodic re-review

The tool catalog accumulates over years. Tools that started as read-only have quietly added side effects; tools that were temporary are still in production; classifications are stale.

**Corrective.** Quarterly catalog review; re-classify drifted tools; deprecate unused ones; verify each tool's classification matches current behavior.

---

## 10. Findings (sprint-assignable)

### ARCH-AUTHZ-001 — Severity: Critical
**Finding.** Side-effect tools are exposed without a registry-level HITL contract; HITL is enforced by convention.
**Recommendation.** Reclassify as `side-effect-with-HITL`; refuse registration without `approval_required` declaration; route side-effect tools through the proposal/approval flow.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-AUTHZ-002 — Severity: Critical
**Finding.** Authorization logic is inline in tool implementations rather than declared at the registry.
**Recommendation.** Move authorization to registration metadata; the registry evaluates; tool implementations do their job only.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-AUTHZ-003 — Severity: Critical
**Finding.** Tool registry does not exist; tools are invoked directly by agent loops via function calls.
**Recommendation.** Build the registry as the chokepoint; lint-rule application code against direct tool invocations.
**Owner.** ai-platform-eng, sprint N+1.

### ARCH-AUTHZ-004 — Severity: High
**Finding.** No tool classification scheme; tools' side-effect posture is implicit.
**Recommendation.** Introduce the four classes (section 2); require `class` at registration; review existing tools for correct classification.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-AUTHZ-005 — Severity: High
**Finding.** Per-record / per-resource scope checks (e.g., caller-has-relationship-to-patient) are not consistently enforced.
**Recommendation.** Add per-record-scope declarations to relevant tools; centralize the relationship-check evaluators.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-AUTHZ-006 — Severity: High
**Finding.** Audit log does not capture authorization decisions (only successful invocations); denied calls are not auditable.
**Recommendation.** Log every authorization decision (allowed, denied, classed) with rule trace; route to the dedicated audit sink.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-AUTHZ-007 — Severity: High
**Finding.** Side-effect lineage (agent invocation → draft → approval → realization) is not linked; reconstructing the chain requires manual log correlation.
**Recommendation.** Engineer the lineage in the audit log (parent_event linkage); verify lineage queries can answer the four core questions in section 7.2.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-AUTHZ-008 — Severity: High
**Finding.** Tool registrations are not security-reviewed; tools with side-effect classes can land without explicit security sign-off.
**Recommendation.** Add a required-reviewer rule on PRs touching tool registrations; security-eng is mandatory reviewer for `side-effect-*` classes.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### ARCH-AUTHZ-009 — Severity: High
**Finding.** Refusals are returned as exceptions; agents retry refused calls, leading to wasted cost and confused user-facing output.
**Recommendation.** Adopt structured refusal pattern (section 6); update agent prompts to handle each refusal class deterministically.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-AUTHZ-010 — Severity: High
**Finding.** Per-tool rate limits do not exist; an agent in a loop can hammer a backend.
**Recommendation.** Declare per-tool, per-caller rate limits at the registry; reject calls exceeding limit with a `rate_limited` refusal.
**Owner.** ai-platform-eng, sprint N+2.

### ARCH-AUTHZ-011 — Severity: Medium
**Finding.** HITL approval surface allows one-click approval without engagement-with-content; rubber-stamp pattern is observed.
**Recommendation.** Engineer the approval surface to require content engagement (scroll, attestation, specific element check-off); measure approval latency as a process-health metric.
**Owner.** ai-platform-eng + clinical-platform-design, sprint N+3.

### ARCH-AUTHZ-012 — Severity: Medium
**Finding.** Quarterly tool-catalog review does not happen; classifications and authorization rules drift from current behavior.
**Recommendation.** Schedule the quarterly review; assign ai-platform-eng + security-eng as owners; re-classify drifted tools.
**Owner.** ai-platform-eng team lead, sprint N+3.

### ARCH-AUTHZ-013 — Severity: Medium
**Finding.** Cross-tenant tooling exists alongside the regular registry without separate authorization gates.
**Recommendation.** Separate `cross_tenant` tool class with explicit out-of-band authorization, per-invocation human approval, dedicated audit sink.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### ARCH-AUTHZ-014 — Severity: Medium
**Finding.** Tool registry has no kill-switch / disable mechanism; in an incident, disabling a misbehaving tool requires a code deploy.
**Recommendation.** Add a tool-disable feature flag readable by the registry at call time; document the kill-switch runbook.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ARCH-AUTHZ-015 — Severity: Medium
**Finding.** Argument schema validation is inconsistent across tools; some tools accept free-text where structured types would be safer.
**Recommendation.** Audit tool argument schemas; replace free-text fields with enums / structured types where the value set is bounded.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-AUTHZ-016 — Severity: Medium
**Finding.** Policy-as-code is used for some tool authorizations but its bundle deployment is decoupled from tool-registry deployment; policy can be out of sync with tool definitions.
**Recommendation.** Bundle policy with tool registry; deploy as one artifact; version together.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-AUTHZ-017 — Severity: Medium
**Finding.** Audit log retention is not aligned with regulatory requirement (currently 90-day default; HIPAA requires 6+ years for clinical-decision support events).
**Recommendation.** Increase retention to 7 years for clinical-audit sink; verify with compliance team.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+4.

### ARCH-AUTHZ-018 — Severity: Medium
**Finding.** The agent prompt does not include guidance for handling structured refusals; agent behavior on refusal is inconsistent.
**Recommendation.** Update the supervisor / worker system prompts to include refusal-handling rules; eval-test the refusal handling.
**Owner.** ai-platform-eng, sprint N+3.

### ARCH-AUTHZ-019 — Severity: Low
**Finding.** Per-tool cost telemetry is captured in the registry but not surfaced in dashboards; cost-per-tool analysis requires manual queries.
**Recommendation.** Add per-tool cost dimension to the FinOps dashboards.
**Owner.** ai-platform-eng, sprint N+4.

### ARCH-AUTHZ-020 — Severity: Low
**Finding.** Tool catalog is undocumented; new engineers cannot discover what tools exist and what they do.
**Recommendation.** Generate the tool catalog documentation from the registry metadata; commit alongside the code; surface in the platform handbook.
**Owner.** ai-platform-eng, sprint N+4.

---

## 11. Adoption sequencing checklist

For a team about to land tool-using agent features:

- [ ] **Sprint 0 — design.** Tool catalog with classifications. Decide which tools are read-only, draft-only, side-effect-with-HITL, side-effect-immediate. Get security-eng sign-off on the classifications.
- [ ] **Sprint 1 — registry.** Build the registry as the chokepoint. Required `tenant_id` / `class` / `authorization` declarations. Lint rule against direct tool invocation. Structured refusal pattern.
- [ ] **Sprint 1 — HITL contract.** Side-effect-with-HITL class requires proposal / approval / compensation declarations. Registry refuses non-conforming registration.
- [ ] **Sprint 2 — audit sink.** Dedicated audit sink for authorization decisions and side-effect lineage. WORM retention. Access controls.
- [ ] **Sprint 2 — agent prompt integration.** Supervisor / worker prompts updated to handle each refusal class. Eval tests for refusal handling.
- [ ] **Pre-launch — security review.** Walk through every side-effect tool. Confirm HITL flow. Confirm audit lineage. Document for the customer compliance package.
- [ ] **Pre-launch — UI engagement design.** Approval surface requires content engagement; approval latency metrics surfaced.
- [ ] **Post-launch — quarterly review.** Tool catalog re-classification; rate-limit calibration; audit-log completeness sampling.
- [ ] **Ongoing — drift detection.** A new tool's classification cannot change without re-registration + security review. Lint and CI rules enforce.

A team that follows this sequencing has the architectural foundation for an agent that takes consequential action safely. A team that defers the registry pattern carries the authorization-by-convention tax forever, and surfaces it as a Sev-1 incident the first time a tool change ships without a corresponding authorization update.

---

## 12. References

- OAuth 2.0 / OIDC for the underlying identity model the registry consumes.
- Open Policy Agent (OPA) documentation for the policy-as-code option referenced in section 4.6.
- AWS Cedar documentation for the alternate policy-as-code engine.
- Anthropic, OpenAI tool-use documentation for the SDK-level patterns the registry wraps.
- MCP (Model Context Protocol) specification for the distributed-tool patterns the registry must accommodate.
- HIPAA Security Rule §164.312(b) (audit controls) — the regulatory frame for the Meridian audit-log design.
- This repo: [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) for the worked architecture context, including finding `ARCH-CARE-002` that this document closes.
- This repo: [reference-patterns/agent-topologies.md](../reference-patterns/agent-topologies.md) for the supervisor / worker context in which the registry operates.
- This repo: [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](../multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) for the parallel pattern on the retrieval side.
- Sibling repo: [ai-engineering-reference-architecture/agent-engineering/agent-engineering-playbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/agent-engineering/agent-engineering-playbook.md) for the tool-implementation engineering practice that complements this architectural pattern.
- Sibling repo: [ai-security-reference-architecture/agent-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) for adversarial considerations — agents being manipulated into making unauthorized tool calls.
