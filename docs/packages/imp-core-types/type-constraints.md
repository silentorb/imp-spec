# Type constraints

## Summary

Declarative **catalogs of nominal type constraints** (`TypeConstraint`) that list which concrete `SignalType` patterns satisfy a named bound. Polymorphic `NodeType` and `GraphType` entries reference constraint ids on `TypeParam.bounds`; `imp-typecheck` enforces them after type-arg inference.

## When to read this

- Defining or changing `TypeConstraint` or `TypeConstraintLibrary`
- Declaring bounds on polymorphic catalog entries (e.g. `bounds: ["Scalar"]`)
- Regenerating constraint interfaces in bindings

## Requirements

### TypeConstraintId

String identity of a named nominal constraint (e.g. `Scalar`). Distinct namespace from `NodeTypeId` and `GraphTypeId` in practice but same underlying string type.

### TypeConstraint

| Field | Type | Required |
| --- | --- | --- |
| `id` | `TypeConstraintId` | must |
| `members` | `SignalType[]` | must (may be empty) |

A **catalog entry** listing concrete type patterns that satisfy the constraint.

- Each `members[]` entry is a **concrete pattern** (nominal `id` + optional `args`), using the same pattern language as implementation dispatch `cases[].match` — see [node-libraries.md](./node-libraries.md).
- `{ "id": "any" }` in a member pattern matches any concrete type when checked (wildcard).
- Membership uses **`signal_type_matches`**: nominal pattern match on resolved concrete type args, including the fixed core subtype relation (a subtype satisfies a supertype pattern; e.g. `list<string>` matches a `collection<string>` member). Not structural record typing. See [type-system.md](../imp-typecheck/type-system.md).

A constraint's `id` must equal its key in the parent `TypeConstraintLibrary.constraints` map when the library is well-formed.

### TypeConstraintLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must |
| `constraints` | map `TypeConstraintId` → `TypeConstraint` | must (may be empty) |

A named, declarative collection of `TypeConstraint`s. Libraries **must not** require `imp-registry`; they implement the core interface alone.

### TypeParam bounds

`TypeParam` on `NodeType` and `GraphType` may declare optional `bounds` — see [graph-model.md](./graph-model.md) and [node-libraries.md](./node-libraries.md).

- Each bound is a `TypeConstraintId` registered via `loadTypeConstraintLibrary`.
- Multiple bounds on one param = **intersection**: the resolved concrete type arg must satisfy **every** listed constraint.
- Omitted `bounds` = unconstrained (current v1 behavior for params without bounds).

### Satisfying a bound

Given a resolved **concrete** type argument and a `TypeConstraintId`:

1. Load `TypeConstraint` from registry.
2. The type arg satisfies the constraint when it **`signal_type_matches`** at least one `members[]` pattern (same rules as dispatch case matching).
3. `{ "id": "any" }` as the resolved type arg satisfies **all** bounds (consistent with unify top-type rules).

### Relationship to other catalogs

| Mechanism | Role |
| --- | --- |
| **Type constraints** | Static rejection: which nominal types may instantiate a type parameter |
| **Implementation dispatch** | Host binding selection after typecheck — see [node-libraries.md](./node-libraries.md) |
| **GraphType** | Subgraph boundary port contracts — see [graph-types.md](./graph-types.md) |

Do not use `GraphType` for nominal membership; use `TypeConstraintLibrary` instead.

## Design rationale

- **Named constraints** let hosts extend membership in one library without editing every polymorphic `NodeType`.
- **Pattern members** reuse dispatch matching so parametric nominal types (e.g. `collection<string>`) can be listed explicitly.
- **Library packaging** matches `NodeLibrary` / `GraphTypeLibrary` — data-only catalogs loaded via registry.

## Behavior / pipeline

This feature is a **data shape** only. Loading, lookup, and `checkGraph` bounds enforcement live in `imp-registry` and `imp-typecheck` — see [registry.md](../imp-registry/registry.md) and [type-system.md](../imp-typecheck/type-system.md).

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md)
- [node-libraries.md](./node-libraries.md)
- [graph-types.md](./graph-types.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [registry.md](../imp-registry/registry.md)
- Root [AGENTS.md](../../../AGENTS.md)
