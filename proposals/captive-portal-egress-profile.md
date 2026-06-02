# Captive Portal Egress Profile of the AuthZEN Access Request Profile

**Status:** Proposal.  Captures what a profile would look like that lets an agent request egress access through the RFC 8908 Captive Portal API, backed by the AuthZEN Access Request profile.  Not yet a draft.

**Profiles:** [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md) (referred to below as "the base profile").

**Builds on:** Captive Portal API, RFC 8908; Captive-Portal Identification in DHCP/RAs, RFC 8910; HTTP 511 Network Authentication Required, RFC 6585.

## Motivation

An agent runtime commonly sends outbound traffic through an egress proxy: a forward proxy that enforces which destinations the agent may reach.  Egress control is the primary data-exfiltration boundary for an agent, and the allowlist of destinations cannot be fully enumerated in advance: a long-running agent discovers, mid-task, that it needs to call an API, fetch a document, or post to a service that was not provisioned when the agent started.

When the proxy blocks such a destination, the agent today sees an opaque connection failure with no machine-actionable path to proceed.  The base profile already defines the path from a denied-but-requestable authorization decision to an approval and a re-evaluation.  What is missing is a standard way for the proxy to *signal* the block to the agent and hand it the request path, using a mechanism an agent (or a thin client in its runtime) can act on without a human at a browser.

RFC 8908 already standardizes exactly that signaling surface for network access: the Captive Portal API.  A client queries a JSON API and learns that it is "captive" (lacks external network access) and where to go to change that.  This profile reuses the Captive Portal API as the egress proxy's signaling surface and binds its "get unblocked" path to the base profile: the proxy is an AuthZEN Policy Enforcement Point that surfaces a requestable denial through `application/captive+json`, and the agent submits an Access Request, awaits approval, and resumes egress after re-evaluation.

## Scope

This profile defines:

- A binding between the RFC 8908 Captive Portal API and the base profile, so an agent can discover that an egress destination is blocked-but-requestable and submit an Access Request for it.
- A Captive Portal API response member (`access-request`) that carries the base profile's requestable denial for the blocked destination.
- A mapping of egress concepts onto AuthZEN Subject, Resource, and Action, and of `seconds-remaining` and re-query behavior onto the base profile's approval expiry and re-evaluation.
- Roles: the egress proxy as AuthZEN PEP plus Captive Portal API server; the agent as the captive client and Access Request requester.

It does **not** define:

- An egress policy language or what destinations are allowed.
- The Captive Portal API discovery mechanism (deferred to RFC 8910 and to the HTTP 511 signal described in the Discovery section).
- A new approval mechanism, completion mode, or media type; approval is the base profile's Re-evaluation Mode and the media type remains `application/captive+json`.

**A known strain.**  RFC 8908's `captive` member models a client's *network-wide* access state, a single boolean over the client's whole situation.  Agent egress control is inherently *per-destination*: one host is allowed, another is blocked, a third needs approval.  A single captive+json response cannot cleanly describe "reachable in general, but destination X needs a request."  This profile resolves the mismatch by scoping the Captive Portal API interaction to a single blocked destination: the proxy hands the agent a Captive Portal API URL that is distinct per denial (extending RFC 8908's guidance that the URL "SHOULD be distinct per client"), so each captive+json response describes one requestable egress denial.  The granularity question is recorded in the Open Questions section.

## Architectural Mapping

| Egress proxy / RFC 8908 | Base profile / AuthZEN |
|---|---|
| Egress proxy enforcing outbound access | AuthZEN PEP, plus Captive Portal API server |
| Blocked egress destination | AuthZEN Resource (the destination) |
| Agent attempting egress | AuthZEN Subject / `client.actor`; the Access Request requester |
| `captive: true` for the blocked destination | AuthZEN `decision: false` that is requestable (`context.access_request` present) |
| `access-request` member of captive+json (this profile) | Base profile Requestable Denial Context surfaced to the agent |
| `user-portal-url` | Human approval fallback (a ticket or review surface) |
| Agent submits the request | Access Request submission to the Access Request Endpoint |
| Approval completes | Task Handle reaches `approved` |
| `captive: false` on re-query | Egress permitted after Re-evaluation Mode |
| `seconds-remaining` | Derived from approval `approved_until` |
| HTTP 511 at the proxy hop | Inline "you are captive, here is the API" signal (the Discovery section) |

The base profile remains the request, approval, and re-evaluation substrate.  This profile supplies the egress-proxy signaling binding.

## The Captive Portal Egress Flow

The agent-driven flow is primary; a proxy-brokered variant is noted at the end.

