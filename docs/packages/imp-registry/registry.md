# Registry

## Summary

**`imp-registry`** loads `NodeLibrary` values and provides lookup of `NodeType`s by `NodeTypeId`. Callers pass library objects they already have — no I/O or dynamic import in v1.

## When to read this

- Implementing or changing registry load/lookup behavior
- Deciding how apps compose multiple type libraries

## Requirements

### API

| Operation | Behavior |
| --- | --- |
| `createRegistry()` | Returns an empty registry |
| `loadLibrary(registry, library)` | Returns a **new** registry with the library's types merged in (immutable) |
| `getNodeType(registry, typeId)` | Returns the registered `NodeType` or undefined |
| `listNodeTypes(registry)` | Returns all registered `NodeType`s |
| `listLibraries(registry)` | Returns the libraries that were loaded (in load order) |

`Registry` is a plain/opaque value operated on by functions (not a class hierarchy). Library shapes come from [node-libraries.md](../imp-core-types/node-libraries.md).

### Conflict policy

Loading a `NodeType` whose `NodeTypeId` is **already registered** **must throw**. No silent overwrite.

### Non-goals (v1)

- Filesystem or URL loading
- Dynamic plugin discovery
- Executable / runtime bindings for types

## Design rationale

- Registry depends on **`imp-core-types` library interfaces**; library packages do **not** depend on the registry.
- Immutable `loadLibrary` keeps composition and tests simple.
- Explicit conflicts avoid accidental type-id collisions across libraries.

## Behavior / pipeline

1. Start from `createRegistry()`.
2. `loadLibrary` for each `NodeLibrary` (throws on duplicate `NodeTypeId`).
3. Resolve graph `Node.type` values via `getNodeType`.

## Configuration

None.

## See also

- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-model.md](../imp-core-types/graph-model.md)
- Root [AGENTS.md](../../../AGENTS.md)
