# Graph model

## Summary

Imp's core data model is a portable **directed graph** of nodes and port-to-port edges, intended as a universal transmission format for DAGs. This document is the language-agnostic source of truth for regenerating code type interfaces.

## When to read this

- Defining or changing `Graph`, `Node`, `Edge`, port, or id types
- Regenerating binding interfaces from the model
- Designing converters that map another graph format onto Imp

## Requirements

### Identifiers

All of the following are **strings**:

| Name | Role |
| --- | --- |
| `NodeId` | Key for a node in `Graph.nodes` |
| `EdgeId` | Key for an edge in `Graph.edges` |
| `NodeTypeId` | Discriminator / type name for a node (function definition id); identity of a catalog `NodeType` — see [node-libraries.md](./node-libraries.md) |
| `PortId` | Key for an input or output port on a node |
| `SignalTypeId` | Identity of an Imp concrete signal type |
| `TypeParamId` | Identity of a type parameter declared on a `NodeDefinition` |
| `TypeConstraintId` | Identity of a named nominal type constraint in a `TypeConstraintLibrary` |

### TypeParam

| Field | Type | Required |
| --- | --- | --- |
| `id` | `TypeParamId` | must |
| `bounds` | `TypeConstraintId[]` | may |

Declares an **opaque** type parameter (e.g. `T`) on a polymorphic catalog entry. Params are scoped to the declaring `NodeDefinition`.

Optional `bounds` reference named constraints from a `TypeConstraintLibrary` — see [type-constraints.md](./type-constraints.md). Omitted = unconstrained.

### SignalType

`SignalType` is a **tagged union** of two variants, JSON-discriminated by field presence:

| Variant | Fields | Example |
| --- | --- | --- |
| **Concrete** | `id` (`SignalTypeId`), optional `args` (`SignalType[]`) | `{ "id": "collection", "args": [{ "param": "T" }] }` |
| **Variable** | `param` (`TypeParamId`) | `{ "param": "T" }` |

Rules:

- **`args`** is only valid on the concrete variant (e.g. `collection<T>`).
- A concrete type with **no** `args` is a nullary signal type (e.g. `{ "id": "boolean" }`).
- Bare `{ "id": "collection" }` without `args` is **not** a well-formed `collection` type after library migration — use `collection<any>` or a type parameter.
- **`any`** is a concrete top type; static checking treats it as compatible with any other signal type.
- Structural/record types and type aliases are out of scope for this revision — see [type-system.md](../imp-typecheck/type-system.md). Nominal type constraints on `TypeParam.bounds` are specified in [type-constraints.md](./type-constraints.md).

### PrimitiveValue

A **literal** value that may appear on a node instance input or as a port `defaultValue`:

| Kind | Value type |
| --- | --- |
| string | string |
| number | number |
| boolean | boolean |
| null | null |

Union: `string | number | boolean | null`.

### Port (pair)

| Field | Type | Required |
| --- | --- | --- |
| `id` | `PortId` | must |
| `type` | `SignalType` | must |
| `defaultValue` | `PrimitiveValue` | may |

One port template: identity, signal type, and optional default. Used on **catalog** `NodeDefinition.inputs` / `NodeDefinition.outputs` — not on graph node instances.

- **`defaultValue` unset** — input port is **required** (must resolve via an incoming edge or a local instance literal).
- **`defaultValue` set** — input port is **optional**; falls back to that value when unwired and no local literal is present.

### Ports (container)

`Ports` is a map `PortId` → `Port`. Used for both `NodeDefinition.inputs` and `NodeDefinition.outputs`.

A port's `id` must equal its key in the parent `Ports` map when the graph/library is well-formed.

### PortReference

| Field | Type | Required |
| --- | --- | --- |
| `node` | `NodeId` | must |
| `port` | `PortId` | must |

Identifies a specific port on a specific node.

### InputValues

Local **literal values** on a graph node instance, keyed by input `PortId`:

Partial map `PortId` → `PrimitiveValue`.

Only keys that are valid input ports on the node's `NodeDefinition` should appear. Keys may be omitted when the value comes from an edge or a catalog `defaultValue`.

### Node

| Field | Type | Required |
| --- | --- | --- |
| `id` | `NodeId` | must |
| `type` | `NodeTypeId` | must |
| `typeArgs` | `SignalType[]` | may |
| `inputs` | `InputValues` | must (may be empty) |

A **node instance**: identity, type reference, and local literal input values. Port templates (signal types, defaults, output ports) live on the catalog `NodeDefinition` — see [node-libraries.md](./node-libraries.md).

When the catalog `NodeDefinition` declares `typeParams`, `typeArgs` supplies **concrete** `SignalType` values for those parameters in declaration order. Omit `typeArgs` when edge wiring provides enough constraints for the checker to infer them; required when a polymorphic node has no inferrable inputs (e.g. a standalone boundary `input` node).

Graph-backed definitions appear as ordinary node instances: `Node.type` references the definition id; the internal `body` graph is resolved late — see [resolve.md](../imp-graph-resolve/resolve.md).

A node's `id` must equal its key in `Graph.nodes` when the graph is well-formed.

### Edge

| Field | Type | Required |
| --- | --- | --- |
| `from` | `PortReference` | must |
| `to` | `PortReference` | must |

Edges connect **ports**, not bare nodes. Direction is `from` → `to` (output toward input in typical dataflow usage).

An edge's identity is its key in `Graph.edges` (`EdgeId`); the `Edge` value itself has no `id` field.

