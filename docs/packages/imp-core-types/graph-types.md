# Graph types

## Summary

Declarative **catalogs of reusable subgraph interfaces** (`GraphType`) that declare external input/output ports with optional opaque type parameters. v1 is interface-only — no embedded graph body or composite `NodeType` lowering.

## When to read this

- Defining or changing `GraphType` or `GraphTypeLibrary`
- Validating that a graph's boundary nodes implement a declared interface
- Regenerating graph-type interfaces in bindings

## Requirements

### GraphType

| Field | Type | Required |
| --- | --- | --- |
| `id` | `GraphTypeId` | must |
| `typeParams` | `TypeParam[]` | may |
| `inputs` | `Ports` | must (may be empty) |
| `outputs` | `Ports` | must (may be empty) |

A **catalog entry** for a subgraph's external interface. Port templates use the same `Port` / `SignalType` shapes as `NodeType` — see [graph-model.md](./graph-model.md) and [node-libraries.md](./node-libraries.md).

- `GraphTypeId` is a string, distinct from `NodeTypeId` namespace in practice but same underlying string type.
- `typeParams` and variable references in port types follow the same rules as polymorphic `NodeType`s.

### GraphTypeLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must |
| `types` | map `GraphTypeId` → `GraphType` | must (may be empty) |

A named, declarative collection of `GraphType`s. Libraries **must not** require `imp-registry`; they implement the core interface alone.

A type's `id` must equal its key in the parent `GraphTypeLibrary.types` map when the library is well-formed.

### Implementing a GraphType

A graph **implements** a `GraphType` when:

1. For each declared **input** port, the graph has a core boundary **`input`** node whose output `value` port type is compatible with the `GraphType` input port type (after type-parameter substitution).
2. For each declared **output** port, the graph has a core boundary **`output`** node whose input `value` port type is compatible with the `GraphType` output port type.

Port identity mapping (which boundary node corresponds to which `GraphType` port) is validated by the checker — see [type-system.md](../imp-typecheck/type-system.md).

v1 does **not** require a single canonical boundary-node naming scheme beyond type compatibility; hosts may supply explicit boundary-node bindings in a future revision.

## Design rationale

- **Interface-only v1** separates declaring reusable subgraph contracts from embedding graph bodies or lowering to composite nodes.
- Reuses `Port` / `SignalType` / `TypeParam` from the graph model so graph-type catalogs stay consistent with node-type catalogs.
- Library packages depend only on **`imp-core-types`**, matching `NodeLibrary` packaging.

## Behavior / pipeline

This feature is a **data shape** only. Loading, lookup, and `checkGraphImplements` live in `imp-registry` and `imp-typecheck` — see [registry.md](../imp-registry/registry.md) and [type-system.md](../imp-typecheck/type-system.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [node-libraries.md](./node-libraries.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
