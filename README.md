# Blackline FinOS — Open Specifications for Governed Agentic AI in the Office of the CFO

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0%20%7C%203.1-brightgreen)](https://spec.openapis.org/oas/latest.html)
[![A2A Protocol](https://img.shields.io/badge/A2A-FinOS%20Agent--to--Agent-orange)](https://a2a-protocol.org/latest/specification/)
[![CloudEvents](https://img.shields.io/badge/CloudEvents-v1.0-yellow)](https://cloudevents.io/)
[![Spec Version](https://img.shields.io/badge/Spec%20Version-1.0.0-lightgrey)]()

**Publisher:** BlackLine, Inc. · **Domain:** Agentic Financial Operations Platform&trade; . 

---

This repository publishes the open API specifications and schemas for **Blackline Agentic Financial Operations Platform&trade;** — the orchestration and governance platform for agentic AI enterprise finance. The **Agentic Orchestration** platform is abbreviated as **FinOS** and **finOS** in the documents and specifications, whereas the governance surface is referred to as **Financial Control Console** or just simply **Control Console**. 

It serves two purposes at once.

**As a standard**, it is an opinionated specification for building governed, deterministic
agentic AI automation, orchestration, and control planes for the office of the CFO. The design
positions here are deliberate: control flow is a statically analysable DAG rather than model
output, policy is versioned data rather than code, human judgement is a first-class node type
rather than a callback bolted on, and every step emits a CloudEvent into an append-only
provenance graph. You can adopt these contracts wholesale, implement the model against your own
services, or take individual pieces. Nothing here requires a Blackline deployment.

**As an integration contract**, it is the API surface for connecting to, observing, and
extending Blackline FinOS through Control Console — agent registration, event publication and
subscription, policy configuration, workflow authoring, and execution audit retrieval.

The two readings share one set of artifacts. The integration patterns in
[§6–§10](#6-integration-pattern-1--control-console-integration) are written against Blackline
endpoints, but each is also a worked reference implementation of the model for anyone building
their own.

---

## Table of Contents

1. [What This Repository Is](#1-what-this-repository-is)
2. [Who This Is For](#2-who-this-is-for)
3. [The Specification as a Standard](#3-the-specification-as-a-standard)
   - 3.1 [Design Tenets](#31-design-tenets)
   - 3.2 [What a Conformant Implementation Must Do](#32-what-a-conformant-implementation-must-do)
   - 3.3 [Adoption Profiles](#33-adoption-profiles)
   - 3.4 [Namespace Placeholders](#34-namespace-placeholders)
   - 3.5 [Cross-Cutting API Conventions](#35-cross-cutting-api-conventions)
4. [Repository Contents](#4-repository-contents)
5. [Platform Architecture](#5-platform-architecture)
6. [Integration Pattern 1 — Control Console Integration](#6-integration-pattern-1--control-console-integration)
   - 6.1 [Registering and Deregistering Agents](#61-registering-and-deregistering-agents)
   - 6.2 [Publishing and Consuming FinOS Events](#62-publishing-and-consuming-finos-events)
   - 6.3 [Managing Policy Configurations](#63-managing-policy-configurations)
   - 6.4 [Retrieving Workflow Execution Status and History](#64-retrieving-workflow-execution-status-and-history)
7. [Integration Pattern 2 — CNAPP and SIEM Event Ingestion](#7-integration-pattern-2--cnapp-and-siem-event-ingestion)
8. [Integration Pattern 3 — Policy Event Response and HITL Implementation](#8-integration-pattern-3--policy-event-response-and-hitl-implementation)
9. [Integration Pattern 4 — Workflow Execution Monitoring UI](#9-integration-pattern-4--workflow-execution-monitoring-ui)
10. [Integration Pattern 5 — Workflow DAG Authoring UI](#10-integration-pattern-5--workflow-dag-authoring-ui)
11. [API and Schema Quick Reference](#11-api-and-schema-quick-reference)
12. [Key FinOS Concepts](#12-key-finos-concepts)
13. [Authentication](#13-authentication)
14. [Contributing](#14-contributing)
15. [License](#15-license)

---

## 1. What This Repository Is

### The problem it addresses

Finance is adopting AI agents faster than governance frameworks can follow. Agents from ERP
vendors, system integrators, BPOs, and internal teams are proliferating across the office of the
CFO — each with its own access model, logging, and risk profile, and none under a common
standard. The work they touch is not tolerant of non-determinism: a reconciliation that routes
differently on identical inputs, a journal entry posted by an agent that was supposed to be
read-only, or an approval with no reconstructible evidence chain are all audit findings.

The response in these specifications is structural rather than procedural. Governance is not a
review step layered over an agent; it is a property of the execution model itself.

### The two subjects

**Blackline FinOS** is the agentic AI orchestration engine for financial close automation. It
designs, deploys, and governs AI-powered workflows that automate financial processes —
transaction matching, reconciliation, journal entry, intercompany accounting, cash application,
and the financial close — by composing AI agents, REST API calls, long-running partner jobs,
human review steps, child workflows, and policy evaluations into auditable Directed Acyclic
Graph (DAG) workflows.

**Blackline Control Console** is the governance and observability plane that operates above
FinOS. It provides a unified surface for registering and monitoring all agents
(Blackline-native, certified partners, and customer-built), governing their behaviour through
policies, observing execution events in real time, and maintaining an immutable audit trail that
satisfies SOX and financial controls requirements.

The relationship is direct: **FinOS executes. Control Console governs.**

| Platform Layer | Function |
|---|---|
| **Control Console** | Agent governance, policy enforcement, real-time observability, SOX audit trail, exception routing |
| **FinOS** | Workflow orchestration engine — nine typed nodes: `FinOSStart`, `FinOSAgent`, `FinOSAPI`, `FinOSAsyncAPI`, `FinOSHITL`, `FinOSPolicy`, `FinOSWorkflow`, `FinOSFanOut`, `FinOSTerminal` |
| **MetaERP** | ERP-agnostic financial data layer; provides the context graph substrate agents reason over |

FinOS and Control Console are the reference implementation of the model in
[§3](#3-the-specification-as-a-standard). The specifications are published separately from that
implementation so the model can be adopted without it.

---

## 2. Who This Is For

### Building your own orchestration or control plane

You are designing an agentic automation platform for finance work and want a governance model
you do not have to derive from first principles. Take the schemas as your contracts, the
integrity rules as your validator's specification, and the provenance model as your audit
design.

| You are building… | Start with |
|---|---|
| A workflow orchestration engine that must route deterministically and survive audit | [`finos-workflow-api.yaml`](./finos-workflow-api.yaml) · [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) · [§3.1](#31-design-tenets) |
| A policy or governance service that gates agent actions at runtime | [`finos-policy-api.yaml`](./finos-policy-api.yaml) · [`4-finos-policy-guide.md`](./4-finos-policy-guide.md) |
| An agent registry, trust model, or agent discovery surface | [`finos-agent-card-schema.yaml`](./finos-agent-card-schema.yaml) · [`finos-agent-registry-api.yaml`](./finos-agent-registry-api.yaml) |
| An audit, provenance, or evidence-chain store for agent activity | [`finos-workflow-execution-history-openapi.yaml`](./finos-workflow-execution-history-openapi.yaml) · [`1-finosworkflow-execution-history.md`](./1-finosworkflow-execution-history.md) |
| A control plane that has to answer "which agent did what, under which policy, on whose authority" | [§3.2](#32-what-a-conformant-implementation-must-do) · [§12](#12-key-finos-concepts) |
| An agent intended to interoperate across more than one FinOS-compatible platform | [`finos-agent-card-schema.yaml`](./finos-agent-card-schema.yaml) · [`2-finos-agent-card.md`](./2-finos-agent-card.md) |

### Integrating with Blackline FinOS and Control Console

You are connecting an existing system to a Blackline deployment. Five integration archetypes
cover most cases; most integrations touch more than one.

| # | You are building… | Primary APIs |
|---|---|---|
| **1** | A system that connects to Control Console to register agents, subscribe to events, manage policies, or query workflow execution status | Agent Registry · Event Service · Policy Service · Execution History |
| **2** | A CNAPP, SIEM, or security/compliance platform that publishes governance events into the FinOS event bus | Event Service |
| **3** | An application that responds to FinOS Policy evaluation outcomes and implements the human-in-the-loop (HITL) review, approval, or rejection step in a running workflow | Event Service · Workflow API |
| **4** | A user interface or dashboard that renders the execution status and audit trail of running or completed FinOS workflows | Execution History API · Event Service |
| **5** | A workflow design tool or low-code authoring surface that allows users to create and manage FinOS workflow DAG definitions | Workflow API |

---

## 3. The Specification as a Standard

This section is the part of the repository that is independent of Blackline. It states what the
model asserts, what an implementation has to honour to be conformant, and how much of it you can
take without taking all of it.

### 3.1 Design Tenets

Each tenet is a position the specification takes, and each is enforced somewhere concrete rather
than left as advice.

| # | Tenet | Where it is enforced |
|---|---|---|
| 1 | **Control flow is a DAG, not model output.** A workflow is typed nodes and condition edges, statically analysable before it runs. An LLM reasons *inside* a node; it never decides which node runs next. | `finos-workflow-api.yaml` · DAG validation endpoint |
| 2 | **Routing is deterministic.** Edges carry jq conditions over workflow context, evaluated in `priority` order, first match wins. The same context produces the same branch on every run. | Edge schema · `0-finos-workflow-guide.md` §4 |
| 3 | **Entry and exit are explicit.** Exactly one `FinOSStart` with no incoming edges; at least one typed `FinOSTerminal` carrying `exitStatus` and a human-readable outcome label. A workflow never ends by running out of edges. | DAG integrity rules |
| 4 | **Branch coverage is exhaustive.** Every outcome a policy node can produce must have a matching outgoing edge. An unrouted outcome is rejected at registration, not discovered in production. | `possibleOutcomes` coverage check |
| 5 | **AI reads; APIs write.** `FinOSAgent` nodes must not mutate systems of record. State changes belong to `FinOSAPI` nodes, where they are explicit, typed, and separately governed. | Node type separation · platform invariant `INV-001` (planned — see `4-finos-policy-guide.md` §8) |
| 6 | **Policy is versioned data, not code.** Conditions are CEL over a declared, typed variable catalog. A tenant-agnostic **Definition** fixes the variable shape and a closed allow-list of outcomes; each tenant's **Configuration** supplies thresholds and pins a Definition version permanently. Tenants set their own limits; they cannot invent outcomes the Definition disallows, and a later Definition edit never retroactively changes a live rule. | `finos-policy-api.yaml` · `4-finos-policy-guide.md` |
| 7 | **Governance fails closed.** A missing configuration, an uncompilable condition, or a strict type-coercion failure yields a deterministic `Failed` / `BLOCK` result — never a `5xx`, and never a silent pass. | Policy evaluate contract |
| 8 | **Human judgement is a node type.** `FinOSHITL` suspends the workflow with its own resume contract, decision vocabulary, escalation policy, and audit record. Approval is part of the graph, not an out-of-band interruption of it. | `FinOSHITLNode` · `5-finos-event-service-guide.md` |
| 9 | **Waiting is modelled, not blocked on.** `FinOSAsyncAPI` suspends for partner work measured in minutes to days and resumes on an authenticated callback or internal event, with a real business deadline and an optional status check to distinguish a lost notification from an unfinished job. | `FinOSAsyncAPINode` |
| 10 | **Agents carry identity and bounded authority.** Every agent publishes an A2A Agent Card and holds a trust tier (`T0`–`T3`) that governs data classification access, write permission, and whether human review is mandatory. Unregistered agents do not execute. | `finos-agent-card-schema.yaml` · `finos-agent-registry-api.yaml` |
| 11 | **Definitions carry no secrets and no vendor coupling.** Node credentials are opaque, platform-recognised references resolved inside the engine; child workflows resolve through a registry. A definition is safe to commit and portable across tenants and environments. | `AuthConfig` · `WorkflowRegistryEntry` |
| 12 | **Audit is structural.** Every step emits a CloudEvents v1.0 vertex into an append-only provenance DAG that records which agent acted, under which policy version, with what context, and who approved. Evidence is a by-product of execution, not a reporting job. | `finos-workflow-execution-history-openapi.yaml` |

The through-line is that determinism and auditability are enforced at the boundaries of the
model — in the graph, the policy service, and the event log — which is what leaves the inside of
an agent free to be probabilistic.

### 3.2 What a Conformant Implementation Must Do

An implementation may substitute its own services, storage, transport, and runtime. It is
conformant with this specification if it honours the following.

| Area | Requirement |
|---|---|
| **Workflow definitions** | Accept the `FinOSWorkflow` document shape — `inputs`, `nodes`, `edges` — and reject a definition that violates any DAG integrity rule in [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) §9, at registration time rather than at execution time |
| **Node type semantics** | Implement the node types it supports with the suspend, resume, retry, and timeout semantics those types define; a subset is permitted, silently redefining one is not |
| **Expression evaluation** | Evaluate `${ }` edge conditions and field templates as jq against workflow context, honouring `priority` ordering and the single-catch-all rule |
| **Policy evaluation** | Preserve the Definition / Configuration split, enforce the closed outcome allow-list and permanent version pinning, coerce variables strictly, and fail closed |
| **Agent invocation** | Resolve agents through a registry, carry `Message.metadata.finos.*` invocation governance on A2A calls, and enforce trust-tier limits on data access and writes |
| **Provenance** | Emit a CloudEvents v1.0 vertex per node transition into an append-only graph, with the vertex and edge type vocabulary of the Execution History schema |
| **Credential handling** | Never resolve a credential field from workflow-authored input in a way that lets the same input both select a secret and choose where it is sent |

Extensions are expected. Add node types, outcome labels, vertex types, or context modes as your
domain requires — but do not change the meaning of the ones defined here, because interoperability
of agents and audit records across implementations depends on them.

### 3.3 Adoption Profiles

You do not have to take all of it.

| Profile | What you take | Typical reason |
|---|---|---|
| **Full** | All six specifications; generate servers and clients directly from the OpenAPI documents | You are building an orchestration platform and want the whole governance model |
| **Model-only** | The schemas as a design reference; your own services and transports implement them | You have existing infrastructure and want the contracts, not the implementation |
| **Provenance-only** | The Execution History schema and the CloudEvent vocabulary | You have an orchestrator you are happy with and need a defensible audit trail |
| **Policy-only** | The Definition / Configuration model and the evaluate contract | You need multi-tenant governance where tenants set thresholds but cannot widen authority |
| **Agent-interop** | The Agent Card schema and trust tier model | You publish agents that must be discoverable and governable across platforms |
| **Integration** | The Blackline endpoints in [§6–§10](#6-integration-pattern-1--control-console-integration) | You are connecting to a Blackline deployment |

The specifications are Apache 2.0 licensed. Implementing them creates no dependency on Blackline
software and no obligation to Blackline.

### 3.4 Namespace Placeholders

Four identifiers in these specifications are **wire identifiers** — values that travel between
systems and that consumers match on. They must be stable, and they must live in a namespace the
publisher actually controls. The specifications therefore do not fix a vendor namespace: examples
use placeholders that an implementation substitutes once, consistently, everywhere.

| Placeholder | Substitute with | Example |
|---|---|---|
| `{vendor-domain}` | An HTTPS domain you control; roots the A2A agent-card extension URI | `example.com` |
| `{event-namespace}` | A reverse-DNS prefix you own; roots the CloudEvents `type` | `com.example` |
| `{source-root}` | A path-form root identifying the emitting platform; roots the CloudEvents `source` | `/example` |
| `{api-host}` | The host serving the FinOS APIs; roots RFC 9457 problem `type` URIs | `api.example.com` |

**Only the namespace root is yours to choose.** The structure after it is normative:

```
https://{vendor-domain}/finos/agent-card/v1     →  https://example.com/finos/agent-card/v1
{event-namespace}.finos.{domain}.{action}        →  com.example.finos.agent.classify
{source-root}/finos/{domain}/{component}         →  /example/finos/agent/invoice-classifier
https://{api-host}/finos/errors/{slug}           →  https://api.example.com/finos/errors/not-found
```

That fixed structure is what lets a consumer written against one implementation recognise events
and agent cards from another by suffix matching, while still letting each implementation own its
own namespace. A conforming implementation substitutes all four and never mixes namespaces within
one deployment.

Substitution is a one-time deployment decision: changing a namespace after agents are registered
and events are flowing breaks every subscription and every registry validation keyed on the old
value.

Host names elsewhere in these documents (`api.example.com`, `erp-api.example.com`, and the like)
are RFC 2606 documentation names, not placeholders — replace them with your own hosts freely.

### 3.5 Cross-Cutting API Conventions

Every specification in this repository follows the same conventions. They are normative: a
conforming implementation does not vary them per service, because a client that can page or
filter one FinOS API should be able to page or filter all of them without a second code path.

#### Resource paths

Lowercase `kebab-case`, plural, with no verbs:

```
/workflows/{workflowId}/nodes/{nodeId}/runtime-status
/workflow-executions/{instanceId}/vertices/{vertexId}
/policy-definitions/{definitionId}/versions
/subscriptions/by-event-type/{subscriptionId}
```

Path parameters are `camelCase` (`{workflowId}`, `{instanceId}`, `{agentId}`), matching the field
names in request and response bodies. Entity names stay PascalCase in **schemas**
(`FinOSWorkflowExecutionHistory`) but never appear in a URL path.

#### API version placement

The major version lives in the **server URL**, never in a path:

```yaml
servers:
  - url: https://api.example.com/finos/v1      # ✅ version in the server URL
paths:
  /workflows:                                   # ✅ path carries no version
```

A path such as `/v1/agent-registry/agents` is wrong even though the assembled URL looks right,
because generated clients then carry the version in two places and `/health` ends up on a
different version prefix from the resources beside it. Every specification here declares `/v1` in
its `servers` block, so each spec has exactly one place to bump when `v2` arrives.

#### Identifier casing

One casing rule governs the whole wire surface: `camelCase` for every query parameter, path
parameter, and body property. A value accepted as a request parameter is returned under the same
name in the response — `pageSize` goes in and `pageSize` comes back, never `page_size` in and
`pageSize` out.

#### Pagination

Zero-based offset on every list endpoint:

| Param | Type | Default | Description |
|---|---|---|---|
| `pageSize` | integer ≥ 1 | 20–25 per service | Results per page |
| `pageStartIndex` | integer ≥ 0 | 0 | Zero-based index of the first result |

```
?pageSize=20&pageStartIndex=0     # first page
?pageSize=20&pageStartIndex=20    # second page
```

List responses return the same names in `meta.pagination`:

```json
{
  "data": [ ... ],
  "meta": {
    "requestId": "...",
    "timestamp": "...",
    "apiVersion": "1.0",
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

There is no `page`, no `cursor`, and no `totalPages`. Offset pagination is the single mode, so a
client never has to discover which of several schemes a given service implements.

#### Filtering

One mechanism — the FinOS filter expression — on every list endpoint:

```
filter=field operator "value" [and field operator "value"]
```

| Operator | Meaning |
|---|---|
| `contains` | Case-insensitive substring match |
| `eq` | Exact equality |

```
?filter=name contains "invoice"
?filter=tenantId eq "tenant-acme-prod" and status eq "ACTIVE"
```

Each specification documents the fields it supports. There is no `q` free-text parameter and no
per-field `nameContains`-style parameter; typed scalar parameters (`executionStatus`, `nodeId`,
`fromDate`) remain where a filter expression would be needlessly indirect.

#### Sorting

`sort` names a single field from a documented enum; `sortOrder` is `asc` or `desc`. The
`sort=-field` prefix convention is not used.

#### Mutability

Most resources support full lifecycle operations. **Execution History is append-only** — it
exposes no `PUT`, `PATCH`, or `DELETE` on an instance, vertex, or edge, because it is the audit
record the other services are attested against. See
[`1-finosworkflow-execution-history.md` §10](./1-finosworkflow-execution-history.md#10-immutability-guarantees).

---

## 4. Repository Contents

Six OpenAPI documents, one worked workflow example, and six developer guides. Each specification
is self-contained and independently adoptable — the Execution History schema does not require
the Workflow API, the Policy Service does not require the Agent Registry. Read them as
implementation contracts if you are building, or as integration contracts if you are connecting.

### API Specifications

| File | Type | Version | Description |
|---|---|---|---|
| [`finos-agent-registry-api.yaml`](./finos-agent-registry-api.yaml) | OpenAPI 3.1.0 | 1.0.0 | Agent Registry — register, discover, update, and deregister FinOS AI agents; enforces A2A Tier A validation |
| [`finos-event-service-api.yaml`](./finos-event-service-api.yaml) | OpenAPI 3.1.0 | 1.0.0 | Event Service — publish CloudEvent-compliant `FinOSEvent` documents; manage webhook subscriptions; serves as the FinOS event bus |
| [`finos-policy-api.yaml`](./finos-policy-api.yaml) | OpenAPI 3.1.0 | 1.0.0 | Policy Service — define, version, activate, and evaluate governance policies; produces the `policy_outcome` that drives DAG edge routing |
| [`finos-workflow-execution-history-openapi.yaml`](./finos-workflow-execution-history-openapi.yaml) | OpenAPI 3.1.0 | 1.0.0 | Execution History — CloudEvent-compliant DAG of vertices and edges representing the complete runtime audit trace of a workflow instance; **append-only** (no replace or delete) |
| [`finos-workflow-api.yaml`](./finos-workflow-api.yaml) | OpenAPI 3.0.3 | 1.0.0 | Workflow API — full lifecycle management of `FinOSWorkflow` DAG definitions; Node CRUD, Edge CRUD, DAG validation, and deployment |
| [`finos-agent-card-schema.yaml`](./finos-agent-card-schema.yaml) | OpenAPI 3.1.0 | 1.0.0 | FinOSAgentCard schema — A2A-native agent discovery and governance document; required for all agent registrations |

### Workflow Examples

| Example | Where |
|---|---|
| **Invoice Approval** — a complete, runnable workflow definition exercising six node types (`FinOSStart`, `FinOSAgent`, `FinOSAPI`, `FinOSPolicy`, `FinOSHITL`, `FinOSTerminal`) in a multi-tier approval DAG | Inline in [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) §12 |

### Developer Guides

| File | Description |
|---|---|
| [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) | FinOSWorkflow YAML anatomy, node type reference, edge condition syntax, HITL patterns, DAG integrity rules |
| [`1-finosworkflow-execution-history.md`](./1-finosworkflow-execution-history.md) | Execution History data model, vertex type hierarchy, edge semantics, CloudEvent compliance, provenance query patterns |
| [`2-finos-agent-card.md`](./2-finos-agent-card.md) | FinOSAgentCard schema reference, A2A protocol integration, context graph modes, invocation governance |
| [`3-finos-agent-registry-guide.md`](./3-finos-agent-registry-guide.md) | Agent registration flow, A2A Tier A validation, trust tier model, lifecycle management |
| [`4-finos-policy-guide.md`](./4-finos-policy-guide.md) | Policy definition model, evaluation flow, `policy_outcome` semantics, integration with DAG edge routing |
| [`5-finos-event-service-guide.md`](./5-finos-event-service-guide.md) | CloudEvent publishing, HITL callback lifecycle, event subscriptions, HMAC delivery |

---

## 5. Platform Architecture

```
External Systems                 Blackline Control Console             FinOS Platform
────────────────                 ─────────────────────────             ──────────────

┌──────────────┐                ┌───────────────────────────────────────────────────┐
│ CNAPP / SIEM │──────────────▶│                                                   │
│ Platform     │  publish       │  FinOS Event Service                              │
└──────────────┘  FinOSEvents  │  POST /event-service/v1/events                    │
                                │  (CloudEvents v1.0 + FinOS extensions)            │
┌──────────────┐                │  ← publishes execution events from workflow nodes │
│ Partner /    │──────────────▶│  ← ingests CNAPP, SIEM, governance events         │
│ Customer     │  register      │  ← drives HITL subscriptions                      │
│ Agent        │  agents        ├───────────────────────────────────────────────────┤
└──────────────┘                │  Agent Registry                                   │
                                │  POST /agent-registry/agents                      │
┌──────────────┐                │  (FinOSAgentCard · A2A Tier A validation)         │
│ HITL Review  │◀──────────────│  T0 Platform Native · T1 Certified Partner       │
│ Application  │  HITL events  │  T2 Registered External · T3 Provisional          │
│              │──────────────▶│                                                   │
│              │  callback POST ├───────────────────────────────────────────────────┤
└──────────────┘                │  Policy Service                                   │
                                │  PUT  /policy-definitions/{definitionId}          │
┌──────────────┐                │  POST /policies/{definitionId}/evaluate           │
│ Execution    │◀──────────────│  outcome: Success · Escalate · AttentionRequired  │
│ Monitoring   │  query DAG    │  · Failed → drives policy_outcome edges           │
│ UI           │  status       ├───────────────────────────────────────────────────┤
└──────────────┘                │                         │                         │
                                │  Workflow API           │  FinOS Workflows Engine  │
┌──────────────┐                │  POST /workflows    │  (runtime)               │
│ DAG Authoring│──────────────▶│  Nodes · Edges · Export │                          │
│ UI           │  CRUD DAG     │  Validate · Run          │  FinOSAgent (A2A)        │
└──────────────┘  definitions  │                         │  FinOSAPI (REST)          │
                                │                         │  FinOSHITL (callback)    │
                                │                         │  FinOSPolicy (evaluate)  │
                                └─────────────────────────┼──────────────────────────┘
                                                           │ posts event after every node
                                                           ▼
                                          ┌────────────────────────────────┐
                                          │  FinOSWorkflowExecutionHistory  │
                                          │  (immutable audit DAG)          │
                                          │  15 vertex types · 18 edge types│
                                          │  CloudEvent-compliant vertices  │
                                          └────────────────────────────────┘
```

**Core execution flow for a FinOSAgent node:**

```
FinOS Workflows Engine
  │
  │  1. Resolve agent   GET /agent-registry/agents/{agentId}
  │  2. Discover card   GET /.well-known/agent-card.json         (A2A)
  │  3. Invoke agent    POST /message:send                        (A2A)
  │     body: Message.metadata.finos.*
  │           = callingContext + contextGraphs + policies + variableBindings
  │
  ▼
FinOS Agent Runtime
  │
  │  4. Validate FinOS governance metadata
  │  5. Resolve context graphs
  │     (RAG_QUERY · FINOSWORKFLOW_GRAPH · STRUCTURED_CHUNKS · RAG_TENSOR)
  │  6. Execute LLM reasoning loop
  │     (Anthropic Claude · OpenAI GPT · vendor-tuned models)
  │  7. Return A2A Task  (COMPLETED · FAILED · INPUT_REQUIRED)
  │  8. Emit FinOSAgentEventVertex → FinOSWorkflowExecutionHistory
```

---

> **The five patterns that follow** are written against Blackline endpoints, but each is also a
> worked reference implementation of the model in [§3](#3-the-specification-as-a-standard) —
> agent registration and trust assignment, event publication and subscription, policy
> configuration and evaluation, provenance retrieval, and DAG authoring with validation. If you
> are building your own control plane rather than integrating with this one, read the request and
> response shapes as the contract and ignore the hostnames.

## 6. Integration Pattern 1 — Control Console Integration

This pattern covers systems that establish an ongoing integration with Blackline Control Console
to manage agents, observe events, administer policies, and query workflow execution state. This
is the primary integration pattern for enterprise Finance Operations platforms, SI-built
orchestration layers, and BPO automation platforms.

**APIs used:**
- `finos-agent-registry-api.yaml` — Agent Registration
- `finos-event-service-api.yaml` — Event Publishing and Subscription
- `finos-policy-api.yaml` — Policy Management
- `finos-workflow-execution-history-openapi.yaml` — Execution Status and Audit

---

### 6.1 Registering and Deregistering Agents

The Agent Registry is the authoritative catalog of all agents operating in the FinOS platform.
Every agent — regardless of who built it — must be registered before it can be invoked within a
workflow. Control Console uses registry state to enforce trust-tier-based data access, require HITL
for elevated-risk operations, and attribute every action in the audit trail to a named agent identity.

#### Agent Categories and Trust Tiers

| Agent Category | Trust Tier | Who Registers | Data Access | Write Permissions |
|---|---|---|---|---|
| Platform Native | `T0` | Platform operations | All data including RESTRICTED | Full write access; HITL not required |
| Certified Partner | `T1` | Certified SI/BPO/ISV | CONFIDENTIAL and below | Write with policy gate; HITL required for RESTRICTED |
| Registered External | `T2` | External / Customer team | INTERNAL and below | Own namespace only; HITL required for CONFIDENTIAL+ |
| Provisional | `T3` | Self-registered (unvetted) | PUBLIC data only | Read-only; HITL required for all actions |

#### Registration Prerequisite: FinOSAgentCard

Every agent registration requires a **FinOSAgentCard** document — the A2A-compatible discovery
document served at `GET /.well-known/agent-card.json` on the agent's runtime endpoint.

The card must declare a FinOS-specific extension under `capabilities.extensions[]` with:
- `uri: "https://{vendor-domain}/finos/agent-card/v1"` and `required: true`
- `params.finOSAgentId` — a UUID that will become the permanent registry identifier
- `params.finOSAgentType` — one of `ORCHESTRATOR · PLANNER · EXECUTOR · REVIEWER · MONITOR · EXTRACTOR · CLASSIFIER · SUMMARIZER · CUSTOM`
- `params.finOSLlmModels` — ordered list of LLMs this agent may use (e.g., `claude-sonnet-4-6`, `gpt-4o`)

Setting `required: true` on the extension tells any A2A caller that does not support FinOS
governance it must not invoke this agent.

```jsonc
// Minimal FinOSAgentCard for registration
{
  "name": "InvoiceClassifierAgent",
  "description": "Classifies incoming invoices by vendor category and spend tier.",
  "version": "2.1.0",
  "skills": [
    { "id": "invoice-classify", "name": "Invoice Classification" }
  ],
  "capabilities": {
    "extensions": [
      {
        "uri": "https://{vendor-domain}/finos/agent-card/v1",
        "required": true,
        "params": {
          "finOSAgentId": "7f3d8c1a-0000-0000-0000-000000000001",
          "finOSAgentType": "CLASSIFIER",
          "finOSLlmModels": [
            { "modelId": "claude-sonnet-4-6", "provider": "anthropic", "isDefault": true }
          ],
          "finOSSupportedContextModes": ["RAG_QUERY", "STRUCTURED_CHUNKS"]
        }
      }
    ]
  },
  "supportedInterfaces": [
    { "protocol": "A2A/1.0", "url": "https://agents.example.com/invoice-classifier/message:send" }
  ],
  "securitySchemes": { "bearerAuth": { "type": "http", "scheme": "bearer" } },
  "securityRequirements": [{ "bearerAuth": ["finos:agent:invoke"] }],
  "defaultInputModes": ["application/json"],
  "defaultOutputModes": ["application/json"]
}
```

**A2A Tier A Validation** — the registry enforces seven checks on every `POST` and card `PUT`:

| # | Check | What is validated |
|---|---|---|
| 1 | Schema | All required A2A fields present: `name`, `description`, `version`, `skills`, `capabilities`, `supportedInterfaces`, `securitySchemes`, `securityRequirements`, `defaultInputModes`, `defaultOutputModes` |
| 2 | FinOS Extension | `capabilities.extensions[]` contains entry with `uri: "https://{vendor-domain}/finos/agent-card/v1"` and `required: true` |
| 3 | Extension Params | `finOSAgentId` (UUID), `finOSAgentType` (valid enum), `finOSLlmModels` (non-empty) all present in extension `params` |
| 4 | Skills | `skills[]` non-empty; each item has `id` and `name` |
| 5 | Interfaces | `supportedInterfaces[]` has ≥ 1 entry with a reachable `url` |
| 6 | Security | `securitySchemes` and `securityRequirements` non-empty |
| 7 | Uniqueness | No active agent with the same `finOSAgentId` in the tenant |

#### Register an Agent

```bash
# POST the FinOSAgentCard to register
curl -X POST https://api.example.com/finos/v1/agent-registry/agents \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: invoice-classifier-v2-1-0" \
  -d @invoice-classifier-agent-card.json
# → 201 Created  { "agentId": "7f3d8c1a-...", "status": "active", "trustTier": "T1" }
```

#### Agent Lifecycle

Agents progress through a defined lifecycle. Once `revoked`, an agent can never be
reactivated. Use `inactive` for temporary deregistration; use `revoked` only for permanent
removal (e.g., security incident or end-of-life).

```
POST /agents         → active
PUT  { status: "inactive" }  ← temporary deregistration; audit record preserved
PUT  { status: "suspended" } ← compliance hold; transitions back to active or inactive
PUT  { status: "revoked" }   ← permanent; TERMINAL — cannot be reactivated
DELETE /agents/{id}          ← soft delete; sets status to inactive
```

#### Deregister an Agent

```bash
# Soft deregister — sets status to inactive, preserves the audit record
curl -X DELETE \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN"
# → 204 No Content

# Or use PUT to set a specific status (e.g., revoked)
curl -X PUT \
  "https://api.example.com/finos/v1/agent-registry/agents/7f3d8c1a-0000-0000-0000-000000000001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "revoked" }'
```

#### Discover Registered Agents

```bash
# List all active certified-partner agents
curl "https://api.example.com/finos/v1/agent-registry/agents
  ?filter=status%20eq%20%22active%22%20and%20role%20eq%20%22operational%22
  &sort=createdAt&sortOrder=desc
  &pageSize=20&pageStartIndex=0" \
  -H "Authorization: Bearer $TOKEN"
```

**Filter expression syntax** (`field operator "value"` joined by `and`):

| Field | Operators | Example |
|---|---|---|
| `name` | `contains`, `eq` | `name contains "match"` |
| `provider` | `eq` | `provider eq "Example Corp"` |
| `version` | `eq` | `version eq "2.1.0"` |
| `role` | `eq` | `role eq "operational"` |
| `status` | `eq` | `status eq "active"` |

> See [`3-finos-agent-registry-guide.md`](./3-finos-agent-registry-guide.md) for the complete
> registration flow, A2A Tier A validation details, and trust tier governance rules.

---

### 6.2 Publishing and Consuming FinOS Events

The FinOS Event Service is the platform-wide event bus. Every significant action in a running
workflow — agent invocation, policy evaluation, human review task creation, API call — is
published as a `FinOSEvent`. Control Console subscribes to these events to drive its real-time
observability dashboard, populate the audit trail, and trigger downstream notifications.

External systems use the same event bus to publish governance events into FinOS (covered in
[Pattern 2](#7-integration-pattern-2--cnapp-and-siem-event-ingestion)) and to receive
workflow events via webhook subscription.

#### FinOSEvent Schema

Every `FinOSEvent` is a **CloudEvents v1.0** document extended with FinOS governance fields.
The `finOSEventType` field is the discriminator for four concrete subtypes:

| `finOSEventType` | `type` prefix | When emitted |
|---|---|---|
| `FinOSAgentEvent` | `{event-namespace}.finos.agent.*` | LLM agent invocation, skill execution, tool call, HITL pause |
| `FinOSAPIEvent` | `{event-namespace}.finos.api.*` | External REST API call and response from a `FinOSAPI` node |
| `FinOSHITLEvent` | `{event-namespace}.finos.hitl.*` | Human-in-the-loop task created, completed, or timed out |
| `FinOSPolicyEvent` | `{event-namespace}.finos.policy.*` | `FinOSPolicy` node evaluation with outcome and applied rules |

**Required CloudEvents base fields present on every event:**

| Field | Example |
|---|---|
| `id` | `"evt-classify-001"` |
| `source` | `"{source-root}/finos/agent/invoice-classifier"` |
| `specversion` | `"1.0"` |
| `type` | `"{event-namespace}.finos.agent.classify"` |
| `time` | `"2026-06-01T09:00:10Z"` |

**Additional FinOS governance fields:**

| Field | Required | Description |
|---|---|---|
| `finOSEventType` | ✓ | Subtype discriminator |
| `tenantId` | ✓ | Tenant identifier |
| `correlationId` | — | Links related events across a workflow session |
| `priority` | — | `LOW · MEDIUM · HIGH · CRITICAL` |
| `workflowId` | — | Workflow definition identifier |
| `workflowInstanceId` | — | Specific execution instance |

#### Publish a Standalone FinOSEvent

```bash
# Publish a FinOSAgentEvent from an agent outside a workflow context
curl -X POST https://api.example.com/event-service/v1/events \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: evt-classify-001" \
  -d '{
    "id": "evt-classify-001",
    "source": "{source-root}/finos/agent/invoice-classifier",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.agent.classify",
    "time": "2026-06-01T09:00:10Z",
    "subject": "invoice/INV-2026-00441",
    "finOSEventType": "FinOSAgentEvent",
    "tenantId": "tenant-acme-corp",
    "correlationId": "corr-7890abcd",
    "priority": "HIGH",
    "agent": {
      "agentId": "7f3d8c1a-0000-0000-0000-000000000001",
      "agentName": "InvoiceClassifierAgent",
      "agentType": "CLASSIFIER",
      "agentVersion": "2.1.0"
    },
    "agentRun": {
      "runId": "run-aaaabbbb",
      "runStatus": "COMPLETED",
      "startTime": "2026-06-01T09:00:08Z",
      "endTime": "2026-06-01T09:00:10Z",
      "tokenUsage": { "inputTokens": 1240, "outputTokens": 340, "totalTokens": 1580 }
    }
  }'
# → 201 Created   Location: /event-service/v1/events/evt-classify-001
```

> **Deduplication:** re-posting an event with the same `id` + `source` within the deduplication
> window (default 24 h) returns `200` with the original event rather than creating a duplicate.

#### Publish in Workflow Context

Events can be scoped to a specific workflow execution. Path parameters take precedence over
body fields:

```bash
curl -X POST \
  "https://api.example.com/event-service/v1/workflows/wf-invoice-v1/executions/inst-abc123/events" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ ... same event body ... }'
```

#### Subscribe to Receive Events (Webhook Push)

Register a `callbackUrl` to receive matching events via HTTP POST. Three subscription scopes
are supported:

| Scope | Endpoint | Filter |
|---|---|---|
| By event type | `POST /subscriptions/by-event-type` | One or more `finOSEventType` values |
| By workflow | `POST /subscriptions/by-workflow` | `workflowId` (any execution) |
| By execution | `POST /subscriptions/by-execution` | `workflowId` + `workflowExecutionId` |

```bash
# Subscribe to all FinOSHITLEvent and FinOSPolicyEvent occurrences across the tenant
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-event-type \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://your-system.example.com/webhooks/finos",
    "filter": { "eventTypes": ["FinOSHITLEvent", "FinOSPolicyEvent"] },
    "deliveryConfig": {
      "secret": "your-hmac-signing-secret",
      "maxRetries": 3,
      "timeoutMs": 5000
    }
  }'
# → 201 Created   { "subscriptionId": "sub-uuid-001", "status": "active" }
```

**Delivery envelope** — your `callbackUrl` receives:

```jsonc
{
  "subscriptionId": "sub-uuid-001",
  "deliveredAt": "2026-06-01T09:05:00Z",
  "event": { /* full FinOSEvent document */ },
  "signature": "sha256=<hmac-sha256-hex>"  // HMAC-SHA256 of the body using your secret
}
```

Verify the `signature` header against the raw request body before processing. Retry behaviour:
- `2xx` → acknowledged, no retry
- `4xx` → not retried (client rejected event; investigate payload)
- `5xx` or timeout → retried with exponential backoff up to `maxRetries`

> See [`5-finos-event-service-guide.md`](./5-finos-event-service-guide.md) for the full event
> schema reference, all `finOSEventType` field tables, subscription lifecycle management, and HMAC
> verification details.

---

### 6.3 Managing Policy Configurations

The FinOS Policy Service is the governance rule engine that gates agent and workflow actions.
`FinOSPolicy` nodes in a workflow DAG call this service at runtime; the `outcome` they return
drives conditional edge routing. Control Console integrations use the Policy API to create new
policies, update existing configurations, monitor evaluation history, and ensure that governance
rules stay current with evolving compliance requirements.

#### Policy Outcomes and DAG Routing

Every `POST /policies/{policyDefinitionId}/evaluate` returns an outcome drawn from the pinned
Policy Definition's `possibleOutcomes` — a **closed allow-list**. The standard set is:

| Outcome | Meaning | Typical DAG routing |
|---|---|---|
| `Success` | Policy conditions met; action is permitted | Route to the next automated node |
| `Escalate` | Conditions require human review before proceeding | Route to a `FinOSHITL` node |
| `AttentionRequired` | Action may proceed but warrants human notification | Route to HITL or emit notification event |
| `Failed` | Policy conditions violated; action is blocked | Route to a rejection `FinOSTerminal` |

A Definition may declare additional custom labels (for example `NO_ITEMS` or
`INSUFFICIENT_DATA` for structurally degenerate inputs). Whatever the set, **every** outcome the
node can produce must have a matching outgoing edge condition, or registration is rejected.

The `policy_outcome` variable is set in the workflow DAG immediately after the evaluation step
and is referenced in outgoing edge conditions. Edges carry no type discriminator — the condition
alone determines the branch, and edges from one source are evaluated in `priority` order with
first match winning:

```yaml
# Edge conditions are jq expressions enclosed in ${ }
edges:
  - edgeId: e-escalate
    sourceNodeId: spending-policy
    targetNodeId: manager-review
    condition: '${ .policy_outcome == "Escalate" or .policy_outcome == "AttentionRequired" }'
    priority: 1

  - edgeId: e-success
    sourceNodeId: spending-policy
    targetNodeId: post-approval
    condition: '${ .policy_outcome == "Success" }'
    priority: 2

  - edgeId: e-failed
    sourceNodeId: spending-policy
    targetNodeId: terminal-rejected
    condition: '${ .policy_outcome == "Failed" }'
    priority: 3
```

#### Create a Policy Definition

A **Policy Definition** is the tenant-agnostic *shape* — the variable catalog and the closed
allow-list of outcomes. It carries no condition, no enforcement, and no tenant.

```bash
curl -X POST https://api.example.com/finos-policy-service/v1/policy-definitions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Invoice Spending Limit",
    "description": "Shape for invoice spending-limit policies: amount + risk inputs.",
    "variables": [
      { "name": "invoiceAmount",      "type": "NUMBER", "source": "EVALUATION_INPUT", "required": true },
      { "name": "approvedSpendLimit", "type": "NUMBER", "source": "EVALUATION_INPUT", "required": true },
      { "name": "vendorRiskScore",    "type": "NUMBER", "source": "EVALUATION_INPUT", "required": false }
    ],
    "possibleOutcomes": ["Success", "AttentionRequired", "Escalate", "Failed"]
  }'
# → 201 Created; starts in DRAFT at version 1.0.0

# Activate it before tenants can configure against it
curl -X PATCH https://api.example.com/finos-policy-service/v1/policy-definitions/def-spending-limit/status \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "status": "ACTIVE" }'
```

Subsequent saves under a caller-supplied id use the idempotent upsert — an unchanged payload is a
no-op, a changed one bumps the minor version:

```bash
curl -X PUT https://api.example.com/finos-policy-service/v1/policy-definitions/def-spending-limit \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d @definition.json
```

#### Author a Policy Configuration

A **Policy Configuration** is the tenant's actual rule at one workflow node placement — the CEL
`condition` and its `enforcement` — pinned to a Definition version at save time. `PUT` is the only
write verb: it creates on first call and version-bumps thereafter.

```bash
curl -X PUT \
  "https://api.example.com/finos-policy-service/v1/policy-configurations/spending-policy/wfc-invoice-acme" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "blueprintId": "wf-invoice-approval-v1",
    "policyDefinitionId": "def-spending-limit",
    "policyDefinitionVersion": 1,
    "condition": {
      "combinator": "OR",
      "subConditions": [
        { "name": "over-cap",   "expression": "invoiceAmount > 500000.0",
          "variables": [ { "name": "invoiceAmount", "type": "NUMBER", "source": "EVALUATION_INPUT" } ] },
        { "name": "over-limit", "expression": "invoiceAmount > approvedSpendLimit",
          "variables": [ { "name": "invoiceAmount",      "type": "NUMBER", "source": "EVALUATION_INPUT" },
                         { "name": "approvedSpendLimit", "type": "NUMBER", "source": "EVALUATION_INPUT" } ] }
      ]
    },
    "enforcement": {
      "kind": "TIERED",
      "outcomes": { "over-cap": "Failed", "over-limit": "Escalate" },
      "defaultOutcome": "Success"
    }
  }'
# → 201 Created at version 1.0.0, with a server-minted configurationId (cfg_<uuid>)
```

Tiers are evaluated in the order they appear in `condition.subConditions`; the outcome mapped to
the **first** matching tier wins. Every outcome referenced must be a member of the pinned
Definition version's `possibleOutcomes`, or the save is rejected with `422`.

Changing a threshold is the same call with a new payload — it bumps the minor version and writes a
new immutable snapshot. The pinned Definition version never moves, so a later Definition edit can
never retroactively change a live rule.

#### Evaluate a Policy

```bash
curl -X POST \
  "https://api.example.com/finos-policy-service/v1/policies/def-spending-limit/evaluate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "workflowInstanceId": "5281a21d-6ccd-4886-9675-678e36d66f32",
    "workflowId": "wf-invoice-approval-v1",
    "nodeId": "spending-policy",
    "workflowConfigId": "wfc-invoice-acme",
    "evaluationInputs": {
      "invoiceAmount": 82000,
      "approvedSpendLimit": 50000,
      "vendorRiskScore": 4.2
    }
  }'
# → 200 OK  { "data": { "outcome": "Escalate", "action": "ESCALATE", "triggered": true,
#                       "policyConfigurationId": "cfg_c6b766a2", ... } }
```

The service resolves the calling tenant's Configuration — directly by `policyConfigurationId` when
the run configuration carries one, otherwise by the `(workflowConfigId, nodeId)` node tuple — and
checks that its `policyDefinitionId` matches the path.

> **Fail-closed.** A missing Configuration, a Definition-id mismatch, a condition that fails to
> compile, or a strict type-coercion failure all return `200` with a deterministic
> `outcome: "Failed" / action: "BLOCK"` envelope and the cause in `reason` — never a `5xx`.
> Inspect `outcome` and `action`, not the HTTP status.

> See [`4-finos-policy-guide.md`](./4-finos-policy-guide.md) for the full policy definition
> model, evaluation context schema, platform-enforced invariants (SOD, closed period lock), and
> integration with `policy_outcome` edge routing.

---

### 6.4 Retrieving Workflow Execution Status and History

The `FinOSWorkflowExecutionHistory` API provides the runtime audit record for every workflow
execution. It models the complete execution trace as a **Directed Acyclic Graph (DAG)** of typed
vertices and edges — every agent action, policy evaluation, human decision, and API call is
recorded as a CloudEvent-compliant vertex. This is the primary data source for Control Console's
audit trail, real-time observability dashboard, and SOX evidence chain.

#### Execution Status Values

| `executionStatus` | Meaning |
|---|---|
| `PENDING` | Execution is queued but has not started |
| `RUNNING` | Execution is active; nodes are being processed |
| `COMPLETED` | All paths reached terminal nodes successfully |
| `FAILED` | A node failed and no recovery path exists |
| `CANCELLED` | Execution was explicitly cancelled |
| `PAUSED` | Execution is suspended at a `FinOSHITL` node awaiting callback |
| `ARCHIVED` | Execution is complete and soft-archived (still queryable) |

#### List Execution Instances

```bash
# All currently running executions for the tenant, newest first
curl "https://api.example.com/finos/v1/workflow-executions
  ?executionStatus=RUNNING
  &sort=executionStartTime&sortOrder=desc
  &pageSize=20&pageStartIndex=0" \
  -H "Authorization: Bearer $TOKEN"
```

#### Retrieve a Full Execution Record

```bash
# Full DAG — vertices + edges — for a specific execution
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64-5717-4562-b3fc-2c963f66afa6
  ?embed=full" \
  -H "Authorization: Bearer $TOKEN"
```

**`embed` parameter values:**

| Value | Returns |
|---|---|
| `none` | Status and metadata only (no vertices or edges) |
| `vertices` | Status, metadata, and `finOSVertices[]` |
| `edges` | Status, metadata, and `finOSEdges[]` |
| `full` | Complete execution DAG |

#### Query Vertices by Type

```bash
# Retrieve only the HITL decision records from an execution
curl "https://api.example.com/finos/v1/workflow-executions/3fa85f64.../vertices
  ?vertexType=FinOSHITLEvent" \
  -H "Authorization: Bearer $TOKEN"
```

#### Retrieve the Full DAG

```bash
# adjacency format — vertex list with outgoing edge map (default)
curl ".../3fa85f64.../graph?format=adjacency" -H "Authorization: Bearer $TOKEN"

# edgelist format — flat list of (source, edge, target) triples
curl ".../3fa85f64.../graph?format=edgelist" -H "Authorization: Bearer $TOKEN"
```

#### Trace a Provenance Path

Find the shortest directed path between two vertices in the DAG — for example, trace an approval
decision back to the agent event that triggered the HITL:

```bash
curl ".../3fa85f64.../graph/path?sourceVertexId=v-001&targetVertexId=v-009" \
  -H "Authorization: Bearer $TOKEN"
```

> See [`1-finosworkflow-execution-history.md`](./1-finosworkflow-execution-history.md) for the
> complete vertex and edge type hierarchy, DAG integrity rules, the append-only immutability
> guarantees, and provenance query patterns.

---

## 7. Integration Pattern 2 — CNAPP and SIEM Event Ingestion

Security and compliance platforms — Cloud-Native Application Protection Platforms (CNAPP),
Security Information and Event Management systems (SIEM), GRC platforms — can publish their
governance findings, alerts, and audit events directly into the FinOS event bus. Once ingested,
these events appear in the Control Console audit trail, can trigger policy evaluations, and can
initiate FinOS workflows if configured to do so.

This pattern enables Control Console to serve as a unified governance surface for both
finance-native agent events and cross-platform security and compliance events.

#### Event Wrapping Convention

CNAPP and SIEM events must be expressed as **CloudEvents v1.0** documents and wrapped in a
`FinOSEvent` with `finOSEventType: FinOSAgentEvent` (for agent-sourced security events) or
`finOSEventType: FinOSAPIEvent` (for system-level findings from non-agent sources). Use the
reverse-DNS `type` field to identify the originating system:

```
type: "{event-namespace}.finos.cnapp.{platform}.{event-category}.{severity}"
type: "{event-namespace}.finos.siem.{platform}.{rule-name}.{classification}"
```

**Recommended CNAPP event type conventions:**

| Originating system | Suggested `type` | `finOSEventType` |
|---|---|---|
| CNAPP policy violation | `{event-namespace}.finos.cnapp.policy-violation.high` | `FinOSPolicyEvent` |
| CNAPP anomaly detection | `{event-namespace}.finos.cnapp.anomaly.drift` | `FinOSAgentEvent` |
| SIEM alert | `{event-namespace}.finos.siem.alert.critical` | `FinOSAPIEvent` |
| GRC control failure | `{event-namespace}.finos.grc.control-failure.sox` | `FinOSPolicyEvent` |

#### Publishing a CNAPP Finding to FinOS

```bash
curl -X POST https://api.example.com/event-service/v1/events \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: cnapp-finding-2026-06-01-0042" \
  -d '{
    "id": "cnapp-finding-0042",
    "source": "/external/cnapp/prisma-cloud",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.cnapp.policy-violation.high",
    "time": "2026-06-01T14:22:00Z",
    "subject": "agent/7f3d8c1a-0000-0000-0000-000000000001",
    "finOSEventType": "FinOSPolicyEvent",
    "tenantId": "tenant-acme-corp",
    "correlationId": "corr-cnapp-session-009",
    "priority": "HIGH",
    "data": {
      "findingId":    "CNAPP-HIGH-0042",
      "policyName":  "Agent Data Exfiltration Attempt",
      "severity":    "HIGH",
      "resource":    "finos-agent/7f3d8c1a-0000-0000-0000-000000000001",
      "description": "Agent attempted to access data classified above its T1 trust tier.",
      "recommendation": "Suspend agent and review access patterns."
    },
    "finOSPolicyId":                 "pol-agent-data-access-v1",
    "finOSPolicyEvaluationOutcome":  "Failed"
  }'
```

#### Publishing a SIEM Alert

```bash
curl -X POST https://api.example.com/event-service/v1/events \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "siem-alert-0089",
    "source": "/external/siem/sentinel",
    "specversion": "1.0",
    "type": "{event-namespace}.finos.siem.alert.critical",
    "time": "2026-06-01T14:30:00Z",
    "subject": "workflow/wf-invoice-approval-v1/execution/inst-abc123",
    "finOSEventType": "FinOSAPIEvent",
    "tenantId": "tenant-acme-corp",
    "priority": "CRITICAL",
    "data": {
      "alertId":      "SIEM-CRIT-0089",
      "ruleName":     "Unusual Approval Pattern",
      "description":  "Agent approved 847 invoices in 90 seconds — outside behavioral baseline.",
      "affectedAsset": "wf-invoice-approval-v1",
      "recommendation": "Pause workflow execution and review approval chain."
    },
    "endpoint":            "https://api.example.com/finos/v1/workflows/wf-invoice-approval-v1",
    "httpMethod":          "GET",
    "responseStatusCode":  200
  }'
```

#### Subscribing to Receive Inbound CNAPP/SIEM Events

If your platform needs to receive confirmations or downstream FinOS events triggered by the
ingested findings (e.g., to confirm that Control Console acted on your alert), subscribe using a
correlation filter:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-event-type \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "callbackUrl": "https://your-cnapp-platform.example.com/finos-outcomes",
    "filter": { "eventTypes": ["FinOSPolicyEvent", "FinOSHITLEvent"] },
    "deliveryConfig": { "secret": "your-secret", "maxRetries": 3 }
  }'
```

---

## 8. Integration Pattern 3 — Policy Event Response and HITL Implementation

When a `FinOSPolicy` node evaluates to `Escalate` or `AttentionRequired`, the DAG routes to a
`FinOSHITL` node. This node suspends the workflow, creates a callback endpoint, and publishes a
`FinOSHITLEvent` to the event bus. An external system — a review portal, a notification service,
a Control Console UI — receives this event, presents the pending decision to a human reviewer, and
resumes the workflow by POSTing the reviewer's decision to the callback URL.

This pattern covers the full HITL integration loop.

#### The HITL Lifecycle

```
FinOS Workflows Engine
  │
  │  FinOSPolicy node evaluates → outcome: "Escalate"
  │  The matching policy_outcome edge routes to the FinOSHITL node
  │
  │  1. Creates callback endpoint (internal)
  │     callbackUrl = "https://workflows.example.com/v1/.../callbacks/{id}"
  │
  │  2. Publishes FinOSHITLEvent to event bus:
  │     {
  │       "finOSEventType": "FinOSHITLEvent",
  │       "type": "{event-namespace}.finos.hitl.task.created",
  │       "callbackUrl": "https://workflows.example.com/v1/.../callbacks/{id}",
  │       "taskType": "REVIEW",
  │       "assignedTo": "FinanceManager",
  │       "dueDate": "2026-06-03T09:00:00Z",
  │       "timeoutAt": "2026-06-03T09:00:00Z"
  │     }
  │
  │  3. Workflow SUSPENDS at await_callback step
  │     (executionStatus → PAUSED)
  │
  │  ... waits for human decision ...
  │
  │  4. Human system POSTs decision to callbackUrl
  │
  │  5. Workflow RESUMES
  │     (executionStatus → RUNNING)
  │     RESUMED_BY edge is recorded in FinOSWorkflowExecutionHistory
  │
  ▼
Next DAG node (APPROVED_BY or REJECTED_BY edge traversed)
```

#### Step 1: Subscribe to FinOSHITLEvent

Your review system subscribes to receive HITL task notifications:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-event-type \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "callbackUrl": "https://your-review-portal.example.com/finos-hitl",
    "filter": { "eventTypes": ["FinOSHITLEvent"] },
    "deliveryConfig": { "secret": "review-portal-hmac-secret", "maxRetries": 5 }
  }'
```

#### Step 2: Receive the FinOSHITLEvent

When a HITL task is created, your `callbackUrl` receives:

```jsonc
{
  "subscriptionId": "sub-hitl-001",
  "deliveredAt":    "2026-06-01T09:10:00Z",
  "event": {
    "id":              "evt-hitl-mgr-review-001",
    "source":          "{source-root}/finos/workflow/wf-invoice-approval-v1",
    "specversion":     "1.0",
    "type":            "{event-namespace}.finos.hitl.task.created",
    "time":            "2026-06-01T09:10:00Z",
    "finOSEventType":  "FinOSHITLEvent",
    "tenantId":        "tenant-acme-corp",
    "workflowId":      "wf-invoice-approval-v1",
    "workflowInstanceId": "inst-abc123",
    "callbackUrl":     "https://workflows.example.com/v1/callbacks/cb-mgr-001",
    "taskType":        "REVIEW",
    "taskDescription": "Invoice INV-2026-00441 for $82,000 requires FinanceManager approval.",
    "assignedTo":      "FinanceManager",
    "dueDate":         "2026-06-03T09:00:00Z",
    "timeoutAt":       "2026-06-03T09:00:00Z",
    "priorityLevel":   "HIGH"
  },
  "signature": "sha256=<hmac>"
}
```

#### Step 3: Present the Decision to a Human Reviewer

Your system:
1. Verifies the HMAC signature against the raw request body
2. Looks up the full execution context from the Execution History API
3. Presents the task, the agent's reasoning, and the relevant workflow data to the reviewer

```bash
# Retrieve the execution context for the review UI
curl "https://api.example.com/finos/v1/workflow-executions/inst-abc123?embed=full" \
  -H "Authorization: Bearer $TOKEN"
```

#### Step 4: POST the Reviewer Decision to the Callback URL

The reviewer's decision resumes the workflow:

```bash
# Approved
curl -X POST "https://workflows.example.com/v1/callbacks/cb-mgr-001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "decision":      "APPROVED",
    "reviewerId":    "user-finance-mgr-001",
    "justification": "Vendor is on approved list; spending within Q2 budget.",
    "completedAt":   "2026-06-01T11:30:00Z"
  }'

# Rejected
curl -X POST "https://workflows.example.com/v1/callbacks/cb-mgr-001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "decision":      "REJECTED",
    "reviewerId":    "user-finance-mgr-001",
    "justification": "Vendor contract expired. Reject and return to procurement.",
    "completedAt":   "2026-06-01T11:32:00Z"
  }'
```

**HITL Decision Values:** No canonical values are mandated. Workflow authors
define their own vocabulary in `allowedDecisions`. The submitted decision is
available as `${ .decision }` in workflow context for edge routing.

After the callback is received, the workflow resumes:
- A `RESUMED_BY` edge is added to the execution DAG in `FinOSWorkflowExecutionHistory`
- The HITL node's `decision` field becomes available as a workflow variable
- `APPROVED_BY` or `REJECTED_BY` edges are traversed based on the decision

#### Timeout Handling

If the reviewer does not act before `timeoutAt`, the workflow escalates automatically.
A `FinOSHITLEvent` with `type: "{event-namespace}.finos.hitl.task.timed-out"` is published to
the event bus. Your system should subscribe to this type to surface timeout notifications:

```bash
# Subscribe to timeout events alongside task creation
"filter": { "eventTypes": ["FinOSHITLEvent"] }
# All FinOSHITLEvent sub-types are delivered: task.created, task.completed, task.timed-out
```

> See [`5-finos-event-service-guide.md`](./5-finos-event-service-guide.md) for the full
> `FinOSHITLEvent` field reference and [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md)
> for the HITL pause-and-resume pattern in the FinOS Workflows YAML.

---

## 9. Integration Pattern 4 — Workflow Execution Monitoring UI

Systems building observability dashboards, audit trail viewers, or real-time workflow status
interfaces consume the `FinOSWorkflowExecutionHistory` API and the `FinOS Event Service` to
render the execution state of running and completed workflows. This pattern supports Control
Tower's real-time visibility panel, Finance Automation Manager exception queues, and external
audit access surfaces.

#### Core Data Model

```
FinOSWorkflowExecutionHistory (instanceId: UUID)
├── executionStatus  PENDING | RUNNING | COMPLETED | FAILED | CANCELLED | PAUSED | ARCHIVED
├── executionStartTime / executionEndTime
├── finOSVertices[]     15 typed vertex types — every node execution, event, decision
└── finOSEdges[]        18 typed directed edge types — causal and temporal relationships
```

**Vertex types relevant to execution UIs:**

| Vertex type | What it represents | Key fields |
|---|---|---|
| `NodeInstance` | A workflow node execution | `nodeId`, `nodeType`, `status`, `startTime`, `endTime` |
| `FinOSAgentEvent` | LLM agent action | `agentId`, `agentType`, `llmModelName`, `runStatus`, `tokenUsage` |
| `FinOSHITLEvent` | Human review task | `taskType`, `assignedTo`, `decision`, `callbackUrl`, `completedAt` |
| `FinOSPolicyEvent` | Policy evaluation | `finOSPolicyId`, `finOSPolicyEvaluationOutcome`, `appliedRules` |
| `FinOSAPIEvent` | External API call | `endpoint`, `httpMethod`, `responseStatusCode`, `latencyMs` |
| `Exception` | Runtime error | `exceptionType`, `message`, `retryAttempt` |
| `ApprovalTier` | Multi-tier approval step | `tierName`, `tierOrder`, `status` |

**Edge types relevant to execution UIs:**

| Edge type | Semantic | Typical source → target |
|---|---|---|
| `FOLLOWED_BY` | Sequential workflow flow | `NodeInstance` → `NodeInstance` |
| `EVALUATED_EDGE` | Conditional branch from `FinOSPolicy` | `FinOSPolicyEvent` → `NodeInstance` |
| `APPROVED_BY` | HITL approval path taken | `FinOSHITLEvent` → `NodeInstance` |
| `REJECTED_BY` | HITL rejection path taken | `FinOSHITLEvent` → `NodeInstance` |
| `WAITED_FOR` | Node waited for async callback | `NodeInstance` → `FinOSHITLEvent` |
| `RESUMED_BY` | HITL node resumed after decision | `FinOSHITLEvent` → `NodeInstance` |
| `ESCALATED_TO` | Escalation path to higher tier | `ApprovalTier` → `ApprovalTier` |
| `EXCEPTION_IN` | An exception occurred within a node | `Exception` → `NodeInstance` |

#### Rendering the Execution DAG

```bash
# Full DAG — use for graph rendering (adjacency list)
curl ".../workflow-executions/{instanceId}/graph?format=adjacency" \
  -H "Authorization: Bearer $TOKEN"

# Edgelist — use for force-directed graph layouts
curl ".../workflow-executions/{instanceId}/graph?format=edgelist" \
  -H "Authorization: Bearer $TOKEN"
```

**Response structure (adjacency format):**

```jsonc
{
  "instanceId": "3fa85f64-...",
  "executionStatus": "PAUSED",
  "vertices": {
    "v-001": {
      "vertexId": "v-001",
      "vertexType": "NodeInstance",
      "label": "InvoiceClassifierAgent",
      "status": "COMPLETED",
      "startTime": "2026-06-01T09:00:08Z",
      "endTime": "2026-06-01T09:00:10Z",
      "outgoingEdges": [
        { "edgeId": "e-001", "edgeType": "FOLLOWED_BY", "targetVertexId": "v-002" }
      ]
    },
    "v-005": {
      "vertexId": "v-005",
      "vertexType": "FinOSHITLEvent",
      "label": "ManagerReview",
      "status": "PENDING",
      "callbackUrl": "https://workflows.example.com/v1/callbacks/cb-mgr-001",
      "assignedTo": "FinanceManager",
      "dueDate": "2026-06-03T09:00:00Z",
      "outgoingEdges": []
    }
  }
}
```

#### Real-Time Updates via Event Subscription

For live execution monitoring, subscribe to events scoped to a specific execution instance.
Events are delivered to your `callbackUrl` as each node completes:

```bash
curl -X POST https://api.example.com/event-service/v1/subscriptions/by-execution \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "callbackUrl": "https://your-dashboard.example.com/execution-updates",
    "filter": {
      "workflowId":          "wf-invoice-approval-v1",
      "workflowExecutionId": "inst-abc123"
    },
    "deliveryConfig": { "secret": "dashboard-hmac-secret", "maxRetries": 3 }
  }'
```

> This subscription automatically expires when the execution reaches a terminal status
> (`COMPLETED`, `FAILED`, `CANCELLED`).

#### Listing All HITL-Paused Executions (Exception Queue)

```bash
# Find all currently paused executions — the HITL exception queue
curl "https://api.example.com/finos/v1/workflow-executions
  ?executionStatus=PAUSED
  &sort=executionStartTime&sortOrder=desc
  &pageSize=50" \
  -H "Authorization: Bearer $TOKEN"
```

> See [`1-finosworkflow-execution-history.md`](./1-finosworkflow-execution-history.md) for the
> complete vertex and edge type reference, the append-only immutability guarantees, and provenance
> traversal operations.

---

## 10. Integration Pattern 5 — Workflow DAG Authoring UI

Systems building workflow design surfaces — graphical DAG editors, low-code automation
builders, or configuration management tools — use the `FinOSWorkflow API` to create and manage
workflow definitions. The API exposes full CRUD on the DAG structure (nodes and edges),
lifecycle management (DRAFT → ACTIVE → DEPRECATED), DAG integrity validation, and export.

All FinOSWorkflow DAG definitions use nine typed node categories:

| Node type | Runtime behaviour | Use for |
|---|---|---|
| `FinOSStart` | Marks the DAG entry point; carries no node-specific fields | The single entry into every workflow; may branch on inputs before any work runs |
| `FinOSAgent` | Invokes a registered FinOS agent via the A2A protocol | AI reasoning, classification, extraction, matching, review generation |
| `FinOSAPI` | Makes an external REST API call | Data retrieval, state mutations, ERP integration, system-of-record writes |
| `FinOSAsyncAPI` | Kicks off a long-running partner job, then suspends until a callback or internal event reports completion | Reconciliation runs, batch jobs, and any partner work measured in minutes to days |
| `FinOSHITL` | Suspends workflow; waits for human callback decision | Human review, approval, rejection, override |
| `FinOSPolicy` | Evaluates a Policy Definition and returns an `outcome` | Governance gates, spend limits, SOD enforcement, access control |
| `FinOSWorkflow` | Invokes another registered workflow as a durable child execution | Multi-step sub-processes that may themselves pause on HITL gates |
| `FinOSFanOut` | Spawns one child workflow execution per item of a collection | Per-account, per-entity, or per-item processing with independent audit trails |
| `FinOSTerminal` | Ends the workflow instance with typed exit semantics | Declaring every distinct outcome — `COMPLETED` for valid business results, `FAILED` for unrecoverable ones |

Every DAG must contain **exactly one** `FinOSStart` node (with zero incoming edges) and
**at least one** `FinOSTerminal` node (with no outgoing edges).

#### Workflow Lifecycle States

```
DRAFT → ACTIVE → DEPRECATED (→ ARCHIVED)
  │                   │
  └── DELETE          └── cannot be deleted; use ARCHIVED
```

Only `DRAFT` and `ARCHIVED` workflows can be hard-deleted. `ACTIVE` workflows can be
`DEPRECATED` (prevents new executions; does not affect running ones).

#### Create a Workflow Definition

```bash
curl -X POST https://api.example.com/finos/v1/workflows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "workflowId":      "wf-expense-review-v1",
    "workflowName":    "Expense Report Review",
    "workflowVersion": "1.0.0",
    "description":     "AI-assisted expense classification and approval workflow.",
    "status":          "DRAFT"
  }'
# → 201 Created
```

#### Add Nodes

```bash
# FinOSAgent node — AI classification step
curl -X POST ".../workflows/wf-expense-review-v1/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "nodeId":       "expense-classifier",
    "nodeType":     "FinOSAgent",
    "nodeName":     "ExpenseClassifierAgent",
    "description":  "Classifies expense line items by category and compliance status.",
    "agentId":      "7f3d8c1a-0000-0000-0000-000000000001",
    "inputSchema":  { "expenseReport": { "type": "object" } },
    "outputSchema": { "classification": { "type": "string" }, "complianceFlag": { "type": "boolean" } }
  }'

# FinOSPolicy node — governance gate
curl -X POST ".../workflows/wf-expense-review-v1/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "nodeId":             "compliance-policy",
    "nodeType":           "FinOSPolicy",
    "nodeName":           "ExpenseComplianceCheck",
    "policyDefinitionId": "def-expense-compliance",
    "possibleOutcomes":   ["Success", "Escalate", "Failed"],
    "evaluationInputMapping": {
      "expenseAmount":  "${ .expense_amount }",
      "complianceFlag": "${ .compliance_flag }"
    }
  }'

# FinOSHITL node — human review step
curl -X POST ".../workflows/wf-expense-review-v1/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "nodeId":           "manager-review",
    "nodeType":         "FinOSHITL",
    "nodeName":         "ManagerExpenseReview",
    "assignedRole":     "ExpenseManager",
    "timeoutSeconds":   172800,
    "allowedDecisions": ["APPROVED", "REJECTED"],
    "taskInstructions": "Review flagged expense report and approve or reject."
  }'

# FinOSTerminal node — every DAG needs at least one
curl -X POST ".../workflows/wf-expense-review-v1/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "nodeId":     "terminal-approved",
    "nodeType":   "FinOSTerminal",
    "nodeName":   "ExpenseApproved",
    "exitStatus": "COMPLETED",
    "label":      "Expense report approved."
  }'
```

#### Add Directed Edges with Conditions

Edge conditions are [jq](https://jqlang.org) expressions enclosed in `${ }`. `null` means
unconditional — a catch-all, of which a source node may have at most one. Edges from the same
source are evaluated in `priority` order and the first match wins. Edges carry no type
discriminator; the condition alone determines the branch.

```bash
# Unconditional flow from classifier to policy
curl -X POST ".../workflows/wf-expense-review-v1/edges" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "edgeId":         "edge-classify-to-policy",
    "sourceNodeId":   "expense-classifier",
    "targetNodeId":   "compliance-policy",
    "condition":      null
  }'

# Conditional route from policy to HITL on Escalate
curl -X POST ".../workflows/wf-expense-review-v1/edges" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "edgeId":         "edge-escalate",
    "sourceNodeId":   "compliance-policy",
    "targetNodeId":   "manager-review",
    "condition":      "${ .policy_outcome == \"Escalate\" or .policy_outcome == \"AttentionRequired\" }",
    "priority":       1
  }'
```

**Edge condition syntax quick reference:**

```
"${ .policy_outcome == \"Escalate\" }"                                 # string equality
"${ .invoice_amount >= 50000 and .manager_decision == \"APPROVED\" }"  # compound boolean
"${ .compliance_flag == true }"                                       # boolean
"${ .items | length > 0 }"                                            # any valid jq
null   →  unconditional catch-all (at most one per source node)
```

#### Validate the DAG

Before activation, validate structural and FinOS compatibility rules:

```bash
curl -X POST ".../workflows/wf-expense-review-v1/validate" \
  -H "Authorization: Bearer $TOKEN"
# → 200 OK  { "valid": true, "checks": [...] }
# → 422 Unprocessable Entity  { "valid": false, "errors": [...] }
```

**DAG integrity rules enforced at validation:**

| Rule | Constraint |
|---|---|
| Acyclicity | No directed cycles |
| Entry point | **Exactly one** `FinOSStart` node, with zero incoming edges |
| Connectivity | All nodes reachable from the `FinOSStart` node |
| Terminal nodes | At least one `FinOSTerminal` node; terminals have no outgoing edges |
| Vertex existence | Every edge `sourceNodeId` / `targetNodeId` references a node defined in the `nodes` list — implicit terminals are not permitted |
| Condition syntax | Edge conditions parse as valid jq expressions |
| Default edges | A source node may have at most one unconditional (`condition: null`) catch-all edge |
| Edge coverage | Every outcome in a `FinOSPolicy` node's `possibleOutcomes` has a matching outgoing edge condition |
| Agent existence | All `agentId` references resolve to registered, active agents in the registry |
| Policy existence | All `policyDefinitionId` references resolve to active Policy Definitions in the Policy Service |

#### Retrieve the DAG for Rendering

```bash
# Retrieve full DAG in adjacency format for graph rendering
curl ".../workflows/wf-expense-review-v1/graph?format=adjacency" \
  -H "Authorization: Bearer $TOKEN"
```

#### Activate and Run

```bash
# Transition from DRAFT to ACTIVE
curl -X PATCH ".../workflows/wf-expense-review-v1/status" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{ "status": "ACTIVE" }'

# Trigger an execution. Config-sourced inputs are loaded from tenant
# configuration and overlaid with run_input; required inputs missing from
# both sources cause the trigger to be rejected.
curl -X POST ".../workflows/wf-expense-review-v1/runs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "tenantId": "tenant-acme-prod", "run_input": { "expense_id": "exp-2026-0042" } }'
# → 202 Accepted  { "instanceId": "5281a21d-...", "status": "RUNNING" }
```

#### Export as FinOS Workflows YAML

```bash
# Export the DAG as an executable FinOS Workflows YAML
curl ".../workflows/wf-expense-review-v1/export" \
  -H "Authorization: Bearer $TOKEN" \
  > wf-expense-review-v1.yaml
```

> See [`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) for the canonical workflow YAML
> format, workflow inputs, per-node authentication, the full node type reference, jq edge
> condition syntax, HITL and async resume patterns, and DAG integrity rules.
> Its §12 carries a complete, runnable Invoice Approval definition demonstrating six of the nine
> node types.

---

## 11. API and Schema Quick Reference

| API / Schema | Base URL | File | Guide |
|---|---|---|---|
| Agent Registry | `https://api.example.com/finos/v1` | `finos-agent-registry-api.yaml` | `3-finos-agent-registry-guide.md` |
| Event Service | `https://api.example.com/event-service/v1` | `finos-event-service-api.yaml` | `5-finos-event-service-guide.md` |
| Policy Service | `https://api.example.com/finos-policy-service/v1` | `finos-policy-api.yaml` | `4-finos-policy-guide.md` |
| Execution History | `https://api.example.com/finos/v1` | `finos-workflow-execution-history-openapi.yaml` | `1-finosworkflow-execution-history.md` |
| Workflow API | `https://api.example.com/finos/v1` | `finos-workflow-api.yaml` | `0-finos-workflow-guide.md` |
| FinOSAgentCard Schema | _(embedded in agent runtime)_ | `finos-agent-card-schema.yaml` | `2-finos-agent-card.md` |

**Generate a client from any spec:**

```bash
# TypeScript client for the Agent Registry
npx openapi-generator-cli generate \
  -i finos-agent-registry-api.yaml \
  -g typescript-fetch \
  -o ./generated/agent-registry-client

# Python client for the Execution History API
npx openapi-generator-cli generate \
  -i finos-workflow-execution-history-openapi.yaml \
  -g python \
  -o ./generated/execution-history-client

# Validate all specs
npx @redocly/cli lint finos-agent-registry-api.yaml
npx @redocly/cli lint finos-event-service-api.yaml
npx @redocly/cli lint finos-policy-api.yaml
npx @redocly/cli lint finos-workflow-execution-history-openapi.yaml
npx @redocly/cli lint finos-workflow-api.yaml
```

---

## 12. Key FinOS Concepts

### Workflow Node Types

| Node type | A2A / Protocol | Constraint |
|---|---|---|
| `FinOSStart` | None — routing only | **Exactly one** per DAG, with zero incoming edges |
| `FinOSAgent` | A2A protocol → FinOS Agent Runtime | **Must not** mutate systems of record |
| `FinOSAPI` | REST HTTP call | **Owns all state mutations** |
| `FinOSAsyncAPI` | REST kickoff, then suspends for an authenticated callback or internal event | The kickoff response is not the result; `timeoutSeconds` is a real business deadline |
| `FinOSHITL` | Suspends; awaits callback | Resumable only via `callbackUrl` POST |
| `FinOSPolicy` | Evaluation API call | Drives edge routing via `policy_outcome` |
| `FinOSWorkflow` | Durable child execution via the Workflow Registry | Waits indefinitely for child HITL gates without timing out the parent |
| `FinOSFanOut` | One child workflow execution per collection item | Each item gets its own state, HITL gates, retries, and audit trail |
| `FinOSTerminal` | None — exit semantics only | **At least one** per DAG; no outgoing edges; `exitStatus` is `COMPLETED` or `FAILED` |

### Agent Trust Tiers

| Tier | Label | Data Access | Write Permissions | HITL Required |
|---|---|---|---|---|
| `T0` | Platform Native | All data including RESTRICTED | Full write access | No |
| `T1` | Certified Partner | CONFIDENTIAL and below | Write with policy gate | For RESTRICTED data |
| `T2` | Registered External | INTERNAL and below | Own namespace only | For CONFIDENTIAL+ data |
| `T3` | Provisional | PUBLIC only | Read-only | For all actions |

### FinOSEvent Subtypes

| `finOSEventType` | `type` prefix | Emitted by |
|---|---|---|
| `FinOSAgentEvent` | `{event-namespace}.finos.agent.*` | `FinOSAgent` nodes, agent runtime |
| `FinOSAPIEvent` | `{event-namespace}.finos.api.*` | `FinOSAPI` nodes; external system ingestion |
| `FinOSHITLEvent` | `{event-namespace}.finos.hitl.*` | `FinOSHITL` nodes |
| `FinOSPolicyEvent` | `{event-namespace}.finos.policy.*` | `FinOSPolicy` nodes; CNAPP/GRC ingestion |

### Context Graph Modes

Context graphs supply the data that FinOS AI agents reason over. Agents declare supported
modes in `finOSSupportedContextModes`; the calling workflow populates
`Message.metadata.finos.contextGraphs[]` at invocation:

| Mode | Description |
|---|---|
| `RAG_QUERY` | Issue a semantic similarity query to a vector store; results injected into context window |
| `RAG_TENSOR` | Load a pre-computed embedding tensor from a storage URI |
| `FINOSWORKFLOW_GRAPH` | Serialize a FinOS workflow DAG subgraph as structured context |
| `STRUCTURED_CHUNKS` | Inject structured data chunks directly into the context window |

### Execution History Vertex and Edge Types

**Selected vertex types** (15 total):

| Type | What it records |
|---|---|
| `NodeInstance` | A single workflow node execution |
| `FinOSAgentEvent` | LLM agent action — model, tools, HITL pause, token usage |
| `FinOSHITLEvent` | Human review task and decision |
| `FinOSPolicyEvent` | Policy evaluation with outcome and applied rules |
| `FinOSAPIEvent` | External API call and response |
| `ApprovalTier` | One tier of a multi-tier escalation |
| `Exception` | Runtime error or failure |

**Selected edge types** (18 total):

| Type | Semantic |
|---|---|
| `FOLLOWED_BY` | Sequential workflow flow |
| `EVALUATED_EDGE` | Conditional branch from a `FinOSPolicy` node |
| `APPROVED_BY` | HITL or agent approval path taken |
| `REJECTED_BY` | HITL or agent rejection path taken |
| `WAITED_FOR` | Node waited for an async event or callback |
| `RESUMED_BY` | HITL node resumed after callback |
| `ESCALATED_TO` | Escalation to a higher approval tier |
| `EXCEPTION_IN` | Exception occurred within a node |

---

## 13. Authentication

All FinOS Platform APIs (except `/health` endpoints) require a bearer token issued by the
the platform identity provider (IdP):

```
Authorization: Bearer <JWT>
```

Tokens are issued via the **OAuth 2.0 client credentials grant** flow:

```bash
curl -X POST https://identity.example.com/oauth2/token \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "scope=finos:agent-registry:read finos:events:write finos:execution-history:read"
```

**Scopes referenced in this guide:**

| Scope | Access granted |
|---|---|
| `finos:agent-registry:read` | List and retrieve agent registrations |
| `finos:agent-registry:write` | Register and update agents |
| `finos:agent-registry:trust:write` | Assign trust tiers T0–T3 |
| `finos:events:read` | Query published events |
| `finos:events:write` | Publish events to the event bus |
| `finos:events:subscribe` | Create and manage event subscriptions |
| `finos:policy:read` | List and retrieve policy definitions |
| `finos:policy:write` | Create, update, and activate policies |
| `finos:execution-history:read` | Query workflow execution histories and DAGs |
| `finos:workflow:read` | Retrieve workflow DAG definitions |
| `finos:workflow:write` | Create and modify workflow DAG definitions |

> Partner and external agent registrations require additional scopes bound to the partner
> onboarding client identity. Contact Blackline for partner credential provisioning.

---

## 14. Contributing

Blackline welcomes contributions from the community. By contributing you agree that your
submissions are licensed under the Apache 2.0 License.

### How to Contribute

1. **Fork** this repository and create a branch from `main`
2. **Make your changes** — bug fixes, schema corrections, documentation improvements, or new examples
3. **Validate** your changes:
   ```bash
   npx @redocly/cli lint <changed-spec>.yaml
   ```
4. **Open a Pull Request** with a clear description of the change and its motivation

### Contribution Guidelines

- **Spec changes** must be backward-compatible where possible; breaking changes require a version bump and a migration note in the PR description
- **Examples** must be valid against the published schemas and must not contain real credentials, personal data, or tenant identifiers
- **Documentation** must stay consumer-first: explain what an implementing or integrating system must do, not how Blackline's internal platform is built
- **Keep examples vendor-neutral** — hostnames use RFC 2606 documentation domains (`example.com` and subdomains), organisations are `Example Corp`, and wire identifiers use the namespace placeholders of [§3.4](#34-namespace-placeholders). `FinOS` is the only project identity that appears in normative text; example LLM model names (`claude-sonnet-4-6`, `gpt-4o`) are acceptable
- **Commit messages** should follow the [Conventional Commits](https://www.conventionalcommits.org/) format:
  ```
  feat(event-service): add CNAPP event ingestion examples
  fix(agent-registry): correct finOSAgentType enum values
  docs(hitl): clarify callback payload and resume flow
  ```

### Reporting Issues

Open a [GitHub Issue](../../issues) for errors in specifications, missing documentation, or
compatibility problems.

### Security Disclosures

Do **not** open public issues for security vulnerabilities. Contact **security@blackline.com**.

---

## 15. License

```
Copyright 2026 BlackLine, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

### Attribution

These specifications were authored by **BlackLine, Inc.** and published openly for two
reasons: to enable the partner and customer ecosystem to build FinOS-compatible agents and
integrations, and to offer the wider industry a governance model for agentic AI in the office of
the CFO that does not have to be reinvented per vendor. Implementing them creates no dependency
on Blackline software.

BlackLine, Inc. retains copyright ownership. The Apache 2.0 license grants broad rights to
use, reproduce, modify, and distribute these specifications and any derivative works, subject to
the license terms.

### Third-Party Standards

| Standard | License | Role in FinOS |
|---|---|---|
| [FinOS Agent-to-Agent (A2A) Protocol](https://a2a-protocol.org/) | Apache 2.0 | Agent invocation and discovery protocol |
| [CloudEvents v1.0](https://cloudevents.io/) | Apache 2.0 | Execution history event format; event bus schema |
| [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) | Apache 2.0 | API specification format for all FinOS APIs |

---

<p align="center">
  <strong>BlackLine, Inc.</strong> · <a href="https://www.blackline.com">blackline.com</a><br>
  <sub>BlackLine (Nasdaq: BL) provides the trust infrastructure where finance drives the agentic era with intelligence, integrity, and trust. The BlackLine Agentic Financial Operations Platform&trade; is where the Office of the CFO puts AI to work, governs it at every step, and guarantees its integrity across finance.</sub>
</p>
