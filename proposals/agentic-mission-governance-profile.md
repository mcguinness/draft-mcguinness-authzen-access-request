# Agentic Mission Governance Profile of the AuthZEN Access Request Profile

**Status:** Proposal.  Captures what a profile would look like that binds Mission-style agent governance to the AuthZEN Access Request profile.  Not yet a draft.

**Profiles:** [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md) (referred to below as "the base profile").

**Related work:** Mission Shaping series and Mission Architecture implementation notes published at https://notes.karlmcguinness.com/notes, especially:

- [The Mission Shaping Problem](https://notes.karlmcguinness.com/notes/the-mission-shaping-problem)
- [Mission Shaping Is Not Enough](https://notes.karlmcguinness.com/notes/mission-shaping-is-not-enough)
- [Mission-Bound OAuth Architecture](https://notes.karlmcguinness.com/notes/mission-bound-oauth-architecture)
- [AAuth Now Has a Mission Layer](https://notes.karlmcguinness.com/notes/aauth-now-has-a-mission-layer)
- [Mapping Mission Architecture to AAuth](https://notes.karlmcguinness.com/notes/mapping-mission-architecture-to-aauth)
- [Implementing Mission Architecture End-to-End](https://notes.karlmcguinness.com/notes/implementing-mission-architecture-end-to-end)

## Motivation

The base profile defines a general protocol handoff for requestable denials: a PEP receives `decision: false`, submits an Access Request, receives a Task Handle, and re-evaluates after approval.  That is useful for many approval workflows, but agentic governance needs a stronger semantic object than "the denied action was approved."

Long-running agents execute under a task, goal, or Mission.  They discover resources at runtime, pause and resume, delegate to child agents, encounter adversarial content, and often require bounded expansion rather than a single yes/no grant.  In that setting, an approval should be tied to a durable Mission authority record: the approved task, current authority envelope, version hash, constraints, gates, delegation bounds, and lifecycle state.

The Mission Shaping series frames the core problem: approved human intent is not automatically governable authority.  A Mission shaper or compiler turns a Mission Proposal into an Approved Mission and then into a Mission Authority Model.  Runtime enforcement points check individual tool calls or API calls against that model.  The companion "Mission Shaping Is Not Enough" note adds the runtime pressure points: scope creep, delegation, headless execution, resumption, containment, trust budgets, and signal-driven governance.  The end-to-end implementation note defines concrete artifacts such as `mission_id`, `constraints_hash`, review packets, approval objects, runtime signals, token projections, child Missions, amendments, and revocation propagation.

This profile maps those Mission architecture concepts onto the base AuthZEN Access Request extension points.  The goal is to make AuthZEN requestable denials a concrete protocol surface for Mission broadening, gated actions, child-Mission derivation, and runtime continuation.

## Scope

This profile defines:

- A Mission-aware request-template vocabulary for agentic authority expansion, gated execution, child-Mission derivation, budget increase, and Mission resumption.
- Submission `context` members that bind an Access Request to a Mission (`context.mission`), to the approved intent (`context.intent`), and to the runtime event that triggered the denial (`client.source` augmentations).
- `requested_access.constraints` members for bounded authority requests.
- A Re-evaluation Mode `approval.state.mission` shape carrying Mission reference, prior and resulting `constraints_hash`, approved scope and constraints, and approval evidence.
- Processing rules for Mission-aware PEPs, PDPs, and Access Request Services.

It does **not** define:

- A universal Mission Authority Model language.
- A Mission shaping or compilation algorithm.
- A Mission Authority Service API.
- A policy language for matching Mission authority to tools, resources, or actions.
- A replacement for OAuth, AAuth, MCP, Cedar, OPA, or AuthZEN.
- Cross-domain semantic interoperability of Mission taxonomies.

## Architectural Mapping

| Mission architecture concept | This profile |
|---|---|
| Approved Mission | Mission record referenced by `context.mission.ref` |
| Mission authority version | `context.mission.constraints_hash`, round-tripped through `approval.state.mission` |
| Mission Authority Service | PDP, Access Request Service, or backend governance service behind either role |
| Mission broadening, gate, delegation, resumption, budget increase | Requestable denial with the corresponding `template` value |
| Approval object | Base profile `result.approval`, extended through `approval.state.mission` |
| Runtime event | `client.source` augmentations (`event_type`, `tool_call_id`) used for audit and routing |
| Tool/MCP enforcement boundary | PEP submitting the Access Request after an AuthZEN denial |

The base profile remains the transport and lifecycle substrate.  This profile supplies Mission-specific vocabulary for agentic governance.  Mission identity, authority version, delegation chain, and approved authority bounds are represented by the submission and approval members defined below; the full Mission authority envelope (resource classes, action classes, attenuation, lifecycle vocabulary) remains deployment-local until cross-deployment implementation experience accumulates.

## Transport Bindings

This profile is a bridge.  PEPs receive Mission state from an upstream transport and normalize it into this profile's wire surface before submitting an Access Request:

* **AAuth deployments.**  Mission state arrives in the `AAuth-Mission` header.  The PEP normalizes the header's Mission reference into `context.mission.ref` and the header's authority version into `context.mission.constraints_hash`.  An internal AAuth Mission identifier MUST NOT be exposed in cross-domain Access Requests; deployments that need it use `context.mission.id` only inside a single administrative trust domain.
* **Mission-Bound OAuth deployments.**  Mission state arrives in the OAuth access token (typically as a claim or as an `authorization_details` entry per RFC 9396).  The PEP normalizes the token's Mission reference into `context.mission.ref` and the token's Mission authority version into `context.mission.constraints_hash`.  The submitted Access Request authenticates as the OAuth client per the base profile's Authorization and Authentication rules.

A PEP that does not receive Mission state from either transport MUST NOT fabricate `context.mission.ref` or `context.mission.constraints_hash` values.  Access Request Services MUST verify the values against trusted Mission state regardless of which transport delivered them.

## Profile Registrations

### Request-template Vocabulary

Registers the following well-known values for `context.access_request.template` in the base profile's Requestable Denial Context.  The PDP sets these values when emitting a requestable denial; the PEP echoes them unchanged at `denial.template` in the submission.

| Value | Meaning |
|---|---|
| `mission_broadening` | The requested action is outside the current Mission authority envelope but may be added by governed expansion. |
| `mission_gate` | The requested action is inside the Mission class but blocked by a stage gate, approval gate, or release gate. |
| `mission_delegation` | The requested action requires deriving a child Mission or delegated authority for a sub-agent or downstream actor. |
| `mission_resumption` | The Mission is paused or suspended and requires review before execution can resume. |
| `mission_budget_increase` | The requested action would exceed an approved budget or runtime limit, and the agent is requesting additional allowance. |

These template values are routing hints, not authorization policy.  The Access Request Service and PDP determine actual approval scope and enforcement semantics from trusted Mission state.

### Capability URN

Registers a capability URN for advertising this profile in PDP metadata's `capabilities` array:

`urn:openid:authzen:capability:access-request:agentic-mission-governance`

## Minimum Wire Contract

A minimally interoperable deployment of this profile uses the following members:

| Location | Member | Purpose |
|---|---|---|
| `context.mission` | `ref` | Protocol-facing Mission reference. |
| `context.mission` | `constraints_hash` | Current Mission authority version known to the PEP. |
| `context.intent` | `intent_hash` | Non-disclosing reference to approved intent. |
| `client.source` | `event_type` | Runtime event that produced the denial or escalation. |
| `client.source` | `tool_call_id` | Tool or API call associated with the denial, when applicable. |
| `requested_access` | `constraints` | Requested bounds for approved authority. |
| `approval.state.mission` | `ref` | Mission reference to which the approval applies. |
| `approval.state.mission` | `constraints_hash` | Approved or successor Mission authority version. |
| `approval.state.mission` | `approval_evidence` | Reference or digest binding the review packet and approval record. |

Mission references are protocol-facing handles.  Deployments MAY carry an internal Mission identifier in `context.mission.id` or `approval.state.mission.id` only inside a single administrative trust domain where disclosure is acceptable.  Cross-domain, federated, or externally exposed deployments SHOULD use `ref` and keep internal identifiers out of protocol messages.

## Submission Extensions

### `context.mission`

Registers the `mission` member at the base profile's Access Request submission `context` extension point.

`context.mission` is an object binding the submitted Access Request to the active Mission and its current authority version.  It has the following well-known members; deployments and downstream profiles MAY register additional members (purpose classification, plan-step identifiers, runtime state vocabulary, proposal references) per the base profile's Naming Extensions rules.

| Member | Type | Description |
|---|---|---|
| `ref` | String | Protocol-facing Mission reference known to the PDP or Access Request Service.  REQUIRED for Mission-governed denials. |
| `id` | String | Internal Mission identifier.  OPTIONAL and suitable only inside a single administrative trust domain. |
| `constraints_hash` | String | Opaque current Mission authority version known to the PEP. |
| `status` | String | PEP-observed Mission lifecycle state, such as `active`, `paused`, `suspended`, `pending_approval`, or `revoked`. |
| `run_id` | String | Identifier of the agent run, workflow run, or durable execution instance that produced the denial. |
| `parent_ref` | String | Parent Mission reference when this request concerns a child Mission or delegated execution. |

The PEP MUST NOT fabricate Mission references, Mission identifiers, or authority version values.  The Access Request Service MUST verify `context.mission.ref` and `context.mission.constraints_hash` against trusted Mission state before relying on them.  When `context.mission.id` is present, the service MUST verify that it maps to the same Mission as `context.mission.ref`.

`constraints_hash` is a comparison token, not a portable authority representation.  This profile treats it as opaque.  A deployment MAY define a canonicalization and digest algorithm for producing it, but a receiver MUST compare it according to the issuing Mission authority service's rules rather than recomputing it from fields in the Access Request submission unless a shared profile defines the exact input and canonicalization.

### `context.intent`

Registers the `intent` member at the Access Request submission `context` extension point.

`context.intent` carries non-authoritative intent and rationale evidence for reviewers, audit, and risk evaluation.  It has the following members:

| Member | Type | Description |
|---|---|---|
| `intent_hash` | String | Hash of the original user instruction or approved intent record, identifying the intent without disclosing its contents. |
| `agent_rationale` | String | Agent-supplied rationale for why the denied action is needed. |
| `rationale_source` | String | Source of the rationale, such as `agent`, `orchestrator`, `mission_shaper`, or `human`. |
| `taint_state` | String | PEP-observed context-integrity state, such as `clean`, `external_content_seen`, `untrusted_content_seen`, or `tainted`. |

`context.intent` is evidence, not authority.  A PDP or Access Request Service MUST NOT treat `agent_rationale` or other free-text intent members as sufficient authorization policy.

`agent_rationale` is supplied by the agent or its runtime, both of which may have processed untrusted content earlier in execution.  An Access Request Service MUST NOT render `agent_rationale` to human reviewers, AI supervisors, or other LLM-based evaluators without sanitization sufficient to neutralize indirect prompt injection.  Sanitization techniques include explicit content escaping in the review surface, presenting `agent_rationale` as quoted untrusted input distinguishable from review-system instructions, or omitting the field entirely when the review surface cannot guarantee separation.

This profile deliberately does not register a member carrying the raw user instruction.  The user instruction can be linked by `intent_hash` and reviewed out of band; including its text in the Access Request submission distributes sensitive prompt content across services and creates an additional indirect prompt injection surface.

### Runtime Event Augmentations on `client.source`

The triggering runtime event can be carried as augmentation members on the base profile's `client.source`.  This profile registers the following well-known names for those augmentations; deployments and downstream profiles register additional members per the base profile's Naming Extensions rules:

* `event_type` (String): the runtime event that produced the denial, such as `tool.denied`, `gate.required`, `budget.exceeded`, `delegation.requested`, `mission.paused`, or `anomaly.flagged`.
* `tool_call_id` (String): tool or API call identifier that triggered the denial, when applicable.

The Access Request Service MUST distinguish verified observations from self-reported evidence before using `client.source` augmentations as authorization input.

### `requested_access.constraints`

Registers the `constraints` member at the `requested_access` extension point.

`constraints` is an object expressing requested bounds on approved authority.  This profile defines the following well-known members; deployments and downstream profiles MAY register additional members per the base profile's Naming Extensions rules.

| Member | Type | Description |
|---|---|---|
| `max_invocations` | Integer | Maximum number of approved calls. |
| `max_duration_seconds` | Integer | Maximum runtime duration for the approved expansion. |
| `allowed_tools` | Array | Requested tool identifiers. |
| `requires_release_gate` | Boolean | Whether irreversible or external effects require a later approval gate. |
| `reusable_within_mission` | Boolean | Whether approval can be reused more than once within the same Mission version. |
| `requested_child_actor` | Object | Actor requested for child Mission derivation when `denial.template` is `mission_delegation`. |

These constraints describe the requester's ask.  Deployment-specific bounds such as record counts, monetary spend, resource-class or action-class allowlists and denylists are commonly needed but are not interoperable enough to register in this profile; downstream profiles or deployments register their own.

Approved constraints are returned in `approval.state.mission.constraints` and are authoritative only after PDP verification during re-evaluation.

The kind of Mission change being requested (broadening, gate satisfaction, delegation, budget increase, resumption) is conveyed by the `template` value in `context.access_request.template` (echoed in the submission as `denial.template`); the requested authority bounds are conveyed by `requested_access.constraints` and `allowed_tools`; rationale is conveyed by `context.intent.agent_rationale`.  For `mission_delegation`, `client.actor` identifies the immediate actor submitting the request, while `requested_access.constraints.requested_child_actor` identifies the proposed child actor.  A separate `mission_delta` object is not required.

Mission changes that affect enforceable authority SHOULD produce a new Mission authority version after approval, represented by a new `constraints_hash` in `approval.state.mission`.

## Approval Result Extension

### `approval.state.mission`

Registers a structured use of `approval.state` for Re-evaluation Mode results.

When this profile is used, `result.approval.state` MUST be an object containing a `mission` member.  The PEP round-trips the full `approval` object unchanged at `context.approval` during re-evaluation.

`approval.state.mission` has the following well-known members; deployments and downstream profiles MAY register additional members (gate satisfaction details, delegation actor information, resumption instructions) per the base profile's Naming Extensions rules.

| Member | Type | Description |
|---|---|---|
| `ref` | String | Protocol-facing Mission reference to which the approval applies.  REQUIRED. |
| `id` | String | Internal Mission identifier.  OPTIONAL and suitable only inside a single administrative trust domain. |
| `parent_ref` | String | Parent Mission reference when this approval creates or authorizes a child Mission (`template: "mission_delegation"`). |
| `prior_constraints_hash` | String | Mission authority version against which the request was evaluated. |
| `constraints_hash` | String | Mission authority version produced or approved by this result. |
| `approved_scope` | Object | Approved tools, actions, or resource scope. |
| `constraints` | Object | Approved runtime constraints and budgets. |
| `approval_evidence` | Object | Reference or digest for the review packet and approval record. |

The PDP MUST verify that `approval.state.mission.ref` identifies the current Mission, that the presented approval applies to the current `constraints_hash` or a valid successor, and that the current evaluation falls within `approved_scope` and `constraints`.  When `approval.state.mission.id` is present, the PDP MUST verify that it maps to the same Mission as `approval.state.mission.ref`.

`approval_evidence` MUST identify or digest the review packet and approval record that caused the Mission approval result.  The evidence binding MUST allow the Access Request Service or PDP to determine, directly or through trusted server-side state, the approver identity or policy authority, the displayed review packet or its digest, the approved scope, the prior `constraints_hash`, the resulting `constraints_hash` when changed, the approval time, and the approval expiry.

## Processing Rules

### PEP Processing Rules

A Mission-aware PEP:

- MUST include `context.mission.ref` when submitting an Access Request for a Mission-governed denial.
- SHOULD include `context.mission.constraints_hash` when known.
- MUST preserve the actor chain required by the base profile and any Mission delegation chain known to the PEP.
- MUST NOT treat `context.intent.agent_rationale` or any free-text Mission purpose as authorization.
- MUST submit Mission broadening, gate, delegation, budget, and resumption requests only when the denied AuthZEN Decision included `context.access_request`.
- MUST round-trip `result.approval` unchanged at `context.approval` during re-evaluation.
- MUST refresh Mission state or stop execution when the PDP indicates that the Mission version is stale, revoked, suspended, or otherwise not applicable.
- MUST NOT continue execution solely because a Mission-related Access Request task reached `approved`; the PDP remains authoritative at re-evaluation time.

### PDP Processing Rules

A Mission-aware PDP:

- MUST bind requestable denials to the current Mission reference, Mission authority version, Subject, Resource, Action, and authorization-relevant Context.
- SHOULD return `context.access_request.template` using one of this profile's template values when the denial corresponds to a Mission governance action.
- MUST validate `context.approval` against trusted Mission state during re-evaluation.
- MUST reject or ignore approval references whose `constraints_hash` is stale, whose Mission has been revoked or suspended, or whose approved scope does not cover the current evaluation.
- SHOULD return Mission freshness information in the re-evaluation response context, such as the current `constraints_hash` and approval expiry, when doing so does not leak sensitive governance state.

### Access Request Service Processing Rules

A Mission-aware Access Request Service:

- MUST verify Mission references and Mission authority versions before accepting a Mission-governed Access Request.
- MUST distinguish self-reported runtime evidence from independently observed evidence.
- SHOULD treat `context.intent.taint_state` as a review-rigor signal: when it reports that external or untrusted content was processed (`external_content_seen`, `untrusted_content_seen`, or `tainted`), the service SHOULD escalate review (for example, require human approval, narrow the approved scope, or disable any auto-approval path) rather than resolve the request on a fast path.  Because `taint_state` is PEP-observed and self-reported, the service MUST NOT treat a `clean` value as assurance that the agent's context is uncompromised.
- MUST evaluate whether the requested Mission governance action is a narrowing, broadening, gate satisfaction, delegation, budget increase, or resumption request.
- MUST NOT approve broadening, child-Mission derivation, gate satisfaction, or budget increase unless local Mission governance policy permits it.
- MUST bind the approval result to the Mission reference, prior authority version, approved authority version, approved scope, constraints, requester, actor chain, and approval evidence.
- SHOULD emit or retain runtime signals sufficient to reconstruct the original denial, Mission state, Access Request task, approval decision, re-evaluation, and final action.

## Polling Cadence

Mission resumption and gated execution can introduce long-running Access Request tasks (minutes to days when the Mission is suspended pending human review).  PEPs SHOULD use the base profile's Task Status Endpoint polling guidance (exponential backoff, `Retry-After` honored, polling stops at `task.expires_at` or terminal status).  Access Request Services SHOULD prefer callbacks or deployment-level event subscriptions for long-running Mission flows; PEPs subscribed to those channels MAY skip polling entirely and rely on push notification, falling back to a single status retrieval after each notification.

## Security Considerations

This profile inherits the base profile's security considerations and adds Mission-specific risks:

| Risk | Mitigation |
|---|---|
| Stale Mission authority version | PEPs carry `constraints_hash`; PDPs verify freshness during re-evaluation and reject stale approvals. |
| Mission reference substitution | Denial binding and approval state are bound to `context.mission.ref`; PDPs verify the reference against trusted Mission state. |
| Child delegation broadening | `mission_delegation` requests identify the requested child actor separately from the submitting actor; Access Request Services apply local delegation and attenuation policy before approval. |
| Prompt-injected rationale | `agent_rationale` is treated as untrusted evidence and MUST be sanitized or omitted before display to human reviewers or AI supervisors. |
| Compromised agent context | A tainted `context.intent.taint_state` escalates review rigor (human approval, scope narrowing, no auto-approval); because the value is self-reported, a `clean` state MUST NOT be treated as assurance that context is uncompromised. |
| Approval evidence spoofing | `approval_evidence` must bind the review packet, approver or policy authority, approved scope, prior and resulting authority versions, approval time, and expiry. |
| Cross-domain Mission leakage | Protocol messages use `ref`; internal Mission identifiers are only for same-trust-domain use. |
| Runtime event self-reporting | Access Request Services distinguish independently observed events from PEP- or agent-reported events before using them as authorization input. |
| Mission revocation bypass | A PDP MUST reject any approval applied to a Mission whose current `context.mission.status` is `revoked`, regardless of `approval.approved_until` or `approval.state.mission.constraints_hash` freshness; revocation overrides time-based and version-based validity. |

## Conformance Checklist

A Mission-aware deployment of this profile:

- MUST publish `access_request_endpoint`, `jwks_uri`, and the base profile's PDP metadata.
- MUST advertise `urn:openid:authzen:capability:access-request:agentic-mission-governance` in `capabilities`.
- MUST use `result.mode: reevaluate` (the only base completion mode).
- MUST use one of `mission_broadening`, `mission_gate`, `mission_delegation`, `mission_resumption`, or `mission_budget_increase` as `context.access_request.template` when the denial corresponds to a Mission governance action, echoing the value at `denial.template`.
- MUST include `context.mission.ref` in submissions for Mission-governed denials.
- SHOULD include `context.mission.constraints_hash` when known.
- MUST NOT include the raw user instruction in `context.intent`; only `intent_hash` is registered for cross-service propagation.
- MUST place Mission approval state inside `approval.state.mission` when issuing approval results, so the PEP round-trips it unchanged at `context.approval.state.mission` during re-evaluation.
- MUST verify Mission state, `constraints_hash` freshness, and approval applicability at re-evaluation; the PDP remains authoritative at enforcement time.
- MUST treat `task.status: expired` as denial of the underlying Mission governance action.

A base-profile-only PEP that submits an Access Request without these Mission extensions remains conformant to the base profile.  Mission-aware Access Request Services validate Mission extensions when present and SHOULD NOT reject submissions that omit them unless deployment policy requires Mission context for a particular denial template.

## Example: Mission Broadening for a Tool Class

An agent is executing Mission reference `mr_01JR9S4YDY6QF5Q9Q54M0YB4V1` with current authority version `sha256-abc123`.  It attempts to call a CRM search tool that is outside the current Mission envelope.  The PDP returns a requestable denial with `template: "mission_broadening"`.

The PEP submits:

```json
{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "tool",
    "id": "crm.search_accounts"
  },
  "action": {
    "name": "invoke"
  },
  "context": {
    "mission": {
      "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
      "constraints_hash": "sha256-abc123",
      "status": "active",
      "run_id": "run_01K2Q4E0H5"
    },
    "intent": {
      "agent_rationale": "The renewal report requires account metadata for affected customers.",
      "rationale_source": "agent",
      "taint_state": "external_content_seen"
    }
  },
  "requested_access": {
    "constraints": {
      "allowed_tools": ["crm.search_accounts", "crm.read_account"],
      "max_invocations": 25,
      "max_duration_seconds": 604800,
      "requires_release_gate": false
    }
  },
  "client": {
    "actor": {
      "id": "agent_renewal_assistant_v3",
      "issuer": "https://agents.example.com",
      "type": "ai_agent"
    },
    "source": {
      "session_id": "sess_01K2Q3Z9C1",
      "event_type": "tool.denied",
      "tool_call_id": "call_01K2Q4E6M4"
    }
  },
  "denial": {
    "evaluation_id": "eval_01K2Q4DP3K",
    "evaluated_at": "2026-05-15T21:00:00Z",
    "reason": "mission_scope_missing",
    "template": "mission_broadening",
    "binding_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6InBkcC0xIn0..."
  }
}
```

If approved, the completed task returns a Re-evaluation Mode result:

```json
{
  "task": {
    "id": "arq_01K2Q4K1XB",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01K2Q4M8FD",
      "approved_at": "2026-05-15T21:04:00Z",
      "approved_until": "2026-05-22T21:04:00Z",
      "state": {
        "mission": {
          "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
          "prior_constraints_hash": "sha256-abc123",
          "constraints_hash": "sha256-def456",
          "approved_scope": {
            "tools": ["crm.search_accounts", "crm.read_account"]
          },
          "constraints": {
            "max_invocations": 25,
            "max_duration_seconds": 604800
          },
          "approval_evidence": {
            "review_id": "rev_01K2Q4HJ2M",
            "review_hash": "sha256-review789"
          }
        }
      }
    }
  }
}
```

The PEP then re-evaluates the original tool invocation with the `approval` object at `context.approval`.  The PDP verifies the Mission state, the successor `constraints_hash`, the approved scope, and any remaining runtime constraints before returning `decision: true`.

## Example: Mission Gate for Release

An agent has already drafted a customer renewal notice under Mission reference `mr_01JR9S4YDY6QF5Q9Q54M0YB4V1`.  The Mission allows drafting but requires a release gate before sending external email.  The PDP denies the send action with `template: "mission_gate"`.  The Access Request Service approves only the release gate, not a broader tool-class expansion.

The completed task result can carry gate satisfaction without changing the Mission authority version:

```json
{
  "task": {
    "id": "arq_01K2Q5AF3E",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01K2Q5D9SH",
      "approved_at": "2026-05-15T21:20:00Z",
      "approved_until": "2026-05-15T21:35:00Z",
      "state": {
        "mission": {
          "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
          "prior_constraints_hash": "sha256-def456",
          "constraints_hash": "sha256-def456",
          "approved_scope": {
            "tools": ["email.send_external"],
            "actions": ["send"],
            "resource": "renewal_notice_01K2Q57Q2R"
          },
          "constraints": {
            "max_invocations": 1,
            "reusable_within_mission": false
          },
          "approval_evidence": {
            "review_id": "rev_01K2Q5BJ6C",
            "review_hash": "sha256-review-gate456"
          }
        }
      }
    }
  }
}
```

During re-evaluation, the PDP verifies that the approval applies to the same Mission reference and current `constraints_hash`, that the requested send matches the approved release artifact, and that the single-use gate has not already been consumed.

## Example: Mission Delegation to a Child Agent

An agent executing Mission reference `mr_01JR9S4YDY6QF5Q9Q54M0YB4V1` (authority version `sha256-def456`) needs to spin up a dedicated sub-agent to compile a report while it continues the renewal review.  Deriving that child requires delegated authority, so the PDP denies with `template: "mission_delegation"`.  The submitting actor is the parent agent; the proposed child actor is named separately in `requested_access.constraints.requested_child_actor`.

The PEP submits:

```json
{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "tool",
    "id": "reports.generate"
  },
  "action": {
    "name": "invoke"
  },
  "context": {
    "mission": {
      "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
      "constraints_hash": "sha256-def456",
      "status": "active",
      "run_id": "run_01K2Q4E0H5"
    },
    "intent": {
      "agent_rationale": "A dedicated sub-agent should compile the quarterly report while the primary agent continues the renewal review.",
      "rationale_source": "agent",
      "taint_state": "clean"
    }
  },
  "requested_access": {
    "constraints": {
      "requested_child_actor": {
        "id": "agent_report_compiler_v1",
        "issuer": "https://agents.example.com",
        "type": "ai_agent"
      },
      "allowed_tools": ["reports.generate", "crm.read_account"],
      "max_invocations": 10,
      "max_duration_seconds": 86400,
      "reusable_within_mission": false
    }
  },
  "client": {
    "actor": {
      "id": "agent_renewal_assistant_v3",
      "issuer": "https://agents.example.com",
      "type": "ai_agent"
    },
    "source": {
      "session_id": "sess_01K2Q3Z9C1",
      "event_type": "delegation.requested",
      "tool_call_id": "call_01K2Q7AB2D"
    }
  },
  "denial": {
    "evaluation_id": "eval_01K2Q7CC9F",
    "evaluated_at": "2026-05-15T22:00:00Z",
    "reason": "mission_delegation_required",
    "template": "mission_delegation",
    "binding_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6InBkcC0xIn0..."
  }
}
```

If approved, the result derives a child Mission whose authority attenuates the parent:

```json
{
  "task": {
    "id": "arq_01K2Q7K4ZP",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01K2Q7M0QE",
      "approved_at": "2026-05-15T22:05:00Z",
      "approved_until": "2026-05-16T22:05:00Z",
      "state": {
        "mission": {
          "ref": "mr_01K2Q7N3CHILD0001",
          "parent_ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
          "prior_constraints_hash": "sha256-def456",
          "constraints_hash": "sha256-child001",
          "approved_scope": {
            "tools": ["reports.generate", "crm.read_account"],
            "actor": {
              "id": "agent_report_compiler_v1",
              "issuer": "https://agents.example.com",
              "type": "ai_agent"
            }
          },
          "constraints": {
            "max_invocations": 10,
            "max_duration_seconds": 86400,
            "reusable_within_mission": false
          },
          "approval_evidence": {
            "review_id": "rev_01K2Q7HP5R",
            "review_hash": "sha256-review-deleg001"
          }
        }
      }
    }
  }
}
```

The PEP re-evaluates with the `approval` object at `context.approval`.  The PDP verifies the parent Mission state, that the child Mission reference is a valid derivation of the parent, that the child actor matches the approved scope, and that the child authority is an attenuation of the parent before returning `decision: true`.  The sub-agent then executes under the child Mission reference.

## Example: Mission Resumption After a Pause

A Mission is suspended overnight.  When a fresh agent process resumes execution under Mission reference `mr_01JR9S4YDY6QF5Q9Q54M0YB4V1`, its next tool call is denied with `template: "mission_resumption"` because the Mission status is `paused` and requires review before execution can resume.  The requested action is already within the existing authority envelope; only resumption is being requested.

The PEP submits:

```json
{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "tool",
    "id": "crm.read_account"
  },
  "action": {
    "name": "invoke"
  },
  "context": {
    "mission": {
      "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
      "constraints_hash": "sha256-def456",
      "status": "paused",
      "run_id": "run_01K2Q9F1AA"
    },
    "intent": {
      "agent_rationale": "Resuming the renewal review after an overnight pause; the next step reads account data already within Mission scope.",
      "rationale_source": "orchestrator",
      "taint_state": "clean"
    }
  },
  "client": {
    "actor": {
      "id": "agent_renewal_assistant_v3",
      "issuer": "https://agents.example.com",
      "type": "ai_agent"
    },
    "source": {
      "session_id": "sess_01K2Q9D8E2",
      "event_type": "mission.paused"
    }
  },
  "denial": {
    "evaluation_id": "eval_01K2Q9GG3H",
    "evaluated_at": "2026-05-16T13:00:00Z",
    "reason": "mission_paused",
    "template": "mission_resumption",
    "binding_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6InBkcC0xIn0..."
  }
}
```

If approved, resumption returns the Mission to active without changing its authority version:

```json
{
  "task": {
    "id": "arq_01K2Q9K7MN",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01K2Q9M2RT",
      "approved_at": "2026-05-16T13:08:00Z",
      "approved_until": "2026-05-16T21:08:00Z",
      "state": {
        "mission": {
          "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
          "prior_constraints_hash": "sha256-def456",
          "constraints_hash": "sha256-def456",
          "approved_scope": {
            "tools": ["crm.read_account"]
          },
          "constraints": {
            "max_duration_seconds": 28800
          },
          "approval_evidence": {
            "review_id": "rev_01K2Q9HJ8P",
            "review_hash": "sha256-review-resume001"
          }
        }
      }
    }
  }
}
```

Resumption does not change the Mission authority version; `prior_constraints_hash` and `constraints_hash` are identical.  During re-evaluation the PDP confirms the Mission status has returned to active and that the requested action remains within the existing authority before returning `decision: true`.  A PDP MUST still deny if the Mission has been revoked rather than merely paused (see Security Considerations).

## Example: Mission Budget Increase

An agent has exhausted the search budget approved for Mission reference `mr_01JR9S4YDY6QF5Q9Q54M0YB4V1` (authority version `sha256-def456`).  The next search would exceed the approved invocation count, so the PDP denies with `template: "mission_budget_increase"`.  The agent requests additional allowance through `requested_access.constraints`.

The PEP submits:

```json
{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "tool",
    "id": "crm.search_accounts"
  },
  "action": {
    "name": "invoke"
  },
  "context": {
    "mission": {
      "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
      "constraints_hash": "sha256-def456",
      "status": "active",
      "run_id": "run_01K2QB2K9C"
    },
    "intent": {
      "agent_rationale": "The renewal cohort is larger than estimated; the 25-invocation search budget is exhausted and roughly 25 more are needed to finish.",
      "rationale_source": "agent",
      "taint_state": "external_content_seen"
    }
  },
  "requested_access": {
    "constraints": {
      "max_invocations": 50,
      "reusable_within_mission": true
    }
  },
  "client": {
    "actor": {
      "id": "agent_renewal_assistant_v3",
      "issuer": "https://agents.example.com",
      "type": "ai_agent"
    },
    "source": {
      "session_id": "sess_01K2QB1A7F",
      "event_type": "budget.exceeded",
      "tool_call_id": "call_01K2QB3D5G"
    }
  },
  "denial": {
    "evaluation_id": "eval_01K2QB4E2J",
    "evaluated_at": "2026-05-16T15:30:00Z",
    "reason": "mission_budget_exceeded",
    "template": "mission_budget_increase",
    "binding_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6InBkcC0xIn0..."
  }
}
```

If approved, the increased budget produces a new Mission authority version:

```json
{
  "task": {
    "id": "arq_01K2QBK6QW",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01K2QBM9XV",
      "approved_at": "2026-05-16T15:34:00Z",
      "approved_until": "2026-05-22T21:04:00Z",
      "state": {
        "mission": {
          "ref": "mr_01JR9S4YDY6QF5Q9Q54M0YB4V1",
          "prior_constraints_hash": "sha256-def456",
          "constraints_hash": "sha256-ghi789",
          "approved_scope": {
            "tools": ["crm.search_accounts", "crm.read_account"]
          },
          "constraints": {
            "max_invocations": 50,
            "max_duration_seconds": 604800
          },
          "approval_evidence": {
            "review_id": "rev_01K2QBHN4S",
            "review_hash": "sha256-review-budget001"
          }
        }
      }
    }
  }
}
```

A budget increase changes enforceable authority, so it produces a new `constraints_hash`.  During re-evaluation the PDP verifies the successor version and enforces the increased budget.  The PEP MUST refresh its `constraints_hash` to the new value so that any subsequent denial carries the current version.

## Relationship to AAuth Mission Work

The AAuth Mission layer introduces Mission as a first-class protocol concept: mission proposal and approval at the Person Server, an `AAuth-Mission` header, mission-aware token choreography, and governance endpoints.  This profile is complementary, not duplicative: AAuth carries Mission context through an agent-native protocol and token choreography; this profile maps Mission governance onto AuthZEN requestable denials and re-evaluation.  In an AAuth deployment, an AAuth Person Server or related Mission service can act as the Access Request Service behind this profile.  The `AAuth-Mission` header's approver and mission hash can be normalized into `context.mission.ref` or a downstream registered extension; this profile does not require exposing an internal Mission identifier.  In an OAuth, MCP, or other non-AAuth deployment, a Mission Authority Service, authorization server, gateway, or PDP can play that role.

This profile deliberately operates above the harder semantic-interoperability problem.  A Mission reference, hash, or header identifies the approved task; this profile uses that identity to attach governance to runtime denials.  Shared meaning for resource classes, action classes, attenuation, and lifecycle vocabulary remains deployment-local until cross-deployment implementation experience accumulates.

## Relationship to the Base Profile

This proposal is a strict extension of the base profile.  It populates the existing extension points for:

- `context` in an Access Request submission.
- `requested_access` in an Access Request submission.
- `context.access_request.template` values.
- `approval.state` in Re-evaluation Mode.
- Capability advertisement through PDP metadata.

It introduces no new endpoints and no new base completion mode.  A base-profile-only PEP remains conformant when it omits these extensions, but it will not receive the Mission-specific interoperability benefits.

Deployments using a token-issuance completion mode (such as the OAuth Transaction Authorization Challenge profile) carry Mission state in the issued token's claims rather than in `result.approval.state`; the mapping of `context.mission` and `approval.state.mission` to token claims is defined by the token-issuance profile, not by this profile.  This profile remains usable as the Mission-state vocabulary in either completion model.

## Open Questions

- **Lifecycle vocabulary registration.**  This profile uses `active`, `paused`, `suspended`, `pending_approval`, and `revoked` informally for `context.mission.status`.  Should these be registered as interoperable values for cross-deployment use?
- **Delegation proof.**  Should `mission_delegation` define a mandatory narrowing proof artifact for child Missions, or remain deployment-defined?
- **AAuth header decomposition.**  The Transport Bindings section maps `AAuth-Mission` directly to `context.mission.ref` + `context.mission.constraints_hash`.  Should a structured `AAuth-Mission` value with separately addressable approver and mission-hash components be normalized into a registered extension rather than collapsed into `ref`?

## References

- [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md): the base profile this proposal profiles.
- [The Mission Shaping Problem](https://notes.karlmcguinness.com/notes/the-mission-shaping-problem): defines Mission shaping as the conversion of approved intent into governable authority.
- [Mission Shaping Is Not Enough](https://notes.karlmcguinness.com/notes/mission-shaping-is-not-enough): describes scope creep, delegation, headless Missions, resumption, containment, runtime trust, and budgets.
- [Mission-Bound OAuth Architecture](https://notes.karlmcguinness.com/notes/mission-bound-oauth-architecture): defines Mission, Mission Proposal, Approved Mission, Mission Authority Model, Mission lifecycle, and projected Mission artifacts.
- [AAuth Now Has a Mission Layer](https://notes.karlmcguinness.com/notes/aauth-now-has-a-mission-layer): analyzes AAuth's Mission layer and the distinction between mission correlation and portable mission semantics.
- [Mapping Mission Architecture to AAuth](https://notes.karlmcguinness.com/notes/mapping-mission-architecture-to-aauth): maps Mission architecture onto AAuth concepts.
- [Implementing Mission Architecture End-to-End](https://notes.karlmcguinness.com/notes/implementing-mission-architecture-end-to-end): defines concrete MAS artifacts including Mission proposal, review packet, governance record, approval object, runtime signal, token projection, `mission_id`, and `constraints_hash`.
- RFC 8693 OAuth 2.0 Token Exchange: delegation plumbing relevant to child Mission derivation.
- RFC 9396 OAuth 2.0 Rich Authorization Requests: structured authorization details related to Mission authority requests.
- Model Context Protocol: tool mediation surface commonly used by agentic PEPs.
