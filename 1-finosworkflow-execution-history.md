# FinOSWorkflowExecutionHistory API

**Version:** `1.0.0`  
**Base path:** `$ROOT/finos/v1`  
**OpenAPI spec:** [`finos-workflow-execution-history-openapi.yaml`](./finos-workflow-execution-history-openapi.yaml)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Data Model](#2-data-model)
   - 2.1 [FinOSWorkflowExecutionHistory](#21-finosworkflowexecutionhistory)
   - 2.2 [FinOSVertex — Type Hierarchy](#22-finosvertex--type-hierarchy)
   - 2.3 [FinOSEvent & CloudEvent Compliance](#23-finosevent--cloudevent-compliance)
   - 2.4 [FinOSEdge — Edge Types](#24-finosedge--edge-types)
3. [Authentication](#3-authentication)
4. [Request & Response Conventions](#4-request--response-conventions)
5. [API Reference](#5-api-reference)
   - 5.1 [Execution History — Append-Only Operations](#51-execution-history--append-only-operations)
   - 5.2 [Vertex Operations](#52-vertex-operations)
   - 5.3 [Edge Operations](#53-edge-operations)
   - 5.4 [Graph Operations](#54-graph-operations)
6. [Error Reference](#6-error-reference)
7. [Usage Examples](#7-usage-examples)
8. [DAG Integrity Rules](#8-dag-integrity-rules)
9. [Status Lifecycle](#9-status-lifecycle)
10. [Immutability Guarantees](#10-immutability-guarantees)

---

## 1. Overview

`FinOSWorkflowExecutionHistory` is the core audit and provenance entity in the **Blackline Agentic AI Orchestration Platform (FinOS)**. It models the complete execution trace of a FinOS workflow as a **Directed Acyclic Graph (DAG)** of typed vertices and edges.

Each instance captures:
- **Who** acted (agents, humans, external systems)
- **What** happened (events, policy evaluations, approvals, tool calls)
- **How** things are related (typed edges encoding execution semantics)
- **When** each step occurred (timestamps on every node and edge)

The graph structure supports rich provenance queries: trace an approval back to its triggering agent event, find all exceptions linked to a workflow execution, or traverse the full decision path from a policy failure to its escalation.

**The API is append-only.** Vertices and edges can be read and appended but never replaced,
patched, or deleted, and the specification exposes no operation that would allow it. The single
mutating operation is `PATCH .../status`, a validated lifecycle transition on the instance
envelope. See [§10 Immutability Guarantees](#10-immutability-guarantees) for what this means for
corrections, archival, and retention.

```
FinOSWorkflowExecutionHistory (instanceId: UUID)
├── executionStatus: PENDING | RUNNING | COMPLETED | FAILED | CANCELLED | PAUSED | ARCHIVED
├── executionStartTime / executionEndTime
├── finOSVertices[]  ─── typed nodes (15 concrete types)
└── finOSEdges[]     ─── typed directed relations (18 types)
```

---

## 2. Data Model

### 2.1 FinOSWorkflowExecutionHistory

| Field | Type | Required | Notes |
|---|---|---|---|
| `instanceId` | `uuid` | R (server) | Globally unique. Server-generated if not provided. |
| `executionStatus` | `ExecutionStatus` | ✓ | See [status lifecycle](#9-status-lifecycle). |
| `executionStartTime` | `datetime` | ✓ | RFC 3339 UTC. |
| `executionEndTime` | `datetime` | — | Null while RUNNING. |
| `finOSVertices[]` | `FinOSVertex[]` | — | All DAG nodes. |
| `finOSEdges[]` | `FinOSEdge[]` | — | All DAG directed edges. |
| `vertexCount` | `integer` | R | Computed by server. |
| `edgeCount` | `integer` | R | Computed by server. |
| `tenantId` | `string` | — | Tenant identifier. |
| `createdAt` / `updatedAt` | `datetime` | R | Server-managed timestamps. |

### 2.2 FinOSVertex — Type Hierarchy

All vertex types share a **base schema** (`FinOSVertexBase`) and are discriminated by the `vertexType` field:

```
FinOSVertexBase
  ├── WorkflowDefinition     Workflow template definition
  ├── WorkflowVersion        A specific semver of a workflow definition
  ├── Execution              A triggered execution instance
  ├── NodeInstance           A single workflow node run
  ├── EdgeEvaluation         Condition evaluation on a DAG edge
  ├── FinOSEvent  ──────────── (extends CloudEvent v1.0)
  │     ├── FinOSAgentEvent   LLM agent action (model, tools, skills, HITL)
  │     ├── FinOSAPIEvent     External API call
  │     ├── FinOSHITLEvent    Human-in-the-loop task/decision
  │     └── FinOSPolicyEvent  Policy evaluation with outcome
  ├── FinOSPolicy            Policy definition referenced in evaluation
  ├── ApprovalTier           Tier in a multi-level approval chain
  ├── Artifact               Generated document, report, file, or model
  ├── Dataset                Input/output dataset snapshot
  ├── Exception              Runtime exception or error
  └── ExternalSystem         External system called (DB, API, queue, etc.)
```

**Base vertex fields (all types):**

| Field | Type | Notes |
|---|---|---|
| `vertexId` | `uuid` | Required. Unique within the DAG. |
| `vertexType` | `FinOSVertexType` | Required. Discriminator. |
| `label` | `string` | Human-readable name for rendering. |
| `finOSEdgeIds[]` | `uuid[]` | Server-populated. IDs of all incident edges. |
| `properties` | `object` | Free-form metadata key-value pairs. |
| `createdAt` / `updatedAt` | `datetime` | Server-managed. |

### 2.3 FinOSEvent & CloudEvent Compliance

All `FinOSEvent`-family vertices fully conform to the [**CloudEvents v1.0 specification**](https://github.com/cloudevents/spec). The required CloudEvent attributes are present on every event vertex:

| CloudEvent Attribute | Type | Required | Description |
|---|---|---|---|
| `id` | `string` | ✓ | Event ID. Unique per `source`. |
| `source` | `uri-reference` | ✓ | URI identifying the event producer. |
| `specversion` | `string` | ✓ | Always `"1.0"`. |
| `type` | `string` | ✓ | Reverse-DNS event type (e.g. `{event-namespace}.finos.agent.classify`). |
| `datacontenttype` | `string` | — | MIME type of `data` (e.g. `application/json`). |
| `dataschema` | `uri` | — | Schema URI for `data`. |
| `subject` | `string` | — | Event subject within the source context. |
| `time` | `datetime` | — | RFC 3339 occurrence timestamp. |
| `data` | `any` | — | Domain-specific event payload. |

**FinOS extensions** added on top of CloudEvent:

| FinOS Extension | Type | Description |
|---|---|---|
| `finOSEventType` | `FinOSEventType` | Sub-type discriminator (`FinOSAgentEvent` etc.) |
| `workflowInstanceId` | `uuid` | Parent execution history `instanceId`. |
| `correlationId` | `uuid` | Distributed tracing correlation ID. |
| `tenantId` | `string` | Tenant identifier. |
| `traceId` | `string` | OpenTelemetry W3C TraceContext trace ID. |
| `spanId` | `string` | OpenTelemetry span ID. |

**FinOSAgentEvent** additional fields:

| Field | Type | Description |
|---|---|---|
| `agent` | `Agent` | Agent identity (ID, name, type, version). |
| `agentRun` | `AgentRun` | Run metadata including token usage and timing. |
| `llmModelName` | `string` | LLM model name (e.g. `claude-sonnet-4-6`). |
| `llmModelVersion` | `string` | Model version string. |
| `promptTemplate` | `string` | Prompt template ID or content. |
| `toolCall` | `ToolCall` | Tool invocation details (name, input, output, latency). |
| `skills[]` | `Skill[]` | Active skills during this event. |
| `humanTask` | `HumanTask` | HITL task generated, if any. |
| `humanDecision` | `HumanDecision` | Human decision recorded, if any. |
| `evidence` | `Evidence` | Evidence artifact, if captured. |

**FinOSPolicyEvent** additional fields:

| Field | Type | Description |
|---|---|---|
| `finOSPolicyId` | `uuid` | Policy that was evaluated. |
| `finOSPolicyEvaluationOutcome` | `enum` | `Success` \| `Escalate` \| `AttentionRequired` \| `Failed` |
| `targetEscalationRole` | `string` | Role to escalate to on `Escalate` / `AttentionRequired`. |
| `evaluationInputs` | `object` | Input context fed to the policy evaluator. |
| `evaluationOutputs` | `object` | Raw output from the evaluator. |

### 2.4 FinOSEdge — Edge Types

Every edge is a directed relation `sourceVertex ──[edgeType]──▶ targetVertex`.

| Edge Type | Semantic Meaning |
|---|---|
| `EXECUTES` | A workflow definition/version executes an execution or node |
| `CONTAINS_NODE` | An execution or parent node contains a child node instance |
| `FOLLOWED_BY` | Temporal/sequential ordering between two nodes |
| `EVALUATED_EDGE` | An edge evaluation result gates the traversal to the target |
| `USED_POLICY` | A node used the target policy during evaluation |
| `CALLED_TOOL` | An agent event called the target external system or tool |
| `READ_OBJECT` | A node read the target dataset or artifact |
| `PROPOSED_ACTION_ON` | An agent proposed an action on the target object |
| `WAITED_FOR` | A node paused waiting for the target node/event |
| `APPROVED_BY` | A proposed action was approved by the target approval tier or human |
| `REJECTED_BY` | A proposed action was rejected by the target approval tier or human |
| `OVERRIDDEN_BY` | A decision or policy outcome was overridden by the target human decision |
| `GENERATED_ARTIFACT` | A node generated the target artifact |
| `TRIGGERED_BY` | A node was triggered by the target event |
| `RESUMED_BY` | A paused node was resumed by the target event or decision |
| `POSTED_TO` | An event or result was posted to the target external system |
| `LINKED_TO_EXCEPTION` | A node or edge is linked to the target exception |
| `DERIVED_FROM` | An artifact or dataset was derived from the target artifact/dataset |

---

## 3. Authentication

All endpoints require one of:

| Method | Header | Notes |
|---|---|---|
| **Bearer JWT** | `Authorization: Bearer <token>` | OAuth 2.0 / OIDC JWT from the platform identity provider. Preferred. |
| **API Key** | `X-FinOS-API-Key: <key>` | For service-to-service machine clients. |

```bash
# JWT example
curl -H "Authorization: Bearer $TOKEN" \
  https://api.example.com/finos/v1/workflow-executions

# API key example
curl -H "X-FinOS-API-Key: $API_KEY" \
  https://api.example.com/finos/v1/workflow-executions
```

---

## 4. Request & Response Conventions

### Content Types

| Scenario | Content-Type |
|---|---|
| `POST` (append an instance, vertex, or edge) | `application/json` |
| `PATCH .../status` (lifecycle transition) | `application/json` |
| Error responses | `application/problem+json` (RFC 7807) |

This API is append-only, so there is no full-replace or merge-patch body type — see
[§10 Immutability Guarantees](#10-immutability-guarantees).

### Response Envelope

All successful responses use a consistent envelope:

```json
{
  "data": { ... },
  "meta": {
    "requestId": "a1b2c3d4-...",
    "timestamp": "2026-05-18T09:00:00Z",
    "apiVersion": "1.0"
  }
}
```

List responses include pagination metadata:

```json
{
  "data": [ ... ],
  "meta": {
    "requestId": "...",
    "timestamp": "...",
    "pagination": {
      "pageSize": 20,
      "pageStartIndex": 0,
      "totalCount": 142,
      "hasNext": true,
      "hasPrev": false
    }
  }
}
```

### Pagination

All FinOS list endpoints use the same zero-based offset pattern:

| Param | Type | Default | Description |
|---|---|---|---|
| `pageSize` | integer 1–200 | 20 | Results per page |
| `pageStartIndex` | integer ≥ 0 | 0 | Zero-based index of the first result |

**Page 1 (first 50):** `?pageSize=50&pageStartIndex=0`
**Page 2 (next 50):** `?pageSize=50&pageStartIndex=50`
**Page 3 (next 50):** `?pageSize=50&pageStartIndex=100`

The request parameters and the `meta.pagination` response fields use the same names, so a client
can echo the values it sent straight back from the response.

### Filtering

List endpoints accept the FinOS standard `filter` expression:

```
filter=field operator "value" [and field operator "value"]
```

| Operator | Meaning |
|---|---|
| `contains` | Case-insensitive substring match |
| `eq` | Exact equality |

Supported fields on execution histories: `instanceId`, `workflowId`, `executionStatus`,
`tenantId`, `triggeredBy`.

```
?filter=executionStatus eq "FAILED"
?filter=tenantId eq "tenant-acme-prod" and executionStatus eq "RUNNING"
```

### Idempotency

`POST` requests accept an optional `Idempotency-Key` header. Resubmitting with the same key within 24 hours returns the original response without creating a duplicate.

```bash
curl -X POST \
  -H "Idempotency-Key: my-unique-key-001" \
  -H "Content-Type: application/json" \
  -d '{ ... }' \
  https://api.example.com/finos/v1/workflow-executions
```

---

## 5. API Reference

### 5.1 Execution History — Append-Only Operations

> **This API is append-only.** An execution history is a SOX audit record. Once a vertex or edge
> is written it cannot be replaced, patched, or deleted, and there is no endpoint that would let a
> caller do so. The only supported writes are `POST` (append a new instance, vertex, or edge) and
> `PATCH .../status` (a validated lifecycle transition on the instance itself). See
> [§10 Immutability Guarantees](#10-immutability-guarantees).

#### `GET /finos/v1/workflow-executions`
List all execution histories with optional filtering.

**Query parameters:**

| Param | Type | Description |
|---|---|---|
| `executionStatus` | `ExecutionStatus` | Filter by lifecycle status. |
| `fromDate` | `datetime` | Start of date range (inclusive). |
| `toDate` | `datetime` | End of date range (inclusive). |
| `filter` | `string` | FinOS filter expression, e.g. `executionStatus eq "FAILED"`. |
| `sort` | `string` | `executionStartTime` \| `executionEndTime` \| `executionStatus` \| `instanceId`. Default `executionStartTime`. |
| `sortOrder` | `string` | `asc` \| `desc`. Default `desc`. |
| `pageSize` | `integer` | Results per page (1–200). Default 20. |
| `pageStartIndex` | `integer` | Zero-based start index. Default 0. |

---

#### `POST /finos/v1/workflow-executions`
Create a new execution history. Returns `201 Created` with `Location` header.

---

#### `GET /finos/v1/workflow-executions/{instanceId}`
Retrieve a specific instance. Use `?embed=none|vertices|edges|full` to control payload size.

---

#### `PATCH /finos/v1/workflow-executions/{instanceId}/status`
The only mutating operation on an existing instance: a validated lifecycle transition
(for example `RUNNING → COMPLETED`). It changes `executionStatus`, `executionEndTime`, and the
recorded `reason` only — it never alters an existing vertex or edge. To retire an instance without
destroying its evidence, transition it to `ARCHIVED`.

```json
{
  "executionStatus": "COMPLETED",
  "executionEndTime": "2026-05-18T09:42:00Z",
  "reason": "All approval tiers cleared"
}
```

---

### 5.2 Vertex Operations

#### `GET /finos/v1/workflow-executions/{instanceId}/vertices`
List vertices. Filter by `?vertexType=FinOSAgentEvent`.

#### `POST /finos/v1/workflow-executions/{instanceId}/vertices`
Append a vertex to the DAG. Body must include the `vertexType` discriminator. A vertex is
write-once: correcting a recorded fact means appending a new vertex that supersedes it, not
editing the original.

#### `GET /finos/v1/workflow-executions/{instanceId}/vertices/{vertexId}`
Retrieve a single vertex.

#### `GET .../vertices/{vertexId}/edges`
Returns all edges connected to a vertex.

| Param | Values | Default |
|---|---|---|
| `direction` | `outgoing` \| `incoming` \| `both` | `both` |
| `edgeType` | any `FinOSEdgeType` | — |

---

### 5.3 Edge Operations

#### `GET /finos/v1/workflow-executions/{instanceId}/edges`
List edges. Filter by `?edgeType=APPROVED_BY`, `?sourceVertexId=...`, `?targetVertexId=...`.

#### `POST /finos/v1/workflow-executions/{instanceId}/edges`
Append a directed edge. Validates that both vertex IDs exist and that no cycle is introduced.
Like vertices, edges are write-once.

#### `GET .../edges/{edgeId}`
Retrieve a single edge.

#### `GET .../edges/{edgeId}/vertices`
Returns the source and target vertices for an edge:

```json
{
  "data": {
    "sourceVertex": { ... },
    "targetVertex": { ... }
  }
}
```

---

### 5.4 Graph Operations

#### `GET .../graph`
Returns the full DAG in one of three formats:

| `?format=` | Description |
|---|---|
| `adjacency` (default) | Per-vertex adjacency list with outgoing edges |
| `edgelist` | Flat list of `(source, edge, target)` triples |
| `cytoscape` | [Cytoscape.js](https://cytoscape.org/) elements format for direct rendering |

#### `POST .../graph/traverse`
Traversal from a starting vertex.

```json
{
  "startVertexId": "a1b2c3d4-...",
  "algorithm": "BFS",
  "maxDepth": 5,
  "direction": "outgoing",
  "edgeTypeFilter": ["EXECUTES", "FOLLOWED_BY", "APPROVED_BY"],
  "vertexTypeFilter": ["NodeInstance", "FinOSAgentEvent", "ApprovalTier"]
}
```

Returns an ordered `visitOrder[]` array with depth, vertex, and the edge traversed to reach it.

#### `GET .../graph/path?sourceVertexId=...&targetVertexId=...`
Shortest directed path between two vertices. Returns `404` if no path exists.

---

## 6. Error Reference

All errors follow **RFC 7807 Problem Details**:

```json
{
  "type": "https://{api-host}/finos/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "FinOSWorkflowExecutionHistory with instanceId '3fa8...' not found.",
  "instance": "{source-root}/finos/v1/workflow-executions/3fa8...",
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "timestamp": "2026-05-18T09:00:00Z"
}
```

Field-level validation errors (422) include an `errors[]` array:

```json
{
  "status": 422,
  "title": "Validation Failed",
  "errors": [
    { "field": "executionStartTime", "code": "required", "message": "This field is required." },
    { "field": "finOSEdges[0].sourceVertexId", "code": "not-found", "message": "Vertex ID does not exist in this execution history." }
  ]
}
```

| HTTP Status | Meaning |
|---|---|
| `400` | Malformed request body or invalid query parameters |
| `401` | Missing or invalid credentials |
| `403` | Authenticated but not authorised |
| `404` | Resource not found |
| `409` | Conflict — duplicate ID, DAG cycle detected, or invalid state transition |
| `422` | Schema validation failed |
| `500` | Unexpected server error |

---

## 7. Usage Examples

### 7.1 Create a new execution history

```bash
curl -X POST https://api.example.com/finos/v1/workflow-executions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "executionStartTime": "2026-05-18T09:00:00Z",
    "executionStatus": "RUNNING",
    "tenantId": "tenant-acme-prod"
  }'
```

**Response `201 Created`:**
```json
{
  "data": {
    "instanceId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "executionStatus": "RUNNING",
    "executionStartTime": "2026-05-18T09:00:00Z",
    "executionEndTime": null,
    "finOSVertices": [],
    "finOSEdges": [],
    "vertexCount": 0,
    "edgeCount": 0,
    "tenantId": "tenant-acme-prod",
    "createdAt": "2026-05-18T09:00:01Z",
    "updatedAt": "2026-05-18T09:00:01Z"
  },
  "meta": { "requestId": "abc123", "timestamp": "2026-05-18T09:00:01Z", "apiVersion": "1.0" }
}
```

---

### 7.2 Add a FinOSAgentEvent vertex

```bash
curl -X POST \
  https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../vertices \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vertexId": "11111111-0000-0000-0000-000000000001",
    "vertexType": "FinOSAgentEvent",
    "label": "Invoice classification by LLM",
    "id": "evt-invoice-classify-001",
    "source": "{source-root}/finos/agent/invoice-classifier",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.agent.classify",
    "time": "2026-05-18T09:00:10Z",
    "subject": "invoice/INV-2026-00441",
    "finOSEventType": "FinOSAgentEvent",
    "workflowInstanceId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "tenantId": "tenant-acme-prod",
    "correlationId": "corr-7890abcd-ef12-3456-7890-abcdef123456",
    "llmModelName": "claude-sonnet-4-6",
    "llmModelVersion": "20250514",
    "agent": {
      "agentId": "agt-99990000-0000-0000-0000-000000000001",
      "agentName": "InvoiceClassifierAgent",
      "agentType": "EXECUTOR",
      "agentVersion": "1.2.0"
    },
    "agentRun": {
      "runId": "run-aaaabbbb-0000-0000-0000-000000000001",
      "runStatus": "COMPLETED",
      "startTime": "2026-05-18T09:00:08Z",
      "endTime": "2026-05-18T09:00:10Z",
      "tokenUsage": {
        "inputTokens": 1240,
        "outputTokens": 340,
        "totalTokens": 1580,
        "estimatedCostUsd": 0.0047
      }
    },
    "skills": [
      {
        "skillId": "sk-00000001-0000-0000-0000-000000000001",
        "skillName": "InvoiceReadSkill",
        "skillVersion": "1.0.0",
        "skillType": "DocumentExtraction"
      }
    ]
  }'
```

---

### 7.3 Add a FinOSPolicyEvent vertex

```bash
curl -X POST \
  https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../vertices \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vertexId": "22222222-0000-0000-0000-000000000002",
    "vertexType": "FinOSPolicyEvent",
    "label": "Spending limit policy check",
    "id": "evt-policy-spending-001",
    "source": "{source-root}/finos/policy/spending-limit",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.policy.evaluate",
    "time": "2026-05-18T09:00:12Z",
    "finOSEventType": "FinOSPolicyEvent",
    "workflowInstanceId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "tenantId": "tenant-acme-prod",
    "finOSPolicyId": "pol-spending-limit-v3",
    "finOSPolicyEvaluationOutcome": "Escalate",
    "targetEscalationRole": "finance-manager",
    "evaluationInputs": {
      "invoiceAmount": 48500.00,
      "currency": "USD",
      "vendorTier": "B"
    }
  }'
```

---

### 7.4 Add an APPROVED_BY edge

```bash
curl -X POST \
  https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../edges \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "edgeId": "edge0001-0000-0000-0000-000000000001",
    "edgeType": "APPROVED_BY",
    "sourceVertexId": "22222222-0000-0000-0000-000000000002",
    "targetVertexId": "33333333-0000-0000-0000-000000000003",
    "label": "Policy escalation approved by Finance Manager",
    "timestamp": "2026-05-18T10:15:00Z"
  }'
```

---

### 7.5 Get all outgoing edges from a vertex

```bash
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../vertices/22222222-.../edges?direction=outgoing" \
  -H "Authorization: Bearer $TOKEN"
```

---

### 7.6 Traverse the DAG from the workflow definition

```bash
curl -X POST \
  https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../graph/traverse \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startVertexId": "a1b2c3d4-0000-0000-0000-000000000001",
    "algorithm": "BFS",
    "maxDepth": 10,
    "direction": "outgoing",
    "edgeTypeFilter": ["EXECUTES", "FOLLOWED_BY", "APPROVED_BY", "GENERATED_ARTIFACT"]
  }'
```

**Truncated response:**
```json
{
  "data": {
    "startVertexId": "a1b2c3d4-...",
    "algorithm": "BFS",
    "visitOrder": [
      {
        "depth": 0,
        "vertex": { "vertexType": "WorkflowDefinition", "label": "Invoice Approval Workflow", "..." },
        "traversedViaEdge": null
      },
      {
        "depth": 1,
        "vertex": { "vertexType": "FinOSAgentEvent", "label": "Invoice classification by LLM", "..." },
        "traversedViaEdge": { "edgeType": "EXECUTES", "..." }
      }
    ],
    "visitedVertexCount": 7,
    "visitedEdgeCount": 6
  }
}
```

---

### 7.7 Update execution status to COMPLETED

```bash
curl -X PATCH \
  https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../status \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "executionStatus": "COMPLETED",
    "executionEndTime": "2026-05-18T10:30:00Z",
    "reason": "All three approval tiers cleared. Invoice INV-2026-00441 approved."
  }'
```

---

### 7.8 Get the full DAG in Cytoscape format for rendering

```bash
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../graph?format=cytoscape" \
  -H "Authorization: Bearer $TOKEN"
```

---

### 7.9 Find the path between a policy event and its approval

```bash
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../graph/path?sourceVertexId=22222222-...&targetVertexId=33333333-..." \
  -H "Authorization: Bearer $TOKEN"
```

---

### 7.10 List only HITL event vertices

```bash
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64-.../vertices?vertexType=FinOSHITLEvent&pageSize=50" \
  -H "Authorization: Bearer $TOKEN"
```

---

## 8. DAG Integrity Rules

The API enforces the following invariants on every write operation:

| Rule | Description |
|---|---|
| **No cycles** | Adding an edge is rejected (`409`) if it would create a cycle. Validated via DFS cycle detection. |
| **Vertex existence** | Edge endpoints must reference existing vertices in the same execution history. |
| **Type endpoint consistency** | Certain edge types enforce vertex type constraints. For example, `APPROVED_BY` target must be `ApprovalTier` or `FinOSHITLEvent`. |
| **Write-once records** | Vertices and edges cannot be replaced, patched, or deleted after they are appended. The API exposes no operation that would do so. |
| **Status-gated writes** | Vertices and edges cannot be added to a history with status `COMPLETED`, `FAILED`, or `ARCHIVED`. Use `PATCH /status` to reopen if permitted. |

---

## 9. Status Lifecycle

```
                         ┌──────────────────────────────────────┐
                         │                                        │
   ┌─────────┐  start    ├──────────┐  complete  ┌───────────┐   │
   │ PENDING │──────────▶│ RUNNING  │────────────▶│ COMPLETED │   │
   └─────────┘           └──────────┘             └───────────┘   │
                              │                                    │
                        fail  │  pause                            │
                              │                                    │
                    ┌─────────┴──────────┐                       │
                    │                    │                        │
               ┌────▼────┐         ┌────▼────┐                  │
               │ FAILED  │         │ PAUSED  │──resume──▶ RUNNING│
               └─────────┘         └─────────┘                  │
                    │                    │                        │
               cancel│              cancel│                       │
                    ▼                    ▼                        │
               ┌──────────────────────────┐                      │
               │        CANCELLED         │                       │
               └──────────────────────────┘                      │
                              │                                   │
                         archive│                                 │
                              ▼                                   │
                         ┌──────────┐                             │
                         │ ARCHIVED │◀────────────────────────────┘
                         └──────────┘        (from any terminal state)
```

| Transition | Allowed From | Notes |
|---|---|---|
| → `RUNNING` | `PENDING`, `PAUSED` | `executionStartTime` set on first transition. |
| → `COMPLETED` | `RUNNING` | `executionEndTime` required. |
| → `FAILED` | `RUNNING` | `executionEndTime` and `reason` recommended. |
| → `PAUSED` | `RUNNING` | Can be resumed. |
| → `CANCELLED` | `PENDING`, `RUNNING`, `PAUSED`, `FAILED` | Terminal. |
| → `ARCHIVED` | Any terminal state | Retires the instance. Data retained for audit; nothing is erased. |

---

## 10. Immutability Guarantees

An execution history is evidence. The API is shaped so that evidence cannot be rewritten, and the
contract — not just convention — is what enforces it.

### What the API exposes

| Operation | Available | Why |
|---|---|---|
| `GET` any resource | Yes | Reading never changes the record. |
| `POST` instance / vertex / edge | Yes | Appending is how the DAG grows. |
| `PATCH .../{instanceId}/status` | Yes | Lifecycle transition on the instance envelope. It touches `executionStatus`, `executionEndTime`, and `reason` only. |
| `PUT` / `PATCH` an instance, vertex, or edge | **No** | Would allow a recorded fact to be rewritten. |
| `DELETE` an instance, vertex, or edge | **No** | Would allow evidence to be destroyed. |

There is no admin scope, header, or query parameter that re-enables the absent operations. A
conforming implementation that adds them is no longer conforming.

### Correcting a record

Because nothing can be edited, a correction is itself an audit event: append a new vertex that
supersedes the earlier one and link it with an edge. The original stays in the DAG, and the
correction is visible as a correction — which is the property an auditor is checking for.

### Retiring a record

`ARCHIVED` is a lifecycle state, not a delete. The instance stops accepting new vertices and edges
and drops out of default list results, but every vertex and edge it holds remains readable.

### Retention and erasure

Retention expiry and statutory erasure (for example a GDPR Article 17 request) are deliberately
outside this API. They are operator actions governed by a retention policy, executed through the
storage layer under separate authorisation and with their own audit trail — not something a holder
of an ordinary API token can trigger.
