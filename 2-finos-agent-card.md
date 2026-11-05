# Blackline FinOS Agent Card

**Schema Version:** `1.0.0`
**OpenAPI Spec:** [`finos-agent-card-schema.yaml`](./finos-agent-card-schema.yaml)
**A2A Protocol:** [Agent-to-Agent (A2A) Specification](https://a2a-protocol.org/latest/specification/)
**Outcome conforms to:** `FinOSAgentEventVertex` in `finos-workflow-execution-history-openapi.yaml`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [A2A Protocol Integration](#3-a2a-protocol-integration)
4. [Schema Reference](#4-schema-reference)
   - 4.1 [FinOSAgentCard — A2A Discovery Document](#41-finosagentcard--a2a-discovery-document)
   - 4.2 [AgentInterface](#42-agentinterface)
   - 4.3 [SecurityScheme & Security Requirements](#43-securityscheme--security-requirements)
   - 4.4 [AgentProvider](#44-agentprovider)
   - 4.5 [AgentExtension](#45-agentextension)
   - 4.6 [FinOSAgentExtensionParams](#46-finosagentextensionparams)
     - 4.6.1 [FinOSLlmModelEntry — Model Roster Entry](#461-finosllmmodelentry--model-roster-entry)
     - 4.6.2 [FinOSAgentIOSchema — I/O Schema Container](#462-finosagentioschema--io-schema-container)
     - 4.6.3 [FinOSAgentPortDeclaration — Port Declaration](#463-finosagentportdeclaration--port-declaration)
   - 4.7 [FinOSMessageMetadata — Invocation Governance](#47-finosmessagemetadata--invocation-governance)
   - 4.8 [Calling Context](#48-calling-context)
   - 4.9 [Additional User Prompt](#49-additional-user-prompt)
   - 4.10 [Context Graphs](#410-context-graphs)
   - 4.11 [FinOS Policies](#411-finos-policies)
   - 4.12 [Variable Bindings](#412-variable-bindings)
   - 4.13 [Session Context Store Reference](#413-session-context-store-reference)
   - 4.14 [Outcome — FinOSAgentCardOutcome](#414-outcome--finosagentcardoutcome)
5. [Context Graph Modes](#5-context-graph-modes)
   - 5.1 [RAG_QUERY](#51-rag_query)
   - 5.2 [RAG_TENSOR](#52-rag_tensor)
   - 5.3 [FINOSWORKFLOW_GRAPH](#53-finosworkflow_graph)
   - 5.4 [STRUCTURED_CHUNKS](#54-structured_chunks)
6. [Outcome Population Contract](#6-outcome-population-contract)
7. [Policy Enforcement Lifecycle](#7-policy-enforcement-lifecycle)
8. [Variable Binding Contract](#8-variable-binding-contract)
9. [Agent I/O Contract](#9-agent-io-contract)
10. [API Reference](#10-api-reference)
11. [Security Model](#11-security-model)
12. [Usage Examples](#12-usage-examples)

---

## 1. Overview

The **FinOS Agent Card** is the A2A-compatible discovery document for Blackline FinOS AI Agents, served at `/.well-known/agent-card.json`. It extends the standard A2A Agent Card with FinOS-specific governance metadata carried in `capabilities.extensions[]`.

Agent invocation uses standard A2A `Message` objects, with FinOS governance data (calling context, context graphs, policies, variables, session store) carried in `Message.metadata.finos.*`.

| Concern | A2A Mechanism | FinOS Extension |
|---|---|---|
| **Discovery** | `GET /.well-known/agent-card.json` | `capabilities.extensions[]` with FinOS params |
| **Invocation** | `POST /message:send` (A2A Message) | `Message.metadata.finos.*` (FinOSMessageMetadata) |
| **Task lifecycle** | A2A Task states (SUBMITTED → WORKING → ...) | FinOSAgentEventVertex outcome |
| **Governance** | — | Policies, variables, session store in metadata |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Caller: WorkflowEngine | Agent | Service | Human       │
│                                                         │
│  1. Discover agent: GET /.well-known/agent-card.json    │
│  2. Send A2A Message: POST /message:send                │
│     → Message.parts[] = user prompt + data              │
│     → Message.metadata.finos = FinOSMessageMetadata     │
└────────────────────────────────┬────────────────────────┘
                                 │  A2A Message
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  FinOS Agent Runtime (A2A-compatible)                   │
│                                                         │
│  1. Validate FinOS metadata:                            │
│     • callingContext present and authorized             │
│     • All finOS policies pass pre-execution evaluation  │
│     • Required input variables present                  │
│                                                         │
│  1. Resolve context graphs:                             │
│     RAG_QUERY  → issue vector store query               │
│     RAG_TENSOR → load tensor from storage URI           │
│     FINOS_GRAPH → fetch DAG subgraph, serialize         │
│     STRUCT_CHUNKS → inject directly                     │
│                                                         │
│  3. Assemble context window (rank-ordered)              │
│                                                         │
│  4. Read sessionContextStore (storeKey bindings)        │
│                                                         │
│  5. Execute LLM reasoning loop                          │
│     → tool calls gated by skill.permittedTools          │
│     → HITL pause → A2A INPUT_REQUIRED state             │
│                                                         │
│  6. Populate output variables →                         │
│     write to sessionContextStore                        │
│                                                         │
│  7. Return A2A Task (COMPLETED | FAILED | ...)          │
│     + Emit FinOSAgentEventVertex to DAG                 │
└─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  FinOSWorkflowExecutionHistory                          │
│  (new FinOSAgentEvent vertex added to DAG)              │
└─────────────────────────────────────────────────────────┘
```

---

## 3. A2A Protocol Integration

### Agent Card Discovery

Each agent publishes a `FinOSAgentCard` at the A2A standard well-known path:

```
GET /.well-known/agent-card.json
```

The card conforms to the A2A Agent Card specification. All FinOS-specific metadata is carried inside `capabilities.extensions[]` under the URI `https://{vendor-domain}/finos/agent-card/v1`. Setting `required: true` means any A2A caller that does not understand this extension MUST NOT invoke the agent.

### A2A Message Delivery

Callers send messages using the A2A `a2a.sendMessage` JSON-RPC method or HTTP binding:

```
POST /message:send
Content-Type: application/json
Authorization: Bearer <token>
A2A-Version: 1.0
A2A-Extensions: https://{vendor-domain}/finos/agent-card/v1

{
  "messageId": "...",
  "role": "user",
  "parts": [{ "kind": "text", "text": "Classify this invoice..." }],
  "metadata": {
    "finos": {
      "callingContext": { ... },
      "contextGraphs": [ ... ],
      "policies": [ ... ],
      "variables": { "inputs": { ... }, "outputs": { ... } },
      "sessionContextStore": { ... }
    }
  }
}
```

### A2A Task States

Agent responses follow the A2A task lifecycle:

| State | Description |
|---|---|
| `SUBMITTED` | Task received, not yet started |
| `WORKING` | Agent actively processing |
| `COMPLETED` | Execution complete |
| `FAILED` | Execution failed |
| `CANCELED` | Canceled before completion |
| `INPUT_REQUIRED` | Agent needs additional input (HITL) |
| `REJECTED` | Agent rejected the task |
| `AUTH_REQUIRED` | Authentication/authorization needed |

### A2A Headers

| Header | Purpose |
|---|---|
| `A2A-Version` | Protocol version negotiation (e.g. `1.0`) |
| `A2A-Extensions` | Comma-separated list of extension URIs to activate |

---

## 4. Schema Reference

### 4.1 FinOSAgentCard — A2A Discovery Document

The `FinOSAgentCard` is the A2A Agent Card with FinOS extensions. Served at `/.well-known/agent-card.json`.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✓ | Human-readable agent name |
| `description` | `string` | ✓ | What this agent does |
| `version` | `string` | ✓ | Agent version (semver) |
| `skills` | `Skill[]` | ✓ | Agent capabilities (A2A required) |
| `supportedInterfaces` | `AgentInterface[]` | ✓ | Protocol binding endpoints |
| `capabilities` | `object` | ✓ | Capability flags + FinOS extensions |
| `defaultInputModes` | `A2AInputMode[]` | ✓ | Supported input modalities |
| `defaultOutputModes` | `A2AOutputMode[]` | ✓ | Supported output modalities |
| `provider` | `AgentProvider` | — | Organization providing this agent |
| `securitySchemes` | `map<string, SecurityScheme>` | — | A2A security scheme definitions |
| `securityRequirements` | `SecurityRequirement[]` | — | Required security schemes |
| `documentationUrl` | `string (uri)` | — | Link to agent documentation |

### 4.2 AgentInterface

Defines a protocol binding endpoint for communicating with this agent.

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | `string (uri)` | ✓ | Endpoint URL |
| `protocolBinding` | `enum` | ✓ | `JSONRPC` \| `GRPC` \| `HTTP+JSON` |
| `protocolVersion` | `string` | — | Protocol version (e.g. "1.0") |
| `tenant` | `string` | — | Tenant identifier if scoped |

### 4.3 SecurityScheme & Security Requirements

Follows the A2A security model (mirrors OpenAPI SecurityScheme):

```json
{
  "securitySchemes": {
    "bearerAuth": {
      "type": "http",
      "scheme": "bearer",
      "bearerFormat": "JWT"
    },
    "apiKeyAuth": {
      "type": "apiKey",
      "in": "header",
      "name": "X-FinOS-API-Key"
    }
  },
  "securityRequirements": [
    { "bearerAuth": ["finos:agent:invoke"] }
  ]
}
```

### 4.4 AgentProvider

| Field | Type | Required | Description |
|---|---|---|
| `organization` | `string` | Provider name (e.g. "Example Corp") |
| `url` | `string (uri)` | Provider URL |

### 4.5 AgentExtension

A2A-compatible extension entry in `capabilities.extensions[]`.

| Field | Type | Required | Description |
|---|---|---|---|
| `uri` | `string (uri)` | ✓ | Extension identifier |
| `required` | `boolean` | — | If true, callers MUST understand this extension |
| `description` | `string` | — | Human-readable description |
| `params` | `object` | — | Extension-specific parameters |

### 4.6 FinOSAgentExtensionParams

Typed parameters for the FinOS extension (`uri: "https://{vendor-domain}/finos/agent-card/v1"`):

| Field | Type | Description |
|---|---|---|
| `finOSAgentId` | `string (uuid)` | ✓ | FinOS-specific globally unique agent ID |
| `finOSAgentType` | `enum` | ✓ | Agent role: ORCHESTRATOR, PLANNER, EXECUTOR, REVIEWER, MONITOR, EXTRACTOR, CLASSIFIER, SUMMARIZER, CUSTOM |
| `finOSLlmModels` | `FinOSLlmModelEntry[]` | ✓ | Ordered model roster (index 0 = default). See §4.6.1. |
| `finOSSupportedContextModes` | `ContextGraphMode[]` | — | Context graph modes the agent can consume |
| `finOSPoliciesRequired` | `string[]` | — | Policy types callers MUST include |
| `finOSDAGNodeTypes` | `string[]` | — | DAG node types this agent may run as |
| `finOSTenantId` | `string` | — | Tenant this agent is registered in |
| `finOSRegisteredAt` | `string (datetime)` | — | Registration timestamp (read-only) |
| `finOSInputSchema` | `FinOSAgentIOSchema` | — | Static input port contract. See §4.6.2–4.6.3 and §9. |
| `finOSOutputSchema` | `FinOSAgentIOSchema` | — | Static output port contract. See §4.6.2–4.6.3 and §9. |

### 4.6.1 FinOSLlmModelEntry — Model Roster Entry

Each entry in `finOSLlmModels` conforms to `FinOSLlmModelEntry`:

| Field | Type | Required | Description |
|---|---|---|---|
| `modelId` | `string` | ✓ | Model identifier as accepted by the provider's API (e.g. `"claude-sonnet-4-6"`) |
| `provider` | `string` | — | Provider identifier: `"anthropic"`, `"openai"`, `"meta"`, `"self-hosted"` |
| `version` | `string` | — | Exact snapshot / version pin (e.g. `"20250514"`). Use when reproducibility is required. |
| `taskAffinity` | `string[]` | — | Sub-task or sub-agent role IDs for which this model is preferentially selected. When null/empty, eligible for all unmatched tasks. |
| `contextWindowTokens` | `integer` | — | Declared context window capacity in tokens. Used for context-graph assembly planning. |
| `isDefault` | `boolean` | — | If `true`, fallback for all tasks with no `taskAffinity` match. At most one entry may be `isDefault: true`; if none, index 0 is default. |

**Routing semantics:** When the orchestrator dispatches a sub-task, it scans `finOSLlmModels` in index order and selects the first entry whose `taskAffinity` list contains the sub-task's role. If no entry matches, the `isDefault: true` entry (or index 0) is used.

**Example — composite agent with two model tiers:**

```json
"finOSLlmModels": [
  {
    "modelId": "claude-sonnet-4-6",
    "provider": "anthropic",
    "isDefault": true,
    "contextWindowTokens": 200000
  },
  {
    "modelId": "claude-haiku-4-5-20251001",
    "provider": "anthropic",
    "taskAffinity": ["DOCUMENT_EXTRACTION", "LINE_ITEM_PARSING"],
    "contextWindowTokens": 200000
  }
]
```

Here the lightweight Haiku model handles high-volume extraction sub-tasks; the default Sonnet model handles all other reasoning.

---

### 4.6.2 FinOSAgentIOSchema — I/O Schema Container

`FinOSAgentIOSchema` is used for both `finOSInputSchema` and `finOSOutputSchema`:

| Field | Type | Required | Description |
|---|---|---|---|
| `ports` | `map<string, FinOSAgentPortDeclaration>` | — | Map of port name → port declaration. Keys must satisfy `^[a-zA-Z_][a-zA-Z0-9_]*$`. |
| `schemaVersion` | `string` | — | Semver version of this I/O schema. Increment on any breaking port change so the orchestrator can detect stale wiring. |

---

### 4.6.3 FinOSAgentPortDeclaration — Port Declaration

Each entry in `FinOSAgentIOSchema.ports` conforms to `FinOSAgentPortDeclaration`:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✓ | Port name — valid identifier matching the map key. Binds to the same-named key in runtime `FinOSVariableBindings`. |
| `type` | `VariableType` | ✓ | Data type: STRING, NUMBER, BOOLEAN, OBJECT, ARRAY, BINARY, DATETIME, UUID, CURRENCY, REFERENCE. |
| `required` | `boolean` | — | **Input:** agent MUST receive a non-null value or aborts. **Output:** agent MUST populate before completing. Orchestrator enforces at compile and execution time. |
| `description` | `string` | — | Human-readable description of what this port represents. |
| `dataClassification` | `DataClassification` | — | PUBLIC / INTERNAL / CONFIDENTIAL / RESTRICTED. Governs redaction and policy enforcement. Defaults to INTERNAL. |
| `constraints` | `object` | — | JSON Schema fragment constraining value space (e.g. `{"minimum": 0, "maximum": 1}`). Enforced at compile and invocation time. |
| `examples` | `any[]` | — | Representative example values for designer hints and SDK test generation. |

**`FinOSAgentPortDeclaration` vs `FinOSVariableBinding`:**

| Concern | `FinOSAgentPortDeclaration` | `FinOSVariableBinding` |
|---|---|---|
| **Where it lives** | `FinOSAgentCard` (discovery doc) | `Message.metadata.finos.variables` |
| **When evaluated** | Workflow compile time | Agent invocation time |
| **Carries a value?** | No — schema declaration only | Yes — carries `value`, `storeKey`, `sourceAgentId` |
| **Purpose** | Designer UI / compile-time validation | Runtime data exchange |

---

### 4.7 FinOSMessageMetadata — Invocation Governance

All FinOS runtime governance data is carried in `Message.metadata.finos` and conforms to the `FinOSMessageMetadata` schema:

| Field | Type | Description |
|---|---|---|
| `callingContext` | `FinOSCallingContext` | Caller identity, DAG position, tracing |
| `additionalUserPrompt` | `FinOSAdditionalPrompt` | Extra prompt to inject |
| `contextGraphs` | `FinOSContextGraph[]` | Context in one or more modes |
| `policies` | `FinOSAgentPolicy[]` | Governing policies |
| `variables` | `FinOSVariableBindings` | Input/output variable bindings |
| `sessionContextStore` | `FinOSSessionContextStoreRef` | Shared state store reference |

### 4.8 Calling Context

Tracks WHO is calling the agent and WHERE in the DAG it sits. Propagated to `FinOSEventBase` fields in the outcome.

```yaml
callingContext:
  callerId: uuid              # Caller's ID
  callerType: enum            # WORKFLOW_ENGINE | AGENT | SERVICE | HUMAN
  callerRole: string          # RBAC role → governs policy enforcement
  sessionId: uuid             # Cross-agent session correlation
  workflowInstanceId: uuid    # → outcome.workflowInstanceId
  workflowDefinitionId: uuid
  workflowVersionId: uuid
  nodeInstanceId: uuid        # DAG node this agent executes as → outcome.vertexId
  dagDepth: integer           # 0 = root node
  parentAgentId: uuid         # Set when callerType = AGENT
  correlationId: uuid         # → outcome.correlationId
  traceId: string             # OpenTelemetry → outcome.traceId
  spanId: string              # OpenTelemetry → outcome.spanId
  executionPriority: enum     # LOW | NORMAL | HIGH | CRITICAL
  maxExecutionMs: integer     # Timeout; agent aborts if exceeded
  retryPolicy: object         # maxAttempts, backoffMs, retryOn
```

### 4.9 Additional User Prompt

Optional extra prompt injected by the caller. Can append to or replace the agent's base system prompt.

```yaml
additionalUserPrompt:
  promptId: uuid
  promptText: string         # The injected text (up to 65536 chars)
  promptRole: SYSTEM | USER | ASSISTANT
  promptPriority: 0–100      # Higher = earlier in context window
  isOverride: boolean        # true = replaces system prompt
  languageCode: string       # BCP-47
  promptTemplateId: uuid     # → outcome.promptTemplate
```

### 4.10 Context Graphs

See [Section 5](#5-context-graph-modes) for full mode details.

```yaml
contextGraphs:
  - contextGraphId: uuid
    mode: RAG_QUERY | RAG_TENSOR | FINOSWORKFLOW_GRAPH | STRUCTURED_CHUNKS
    rank: integer            # Assembly order (lower = earlier in context)
    ragQuery: ...            # Populated when mode=RAG_QUERY
    ragTensor: ...           # Populated when mode=RAG_TENSOR
    finOSWorkflowGraph: ...  # Populated when mode=FINOSWORKFLOW_GRAPH
    structuredChunks: ...    # Populated when mode=STRUCTURED_CHUNKS
```

### 4.11 FinOS Policies

```yaml
policies:
  - policyId: uuid
    policyName: string
    policyVersion: semver
    policyType: APPROVAL | COMPLIANCE | RISK | ACCESS_CONTROL | DATA_GOVERNANCE | SPENDING_LIMIT | RATE_LIMIT
    enforcementMode: BLOCK | WARN | AUDIT
    escalationRole: string
    rules:
      - ruleId: string
        ruleExpression: string    # FPEL or OPA Rego
        ruleDescription: string
        severity: INFO | WARNING | ERROR | CRITICAL
```

### 4.12 Variable Bindings

Variables are carried in `Message.metadata.finos.variables`:

```yaml
variables:
  inputs:
    variables:
      invoiceAmount:
        variableId: uuid
        name: invoiceAmount
        type: CURRENCY
        value: { "amount": 48500.00, "currency": "USD" }
        dataClassification: CONFIDENTIAL
        required: true
        storeKey: "invoiceAmount"
  outputs:
    variables:
      classification:
        variableId: uuid
        name: classification
        type: OBJECT
        value: null            # Populated by agent on completion
        required: true
        storeKey: "classification"
```

### 4.13 Session Context Store Reference

```yaml
sessionContextStore:
  storeId: uuid
  storeType: REDIS | DYNAMODB | POSTGRES | IN_MEMORY | VECTOR_STORE | OBJECT_STORE
  storeUri: uri
  namespace: string          # {tenantId}/{workflowInstanceId}/{sessionId}
  accessMode: READ | WRITE | READ_WRITE
  credentialRef: uuid        # Vault reference (never inline)
  ttlSeconds: integer
  readKeys: string[]
  readOnlyKeys: string[]
```

### 4.14 Outcome — FinOSAgentCardOutcome

The outcome conforms to `FinOSAgentEventVertex` (CloudEvents v1.0). Key fields:

| Field | Source | Description |
|---|---|---|
| `id` | A2A Task ID | CloudEvent event ID |
| `source` | Agent URI | CloudEvent source |
| `specversion` | `"1.0"` | CloudEvent version |
| `type` | Agent type | e.g. `{event-namespace}.finos.agent.executor.completed` |
| `agent` | Agent card | Agent identity |
| `agentRun` | Runtime | Execution metadata (runId, status, timing, tokens) |
| `toolCalls` | Runtime | Tool calls made during execution (array) |
| `evidence` | Runtime | Evidence artifact with output variables |

---

## 5. Context Graph Modes

### 5.1 RAG_QUERY

Live vector similarity search at invocation time. The agent issues a query against the specified vector store and injects retrieved chunks into context.

Key fields: `vectorStoreId`, `queryText`, `embeddingModel`, `topK`, `similarityThreshold`, `namespace`, `metadataFilter`, `rerankModel`.

### 5.2 RAG_TENSOR

Pre-tokenized embedding matrix injected directly, bypassing live retrieval. Supports dense embedding matrices (`[numChunks, embeddingDim]`) and raw token ID sequences.

Key fields: `tensorId`, `embeddingModel`, `tensorMode`, `dtype`, `shape`, `storageMode`, `storageUri`.

### 5.3 FINOSWORKFLOW_GRAPH

Subgraph of a `FinOSWorkflowExecutionHistory` DAG serialized and injected as context. Useful for agents that reason over execution history.

Key fields: `workflowInstanceId`, `anchorVertexId`, `includeVertexTypes`, `includeEdgeTypes`, `maxDepth`, `serializationFormat`.

### 5.4 STRUCTURED_CHUNKS

Pre-fetched, pre-chunked text documents provided directly. Used when the caller has already performed retrieval.

Key fields: `chunks[]` with `chunkId`, `text`, `sourceId`, `relevanceScore`, `rank`, `dataClassification`.

---

## 6. Outcome Population Contract

Fields from `Message.metadata.finos` propagate to the `FinOSAgentEventVertex` outcome:

| Message Metadata Field | Outcome Field |
|---|---|
| `callingContext.workflowInstanceId` | `workflowInstanceId` |
| `callingContext.correlationId` | `correlationId` |
| `callingContext.traceId` | `traceId` |
| `callingContext.spanId` | `spanId` |
| `callingContext.nodeInstanceId` | `vertexId` |
| `additionalUserPrompt.promptTemplateId` | `promptTemplate` |
| `variables.outputs` | `data.outputVariables` |

Agent identity fields (`agentId`, `agentName`, `llmModelName`, etc.) come from the `FinOSAgentCard` discovery document.

---

## 7. Policy Enforcement Lifecycle

```
Message received
  → Extract metadata.finos.policies[]
  → For each policy (top-to-bottom):
      evaluate rules against inputVariables
      │
      ├─ All rules pass   → continue
      ├─ WARN mode fail   → log warning, continue
      ├─ AUDIT mode fail  → record event, continue
      └─ BLOCK mode fail  → emit FinOSPolicyEvent
                              set task state = FAILED
                              return PolicyViolationProblem (HTTP 403)
  → All policies pass → proceed to execution
```

---

## 8. Variable Binding Contract

### 8.1 Static Schema (Agent Card) vs Runtime Bindings (Message Metadata)

FinOS uses two complementary layers for agent I/O. Understanding their difference is essential for authoring workflow definitions and agent cards correctly.

| Concern | Static schema (`FinOSAgentCard`) | Runtime binding (`Message.metadata.finos.variables`) |
|---|---|---|
| **Schema type** | `FinOSAgentIOSchema` / `FinOSAgentPortDeclaration` | `FinOSVariableBindings` / `FinOSVariableBinding` |
| **Location** | `capabilities.extensions[finos].params.finOSInputSchema` / `finOSOutputSchema` | `Message.metadata.finos.variables.inputs` / `.outputs` |
| **When evaluated** | Workflow compile time — before any execution | Agent invocation time — during a specific run |
| **Carries values?** | No — schema declarations only | Yes — carries actual `value`, `storeKey`, `sourceAgentId` |
| **Purpose** | Workflow designer UI; compile-time wiring validation; auto-resolving `outputMapping` paths | Runtime data exchange between agents via the session context store |

### 8.2 Runtime Binding Contract

- **Input variables** (`variables.inputs`): Agent reads at execution start. Bindings with `storeKey` are also read from the session store.
- **Output variables** (`variables.outputs`): Templates with `value: null`. Agent populates `value` on completion and writes to the session store under `storeKey`.
- Variables marked `CONFIDENTIAL` or `RESTRICTED` are automatically redacted from logs and evidence.
- Both inputs and outputs are nullable — stateless agents may use neither.

### 8.3 How Static Schema Resolves `outputMapping`

When `finOSOutputSchema` is declared on the agent card, the orchestrator can resolve `outputMapping` automatically:

```
Without finOSOutputSchema (legacy):
  outputMapping:
    invoiceCategory:  "$.task.output.invoiceCategory"   ← hand-authored JSONPath
    confidenceScore:  "$.task.output.confidenceScore"   ← brittle, undeclared

With finOSOutputSchema declared:
  outputMapping is auto-generated by name:
    invoiceCategory  → FinOSAgentPortDeclaration { type: STRING, required: true }
    confidenceScore  → FinOSAgentPortDeclaration { type: NUMBER, constraints: {min:0,max:1} }
  ↳ Type mismatch caught at compile time, not at first execution.
```

---

## 9. Agent I/O Contract

This section describes how `finOSInputSchema` and `finOSOutputSchema` are used by the orchestrator and the visual workflow designer.

### 9.1 The Three Problems They Solve

**Problem 1 — No designer binding UI**

Without a static schema, the visual workflow designer (React Flow / Overflow) has no information about what ports Agent A produces or what Agent B expects. Bindings must be typed by convention (`$.task.output.outcome`), offer no field suggestions, enforce no types, and fail silently when a key is misspelled.

With `finOSOutputSchema` and `finOSInputSchema` declared, the designer can:
- Render Agent A's declared output ports as draggable binding sources.
- Render Agent B's declared input ports as typed binding targets.
- Display type badges and `required` indicators.
- Warn immediately when a NUMBER port is wired to a STRING port.

**Problem 2 — No compile-time wiring validation**

Without a schema, wiring errors (misnamed key, wrong type, missing required input) are only discovered when the workflow executes against a live agent — potentially after a multi-step workflow has already mutated state in upstream nodes.

With declared schemas, the orchestrator's `POST /workflows/{workflowId}/validate` endpoint:
- Checks that every `required: true` input port of each agent is wired to an output port of compatible type.
- Rejects the workflow definition before deployment if any binding is unresolvable.

**Problem 3 — Fragile, redundant JSONPath outputMapping**

`outputMapping` expressions like `$.task.output.invoiceCategory` reference an undeclared structure. They break when the agent's output shape changes and give no indication to the orchestrator of types or constraints.

When `finOSOutputSchema` is present, the orchestrator resolves port bindings by name and auto-generates the equivalent JSONPath, making hand-authored `outputMapping` redundant for declared ports. Agents that have not yet declared `finOSOutputSchema` continue to work with explicit `outputMapping` for backward compatibility.

### 9.2 Orchestrator Compile-Time Validation Algorithm

```
For each FinOSAgent node in the workflow DAG:
  1. Fetch agent's FinOSAgentCard from agent registry
  2. Read finOSInputSchema.ports and finOSOutputSchema.ports

  For each input port in finOSInputSchema.ports:
    a. Find the upstream node whose output is wired to this port
    b. If upstream node has finOSOutputSchema, look up the source port
       → Assert source.type == target.type (or compatible subtype)
       → If incompatible → COMPILE ERROR: type mismatch
    c. If port.required == true and no upstream binding → COMPILE ERROR: missing required input

  For each output port in finOSOutputSchema.ports:
    → Auto-register as resolvable by name for downstream binding
    → Replace manual outputMapping entries with name-based references
```

### 9.3 Designer Binding Flow

```
Tenant drags Agent A's output handle onto Agent B's input handle
  │
  ├─ Designer queries Agent A's finOSOutputSchema.ports
  │    → Shows: [ invoiceCategory: STRING, extractedLineItems: ARRAY, confidenceScore: NUMBER ]
  │
  ├─ Designer queries Agent B's finOSInputSchema.ports
  │    → Shows: [ category: STRING (required), lineItems: ARRAY (required) ]
  │
  ├─ Tenant selects: A.invoiceCategory → B.category (STRING → STRING ✓)
  │                  A.extractedLineItems → B.lineItems (ARRAY → ARRAY ✓)
  │
  └─ Designer stores typed binding — no JSONPath required
       At validate time: orchestrator confirms types match ✓
```

### 9.4 Backward Compatibility

Declaring `finOSInputSchema` and `finOSOutputSchema` is **optional**. Agents that do not declare them continue to work with existing `outputMapping` JSONPath expressions. The orchestrator skips compile-time schema validation for agent nodes whose cards have `null` schemas.

Migration path: add `finOSInputSchema` and `finOSOutputSchema` to existing agent cards without changing invocation mechanics. Once declared, `outputMapping` entries for those ports can be removed from workflow definitions.

---

## 10. API Reference

### Discovery

```
GET /.well-known/agent-card.json
→ 200: FinOSAgentCard
```

### Message Delivery (HTTP Binding)

```
POST /message:send
Content-Type: application/json
Authorization: Bearer <token>
A2A-Version: 1.0
A2A-Extensions: https://{vendor-domain}/finos/agent-card/v1

Body: A2A Message with metadata.finos = FinOSMessageMetadata

→ 200: A2A Task (completed synchronously)
→ 202: A2A Task (accepted for async processing)
→ 400: Bad request
→ 401: Unauthorized
→ 403: Policy violation
→ 422: Validation failed
→ 500: Internal error
```

### JSON-RPC Binding

```
POST <supportedInterfaces[].url>
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "a2a.sendMessage",
  "id": "...",
  "params": { <A2A Message> }
}
```

---

## 11. Security Model

Authentication and authorization use the A2A `securitySchemes` / `securityRequirements` pattern on the agent card:

```json
{
  "securitySchemes": {
    "bearerAuth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" },
    "apiKeyAuth": { "type": "apiKey", "in": "header", "name": "X-FinOS-API-Key" }
  },
  "securityRequirements": [
    { "bearerAuth": ["finos:agent:invoke"] }
  ]
}
```

**Trust tiers** (T0–T3) govern what data an agent may access and whether HITL is required. The trust tier is declared in the FinOS extension params (`finOSAgentExtensionParams`) and enforced by the policy engine.

Session store credentials are NEVER inline — they use vault references (`credentialRef`).

---

## 12. Usage Examples

### Workflow Engine → Agent (A2A Message)

```json
{
  "messageId": "msg-0001",
  "role": "user",
  "parts": [
    { "kind": "text", "text": "Classify the attached invoice..." },
    { "kind": "data", "data": { "invoicePdf": "<base64>" } }
  ],
  "metadata": {
    "finos": {
      "callingContext": {
        "callerId": "workflow-engine-01",
        "callerType": "WORKFLOW_ENGINE",
        "sessionId": "sess-001",
        "workflowInstanceId": "wf-001",
        "correlationId": "corr-001",
        "executionPriority": "HIGH"
      },
      "policies": [
        {
          "policyId": "pol-spending-limit",
          "policyName": "Spending Limit Gate",
          "policyType": "SPENDING_LIMIT",
          "enforcementMode": "BLOCK",
          "rules": [
            {
              "ruleId": "rule-001",
              "ruleExpression": "inputVariables.invoiceAmount <= 50000"
            }
          ]
        }
      ],
      "variables": {
        "inputs": {
          "variables": {
            "invoiceAmount": {
              "variableId": "var-001",
              "name": "invoiceAmount",
              "type": "CURRENCY",
              "value": { "amount": 48500.00, "currency": "USD" },
              "required": true
            }
          }
        }
      }
    }
  },
  "extensions": ["https://{vendor-domain}/finos/agent-card/v1"]
}
```

### Agent Card (Discovery Document)

```json
{
  "name": "InvoiceClassifierAgent",
  "description": "Classifies invoices and extracts structured data",
  "version": "2.1.0",
  "provider": {
    "organization": "Example Corp",
    "url": "https://www.blackline.com"
  },
  "supportedInterfaces": [
    {
      "url": "https://agents.example.com/invoice-classifier/message:send",
      "protocolBinding": "HTTP+JSON",
      "protocolVersion": "1.0"
    }
  ],
  "capabilities": {
    "streaming": false,
    "pushNotifications": false,
    "stateTransitionHistory": true,
    "extensions": [
      {
        "uri": "https://{vendor-domain}/finos/agent-card/v1",
        "required": true,
        "description": "FinOS agent extension",
        "params": {
          "finOSAgentId": "agt-invoice-classifier-001",
          "finOSAgentType": "EXECUTOR",
          "finOSLlmModels": [
            {
              "modelId": "claude-sonnet-4-6",
              "provider": "anthropic",
              "isDefault": true,
              "contextWindowTokens": 200000
            },
            {
              "modelId": "claude-haiku-4-5-20251001",
              "provider": "anthropic",
              "taskAffinity": ["DOCUMENT_EXTRACTION", "LINE_ITEM_PARSING"],
              "contextWindowTokens": 200000
            }
          ],
          "finOSSupportedContextModes": ["RAG_QUERY", "STRUCTURED_CHUNKS"],
          "finOSPoliciesRequired": ["SPENDING_LIMIT"],
          "finOSTenantId": "tenant-acme-prod",
          "finOSInputSchema": {
            "schemaVersion": "1.0",
            "ports": {
              "invoiceDocument": {
                "name": "invoiceDocument",
                "type": "OBJECT",
                "required": true,
                "description": "Raw invoice document (PDF text or structured JSON)",
                "dataClassification": "CONFIDENTIAL"
              },
              "vendorId": {
                "name": "vendorId",
                "type": "UUID",
                "required": true,
                "description": "Vendor identifier for cross-referencing vendor master data",
                "dataClassification": "INTERNAL"
              }
            }
          },
          "finOSOutputSchema": {
            "schemaVersion": "1.0",
            "ports": {
              "invoiceCategory": {
                "name": "invoiceCategory",
                "type": "STRING",
                "required": true,
                "description": "Classified invoice category (e.g. VENDOR_INVOICE, EXPENSE_REPORT)",
                "constraints": { "enum": ["VENDOR_INVOICE", "CREDIT_NOTE", "EXPENSE_REPORT"] }
              },
              "extractedLineItems": {
                "name": "extractedLineItems",
                "type": "ARRAY",
                "required": true,
                "description": "Structured list of extracted line items with amounts and descriptions"
              },
              "confidenceScore": {
                "name": "confidenceScore",
                "type": "NUMBER",
                "required": true,
                "description": "Classification confidence between 0.0 and 1.0",
                "constraints": { "minimum": 0, "maximum": 1 }
              }
            }
          }
        }
      }
    ]
  },
  "securitySchemes": {
    "bearerAuth": {
      "type": "http",
      "scheme": "bearer",
      "bearerFormat": "JWT"
    }
  },
  "securityRequirements": [
    { "bearerAuth": ["finos:agent:invoke"] }
  ],
  "defaultInputModes": ["text", "data"],
  "defaultOutputModes": ["data"],
  "skills": [
    {
      "id": "invoice-classification",
      "name": "Invoice Classification",
      "description": "Classifies invoices by type and extracts structured fields",
      "tags": ["finance", "classification", "extraction"],
      "examples": ["Classify this invoice and extract line items"]
    }
  ]
}
```
