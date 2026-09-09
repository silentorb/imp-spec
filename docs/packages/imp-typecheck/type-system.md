# Type system

## Summary

**`imp-typecheck`** provides **static** signal-type checking for Imp graphs: opaque type-parameter unification, edge compatibility, graph-backed definition validation, and definition-reference cycle detection. No runtime type tags on execution rows.

## When to read this

- Implementing or changing type checking in language implementations
- Understanding how polymorphic `NodeDefinition`s are validated

## Requirements

### Substitution

A **substitution** maps `TypeParamId` → concrete `SignalType` (concrete variant only). Substitutions are extended during unification.

### Unification rules

`unify(a, b, subst)` attempts to make two `SignalType` values equal under `subst`:

| Case | Rule |
| --- | --- |
| `any` on either side | Succeed; result is the other type |
| Both concrete, same `id`, same `args` length | Recursively unify each arg pair |
| Both concrete, same `id`, different `args` length | **Error** |
| Both concrete, different `id` | **Error** |
| One variable `{ param: T }`, one concrete | Bind `subst[T] = concrete` if `T` unbound; else unify `subst[T]` with concrete |
| Both variables | Bind one to the other (pick canonical param id) |

Type variables must not appear in substitution values as variables — only concrete types.

### instantiateNodeDefinition

Given a `NodeDefinition` with `typeParams` and a list of concrete `typeArgs` (same length, same order), return a new definition-shaped port template object with all `{ param }` references in port types and `body` replaced by the corresponding concrete args. **Must throw** on arity mismatch or non-concrete args.

Also applies to renaming/substituting into embedded `body` graphs during resolution.

### resolvePortType

For a node instance in a graph, resolve the effective `SignalType` of a port:

1. Load catalog `NodeDefinition` from registry.
2. If `Node.typeArgs` present, instantiate port templates first.
3. For **output** ports, start from catalog port type; for **input** ports with an incoming edge, unify catalog input type with resolved source output type.
4. Apply accumulated substitution from edge constraints across the graph (fixed-point over nodes).

Composite (graph-backed) nodes type-check against **declared catalog ports only** — bodies are not inlined during `checkGraph`.

### checkGraph

Validate a `Graph` against a registry:

| Check | Behavior |
| --- | --- |
| Definition exists | Every `Node.type` resolves via `getNodeDefinition` |
| Port existence | Every edge references valid ports on source/target definitions |
| Edge compatibility | For each edge, unify source output type with target input type |
| Required inputs | Required ports (no default, no edge, no literal) are reported as errors |
| Type-param arity | When `Node.typeArgs` set, length must match `NodeDefinition.typeParams` |
| Unresolved params | After propagation, no free type variables remain on any wired port |
| Type-param bounds | After inference, each resolved concrete type arg must satisfy every `TypeParam.bounds` entry |

Returns a list of structured errors (node id, port id, message). Empty list means success.

### checkDefinition

Validate a single graph-backed `NodeDefinition`:

1. `body` and `bindings` present; `implementation` absent
2. Every declared input/output port has a binding entry
3. Bound nodes exist in `body` with correct core types (`input` / `output`)
4. Bound boundary port types unify with declared port types (after applying definition `typeParams`)
5. No unbound boundary nodes beyond those listed in bindings (surplus boundary nodes → error)

### checkGraphDefinitions

Validate all graph-backed entries in a registry:

1. Run `checkDefinition` on each graph-backed definition
2. Run `checkGraph` on each `body` (against registry including primitives)
3. Detect **definition-reference cycles**: if definition A's body contains a node whose `type` resolves to graph-backed B, and B transitively references A → error

### checkTypeParamBounds

After edge unification and unresolved-variable checks inside `checkGraph`:

1. For each graph node whose catalog `NodeDefinition` declares `typeParams`, resolve concrete type args via `resolveTypeArgs`.
2. For each `TypeParam` with `bounds`, at the matching arg index, verify constraint membership.

`checkGraph` **must** run this pass.

### resolveTypeArgs

For a node instance with `NodeDefinition.typeParams`:

1. When `Node.typeArgs` is set, return those concrete types (after arity validation).
2. Otherwise infer from edge unification: bind each declared param using scoped keys `nodeId::paramId`.
3. Return `undefined` when any param remains unresolved.

### resolveImplementation

Given a graph node and registry (primitive definitions only):

1. Load `NodeDefinition.implementation`, or default to `{ "kind": "universal", "id": "<NodeDefinition.id>" }`.
2. **`universal`** — return `implementation.id`.
3. **`dispatch`** — `resolveTypeArgs`, match `cases[].match`, return first hit, else `default`, else error.

Graph-backed definitions have no `implementation`; hosts resolve them via `imp-graph-resolve`.

### Error shape

| Field | Type | Role |
| --- | --- | --- |
| `nodeId` | `NodeId` | may — absent for graph-level errors |
| `portId` | `PortId` | may |
| `definitionId` | `NodeTypeId` | may — for catalog-level errors |
| `message` | `string` | must |

## Design rationale

- **Opaque generics** keep the model minimal.
- **Edge-driven inference** avoids requiring `typeArgs` on every polymorphic node.
- **Strict bindings** replace loose boundary matching.
- **Definition cycle detection** catches library authoring errors before runtime resolution.

## Behavior / pipeline

1. Build registry from `NodeLibrary`, `GraphLibrary`, and `TypeConstraintLibrary` values.
2. `checkGraphDefinitions(registry)` when loading graph libraries.
3. `checkGraph(graph, registry)` before transmission-sensitive operations.

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-libraries.md](../imp-core-types/graph-libraries.md)
- [type-constraints.md](../imp-core-types/type-constraints.md)
- [registry.md](../imp-registry/registry.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- Root [AGENTS.md](../../../AGENTS.md)
