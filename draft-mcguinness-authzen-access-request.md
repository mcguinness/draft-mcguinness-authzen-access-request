---
title: "AuthZEN Access Request and Approval Profile"
abbrev: "AuthZEN Access Request"
category: std

docname: draft-mcguinness-authzen-access-request-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
# area: "Security"
# workgroup: "OpenID AuthZEN Working Group"
keyword:
  - authorization
  - access request
  - approval workflow
  - policy decision point
  - policy enforcement point

venue:
#  group: "OpenID AuthZEN Working Group"
#  type: "Working Group"
#  mail: "openid-specs-authzen@lists.openid.net"
#  arch: "https://lists.openid.net/pipermail/openid-specs-authzen/"

author:
  -
    name: Karl McGuinness
    org: Independent
    email: public@karlmcguinness.com

normative:
  RFC9110:
  RFC9457:
  RFC3339:
  RFC6749:
  RFC6750:
  RFC8417:
  SSF:
    title: "OpenID Shared Signals Framework Specification 1.0"
    target: "https://openid.net/specs/openid-sharedsignals-framework-1_0.html"
    author:
      -
        ins: A. Tulshibagwale
        name: Atul Tulshibagwale
      -
        ins: T. Cappalli
        name: Tim Cappalli
      -
        ins: M. Scurtescu
        name: Marius Scurtescu
      -
        ins: A. Backman
        name: Annabelle Backman
      -
        ins: J. Bradley
        name: John Bradley
    date: 2024
  AuthZEN:
    title: "Authorization API 1.0"
    target: "https://openid.github.io/authzen/"
    author:
      -
        ins: O. Gazitt
        name: Omri Gazitt
      -
        ins: D. Brossard
        name: David Brossard
      -
        ins: A. Tulshibagwale
        name: Atul Tulshibagwale
    date: 2026-04-29

informative:
  XACML:
    title: "eXtensible Access Control Markup Language (XACML) Version 3.0"
    target: "https://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html"
    author:
      -
        ins: OASIS
        name: OASIS
    date: 2013-01

--- abstract

This specification defines an extension profile for the OpenID AuthZEN Authorization API that allows a Policy Enforcement Point (PEP) to submit an access request when an authorization decision is denied but requestable.  The profile preserves the AuthZEN decision model: a denied decision remains a denial and MUST NOT be treated as access.  It adds a requestable denial context, an access request endpoint, a task handle for asynchronous approval workflows, and completion semantics that allow a PEP to re-evaluate or consume a final authorization result after approval.

--- middle

# Introduction

The AuthZEN Authorization API enables a Policy Enforcement Point (PEP) to ask a Policy Decision Point (PDP) whether a Subject may perform an Action on a Resource within a Context.  The PDP returns a Decision indicating whether the operation is allowed or denied.

Many real-world authorization systems require a remediation path when access is denied.  For example, a user may be denied access to a customer record because manager approval is required, or a machine workload may be denied a privileged operation until a change-control task is approved.  Without an interoperable protocol surface, PEPs commonly fall back to non-standard user-interface messages, out-of-band tickets, or vendor-specific governance integrations.

This profile defines a narrow, interoperable mechanism for requestable denials:

1.  The PEP evaluates access using the AuthZEN Access Evaluation API.
2.  The PDP returns `decision: false` and a structured `access_request` object in the Decision Context when the denial is eligible for an approval workflow.
3.  The PEP submits an access request to the Access Request Endpoint.
4.  The Access Request Service returns an opaque task handle.
5.  The PEP can poll the task, receive a callback, or otherwise use the task handle to determine completion.
6.  When the task is approved, the PEP either performs a new AuthZEN evaluation or receives an approval-bound result that can be enforced according to this profile.

This specification intentionally does not define a workflow engine, approval policy language, ticketing system, entitlement catalog, or user interface.  Those capabilities remain behind the PDP or Access Request Service.  The purpose of this profile is to standardize the handoff between authorization enforcement and approval workflow.

# Requirements Notation and Conventions

{::boilerplate bcp14-tagged}

The terms Policy Decision Point (PDP), Policy Enforcement Point (PEP), Subject, Resource, Action, Context, and Decision are used as defined by {{AuthZEN}}.

# Design Goals

This profile has the following design goals:

* Preserve the AuthZEN allow/deny decision model.
* Make requestability explicit and machine-readable.
* Provide an opaque task handle suitable for asynchronous approval workflows.
* Avoid embedding a workflow policy language in the authorization response.
* Allow approval systems such as ITSM, IGA, chat approval, case management, or custom governance systems to sit behind a common endpoint.
* Support re-evaluation after approval so the PDP remains authoritative at enforcement time.
* Provide enough audit correlation to bind the original denial, submitted request, approver action, and final authorization result.

# Terminology

Access Request:
: A request submitted after a denied AuthZEN decision asking that access be approved, granted, or otherwise remediated.

Access Request Service:
: The service that receives Access Request submissions and manages the resulting approval task.  It MAY be implemented by the PDP, by a governance system, or by an external workflow engine acting on behalf of the PDP.

Requestable Denial:
: An AuthZEN Decision with `decision` set to `false` and a Decision Context indicating that the denied access can be requested through an Access Request Endpoint.

