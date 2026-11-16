# Type system

## Summary

**`imp-typecheck`** provides **static** signal-type checking for Imp graphs: opaque type-parameter unification, nominal subtyping, edge assignability, **per-symbol overload resolution**, graph-backed definition validation, and definition-reference cycle detection. No runtime type tags on execution rows.

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

### Subtyping

Imp has a **fixed core subtype relation** on concrete signal type ids (not a host-extensible catalog in this revision):

| Subtype | Supertype |
| --- | --- |
| `list` | `collection` |
| `set` | `collection` |

Rules:

- Relation is **nominal on the outer id**.
- Type arguments are **invariant**: `list<T> <: collection<U>` (and likewise for `set`) only when `T` and `U` unify under the usual rules (including `any`).
- Every type is a subtype of itself via ordinary same-id unification.
- `any` remains the existing top-type escape in unify / bounds; it is not redefined as a lattice parent of `collection`.
- `list` and `set` are **incomparable** with each other.
- Future container kinds (`map`, …) may extend this table later; they are not part of this revision.

`is_subtype_id(sub, super)` is true when `sub === super` or `sub` is a declared subtype of `super` (directly; no deep hierarchy yet beyond the table above).

### Assignability

`assignable(source, target, subst)` succeeds when a value of type `source` may flow into a port of type `target`:

1. Attempt `unify(source, target, subst)` (covers `any`, type variables, and same concrete id with invariant args).
2. If unify fails solely because both sides are concrete with **different ids**, succeed when `is_subtype_id(source.id, target.id)` **and** corresponding `args` unify pairwise (same arity; invariant).
3. Otherwise **error** (including the reverse direction: `collection` is not assignable to `list` or `set`).

Wired **edges** use assignability (directional). Other checks that still need equality of two types (e.g. some boundary contracts) may continue to use `unify` directly.

### Overload resolution

A `NodeTypeId` may map to an **overload set**: an ordered list of `NodeDefinition` signatures that share the same `id` (the graph `Node.type` symbol). See [node-libraries.md](../imp-core-types/node-libraries.md) and [registry.md](../imp-registry/registry.md).

For each graph node whose type has more than one registered signature, typecheck **selects** exactly one:

1. For each candidate signature, open a **trial** substitution cloned from the current graph substitution.
2. Using that candidate’s port templates (scoped to the node id; instantiate if `Node.typeArgs` present), attempt `assignable` for every wired edge that touches the node — both **incoming** (neighbor output → this input) and **outgoing** (this output → neighbor input). Neighbor ports use their **already selected** (or sole) definitions when available; otherwise use each neighbor’s sole definition or skip edges to still-unresolved overloaded neighbors for this trial round.
3. A candidate **survives** when every tried edge assignability succeeds.
4. After considering all candidates for the node:
   - **Exactly one** survivor → select it; merge its trial substitution into the graph substitution.
   - **Zero** survivors → error (no matching overload).
   - **Two or more** survivors → leave the node unresolved for this round (or, at fixed-point exhaustion, error as ambiguous — e.g. when `any` makes both `list` and `set` overloads succeed). Authors must tighten types (boundary `typeArgs`, etc.).

Overload resolution participates in the same **fixed-point** as substitution accumulation (see below). Downstream sinks constrain upstream overloads: e.g. `filter → limit` eliminates a `set→set` `filter` overload because `set` is not assignable to `list`.

`collection` is a **family** type only. There is no general `collection→collection` filter (etc.); concrete kinds use per-kind overloads (`list→list`, `set→set`).

### instantiateNodeDefinition

Given a `NodeDefinition` with `typeParams` and a list of concrete `typeArgs` (same length, same order), return a new definition-shaped port template object with all `{ param }` references in port types and `body` replaced by the corresponding concrete args. **Must throw** on arity mismatch or non-concrete args.

Also applies to renaming/substituting into embedded `body` graphs during resolution.

### resolvePortType

For a node instance in a graph, resolve the effective `SignalType` of a port:

1. Load the **selected** catalog `NodeDefinition` for the node (sole overload, or the overload chosen by overload resolution).
2. If `Node.typeArgs` present, instantiate port templates first.
3. For **output** ports, start from catalog port type; for **input** ports with an incoming edge, check that the resolved source output type is **assignable** to the catalog input type (see Assignability).
4. Apply accumulated substitution from edge constraints across the graph (**fixed-point** over edges / overload selection until substitution and selections stabilize or a round makes no progress).

