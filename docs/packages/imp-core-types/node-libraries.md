# Node libraries

## Summary

Declarative **catalogs of node definitions** that library packages can ship without depending on a registry. Each `NodeDefinition` describes the external port interface for graph node instances; entries are either **primitive** (host translation via `implementation`) or **graph-backed** (internal `body` graph). Type shapes live in `imp-core-types`; loading/lookup lives in `imp-registry`.

## When to read this

- Defining or changing `NodeDefinition` or `NodeLibrary`
- Authoring a package that exports a library of Imp node types
- Regenerating library interfaces in bindings

## Requirements

### TypeParam

| Field | Type | Required |
| --- | --- | --- |
| `id` | `TypeParamId` | must |
| `bounds` | `TypeConstraintId[]` | may |

Opaque type parameter declared on a polymorphic catalog entry. See [graph-model.md](./graph-model.md) for `SignalType` variable references and [type-constraints.md](./type-constraints.md) for `bounds`.

### NodeDefinition

| Field | Type | Required |
| --- | --- | --- |
| `id` | `NodeTypeId` | must |
| `name` | `string` | may — human display label, decoupled from `id` |
| `typeParams` | `TypeParam[]` | may |
| `inputs` | `Ports` | must (may be empty) |
| `outputs` | `Ports` | must (may be empty) |
| `implementation` | `NodeImplementation` | may — **primitive only** |
| `body` | `Graph` | may — **graph-backed only** |
| `bindings` | `BoundaryBindings` | may — **required when `body` present** |

A **catalog entry** for a node type: **port templates** keyed by the same `NodeTypeId` used on graph `Node.type`.

Two kinds (discriminated by fields, not a separate enum):

| Kind | Required fields | Forbidden fields |
| --- | --- | --- |
| **Primitive** | `inputs`, `outputs`; `implementation` optional (defaults to universal) | `body`, `bindings` |
| **Graph-backed** | `inputs`, `outputs`, `body`, `bindings` | `implementation` |

- `NodeDefinition.typeParams` declares opaque parameters (e.g. `T`) referenced in port `SignalType` templates via `{ "param": "T" }`.
- Params are **scoped to the definition**; the checker unifies them across that definition's ports and with wired edges.
- Graph **`Node.inputs`** are **local literal values** (`InputValues`), not `Ports` — see [graph-model.md](./graph-model.md).
- **`defaultValue` on an input `Port`** makes that port optional; omitting it means the port is required.
- Parent-graph `Node.type` references a `NodeDefinition.id` — no special reference node type. Transmitted graphs **never** embed resolved subgraph bodies on instances.

### BoundaryBindings

| Field | Type | Required |
| --- | --- | --- |
| `inputs` | map `PortId` → `NodeId` | must when `body` present |
| `outputs` | map `PortId` → `NodeId` | must when `body` present |

Maps each declared external port to a core boundary node in `body`:

- `inputs[portId]` → an `input` node id in `body`
- `outputs[portId]` → an `output` node id in `body`

Every key in `NodeDefinition.inputs` / `outputs` must appear in the corresponding bindings map. Bound boundary nodes must exist in `body.nodes` and have the expected core node types.

### Implementation dispatch

Primitive `NodeDefinition` entries may share one graph-facing interface while resolving to **different host implementations** depending on resolved type arguments.

`ImplementationId` is a string naming a host translation binding (often equal to `NodeDefinition.id` for universal nodes).

#### NodeImplementation

Tagged union:

| Variant | Fields | Role |
| --- | --- | --- |
| `universal` | `kind: "universal"`, `id` | Always use `id` regardless of type args |
| `dispatch` | `kind: "dispatch"`, `cases`, optional `paramIndex`, optional `default` | Pick `implementation` from resolved type args |

**Omitted `implementation`** on a primitive definition is equivalent to `{ "kind": "universal", "id": "<NodeDefinition.id>" }`.

#### ImplementationCase

| Field | Type | Required |
| --- | --- | --- |
| `match` | concrete `SignalType` | must |
| `implementation` | `ImplementationId` | must |

`match` is a **pattern**: `{ "id": "any" }` matches any concrete type; otherwise `id` and `args` must match structurally.

#### Dispatch resolution

1. Resolve concrete type arguments for the node instance (from `Node.typeArgs` or edge inference) — see [type-system.md](../imp-typecheck/type-system.md).
2. For `universal`, return `implementation.id`.
3. For `dispatch`, read `typeArgs[paramIndex]` (default index `0`), return the first matching `cases[].implementation`, else `default`, else error.

### NodeLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must |
| `definitions` | `NodeDefinition[]` | must (may be empty) |

A named, declarative collection of `NodeDefinition`s. Libraries **must not** require `imp-registry`; they implement the core library interface alone.

Each definition's `id` must be unique within the library. Port map key / `Port.id` equality matches [graph-model.md](./graph-model.md).

Graph-backed definitions in JSON files use the same shape — see [graph-libraries.md](./graph-libraries.md).

### Naming convention

Use **full words** for `NodeTypeId`s unless the abbreviation is a widely understood term on its own (e.g. `filter`, `sort`, `limit` are fine; prefer `equals` / `not_equals` over `eq` / `neq`).

For library-authored graph-backed definitions, **ULIDs** are recommended for `id` to reduce cross-library collisions — see [graph-libraries.md](./graph-libraries.md).

## Design rationale

- **`Node` stays the graph instance**; **`NodeDefinition` is the catalog** so instance vs type naming stays clear (`Node.type` ↔ `NodeDefinition.id`).
- **Single surface** — primitive operators and graph-backed composites both appear as `Node.type` references; whether internals are builtin translation or a nested graph is a catalog concern.
- Library packages depend only on **`imp-core-types`**, so they can be published and composed without pulling registry machinery.
- **`typeParams`** enable polymorphic definitions without a separate type-expression language.
- **`implementation`** keeps the graph/catalog interface separate from host-specific translation bindings.
- **`body` + `bindings`** replace the former separate `GraphType` catalog — declared ports are the external contract; `body` is the implementation.

## Behavior / pipeline

This feature is a **data shape** only. Loading, validation, and late resolution are documented in [registry.md](../imp-registry/registry.md), [type-system.md](../imp-typecheck/type-system.md), and [resolve.md](../imp-graph-resolve/resolve.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [graph-libraries.md](./graph-libraries.md)
- [type-constraints.md](./type-constraints.md)
- [registry.md](../imp-registry/registry.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- Root [AGENTS.md](../../../AGENTS.md)