Task Handle:
: An opaque identifier and associated status endpoint representing the lifecycle of an Access Request.

Approval Result:
: The completed result of an Access Request task.  An Approval Result does not itself permit access unless it is returned as, or is used to obtain, an AuthZEN allow decision according to this profile.

# Protocol Overview

~~~ ascii-art
+---------+                         +---------+                    +----------------+
|   PEP   |                         |   PDP   |                    | Access Request |
|         |                         |         |                    |    Service     |
+----+----+                         +----+----+                    +-------+--------+
     |                                   |                                 |
     | 1. Access Evaluation              |                                 |
     |---------------------------------->|                                 |
     |                                   |                                 |
     | 2. decision=false                 |                                 |
     |    context.access_request         |                                 |
     |<----------------------------------|                                 |
     |                                   |                                 |
     | 3. Submit Access Request          |                                 |
     |------------------------------------------------------------------->|
     |                                   |                                 |
     | 4. task handle                    |                                 |
     |<-------------------------------------------------------------------|
     |                                   |                                 |
     | 5. Poll task or receive callback  |                                 |
     |------------------------------------------------------------------->|
     |<-------------------------------------------------------------------|
     |                                   |                                 |
     | 6. Re-evaluate or consume result  |                                 |
     |---------------------------------->|                                 |
     |<----------------------------------|                                 |
~~~

The access evaluation in step 1 uses the existing AuthZEN Access Evaluation API.  The denial in step 2 is still a denial.  The PEP MUST NOT permit the requested operation based only on the presence of `context.access_request`.

The Access Request Service MAY additionally publish lifecycle events for governance, audit, and analytics consumers using the OPTIONAL Shared Signals Framework binding defined in {{ssf-binding}}.  That channel is independent of the per-task callback and is not used for enforcement.

# Discovery

A PDP supporting this profile MUST publish an `access_request_endpoint` in PDP metadata.  The endpoint value MUST be an HTTPS URI.

A PDP supporting this profile SHOULD include the following capability URN in the `capabilities` array:

`urn:openid:authzen:capability:access-request`

The following is a non-normative metadata example:

~~~ json
{
  "policy_decision_point": "https://pdp.example.com",
  "access_evaluation_endpoint": "https://pdp.example.com/access/v1/evaluation",
  "access_evaluations_endpoint": "https://pdp.example.com/access/v1/evaluations",
  "access_request_endpoint": "https://pdp.example.com/access/v1/requests",
  "capabilities": [
    "urn:openid:authzen:capability:access-request",
    "urn:openid:authzen:capability:access-request-ssf"
  ]
}
~~~

The `access-request-ssf` capability is OPTIONAL and is included only when the deployment supports the Shared Signals Framework binding ({{ssf-binding}}).

The `access_request_endpoint` MAY be hosted by the PDP or by another service trusted by the PDP.  When hosted by another service, the PDP metadata MUST identify the endpoint actually used by the PEP to submit access requests.

# Requestable Denial Context

When an AuthZEN Access Evaluation response denies access and the denial is eligible for an access request, the PDP MAY include an `access_request` object in the Decision Context.

The `access_request` object has the following members:

`requestable`:
: REQUIRED.  Boolean.  When `true`, indicates that the denied request is eligible for submission to an Access Request Endpoint.  When absent or `false`, the PEP MUST NOT treat the denial as requestable under this profile.

`endpoint`:
: OPTIONAL.  HTTPS URI.  The endpoint to which the PEP submits the access request.  If omitted, the PEP MUST use the `access_request_endpoint` from PDP metadata.

`template`:
: OPTIONAL.  String.  An opaque template identifier that can guide the Access Request Service.  The value is not a policy language and MUST NOT be interpreted by the PEP except for display or request submission.

`reason`:
: OPTIONAL.  String.  A stable, machine-readable reason code for the requestable denial.

`display`:
: OPTIONAL.  Object.  Localizable user-interface hints such as title, description, or recommended call-to-action text.  The PEP MAY ignore this member.

`expires_at`:
: OPTIONAL.  String containing an {{RFC3339}} timestamp.  Indicates when the requestable denial hint expires.

`expires_in`:
: OPTIONAL.  Number.  Lifetime in seconds of the requestable denial hint from the time the Decision was produced.

`request_context`:
: OPTIONAL.  Object.  Opaque context to be returned to the Access Request Service when submitting the access request.  The PEP MUST NOT modify or interpret this object.  The PEP returns it unchanged inside `denial.access_request.request_context` when submitting the Access Request ({{access-request-submission}}).

`required_fields`:
: OPTIONAL.  Object identifying additional request fields the Access Request Service expects.  Each member name identifies a destination object in the Access Request submission body and the value is an array of field names that the Access Request Service expects within that destination.  Defined destination names are `context` (members of the submission `context` object) and `requested_access` (members of the submission `requested_access` object).  Implementations MAY define additional destinations for extension fields.

If both `expires_at` and `expires_in` are present, `expires_at` takes precedence and the PEP MUST ignore `expires_in`.

The `reason` member of the `access_request` object is independent of any `reason` member that the PDP may include directly in `context`.  The outer `context.reason` describes why the Decision was denied; `context.access_request.reason` describes why the denial is requestable and which approval flow applies.

