# Registry

## Summary

**`imp-registry`** loads `NodeLibrary` and `GraphLibrary` values and provides lookup of `NodeDefinition`s by `NodeTypeId`. Callers pass library objects they already have — registry performs no file I/O.

## When to read this

- Implementing or changing registry load/lookup behavior
- Deciding how apps compose multiple type libraries

## Requirements

### API

| Operation | Behavior |
| --- | --- |
| `createRegistry()` | Returns an empty registry |
| `loadNodeLibrary(registry, library)` | Returns a **new** registry with primitive + graph-backed definitions merged in (immutable) |
| `loadGraphLibrary(registry, library)` | Same merge as `loadNodeLibrary`; validates entries have `body` (graph-backed profile) |
| `loadTypeConstraintLibrary(registry, library)` | Returns a **new** registry with type constraints merged in (immutable) |
| `getNodeDefinition(registry, typeId)` | Returns the registered `NodeDefinition` or undefined |
| `getTypeConstraint(registry, constraintId)` | Returns the registered `TypeConstraint` or undefined |
| `listNodeDefinitions(registry)` | Returns all registered `NodeDefinition`s |
| `listTypeConstraints(registry)` | Returns all registered `TypeConstraint`s |
| `listLibraries(registry)` | Returns node libraries loaded (in load order) |
| `listGraphLibraries(registry)` | Returns graph libraries loaded (in load order) |
| `listTypeConstraintLibraries(registry)` | Returns type-constraint libraries loaded (in load order) |

`Registry` is a plain/opaque value operated on by functions (not a class hierarchy). Library shapes come from [node-libraries.md](../imp-core-types/node-libraries.md), [graph-libraries.md](../imp-core-types/graph-libraries.md), and [type-constraints.md](../imp-core-types/type-constraints.md).

### Conflict policy

Loading a `NodeDefinition` whose `NodeTypeId` is **already registered** **must throw**. No silent overwrite.

Loading a `TypeConstraint` whose `TypeConstraintId` is **already registered** **must throw**. No silent overwrite.

### Non-goals

- Filesystem or URL loading (parsing lives in `imp-graph-library`; hosts call load after parse)
- Dynamic plugin discovery
- Executable / runtime bindings for types

## Design rationale

- Registry depends on **`imp-core-types` library interfaces**; library packages do **not** depend on the registry.
- Immutable load keeps composition and tests simple.
- Single `definitions` namespace for primitive and graph-backed entries.
- Explicit conflicts avoid accidental type-id collisions across libraries.

## Behavior / pipeline

1. Start from `createRegistry()`.
2. `loadNodeLibrary` for each operator `NodeLibrary`.
3. `loadGraphLibrary` for each parsed graph library JSON.
4. `loadTypeConstraintLibrary` for each constraint library.
5. Resolve graph `Node.type` values via `getNodeDefinition`.
6. Late resolution via `imp-graph-resolve` at compile/execute boundaries.

## Configuration

None.

## See also

- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-libraries.md](../imp-core-types/graph-libraries.md)
- [type-constraints.md](../imp-core-types/type-constraints.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- [graph-model.md](../imp-core-types/graph-model.md)
- Root [AGENTS.md](../../../AGENTS.md)
