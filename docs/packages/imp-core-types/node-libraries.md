# Node libraries

## Summary

Declarative **catalogs of node types** that library packages can ship without depending on a registry. Type shapes are defined alongside the graph model in `imp-core-types`; loading/lookup lives in `imp-registry`.

## When to read this

- Defining or changing `NodeType` or `NodeLibrary`
- Authoring a package that exports a library of Imp node types
- Regenerating library interfaces in bindings

## Requirements

### NodeType

| Field | Type | Required |
| --- | --- | --- |
| `id` | `NodeTypeId` | must |
| `inputs` | `Ports` | must (may be empty) |
| `outputs` | `Ports` | must (may be empty) |

A **catalog entry** for a node type: **port templates** keyed by the same `NodeTypeId` used on graph `Node.type`. Informal "node definition" in prose maps to `NodeType`.

- `NodeType.inputs` / `outputs` are `Ports` maps of `Port` templates (`id`, `type`, optional `defaultValue`).
- Graph **`Node.inputs`** are **local literal values** (`InputValues`), not `Ports` — see [graph-model.md](./graph-model.md).
- **`defaultValue` on an input `Port`** makes that port optional; omitting it means the port is required.

A type's `id` must equal its key in the parent `NodeLibrary.types` map when the library is well-formed. Port map key / `Port.id` equality matches [graph-model.md](./graph-model.md).

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

## Behavior / pipeline

This feature is a **data shape** only. Loading and conflict policy are documented in [registry.md](../imp-registry/registry.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