The following is a non-normative example:

~~~ json
{
  "decision": false,
  "context": {
    "reason": "approval_required",
    "access_request": {
      "requestable": true,
      "endpoint": "https://pdp.example.com/access/v1/requests",
      "template": "manager_approval",
      "reason": "manager_approval_required",
      "expires_in": 600,
      "required_fields": {
        "context": ["business_justification"],
        "requested_access": ["requested_duration"]
      },
      "display": {
        "title": "Request access",
        "description": "Manager approval is required before this document can be opened."
      }
    }
  }
}
~~~

# Access Request Endpoint

The Access Request Endpoint accepts an Access Request submission and returns a Task Handle.

The endpoint is identified by the `access_request_endpoint` PDP metadata parameter or by the `context.access_request.endpoint` value returned in a requestable denial.  The endpoint path is deployment-specific.

## Access Request Submission {#access-request-submission}

The PEP submits an Access Request using the HTTP `POST` method as defined in {{RFC9110}}.

The request body is a JSON object with the following members:

`subject`:
: REQUIRED.  The AuthZEN Subject from the denied evaluation.

`resource`:
: REQUIRED.  The AuthZEN Resource from the denied evaluation.

`action`:
: REQUIRED.  The AuthZEN Action from the denied evaluation.

`context`:
: OPTIONAL.  The AuthZEN Context from the denied evaluation, augmented with submission-time fields such as business justification.

`denial`:
: REQUIRED.  Object binding the Access Request to the denied AuthZEN Decision.

`requested_access`:
: OPTIONAL.  Object containing request-specific information such as requested duration, requested role, requested entitlement, or requested scope.  This object does not define policy semantics and is interpreted by the Access Request Service.

`callback`:
: OPTIONAL.  Object describing a callback endpoint where the Access Request Service can send completion notifications.

`client`:
: OPTIONAL.  Object identifying the PEP or calling application submitting the Access Request, supplementing the authenticated caller identity.  The following members are defined; implementations MAY include additional members.

  * `id`: OPTIONAL.  String.  Stable identifier for the calling application or PEP deployment.
  * `instance_id`: OPTIONAL.  String.  Identifier for a specific running instance of the PEP.
  * `name`: OPTIONAL.  String.  Human-readable name of the calling application.

The `denial` object has the following members:

`decision`:
: REQUIRED.  The denied AuthZEN Decision returned by the PDP, including its Decision Context.

`access_request`:
: REQUIRED.  The `context.access_request` object from the denied decision.

`evaluation_id`:
: RECOMMENDED.  A stable identifier for the denied evaluation.  When the PDP returns an evaluation identifier (for example in the Decision Context or in a response header), the PEP SHOULD include it here.  `evaluation_id` provides the strongest audit binding between the original denial and the submitted Access Request and SHOULD be preferred over `evaluated_at` alone.

`evaluated_at`:
: OPTIONAL.  {{RFC3339}} timestamp indicating when the denial was produced.

A PEP MUST submit an Access Request only for an AuthZEN Decision with `decision` equal to `false` and `context.access_request.requestable` equal to `true`.

A PEP SHOULD include an `Idempotency-Key` header, following the conventions described in {{?I-D.ietf-httpapi-idempotency-key-header}}.  The Access Request Service SHOULD treat repeated submissions with the same `Idempotency-Key`, requester, subject, action, resource, and denial binding as the same request and return the same Task Handle while the original request remains available.

Non-normative example:

~~~ http
POST /access/v1/requests HTTP/1.1
Host: pdp.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
Content-Type: application/json
Idempotency-Key: 7b8d0f0d-65a1-4af1-9fd3-a684f08a5d13

{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "document",
    "id": "q4-plan"
  },
  "action": {
    "name": "can_read"
  },
  "context": {
    "business_justification": "Needed for customer renewal review"
  },
  "requested_access": {
    "requested_duration": "PT4H"
  },
  "denial": {
    "evaluation_id": "eval_01HX4Y2P8BQ4Y3F0V0K9D6Z7M1",
    "evaluated_at": "2026-04-30T20:15:00Z",
    "decision": {
      "decision": false,
      "context": {
        "reason": "approval_required"
      }
    },
    "access_request": {
      "requestable": true,
      "template": "manager_approval",
      "reason": "manager_approval_required"
    }
  }
}
~~~

## Access Request Response

A successful Access Request submission returns HTTP status code `201 Created` or `202 Accepted` and a JSON object containing a `task` member.  The `task.status_endpoint` member is authoritative for subsequent status retrieval.  A response MAY also include an HTTP `Location` header equal to `task.status_endpoint`.

The `task` object has the following members:

`id`:
: REQUIRED.  Opaque task identifier.  The PEP MUST NOT parse or infer semantics from this value.

`status`:
: REQUIRED.  Current task status.  Values are defined in {{task-status}}.

`status_endpoint`:
: REQUIRED.  HTTPS URI used to retrieve task status.

`expires_at`:
: OPTIONAL.  {{RFC3339}} timestamp after which the task handle is no longer valid.

`display`:
: OPTIONAL.  Object containing user-interface hints for the pending request.