1.  The agent attempts to reach destination D through the egress proxy.
2.  The proxy, acting as PEP, evaluates the egress with the AuthZEN Access Evaluation API.  The PDP returns `decision: false` with a requestable denial (`context.access_request`), where the AuthZEN Resource is D.
3.  The proxy holds the connection in a captive state for D and signals the agent: it returns HTTP 511 (the Discovery section) referencing a Captive Portal API URL distinct to this denial, or the agent already discovered the API per RFC 8910.
4.  The agent issues an HTTP `GET` to the Captive Portal API URL with `Accept: application/captive+json` and receives a response with `captive: true` and an `access-request` member carrying the requestable denial for D (the `access-request` member registration below).
5.  The agent submits a base-profile Access Request to the Access Request Endpoint, authenticating per the base profile, and receives a Task Handle.  It awaits completion by polling or callback.
6.  On approval, the agent retries egress to D.  The proxy re-evaluates the egress through the AuthZEN Access Evaluation API (Re-evaluation Mode); if the approval applies and current policy permits, the PDP returns `decision: true` and the proxy lifts the captive state for D.
7.  The agent re-queries the Captive Portal API (per Section 5 of RFC 8908) and observes `captive: false` with `seconds-remaining` derived from the approval's `approved_until`.  Egress to D now flows.
8.  When `approved_until` passes, the proxy returns D to the captive state, and the agent may request again.