### Graph

| Field | Type | Required |
| --- | --- | --- |
| `nodes` | map `NodeId` → `Node` | must (may be empty) |
| `edges` | map `EdgeId` → `Edge` | must (may be empty) |

Maps in other languages should use that language's idiomatic string-keyed dictionary.

### Input port resolution

Each input port on a node instance may have a **local literal** in `Node.inputs`, an **incoming edge**, both, or neither. Storage is not mutually exclusive. Resolution order:

1. **Edge** — if an edge targets the port, use the wired value (overrides any local literal).
2. Else **local literal** — `Node.inputs[portId]` when present.
3. Else **`Port.defaultValue`** from the catalog `NodeDefinition` input template.
4. Else **error** — the port is required and unsatisfied.

When an edge is removed in an editor, the node reverts to its stored local literal (or catalog default) instead of going blank.

### Core boundary nodes

Graphs act as **subgraphs** with an external interface. Core Imp ships boundary definitions in `coreNodeLibrary` (`id: "imp.core"`):

| Definition | Ports | Role |
| --- | --- | --- |
| `input` | outputs: single `value` port | Brings a value **into** the graph from outside |
| `output` | inputs: single `value` port | Sends a value **out** of the graph to the host |
| `parameter` | inputs: optional `label` (`string`, default `""`), `value` (`any`, default `null`); outputs: `value` (`T`) | Declares a host-configurable graph parameter. The instance `value` is the default; hosts may override it at execute time. `label` is UI metadata (not used in SQL). Lowers like `literal` (bound parameter / literal from `value`). Boundary nodes are polymorphic over `T` on `value` ports — see [node-libraries.md](./node-libraries.md). |

**One node instance per boundary port** (not a multi-port patch board). A subgraph with three inputs and one output is three `input` nodes + one `output` node. Editors may group boundary nodes visually without changing the transmission model.

A host wires external values into `input` nodes and reads results from `output` nodes. Hosts that expose settings UI discover `parameter` nodes, present `label` + current `value`, and bind overrides into each parameter node's `inputs.value` before lowering. Reusable subgraphs are **`NodeDefinition` entries with `body` + `bindings`** — see [node-libraries.md](./node-libraries.md) and [graph-libraries.md](./graph-libraries.md). Parent graphs reference them by `Node.type`; resolution is late — see [resolve.md](../imp-graph-resolve/resolve.md).

### Invariants (well-formed graphs)

These are design requirements for validators and converters:

1. Every `Node.id` must equal its key in `Graph.nodes`.
2. Every `Port.id` must equal its key in the parent `Ports` map on a `NodeDefinition`.
3. For every edge, `from.node` and `to.node` must exist in `Graph.nodes`.
4. For every edge, `from.port` must exist in that node's `NodeDefinition.outputs`, and `to.port` must exist in that node's `NodeDefinition.inputs` (when types are known via a registry).
5. `Node.inputs` keys must be valid input `PortId`s for the node's `NodeDefinition` (when type is known).
6. Required input ports (no `defaultValue`, no incoming edge, no local literal) are ill-formed when validated.
7. An input port **may** have both an edge and a local literal; the edge wins at resolution.
8. The graph is intended to be a **DAG** (no directed cycles) for Imp transmission use cases; cycle detection is a future validation concern.

## Design rationale

- **Port-level edges** keep connectivity precise when a node has many inputs/outputs, and align with handle-based UI graphs (see [react-flow.md](../imp-react-flow/react-flow.md)).
- **Nodes as functions** — catalog `NodeDefinition` ports are parameter and return slots; instance `Node.inputs` hold local literal values for parameters.
- **Instance literals on the node** — initial values must live somewhere; storing them on the instance (not a side map) matches common dataflow tooling and lets an edge override without discarding the local fallback.
- **`defaultValue` on port templates** — required vs optional is expressed by omitting or setting a default on the catalog `Port`, not a separate flags field.
- **`SignalType`** names the Imp type of a signal on a port; the concrete / variable union supports opaque generics (`collection<T>`) without a separate type-expression AST.
- **Record/map keyed by id** makes merge, lookup, and partial update straightforward for transmission and UI state.
- **Separate `EdgeId`** allows multiple edges and stable identity without encoding topology into the id.
- **Boundary `input` / `output` / `parameter` nodes** — `input`/`output` are one instance per host port; `parameter` declares host-configurable defaults that editors can expose as settings.
- **Catalog vs instance** — `NodeDefinition` holds port templates (and optional `body`); `Node` holds identity, type id, and local input values (`Node.type` ↔ `NodeDefinition.id`). See [node-libraries.md](./node-libraries.md).

## Behavior / pipeline

This feature is a **data shape** only. Serialization format (JSON, etc.), validation, and converters are separate packages.

## Configuration

None.

## See also

- [node-libraries.md](./node-libraries.md) — `NodeDefinition` / `NodeLibrary` catalogs
- [graph-libraries.md](./graph-libraries.md) — JSON graph library format
- [resolve.md](../imp-graph-resolve/resolve.md) — late nested graph resolution
- [type-constraints.md](./type-constraints.md) — `TypeConstraint` / `TypeConstraintLibrary` catalogs
- [type-system.md](../imp-typecheck/type-system.md) — static type checking
- [registry.md](../imp-registry/registry.md) — loading libraries and looking up types
- [react-flow.md](../imp-react-flow/react-flow.md)
- Root [AGENTS.md](../../../AGENTS.md)