`links`:
: OPTIONAL.  Object containing related URLs.  Each member name is a link relation type and the value is an HTTPS URI.  The following relation types are defined; implementations MAY define additional relation types.

  * `ticket`: URL where the requester (Subject) can view the request and its status.
  * `review`: URL where an approver or administrator can review or act on the request.

Non-normative example:

~~~ http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4

{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "pending",
    "status_endpoint": "https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "expires_at": "2026-04-30T23:00:00Z",
    "display": {
      "title": "Access request submitted",
      "description": "Your manager has been asked to approve access."
    }
  }
}
~~~

# Task Status Endpoint

The Task Status Endpoint allows the PEP to retrieve the state of a previously submitted Access Request.

The PEP calls the `status_endpoint` using the HTTP `GET` method as defined in {{RFC9110}}.

Non-normative example:

~~~ http
GET /access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4 HTTP/1.1
Host: pdp.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
Accept: application/json
~~~

A successful response returns a JSON object containing a `task` member and, when the task is complete, MAY include a `result` member.

## Task Status Values {#task-status}

The following task status values are defined:

`pending`:
: The request has been accepted and is awaiting processing or approval.

`approved`:
: The request was approved.  Approval does not by itself grant access unless accompanied by a result that can be enforced under {{completion-semantics}}.

`denied`:
: The request was denied by the approval workflow.

`expired`:
: The request expired before completion.

`cancelled`:
: The request was cancelled by the requester, approver, administrator, or system.

`failed`:
: The request could not be completed due to an error.

Implementations MAY define additional status values.  A PEP that receives an unknown status value MUST treat the task as not approved.

## Pending Task Response

Non-normative example:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "pending",
    "status_endpoint": "https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "expires_at": "2026-04-30T23:00:00Z"
  }
}
~~~

## Completed Task Response

A completed task response MAY include a `result` object.  The `result` object MUST use one of the completion forms defined in {{completion-semantics}}.

A task remains retrievable from the Task Status Endpoint after it has reached a terminal status, until `task.expires_at` is reached or the Access Request Service removes it according to local retention policy.  After expiry or removal, the Task Status Endpoint MUST return `urn:openid:authzen:access-request:error:task_expired` or `urn:openid:authzen:access-request:error:unknown_task` as appropriate.

Cancellation of a pending Access Request is administrative and is performed by the Access Request Service, the requester through a separate user interface, or an approver.  This profile does not define a PEP-initiated cancellation API; PEPs that need to abandon an outstanding request stop polling and rely on `task.expires_at` and Access Request Service expiry to release resources.

Non-normative example using a re-evaluation requirement:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
      "approved_at": "2026-04-30T20:42:00Z",
      "approved_until": "2026-05-01T00:42:00Z"
    },
    "approval_context": {
      "approval_id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH"
    }
  }
}
~~~

# Completion Semantics {#completion-semantics}

This profile defines three completion modes, identified by `result.mode`: `reevaluate`, `decision`, and `token`.  Implementations MAY define additional completion modes.  A PEP that receives an unknown `result.mode` value MUST treat the task as not approved and MUST NOT permit access on the basis of that result.

## Re-evaluation Mode

In re-evaluation mode, the Access Request Service returns an approval reference and instructs the PEP to perform a new AuthZEN Access Evaluation.

The `result.mode` value is `reevaluate`.

The result MAY include an `approval_context` member.  `approval_context` is an opaque object populated by the Access Request Service or PDP.  When present, the PEP MUST include it as a member named `authzen_access_request_approval` inside the AuthZEN request `context` when re-evaluating.  The PEP MUST NOT modify or interpret the contents of `approval_context`.

The PDP MUST evaluate the new request using current policy and the approval reference.  The PDP MAY still deny access if policy, subject, resource, action, context, approval lifetime, or risk state no longer permits access.

Non-normative re-evaluation request:

~~~ json
{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "document",
    "id": "q4-plan"
  },
  "action": {
    "name": "can_read"
  },
  "context": {
    "authzen_access_request_approval": {
      "approval_id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH"
    },
    "time": "2026-04-30T20:43:00Z"
  }
}
~~~

Non-normative re-evaluation response:

~~~ json
{
  "decision": true,
  "context": {
    "approval": {
      "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
      "approved_until": "2026-05-01T00:42:00Z"
    }
  }
}
~~~

Re-evaluation mode is the RECOMMENDED completion mode because it ensures the PDP remains authoritative at enforcement time.

## Decision Result Mode

In decision result mode, the completed task response includes an AuthZEN Decision in `result.decision`.

The `result.mode` value is `decision`.

A `result` with `mode` equal to `decision` MUST include a `binding` member.  The `binding` object has the following members:

* `task_id`: REQUIRED.  String.  The task identifier this Decision is bound to; MUST equal the `task.id` returned for the same Access Request.
* `subject`: REQUIRED.  The AuthZEN Subject the Decision applies to.
* `resource`: REQUIRED.  The AuthZEN Resource the Decision applies to.
* `action`: REQUIRED.  The AuthZEN Action the Decision applies to.
* `approval_id`: RECOMMENDED.  String.  Identifier of the approval that produced the Decision.
* `expires_at`: REQUIRED.  {{RFC3339}} timestamp after which the Decision MUST NOT be enforced.

