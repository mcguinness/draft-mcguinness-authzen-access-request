# AAuth Agent Egress Governance with Rich Resource Requests

**Status:** Proposal.  Describes how an AAuth deployment governs agent egress: the egress proxy is a Rich Resource Request (R3) enforcement point, the agent carries a Mission-bound auth token expressing operation-level grants, "request access" is a Mission broadening that re-projects the token, and the RFC 8908 Captive Portal API is reused as the block-and-request signaling layer.  Not yet a draft.

**Builds on:** the AAuth protocol (draft-hardt-aauth-protocol) and AAuth Rich Resource Requests / R3 (draft-hardt-aauth-r3, dated 2026-04-13, https://dickhardt.github.io/AAuth/draft-hardt-aauth-r3.html); RFC 8908 (Captive Portal API), RFC 8910 (captive-portal discovery), and RFC 6585 (HTTP 511); and the Mission Architecture notes at https://notes.karlmcguinness.com/notes.

## Motivation

An agent runtime sends outbound traffic through an egress proxy that enforces which destinations, and which operations on them, the agent may reach.  Egress control is the agent's primary data-exfiltration boundary, and the allowlist cannot be fully enumerated up front: a long-running agent discovers, mid-task, that it needs an operation on a destination that was not provisioned when it started.  When that happens the agent needs a machine-actionable path to request the additional authority, obtain it under human governance, and resume, without a human present at the moment of the block.

AAuth already supplies the pieces.  A Mission is the approved task and its authority envelope.  Rich Resource Requests (R3) let a resource describe its operations in a precise vocabulary (OpenAPI, MCP, gRPC, and others), and let the Authorization Server evaluate a Mission against that vocabulary and issue a token carrying the granted operations.  This proposal binds R3 to the egress boundary, uses Mission broadening as the "request access" step, and reuses the RFC 8908 Captive Portal API as the standard signal for "you are blocked, here is where to request authority."

## Scope

This proposal defines:

- An egress-governance model in which the egress proxy enforces a Mission-bound auth token's R3 grants locally.
- Use of R3 as the egress authority vocabulary: operation-level grants (`r3_granted`) and per-call validated grants (`r3_conditional`).
- The blocked-operation, broaden, resume protocol flow expressed in AAuth Mission and R3 terms.
- Reuse of the RFC 8908 Captive Portal API as the block-and-request signaling layer.
- How the egress proxy fronts non-AAuth destinations by deriving an R3 document from the destination's API description.

It does **not** define:

- The AAuth protocol or the R3 document and claim formats; those belong to draft-hardt-aauth-protocol and draft-hardt-aauth-r3.  This proposal uses their named constructs and marks anything format-specific as deferred.
- The Mission shaping or compilation algorithm, or a portable Mission Authority Model.

## Roles and components

| Component | Role in egress governance |
|---|---|
| Mission | The approved task and its authority envelope, versioned by `constraints_hash`. |
| Person Server (PS) | Where a human proposes and approves a Mission, and approves later broadening, using the resource's R3 `display` text.  Reached as the captive+json `user-portal-url`. |
| Authorization Server (AS) | Fetches and verifies a destination's R3 document, evaluates the Mission against the requested operations, and issues Mission-bound auth tokens carrying `r3_granted` and `r3_conditional`.  Performs per-call evaluation for conditional operations. |
| Egress proxy | The R3 enforcement point at the agent's outbound boundary, and the Captive Portal API server.  Matches each outbound call against the token's grants and signals blocks via captive+json. |
| Destination resource | Publishes an R3 document describing its operations.  For non-AAuth destinations, the egress proxy or its AS derives the R3 document from the destination's API description. |
| Mission-bound auth token (`AAuth-Mission`) | The credential the agent carries.  It carries `r3_granted`, `r3_conditional`, `r3_uri`, and `r3_s256`. |
| Agent | Sends an authorization request with `r3_operations`, carries the opaque `r3_s256` hash, makes calls, and handles `AAuth-Requirement` challenges and captive signals. |

## Egress authority as Rich Resource Requests

R3 supplies the vocabulary for both the request and the grant, so the egress envelope is operation-level rather than host-level.

- A destination publishes an **R3 document**: a `vocabulary` URN (for example `urn:aauth:vocabulary:openapi`), an `operations` array in that vocabulary (OpenAPI `operationId`, MCP tool name, gRPC method, and so on), and a `display` section (`summary`, `implications`, `data_accessed`, `irreversible`).  It is discovered through the destination's `/.well-known/aauth-resource.json` (`r3_vocabularies`).
- The **AS**, not the agent, fetches the document (AS-only access, authenticated with an HTTP Message Signature), verifies its `r3_s256` hash, evaluates the requested operations against the Mission, and MAY narrow the grant.
- The agent's **Mission-bound auth token** carries `r3_granted` (operations served immediately) and `r3_conditional` (operations authorized in principle but requiring per-call validation), plus `r3_uri` and `r3_s256`.  The agent carries only the opaque hash and never reads the document.

The egress allowlist is therefore the destination's own API description, evaluated against the Mission, rather than a hand-maintained host list.

## Where the decision lives

The Mission-bound token is the decision, in two tiers:

- **`r3_granted` is carried.**  The token states the fully authorized operations; the egress proxy serves them immediately with no per-call backend call.  This is the fast path: the authority travels with the agent.
- **`r3_conditional` is evaluated at the call.**  For these operations the proxy challenges back to the AS with the concrete parameters of the actual call (the R3 `AAuth-Requirement` mechanism); the AS makes a parameter-aware decision and issues a per-call token.

A deployment routes parameter-sensitive, high-risk, or irreversible egress into `r3_conditional` (evaluated at enforcement) and leaves routine reads in `r3_granted` (carried), so per-call evaluation is paid only where it is needed.

## Protocol Flow

The flow has three phases: a blocked operation is signaled (RFC 8908), the agent broadens the Mission (AAuth authorization request plus human approval), and the agent resumes with a re-projected token.  The agent-driven flow is primary; a proxy-brokered variant is noted afterward.

~~~ ascii-art
+-------+         +--------------+        +--------+      +---------------+    +-------------+
| Agent |         | Egress Proxy |        |   AS   |      | Person Server |    | Destination |
|       |         | R3 enforce + |        |        |      |   (approve)   |    |      D      |
|       |         | Captive API  |        |        |      |               |    |             |
+---+---+         +------+-------+        +---+----+      +-------+-------+    +------+------+
    |                    |                    |                   |                  |
    | 1 call op on D     |                    |                   |                  |
    |  (Mission token)   |                    |                   |                  |
    |------------------->|                    |                   |                  |
    |                    | op not in grants   |                   |                  |
    | 2 HTTP 511 + captive API URL            |                   |                  |
    |<-------------------|                    |                   |                  |
    | 3 GET captive+json |                    |                   |                  |
    |------------------->|                    |                   |                  |
    | 4 captive:true,    |                    |                   |                  |
    |  access-request    |                    |                   |                  |
    |  (AS, Mission,     |                    |                   |                  |
    |   r3_operations,   |                    |                   |                  |
    |   user-portal-url) |                    |                   |                  |
    |<-------------------|                    |                   |                  |
    | 5 authz request: r3_operations + Mission ref               |                  |
    |---------------------------------------->|                   |                  |
    |                    |       6 fetch + verify R3 doc (r3_s256)|                  |
    |                    |          -------------------------------------------->    |
    |                    |          <--------------------------------------------    |
    |                    |       7 route for approval             |                  |
    |                    |                    |------------------>|                  |
    |                    |       8 human approves (R3 display);    |                  |
    |                    |         Mission authority -> new        |                  |
    |                    |         constraints_hash                |                  |
    |                    |                    |<------------------|                  |
    | 9 re-projected Mission token            |                   |                  |
    |  (r3_granted / r3_conditional)          |                   |                  |
    |<----------------------------------------|                   |                  |
    | 10 retry op on D (new token)            |                   |                  |
    |------------------->|                    |                   |                  |
    |                    | op in r3_granted: forward to D ----------------------->   |
    |                    | 11 response        |                   |                  |
    |                    | <--------------------------------------------------       |
    | 11 response        |                    |                   |                  |
    |<-------------------|                    |                   |                  |
    | 12 (optional) re-query captive+json -> captive:false        |                  |
    |------------------->|                    |                   |                  |
~~~

Step by step:

1. The agent attempts operation `op` on destination D through the proxy, presenting its Mission-bound auth token.
2. The proxy identifies D's grants in the token (by `r3_uri` and `r3_s256`) and finds `op` in neither `r3_granted` nor `r3_conditional`.  It blocks and signals with HTTP 511, referencing a per-denial Captive Portal API URL.
3. The agent issues a GET to that URL with `Accept: application/captive+json`.
4. The proxy returns `captive: true` and an `access-request` member: the AS authorization endpoint, the Mission reference, the requested `r3_operations` (`vocabulary` and `operations`) for D, D's `r3_uri` and `r3_s256`, and the Person Server as `user-portal-url`.
5. The agent issues an AAuth authorization request to the AS with those `r3_operations`, the Mission reference, and the current `constraints_hash`.
6. The AS fetches D's R3 document at `r3_uri` (AS-only, HTTP Message Signature), verifies `r3_s256`, and evaluates the Mission against the requested operations, recording the hash for audit.
7. Where a human must approve, the AS routes the request to the Person Server.
8. The human approves at the Person Server against D's R3 `display` text (`summary`, `implications`, `data_accessed`, `irreversible`).  Approval updates the Mission authority, producing a new `constraints_hash`.
9. The AS issues a refreshed Mission-bound auth token whose `r3_granted` and `r3_conditional` now include the approved operations.
10. The agent retries `op` on D with the new token.
11. The proxy finds `op` in `r3_granted` and forwards the call to D, returning the response.  If `op` is in `r3_conditional`, the per-call sub-flow below runs first.
12. Optionally, the agent re-queries the Captive Portal API and observes `captive: false`, with `seconds-remaining` reflecting the token lifetime.

**Conditional (`r3_conditional`) per-call sub-flow.**  When `op` matches `r3_conditional`, the proxy does not forward immediately.  It returns an `AAuth-Requirement` carrying a resource token with the actual call parameters; the agent round-trips to the AS; the AS evaluates the concrete parameters against the Mission and issues a per-call auth token (or denies); the agent retries and the proxy forwards on success.

**Proxy-brokered variant.**  Instead of returning HTTP 511 or `AAuth-Requirement` to the agent, the proxy MAY drive the broadening or the per-call decision against the AS itself and present a single result to the agent, for runtimes that do not implement the captive or `AAuth-Requirement` handshakes.

**Lapse.**  The grant lapses through the Mission lifecycle: token expiry, `constraints_hash` supersession, or revocation propagation.  Operations flagged `irreversible` in the R3 `display` SHOULD be carried as `r3_conditional` or release-gated rather than `r3_granted`.

## Signaling with the Captive Portal API

The block signal in step 2 reuses the RFC 8908 Captive Portal API rather than a new AAuth-specific channel.  RFC 8908 was designed for "you are restricted, here is the human path to get unblocked," which is precisely the egress-proxy to Person-Server interaction.  The egress proxy acts as a Captive Portal API server, returning `captive: true` with an AAuth broadening descriptor:

| RFC 8908 element | AAuth meaning |
|---|---|
| `captive: true` (per-denial captive API URL) | This call to destination D is blocked; authority must be broadened |
| `access-request` member | AAuth broadening descriptor: the AS authorization endpoint, the Mission reference, the requested `r3_operations` for D, and D's `r3_uri` / `r3_s256` |
| `user-portal-url` | The Person Server human-consent page |
| `seconds-remaining` | The Mission-bound token's lifetime |
| `can-extend-session` | Whether the agent may re-broaden before expiry |
| re-query returning `captive: false` | The re-projected Mission-bound token is accepted and the proxy lifts captivity for D |
| HTTP 511 (RFC 6585) at the proxy hop, or RFC 8910 | Inline or network-layer discovery of the captive API URL |

Three caveats apply:

- **Per-destination strain.**  RFC 8908's `captive` is a network-wide boolean; egress is per-destination.  The proxy hands the agent a captive API URL that is distinct per denial, so each captive+json describes one blocked egress.
- **Granularity mismatch.**  RFC 8908 signals at destination level, but R3 grants at operation level.  `captive: true` therefore means "this call to D is blocked, broaden these `r3_operations`"; it cannot express "captive for op X on D but not op Y."  The per-denial URL scoping absorbs this.
- **Reaching the captive API.**  The agent cannot always reach the captive API over the connection being blocked; HTTP 511 at the proxy hop, RFC 8910, or a runtime-configured URL bootstraps it.

## Fronting non-AAuth destinations

Most egress destinations are third parties that do not speak AAuth.  The egress boundary still works:

- The egress proxy, or its AS, **derives an R3 document** from the destination's published API description (its OpenAPI document, MCP tool list, or gRPC service definition), assigns it an `r3_uri` and `r3_s256`, and enforces the agent's Mission-bound token against it.  The destination is unchanged and unaware.  This gives operation-level control over third-party APIs.
- For an **AAuth-native destination**, R3 is enforced end to end at the resource; the egress proxy is a second gate or a pass-through.

In the non-AAuth case the proxy also stands in for the resource in the `r3_conditional` handshake: it issues the `AAuth-Requirement` on the destination's behalf and relays the concrete parameters to the AS.

## Bounded egress authority

Operation-level scoping is inherent (which operations, in which vocabulary).  Finer bounds come from the two tiers and the Mission: `r3_conditional` carries parameter-level limits the AS evaluates per call (monetary caps, record sets, rate); duration and call counts come from token lifetime and Mission constraints; irreversible effects (R3 `display.irreversible`) map to `r3_conditional` or a release gate.

## Security considerations

- **The token is a bearer of authority.**  Mission-bound egress tokens SHOULD be sender-constrained (DPoP-style proof-of-possession or mutual TLS) and short-lived, and bound to the agent identity, so a stolen token cannot be replayed by another actor.
- **The agent holds only an opaque hash.**  Because the agent carries `r3_s256` and never the R3 document, prompt injection cannot manipulate the authorization semantics through the agent; the AS does the semantic work server-side.
- **R3 documents are AS-only.**  The resource serves the R3 document only to a request signed by its AS (HTTP Message Signature), so a compromised agent or proxy cannot substitute a forged operation set.
- **Per-call validation catches in-envelope misuse.**  Routing parameter-sensitive and irreversible operations to `r3_conditional` means the AS sees the concrete call at enforcement time, narrowing the window in which a drifting or hijacked agent can misuse an operation it is nominally allowed.
- **Curated consent.**  The Person Server approves against the resource-authored R3 `display` text, not agent-supplied prose; agent rationale in a broadening request is untrusted and MUST be sanitized before display.  The captive+json `user-portal-url` and other URLs are untrusted input to an autonomous agent and MUST NOT be auto-navigated or fed to a model as instructions.
- **Audit binding.**  `r3_s256` ties a token to the exact operation semantics that were approved, giving a permanent record across re-projections.
- **Confused-deputy egress.**  The proxy MUST enforce the token's grants for the specific destination (keyed by `r3_uri` and `r3_s256`), never operations the agent merely asserts.
- **Stale authority window.**  Carried `r3_granted` is only as fresh as the token lifetime; deployments bound the lifetime to their revocation tolerance and route higher-risk egress into `r3_conditional`.

## Open Questions

- **R3 derivation for non-AAuth destinations.**  Should the proxy's OpenAPI/MCP/gRPC to R3 derivation be standardized (so two proxies derive the same `operations` and `r3_s256` for the same destination), or left to deployment?
- **Who issues `AAuth-Requirement` at the egress hop.**  For a non-AAuth destination the proxy stands in for the resource in the conditional handshake; this stand-in role and its token formats need definition.
- **Broadening descriptor vocabulary.**  Should the captive+json `access-request` member and the AAuth authorization request share one registered descriptor (Mission reference plus `r3_operations`) so any proxy and AS interoperate?
- **Sender-constraining.**  Should proof-of-possession (DPoP or mutual TLS) be mandatory for Mission-bound egress tokens rather than recommended?
- **Raw and CONNECT egress.**  R3 vocabularies are API-operation oriented.  For opaque TLS tunnels or non-HTTP egress where the proxy cannot see operations, the grant degrades to host or connection level; how that coarser grant and its captive signal are expressed is open.

## References

- AAuth protocol: draft-hardt-aauth-protocol, https://datatracker.ietf.org/doc/draft-hardt-aauth-protocol/.
- AAuth Rich Resource Requests (R3): draft-hardt-aauth-r3, https://dickhardt.github.io/AAuth/draft-hardt-aauth-r3.html (dated 2026-04-13).
- RFC 8908 Captive Portal API; RFC 8910 (captive-portal discovery in DHCP and Router Advertisements); RFC 6585 (HTTP 511 Network Authentication Required): the signaling layer reused here.
- Mission-Bound OAuth Architecture, https://notes.karlmcguinness.com/notes/mission-bound-oauth-architecture; AAuth Now Has a Mission Layer, https://notes.karlmcguinness.com/notes/aauth-now-has-a-mission-layer; Implementing Mission Architecture End-to-End, https://notes.karlmcguinness.com/notes/implementing-mission-architecture-end-to-end.
