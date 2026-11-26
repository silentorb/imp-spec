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
| `bounds` | `Type[]` | may |

Opaque type parameter declared on a polymorphic catalog entry. See [graph-model.md](./graph-model.md) for `Type` variable references. Each bound is a type, typically a named union from a [type definition](./type-definitions.md) (e.g. `A: Audio`); a type argument satisfies the param when it is assignable to every bound — see [type-system.md](../imp-typecheck/type-system.md).

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

- `NodeDefinition.typeParams` declares opaque parameters (e.g. `T`) referenced in port `Type` templates via `{ "param": "T" }`.
- Params are **scoped to the definition**; the checker unifies them across that definition's ports and with wired edges.
- Graph **`Node.inputs`** are **local literal values** (`InputValues`), not `Ports` — see [graph-model.md](./graph-model.md).
- **`defaultValue` on an input `Port`** makes that port optional; omitting it means the port is required.
- **`variadic: true` on an input `Port`** marks multi-edge fan-in; `Port.type` is the element type. Allowed on both primitive and graph-backed definitions — see [graph-model.md](./graph-model.md).
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

**Variadic inputs:** bindings stay **1:1** (one declared port → one boundary `input` node). For a variadic declared input of element type `T`, the bound boundary `input`’s `value` type must be **`list<T>`** (not `T`). Late resolve gathers ordered parent edges via core `pack` into that list signal — see [resolve.md](../imp-graph-resolve/resolve.md) and [type-system.md](../imp-typecheck/type-system.md).

### Implementation dispatch

Primitive `NodeDefinition` entries may share one graph-facing **signature** while resolving to **different host implementations** depending on resolved type arguments. That is distinct from **overload sets** (multiple signatures under one `NodeTypeId`) — see below.

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
| `match` | `Type` | must |
| `implementation` | `ImplementationId` | must |

`match` is a **pattern** type. A case matches when the resolved type argument is **assignable** to the normalized pattern: `{ "id": "any" }` matches every type, a union pattern (or a named one such as `Audio`) matches any of its members, and the fixed subtype relation applies (`list<string>` matches `collection<string>`). Patterns must not contain type variables.

#### Dispatch resolution

1. Resolve (normalized) type arguments for the node instance (from `Node.typeArgs` or edge inference) — see [type-system.md](../imp-typecheck/type-system.md).
2. For `universal`, return `implementation.id`.
3. For `dispatch`, read `typeArgs[paramIndex]` (default index `0`), return the first matching `cases[].implementation`, else `default`, else error.

### NodeLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must |
| `definitions` | `NodeDefinition[]` | must (may be empty) |
| `types` | `TypeDefinition[]` | may |

A named, declarative collection of `NodeDefinition`s, plus the [type definitions](./type-definitions.md) they rely on (e.g. a library that bounds its audio params by `Audio` ships `Audio` here). Libraries **must not** require `imp-registry`; they implement the core library interface alone.

### Overload sets

Multiple `NodeDefinition` entries **may share the same `id`**. Together they form an **overload set** for that `NodeTypeId` (the graph `Node.type` symbol).

Rules:

- Each entry is a full signature (ports, `typeParams`, `implementation` / body).
- Signatures in one overload set should be **distinguishable** by port types so edge-driven selection can pick one — see [type-system.md](../imp-typecheck/type-system.md) (Overload resolution).
- Loading **must reject** a new signature that is **equivalent** to an already-registered overload for the same id (same port ids with unify-equal types on each corresponding port after ignoring param renaming). Distinct kinds (e.g. `list<T>` vs `set<T>` on the same port names) are allowed.
- `implementation.dispatch` remains a mechanism **within one signature**. Kind-specific authoring polymorphism uses overload sets (e.g. `filter` as `list→list` and `set→set`), typically with distinct universal implementation ids such as `filter_list` / `filter_set`.

Port map key / `Port.id` equality matches [graph-model.md](./graph-model.md).

Graph-backed and primitive definitions in JSON travel in the `definitions` section of an [Imp file](../imp-file/imp-files.md), with the same shape.

### Naming convention

Use **full words** for `NodeTypeId`s unless the abbreviation is a widely understood term on its own (e.g. `filter`, `sort`, `limit` are fine; prefer `equals` / `not_equals` over `eq` / `neq`).

For library-authored graph-backed definitions, **ULIDs** are recommended for `id` to reduce cross-library collisions — see [imp-files.md](../imp-file/imp-files.md).

## Design rationale

- **`Node` stays the graph instance**; **`NodeDefinition` is the catalog** so instance vs type naming stays clear (`Node.type` ↔ `NodeDefinition.id`).
- **Single surface** — primitive operators and graph-backed composites both appear as `Node.type` references; whether internals are builtin translation or a nested graph is a catalog concern.
- Library packages depend only on **`imp-core-types`**, so they can be published and composed without pulling registry machinery.
- **`typeParams`** enable polymorphic definitions without a separate type-expression language.
- **`implementation`** keeps the graph/catalog interface separate from host-specific translation bindings.
- **Overload sets** let one `Node.type` symbol map to kind-specific signatures without a single general family definition.
- **`body` + `bindings`** replace the former separate `GraphType` catalog — declared ports are the external contract; `body` is the implementation.
- **Variadic → `list`** lets composites consume N ordered external wires as one internal sequence without a second container type.

## Behavior / pipeline

This feature is a **data shape** only. Loading, validation, and late resolution are documented in [registry.md](../imp-registry/registry.md), [type-system.md](../imp-typecheck/type-system.md), and [resolve.md](../imp-graph-resolve/resolve.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [imp-files.md](../imp-file/imp-files.md)
- [type-definitions.md](./type-definitions.md)
- [registry.md](../imp-registry/registry.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- Root [AGENTS.md](../../../AGENTS.md)