The PEP MAY enforce the returned Decision only if all of the following are true:

* The response was retrieved over an authenticated channel from the Access Request Service identified in PDP metadata or the requestable denial.
* The `binding.task_id` equals the task identifier the PEP submitted, and the `binding.subject`, `binding.resource`, and `binding.action` equal the values the PEP submitted in the Access Request.
* The current time is at or before `binding.expires_at`.
* The PEP understands all obligations required for enforcement.

A Decision Result that is not signed using a mechanism agreed between the PEP and Access Request Service relies on the integrity of the transport channel and the trust the PEP places in the Access Request Service.  Deployments requiring stronger integrity SHOULD wrap the result in a JSON Web Signature (JWS) whose payload contains the `binding` and `decision` members; this profile does not mandate a specific signing format.

Non-normative example:

~~~ json
{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "approved"
  },
  "result": {
    "mode": "decision",
    "binding": {
      "task_id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
      "subject": {
        "type": "user",
        "id": "alice@example.com"
      },
      "resource": {
        "type": "document",
        "id": "q4-plan"
      },
      "action": {
        "name": "can_read"
      },
      "approval_id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
      "expires_at": "2026-05-01T00:42:00Z"
    },
    "decision": {
      "decision": true,
      "context": {
        "approval": {
          "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
          "approved_until": "2026-05-01T00:42:00Z"
        },
        "obligations": [
          {
            "type": "audit",
            "message": "Record approved access usage"
          }
        ]
      }
    }
  }
}
~~~

## Token Result Mode

In token result mode, the completed task response returns a token or token reference that the PEP can use to call a downstream resource server, or to exchange for a downstream credential, in order to perform the approved operation.  The PEP MUST NOT treat receipt of the token as itself an authorization decision; the token is consumed by the system that the token is audience-restricted to, and that system performs its own authorization checks.

The `result.mode` value is `token`.

This profile does not define the token format.  When OAuth 2.0 access tokens are used, the token response and token usage MUST follow the applicable OAuth specifications, including {{RFC6749}} and {{RFC6750}}.

The token MUST be audience-restricted to the intended recipient, time-bounded, and bound to the approved subject, resource, action, and task to the extent supported by the token format.

Non-normative example:

~~~ json
{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "approved"
  },
  "result": {
    "mode": "token",
    "token_type": "Bearer",
    "access_token": "eyJhbGciOi...",
    "expires_in": 900
  }
}
~~~

# Callback Completion {#callback-completion}

A PEP MAY request callback notification by including a `callback` object in the Access Request submission.

The `callback` object has the following members:

`endpoint`:
: REQUIRED.  HTTPS URI to which the Access Request Service sends completion notifications.

`state`:
: OPTIONAL.  Opaque value supplied by the PEP and returned unmodified in the callback.

`events`:
: OPTIONAL.  Array of event names requested by the PEP.  Defined event names are `approved`, `denied`, `expired`, `cancelled`, and `failed`.

The Access Request Service MUST authenticate to the callback endpoint using a mechanism agreed between the PEP and Access Request Service.  This specification does not mandate a single callback authentication mechanism, but implementations SHOULD use one of the following: an OAuth 2.0 bearer token {{RFC6750}} issued to the Access Request Service, mutual TLS, or an HMAC signature over the request body using a pre-shared key.  Unauthenticated callbacks MUST NOT be accepted.

Callback delivery is a notification optimization.  The Task Status Endpoint remains authoritative unless the callback contains an enforceable completion result under {{completion-semantics}}.

Non-normative callback example:

~~~ http
POST /callbacks/access-requests HTTP/1.1
Host: pep.example.com
Content-Type: application/json

{
  "state": "b3Blbi1kb2N1bWVudC1mbG93",
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "approved",
    "status_endpoint": "https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4"
  }
}
~~~

# Shared Signals Framework Binding {#ssf-binding}

This section defines an OPTIONAL binding that publishes Access Request lifecycle events using the OpenID Shared Signals Framework {{SSF}} and Security Event Tokens {{RFC8417}}.

The SSF binding does not replace the per-task callback defined in {{callback-completion}}.  The callback addresses synchronous correlation between a single PEP and the Access Request task it submitted.  The SSF binding addresses fan-out to governance, audit, analytics, and operations consumers that need to observe Access Request activity across many submissions and submitters.

## Event Types

This profile defines the following SET event type:

`urn:openid:authzen:event:access-request-completed`:
: Emitted when an Access Request reaches a terminal status (`approved`, `denied`, `expired`, `cancelled`, or `failed`).

Implementations MAY define additional event types for non-terminal transitions, such as submission, assignment, or escalation.  Receivers MUST ignore SET event types that they do not recognize.

## SET Payload

A Security Event Token carrying an `access-request-completed` event has the following structure:

* `iss` — the issuer identifier of the Access Request Service.
* `aud` — the intended receiver of the event.
* `iat` — the time the event was issued.
* `jti` — a unique identifier for the SET.
* `events` — a JSON object whose member name is the event type URN and whose member value is the event payload.

The event payload is a JSON object with the following members:

`task`:
: REQUIRED.  A `task` object as defined in {{task-status}}, including the terminal status.