Composite (graph-backed) nodes type-check against **declared catalog ports only** — bodies are not inlined during `checkGraph`.

### checkGraph

Validate a `Graph` against a registry:

| Check | Behavior |
| --- | --- |
| Definition exists | Every `Node.type` resolves to a non-empty overload set via registry lookup |
| Overload selection | Each overloaded node selects exactly one signature (see Overload resolution); ambiguity or no match → error |
| Endpoint resolution | Every edge `from.node` / `to.node` must resolve (today: present in `Graph.nodes`); unresolved → error. Edges without a local node entry are allowed in the data shape but fail this check |
| Port existence | Every edge whose endpoints resolve references valid ports on source/target **selected** definitions |
| Edge compatibility | For each edge with resolved endpoints, source output type must be **assignable** to target input type under the fixed-point substitution (for variadic targets, each inbound edge’s source must be assignable to the **element** `Port.type`) |
| Required inputs | Required singleton ports (no default, no edge, no literal) are errors. Required variadic ports need **≥1** inbound edge |
| Singleton multiplicity | More than one inbound edge on a non-variadic input → error |
| Variadic `order` | Every edge targeting a variadic input must set `order`; `order` values must be **unique** among edges targeting the same `(node, port)` |
| Type-param arity | When `Node.typeArgs` set, length must match the **selected** `NodeDefinition.typeParams` |
| Unresolved params | After propagation, no free type variables remain on any wired port |
| Type-param bounds | After inference, each resolved concrete type arg must satisfy every `TypeParam.bounds` entry |

Returns a list of structured errors (node id, port id, message). Empty list means success.

### checkDefinition

Validate a single graph-backed `NodeDefinition`:

1. `body` and `bindings` present; `implementation` absent
2. Every declared input/output port has a binding entry
3. Bound nodes exist in `body` with correct core types (`input` / `output`)
4. Bound boundary port types match declared port types (after applying definition `typeParams`):
   - **Non-variadic** inputs/outputs: unify boundary `value` type with the declared port type
   - **Variadic** inputs: boundary `value` type must be **`list<T>`** where `T` is the declared element `Port.type` (unify / assignability against `listOf(declared.type)`)
5. No unbound boundary nodes beyond those listed in bindings (surplus boundary nodes → error)

### checkGraphDefinitions

Validate all graph-backed entries in a registry:

1. Run `checkDefinition` on each graph-backed definition
2. Run `checkGraph` on each `body` (against registry including primitives)
3. Detect **definition-reference cycles**: if definition A's body contains a node whose `type` resolves to graph-backed B, and B transitively references A → error

When listing graph-backed definitions for cycle detection, treat each overload signature that is graph-backed independently (same `id` may appear more than once only for primitive overloads in this revision; graph-backed overloads are allowed by the data model but uncommon).

### checkTypeParamBounds

After edge unification and unresolved-variable checks inside `checkGraph`:

1. For each graph node whose **selected** catalog `NodeDefinition` declares `typeParams`, resolve concrete type args via `resolveTypeArgs`.
2. For each `TypeParam` with `bounds`, at the matching arg index, verify constraint membership.

`checkGraph` **must** run this pass.

### resolveTypeArgs

For a node instance with selected `NodeDefinition.typeParams`:

1. When `Node.typeArgs` is set, return those concrete types (after arity validation).
2. Otherwise infer from edge unification: bind each declared param using scoped keys `nodeId::paramId`.
3. Return `undefined` when any param remains unresolved.

### resolveImplementation

Given a graph node and registry (primitive definitions only):

1. Load the **selected** `NodeDefinition.implementation`, or default to `{ "kind": "universal", "id": "<NodeDefinition.id>" }`.
2. **`universal`** — return `implementation.id` (kind-specific overloads typically use distinct ids such as `filter_list` / `filter_set`).
3. **`dispatch`** — `resolveTypeArgs`, match `cases[].match`, return first hit, else `default`, else error.

`implementation.dispatch` remains a mechanism **behind one selected signature**. Per-kind polymorphism for authors is expressed as **overload sets**, not as a single `collection` signature with dispatch alone.

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
- **Fixed nominal subtypes** let specialized containers flow into family-typed ports when those still exist (e.g. `traverse` on `collection`).
- **Per-symbol overloads** give kind-specific definitions (`list→list`, `set→set`) under one graph `Node.type` without a fake general `collection` filter.
- **Edge-driven inference** and **fixed-point overload selection** avoid requiring `typeArgs` on every polymorphic node and let sinks (e.g. `limit`) refine upstream overload choice.
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
