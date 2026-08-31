# Node libraries

## Summary

Declarative **catalogs of node types** that library packages can ship without depending on a registry. Type shapes are defined alongside the graph model in `imp-core-types`; loading/lookup lives in `imp-registry`.

## When to read this

- Defining or changing `NodeType` or `NodeLibrary`
- Authoring a package that exports a library of Imp node types
- Regenerating library interfaces in bindings

## Requirements

### TypeParam

| Field | Type | Required |
| --- | --- | --- |
| `id` | `TypeParamId` | must |

Opaque type parameter declared on a polymorphic catalog entry. See [graph-model.md](./graph-model.md) for `SignalType` variable references.

### NodeType

| Field | Type | Required |
| --- | --- | --- |
| `id` | `NodeTypeId` | must |
| `typeParams` | `TypeParam[]` | may |
| `inputs` | `Ports` | must (may be empty) |
| `outputs` | `Ports` | must (may be empty) |
| `implementation` | `NodeImplementation` | may |

A **catalog entry** for a node type: **port templates** keyed by the same `NodeTypeId` used on graph `Node.type`. Informal "node definition" in prose maps to `NodeType`.

- `NodeType.typeParams` declares opaque parameters (e.g. `T`) referenced in port `SignalType` templates via `{ "param": "T" }`.
- Params are **scoped to the node**; the checker unifies them across that node's ports and with wired edges.
- `NodeType.implementation` declares how hosts pick a lowering/execution binding — see [Implementation dispatch](#implementation-dispatch) below.
- `NodeType.inputs` / `outputs` are `Ports` maps of `Port` templates (`id`, `type`, optional `defaultValue`).
- Graph **`Node.inputs`** are **local literal values** (`InputValues`), not `Ports` — see [graph-model.md](./graph-model.md).
- **`defaultValue` on an input `Port`** makes that port optional; omitting it means the port is required.

A type's `id` must equal its key in the parent `NodeLibrary.types` map when the library is well-formed. Port map key / `Port.id` equality matches [graph-model.md](./graph-model.md).

### Implementation dispatch

Higher-level `NodeType` catalog entries may share one graph-facing interface while resolving to **different host implementations** depending on resolved type arguments. Other nodes use a **single universal** implementation for all instantiations.

`ImplementationId` is a string naming a host lowering/execution binding (often equal to `NodeTypeId` for universal nodes).

#### NodeImplementation

Tagged union:

| Variant | Fields | Role |
| --- | --- | --- |
| `universal` | `kind: "universal"`, `id` | Always use `id` regardless of type args |
| `dispatch` | `kind: "dispatch"`, `cases`, optional `paramIndex`, optional `default` | Pick `implementation` from resolved type args |

**Omitted `implementation`** on a `NodeType` is equivalent to `{ "kind": "universal", "id": "<NodeTypeId>" }`.

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

Collection-preserving transforms (e.g. `filter`) typically use **universal** implementations (`filter` lowers the same way for every `collection<T>`). Type-parametric nodes that need different lowering per `T` (e.g. a generic `cast<T>`) use **dispatch**.

### NodeLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must |
| `types` | map `NodeTypeId` → `NodeType` | must (may be empty) |

A named, declarative collection of `NodeType`s. Libraries **must not** require `imp-registry`; they implement the core library interface alone.

A library does not execute nodes. Executable bindings to types, if added later, may stay separate.

### Naming convention

Use **full words** for `NodeTypeId`s unless the abbreviation is a widely understood term on its own (e.g. `filter`, `sort`, `limit` are fine; prefer `equals` / `not_equals` over `eq` / `neq`).

## Design rationale

- **`Node` stays the graph instance**; **`NodeType` is the catalog** so instance vs type naming stays clear (`Node.type` ↔ `NodeType.id`).
- Library packages depend only on **`imp-core-types`**, so they can be published and composed without pulling registry machinery.
- Catalogs are **data shapes** (plain objects), not classes with behavior.
- Port templates on the catalog carry signal types and defaults; instance literals live on `Node.inputs`.
- **`typeParams`** enable polymorphic node definitions (e.g. `filter` preserving `collection<T>`) without a separate type-expression language.
- **`implementation`** keeps the graph/catalog interface separate from host-specific lowering bindings, with optional per-type-arg dispatch.

## Behavior / pipeline

This feature is a **data shape** only. Loading and conflict policy are documented in [registry.md](../imp-registry/registry.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [graph-types.md](./graph-types.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