`subject`:
: RECOMMENDED.  The AuthZEN Subject from the original denied evaluation.

`resource`:
: RECOMMENDED.  The AuthZEN Resource from the original denied evaluation.

`action`:
: RECOMMENDED.  The AuthZEN Action from the original denied evaluation.

`evaluation_id`:
: OPTIONAL.  The evaluation identifier from the original denied evaluation, when available.

`result`:
: OPTIONAL.  A completion result object as defined in {{completion-semantics}}.  A receiver MUST NOT enforce a `result` carried in a SET as an authorization decision; SET delivery is a notification channel, not an enforcement channel.  Receivers requiring an enforceable result MUST obtain it from the Task Status Endpoint or from a per-task callback.

Non-normative example:

~~~ json
{
  "iss": "https://pdp.example.com",
  "aud": "https://audit.example.com",
  "iat": 1714508520,
  "jti": "set_01HX4Y9V3ZJ9C2X7K8M0N1P2Q3",
  "events": {
    "urn:openid:authzen:event:access-request-completed": {
      "task": {
        "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
        "status": "approved"
      },
      "subject": {
        "type": "user",
        "id": "alice@example.com"
      },
      "resource": {
        "type": "document",
        "id": "q4-plan"
      },
      "action": {
        "name": "can_read"
      },
      "evaluation_id": "eval_01HX4Y2P8BQ4Y3F0V0K9D6Z7M1",
      "result": {
        "mode": "reevaluate",
        "approval": {
          "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
          "approved_until": "2026-05-01T00:42:00Z"
        }
      }
    }
  }
}
~~~

## Stream Management and Delivery

Stream configuration, verification, and delivery (push or poll) follow {{SSF}}.  The Access Request Service acts as an SSF Transmitter; consumers act as SSF Receivers.  Receivers verify SET signatures using keys discovered through the Transmitter's configuration metadata.

## Relationship to Per-Task Callback

A deployment MAY support the per-task callback ({{callback-completion}}), the SSF binding, both, or neither.  When both are supported:

* The same Access Request lifecycle event MAY be delivered through both channels.
* Receivers MUST NOT assume ordering between callback delivery and SET delivery.
* A `result` member that is enforceable under {{completion-semantics}} MAY appear in the per-task callback.  An equivalent `result` carried in a SET is a notification only and MUST NOT be enforced by the receiver.

## SSF Binding Discovery

A PDP supporting the SSF binding SHOULD include the following capability URN in PDP metadata:

`urn:openid:authzen:capability:access-request-ssf`

When present, the PDP metadata SHOULD also identify the SSF Transmitter configuration endpoint that publishes Access Request events, using the discovery mechanism defined by {{SSF}}.

# Error Responses

HTTP error responses from the Access Request Endpoint and Task Status Endpoint SHOULD use `application/problem+json` as defined by {{RFC9457}}.

The following problem types are defined:

`urn:openid:authzen:access-request:error:not_requestable`:
: HTTP `400 Bad Request`.  The submitted denial is not requestable.

`urn:openid:authzen:access-request:error:expired_denial`:
: HTTP `410 Gone`.  The requestable denial hint has expired.

`urn:openid:authzen:access-request:error:invalid_denial_binding`:
: HTTP `400 Bad Request`.  The submitted Access Request cannot be bound to the denied AuthZEN Decision.

`urn:openid:authzen:access-request:error:duplicate_request`:
: HTTP `409 Conflict`.  A semantically duplicate Access Request exists and cannot be reused.

`urn:openid:authzen:access-request:error:unknown_task`:
: HTTP `404 Not Found`.  The task handle is unknown or unavailable to the caller.

`urn:openid:authzen:access-request:error:task_expired`:
: HTTP `410 Gone`.  The task handle has expired.

Non-normative example:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "urn:openid:authzen:access-request:error:not_requestable",
  "title": "Access is not requestable",
  "status": 400,
  "detail": "The denied decision did not contain context.access_request.requestable=true."
}
~~~

# PEP Processing Rules

A PEP implementing this profile:

* MUST treat `decision: false` as a denial, even when the Decision Context contains an `access_request` object.
* MUST NOT submit an Access Request unless the denied Decision contains `context.access_request.requestable` set to `true`.
* MUST use the `endpoint` from the denial context when present; otherwise it MUST use the `access_request_endpoint` from PDP metadata.
* MUST preserve the Subject, Resource, Action, and relevant Context of the denied evaluation when submitting the Access Request.
* SHOULD include `denial.evaluation_id` when the PDP returned an evaluation identifier.
* SHOULD include an idempotency key for Access Request submissions.
* MUST treat a Task Handle as opaque.
* MUST NOT infer approval from a task identifier, link, or display text.
* MUST treat unknown task status values as not approved.
* MUST enforce an approved result only according to {{completion-semantics}}.
* MUST treat unknown `result.mode` values as not approved.
* When using Re-evaluation Mode, MUST include any returned `approval_context` unchanged as a member named `authzen_access_request_approval` inside the AuthZEN request `context`.
* When using Decision Result Mode, MUST verify the `result.binding` matches the originally submitted task identifier, Subject, Resource, and Action, and MUST NOT enforce the Decision after `binding.expires_at`.
* SHOULD re-evaluate access after approval unless the deployment explicitly supports Decision Result Mode or Token Result Mode.

