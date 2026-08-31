# Type system

## Summary

**`imp-typecheck`** provides **static** signal-type checking for Imp graphs: opaque type-parameter unification, edge compatibility, and `GraphType` implementation validation. No runtime type tags on execution rows.

## When to read this

- Implementing or changing type checking in language bindings
- Understanding how polymorphic `NodeType`s and `GraphType`s are validated

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

### instantiateNodeType

Given a `NodeType` with `typeParams` and a list of concrete `typeArgs` (same length, same order), return a new `NodeType` with all `{ param }` references in port types replaced by the corresponding concrete args. **Must throw** on arity mismatch or non-concrete args.

### resolvePortType

For a node instance in a graph, resolve the effective `SignalType` of a port:

1. Load catalog `NodeType` from registry.
2. If `Node.typeArgs` present, `instantiateNodeType` first.
3. For **output** ports, start from catalog port type; for **input** ports with an incoming edge, unify catalog input type with resolved source output type.
4. Apply accumulated substitution from edge constraints across the graph (fixed-point over nodes).

### checkGraph

Validate a `Graph` against a registry (and optional graph-type libraries):

| Check | Behavior |
| --- | --- |
| Node type exists | Every `Node.type` resolves via `getNodeType` |
| Port existence | Every edge references valid ports on source/target node types |
| Edge compatibility | For each edge, unify source output type with target input type |
| Required inputs | Required ports (no default, no edge, no literal) are reported as errors |
| Type-param arity | When `Node.typeArgs` set, length must match `NodeType.typeParams` |
| Unresolved params | After propagation, no free type variables remain on any wired port |
| Type-param bounds | After inference, each resolved concrete type arg must satisfy every `TypeParam.bounds` entry via registered `TypeConstraint` membership — see [type-constraints.md](../imp-core-types/type-constraints.md) |

Returns a list of structured errors (node id, port id, message). Empty list means success.

### checkTypeParamBounds

After edge unification and unresolved-variable checks inside `checkGraph`:

1. For each graph node whose catalog `NodeType` declares `typeParams`, resolve concrete type args via `resolveTypeArgs`.
2. For each `TypeParam` with `bounds`, at the matching arg index:
   - Skip bounds check when the arg is not concrete (unresolved-param errors already reported).
   - `{ "id": "any" }` satisfies all bounds.
   - For each bound id, load `TypeConstraint` from registry; the arg must match at least one `members[]` pattern (`signal_type_matches`, same as dispatch case matching).
3. Missing constraint id in registry → error on the node.
4. No matching member → error (e.g. `type argument "boolean" does not satisfy bound "Scalar"`).

`checkGraph` **must** run this pass; bindings may also expose `checkTypeParamBounds` separately for tests.

### checkGraphImplements

Given a `GraphTypeId` and graph:

1. Load `GraphType` from registry.
2. Find core boundary `input` / `output` nodes in the graph.
3. Match boundary port types against `GraphType` declared inputs/outputs under `GraphType.typeParams`.
4. Report errors for missing boundaries or incompatible types.

v1: one boundary node per `GraphType` port is sufficient when types unify; explicit port-to-node binding maps are future work.

### resolveTypeArgs

For a node instance with `NodeType.typeParams`:

1. When `Node.typeArgs` is set, return those concrete types (after arity validation).
2. Otherwise infer from edge unification: bind each declared param using scoped keys `nodeId::paramId` from the graph substitution.
3. Return `undefined` when any param remains unresolved.

### resolveImplementation

Given a graph node and registry:

1. Load `NodeType.implementation`, or default to `{ "kind": "universal", "id": "<NodeTypeId>" }`.
2. **`universal`** — return `implementation.id`.
3. **`dispatch`** — `resolveTypeArgs`, match `cases[].match` against `typeArgs[paramIndex]` (pattern match; `any` id is wildcard), return first hit, else `default`, else error.

Hosts call `resolveImplementation` after type checking when lowering or executing a node instance.

### Error shape

Bindings should expose a stable error record:

| Field | Type | Role |
| --- | --- | --- |
| `nodeId` | `NodeId` | may — absent for graph-level errors |
| `portId` | `PortId` | may |
| `message` | `string` | must |

## Design rationale

- **Opaque generics** keep v1 minimal — no record/schema types as arguments.
- **Edge-driven inference** avoids requiring `typeArgs` on every polymorphic node.
- **Static only** — `imp-sql` and `imp-execution` may call `checkGraph` before lower/execute but do not need runtime type tags for v1.

## Behavior / pipeline

1. Build registry from `NodeLibrary` (+ optional `GraphTypeLibrary`, `TypeConstraintLibrary`) values.
2. Call `checkGraph(graph, registry)` before transmission-sensitive operations.
3. Optionally call `checkGraphImplements` when validating reusable subgraph contracts.

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-types.md](../imp-core-types/graph-types.md)
- [type-constraints.md](../imp-core-types/type-constraints.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
