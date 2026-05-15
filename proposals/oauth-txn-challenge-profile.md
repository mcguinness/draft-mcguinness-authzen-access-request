# OAuth Transaction Authorization Challenge Profile of the AuthZEN Access Request Profile

**Status:** Proposal.  Captures what a profile would look like that bridges the OAuth Transaction Authorization Challenge draft and the AuthZEN Access Request profile.  Not yet a draft.

**Profiles:** [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md) (referred to below as "the base profile").

**Aligns with:** OAuth Transaction Authorization Challenge, https://yaroslavros.github.io/oauth-txn-challenge/draft-rosomakho-oauth-txn-challenge.html ("OAuth Txn Challenge").

## Motivation

OAuth Txn Challenge solves a problem that is structurally near-isomorphic to this profile: a request is denied, the response carries a signed challenge identifying the operation, an asynchronous workflow runs (possibly with human approval), and the caller eventually obtains a signed transaction token bound to the original operation.

The two specs cover the same problem from different starting points:

* OAuth Txn Challenge lives in the OAuth Resource Server / Authorization Server / Client world; the final artifact is a transaction token the RS validates at request time.
* The base profile lives in the AuthZEN PDP / PEP / Access Request Service world; the final artifact is an approval the PDP considers during re-evaluation.

This profile bridges the two so that:

* An AuthZEN-backed Access Request Service can expose an OAuth Txn Challenge-compatible surface, letting OAuth-native callers integrate without learning AuthZEN.
* An AuthZEN PEP can complete an Access Request by obtaining an OAuth transaction token rather than re-evaluating, when the deployment's enforcement layer consumes OAuth tokens rather than AuthZEN decisions.
* Implementers building either side learn one model and can interoperate with the other.

The base profile already anticipates this binding.  §10 says: "Implementations that bind approval to a specific issuance flow, such as OAuth token issuance where the issued token is itself the decision representation, MUST do so through a profile that defines a completion mode appropriate to that flow."  This proposal is that profile.

## Scope

This profile defines:

* A new `result.mode` value that returns an OAuth transaction token as the Access Request completion artifact.
* The shape of the token result and the binding between this profile's wire surface and OAuth Txn Challenge claims.
* PDP, PEP, and Access Request Service processing rules specific to the token completion mode.
* An identity mapping between OAuth Txn Challenge's `act` claim and the base profile's `client.actor`.

It does **not** define:

* The OAuth Txn Challenge wire surface itself; this profile assumes deployments that want OAuth-native interoperability also implement OAuth Txn Challenge alongside this profile, with the two surfaces sharing backing workflow.
* A replacement for OAuth Txn Challenge's `Txn-Token` header or transaction token validation rules; those remain as defined by OAuth Txn Challenge.
* AuthZEN behavior at the PEP-to-RS boundary; once the token is issued, downstream enforcement is OAuth-native.

## Architectural Mapping

| OAuth Txn Challenge | Base profile |
|---|---|
| Protected Resource (RS) issues challenge | PEP receives `decision: false` with `context.access_request` |
| `transaction_challenge` in `WWW-Authenticate` (signed JWT) | `context.access_request.binding_token` (signed JWT, this profile constrains the JWT shape) |
| Authorization Server (AS) | Access Request Service |
| `transaction_authorization_endpoint` | Access Request Endpoint (`access_request_endpoint`) |
| Client posts challenge to AS | PEP submits Access Request, echoing `binding_token` as `denial.binding_token` |
| `transaction_authorization_id` | `task.id` |
| Polling at AS with `interval` / `slow_down` | Polling at Task Status Endpoint with `Retry-After` |
| `authorization_pending` | `task.status: pending` |
| `access_denied` (approval refused) | `task.status: denied` |
| `expired_token` (challenge or token expired) | `task.status: expired` |
| Transaction token response | `result.mode = urn:ietf:params:oauth:result-mode:txn-token` with token members |
| `txn` claim binding challenge to token | `denial.evaluation_id` plus `binding_token` JWS payload |
| `act` claim (RFC 8693) on the challenge | `client.actor` per §B.2 of the base profile |
| RS validates token at request time | PEP enforces token validity per OAuth Txn Challenge rules; AuthZEN re-evaluation is not used in this mode |

## Profile Registrations

### Completion mode

Registers the following value at the base profile's `result.mode` extension point ({{completion-semantics}}):

`urn:ietf:params:oauth:result-mode:txn-token`

When this mode is used, the `result` object MUST include the members defined in "Token Result Shape" below.  The PEP MUST NOT perform a new AuthZEN evaluation as the completion step; the issued transaction token is the decision representation, validated by the PEP (or the downstream Resource Server) according to OAuth Txn Challenge rules.

