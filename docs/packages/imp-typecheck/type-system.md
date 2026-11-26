# Type system

## Summary

**`imp-typecheck`** provides **static** type checking for Imp graphs: type-definition expansion, union types, opaque type-parameter unification, nominal subtyping, edge assignability, **per-symbol overload resolution**, graph-backed definition validation, and definition-reference cycle detection. No runtime type tags on execution rows.

## When to read this

- Implementing or changing type checking in language implementations
- Understanding how polymorphic `NodeDefinition`s are validated
- Understanding how union types and [type definitions](../imp-core-types/type-definitions.md) are checked

## Requirements

### Normalization

Every type read from the registry or a graph (port templates, `Node.typeArgs`, bounds, dispatch patterns) is **normalized** before any rule below applies. `normalizeType(type, registry)`:

1. **Variable** — unchanged.
2. **Concrete whose `id` names a registered `TypeDefinition`** — replace it by the definition's `type` with each `{ param }` of the definition substituted by the matching argument, then normalize the result.
3. **Other concrete** — normalize each argument.
4. **Union** — normalize each member, then:
   - flatten members that are themselves unions;
   - if any member is `any`, the result is `any`;
   - drop duplicate members (structural equality), keeping the first occurrence;
   - a union left with one member becomes that member.

Normalization terminates because valid definition sets are non-recursive (see [checkTypeDefinitions](#checktypedefinitions)). An argument-count mismatch on a defined name, or a reference to an undefined param, is an error.

Normalized types never contain defined names, nested unions or one-member unions. The rules below assume normalized inputs.

### Substitution

A **substitution** maps `TypeParamId` → `Type` (concrete or union, normalized). Substitutions are extended during unification.

### Unification rules

`unify(a, b, subst)` attempts to make two `Type` values equal under `subst`:

| Case | Rule |
| --- | --- |
| `any` on either side | Succeed; result is the other type |
| Both concrete, same `id`, same `args` length | Recursively unify each arg pair |
| Both concrete, same `id`, different `args` length | **Error** |
| Both concrete, different `id` | **Error** |
| One variable `{ param: T }`, one concrete or union | Bind `subst[T] = other` if `T` unbound; else unify `subst[T]` with the other |
| Both variables | Bind one to the other (pick canonical param id) |
| Both unions | Succeed when each side is **assignable** to the other (member order does not matter), threading `subst` |
| One union, one concrete | **Error** (normalization already collapsed one-member unions) |

Type variables must not appear in substitution values as variables — only concrete types.

### Subtyping

Imp has a **fixed core subtype relation** on concrete type ids (not a host-extensible catalog in this revision):

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

`assignable(source, target, subst)` succeeds when a value of type `source` may flow into a port of type `target`. Apply `subst` to both sides, then take the first rule that applies:

1. **`any`** on either side — as `unify` (succeed; type variables on the other side bind to `any`).
2. **Union source** `S₁ | … | Sₙ` — a type-variable target binds to the **whole** union; otherwise succeed when **every** member `Sᵢ` is assignable to `target`, threading `subst` member by member. A union-typed output cannot feed a port that accepts only some of its members.
3. **Rigid source** (see [Generic graph-backed definitions](#generic-graph-backed-definitions)) — succeed when `target` is the same rigid type, a type variable (which binds to the rigid type, as in `unify`), or when one of the rigid parameter's own **bounds** is assignable to `target`. Otherwise **error**.
4. **Variable source, union target** — an unbound source variable binds to the **whole** union (`T := A | B`).
5. **Union target** `T₁ | … | Tₙ` (source not a union, variable or rigid) — try each member with a trial copy of `subst`:
   - no member accepts `source` → **error**;
   - every accepting member yields the **same** substitution → succeed with it (in particular, a fully resolved source just needs one accepting member);
   - accepting members yield **different** substitutions → **error**: the source is ambiguous between members (e.g. `list<T>` into `list<string> | list<number>` with `T` unbound). Authors resolve it with explicit `typeArgs`.
6. Attempt `unify(source, target, subst)` (covers type variables and same concrete id with invariant args).
7. If unify fails solely because both sides are concrete with **different ids**, succeed when `is_subtype_id(source.id, target.id)` **and** corresponding `args` unify pairwise (same arity; invariant).
8. Otherwise **error** (including the reverse direction: `collection` is not assignable to `list` or `set`).

Wired **edges** use assignability (directional). Other checks that still need equality of two types (e.g. some boundary contracts) may continue to use `unify` directly.

#### Weak edges: into `any` or a union

An edge **target** typed `any` or a union is a **weak** constraint during inference: it must not decide a type variable that another edge could decide more precisely.

- Inference (the fixed point under Overload resolution and resolvePortType) first stabilizes using every edge whose target port type is neither `any` nor a union.
- Only then are weak edges applied. A source type variable still unbound at that point binds to `any` (for an `any` target) or to the whole union (for a union target). If that changes the substitution, the fixed point runs again.
- The result is independent of edge iteration order. Example: `mix<A>(a: A, b: A) → A` fed `audio_mono` and `audio_stereo` reports the conflict even when `mix`'s output also feeds an `any` or `Audio` port.

#### Type-parameter conflicts

When an edge fails because its target port's type parameter is already bound through another inbound edge of the same node, the error names the parameter, both types and both ports — e.g. `type parameter A is audio_mono (from input a) but input b receives audio_stereo` — rather than reporting only the later edge as an assignment failure.

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

Every other `Port` field (`id`, `defaultValue`, **`variadic`**) is preserved; only the port type is substituted.

### resolvePortType

For a node instance in a graph, resolve the effective `Type` of a port:

1. Load the **selected** catalog `NodeDefinition` for the node (sole overload, or the overload chosen by overload resolution).
2. If `Node.typeArgs` present, instantiate port templates first.
3. For **output** ports, start from catalog port type; for **input** ports with an incoming edge, check that the resolved source output type is **assignable** to the catalog input type (see Assignability).
4. Apply accumulated substitution from edge constraints across the graph (**fixed-point** over edges / overload selection until substitution and selections stabilize or a round makes no progress).

Composite (graph-backed) nodes type-check against **declared catalog ports only** — bodies are not inlined during `checkGraph`.

### checkGraph

Validate a [graph source](../imp-core-types/graph-sources.md) against a registry — `checkGraph(source, registry)`; callers holding a serialized `Graph` pass `graphSource(graph)`:

| Check | Behavior |
| --- | --- |
| Definition exists | Every `Node.type` resolves to a non-empty overload set via registry lookup |
| Overload selection | Each overloaded node selects exactly one signature (see Overload resolution); ambiguity or no match → error |
| Endpoint resolution | Every edge `from.node` / `to.node` must **resolve** through the source's node provider (`getNode`). Unresolved → error. Where a node is stored does not matter; edges without a local node entry remain valid in the data shape |
| Port existence | Every edge whose endpoints resolve references valid ports on source/target **selected** definitions |
| Edge compatibility | For each edge with resolved endpoints, source output type must be **assignable** to target input type under the fixed-point substitution (for variadic targets, each inbound edge’s source must be assignable to the **element** `Port.type`) |
| Required inputs | Required singleton ports (no default, no edge, no literal) are errors. Required variadic ports need **≥1** inbound edge |
| Singleton multiplicity | More than one inbound edge on a non-variadic input → error |
| Variadic `order` | Every edge targeting a variadic input must set `order`; `order` values must be **unique** among edges targeting the same `(node, port)` |
| Type-param arity | When `Node.typeArgs` set, length must match the **selected** `NodeDefinition.typeParams` |
| Unresolved params | After propagation, no free type variables remain on any wired port |
| Type-param bounds | After inference, each resolved type arg must be **assignable** to every `TypeParam.bounds` entry |

Returns a list of structured errors (node id, port id, message). Empty list means success.

Every resolved node — whichever provider supplied it — is checked the same way: its port templates come from the registry via `Node.type`, with overload selection. Node-level checks (definition exists, required inputs, type-param arity, bounds) cover the source's **resolved members** — see Membership in [graph-sources.md](../imp-core-types/graph-sources.md).

### checkDefinition

Validate a single graph-backed `NodeDefinition`:

1. `body` and `bindings` present; `implementation` absent
2. Every declared input/output port has a binding entry
3. Bound nodes exist in `body` with correct core types (`input` / `output`)
4. Bound boundary port types match declared port types (after applying definition `typeParams`):
   - **Non-variadic** inputs/outputs: unify boundary `value` type with the declared port type
   - **Variadic** inputs: boundary `value` type must be **`list<T>`** where `T` is the declared element `Port.type` (unify / assignability against `listOf(declared.type)`)
5. No unbound boundary nodes beyond those listed in bindings (surplus boundary nodes → error)

#### Generic graph-backed definitions

A graph-backed definition may declare `typeParams` (with `bounds`) and use them in its declared port types, e.g. `echo<A: Audio>(audio: A) → A`. Its body is checked **once, for every instantiation**, by treating each parameter as **rigid**:

1. Each `TypeParam` becomes a rigid type carrying its declared `bounds`. It unifies only with itself, with type variables (which bind to it) and with `any`. No concrete or union type is assignable to it. It is assignable to another type only as described under [Assignability](#assignability) (rule 3): for example `A: Audio` may feed a port typed `Audio`, but not one typed `audio_mono`.
2. A rigid type **satisfies a bound** `C` when it is assignable to `C`, i.e. when one of its own bounds is assignable to `C` (`A: Audio` satisfies `Audio`, and any bound that `Audio` is assignable to).
3. Bound boundary nodes without `typeArgs` whose declared port type mentions a type parameter take the declared type (rigid; `list<…>` for variadic inputs) as their `typeArgs` for the body check. Boundary nodes may also state `typeArgs` that reference the enclosing definition's parameters (`{ "param": "A" }`); these are the only `{ param }` references allowed in body `typeArgs`.
4. The body is then checked with `checkGraph` as usual. Rigid types are not variables, so "Unresolved params" does not fire for them.

Rigid types are a checking device. Implementations may represent them as they like, but they never appear in serialized graphs or in use-site type args, and errors print the parameter id (e.g. `A`).

Use sites type-check against the declared ports like any other polymorphic node (inference from edges, or explicit `Node.typeArgs`). See [resolve.md](../imp-graph-resolve/resolve.md) for how bodies are inlined.

### checkGraphDefinitions

Validate all graph-backed entries in a registry:

1. Run `checkTypeDefinitions` on the registry's type definitions (reported first; normalization of later checks depends on them)
2. Run `checkDefinition` on each graph-backed definition
3. Run `checkGraph` on each `body` (against registry including primitives)
4. Detect **definition-reference cycles**: if definition A's body contains a node whose `type` resolves to graph-backed B, and B transitively references A → error

When listing graph-backed definitions for cycle detection, treat each overload signature that is graph-backed independently (same `id` may appear more than once only for primitive overloads in this revision; graph-backed overloads are allowed by the data model but uncommon).

### checkTypeDefinitions

Validate every registered `TypeDefinition` (see [type-definitions.md](../imp-core-types/type-definitions.md#validity)):

| Check | Behavior |
| --- | --- |
| Params | Every `{ param }` inside `type` names one of the definition's own `params` |
| Arity | Every reference to a defined name passes exactly as many `args` as that definition has `params` |
| Empty unions | Every union has at least one member |
| Cycles | No definition refers to itself, directly or through other definitions (including via `args` and union members) |

Errors set `typeId` to the offending definition. Arity checks also apply to defined names used in port templates, `Node.typeArgs` and bounds; those are reported where the type is read.

### checkTypeParamBounds

After edge unification and unresolved-variable checks inside `checkGraph`:

1. For each graph node whose **selected** catalog `NodeDefinition` declares `typeParams`, resolve type args via `resolveTypeArgs` (normalized).
2. For each `TypeParam` with `bounds`, at the matching arg index, check that the arg is **assignable** to each normalized bound with an empty substitution. `any` satisfies every bound. A union arg satisfies a bound only when every member does.

`checkGraph` **must** run this pass.

### resolveTypeArgs

For a node instance with selected `NodeDefinition.typeParams`:

1. When `Node.typeArgs` is set, return those types, normalized (after arity validation).
2. Otherwise infer from edge unification: bind each declared param using scoped keys `nodeId::paramId`.
3. Return `undefined` when any param remains unresolved.

Resolved type args are always **normalized**: defined names are expanded, so hosts see `audio_mono | audio_stereo`, never `Audio`.

### resolveImplementation

Given a graph node and registry (primitive definitions only):

1. Load the **selected** `NodeDefinition.implementation`, or default to `{ "kind": "universal", "id": "<NodeDefinition.id>" }`.
2. **`universal`** — return `implementation.id` (kind-specific overloads typically use distinct ids such as `filter_list` / `filter_set`).
3. **`dispatch`** — `resolveTypeArgs`, then return the first case whose normalized `cases[].match` pattern the type arg is **assignable** to (`{ "id": "any" }` matches everything), else `default`, else error.

`implementation.dispatch` remains a mechanism **behind one selected signature**. Per-kind polymorphism for authors is expressed as **overload sets**, not as a single `collection` signature with dispatch alone.

Graph-backed definitions have no `implementation`; hosts resolve them via `imp-graph-resolve`.

### Hosts and union types

Imp values carry **no runtime type tags**. A host that lowers or executes a graph decides representations from **statically resolved, normalized** types (`resolveTypeArgs`, port templates passed through `normalizeType`). A port whose resolved type is still a **union** carries a value of one member without saying which, so a host that cannot handle every member at runtime **must reject** that port with a clear error rather than guess. Unions are most useful today as names for bounds (`A: Audio`), where each use site resolves to one member.

### Error shape

| Field | Type | Role |
| --- | --- | --- |
| `nodeId` | `NodeId` | may — absent for graph-level errors |
| `portId` | `PortId` | may |
| `definitionId` | `NodeTypeId` | may — for catalog-level errors |
| `typeId` | `TypeId` | may — for type-definition errors |
| `message` | `string` | must |

## Design rationale

- **Opaque generics** keep the model minimal.
- **Fixed nominal subtypes** let specialized containers flow into family-typed ports when those still exist (e.g. `traverse` on `collection`).
- **Per-symbol overloads** give kind-specific definitions (`list→list`, `set→set`) under one graph `Node.type` without a fake general `collection` filter.
- **Edge-driven inference** and **fixed-point overload selection** avoid requiring `typeArgs` on every polymorphic node and let sinks (e.g. `limit`) refine upstream overload choice.
- **Strict bindings** replace loose boundary matching.
- **Definition cycle detection** catches library authoring errors before runtime resolution.
- **Transparent type definitions** add names without changing identity: comparing expansions keeps unification and subtyping unchanged.
- **Unions replace type constraints.** Bounds, dispatch patterns and edges all use assignability, so a named union serves as a bound, a pattern and a port type with one rule.
- **Weak union targets** mirror `any`: a union port accepts several types, so it should not be the edge that decides an inferred parameter.

## Behavior / pipeline

1. Build registry from `NodeLibrary` values and [Imp files](../imp-file/imp-files.md) (node definitions and type definitions).
2. `checkGraphDefinitions(registry)` after loading (type definitions first, then graph-backed definitions).
3. `checkGraph(graph, registry)` before transmission-sensitive operations.

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [imp-files.md](../imp-file/imp-files.md)
- [type-definitions.md](../imp-core-types/type-definitions.md)
- [registry.md](../imp-registry/registry.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- Root [AGENTS.md](../../../AGENTS.md)
