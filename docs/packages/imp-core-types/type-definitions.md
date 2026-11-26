# Type definitions

## Summary

A **type definition** gives a name to a type expression: `name = type`. Definitions are data: they ship in node libraries and [Imp files](../imp-file/imp-files.md), load into the registry's single type namespace, and are not tied to any graph. A defined name is a **transparent alias**: checking treats it exactly as the type it names. The motivating case is naming a union, such as `Audio` = `audio_mono | audio_stereo`.

## When to read this

- Defining or changing `TypeDefinition`
- Declaring named unions or generic aliases for a library
- Writing bounds on type parameters (`bounds` name types, usually defined unions)
- Regenerating type-definition interfaces in bindings

## Requirements

### TypeDefinition

| Field | Type | Required |
| --- | --- | --- |
| `id` | `TypeId` | must |
| `params` | `TypeParam[]` | may |
| `type` | `Type` | must |

- `id` is the defined name. References use the ordinary concrete form `{ "id": "<id>", "args": [...] }`.
- `params` are the definition's own type parameters. Inside `type`, `{ "param": "<id>" }` refers to them. A reference must pass exactly one argument per param. `params[].bounds` are ignored in this revision.
- `type` is any `Type`: concrete (possibly naming other definitions), a reference to one of the definition's own params, or a union.

Examples:

```json
{ "id": "Audio", "type": { "union": [{ "id": "audio_mono" }, { "id": "audio_stereo" }] } }
```

```json
{
  "id": "Pair",
  "params": [{ "id": "T" }],
  "type": { "id": "list", "args": [{ "param": "T" }] }
}
```

### Transparency

A defined name and its expansion are the **same type**. `Audio` is interchangeable with `audio_mono | audio_stereo` everywhere: ports, `Node.typeArgs`, bounds, dispatch patterns, and inside other types and definitions. Before comparing types, `imp-typecheck` **normalizes** them (see [type-system.md](../imp-typecheck/type-system.md#normalization)).

A concrete `id` with **no** definition stays an **opaque nominal type**, as before (open world). Loading definitions never makes undeclared names invalid.

### Validity

A set of loaded definitions is valid when, for every definition:

1. Every `{ "param": P }` inside `type` names one of the definition's own `params`.
2. Every reference to a defined name passes exactly as many `args` as that definition has `params`.
3. Every union has at least one member.
4. Expansion terminates: no definition refers to itself, directly or through other definitions, including through `args` or union members. Recursive types are out of scope.

`checkTypeDefinitions` reports violations — see [type-system.md](../imp-typecheck/type-system.md#checktypedefinitions).

### Where definitions live

| Carrier | Field |
| --- | --- |
| `NodeLibrary` (code-defined catalogs) | `types` — see [node-libraries.md](./node-libraries.md) |
| Imp file | `types` section — see [imp-files.md](../imp-file/imp-files.md) |

All carriers load into the registry's **one global type namespace**. Loading a `TypeDefinition` whose `id` is already registered **must throw** — see [registry.md](../imp-registry/registry.md).

Definitions must not depend on `imp-registry`. They are plain data.

## Design rationale

- **Transparent aliases** keep the type system's identity rules unchanged: two types are equal or assignable exactly when their expansions are.
- **Named unions replace type constraints.** A constraint listed member patterns that could fill a parameter. A named union says the same thing as a type, which can also type a port or nest inside another type, and which bounds check with the same assignability rule as edges.
- **Structured JSON** matches the `Type` shape used everywhere else. There is no separate expression grammar.
- **One global namespace** matches node definitions and keeps lookup independent of where a definition was written.
- **No recursion** keeps normalization a terminating expansion.

## Behavior / pipeline

1. A host builds `NodeLibrary` values and parses Imp files.
2. `loadNodeLibrary` / `loadImpFile` merge their `types` into the registry, throwing on duplicate ids.
3. `checkGraphDefinitions` runs `checkTypeDefinitions` before checking graph-backed definitions.
4. `checkGraph` normalizes every type it reads before unification, assignability, bounds and dispatch.

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md) — `Type`, including the union variant
- [node-libraries.md](./node-libraries.md)
- [imp-files.md](../imp-file/imp-files.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
