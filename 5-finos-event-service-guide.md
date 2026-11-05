# FinOS Event Service — Developer Guide

**Service:** `event-service`  
**API Version:** 1.0.0  
**Spec:** [`finos-event-service-api.yaml`](./finos-event-service-api.yaml)  
**Base URL:** `https://api.example.com/event-service/v1`  
**Aligned with:** [`finos-workflow-execution-history-openapi.yaml`](./finos-workflow-execution-history-openapi.yaml) · [`finos-workflow-api.yaml`](./finos-workflow-api.yaml)

---

## Table of Contents

1. [Overview](#1-overview)
   - 1.1 [Namespace Placeholders](#11-namespace-placeholders)
2. [FinOSEvent Schema](#2-finosevent-schema)
   - 2.1 [CloudEvent Base Fields](#21-cloudevent-base-fields)
   - 2.2 [FinOS Extension Fields](#22-finos-extension-fields)
   - 2.3 [New Optional Fields](#23-new-optional-fields)
   - 2.4 [FinOSAgentEvent](#24-finosagentevent)
   - 2.5 [FinOSAPIEvent](#25-finosapievent)
   - 2.6 [FinOSHITLEvent](#26-finoshitlevent)
   - 2.7 [FinOSPolicyEvent](#27-finospolicyevent)
3. [Publishing APIs](#3-publishing-apis)
   - 3.1 [Use case (a) — Standalone event push](#31-use-case-a--standalone-event-push)
   - 3.2 [Use case (b) — Workflow-context event push](#32-use-case-b--workflow-context-event-push)
   - 3.3 [Use case (f-a) — Re-publish standalone event](#33-use-case-f-a--re-publish-standalone-event)
   - 3.4 [Use case (f-b) — Re-publish workflow-context event](#34-use-case-f-b--re-publish-workflow-context-event)
4. [Subscription APIs](#4-subscription-apis)
   - 4.1 [Use case (c) — Subscribe by FinOSEventType](#41-use-case-c--subscribe-by-finoseventtype)
   - 4.2 [Use case (d) — Subscribe by workflow ID](#42-use-case-d--subscribe-by-workflow-id)
   - 4.3 [Use case (e) — Subscribe by workflow ID + execution ID](#43-use-case-e--subscribe-by-workflow-id--execution-id)
   - 4.4 [Use case (f-c) — Re-register by-event-type subscription](#44-use-case-f-c--re-register-by-event-type-subscription)
   - 4.5 [Use case (f-d) — Re-register by-workflow subscription](#45-use-case-f-d--re-register-by-workflow-subscription)
   - 4.6 [Use case (f-e) — Re-register by-execution subscription](#46-use-case-f-e--re-register-by-execution-subscription)
5. [Event Delivery](#5-event-delivery)
   - 5.1 [Delivery Envelope](#51-delivery-envelope)
   - 5.2 [HMAC Signature Verification](#52-hmac-signature-verification)
   - 5.3 [Retry Behaviour](#53-retry-behaviour)
6. [Schema Alignment with Execution History](#6-schema-alignment-with-execution-history)
7. [Workflow Engine Integration](#7-workflow-engine-integration)
8. [Error Reference](#8-error-reference)
9. [Appendix — CloudEvent Type Registry](#9-appendix--cloudevent-type-registry)

---

## 1. Overview

The **FinOS Event Service** (`event-service`) is the central event bus for the Blackline
FinOS Orchestration Platform. It provides:

| Capability | Detail |
|---|---|
| **Event Publishing** | Push any `FinOSEvent` subtype — standalone or in the context of a running workflow execution |
| **Event Subscriptions** | Register a `callbackUrl` to receive matching events via HTTPS POST (webhook push model) |
| **Re-publishing** | Replace and re-dispatch a previously published event |
| **Re-registration** | Fully replace a subscription's callback URL, filters, or signing secret |

**Relationship to `FinOSWorkflowExecutionHistory`:**

The event service and the execution history API serve complementary roles:

| Service | Role |
|---|---|
| **Event Service** (`event-service`) | Fanout routing — publish once, deliver to N subscribers via push |
| **Execution History** (`FinOSWorkflowExecutionHistory`) | Persistent audit DAG — every event is a vertex in the provenance graph |

A `FinOSEvent` published to the event service is routed to subscribers. The same event
(or a corresponding vertex) should also be recorded in `FinOSWorkflowExecutionHistory`
for long-term audit — the engine emits both as part of each node transition (see [§7](#7-workflow-engine-integration)).

### 1.1 Namespace Placeholders

CloudEvents `type` and `source` are **wire identifiers**: consumers filter and subscribe on
them, so they must be stable and must belong to a namespace the publisher actually controls.
This specification therefore does not fix a vendor namespace. Examples use placeholders that an
implementation substitutes once, consistently, across every event it emits.

| Placeholder | Substitute with | Example substitution |
|---|---|---|
| `{event-namespace}` | A reverse-DNS prefix your organisation owns, used to root CloudEvents `type` | `com.example` |
| `{source-root}` | A path-form root for CloudEvents `source`, identifying the emitting platform | `/example` |
| `{vendor-domain}` | An HTTPS domain you control, used to root the A2A extension URI | `example.com` |
| `{api-host}` | The host serving the FinOS APIs, used to root RFC 9457 problem `type` URIs | `api.example.com` |

So `{event-namespace}.finos.agent.classify` becomes `com.example.finos.agent.classify`, and
`{source-root}/finos/agent/invoice-classifier` becomes `/example/finos/agent/invoice-classifier`.

**The segment after the namespace is normative.** `.finos.{domain}.{action}` for `type`, and
`/finos/{domain}/{component}` for `source`, are fixed by this specification — only the namespace
root is yours to choose. That is what lets a consumer written against one implementation
recognise events from another by suffix matching.

Substitution is a one-time deployment decision. Changing a namespace after agents are emitting
events breaks every subscription filtering on the old prefix.

---

## 2. FinOSEvent Schema

Every `FinOSEvent` is a valid **CloudEvents v1.0** document extended with FinOS
governance fields. The `finOSEventType` field is the discriminator for the four concrete subtypes.

### 2.1 CloudEvent Base Fields

All events carry the required CloudEvents v1.0 attributes:

| Field | Required | Description | Example |
|---|---|---|---|
| `id` | ✓ | Unique event ID per `source`. Used as idempotency key. | `"evt-classify-001"` |
| `source` | ✓ | URI identifying the event producer. | `"{source-root}/finos/agent/invoice-classifier"` |
| `specversion` | ✓ | Always `"1.0"`. | `"1.0"` |
| `type` | ✓ | Reverse-DNS event type. FinOS convention: `{event-namespace}.finos.{domain}.{action}` | `"{event-namespace}.finos.agent.classify"` |
| `datacontenttype` | — | MIME type of `data`. | `"application/json"` |
| `dataschema` | — | URI of the schema for `data`. | |
| `subject` | — | Subject of the event in producer context. | `"invoice/INV-2026-00441"` |
| `time` | — | RFC 3339 occurrence timestamp. | `"2026-05-18T09:00:10Z"` |
| `data` | — | Domain-specific event payload. | |

### 2.2 FinOS Extension Fields

FinOS adds governance extensions on top of CloudEvent (aligned with `FinOSEventBase`
in `finos-workflow-execution-history-openapi.yaml`):

| Field | Required | Description |
|---|---|---|
| `finOSEventType` | ✓ | Subtype discriminator: `FinOSAgentEvent` · `FinOSAPIEvent` · `FinOSHITLEvent` · `FinOSPolicyEvent` |
| `tenantId` | ✓ | Tenant identifier. |
| `workflowInstanceId` | Contextual* | Execution instance ID. Required for use case (b); optional for (a). |
| `correlationId` | — | UUID for cross-service distributed tracing. |
| `traceId` | — | OpenTelemetry W3C TraceContext trace ID. |
| `spanId` | — | OpenTelemetry span ID. |

*`workflowInstanceId` is injected automatically from the path when using the workflow-context endpoint (use case b).

### 2.3 New Optional Fields

The event service spec adds the following optional fields to `FinOSEventCore`
(not present in `FinOSEventBase` in the execution history spec):

| Field | Type | Description |
|---|---|---|
| `workflowId` | `string` | Workflow **definition** ID (the execution history only captures `workflowInstanceId`). |
| `nodeId` | `string` | Workflow DAG node that emitted this event. |
| `sessionId` | `string` | Multi-turn agent session identifier. |
| `priority` | `enum` | `LOW` · `MEDIUM` · `HIGH` · `CRITICAL` — delivery ordering hint. |
| `expiresAt` | `datetime` | Event TTL; the service stops retrying delivery after this time. |

### 2.4 FinOSAgentEvent

`finOSEventType: FinOSAgentEvent` · type prefix: `{event-namespace}.finos.agent.*`

Additional fields (all optional unless noted):

| Field | Description |
|---|---|
| `agent` | Agent identity: `agentId`, `agentName`, `agentType`, `agentVersion` |
| `agentRun` | Run metadata: `runId`, `runStatus`, `startTime`, `endTime`, `tokenUsage` |
| `llmModelName` | LLM model name (e.g. `claude-sonnet-4-6`) |
| `llmModelVersion` | Model version string |
| `promptTemplate` | Prompt template ID or content |
| `toolCalls[]` | Array of `ToolCall` (name, input, output, latency, status) |
| `skills[]` | Active `Skill` objects |
| `humanTask` | `HumanTask` generated by this event (if HITL was triggered) |
| `humanDecision` | `HumanDecision` recorded (if a previous HITL was resolved) |
| `evidence` | `Evidence` artifact captured |
| `agentCardVersion` | **NEW** — FinOSAgentCard version in effect |
| `contextModes[]` | **NEW** — Context graph modes used: `RAG_QUERY` · `RAG_TENSOR` · `FINOSWORKFLOW_GRAPH` · `STRUCTURED_CHUNKS` |
| `a2aProtocolVersion` | **NEW** — A2A protocol version (e.g. `"1.0"`) |

### 2.5 FinOSAPIEvent

`finOSEventType: FinOSAPIEvent` · type prefix: `{event-namespace}.finos.api.*`

| Field | Description |
|---|---|
| `endpoint` | URI of the called API endpoint |
| `httpMethod` | `GET` · `POST` · `PUT` · `PATCH` · `DELETE` |
| `requestHeaders` / `requestBody` | Request (sensitive headers redacted) |
| `responseStatusCode` / `responseHeaders` / `responseBody` | Response |
| `latencyMs` | Round-trip latency in milliseconds |
| `retryAttempt` | Zero-based retry count |
| `retryStrategy` | **NEW** — `EXPONENTIAL_BACKOFF` · `FIXED_DELAY` · `NONE` |
| `cacheHit` | **NEW** — Whether response was served from cache |
| `authScheme` | **NEW** — `OIDC` · `API_KEY` · `BEARER` · `NONE` |

### 2.6 FinOSHITLEvent

`finOSEventType: FinOSHITLEvent` · type prefix: `{event-namespace}.finos.hitl.*`

| Field | Description |
|---|---|
| `assignedTo` | User or role assigned to the task |
| `taskType` | `REVIEW` · `APPROVE` · `REJECT` · `ESCALATE` · `ACKNOWLEDGE` · `OVERRIDE` |
| `taskDescription` | Human-readable task instructions |
| `dueDate` | Task due date |
| `completedAt` | When the task was completed |
| `decision` | `APPROVED` · `REJECTED` · `ESCALATED` · `ACKNOWLEDGED` · `DEFERRED` |
| `justification` | Reviewer's justification text |
| `attachments[]` | URIs of supporting documents |
| `escalationChain[]` | Ordered escalation path |
| `callbackUrl` | **NEW** — FinOS Workflows callback URL for workflow resume |
| `timeoutAt` | **NEW** — When the HITL task expires (raises `TimeoutError`) |
| `notificationChannels[]` | **NEW** — Channels notified: `"email"`, `"slack"`, `"task-portal"` |
| `priorityLevel` | **NEW** — `LOW` · `MEDIUM` · `HIGH` · `CRITICAL` |

### 2.7 FinOSPolicyEvent

`finOSEventType: FinOSPolicyEvent` · type prefix: `{event-namespace}.finos.policy.*`

| Field | Required | Description |
|---|---|---|
| `finOSPolicyId` | ✓ | FinOSPolicy identifier |
| `finOSPolicyEvaluationOutcome` | ✓ | `Success` · `Escalate` · `AttentionRequired` · `Failed` |
| `targetEscalationRole` | — | Role to escalate to (when outcome = Escalate) |
| `evaluationInputs` | — | Input context supplied to the evaluator |
| `evaluationOutputs` | — | Raw output from the evaluator |
| `evaluatedAt` | — | Evaluation timestamp |
| `policyCategory` | — | **NEW** — Policy category (denormalized from FinOSPolicy record) |
| `policyVersion` | — | **NEW** — Semantic version of the policy evaluated |
| `overrideApplied` | — | **NEW** — Whether a human override was applied |
| `overrideReason` | — | **NEW** — Justification for the override |

---

## 3. Publishing APIs

**Base URL:** `https://api.example.com/event-service/v1`

### 3.1 Use case (a) — Standalone event push

Push a FinOSEvent with no required workflow context:

```bash
curl -X POST https://api.example.com/event-service/v1/events \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: evt-classify-001" \
  -d '{
    "id": "evt-classify-001",
    "source": "{source-root}/finos/agent/invoice-classifier",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.agent.classify",
    "time": "2026-05-18T09:00:10Z",
    "subject": "invoice/INV-2026-00441",
    "finOSEventType": "FinOSAgentEvent",
    "tenantId": "tenant-acme-prod",
    "correlationId": "corr-7890abcd-ef12-3456-7890-abcdef123456",
    "priority": "HIGH",
    "llmModelName": "claude-sonnet-4-6",
    "agent": {
      "agentId": "agt-00000001",
      "agentName": "InvoiceClassifierAgent",
      "agentType": "CLASSIFIER",
      "agentVersion": "1.2.0"
    },
    "agentRun": {
      "runId": "run-aaaabbbb",
      "runStatus": "COMPLETED",
      "startTime": "2026-05-18T09:00:08Z",
      "endTime": "2026-05-18T09:00:10Z",
      "tokenUsage": {
        "inputTokens": 1240, "outputTokens": 340, "totalTokens": 1580
      }
    }
  }'
# → 201 Created  Location: /event-service/v1/events/evt-classify-001
```

**Deduplication:** if an event with the same `id` + `source` is re-posted within the
deduplication window (configurable, default 24 h), the service returns `200` with the
original event rather than creating a duplicate.

### 3.2 Use case (b) — Workflow-context event push

Push a FinOSEvent scoped to a specific workflow execution. The `workflowId` and `executionId`
path parameters are authoritative — they override any values in the request body:

```bash
# Emitted by the FinOS Workflows Engine after a node transition (see §7)
curl -X POST \
  "https://api.example.com/event-service/v1/workflows/wf-invoice-approval-v1/executions/inst-abc123/events" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Workflow-Instance-Id: inst-abc123" \
  -d '{
    "id": "evt-policy-spending-001",
    "source": "{source-root}/finos/policy/spending-limit",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.policy.evaluate",
    "time": "2026-05-18T09:00:12Z",
    "finOSEventType": "FinOSPolicyEvent",
    "tenantId": "tenant-acme-prod",
    "nodeId": "node-spending-policy",
    "finOSPolicyId": "pol-spending-limit-v3",
    "finOSPolicyEvaluationOutcome": "Escalate",
    "targetEscalationRole": "finance-manager",
    "policyVersion": "3.0.0",
    "evaluationInputs": {
      "invoiceAmount": 48500.00, "currency": "USD", "vendorTier": "B"
    }
  }'
```

The service automatically sets:
- `workflowId = "wf-invoice-approval-v1"` (from path)
- `workflowInstanceId = "inst-abc123"` (from path)

This event is then routed to:
1. All subscriptions with `filter.eventTypes` including `FinOSPolicyEvent`
2. All subscriptions with `filter.workflowId = "wf-invoice-approval-v1"`
3. All subscriptions with `filter.workflowId = "wf-invoice-approval-v1"` and `filter.workflowExecutionId = "inst-abc123"`

### 3.3 Use case (f-a) — Re-publish standalone event

Replace and re-dispatch a standalone event. Use when the original event had errors or
when new subscribers need to receive a historical event:

```bash
curl -X PUT https://api.example.com/event-service/v1/events/evt-classify-001 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "evt-classify-001",
    "source": "{source-root}/finos/agent/invoice-classifier",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.agent.classify",
    "time": "2026-05-18T09:00:10Z",
    "finOSEventType": "FinOSAgentEvent",
    "tenantId": "tenant-acme-prod",
    "llmModelName": "claude-sonnet-4-6",
    "agentCardVersion": "1.2.0",
    "agent": { "agentId": "agt-00000001", "agentName": "InvoiceClassifierAgent" }
  }'
# → 200 OK — event replaced and re-dispatched to active subscribers
```

**Immutable after creation:** `finOSEventType`, `source`. All other fields may be updated.

### 3.4 Use case (f-b) — Re-publish workflow-context event

```bash
curl -X PUT \
  "https://api.example.com/event-service/v1/workflows/wf-invoice-approval-v1/executions/inst-abc123/events/evt-policy-spending-001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ ... updated FinOSPolicyEvent body ... }'
```

---

## 4. Subscription APIs

### 4.1 Use case (c) — Subscribe by FinOSEventType

Register to receive **all events** of given `FinOSEventType` values across the tenant,
regardless of which workflow produced them:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-event-type \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service.example.com/webhooks/finos",
    "filter": {
      "eventTypes": ["FinOSAgentEvent", "FinOSPolicyEvent"],
      "sourcePattern": "{source-root}/finos/*"
    },
    "deliveryConfig": {
      "secret": "my-hmac-secret-at-least-16-chars",
      "maxRetries": 3,
      "timeoutMs": 5000
    },
    "expiresAt": "2027-01-01T00:00:00Z"
  }'
# → 201 Created  Location: /event-service/v1/subscriptions/by-event-type/{subscriptionId}
```

### 4.2 Use case (d) — Subscribe by workflow ID

Register to receive all events from **any execution** of a specific workflow definition:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-workflow \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service.example.com/webhooks/workflow-monitor",
    "filter": {
      "workflowId": "wf-invoice-approval-v1",
      "eventTypes": ["FinOSHITLEvent", "FinOSPolicyEvent"]
    },
    "deliveryConfig": {
      "secret": "workflow-monitor-secret",
      "maxRetries": 5,
      "timeoutMs": 10000
    }
  }'
```

Receives events from ALL execution instances of `wf-invoice-approval-v1`. The optional
`eventTypes` filter narrows delivery to specific subtypes.

### 4.3 Use case (e) — Subscribe by workflow ID + execution ID

Register to receive all events from a **specific execution instance** of a workflow:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-execution \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service.example.com/webhooks/execution-trace",
    "filter": {
      "workflowId": "wf-invoice-approval-v1",
      "workflowExecutionId": "inst-abc123"
    },
    "deliveryConfig": {
      "secret": "execution-trace-secret",
      "maxRetries": 3,
      "timeoutMs": 5000
    }
  }'
```

This subscription automatically transitions to `EXPIRED` when the execution instance
reaches a terminal state (`COMPLETED`, `FAILED`, `CANCELLED`).

### 4.4 Use case (f-c) — Re-register by-event-type subscription

Fully replace a by-event-type subscription (e.g., to rotate the HMAC secret or add event types):

```bash
# Rotate the signing secret and add FinOSHITLEvent to the filter
curl -X PUT \
  "https://api.example.com/event-service/v1/subscriptions/by-event-type/sub-uuid-001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service.example.com/webhooks/finos",
    "filter": {
      "eventTypes": ["FinOSAgentEvent", "FinOSPolicyEvent", "FinOSHITLEvent"]
    },
    "deliveryConfig": {
      "secret": "new-rotated-hmac-secret",
      "maxRetries": 3,
      "timeoutMs": 5000
    }
  }'
# → 200 OK
```

Re-registration is **atomic**: in-flight deliveries against the old subscription complete
before the new configuration takes effect. No events are lost.

### 4.5 Use case (f-d) — Re-register by-workflow subscription

```bash
# Retarget to a different workflow version and change callback URL
curl -X PUT \
  "https://api.example.com/event-service/v1/subscriptions/by-workflow/sub-uuid-002" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service-v2.example.com/webhooks/workflow",
    "filter": {
      "workflowId": "wf-invoice-approval-v2",
      "eventTypes": ["FinOSHITLEvent"]
    },
    "deliveryConfig": {
      "secret": "new-secret", "maxRetries": 3, "timeoutMs": 8000
    }
  }'
```

### 4.6 Use case (f-e) — Re-register by-execution subscription

```bash
# Retarget to a new execution instance (e.g. failed run restarted)
curl -X PUT \
  "https://api.example.com/event-service/v1/subscriptions/by-execution/sub-uuid-003" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://my-service.example.com/webhooks/execution-trace",
    "filter": {
      "workflowId": "wf-invoice-approval-v1",
      "workflowExecutionId": "inst-def456"
    },
    "deliveryConfig": {
      "secret": "execution-trace-secret", "maxRetries": 3, "timeoutMs": 5000
    }
  }'
```

---

## 5. Event Delivery

### 5.1 Delivery Envelope

When an event matches a subscription, the event service sends an HTTP POST to the
subscriber's `callbackUrl` with the following structure:

**Request headers:**
```
Content-Type:              application/json
X-FinOS-Delivery-Id:       {uuid}
X-FinOS-Subscription-Id:   {subscriptionId}
X-FinOS-Event-Type:        {finOSEventType}
X-FinOS-Signature:         sha256={hmac}      ← present when secret is configured
```

**Request body (`FinOSEventDelivery`):**
```json
{
  "deliveryId": "del-uuid-001",
  "subscriptionId": "sub-uuid-001",
  "attempt": 1,
  "deliveredAt": "2026-05-18T09:00:15Z",
  "eventId": "evt-classify-001",
  "event": {
    "id": "evt-classify-001",
    "source": "{source-root}/finos/agent/invoice-classifier",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.agent.classify",
    "finOSEventType": "FinOSAgentEvent",
    "tenantId": "tenant-acme-prod",
    "workflowId": "wf-invoice-approval-v1",
    "workflowInstanceId": "inst-abc123",
    ...
  }
}
```

The subscriber **must respond with HTTP 2xx** within `deliveryConfig.timeoutMs` (default 5,000 ms)
to acknowledge receipt. Any non-2xx response or timeout triggers a retry.

### 5.2 HMAC Signature Verification

When `deliveryConfig.secret` is set, verify the delivery as follows:

**Python:**
```python
import hmac
import hashlib

def verify_finos_delivery(request_body: bytes, secret: str, signature_header: str) -> bool:
    """
    Verify the X-FinOS-Signature header on a FinOS event delivery.
    signature_header format: "sha256={hex_hmac}"
    """
    if not signature_header.startswith("sha256="):
        return False
    expected_sig = signature_header[len("sha256="):]
    computed_sig = hmac.new(
        key=secret.encode("utf-8"),
        msg=request_body,
        digestmod=hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(computed_sig, expected_sig)
```

**Node.js:**
```javascript
const crypto = require('crypto');

function verifyFinOSDelivery(rawBody, secret, signatureHeader) {
  const [algo, sig] = signatureHeader.split('=');
  if (algo !== 'sha256') return false;
  const computed = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(sig));
}
```

> Always use a **timing-safe comparison** (`hmac.compare_digest` / `timingSafeEqual`)
> to prevent timing-based side-channel attacks.

### 5.3 Retry Behaviour

| Condition | Behaviour |
|---|---|
| `callbackUrl` returns 2xx | Delivery acknowledged; no retry |
| `callbackUrl` returns 4xx (client error) | **Not retried** — a 4xx signals the subscriber rejected the event; investigate the payload |
| `callbackUrl` returns 5xx or timeout | Retried with exponential backoff up to `maxRetries` |
| All retries exhausted | Event marked as delivery-failed; stored in the dead-letter queue (DLQ); notification sent |
| Event `expiresAt` reached | No further retries; event expired |

Default delivery retry schedule with `maxRetries: 3` and `retryBackoffMs: 2000`:

| Attempt | Delay before attempt |
|---|---|
| 1 | Immediate |
| 2 | 2 s |
| 3 | 4 s |
| 4 | 8 s |

---

## 6. Schema Alignment with Execution History

The `FinOSEventCore` schema in this spec is designed for **forward compatibility** with
`FinOSEventBase` and the concrete vertex schemas in
`finos-workflow-execution-history-openapi.yaml`.

| This spec | Execution History spec | Note |
|---|---|---|
| `FinOSEventCore` | `FinOSEventBase` | Identical fields; `workflowInstanceId` optional here, required there |
| `FinOSEvent` (union) | `FinOSVertex` (union) | Vertex adds `vertexId`, `vertexType`, `label`, `finOSEdgeIds` |
| `FinOSAgentEvent` | `FinOSAgentEventVertex` | Identical event fields; vertex adds vertex base fields |
| `FinOSAPIEvent` | `FinOSAPIEventVertex` | Identical |
| `FinOSHITLEvent` | `FinOSHITLEventVertex` | Identical |
| `FinOSPolicyEvent` | `FinOSPolicyEventVertex` | Identical |
| `FinOSEventType` | `FinOSEventType` | Exact same enum |
| `FinOSPolicyEvaluationOutcome` | `FinOSPolicyEvaluationOutcome` | Exact same enum |
| All supporting schemas | Same schemas | `Agent`, `AgentRun`, `Skill`, `ToolCall`, `HumanTask`, `HumanDecision`, `Evidence` |

**Workflow pattern:** A single event is typically published in two places:

1. **Event service** (this API) — for fan-out delivery to live subscribers
2. **Execution history** — as a vertex in the workflow's provenance DAG

Both accept the same `finOSEventType`, CloudEvent fields, and FinOS extension fields.

---

## 7. Workflow Engine Integration

The FinOS Workflows Engine emits one `FinOSEvent` per node transition. This emission is
performed by the engine itself as part of executing the workflow — it is **not** a node a
workflow author declares, and it does not appear anywhere in the workflow definition
(`inputs` / `nodes` / `edges`). Authors never wire up event publication; they get it by
running on a conforming engine.

For each transition the engine POSTs to the workflow-context endpoint
(`POST /workflows/{workflowId}/executions/{executionId}/events`), deriving the body from the
node that just ran:

| Event field | Derived from |
|---|---|
| `id` | `{instanceId}-{nodeId}-{nodeStatus}` — stable, so redelivery is idempotent |
| `source` | The emitting component, as a URI reference: `{source-root}/finos/{domain}/{component}` |
| `specversion` | Always `"1.0"` |
| `type` | `{event-namespace}.finos.{domain}.{action}` — see [§1.1](#11-namespace-placeholders) |
| `time` | RFC 3339 timestamp of the transition |
| `finOSEventType` | The node type that produced it — `FinOSAgentEvent`, `FinOSAPIEvent`, `FinOSHITLEvent`, or `FinOSPolicyEvent` |
| `tenantId` | The run's tenant |
| `nodeId` | The node that just completed |
| `correlationId` | The workflow instance ID, so every event in a run correlates |

A representative emission for a completed `FinOSAgent` node:

```json
{
  "id": "inst-abc123-node-classify-invoice-COMPLETED",
  "source": "{source-root}/finos/agent/invoice-classifier",
  "specversion": "1.0",
  "type": "{event-namespace}.finos.agent.classify",
  "time": "2026-05-18T09:00:10Z",
  "finOSEventType": "FinOSAgentEvent",
  "tenantId": "tenant-acme-prod",
  "workflowId": "wf-invoice-approval-v1",
  "workflowInstanceId": "inst-abc123",
  "nodeId": "node-classify-invoice",
  "correlationId": "inst-abc123",
  "properties": { "nodeStatus": "COMPLETED" }
}
```

`workflowId` and `workflowInstanceId` are taken from the path and overwrite whatever the body
carries, so an engine cannot misattribute an event to another run.

**Delivery guarantees.** Emission is at-least-once. Because `id` is deterministic for a given
`(instance, node, status)`, a replayed event is a no-op for any consumer that de-duplicates on
`id` — which the Execution History service does when it persists the event as a vertex.

**Subscription pattern for workflow monitoring:**

```python
# Register to receive all HITL and Policy events from invoice-approval workflow
import requests

requests.post(
    "https://api.example.com/event-service/v1/subscriptions/by-workflow",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "callbackUrl": "https://my-monitor.example.com/finos-events",
        "filter": {
            "workflowId": "wf-invoice-approval-v1",
            "eventTypes": ["FinOSHITLEvent", "FinOSPolicyEvent"]
        },
        "deliveryConfig": {
            "secret": "my-secret-key-for-hmac-verification",
            "maxRetries": 3,
            "timeoutMs": 8000
        }
    }
)
```

---

## 8. Error Reference

| Status | Type URI | When |
|---|---|---|
| `400` | `.../errors/bad-request` | Malformed JSON or invalid `specversion` |
| `401` | `.../errors/unauthorized` | Missing or expired credentials |
| `403` | `.../errors/forbidden` | Authenticated but insufficient tenant scope |
| `404` | `.../errors/not-found` | Event or subscription not found |
| `409` | `.../errors/conflict` | Cancelling an already-cancelled subscription |
| `422` | `.../errors/validation-failed` | Missing required `finOSEventType`, invalid field format |
| `503` | `.../errors/service-unavailable` | Datastore or message queue unavailable |

**Example 422 — missing required CloudEvent field:**
```json
{
  "type": "https://api.example.com/event-service/errors/validation-failed",
  "title": "Event Validation Failed",
  "status": 422,
  "detail": "CloudEvent required fields are missing.",
  "errors": [
    { "field": "source", "code": "required", "message": "'source' is required per CloudEvents v1.0." },
    { "field": "finOSEventType", "code": "required", "message": "'finOSEventType' is required." }
  ]
}
```

---

## 9. Appendix — CloudEvent Type Registry

FinOS reverse-DNS event type strings by subtype:

### FinOSAgentEvent (`{event-namespace}.finos.agent.*`)

| `type` value | Description |
|---|---|
| `{event-namespace}.finos.agent.classify` | Invoice or document classification |
| `{event-namespace}.finos.agent.extract` | Data extraction from a document |
| `{event-namespace}.finos.agent.match` | Transaction or record matching |
| `{event-namespace}.finos.agent.summarize` | Summarization task |
| `{event-namespace}.finos.agent.review` | Agent-driven review task |
| `{event-namespace}.finos.agent.post` | ERP / system-of-record post (read-only agent, via API node) |
| `{event-namespace}.finos.agent.llm-call` | Raw LLM invocation |

### FinOSAPIEvent (`{event-namespace}.finos.api.*`)

| `type` value | Description |
|---|---|
| `{event-namespace}.finos.api.call` | Generic external API call |
| `{event-namespace}.finos.api.erp.get` | ERP data retrieval |
| `{event-namespace}.finos.api.erp.post` | ERP data write (via FinOSAPI node) |
| `{event-namespace}.finos.api.vendor.get` | Vendor data retrieval |

### FinOSHITLEvent (`{event-namespace}.finos.hitl.*`)

| `type` value | Description |
|---|---|
| `{event-namespace}.finos.hitl.task.created` | HITL task created; workflow paused |
| `{event-namespace}.finos.hitl.task.completed` | Reviewer submitted decision; workflow resumed |
| `{event-namespace}.finos.hitl.task.escalated` | Task escalated after timeout |
| `{event-namespace}.finos.hitl.task.overridden` | Policy override applied by reviewer |
| `{event-namespace}.finos.hitl.task.expired` | Task timed out with no decision |

### FinOSPolicyEvent (`{event-namespace}.finos.policy.*`)

| `type` value | Description |
|---|---|
| `{event-namespace}.finos.policy.evaluate` | Policy evaluated; outcome returned |
| `{event-namespace}.finos.policy.override` | Policy evaluation overridden |
| `{event-namespace}.finos.policy.escalate` | Escalation triggered by policy outcome |

---

## Appendix — Use Case Index

| Use case | Method | Path |
|---|---|---|
| **(a)** Push standalone event | `POST` | `/event-service/v1/events` |
| **(b)** Push workflow-context event | `POST` | `/event-service/v1/workflows/{workflowId}/executions/{executionId}/events` |
| **(c)** Register by event type | `POST` | `/event-service/v1/subscriptions/by-event-type` |
| **(d)** Register by workflow ID | `POST` | `/event-service/v1/subscriptions/by-workflow` |
| **(e)** Register by workflow + execution ID | `POST` | `/event-service/v1/subscriptions/by-execution` |
| **(f-a)** Re-publish standalone event | `PUT` | `/event-service/v1/events/{eventId}` |
| **(f-b)** Re-publish workflow-context event | `PUT` | `/event-service/v1/workflows/{workflowId}/executions/{executionId}/events/{eventId}` |
| **(f-c)** Re-register by-event-type subscription | `PUT` | `/event-service/v1/subscriptions/by-event-type/{subscriptionId}` |
| **(f-d)** Re-register by-workflow subscription | `PUT` | `/event-service/v1/subscriptions/by-workflow/{subscriptionId}` |
| **(f-e)** Re-register by-execution subscription | `PUT` | `/event-service/v1/subscriptions/by-execution/{subscriptionId}` |