### Token Result Shape

When `result.mode` is the txn-token URI, the `result` object carries the OAuth Txn Challenge transaction token response shape:

| Member | Type | Description |
|---|---|---|
| `access_token` | String | The transaction token, in JWS Compact Serialization, with the JOSE `typ` header value defined by OAuth Txn Challenge. |
| `issued_token_type` | String | `urn:ietf:params:oauth:token-type:txn_token` per OAuth Txn Challenge. |
| `token_type` | String | `N_A` per OAuth Txn Challenge. |
| `expires_in` | Number | Token lifetime in seconds. |

The transaction token MUST satisfy the binding requirements in OAuth Txn Challenge: its `txn` claim MUST match the challenge's `txn` claim, its `aud` MUST identify the consumer (typically the Resource Server), and its `iss` MUST identify the Access Request Service acting as Authorization Server.

The `result` object MAY also include the base profile's `approval` member for audit correlation, but the PEP MUST NOT use `approval.id` or `approval.state` as the enforcement input when `mode` is txn-token; the transaction token is authoritative.

### Binding Token Claim Mapping

When this profile is in use, `context.access_request.binding_token` is a JWS that MUST conform to the OAuth Txn Challenge transaction challenge claim set:

| Claim | Source |
|---|---|
| `iss` | PDP identifier (which acts as the OAuth Txn Challenge Protected Resource for issuance purposes). |
| `aud` | Access Request Service identifier (which acts as the OAuth Txn Challenge Authorization Server). |
| `iat`, `exp`, `jti` | Standard JWT token-hygiene claims per §16.3 of the base profile. |
| `txn` | The transaction identifier, equal to `denial.evaluation_id` echoed in the submission. |
| `authorization_details` | OAuth RAR description of the operation; corresponds to the base profile's submitted Subject, Resource, and Action plus any `requested_access` augmentations. |
| `reason` | Human-readable rationale; corresponds to `context.reason` on the AuthZEN response. |
| `reason_uri` | Optional URI for additional context; corresponds to `context.access_request.form_url` when populated. |
| `act` | Actor / delegation information per RFC 8693; corresponds to `client.actor` per §B.2. |

This claim shape is compatible with both the base profile's §16.3 JWT guidance and OAuth Txn Challenge's required claim set, allowing the same signed JWT to satisfy both specifications.

### Capability URN

Registers a capability URN for advertising OAuth Txn Challenge bridge conformance in PDP metadata's `capabilities` array:

`urn:openid:authzen:capability:access-request:oauth-txn`

A PDP that advertises this capability MUST also publish OAuth Txn Challenge's protected-resource metadata (`txn_challenge_jwks_uri`, `txn_challenge_signing_alg_values_supported`) so that an OAuth-native AS can verify challenges issued in the binding_token role.

## Polling Cadence Alignment

OAuth Txn Challenge defines a fixed polling cadence: the AS returns `interval` (default 5 seconds) and clients increase the interval by 5 seconds on `slow_down`.  This profile reuses the base profile's polling guidance from §9 (exponential backoff, `Retry-After` honored).  When operating under this profile, an Access Request Service SHOULD respond to polls with the `Retry-After` HTTP header set to the next valid poll interval; PEPs SHOULD treat `Retry-After` as authoritative.  The OAuth Txn Challenge `interval` and `slow_down` semantics MAY be conveyed through `Retry-After` rather than through OAuth-style error response members when the wire surface in use is the base profile's Task Status Endpoint.

## PEP Processing Rules

A PEP implementing this profile, in addition to the base profile's PEP Processing Rules:

* MUST recognize `urn:ietf:params:oauth:result-mode:txn-token` as a known `result.mode` value.
* MUST validate the transaction token according to OAuth Txn Challenge rules: signature, `iss` (matched to the challenge's `aud`), `aud`, `exp`, and `txn` claim equality with the original challenge.
* MUST NOT enforce the operation past `result.expires_in` from the time the token was received.
* SHOULD treat the transaction token as single-use when the operation is non-idempotent, per OAuth Txn Challenge guidance.
* MUST NOT use `approval.id` or `approval.state` (when present in the result) as enforcement input under this profile; those members are for audit correlation only.

## Access Request Service Processing Rules

An Access Request Service implementing this profile, in addition to the base profile's Access Request Service Processing Rules:

* MUST issue the transaction token as a JWS satisfying OAuth Txn Challenge's transaction token requirements.
* MUST verify that the requester presented in the Access Request is consistent with the `act` claim of the challenge, when the `act` claim was present.
* MUST bind the transaction token's `txn` claim to the same value carried in the challenge's `txn` claim and in `denial.evaluation_id`.
* MUST set the token's `aud` to the Resource Server identifier expected by the deployment, defaulting to the challenge's `iss` per OAuth Txn Challenge unless a deployment policy specifies otherwise.

## Conformance Checklist

An OAuth-Txn-Challenge-conformant deployment of this profile:

* MUST publish `access_request_endpoint`, `jwks_uri`, and the OAuth Txn Challenge protected-resource metadata in PDP metadata.
* MUST advertise `urn:openid:authzen:capability:access-request:oauth-txn` in `capabilities`.
* MUST issue `context.access_request.binding_token` as a JWT conforming to both this profile's claim mapping and OAuth Txn Challenge's transaction challenge claim set.
* MUST return `result.mode = urn:ietf:params:oauth:result-mode:txn-token` on approval, with `access_token`, `issued_token_type`, `token_type`, and `expires_in`.
* MUST treat `task.status: expired` (or OAuth `expired_token`) as denial of the underlying operation.

A base-profile-only PEP that does not understand the txn-token mode MUST treat the result as not approved per the base profile's "MUST treat unknown `result.mode` values as not approved" rule.  AuthZEN-only PEPs and OAuth-Txn-Challenge-only clients are mutually compatible only when both implement this profile or the deployment provides bridging.

## Open Questions

* **Capability URN scope.**  Should this profile register a single capability URN, or also register OAuth Txn Challenge's existing protected-resource metadata as AuthZEN-discoverable?  This proposal does the former; a future revision could deepen the integration.
* **Re-evaluation as fallback.**  When a PEP receives `result.mode = txn-token` but cannot validate or use OAuth Txn Challenge tokens (for example, the PEP is AuthZEN-only), can the result also carry `approval.id` so the PEP could fall back to re-evaluation?  This proposal allows `approval` in the result for audit but forbids using it for enforcement under txn-token mode; an alternative would permit hybrid enforcement.
* **Audience binding for multi-RS deployments.**  OAuth Txn Challenge tokens have a single `aud` (typically the RS).  When the AuthZEN PEP enforces for multiple Resource Servers, the audience binding becomes ambiguous.  This proposal defers to OAuth Txn Challenge's existing rules and treats multi-RS as a deployment concern.
* **`Accept-Txn-Challenge` header.**  OAuth Txn Challenge uses this header to signal the agent's willingness to handle challenges.  Does an AuthZEN PEP signal equivalent willingness through PDP metadata (capability URN), through an HTTP header on its AuthZEN evaluate request, or implicitly by the deployment configuration?  This proposal treats willingness as implicit (the PEP supports this profile if its deployment configures it); explicit signaling could be added.
* **Interactive approval URL.**  OAuth Txn Challenge's `authorization_uri` is the AS-hosted URL for interactive approval.  Does this map to the base profile's `task.links.review`, `task.links.ticket`, or a new link relation?  This proposal does not register a mapping; profiles or deployments may publish the URL through whichever `task.links` member best fits.
* **`act` claim normalization.**  OAuth Txn Challenge's `act` claim is JWT-shaped; the base profile's `client.actor` is an Access Request submission member with similar but not identical shape.  This proposal asserts equivalence in §B.2 terms but a future revision could pin a canonical normalization.

## Relationship to the Base Profile

This proposal is a strict extension: it populates the `result.mode` extension point the base profile defined, registers names in the AuthZEN Access Request Member Names registry (for the token result members), registers a capability URN, and constrains the JWT shape of an existing binding_token to align with OAuth Txn Challenge's claim set.  It introduces no new endpoints in the base profile and no new wire shapes outside what the base profile's extension points already permit.

A future revision could go further and define a unified protocol surface in which the AuthZEN Access Request Endpoint IS the OAuth Txn Challenge `transaction_authorization_endpoint` (one URL, two specifications worth of conformance).  That work is out of scope for this proposal; this proposal sketches the minimum interoperability bridge.

## References

- [draft-mcguinness-authzen-access-request](../draft-mcguinness-authzen-access-request.md): the base profile this proposal profiles.
- OAuth Transaction Authorization Challenge, draft-rosomakho-oauth-txn-challenge: https://yaroslavros.github.io/oauth-txn-challenge/draft-rosomakho-oauth-txn-challenge.html.
- RFC 8693 OAuth Token Exchange: defines the `act` claim used in both specs for delegation chains.
- RFC 9396 OAuth Rich Authorization Requests: defines the `authorization_details` claim used in OAuth Txn Challenge.