Because the agent reaches the proxy over the egress connection itself rather than over the AuthZEN API, the approval is typically resolved by the PDP from server-side state at re-evaluation (the base profile's lookup topology): the agent does not need to inject `context.approval` into the proxy's evaluation.  Deployments that prefer the bound-reference topology MAY have the agent present the approval to the proxy out of band (for example, in a request header the proxy maps to `context.approval`); see the Open Questions section.

## Profile Registrations

### Captive Portal API `access-request` member

Registers a new member of the RFC 8908 Captive Portal API response object:

`access-request`:
: Object.  Present when `captive` is `true` because the destination is blocked but requestable.  Carries what the agent needs to submit a base-profile Access Request for the blocked destination.  It has the following members:

  * `subject`: REQUIRED.  The AuthZEN Subject for the denied egress.
  * `resource`: REQUIRED.  The AuthZEN Resource for the blocked destination (for example, `{"type": "egress", "id": "https://api.partner.example"}`).
  * `action`: REQUIRED.  The AuthZEN Action for the egress (for example, `{"name": "connect"}`).
  * `context`: REQUIRED.  The denied evaluation's AuthZEN Context, including the base profile's `access_request` object and its denial-binding material (`evaluation_id` or an integrity-protected `binding_token`) and `reason`.

The agent uses these values to assemble a base-profile Access Request submission (the Access Request Submission section of the base profile): the captive+json `subject`, `resource`, and `action` become the submission's `subject`, `resource`, and `action`; `context.access_request.endpoint` is the submission target; and the denial-binding material is echoed in the submission's `denial` object.

The outer Captive Portal API keys are hyphenated (`access-request`, matching `user-portal-url`), while the embedded base-profile object retains its snake_case members (`access_request`, `binding_token`); this mismatch is deliberate so each layer keeps its native convention.  Clients that do not recognize the `access-request` member ignore it, per normal JSON extension practice.

### Capability URN

Registers a capability URN for advertising this profile in PDP metadata's `capabilities` array:

`urn:openid:authzen:capability:access-request:captive-portal-egress`

Non-normative captive+json example:

```json
{
  "captive": true,
  "user-portal-url": "https://proxy.example.com/portal/req_01J8Z3",
  "access-request": {
    "subject": { "type": "agent", "id": "agent_renewal_assistant_v3" },
    "resource": { "type": "egress", "id": "https://api.partner.example" },
    "action": { "name": "connect" },
    "context": {
      "evaluation_id": "eval_01K2Q4DP3K",
      "reason": "egress_not_allowed",
      "access_request": {
        "endpoint": "https://pdp.example.com/access/v1/requests",
        "template": "egress_destination",
        "expires_at": "2026-05-31T20:25:00Z",
        "binding_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6InBkcC0xIn0...",
        "display": {
          "title": "Egress to api.partner.example is blocked",
          "description": "Approval is required before this agent may call api.partner.example."
        }
      }
    }
  }
}
```

## Discovery

An agent discovers the Captive Portal API URL for a blocked destination by either mechanism:

* **Network-layer (RFC 8910).**  When the agent is a managed network client, the Captive Portal API URL is provisioned through the DHCP or Router Advertisement options defined by RFC 8910.  This is unchanged from RFC 8908.
* **HTTP 511 at the proxy hop.**  When the egress proxy is an HTTP proxy, it MAY signal a blocked attempt inline by returning `511 Network Authentication Required` (RFC 6585) on the proxy-visible request (the `CONNECT` response or a cleartext request), with a `Link` header referencing the per-denial Captive Portal API URL.

The 511 signal works only at the proxy hop: for an HTTPS destination tunneled through `CONNECT`, the proxy cannot inject 511 inside the TLS stream without acting as a TLS man-in-the-middle.  Deployments that do not terminate TLS therefore rely on the `CONNECT` response, on RFC 8910 provisioning, or on the agent runtime being configured with the API URL directly.  See the Open Questions section.

## Time and Re-query

`seconds-remaining` carries how long egress to the destination remains permitted, computed by the proxy from the approval's `approved_until` returned during re-evaluation.  When `seconds-remaining` reaches zero the proxy returns the destination to the captive state, consistent with RFC 8908.  `can-extend-session` MAY be set when the deployment allows the agent to request again before expiry.

The re-query behavior in step 7 is exactly the client behavior Section 5 of RFC 8908 describes ("the client SHOULD query the API server again to verify that it is no longer captive"); here a non-captive result corresponds to a successful base-profile re-evaluation.

## Security Considerations

This profile inherits the base profile's security considerations.  Because egress control is an agent's data-exfiltration boundary, the following agent-specific risks are foregrounded:

| Risk | Mitigation |
|---|---|
| Untrusted URLs in captive+json | `user-portal-url` and `venue-info-url` are attacker-influenceable input to an LLM-driven agent.  An agent MUST NOT auto-navigate, fetch, or feed these URLs to a model as instructions; treat them as untrusted, the same way the base profile and companion profiles treat agent-supplied rationale.  The machine path is the `access-request` member, not these URLs. |
| Confused-deputy egress | The AuthZEN Resource MUST be bound to the exact destination, so an approval for egress to one destination cannot be replayed to authorize another.  The base profile's denial-binding and re-evaluation rules apply unchanged. |
| Captive+json disclosure | The per-denial Captive Portal API URL is distinct per denial and the response is per-client; servers set `Cache-Control: private` or stricter per RFC 8908, so one agent cannot read another's requestable denial. |
| Request not a grant | `captive: true` with an `access-request` member is a requestable denial, not access.  The proxy MUST keep the destination captive until a base-profile re-evaluation returns `decision: true`. |
| Approval expiry bypass | The proxy MUST stop permitting egress at `approved_until` and re-evaluate, never extending access on `seconds-remaining` alone. |

The Captive Portal API and any portal URL MUST be served over TLS, as required by RFC 8908.  The Access Request submission authenticates per the base profile; the captive+json `access-request` member conveys no bearer authority.

## Conformance Checklist

A deployment of this profile:

- MUST publish the base profile's PDP metadata (`access_request_endpoint`, and `jwks_uri` when issuing signed `binding_token` values).
- SHOULD advertise `urn:openid:authzen:capability:access-request:captive-portal-egress` in `capabilities`.
- MUST serve `application/captive+json` per RFC 8908, with `captive: true` for a blocked-but-requestable destination and an `access-request` member carrying the requestable denial.
- MUST bind the AuthZEN Resource to the exact egress destination.
- MUST keep the destination captive until a base-profile re-evaluation returns `decision: true`, and MUST return to captive at `approved_until`.
- MUST use the base profile's Re-evaluation Mode (`result.mode: reevaluate`).
- An agent MUST NOT treat `user-portal-url` or `venue-info-url` as trusted instructions or auto-navigate them.

A base-profile-only deployment that does not expose a Captive Portal API remains conformant to the base profile.

## Open Questions

- **Captivity granularity.**  This profile scopes the Captive Portal API interaction to one destination via a per-denial API URL.  Is a per-destination URL the right model, or should captive+json carry a list of pending egress denials, or should `captive` be redefined as "captive with respect to this destination"?
- **HTTPS egress without TLS termination.**  Without MITM, the proxy cannot inject 511 inside a tunneled TLS stream.  Should this profile recommend `CONNECT`-response signaling, an agent-runtime-configured API URL, or a network-layer (RFC 8910) provisioning as the canonical path for opaque HTTPS egress?
- **Carrying the approval to the proxy.**  Server-side approval resolution is the default; should this profile define a standard header by which an agent presents `context.approval` to the proxy for the bound-reference topology?
- **Subject and action vocabulary.**  Should `resource.type: "egress"` and `action.name: "connect"` be registered as interoperable values, or left deployment-defined?

## Relationship to the Base Profile

This proposal is a strict extension: it adds one Captive Portal API response member and one capability URN, and maps egress concepts onto existing AuthZEN Subject, Resource, and Action and the base profile's request, approval, and re-evaluation flow.  It introduces no new approval mechanism, no new completion mode, and no new media type.  A base-profile Access Request submitted from a captive+json `access-request` member is an ordinary submission; the Access Request Service need not know it originated from a Captive Portal API interaction.

## References

- [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md): the base profile this proposal profiles.
- RFC 8908, Captive Portal API: the signaling surface this profile reuses for egress.
- RFC 8910, Captive-Portal Identification in DHCP and Router Advertisements (RAs): network-layer discovery of the Captive Portal API URL.
- RFC 6585, Additional HTTP Status Codes: the 511 Network Authentication Required status used for inline proxy-hop signaling.
