# FinOS Policy Service — Developer Guide

**Service:** `finos-policy-service`  
**API Version:** 1.0.0  
**Spec:** [`finos-policy-api.yaml`](./finos-policy-api.yaml)  
**Base URL:** `https://api.example.com/finos-policy-service/v1`  
**Related specs:** [`finos-workflow-api.yaml`](./finos-workflow-api.yaml) · [`finos-workflow-execution-history-openapi.yaml`](./finos-workflow-execution-history-openapi.yaml)

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Two-Layer Model](#2-the-two-layer-model)
   - 2.1 [Policy Definition — Reusable Shape](#21-policy-definition--reusable-shape)
   - 2.2 [Policy Configuration — Tenant-Specific Rule](#22-policy-configuration--tenant-specific-rule)
   - 2.3 [Why two layers?](#23-why-two-layers)
3. [Schema Architecture](#3-schema-architecture)
   - 3.1 [PolicyDefinition](#31-policydefinition)
   - 3.2 [Policy Configuration (condition + enforcement)](#32-policy-configuration-condition--enforcement)
   - 3.3 [Variable Data Types, Coercion & Operators](#33-variable-data-types-coercion--operators)
   - 3.4 [Versioning](#34-versioning)
   - 3.5 [Evaluation Record](#35-evaluation-record)
   - 3.6 [Platform Invariants](#36-platform-invariants)
4. [Authoring & Evaluation Lifecycle](#4-authoring--evaluation-lifecycle)
5. [Evaluation Outcomes & Workflow Routing](#5-evaluation-outcomes--workflow-routing)
6. [API Reference](#6-api-reference)
7. [Workflow Integration](#7-workflow-integration)
8. [Platform Invariants](#8-platform-invariants)
9. [Error Reference](#9-error-reference)
10. [Appendix — Schema Field Reference](#10-appendix--schema-field-reference)

---

## 1. Overview

The **FinOS Policy Service** is the governance authority for policy evaluation on the
BlackLine FinOS Orchestration Platform. It gates agentic workflows at runtime and manages
the full authoring lifecycle behind those gates.

It uses a **two-layer model**:

- A **Policy Definition** is a **tenant-agnostic, reusable shape** — a declared variable
  catalog and a closed vocabulary of possible outcomes, versioned. A workflow blueprint's
  Policy Node references a Definition. Definitions carry **no** condition, enforcement, or
  tenant.
- A **Policy Configuration** is the **tenant-specific configuration** of a Definition at one
  workflow node — the actual `condition` and `enforcement`, pinned to a specific Definition
  version at save time. One Configuration exists per `(tenant, workflowConfigId, nodeId)`.

At runtime the service resolves the calling tenant's **Policy Configuration** for a given
`(workflowConfigId, nodeId)` placement — or loads it directly by `policyConfigurationId` — and
evaluates its condition and enforcement.

**Core responsibilities:**

| Responsibility | Detail |
|---|---|
| **Definitions** | Author, version, and read tenant-agnostic Policy Definitions (variables + outcome vocabulary) |
| **Configurations** | Author, version, and read tenant-specific Policy Configurations (condition + enforcement) bound to a workflow node |
| **Evaluation** | Resolve and evaluate the tenant's Configuration for a `(workflowConfigId, nodeId)` placement against caller-supplied inputs |
| **Evaluation audit** | Record every evaluation event with inputs, matched condition, and outcome |
| **Platform invariants** | Expose read-only, system-seeded, cross-cutting governance invariants |

---

## 2. The Two-Layer Model

### 2.1 Policy Definition — Reusable Shape

A **Policy Definition** publishes the *shape* of a governance rule that many tenants can
configure differently against the same workflow node. It is tenant-agnostic and owns only:

```
PolicyDefinition
├── definitionId         — unique identifier
├── name                 — human-readable name
├── description          — purpose
├── variables            — declared variable catalog (name, type, source) that a
│                          Policy's condition may reference
├── possibleOutcomes     — CLOSED allow-list of outcome labels a Policy may produce
├── currentVersion       — semantic version (e.g. "1.0.0")
├── status               — DRAFT | ACTIVE | INACTIVE | DEPRECATED | DISABLED
└── audit fields         — createdBy, createdAt, lastModifiedBy, lastModifiedAt
```

- `possibleOutcomes` is a **closed allow-list**. Every outcome a tenant's Configuration
  references (in any tier or as a default) **must** be a member of the pinned Definition
  version's list; an out-of-list outcome is rejected at save time.
- Changing `variables` or `possibleOutcomes` creates a new Definition version. A Configuration
  always pins the exact Definition version it was authored against, so later Definition edits
  never retroactively change an existing Configuration.

**Usage pills.** A Definition reports how it is being used:

| Field | Meaning |
|---|---|
| `availableToBlueprints` | Count of distinct workflow definitions whose Policy Node references this Definition |
| `appliedToWorkflows` | Count of distinct workflows for which a tenant has authored a Policy Configuration against this Definition |

### 2.2 Policy Configuration — Tenant-Specific Rule

A **Policy Configuration** is the concrete, tenant-owned rule at a single workflow node
placement. It carries the actual `condition` and `enforcement`, and pins to a Definition version
at save time.

```
PolicyConfiguration
├── configurationId          — server-minted identifier (cfg_<uuid>)
├── tenantId                 — owning tenant
├── blueprintId              — the workflow blueprint this placement belongs to
├── workflowConfigId         — the tenant workflow configuration holding the placement
├── nodeId                   — the evaluation-site node inside that workflow
├── policyDefinitionId       — the Definition this Configuration configures
├── policyDefinitionVersion  — the Definition version pinned at save time (immutable)
├── condition                — top-level PolicyCondition tree (see §3.2)
├── enforcement              — SIMPLE or TIERED outcome mapping (see §3.2)
├── currentVersion           — this Configuration's own version (e.g. "2.0.0")
└── audit fields             — createdBy, createdAt, lastModifiedBy, lastModifiedAt
```

Key properties:

- **1:1 with `(tenantId, workflowConfigId, nodeId)`.** A Configuration is authored for one node
  placement, not as a separately-bound reusable object. There is no `scope` field — it is already
  concretely scoped to one node.
- **Always active.** A Configuration has no status lifecycle: it is not disabled or
  soft-deleted. Changes happen exclusively through version bumps.
- **Pins a Definition version permanently.** `policyDefinitionVersion` is fixed at save time;
  the Configuration's variables and outcome vocabulary are validated against that frozen
  Definition version, not the live Definition.

### 2.3 Why two layers?

- A platform team publishes a Definition once (the variable catalog + the outcomes a node can
  produce). A workflow blueprint's Policy Node references that Definition.
- Each tenant then authors its own Configuration — its own thresholds, tiers, and enforcement —
  against the same node, without redefining the shape and without being able to invent outcomes
  the Definition doesn't allow.

---

## 3. Schema Architecture

### 3.1 PolicyDefinition

```yaml
# A Policy Definition — tenant-agnostic, versioned
definitionId: "def-invoice-approval"
name: "Invoice Approval Gate"
description: "Shape for invoice-approval policies: amount + risk inputs, standard outcomes."
variables:
  - name: "invoiceAmount"
    type: "NUMBER"
    source: "EVALUATION_INPUT"
    required: true
  - name: "approvedSpendLimit"
    type: "NUMBER"
    source: "EVALUATION_INPUT"
    required: true
  - name: "vendorRiskScore"
    type: "NUMBER"
    source: "EVALUATION_INPUT"
    required: false
possibleOutcomes: ["Success", "AttentionRequired", "Escalate", "Failed"]
currentVersion: "1.0.0"
status: "ACTIVE"
```

A Definition may **optionally** carry a `condition` (a `PolicyCondition` tree) as well:

- **With a `condition`** — `variables` is *derived* from the condition's leaves and need not be
  supplied separately. A Definition-level condition can reference `CONFIGURATION_INPUT` operands
  (per-tenant values filled in when a Policy is authored against it).
- **Without a `condition`** (the example above) — the Definition declares `variables` directly,
  and each authored Configuration supplies its own condition over that variable set.

Either way the Definition owns the reusable *shape* (variables + `possibleOutcomes`); the
concrete condition that runs at a node is fixed when a tenant authors the Configuration.

**Variable sources.** A variable's `source` determines where its value comes from — and a
variable is only ever resolved from its declared source:

| Source | Resolved from (evaluate map) | Where the value comes from |
|---|---|---|
| `EVALUATION_INPUT` | `evaluationInputs` | Provided in the evaluate call at runtime |
| `CONFIGURATION_INPUT` | — (authoring time) | A value the **tenant fills in when authoring the Configuration** (e.g. a per-tenant threshold). It is baked into the saved condition as a literal, so it carries **no runtime value** — it never appears in an evaluate map, and a stray `CONFIGURATION_INPUT` operand at evaluate fails closed. |

A Configuration's condition may reference **only** variables declared in the pinned Definition
version, matched on `name` + `type` + `source`.

### 3.2 Policy Configuration (condition + enforcement)

A Configuration carries a **top-level `condition`** (a single, pure `PolicyCondition` tree) and an
`enforcement` that maps the condition's result to an outcome. Enforcement is a discriminated
union on `kind`. There are two kinds:

**SIMPLE** — one condition, met/not-met. The `condition` is a leaf (or nested boolean tree);
`enforcement` only carries the outcome mapping:

```yaml
condition:
  expression: "invoiceAmount > approvedSpendLimit"
  variables:
    - { name: "invoiceAmount",       type: "NUMBER", source: "EVALUATION_INPUT" }
    - { name: "approvedSpendLimit",  type: "NUMBER", source: "EVALUATION_INPUT" }
enforcement:
  kind: "SIMPLE"
  outcome: "Escalate"          # produced when the condition is met
  defaultOutcome: "Success"    # produced when the condition is not met
```

**TIERED** — the top-level `condition` is a combinator tree whose immediate `subConditions`
are the tiers, each carrying a unique non-empty `name`. `enforcement.outcomes` is a
**name-keyed map** from tier name to outcome; first matching tier wins, else the default:

```yaml
condition:
  combinator: "OR"
  subConditions:
    - name: "high"
      expression: "invoiceAmount > 50000.0"
      variables: [ { name: "invoiceAmount", type: "NUMBER", source: "EVALUATION_INPUT" } ]
    - name: "medium"
      expression: "invoiceAmount > 10000.0"
      variables: [ { name: "invoiceAmount", type: "NUMBER", source: "EVALUATION_INPUT" } ]
enforcement:
  kind: "TIERED"
  outcomes:
    high: "Failed"
    medium: "Escalate"
  defaultOutcome: "Success"    # no tier matched
```

- `outcome` (SIMPLE) / each value in `outcomes` (TIERED) is the label produced when the
  condition/tier fires; `defaultOutcome` is produced when nothing fires.
- Every outcome referenced by `enforcement` must be a member of the pinned Definition
  version's `possibleOutcomes` (validated at save time).
- **TIERED** evaluates tiers (the top-level `subConditions`) in order and returns the outcome
  mapped to the **first** tier whose condition is `true`; lower-priority tiers are never
  reported even if they would also match. A tier name in `condition.subConditions` and a key in
  `enforcement.outcomes` must correspond one-to-one (validated at save time).
- The `condition` is **not** nested inside `enforcement` — it is a sibling, top-level field.
  This lets the same condition tree carry per-clause audit attribution independent of the
  outcome mapping.

These shapes map directly onto `SimpleEnforcement` and `TieredEnforcement` in
[`finos-policy-api.yaml`](./finos-policy-api.yaml), discriminated on `kind`.


### 3.3 Variable Data Types, Coercion & Operators

Every variable in a condition declares a `type` and a `source`. At evaluate time the raw value
resolved from the matching source map is **coerced** to the declared type before the expression
runs. Coercion is **strict and fail-closed**: a value that doesn't cleanly match the declared
type raises an error and the whole evaluation returns a deterministic `Failed / BLOCK` envelope
(it never silently truncates, widens with loss, or guesses).

#### Allowed data types

The FinOS type set is deliberately coarser than a raw language's native types — six types cover
the governance surface:

| `type`   | Accepts (native)                  | Accepts (string form)                    | Coerced to | Notes |
|----------|-----------------------------------|------------------------------------------|------------|-------|
| `NUMBER` | `int`, `float` (i.e. double/long) | `"10"`, `"10.5"`, `"-3"`, `"1.2e3"`      | float64    | **One numeric type covers int, double, float, long.** An integer is widened to float64; an integer beyond ±2⁵³ is **rejected** (precision loss) rather than rounded. `"1.0"` is fine; a non-numeric string fails. |
| `STRING` | `str`                             | any string (passed through)              | string     | Case sensitivity is configurable; by default comparisons are case-insensitive (values lower-cased at coercion). |
| `BOOLEAN`| `bool`                            | `"true"` / `"false"` (case-insensitive)  | bool       | A real `bool` for a non-BOOLEAN variable is rejected (no truthy coercion of `1`/`0`). |
| `DATE`   | ISO-8601 date-time                | `"2026-01-31"`, `"2026-01-31T12:00:00Z"` | timestamp  | `Z` is accepted and normalised to `+00:00`. |
| `OBJECT` | `dict`                            | JSON object string                       | map        | Must be a real object; a scalar/list fails. |
| `ARRAY`  | `list`                            | JSON array string                        | list       | Must be a real list; a scalar/object fails. |

**How a value is considered — worked examples for `NUMBER`:**

- `5000` (int) → widened to `5000.0` ✓
- `5000.50` (double) → `5000.5` ✓
- `"5000"` (string int) → `5000.0` ✓
- `"5000.50"` (string double) → `5000.5` ✓
- `9007199254740993` (int > 2⁵³) → **rejected** (cannot represent exactly as float64)
- `"abc"` / `true` for a NUMBER → **rejected**

So a `NUMBER`-typed variable transparently accepts a value that is an **int, float, double, or
long** (or their string forms) and normalises them all to one numeric representation before
comparison — you never declare `INT` vs `DOUBLE` separately.

**Source-aware coercion.** Coercion is per-variable and keyed on `source`: an
`EVALUATION_INPUT` variable is looked up in the evaluate call's inputs only. A required variable
missing from its source fails closed.

#### Allowed operators & functions

Conditions are written in **CEL** (Common Expression Language). All operators satisfy the strict
typing above.

- **Comparison**: `==`, `!=`, `<`, `<=`, `>`, `>=`
- **Logical**: `&&`, `||`, `!`
- **Arithmetic**: `+`, `-`, `*`, `/`, `%`
- **Membership / collections**: `in`, list/map indexing, `size()`
- **Grouping**: parentheses

Per-type operator notes:

- **NUMBER** — all comparison and arithmetic operators. Threshold/inequality comparisons
  (`amount > 10000.00`) are always safe. **Equality (`==`/`!=`) on a *computed sum* of monetary
  NUMBERs is fragile** because NUMBER is an IEEE-754 double (`0.1 + 0.2 == 0.3` is `false`). For
  "sum equals total" money checks, scale to integer cents with the `math` extension:
  `math.round(a*100.0) + math.round(b*100.0) == math.round(c*100.0)`.
- **STRING** — `==`, `!=`, plus functions: `contains`, `startsWith`, `endsWith`, `matches`
  (regex). Case-insensitive comparison uses the `strings` extension:
  `status.lowerAscii() == 'open'` (or `.upperAscii()`).
- **BOOLEAN** — `==`, `!=`, logical operators.
- **DATE** — equality (`==`/`!=`), ordering (`<`/`>`/`<=`/`>=`), and string functions work on a
  DATE-declared variable (already a timestamp). Wrap a *literal* in `timestamp(...)` when
  comparing against one.
- **OBJECT** — field access (`metadata.owner`) and `in` for key membership. Accessing a **missing
  key raises** (`NOT_FOUND`) and fails the evaluation closed — guard optional fields with `in`
  (e.g. `'owner' in metadata && metadata.owner == 'alice'`).
- **ARRAY** — indexing, `in`, `size()`.

### 3.4 Versioning

Both Definitions and Configurations are versioned; version records are append-only and immutable.

- **Definition versions** — a new version is created whenever `variables`, `condition`, or
  `possibleOutcomes` change. Each version is an immutable snapshot.
- **Configuration versions** — a new version is created whenever a re-`PUT` changes the payload.
  An unchanged payload is a no-op. Each version snapshots the `condition` and `enforcement` and
  records the Definition version pinned at that time.
- A Configuration pins **one Definition version** for its entire life; a Definition edit after a
  Configuration is authored never affects that Configuration.
- The evaluation record stores which Configuration version and which Definition version were
  active, enabling precise audit reconstruction.

### 3.5 Evaluation Record

Every evaluate call persists one record:

```
Evaluation
├── evaluationId              — server-minted UUID
├── tenantId
├── workflowId
├── workflowInstanceId        — the specific run
├── nodeId
├── policyConfigurationId     — the tenant Configuration that was evaluated
├── policyDefinitionId        — the Definition it configures
├── policyDefinitionVersion   — the pinned Definition version
├── policyVersion             — the Configuration version evaluated
├── triggered                 — did any condition/tier fire (outcome != defaultOutcome)?
├── action                    — enforcement action taken (BLOCK / ESCALATE / ... / none)
├── outcome                   — the produced outcome label
├── contextSnapshot           — snapshot of the evaluate call's input maps
├── reason                    — human-readable explanation
└── evaluatedAt               — timestamp
```

### 3.6 Platform Invariants

> **Status: Planned.** Platform invariants are not yet implemented. This section describes the
> intended design; the `/invariants` endpoint and cross-cutting enforcement are not available
> today.

Platform invariants are **system-seeded, tenant-independent, cross-cutting** governance rules.
They cannot be created, modified, or deleted via this API — they are read-only and describe
non-negotiable platform-level constraints, evaluated on every call regardless of which
Configuration is targeted.

Key examples:
- AI Agents (`FinOSAgent` nodes) **must not** mutate systems of record
- All `FinOSHITL` decisions **must** be attributed to a named reviewer
- Agent invocations **must** include a valid `callingContext`

---

## 4. Authoring & Evaluation Lifecycle

```
1. Publish a Policy Definition (tenant-agnostic shape)
   POST /policy-definitions                             → mint id, create
   PUT  /policy-definitions/{definitionId}              → idempotent upsert
                                                          (create-or-version-bump)

2. A workflow blueprint's Policy Node references the Definition by
   policyDefinitionId (registered by the workflow-authoring service)

3. A tenant authors a Policy Configuration for its (workflowConfigId, nodeId)
   PUT /policy-configurations/{nodeId}/{workflowConfigId}
     body: { blueprintId, policyDefinitionId, policyDefinitionVersion?,
             condition, enforcement, workflowConfigName?, changeDescription? }
   → validated against the pinned Definition version, returns a server-minted
     configurationId. Re-PUT is idempotent: create → version-bump; unchanged
     payload is a no-op.

4. At runtime the workflow evaluates the node
   POST /policies/{policyDefinitionId}/evaluate
     body: { workflowId, nodeId, workflowInstanceId, evaluationInputs,
             policyConfigurationId?, workflowConfigId?, ... }
   → the service loads the Configuration (by policyConfigurationId when
     supplied, else by the (workflowConfigId, nodeId) node tuple), enforces
     that its policyDefinitionId matches the path {policyDefinitionId}, and
     evaluates it. A missing Configuration or a mismatched Definition fails
     closed (outcome=Failed, action=BLOCK).

5. Update a Configuration → re-PUT the same path; unchanged payload is a
   no-op, changed payload bumps the minor version and writes a new snapshot.
```

**Definition statuses** follow the usual DRAFT → ACTIVE → INACTIVE / DEPRECATED / DISABLED
lifecycle. **Policy Configurations** have no status lifecycle — they are always active and evolve
only by versioning.

---

## 5. Evaluation Outcomes & Workflow Routing

The `outcome` returned by evaluate maps directly to the `policy_outcome` variable in the calling
workflow. Outcome labels are drawn from the Definition's `possibleOutcomes`.

### Outcome semantics (standard set)

| Outcome | `triggered` | Typical `action` | Workflow routing |
|---|---|---|---|
| `Success` | `false` | — (none) | Proceed to next node (auto-approve path) |
| `Escalate` | `true` | `ESCALATE` | Route to `FinOSHITL` node (human review) |
| `AttentionRequired` | `true` | `ESCALATE` | Route to HITL or notify; workflow may continue |
| `Failed` | `true` | `BLOCK` | Reject / terminal failure path |

`triggered` is `true` whenever the produced `outcome` differs from `defaultOutcome`. `action` is
derived from the outcome: `Failed → BLOCK`, any other triggered outcome → `ESCALATE`, otherwise
none.

### Outcome routing on `policy_outcome`

Routing is expressed as ordinary conditional edges out of the `FinOSPolicy` node, each testing
the `policy_outcome` variable. Edges are evaluated in `priority` order; the first whose
`condition` is truthy is traversed. Edges carry no type discriminator — the condition alone
determines the branch.

```yaml
edges:
  - edgeId: e-escalate
    sourceNodeId: confidence-gate
    targetNodeId: review-suggestions
    condition: '${ .policy_outcome == "Escalate" }'
    priority: 1
  - edgeId: e-success
    sourceNodeId: confidence-gate
    targetNodeId: terminal-complete
    condition: '${ .policy_outcome == "Success" }'
    priority: 2
  - edgeId: e-failed
    sourceNodeId: confidence-gate
    targetNodeId: terminal-rejected
    condition: '${ .policy_outcome == "Failed" }'
    priority: 3
```

The Definition's `possibleOutcomes` is what the workflow validation endpoint uses to verify that
every outcome a node can produce has a matching outgoing edge condition; registration is rejected
if any outcome is unrouted.

---

## 6. API Reference

**Base URL:** `https://api.example.com/finos-policy-service/v1`

**Authentication:** All endpoints (except `/health`) require a Bearer JWT. The tenant is always
taken from the authenticated principal — never from a request body or query string.

**Envelope:** Non-evaluate responses are wrapped as `{ "data": <entity>, "meta": {...} }`.

### Endpoint summary

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness / datastore check |
| POST | `/policy-definitions` | Create a Policy Definition with a server-minted id (`pol_<uuid>_v1`) |
| PUT | `/policy-definitions/{definitionId}` | **Idempotent upsert by caller-supplied id.** Creates the Definition under `{definitionId}` if absent, or updates it (minor version bump; unchanged payload is a no-op) if it exists. First-time authoring can `POST` to get a minted id, then `PUT {id}` on every subsequent save. |
| GET | `/policy-definitions` | List Definitions (with usage pills) |
| GET | `/policy-definitions/{definitionId}` | Get one Definition (with usage pills) |
| PATCH | `/policy-definitions/{definitionId}` | Update a Definition (version bump on variables/outcomes) |
| PATCH | `/policy-definitions/{definitionId}/status` | Definition lifecycle transition |
| GET | `/policy-definitions/{definitionId}/versions` | Definition version history |
| GET | `/policy-definitions/{definitionId}/policies` | Tenant Policies configured against this Definition |
| PUT | `/policy-configurations/{nodeId}/{workflowConfigId}` | Idempotent upsert of the tenant's **Policy Configuration** for one `(nodeId, workflowConfigId)` placement (create → version-bump). Returns a server-minted `configurationId` (`cfg_<uuid>`) the caller stores in the workflow run-config. |
| GET | `/policy-configurations/{nodeId}/{workflowConfigId}` | Get the Configuration at one node placement |
| GET | `/policy-configurations?nodeId=&workflowConfigId=&policyDefinitionId=&blueprintId=` | List/filter Configurations. |
| GET | `/policy-configurations/{configurationId}/versions` | Configuration version history |
| POST | `/policies/{policyDefinitionId}/evaluate` | Evaluate the tenant's Configuration for this `(workflowId, nodeId, workflowInstanceId)` run. Accepts an optional `policyConfigurationId` on the request body to load the Configuration directly by id (bypassing the `(workflowConfigId, nodeId)` node-tuple resolution); when omitted, resolution falls back to the tuple. The loaded Configuration's `policyDefinitionId` MUST match the `{policyDefinitionId}` in the path or the call fails closed (BLOCK). |
| GET | `/invariants` | List active platform invariants (read-only) — **planned, not yet implemented** |

---

### 6.1 Create a Policy Definition

```bash
curl -X POST https://api.example.com/finos-policy-service/v1/policy-definitions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Invoice Approval Gate",
    "description": "Shape for invoice-approval policies.",
    "variables": [
      { "name": "invoiceAmount",      "type": "NUMBER", "source": "EVALUATION_INPUT", "required": true },
      { "name": "approvedSpendLimit", "type": "NUMBER", "source": "EVALUATION_INPUT", "required": true },
      { "name": "vendorRiskScore",    "type": "NUMBER", "source": "EVALUATION_INPUT", "required": false }
    ],
    "possibleOutcomes": ["Success", "AttentionRequired", "Escalate", "Failed"]
  }'
# → 201 Created; starts in DRAFT with version 1.0.0
```

Activate it before Policies can be authored against it:

```bash
curl -X PATCH https://api.example.com/finos-policy-service/v1/policy-definitions/def-invoice-approval/status \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "status": "ACTIVE" }'
```

### 6.2 List / Get Definitions (usage pills)

```bash
curl "https://api.example.com/finos-policy-service/v1/policy-definitions?pageSize=25&pageStartIndex=0" \
  -H "Authorization: Bearer $TOKEN"

curl https://api.example.com/finos-policy-service/v1/policy-definitions/def-invoice-approval \
  -H "Authorization: Bearer $TOKEN"
```

The response includes `availableToBlueprints` and `appliedToWorkflows` counts (see §2.1).

### 6.3 Author a Policy Configuration for a node placement

`PUT` is the only write verb: it creates the Configuration on first call and version-bumps it on
every subsequent save. An unchanged payload is a no-op.

```bash
curl -X PUT \
  https://api.example.com/finos-policy-service/v1/policy-configurations/confidence-gate/wfc-invoice-acme \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "blueprintId": "wf-invoice-v1",
    "policyDefinitionId": "def-invoice-approval",
    "policyDefinitionVersion": 1,
    "condition": {
      "expression": "invoiceAmount > approvedSpendLimit",
      "variables": [
        { "name": "invoiceAmount",      "type": "NUMBER", "source": "EVALUATION_INPUT" },
        { "name": "approvedSpendLimit", "type": "NUMBER", "source": "EVALUATION_INPUT" }
      ]
    },
    "enforcement": {
      "kind": "SIMPLE",
      "outcome": "Escalate",
      "defaultOutcome": "Success"
    }
  }'
# → 201 Created at version 1.0.0, with a server-minted configurationId (cfg_<uuid>)
```

The Configuration is validated against Definition version 1: every referenced variable must be
declared there, and `outcome` / `defaultOutcome` must be in its `possibleOutcomes`. If
`policyDefinitionVersion` is omitted, the Definition's current version is pinned — permanently.

Store the returned `configurationId` in the workflow run configuration; passing it on evaluate
loads the Configuration directly and skips node-tuple resolution.

### 6.4 Read and update the Configuration at a node

```bash
# Get
curl https://api.example.com/finos-policy-service/v1/policy-configurations/confidence-gate/wfc-invoice-acme \
  -H "Authorization: Bearer $TOKEN"

# Update → re-PUT the same path. Changed payload bumps the minor version;
# unchanged payload is a no-op. The pinned Definition version does not move.
curl -X PUT \
  https://api.example.com/finos-policy-service/v1/policy-configurations/confidence-gate/wfc-invoice-acme \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "blueprintId": "wf-invoice-v1",
    "policyDefinitionId": "def-invoice-approval",
    "policyDefinitionVersion": 1,
    "condition": {
      "combinator": "OR",
      "subConditions": [
        { "name": "high",   "expression": "invoiceAmount > 50000.0", "variables": [ { "name": "invoiceAmount", "type": "NUMBER", "source": "EVALUATION_INPUT" } ] },
        { "name": "medium", "expression": "invoiceAmount > 10000.0", "variables": [ { "name": "invoiceAmount", "type": "NUMBER", "source": "EVALUATION_INPUT" } ] }
      ]
    },
    "enforcement": {
      "kind": "TIERED",
      "outcomes": { "high": "Failed", "medium": "Escalate" },
      "defaultOutcome": "Success"
    }
  }'
```

### 6.5 Evaluate

The primary runtime endpoint — called by the FinOS Workflows Engine when executing a Policy
Node. The path id is the **Policy Definition id**; the service resolves the calling tenant's
**Policy Configuration** — directly by `policyConfigurationId` when supplied, otherwise by the
`(workflowConfigId, nodeId)` node tuple — and evaluates it. `workflowId`, `nodeId`, and
`workflowInstanceId` are required. The loaded Configuration's `policyDefinitionId` must match the
`{policyDefinitionId}` in the path, or the call fails closed.

```bash
curl -X POST \
  https://api.example.com/finos-policy-service/v1/policies/def-invoice-approval/evaluate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "workflowInstanceId": "5281a21d-6ccd-4886-9675-678e36d66f32",
    "workflowId": "wf-invoice-v1",
    "nodeId": "confidence-gate",
    "evaluationInputs": {
      "invoiceAmount": 75000.00,
      "approvedSpendLimit": 50000.00,
      "vendorRiskScore": 4.2
    },
    "workflowContext": {},
    "agentOutput": {},
    "systemContext": {}
  }'
```

**200 Response:**
```json
{
  "data": {
    "evaluationId": "eval-8f6d2b1df2ea4d29a720eee26e472b69",
    "policyConfigurationId": "cfg_c6b766a2",
    "policyDefinitionId": "def-invoice-approval",
    "policyDefinitionVersion": 1,
    "policyVersion": "1.0.0",
    "versionSource": "current-fallback",
    "outcome": "Escalate",
    "action": "ESCALATE",
    "triggered": true,
    "reason": "Condition met: invoiceAmount > approvedSpendLimit → Escalate",
    "enforcement": { "kind": "SIMPLE" },
    "coercedInputs": { "invoiceAmount": 75000.0, "approvedSpendLimit": 50000.0 },
    "evaluationOutputs": {
      "matchedCondition": { "expression": "invoiceAmount > approvedSpendLimit", "outcome": "Escalate" },
      "defaultOutcome": "Success"
    },
    "evaluatedAt": "2026-05-18T09:01:22Z"
  },
  "meta": { "requestId": "req-xyz789", "timestamp": "2026-05-18T09:01:22Z", "apiVersion": "1.0" }
}
```

For a **TIERED** Configuration, `evaluationOutputs.matchedCondition` additionally carries `tierIndex`
(the 0-based index of the tier that fired). When nothing fires, `matchedCondition` is `null` and
`outcome` equals `defaultOutcome`.

**Fail-closed.** A missing Configuration at `(tenant, workflowConfigId, nodeId)`, a Definition-id
mismatch between the loaded Configuration and the path, a stored condition that fails to compile,
a strict-coercion failure, or an evaluator runtime error all produce a deterministic
`outcome: "Failed" / action: "BLOCK"` envelope with the reason in `reason` — never a 500. In
particular, if no Configuration exists for the node the response is a BLOCK explaining that one
must be configured first. (A firing platform invariant will also fail closed once invariants are
implemented — see §3.6.)

The calling workflow then assigns `policy_outcome = response.data.outcome` and routes on its
outgoing edge conditions (see §5 and §7).

### 6.6 List Configurations for a Definition or a workflow

```bash
# Every tenant Policy Configuration authored against a Definition
# (the "applied workflows" detail behind the usage pill)
curl "https://api.example.com/finos-policy-service/v1/policy-definitions/def-invoice-approval/policies?pageSize=25" \
  -H "Authorization: Bearer $TOKEN"

# Every Configuration in one workflow blueprint
curl "https://api.example.com/finos-policy-service/v1/policy-configurations?blueprintId=wf-invoice-v1" \
  -H "Authorization: Bearer $TOKEN"

# Every Configuration in one tenant workflow configuration
curl "https://api.example.com/finos-policy-service/v1/policy-configurations?workflowConfigId=wfc-invoice-acme" \
  -H "Authorization: Bearer $TOKEN"
```

### 6.7 Version history

```bash
# Definition versions
curl https://api.example.com/finos-policy-service/v1/policy-definitions/def-invoice-approval/versions \
  -H "Authorization: Bearer $TOKEN"

# Configuration versions (per node placement), by configurationId
curl https://api.example.com/finos-policy-service/v1/policy-configurations/cfg_c6b766a2/versions \
  -H "Authorization: Bearer $TOKEN"
```

### 6.8 List platform invariants

> **Status: Planned** — the `/invariants` endpoint is not yet implemented (see §3.6).

```bash
curl https://api.example.com/finos-policy-service/v1/invariants \
  -H "Authorization: Bearer $TOKEN"
```

---

## 7. Workflow Integration

A workflow uses a **`FinOSPolicy` node** whose `policyDefinitionId` references a **Policy
Definition**. The Definition declares the variables and the outcome vocabulary the node can
produce; the tenant's **Policy Configuration** for that node supplies the actual condition and
enforcement, and is resolved at evaluate time from `(workflowConfigId, nodeId)` — or directly by
`policyConfigurationId` when the run configuration carries one.

A workflow definition is declarative: a list of `nodes` and a list of `edges`. The `FinOSPolicy`
node's outcome is exposed as the `policy_outcome` workflow variable, and routing is expressed with
ordinary conditional edges that test that variable. Edges are evaluated in `priority` order; the
first whose `condition` is truthy (or omitted) is traversed. See
[`0-finos-workflow-guide.md`](./0-finos-workflow-guide.md) §4 for edge routing in full.

```yaml
workflowId: wf-suggest-policy-config-v1
workflowName: Suggest Supporting Items with Confidence Gate
workflowVersion: "1.0.0"

description: >
  Generates supporting-item suggestions, routes low-confidence suggestions to a
  controller review gate, and terminates when the review is complete.

inputs:
  - { name: account_id,             type: string, required: true,  source: runtime }
  - { name: document_analysis,      type: object, required: true,  source: runtime }
  - { name: auto_approve_threshold, type: number, required: true,  source: config  }

nodes:

  # ── Entry point ───────────────────────────────────────────────────────────
  - nodeId: start
    nodeType: FinOSStart
    name: Workflow Entry

  # ── Generate suggestions ──────────────────────────────────────────────────
  - nodeId: suggest-items
    nodeType: FinOSAgent
    name: Suggest Supporting Items
    agentId: agt-item-suggestion-001
    timeoutSeconds: 300
    taskInputTemplate:
      account_id: "${ .account_id | tonumber }"
      document_analysis: "${ .document_analysis }"
    outputMapping:
      suggestion_summary: "${ .summary }"
      confidence_score: "${ .confidence }"

  # ── Policy gate — confidence-based routing ────────────────────────────────
  # `policyDefinitionId` references a Policy Definition that declares the
  # available variables and the closed outcome allow-list. The tenant's Policy
  # Configuration for (workflowConfigId, nodeId) supplies the condition and
  # enforcement, and is resolved at evaluate time.
  #
  #   PolicyDefinition (def-confidence-gate) declares:
  #     variables:
  #       - confidence_score        (NUMBER, EVALUATION_INPUT, required)
  #       - auto_approve_threshold  (NUMBER, EVALUATION_INPUT, required)
  #     possibleOutcomes: [Success, Escalate, Failed]
  #
  # `evaluationInputMapping` maps workflow context values to the policy
  # service's evaluation inputs at runtime.
  - nodeId: confidence-gate
    nodeType: FinOSPolicy
    name: Auto Approve Confidence Gate
    description: >
      Evaluates confidence scores against the configured thresholds. Routes to
      auto-approve (Success), controller review (Escalate), or block (Failed).
    policyDefinitionId: def-confidence-gate
    possibleOutcomes: [Success, Escalate, Failed]
    evaluationInputMapping:
      confidence_score: "${ .confidence_score }"
      auto_approve_threshold: "${ .auto_approve_threshold }"

  # ── HITL — controller reviews suggestions ─────────────────────────────────
  - nodeId: review-suggestions
    nodeType: FinOSHITL
    name: Review Suggestions
    assignedRole: SYSTEM_ADMIN
    timeoutSeconds: 86400
    taskContextTemplate:
      suggestion_summary: "${ .suggestion_summary }"
      confidence_score: "${ .confidence_score }"
      policyReason: "${ .policy_reason }"
    allowedDecisions:
      - APPROVED
      - REJECTED

  # ── Terminals ─────────────────────────────────────────────────────────────
  - nodeId: terminal-complete
    nodeType: FinOSTerminal
    name: Complete
    exitStatus: COMPLETED
    label: "Suggestions accepted."

  - nodeId: terminal-blocked
    nodeType: FinOSTerminal
    name: Blocked
    exitStatus: COMPLETED
    label: "Policy blocked the suggestions. Valid close outcome."

edges:

  - edgeId: e-start-to-suggest
    sourceNodeId: start
    targetNodeId: suggest-items

  - edgeId: e-suggest-to-gate
    sourceNodeId: suggest-items
    targetNodeId: confidence-gate

  # Policy outcome routing — every outcome in possibleOutcomes is routed:
  - edgeId: e-block-terminal
    sourceNodeId: confidence-gate
    targetNodeId: terminal-blocked
    condition: '${ .policy_outcome == "Failed" }'
    priority: 1

  - edgeId: e-allow-terminal
    sourceNodeId: confidence-gate
    targetNodeId: terminal-complete
    condition: '${ .policy_outcome == "Success" }'
    priority: 2

  - edgeId: e-require-hitl
    sourceNodeId: confidence-gate
    targetNodeId: review-suggestions
    condition: '${ .policy_outcome == "Escalate" }'
    priority: 3

  - edgeId: e-review-approved
    sourceNodeId: review-suggestions
    targetNodeId: terminal-complete
    condition: '${ .decision == "APPROVED" }'
    priority: 1

  - edgeId: e-review-rejected
    sourceNodeId: review-suggestions
    targetNodeId: terminal-blocked
    condition: '${ .decision == "REJECTED" }'
    priority: 2
```

**At runtime** the interpreter, on reaching the `confidence-gate` node:

1. Resolves `evaluationInputMapping` against the workflow context to build `evaluationInputs`.
2. Calls `POST /policies/{policyDefinitionId}/evaluate` with `workflowId`, `nodeId`,
   `workflowInstanceId`, the run's `workflowConfigId` (or a `policyConfigurationId`), and the
   resolved `evaluationInputs`.
3. Assigns `policy_outcome`, `policy_action`, and `policy_reason` from the response `data`.
4. Traverses the first outgoing edge whose `condition` matches `policy_outcome`.

Every outcome the node can produce — the Definition's `possibleOutcomes` — must have a matching
outgoing edge condition; the workflow-validation endpoint checks this coverage and rejects
registration if any outcome is unrouted.

> **Degenerate-input convention.** A policy that may receive structurally degenerate inputs
> (an empty collection, a null value) must return a **distinct** outcome for each such case
> (e.g. `NO_ITEMS`, `INSUFFICIENT_DATA`) with a matching edge routing it to an appropriate
> `FinOSTerminal`. Returning a normal-path outcome such as `Escalate` for a degenerate input is a
> policy specification error — it routes the workflow into a review gate with no meaningful
> context for the reviewer.

The interpreter may additionally record a `FinOSPolicyEvent` in the Execution History graph (see
`finos-workflow-execution-history-openapi.yaml`), carrying the `outcome`, `evaluationOutputs`, and
the `policyDefinitionId` / `policyDefinitionVersion` / `policyConfigurationId` identifiers.

---

## 8. Platform Invariants

> **Status: Planned.** Not yet implemented — see §3.6.

Platform invariants are distinguished from tenant Policy Configurations:

| Characteristic | Policy Configuration | Platform Invariant |
|---|---|---|
| Authored by | Tenants | Platform only (system-seeded) |
| Tenant-scoped | Yes | No (platform-wide) |
| Mutable via API | Yes (versioned) | No (read-only) |
| Bound to a node | Yes (`workflowConfigId`, `nodeId`) | No (cross-cutting) |
| Evaluated | On its node's evaluate call | On **every** evaluate call, before the target Configuration |

A firing invariant BLOCKs and overrides whatever the target Configuration would have decided, even
for a nonexistent Policy Definition id. The envelope names which invariant fired.

Examples:
- `INV-001` — FinOS AI Agents must not perform write mutations on systems of record
- `INV-002` — HITL decisions must carry a named reviewer identity
- `INV-003` — Agent invocations must carry a valid `callingContext`

---

## 9. Error Reference

| Status | Type URI | When |
|---|---|---|
| `400` | `.../errors/bad-request` | Malformed JSON or invalid query parameters |
| `401` | `.../errors/unauthorized` | Missing or expired credentials |
| `403` | `.../errors/forbidden` | Authenticated but insufficient scope |
| `404` | `.../errors/not-found` | Definition, Configuration, version, or invariant not found |
| `409` | `.../errors/conflict` | Invalid Definition status transition |
| `422` | `.../errors/validation-failed` | Condition references a variable not in the pinned Definition version, an outcome not in its `possibleOutcomes`, a tier-name/outcome-key mismatch, or a required input absent from `evaluationInputs` |
| `503` | `.../errors/service-unavailable` | Datastore unavailable or critical dependency down |

Runtime evaluation failures (missing Configuration, Definition-id mismatch, compile error,
coercion failure, invariant match) do **not** surface as HTTP errors — they return a deterministic `outcome: "Failed" / action: "BLOCK"` envelope
(see §6.5).

**Example 422 on Configuration save — outcome not in the Definition allow-list:**
```json
{
  "type": "https://api.example.com/finos-policy-service/errors/validation-failed",
  "title": "Policy Validation Failed",
  "status": 422,
  "detail": "Outcome 'RequiresApproval' is not in the pinned Definition version's allowed list: [AttentionRequired, Escalate, Failed, Success].",
  "errors": [
    {
      "field": "enforcement.tiers[0].outcome",
      "code": "outcome-not-in-allowlist",
      "message": "'RequiresApproval' is not a member of Definition def-invoice-approval version 1 possibleOutcomes."
    }
  ]
}
```

---

## 10. Appendix — Schema Field Reference

### PolicyDefinition — All Fields

| Field | Type | Mutable | Notes |
|---|---|---|---|
| `definitionId` | `string` | No | Client-supplied or server-generated |
| `name` | `string` | Yes | |
| `description` | `string` | Yes | |
| `variables` | `ConditionVariable[]` | Yes | Derived from `condition` when one is present, else declared directly. **Creates new version on change** |
| `condition` | `PolicyCondition` | Yes | **Optional.** When present, `variables` is derived from its leaves and may carry `CONFIGURATION_INPUT` operands. **Creates new version on change** |
| `possibleOutcomes` | `string[]` | Yes | **Closed allow-list**; **creates new version on change** |
| `currentVersion` | `string` | Yes (server) | Semantic version |
| `status` | `PolicyStatus` | Via `/status` | DRAFT / ACTIVE / INACTIVE / DEPRECATED / DISABLED |
| `availableToBlueprints` | `integer` | Read-only | Distinct blueprints referencing this Definition |
| `appliedToWorkflows` | `integer` | Read-only | Distinct workflows with a Policy Configuration against this Definition |
| `createdBy` / `createdAt` | | No | Server-set |
| `lastModifiedBy` / `lastModifiedAt` | | Yes / server | |

### PolicyConfiguration — All Fields

| Field | Type | Mutable | Notes |
|---|---|---|---|
| `configurationId` | `string` | No | Server-minted (`cfg_<uuid>`) |
| `tenantId` | `string` | No | From the authenticated principal |
| `blueprintId` | `string` | No | The workflow blueprint the placement belongs to |
| `workflowConfigId` | `string` | No | Bound at creation; part of identity |
| `nodeId` | `string` | No | Bound at creation; part of identity |
| `policyDefinitionId` | `string` | No | The Definition this Configuration configures |
| `policyDefinitionVersion` | `integer` | No | Pinned permanently at save time |
| `condition` | `PolicyCondition` | Yes | **Top-level** condition tree (not inside `enforcement`). **Creates new version on change** |
| `enforcement` | `SIMPLE \| TIERED` | Yes | Outcome mapping only. **Creates new version on change** |
| `currentVersion` | `string` | Yes (server) | This Configuration's own version |
| `createdBy` / `createdAt` | | No | Server-set |
| `lastModifiedBy` / `lastModifiedAt` | | Yes / server | |

> A Configuration is identified by **(tenantId, workflowConfigId, nodeId)** and has **no**
> `status`, `scope`, `enabled`, `overridable`, or standalone `possibleOutcomes` field — it is
> always active, already scoped to one node, and its outcome vocabulary is owned by the pinned
> Definition version.

### Enforcement (Policy Configuration) — kinds

The `condition` is a sibling top-level field; `enforcement` carries only the outcome mapping.

| `kind` | Shape |
|---|---|
| `SIMPLE` | `{ outcome, defaultOutcome }` |
| `TIERED` | `{ outcomes: { <tierName>: <outcome>, ... }, defaultOutcome }` — keyed by the `name` of each top-level `condition.subConditions` tier |

### Standard Outcome Values

| Value | Meaning | Typical workflow routing |
|---|---|---|
| `Success` | No condition/tier fired (equals `defaultOutcome`) | Proceed to auto-approval path |
| `Escalate` | A condition met; human review required | Route to `FinOSHITL` node |
| `AttentionRequired` | Advisory; action warranted but not blocking | Route to HITL or notify |
| `Failed` | Block action | Route to rejection terminal |

The actual set of valid outcomes for any node is the pinned Definition version's
`possibleOutcomes` — a Definition may declare additional custom labels (e.g. `NO_ITEMS`,
`INSUFFICIENT_DATA`). See the degenerate-input convention in [§7](#7-workflow-integration).
