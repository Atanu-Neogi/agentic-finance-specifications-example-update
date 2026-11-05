# FinOS Agent Registry — Developer Guide

**Service:** `agent-registry`  
**API Version:** 1.0.0  
**Spec:** [`finos-agent-registry-api.yaml`](./finos-agent-registry-api.yaml)  
**Agent Card Schema:** [`finos-agent-card-schema.yaml`](./finos-agent-card-schema.yaml)  
**Base URL:** `https://api.example.com/finos/v1`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [FinOSAgentCard — What Gets Stored](#3-finosagentcard--what-gets-stored)
4. [Trust Tiers](#4-trust-tiers)
5. [Agent Lifecycle & Status](#5-agent-lifecycle--status)
6. [A2A Tier A Validation](#6-a2a-tier-a-validation)
7. [Filter Expression Syntax](#7-filter-expression-syntax)
8. [Pagination](#8-pagination)
9. [API Reference](#9-api-reference)
   - 9.1 [GET /health](#91-get-health)
   - 9.2 [GET /agent-registry/agents](#92-get-agent-registryagents)
   - 9.3 [GET /agent-registry/agents/{agentId}](#93-get-agent-registryagentsagentid)
   - 9.4 [POST /agent-registry/agents](#94-post-agent-registryagents)
   - 9.5 [PUT /agent-registry/agents/{agentId}](#95-put-agent-registryagentsagentid)
   - 9.6 [DELETE /agent-registry/agents/{agentId}](#96-delete-agent-registryagentsagentid)
10. [Error Reference](#10-error-reference)
11. [Integration Patterns](#11-integration-patterns)

---

## 1. Overview

The **FinOS Agent Registry** is the authoritative catalog of registered `FinOSAgent`
instances on the Blackline FinOS Orchestration Platform. It stores and serves
[FinOSAgentCard](./finos-agent-card-schema.yaml) documents — the A2A-native discovery
format — alongside registry-managed metadata such as lifecycle status, trust tier,
and tenant assignment.

**Key responsibilities:**

| Responsibility | Detail |
|---|---|
| **Registration** | Accept and validate FinOSAgentCard documents; enforce A2A Tier A rules |
| **Discovery** | Serve filterable, sortable, paginated agent lists for workflow engines and peer agents |
| **Lifecycle** | Track active → inactive → suspended → revoked status transitions |
| **Trust** | Assign and manage T0–T3 trust tiers controlling data access and HITL requirements |
| **Deregistration** | Soft-delete agents (status → `inactive`); retain records for audit |



---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Blackline FinOS Orchestration Platform                │
│                                                                         │
│  Callers                    Agent Registry Service                      │
│  ──────                     ───────────────────────                     |
│  WorkflowEngine             GET  /agent-registry/agents                 │
│  FinOSWorkflow              GET  /agent-registry/agents/{id}            │
│  Peer Agents        ──────▶ POST /agent-registry/agents                 |
│  Platform Ops               PUT  /agent-registry/agents/{id}            │
│  CI/CD Pipeline             DELETE /agent-registry/agents/{id}          │
│                                          │                              │
│                                 A2A Tier A Validation                   │
│                                 (on POST + card updates)                │
│                                          │                              │
│                                 ┌────────▼────────┐                     │
│                                 │                 │                     │
│                                 │                 │                     │
│                                 │ (agent-registry │                     │
│                                 │   database)     │                     │
│                                 └─────────────────┘                     │
│                                          │                              │
│                                 CDN Cache (list responses)              | 
└─────────────────────────────────────────────────────────────────────────┘
```

When the `FinOSWorkflow` engine encounters a `FinOSAgent` node at runtime, it:
1. Resolves `agentId` from the DAG node definition
2. Calls `GET /agent-registry/agents/{agentId}` to retrieve the live `FinOSAgentCard`
3. Extracts `supportedInterfaces[].url` as the A2A message delivery endpoint
4. Posts the A2A `Message` with `Message.metadata.finos.*` governance data

---

## 3. FinOSAgentCard — What Gets Stored

A registration stores two layers:

**1. Registry metadata** (managed by the service):

| Field | Source | Mutable |
|---|---|---|
| `agentId` | `extension.params.finOSAgentId` or generated | No |
| `name` | `agentCard.name` (denormalized) | Via card update |
| `provider` | `agentCard.provider.organization` (denormalized) | Via card update |
| `version` | `agentCard.version` (denormalized) | Via card update |
| `agentType` | `extension.params.finOSAgentType` (denormalized) | Via card update |
| `llmModel` | `extension.params.finOSLlmModel` (denormalized) | Via card update |
| `a2aEndpoint` | `agentCard.supportedInterfaces[0].url` (denormalized) | Via card update |
| `status` | Registry-managed | Yes (via PUT) |
| `trustTier` | Assigned at registration | Yes (via PUT, elevated perms) |
| `role` | Assigned at registration | Yes (via PUT) |
| `tenantId` | From request / caller JWT | No |
| `createdAt` | Server-set | No |
| `updatedAt` | Server-set on every write | No |

**2. Full FinOSAgentCard** (stored verbatim after validation):

The complete A2A card document, including:
- `capabilities.extensions[]` with FinOS extension params (`finOSAgentId`, `finOSAgentType`, `finOSLlmModel`, `finOSSupportedContextModes`, `finOSPoliciesRequired`)
- `skills[]` — what the agent can do
- `supportedInterfaces[]` — A2A protocol binding endpoints
- `securitySchemes` + `securityRequirements` — how callers must authenticate
- `defaultInputModes` / `defaultOutputModes`

---

## 4. Trust Tiers

Trust tiers are assigned at registration and govern data access, write permissions,
and HITL requirements across the FinOS platform.

| Tier | Label | Data Access | Write Permissions | HITL Requirement |
|---|---|---|---|---|
| `T0` | Platform Native | All data incl. RESTRICTED | Full write access | Not required (trusted) |
| `T1` | Certified Partner | CONFIDENTIAL and below | Write with policy gate | Required for RESTRICTED data |
| `T2` | Registered External | INTERNAL and below | Write to own namespace only | Required for CONFIDENTIAL+ |
| `T3` | Provisional | PUBLIC only | Read-only; no writes | HITL on all actions |

**Changing trust tier** requires the caller to hold the
`finos:agent-registry:trust:write` scope in their JWT.

---

## 5. Agent Lifecycle & Status

```
                    ┌──────────────────────────────────────┐
  POST (register)   │                                        │
       │            │                                        │
       ▼            │ PUT { status: active }                 │
  ┌─────────┐       │                                        │
  │ active  │◀──────┤◀─────── PUT { status: active } ───────│
  └─────────┘       │                                        │
      │  │          │                                        │
      │  │          │                                        │
   DELETE │         ▼                                        │
      │  │     ┌──────────┐                                 │
      │  └────▶│suspended │──── PUT { status: active } ─────┘
      │         └──────────┘
      │              │
      │         PUT { status: revoked }
      │              │
      ▼              ▼
  ┌──────────────────────┐
  │       inactive       │──── PUT { status: active } ─────▶ active
  └──────────────────────┘
              │
        PUT { status: revoked }
              │
              ▼
         ┌─────────┐
         │ revoked │  ← TERMINAL — cannot be reactivated
         └─────────┘
```

**Status behaviour in the list endpoint:**

| Status | Appears in `GET /agent-registry/agents` list? | Invocable? |
|---|---|---|
| `active` | Yes | Yes |
| `inactive` | No | No |
| `suspended` | No | No |
| `revoked` | No | No (ever) |

Use `GET /agent-registry/agents/{agentId}` directly to retrieve non-active agents.

---

## 6. A2A Tier A Validation

Every `POST /agent-registry/agents` request and every `PUT /agent-registry/agents/{agentId}` request that includes an
`agentCard` field undergoes **A2A Tier A Validation** before the agent is persisted.

The seven checks are run in order; all must pass for a `201/200` response:

| # | Check | What is Validated | Failure Code |
|---|---|---|---|
| 1 | **Schema** | All required `FinOSAgentCard` A2A fields present: `name`, `description`, `version`, `skills`, `supportedInterfaces`, `capabilities`, `defaultInputModes`, `defaultOutputModes` | `422` |
| 2 | **FinOS Extension** | `capabilities.extensions[]` contains an entry with `uri: "https://{vendor-domain}/finos/agent-card/v1"` and `required: true` | `422` |
| 3 | **Extension Params** | That extension entry's `params` includes `finOSAgentId` (uuid), `finOSAgentType` (valid enum), `finOSLlmModel` (non-empty string) | `422` |
| 4 | **Skills** | `skills[]` is non-empty; each skill has `id` and `name` | `422` |
| 5 | **Interfaces** | `supportedInterfaces[]` has ≥1 entry with a valid, reachable `url` | `422` |
| 6 | **Security** | `securitySchemes` non-empty; `securityRequirements` non-empty | `422` |
| 7 | **Uniqueness** | No active agent with the same `finOSAgentId` exists in the tenant | `409` |

**Failure response** (`422`) is an `A2AValidationError` with structured per-check results:

```json
{
  "type": "https://{api-host}/finos/errors/a2a-validation-failed",
  "title": "A2A Tier A Validation Failed",
  "status": 422,
  "detail": "2 validation check(s) failed.",
  "validationChecks": [
    { "check": "FinOS Extension", "passed": false, "message": "..." },
    { "check": "Extension Params", "passed": false, "message": "..." }
  ],
  "fieldErrors": [
    { "field": "agentCard.capabilities.extensions", "code": "missing-finos-extension", "message": "..." }
  ]
}
```

---

## 7. Filter Expression Syntax

The `filter` query parameter on `GET /agent-registry/agents` uses the
**FinOS standard filter expression** syntax.

### Grammar

```
filter_expr ::= condition ("and" condition)*
condition   ::= field operator '"' value '"'
field       ::= "name" | "provider" | "version" | "role" | "status"
operator    ::= "contains" | "eq"
```

### Operators

| Operator | Behaviour | Example |
|---|---|---|
| `contains` | Case-insensitive substring match | `name contains "match"` |
| `eq` | Exact equality (case-sensitive) | `status eq "active"` |

### Supported Fields

| Filter Field | Registry Field | Notes |
|---|---|---|
| `name` | `name` (denormalized from `agentCard.name`) | Use `contains` for partial matches |
| `provider` | `provider` (from `agentCard.provider.organization`) | Use `eq` for exact provider name |
| `version` | `version` (semver from `agentCard.version`) | Use `eq`; no semver range support |
| `role` | `role` | Values: `operational`, `monitoring` |
| `status` | `status` | Values: `active`, `inactive`, `suspended` |

> **Note:** `revoked` agents are always excluded from list results regardless of filter.

### Examples

```bash
# All active operational agents
?filter=status eq "active" and role eq "operational"

# All agents from a specific provider
?filter=provider eq "Example Corp"

# Agents whose name contains "match"
?filter=name contains "match"

# Compound: active first-party agents with "match" in name
?filter=provider eq "Example Corp" and name contains "match" and status eq "active"

# A specific version
?filter=version eq "2.1.0"
```

---

## 8. Pagination

The Agent Registry uses the **FinOS zero-based offset pagination** pattern.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `pageSize` | integer 1–100 | 20 | Results per page |
| `pageStartIndex` | integer ≥ 0 | 0 | Zero-based index of the first result |

**Page 1 (first 20):** `?pageSize=20&pageStartIndex=0`  
**Page 2 (next 20):** `?pageSize=20&pageStartIndex=20`  
**Page 3 (next 20):** `?pageSize=20&pageStartIndex=40`

**Response pagination metadata:**

```json
"meta": {
  "pagination": {
    "pageSize": 20,
    "pageStartIndex": 0,
    "totalCount": 47,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

## 9. API Reference

**Authentication:** All endpoints (except `/health`) require either:
- `Authorization: Bearer <JWT>` — OAuth 2.0 / OIDC JWT from the platform identity provider
- `X-FinOS-API-Key: <key>` — service-to-service API key

---

### 9.1 GET /health

Liveness check and connectivity probe.

**Authentication:** None required (exempt from auth for probe use).

```bash
curl https://api.example.com/finos/health
```

**200 Response — healthy:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "timestamp": "2026-05-18T09:00:00Z",
  "checks": {
    "liveness": { "status": "pass", "latencyMs": 1 },
    "db":  { "status": "pass", "latencyMs": 4,
                  "instance": "projects/acme-prod/instances/finos",
                  "database": "agent-registry" }
  }
}
```

**200 Response — degraded (DB latency elevated):**
```json
{
  "status": "degraded",
  "checks": {
    "liveness": { "status": "pass", "latencyMs": 1 },
    "db":  { "status": "fail", "latencyMs": 4200, "errorMessage": "Deadline exceeded" }
  }
}
```

---

### 9.2 GET /agent-registry/agents

List all non-revoked agents, with optional filter, sort, and pagination.

```bash
# List all active operational agents, newest first
curl "https://api.example.com/finos/v1/agent-registry/agents
  ?filter=status%20eq%20%22active%22%20and%20role%20eq%20%22operational%22
  &sort=createdAt
  &sortOrder=desc
  &pageSize=20
  &pageStartIndex=0" \
  -H "Authorization: Bearer $TOKEN"
```

**200 Response:**
```json
{
  "data": [
    {
      "agentId": "7f3d8c1a-0000-0000-0000-000000000001",
      "name": "TxnMatchAgent",
      "provider": "Example Corp",
      "version": "2.1.0",
      "role": "operational",
      "status": "active",
      "trustTier": "T0",
      "agentType": "EXECUTOR",
      "llmModel": "claude-sonnet-4-6",
      "tenantId": "tenant-acme-prod",
      "a2aEndpoint": "https://agents.example.com/txn-match/message:send",
      "wellKnownUrl": "https://agents.example.com/txn-match/.well-known/agent-card.json",
      "createdAt": "2026-01-15T08:00:00Z",
      "updatedAt": "2026-04-10T14:30:00Z"
    }
  ],
  "meta": {
    "requestId": "req-abc123",
    "timestamp": "2026-05-18T09:00:00Z",
    "apiVersion": "1.0",
    "pagination": {
      "pageSize": 20,
      "pageStartIndex": 0,
      "totalCount": 1,
      "hasNext": false,
      "hasPrev": false
    }
  }
}
```

> The `agentCard` full document is **not** included in list responses to keep
> payload size manageable. Use `GET /agent-registry/agents/{agentId}` to retrieve the full card.

---

### 9.3 GET /agent-registry/agents/{agentId}

Retrieve the full registration record for a single agent, including the complete `FinOSAgentCard` document.

Unlike the list endpoint, revoked agents **are** returned. Callers should check `status`.

```bash
curl "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN"
```

**200 Response** — includes full `agentCard`:
```json
{
  "data": {
    "agentId": "7f3d8c1a-0000-0000-0000-000000000001",
    "name": "TxnMatchAgent",
    "provider": "Example Corp",
    "version": "2.1.0",
    "role": "operational",
    "status": "active",
    "trustTier": "T0",
    "agentType": "EXECUTOR",
    "llmModel": "claude-sonnet-4-6",
    "supportedContextModes": ["RAG_QUERY", "STRUCTURED_CHUNKS"],
    "tenantId": "tenant-acme-prod",
    "a2aEndpoint": "https://agents.example.com/txn-match/message:send",
    "wellKnownUrl": "https://agents.example.com/txn-match/.well-known/agent-card.json",
    "agentCard": {
      "name": "TxnMatchAgent",
      "description": "Example Transaction Matching Agent.",
      "version": "2.1.0",
      "skills": [ { "id": "transaction-matching", "name": "Transaction Matching" } ],
      "capabilities": {
        "extensions": [
          {
            "uri": "https://{vendor-domain}/finos/agent-card/v1",
            "required": true,
            "params": {
              "finOSAgentId": "7f3d8c1a-0000-0000-0000-000000000001",
              "finOSAgentType": "EXECUTOR",
              "finOSLlmModel": "claude-sonnet-4-6"
            }
          }
        ]
      }
    },
    "createdAt": "2026-01-15T08:00:00Z",
    "updatedAt": "2026-04-10T14:30:00Z"
  },
  "meta": { "requestId": "req-def456", "timestamp": "2026-05-18T09:00:00Z", "apiVersion": "1.0" }
}
```

**404 Response:**
```json
{
  "type": "https://{api-host}/finos/errors/not-found",
  "title": "Agent Not Found",
  "status": 404,
  "detail": "No agent with ID '7f3d8c1a-...' exists."
}
```

---

### 9.4 POST /agent-registry/agents

Register a new FinOSAgent. A2A Tier A validation is applied before persistence.

```bash
curl -X POST \
  "https://api.example.com/finos/v1/agent-registry/agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: reg-txn-match-v210" \
  -d '{
    "agentId": "7f3d8c1a-0000-0000-0000-000000000001",
    "role": "operational",
    "trustTier": "T0",
    "tenantId": "tenant-acme-prod",
    "registeredBy": "platform-eng@blackline.com",
    "agentCard": {
      "name": "TxnMatchAgent",
      "description": "Example Transaction Matching Agent.",
      "version": "2.1.0",
      "provider": { "organization": "Example Corp", "url": "https://www.blackline.com" },
      "supportedInterfaces": [
        {
          "url": "https://agents.example.com/txn-match/message:send",
          "protocolBinding": "HTTP+JSON",
          "protocolVersion": "1.0"
        }
      ],
      "capabilities": {
        "streaming": false,
        "stateTransitionHistory": true,
        "extensions": [
          {
            "uri": "https://{vendor-domain}/finos/agent-card/v1",
            "required": true,
            "description": "FinOS agent governance extension",
            "params": {
              "finOSAgentId": "7f3d8c1a-0000-0000-0000-000000000001",
              "finOSAgentType": "EXECUTOR",
              "finOSLlmModel": "claude-sonnet-4-6",
              "finOSSupportedContextModes": ["RAG_QUERY", "STRUCTURED_CHUNKS"],
              "finOSPoliciesRequired": ["DATA_GOVERNANCE", "COMPLIANCE"],
              "finOSTenantId": "tenant-acme-prod"
            }
          }
        ]
      },
      "securitySchemes": {
        "bearerAuth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" }
      },
      "securityRequirements": [{ "bearerAuth": ["finos:agent:invoke"] }],
      "defaultInputModes": ["text", "data"],
      "defaultOutputModes": ["data"],
      "skills": [
        {
          "id": "transaction-matching",
          "name": "Transaction Matching",
          "description": "Matches transactions against accounting records to identify and resolve discrepancies.",
          "tags": ["transaction", "matching", "reconciliation"],
          "inputModes": ["data", "file"],
          "outputModes": ["data"]
        }
      ]
    }
  }'
```

**201 Response:**
```json
{
  "data": {
    "agentId": "7f3d8c1a-0000-0000-0000-000000000001",
    "name": "TxnMatchAgent",
    "status": "active",
    "trustTier": "T0",
    ...
  },
  "meta": { "requestId": "req-ghi789", "timestamp": "2026-05-18T09:00:01Z", "apiVersion": "1.0" }
}
```

**409 Response — duplicate agent:**
```json
{
  "type": "https://{api-host}/finos/errors/conflict",
  "title": "Agent Already Registered",
  "status": 409,
  "detail": "An active agent with ID '7f3d8c1a-...' already exists in tenant 'tenant-acme-prod'."
}
```

**422 Response — validation failure:**
```json
{
  "type": "https://{api-host}/finos/errors/a2a-validation-failed",
  "title": "A2A Tier A Validation Failed",
  "status": 422,
  "detail": "1 validation check(s) failed.",
  "validationChecks": [
    { "check": "Skills", "passed": false, "message": "agentCard.skills must be non-empty." }
  ]
}
```

---

### 9.5 PUT /agent-registry/agents/{agentId}

Partial update — only fields present in the request body are modified.

**Update status (suspend an agent):**
```bash
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "suspended" }'
```

**Elevate trust tier:**
```bash
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "trustTier": "T1" }'
```

**Update agent card to new version** (triggers full A2A Tier A re-validation):
```bash
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "agentCard": {
      "name": "TxnMatchAgent",
      "version": "2.2.0",
      "description": "Updated to support multi-currency transaction matching."
      ... (full updated FinOSAgentCard)
    }
  }'
```

**Permanently revoke** (terminal — cannot be undone):
```bash
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "revoked" }'
```

---

### 9.6 DELETE /agent-registry/agents/{agentId}

Soft-delete (deregister) an agent by setting its status to `inactive`.

The agent record is retained in DB. Existing in-flight executions are not interrupted.

```bash
curl -X DELETE \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN"
# → 204 No Content
```

**Reactivate after soft delete:**
```bash
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "active" }'
```

---

## 10. Error Reference

All errors follow **RFC 7807 Problem Details** (`application/problem+json`).

| HTTP Status | Error Type URI | When |
|---|---|---|
| `400` | `.../errors/bad-request` | Malformed JSON, invalid query parameter syntax |
| `401` | `.../errors/unauthorized` | Missing or expired credentials |
| `403` | `.../errors/forbidden` | Authenticated but insufficient scope |
| `404` | `.../errors/not-found` | Agent ID does not exist |
| `409` | `.../errors/conflict` | Duplicate `agentId`, invalid status transition, or update of a revoked agent |
| `422` | `.../errors/a2a-validation-failed` | A2A Tier A validation failure (see structured `validationChecks` array) |
| `503` | `.../errors/service-unavailable` | DB unavailable or critical dependency down |

**Filter syntax errors** return `400` with a `detail` explaining the parsing failure:
```json
{
  "status": 400,
  "title": "Invalid Filter Expression",
  "detail": "Unknown operator 'like' at position 5. Supported operators: contains, eq."
}
```

---

## 11. Integration Patterns

### Pattern A — CI/CD Agent Registration Pipeline

Register agents as part of a deployment pipeline:

```bash
# 1. Register (idempotent with Idempotency-Key)
curl -X POST "$REGISTRY/agent-registry/agents" \
  -H "Authorization: Bearer $CI_TOKEN" \
  -H "Idempotency-Key: deploy-${AGENT_NAME}-${VERSION}" \
  -d @agent-registration.json

# 2. If agent already exists (409), update the card version
curl -X PUT "$REGISTRY/agent-registry/agents/$AGENT_ID" \
  -H "Authorization: Bearer $CI_TOKEN" \
  -d "{ \"agentCard\": $(cat updated-agent-card.json) }"

# 3. On rollback, reactivate the previous version
curl -X PUT "$REGISTRY/agent-registry/agents/$PREV_AGENT_ID" \
  -d '{ "status": "active" }'
```

### Pattern B — Workflow Engine Agent Discovery

At workflow node execution time, the engine resolves the live agent endpoint:

```python
# Python pseudocode
def resolve_agent(agent_id: str) -> str:
    resp = requests.get(
        f"{REGISTRY}/agent-registry/agents/{agent_id}",
        headers={"Authorization": f"Bearer {token}"}
    )
    reg = resp.json()["data"]
    assert reg["status"] == "active", f"Agent {agentId} is {reg['status']}"
    return reg["a2aEndpoint"]   # → POST /message:send target
```

### Pattern C — Monitoring Agent Listing

A monitoring dashboard lists all active agents by type:

```bash
# All monitoring-role agents (MONITOR, REVIEWER)
curl "$REGISTRY/agent-registry/agents?filter=role%20eq%20%22monitoring%22&status=active&sort=name"

# All T3 provisional agents (for review)
curl "$REGISTRY/agent-registry/agents?filter=status%20eq%20%22active%22" | \
  jq '.data[] | select(.trustTier == "T3")'
```

---

## Appendix — Query Parameter Summary

| Parameter | Type | Default | Values | Description |
|---|---|---|---|---|
| `filter` | string | — | See §7 | FinOS filter expression |
| `sort` | string | `name` | `name` `provider` `version` `role` `createdAt` | Sort field |
| `sortOrder` | string | `asc` | `asc` `desc` | Sort direction |
| `pageSize` | integer | 20 | 1–100 | Results per page |
| `pageStartIndex` | integer | 0 | ≥ 0 | Zero-based offset |

## Appendix — FinOSAgentCard Required Fields (A2A Tier A)

| Field | Validation Rule |
|---|---|
| `name` | Non-empty string |
| `description` | Non-empty string |
| `version` | Non-empty string (semver recommended) |
| `skills[]` | Array, minItems: 1; each item: `id` + `name` |
| `supportedInterfaces[]` | Array, minItems: 1; each item: `url` + `protocolBinding` |
| `capabilities.extensions[]` | Must contain BlackLine FinOS entry (`uri` + `required: true` + `params`) |
| `capabilities.extensions[].params.finOSAgentId` | UUID |
| `capabilities.extensions[].params.finOSAgentType` | Valid `FinOSAgentType` enum |
| `capabilities.extensions[].params.finOSLlmModel` | Non-empty string |
| `securitySchemes` | Non-empty object |
| `securityRequirements[]` | Non-empty array |
| `defaultInputModes[]` | Non-empty array of `text \| data \| file` |
| `defaultOutputModes[]` | Non-empty array of `text \| data \| file \| stream` |