# PDP Processing Rules

A PDP implementing this profile:

* MAY include `context.access_request` in a denied AuthZEN Decision when the denied access is eligible for approval.
* MUST NOT include `context.access_request.requestable=true` unless an Access Request Endpoint is available to process the request.
* SHOULD include a stable machine-readable reason code when returning a requestable denial.
* SHOULD include an expiration time or lifetime for the requestable denial hint.
* SHOULD return a stable evaluation identifier that the PEP can supply as `evaluation_id` when submitting an Access Request, either in the Decision Context or in a response header.
* MUST validate approval references presented during re-evaluation.
* MUST ensure that approval does not override policy conditions that remain mandatory at enforcement time, such as subject status, resource sensitivity, action constraints, environmental risk, and approval expiry.

# Access Request Service Processing Rules

An Access Request Service implementing this profile:

* MUST authenticate and authorize the PEP before accepting Access Request submissions.
* MUST validate that the submission is based on a requestable denial.
* MUST bind the task to the submitted Subject, Resource, Action, Context, denial, requester, and client.
* MUST return an opaque Task Handle for accepted requests.
* SHOULD support idempotent request submission using the `Idempotency-Key` header.
* MUST expire Access Requests and approvals according to local policy.
* MUST not return `approved` unless the configured approval workflow has completed successfully.
* MUST retain sufficient audit records to reconstruct the request, approval, denial, and completion result.

# Authorization and Authentication

The Access Request Endpoint and Task Status Endpoint are protected APIs.  Support for OAuth 2.0 {{RFC6749}} is RECOMMENDED.  When OAuth 2.0 bearer tokens are used, the endpoints MUST follow {{RFC6750}}.

The Access Request Service MUST authenticate the PEP or caller before accepting a submission or returning task status.  The service MUST verify that the caller is authorized to submit or view the request for the supplied Subject, Resource, and Action.

A task status response MUST NOT disclose approval details, approver identities, policy identifiers, or resource metadata to a caller that is not authorized to receive them.

# Privacy Considerations

Access Requests may contain sensitive information, including user identifiers, resource identifiers, business justifications, approval chains, and policy reasons.  Implementations SHOULD minimize the amount of information returned to the PEP and displayed to the end user.

The Access Request Service SHOULD separate end-user display reasons from administrator diagnostic reasons.  A requestable denial response SHOULD avoid exposing internal policy identifiers unless the PEP is authorized for administrative diagnostics.

Approval records SHOULD be retained only as long as required by business, security, and compliance policy.

# Security Considerations

## Denial Remains Denial

The presence of `context.access_request` does not weaken the AuthZEN decision.  A PEP MUST NOT grant access based on a requestable denial.  Access is permitted only after an approved completion result is enforced according to this profile.

## Confused Deputy and Request Substitution

An attacker could attempt to obtain approval for one resource and apply it to another.  Implementations MUST bind Access Requests and approval results to the Subject, Resource, Action, Context, task, and requester.  PDPs MUST validate this binding during re-evaluation.

## Approval Replay

Approval references and tokens can be replayed if not time-bounded and audience-restricted.  Approval results MUST expire.  Token Result Mode MUST use short-lived, audience-restricted tokens.  Re-evaluation Mode SHOULD bind approval references to the original request tuple.

## Overbroad Approval

This profile does not define an approval policy language.  Implementations MUST NOT treat the `template`, `reason`, `requested_access`, or `display` fields as sufficient authorization policy.  Actual approval scope and enforcement semantics are determined by the PDP and Access Request Service.

## Task Handle Leakage

Task handles can reveal workflow state or be used to poll for sensitive information.  Task handles MUST be opaque, unguessable, and protected by authentication and authorization checks.  A leaked task handle MUST NOT be sufficient to retrieve task status without caller authorization.

## Callback Security

Callback endpoints can be abused for spoofing, replay, and request forgery.  Callback notifications MUST be authenticated.  PEPs SHOULD verify callback origin, bind callbacks to expected task identifiers and state values, and treat callbacks as notifications unless they contain an enforceable result under this profile.

## Shared Signals Framework Binding Security

When the SSF binding ({{ssf-binding}}) is used, SETs carry information about denied evaluations, approval outcomes, subjects, and resources.  Transmitters MUST authorize Receivers before delivering events and MUST NOT broadcast events to Receivers that are not entitled to observe the affected Subject, Resource, or Action.  Receivers MUST validate SET signatures and MUST reject SETs whose `iss` does not match the configured Transmitter for the stream.  A SET delivered by the SSF binding MUST NOT be treated as an enforceable authorization decision; the Task Status Endpoint or per-task callback remains authoritative for enforcement.

## PEP Acting on Behalf of the Subject

A PEP submitting an Access Request typically acts on behalf of the Subject identified in the original AuthZEN evaluation.  The Access Request Service MUST verify that the authenticated caller is authorized to act for that Subject for that Resource and Action, including that the caller is a recognized PEP and that the Subject has consented or been delegated to where required.  Deployments requiring explicit delegation MAY use OAuth 2.0 Token Exchange {{?RFC8693}} so the PEP presents a token that names the Subject as the on-behalf-of party.

