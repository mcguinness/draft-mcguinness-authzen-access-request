# AARM Profile of the AuthZEN Access Request Profile

**Status:** Proposal.  Captures what a profile would look like that binds AARM's Approval Service component to the AuthZEN Access Request profile.  Not yet a draft.

**Profiles:** [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md) (referred to below as "the base profile").

**Aligns with:** AARM Approval Service component, https://aarm.dev/components/approval-service.

## Motivation

AARM (Action Authorization and Risk Management) defines an Approval Service component for human-in-the-loop and policy-engine-driven approval of high-risk or unresolved actions.  Its operations and data shapes overlap substantially with the base profile's Access Request Service, but AARM's vocabulary (`STEP_UP`, `DEFER escalation`, risk levels, policy confidence, semantic distance, ActionIdentity chains) is opinionated in ways the base profile deliberately leaves generic.

This profile registers the AARM-specific vocabulary and conventions at the base profile's extension points, so an AARM-conformant deployment can be exposed as an Access Request Service per the base profile without changing wire shape.

## Scope

This profile defines:

- A request-reason vocabulary that names AARM's STEP_UP and DEFER classes on the wire.
- `context` augmentations that carry AARM's risk-level, policy-confidence, and semantic-distance fields.
- A `result.approval` augmentation that carries AARM's identity-verification flag.
- Task-lifecycle expectations that match AARM's DENY-on-timeout default.
- An identity convention that maps AARM's ActionIdentity chain to the base profile's `subject` and `client.actor` shape.

It does **not** define:

- An approver-facing API.  AARM's `approve(request_id, approver)` and `deny(request_id, approver, reason)` are internal to the Access Request Service and out of scope for both the base profile and this profile.
- A notification transport.  AARM's Slack, email, and webhook channels are approver-facing and orthogonal to the base profile's PEP-facing callback.
- A new completion mode.  AARM's `ApprovalResult` maps onto the base profile's Re-evaluation Mode; no `mode: token` or similar binding is introduced here.

## Architectural Mapping

| AARM | Base profile |
|---|---|
| Policy Engine `STEP_UP` / `DEFER escalation` decision | AuthZEN `decision: false` with `context.access_request` present |
| Approval Service | Access Request Service |
| `request()` (blocks until decision) | `POST` to Access Request Endpoint; synchronous completion when fast, polling or callback otherwise |
| `approve()` / `deny()` | Internal to the Access Request Service; out of scope |
| `ApprovalResult.grant=true` | `task.status: approved` with `result.mode: reevaluate` |
| `ApprovalResult.grant=false` | `task.status: denied` |
| DENY-on-timeout default | `task.status: expired` after `task.expires_at`; PEP MUST treat as not approved |
| Identity re-verification at execution | Re-evaluation Mode: the PDP re-checks identity, risk, and policy at enforcement time |

AARM's `request()` is conceptually blocking; this profile's wire is async.  AARM's blocking semantic is a library convention over the async surface (synchronous completion when the Access Request Service resolves the request within the request lifetime, polling or callback otherwise).  Because human approval can take minutes to days, AARM implementations already wrap the async surface; this profile makes that wrap explicit.

## Profile Registrations

### Request-reason vocabulary

Registers the following well-known values for `context.access_request.reason` in the base profile's Requestable Denial Context:

| Value | Meaning |
|---|---|
| `step_up` | Request triggered by AARM's STEP_UP decision class (direct high-risk action). |
| `defer_escalation` | Request triggered by AARM's DEFER escalation (unresolved policy deferral requiring human judgment). |

These map to AARM's `ApprovalRequest.source` field.

### Context augmentations

Registers the following members at the `context` extension point in an Access Request submission:

| Member | Type | Description |
|---|---|---|
| `risk_level` | String | AARM risk classification.  One of `low`, `medium`, `high`, `critical`. |
| `policy_confidence` | Number | AARM policy confidence score, in the range `0.0`–`1.0`. |
| `semantic_distance` | Number | Semantic distance from the original user intent, when computed by the upstream system. |

Lowercase is used here for consistency with the base profile's existing `risk_level` Catalog Item member.  AARM-conformant senders MAY accept either case; this profile uses lowercase on the wire.

### `result.approval` augmentation

Registers the following member on `result.approval` for the Re-evaluation Mode result:

