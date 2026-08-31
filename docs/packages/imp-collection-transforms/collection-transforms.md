# Collection transforms

## Summary

A **`NodeLibrary`** of common functional collection combinators (`filter`, `sort`, `limit`, …) and predicate/scalar helpers. Library id `imp.collection.transforms`. Catalog only — no execution. SQL lowering lives in [sql.md](../imp-sql/sql.md).

## When to read this

- Adding or changing collection transform / predicate node types
- Authoring Imp graphs that transform collections for SQL or other backends

## Requirements

### Library identity

| Field | Value |
| --- | --- |
| Library `id` | `imp.collection.transforms` |
| Depends on | `imp-core-types` only (not `imp-registry`) |

### Naming

Use **full words** for `NodeTypeId`s unless the abbreviation is a widely understood term (`filter`, `sort`, `limit` are fine; use `equals` / `not_equals` / `less_than` / `greater_than`, not `eq` / `neq` / `lt` / `gt`).

### Signal type ids (v1)

| Id | Role |
| --- | --- |
| `collection` | Ordered/unordered row set flowing through transforms |
| `boolean` | Predicate results |
| `string` | Column names, sort direction, projected column lists |
| `number` | Limits, offsets, numeric literals |
| `any` | Top-type scalar / literal payload; unifies with any signal type in static checking |

Collection-preserving transforms declare `typeParams: [{ id: "T" }]` and use `collection<T>` on collection ports (`{ "id": "collection", "args": [{ "param": "T" }] }`). Predicate and scalar helper nodes use concrete primitive types only.

### No table source node

There is **no** `from` / table-source node. The incoming collection arrives via a core boundary **`input`** node (`imp.core`). The host wires the upstream collection; the subgraph is a collection → collection function.

### Node types

#### Transforms

| NodeType | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| `filter` | `collection` (`collection<T>`), `predicate` (`boolean`) | `collection` (`collection<T>`) | Keep rows where predicate is true |
| `except` | `collection` (`collection<T>`), `exclude` (`collection<T>`) | `collection` (`collection<T>`) | Keep rows from `collection` whose `id` does not appear in `exclude` (set difference by identity) |
| `sort` | `collection` (`collection<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`collection<T>`) | `direction` is `"asc"` or `"desc"` |
| `limit` | `collection` (`collection<T>`), `count` (`number`) | `collection` (`collection<T>`) | Take first `count` rows |
| `offset` | `collection` (`collection<T>`), `count` (`number`) | `collection` (`collection<T>`) | Skip first `count` rows |
| `project` | `collection` (`collection<T>`), `columns` (`string`) | `collection` (`collection<T>`) | `columns` is a comma-separated column name list (v1) |
| `group` | `collection` (`collection<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`collection<T>`) | Partition rows by `column`; `direction` is `"asc"` or `"desc"`. Rows stay flat; hosts use group order for bands/sections. SQL lowering is `ORDER BY` on the group column; partition and enum-weight sort happen after execute. |
| `search` | `collection` (`collection<T>`), `query` (`string`) | `collection` (`collection<T>`) | Declarative text retrieval. Imp defines intent only — ranking, field coverage, fuzzy matching, case rules, empty-query behavior, and preview metadata are **host-defined**. Generic `imp-sql` does **not** lower `search`; hosts intercept at compile or execute (see [sql.md](../imp-sql/sql.md)). |

`except` is **declarative** only. Lowerers must resolve it lazily (e.g. SQL anti-membership over a subquery). They must **not** materialize `exclude` into an in-memory id set and subtract.

#### Predicates

| NodeType | Inputs | Outputs |
| --- | --- | --- |
| `contains` | `haystack` (`any`), `needle` (`string`) | `value` (`boolean`) | Precise substring match when both operands coerce to strings; adapters define case rules. **Not** a declarative search — use `search` for open-ended retrieval. |
| `equals` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `not_equals` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `less_than` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `greater_than` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `and` | `left` (`boolean`), `right` (`boolean`) | `value` (`boolean`) |
| `or` | `left` (`boolean`), `right` (`boolean`) | `value` (`boolean`) |
| `not` | `value` (`boolean`) | `value` (`boolean`) |

#### Scalars

| NodeType | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| `column` | `name` (`string`) | `value` (`any`) | Column reference by name |
| `literal` | `value` (`any`) | `value` (`any`) | Instance literal on `inputs.value`; passthrough out |

## Design rationale

- Combinators are declarative catalog data so editors and lowerers share one vocabulary.
- Host-provided collections (via core `input`) integrate as collection → collection functions without baking table names into Imp.
- Full-word type ids keep graphs readable in UI and docs.

## Behavior / pipeline

This library ships **data only**. Compose with `imp-registry` (`loadLibrary`) and lower with `imp-sql` ([sql.md](../imp-sql/sql.md)).

Typical pipeline:

1. Core `input` (collection) → `filter` / `except` / `sort` / `group` / `limit` / … → core `output`.
2. Predicates and scalars attach as subgraphs into `filter.predicate` (and similar).
3. `except.exclude` is a second collection-producing subgraph (often sharing the same `input`, e.g. via `traverse`).

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [sql.md](../imp-sql/sql.md)
- Root [AGENTS.md](../../../AGENTS.md)
