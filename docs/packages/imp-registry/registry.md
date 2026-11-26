# Registry

## Summary

**`imp-registry`** loads `NodeLibrary` values and [Imp files](../imp-file/imp-files.md) and provides lookup of `NodeDefinition`s by `NodeTypeId` and `TypeDefinition`s by `TypeId`. Callers pass library objects they already have — registry performs no file I/O.

## When to read this

- Implementing or changing registry load/lookup behavior
- Deciding how apps compose multiple type libraries

## Requirements

### API

| Operation | Behavior |
| --- | --- |
| `createRegistry()` | Returns an empty registry |
| `loadNodeLibrary(registry, library)` | Returns a **new** registry with the library's primitive + graph-backed definitions and its `types` merged in (immutable), appending overload signatures per id |
| `loadImpFile(registry, file)` | Same merge for an Imp file's `definitions` (primitive or graph-backed) and `types`. The file's `nodes` / `edges` are not registry content — see [imp-files.md](../imp-file/imp-files.md) |
| `getNodeDefinitions(registry, typeId)` | Returns the registered **overload set** (`NodeDefinition[]`) for `typeId`, or an empty list if unknown |
| `getNodeDefinition(registry, typeId)` | Convenience: returns the sole `NodeDefinition` when the overload set has length **1**; otherwise **undefined** (unknown or overloaded — callers that need a selected signature must use typecheck selection or `getNodeDefinitions`) |
| `getTypeDefinition(registry, typeId)` | Returns the registered `TypeDefinition` or undefined |
| `listNodeDefinitions(registry)` | Returns all registered `NodeDefinition` signatures (flattened overload sets) |
| `listTypeDefinitions(registry)` | Returns all registered `TypeDefinition`s |
| `listLibraries(registry)` | Returns node libraries loaded (in load order) |
| `listImpFiles(registry)` | Returns Imp files loaded (in load order) |

`Registry` is a plain/opaque value operated on by functions (not a class hierarchy). Library shapes come from [node-libraries.md](../imp-core-types/node-libraries.md), [imp-files.md](../imp-file/imp-files.md), and [type-definitions.md](../imp-core-types/type-definitions.md).

Internal storage maps each `NodeTypeId` to an ordered **overload set** (`NodeDefinition[]`). Load order within and across libraries appends to that list.

### Conflict policy

Loading a `NodeDefinition` whose signature is **equivalent** to an already-registered overload for the same `NodeTypeId` **must throw**. No silent overwrite. Distinct signatures under the same id (overloads) are allowed — see [node-libraries.md](../imp-core-types/node-libraries.md).

Loading a `TypeDefinition` whose `TypeId` is **already registered** **must throw**. No silent overwrite. Validity of the loaded definitions (params, arity, cycles) is checked by `imp-typecheck` (`checkTypeDefinitions`), not at load.

### Non-goals

- Filesystem or URL loading (parsing lives in `imp-file`; hosts call load after parse)
- Dynamic plugin discovery
- Executable / runtime bindings for types
- Choosing which overload applies to a graph node (that is `imp-typecheck`)

## Design rationale

- Registry depends on **`imp-core-types` library interfaces**; library packages do **not** depend on the registry.
- Immutable load keeps composition and tests simple.
- Single `definitions` namespace for primitive and graph-backed entries, with overload sets per id.
- Single global `types` namespace: a type defined in any library or Imp file is visible to every graph checked against the registry.
- Equivalent-signature conflicts avoid accidental duplicate overloads; distinct kinds under one symbol are intentional.

## Behavior / pipeline

1. Start from `createRegistry()`.
2. `loadNodeLibrary` for each operator `NodeLibrary`.
3. `loadImpFile` for each parsed Imp file that carries definitions or types.
4. Resolve graph `Node.type` values via `getNodeDefinitions` / typecheck overload selection.
5. Late resolution via `imp-graph-resolve` at compile/execute boundaries.

## Configuration

None.

## See also

- [node-libraries.md](../imp-core-types/node-libraries.md)
- [imp-files.md](../imp-file/imp-files.md)
- [type-definitions.md](../imp-core-types/type-definitions.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- [graph-model.md](../imp-core-types/graph-model.md)
- Root [AGENTS.md](../../../AGENTS.md)