| Member | Type | Description |
|---|---|---|
| `identity_verified` | Boolean | When `true`, the approver's identity (or the actor's identity, where the deployment requires identity re-verification at decision time) was re-validated at the moment of decision.  Maps to AARM's `ApprovalResult.identity_verified`. |

### Capability URN

Registers a capability URN for advertising AARM conformance in PDP metadata's `capabilities` array:

`urn:openid:authzen:capability:access-request:aarm`

## Submission Identity Convention

PEPs MUST convey AARM's ActionIdentity chain using the conventions defined in the base profile's Subjects, Principals, and Actors subsection.  Specifically:

- The AuthZEN `subject` carries the human principal (or the organizational principal, when no human is in the chain).
- `client.actor` carries the immediate actor (typically the agent or service) with `id`, `issuer`, and `type` per the base profile.
- Delegation chains (human → service → agent → role scope) are conveyed through nested `act` claims on the subject, following draft-mcguinness-oauth-actor-profile.

The canonical actor identifier in this profile is the (`act.iss`, `act.sub`) pair.

## Task Lifecycle Expectations

- `task.expires_at` SHOULD default to 3600 seconds from submission unless overridden by deployment policy.  This matches AARM's default timeout.
- An expired Access Request MUST be treated as a denial of the underlying action.  PEPs MUST NOT enforce the action when `task.status` is `expired`, consistent with AARM's DENY-on-timeout default.
- Cancellation by the requester or administrator follows the base profile's Cancellation section; AARM does not require additional cancellation semantics.

## Conformance Checklist

An AARM-conformant deployment of this profile:

- MUST publish `access_request_endpoint` and `jwks_uri` in PDP metadata per the base profile.
- MUST advertise `urn:openid:authzen:capability:access-request:aarm` in `capabilities`.
- MUST use `result.mode: reevaluate` (the only base completion mode).
- MUST use `step_up` or `defer_escalation` as `context.access_request.reason` for STEP_UP and DEFER classes respectively.
- MAY include `risk_level`, `policy_confidence`, and `semantic_distance` in `context` when these values are produced by the upstream Policy Engine.
- MUST re-validate identity at re-evaluation when AARM policy requires identity verification at execution; the PDP's re-evaluation is the authoritative point for that check.
- MUST treat `task.status: expired` as denial of the underlying action.

A base-profile-only PEP that submits an Access Request without AARM-specific extensions remains conformant to the base profile.  AARM-conformant Access Request Services validate AARM extensions when present and SHOULD NOT reject submissions that omit them, unless the deployment policy requires the extension for a particular action class.

## Open Questions

- **Risk-level placement.**  Belongs in `context`, `requested_access`, or `context.access_request.reason`?  This proposal puts it in `context` to match how AARM models it (a property of the action under evaluation, not of the requester's ask).  Alternative: put it in `context.access_request.reason` as a structured reason code.
- **Configured approvers.**  AARM's `ApprovalRequest` includes a configured-approvers list.  The base profile's privacy considerations argue against exposing approver identities to the PEP.  This proposal does not register a field for configured approvers; the Access Request Service handles approver selection internally.  Confirm this is acceptable for AARM use cases.
- **SessionContext.**  AARM's SessionContext (accumulated session history) is rich and deployment-specific.  This proposal suggests carrying it through `client.source.external_url` or `client.source.conversation_id` for audit, rather than embedding the full session.  Confirm whether AARM needs a structured `session_context` augmentation.
- **`policy_confidence` semantics.**  Whether the score is advisory (informational to the approver or re-evaluator) or normative input to the Access Request Service workflow.  This proposal treats it as advisory; AARM may require otherwise.
- **Identity-verification scope.**  `result.approval.identity_verified` could mean (a) the approver's identity was verified, or (b) the actor's identity was verified, or (c) both.  AARM's documentation suggests (b) primarily.  Profile text should resolve.

## Relationship to the Base Profile

This proposal is a strict extension: it populates extension points the base profile defined, registers names in the AuthZEN Access Request Member Names registry, and adds one capability URN.  It introduces no new wire shapes, no new completion modes, and no new endpoints.

A future companion profile could define `mode: token` for AARM-style deployments that want token issuance bound to the approval (rather than re-evaluation).  That work is out of scope for this proposal.

## References

- [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md) — the base profile this proposal profiles.
- AARM Approval Service component, https://aarm.dev/components/approval-service.
- draft-mcguinness-oauth-actor-profile — `act` claim conventions referenced by both this proposal and the base profile.