## Idempotency Key Abuse

Idempotency keys can be used to correlate requests.  Implementations SHOULD scope idempotency keys to the authenticated caller and avoid storing them longer than necessary.

## Availability

Approval workflows can introduce latency and dependency on external systems.  PEPs SHOULD fail closed when task status cannot be determined.  Access Request Services SHOULD apply rate limits and abuse detection to request submission and polling endpoints.

# IANA Considerations

## AuthZEN Policy Decision Point Metadata Registry

This specification requests registration of the following PDP metadata parameter in the AuthZEN Policy Decision Point Metadata Registry.

Name:
: `access_request_endpoint`

Description:
: HTTPS endpoint used to submit Access Requests for requestable denials.

Change Controller:
: OpenID Foundation AuthZEN Working Group

Specification Document:
: This document.

## AuthZEN Policy Decision Point Capabilities Registry

This specification requests registration of the following PDP capabilities in the AuthZEN Policy Decision Point Capabilities Registry.

Capability Name:
: `access-request`

Capability URN:
: `urn:openid:authzen:capability:access-request`

Capability Description:
: Indicates that the PDP supports requestable denials and the Access Request Endpoint defined by this specification.

Change Controller:
: OpenID Foundation AuthZEN Working Group

Specification Document:
: This document.

Capability Name:
: `access-request-ssf`

Capability URN:
: `urn:openid:authzen:capability:access-request-ssf`

Capability Description:
: Indicates that the PDP or Access Request Service publishes Access Request lifecycle events using the Shared Signals Framework binding defined by this specification.

Change Controller:
: OpenID Foundation AuthZEN Working Group

Specification Document:
: This document.

## Security Event Type Identifiers Registry

This specification requests registration of the following event type in the IANA "Security Event Type Identifiers" registry established by {{RFC8417}}.

Name:
: AuthZEN Access Request Completed

URI:
: `urn:openid:authzen:event:access-request-completed`

Description:
: Indicates that an AuthZEN Access Request has reached a terminal status.

Change Controller:
: OpenID Foundation AuthZEN Working Group

Specification Document:
: This document.

# Examples

## End-to-End Manager Approval

### Initial Evaluation Request

~~~ http
POST /access/v1/evaluation HTTP/1.1
Host: pdp.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
Content-Type: application/json

{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "document",
    "id": "q4-plan"
  },
  "action": {
    "name": "can_read"
  },
  "context": {
    "time": "2026-04-30T20:15:00Z"
  }
}
~~~

### Requestable Denial

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "decision": false,
  "context": {
    "reason": "approval_required",
    "access_request": {
      "requestable": true,
      "template": "manager_approval",
      "reason": "manager_approval_required",
      "expires_in": 600,
      "required_fields": {
        "context": ["business_justification"]
      }
    }
  }
}
~~~

### Submitting the Access Request

~~~ http
POST /access/v1/requests HTTP/1.1
Host: pdp.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
Content-Type: application/json
Idempotency-Key: 7b8d0f0d-65a1-4af1-9fd3-a684f08a5d13

{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "document",
    "id": "q4-plan"
  },
  "action": {
    "name": "can_read"
  },
  "context": {
    "business_justification": "Needed for customer renewal review"
  },
  "denial": {
    "evaluation_id": "eval_01HX4Y2P8BQ4Y3F0V0K9D6Z7M1",
    "evaluated_at": "2026-04-30T20:15:00Z",
    "decision": {
      "decision": false,
      "context": {
        "reason": "approval_required"
      }
    },
    "access_request": {
      "requestable": true,
      "template": "manager_approval",
      "reason": "manager_approval_required"
    }
  }
}
~~~

### Task Handle

~~~ http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4

{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "pending",
    "status_endpoint": "https://pdp.example.com/access/v1/requests/arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "expires_at": "2026-04-30T23:00:00Z"
  }
}
~~~

### Completed Task

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "task": {
    "id": "arq_01HX4Y3AJZ7Y56W2F9H8Q8C1V4",
    "status": "approved"
  },
  "result": {
    "mode": "reevaluate",
    "approval": {
      "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
      "approved_until": "2026-05-01T00:42:00Z"
    },
    "approval_context": {
      "approval_id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH"
    }
  }
}
~~~

### Re-evaluation After Approval

~~~ http
POST /access/v1/evaluation HTTP/1.1
Host: pdp.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
Content-Type: application/json

{
  "subject": {
    "type": "user",
    "id": "alice@example.com"
  },
  "resource": {
    "type": "document",
    "id": "q4-plan"
  },
  "action": {
    "name": "can_read"
  },
  "context": {
    "time": "2026-04-30T20:43:00Z",
    "authzen_access_request_approval": {
      "approval_id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH"
    }
  }
}
~~~

### Final Decision

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "decision": true,
  "context": {
    "approval": {
      "id": "apr_01HX4Y8E2NE3Y2X7P0K4JE6WVH",
      "approved_until": "2026-05-01T00:42:00Z"
    }
  }
}
~~~

--- back

# Acknowledgements

TODO.

# Document History

-00
: Initial version defining requestable denials, the Access Request Endpoint, task handles, task completion, re-evaluation, the optional Shared Signals Framework binding, and registry additions.
