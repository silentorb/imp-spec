# Collection transforms

## Summary

A **`NodeLibrary`** of common functional collection combinators (`filter`, `sort`, `limit`, …) and predicate/scalar helpers. Library id `imp.collection.transforms`. Catalog only — no execution. Imp Translators consume this catalog.

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
| `collection` | **Container family** over an element type — not itself ordered or unordered; subtypes carry those properties. Future kinds include `set` and `map` (not implemented here). |
| `list` | Ordered collection of single values; **duplicates allowed**. Subtype of `collection` (`list` <: `collection`). |
| `boolean` | Predicate results |
| `string` | Column names, sort direction, projected column lists |
| `number` | Limits, offsets, numeric literals |
| `any` | Top-type scalar / literal payload; unifies with any signal type in static checking |

Family-level and identity-set transforms declare `typeParams: [{ id: "T" }]` and use `collection<T>` on collection ports. Order/position transforms use `list<T>` (`{ "id": "list", "args": [{ "param": "T" }] }`). Predicate and scalar helper nodes use concrete primitive types only.

Family-level ops (`filter`, `project`, `search`) are polymorphic over the container family. Container-kind preservation and resolving them to concrete subtype implementations are **future work**.

### No table source node

There is **no** `from` / table-source node. The incoming collection arrives via a core boundary **`input`** node (`imp.core`). The host wires the upstream collection; the subgraph is a collection → collection function.

### Node types

#### Transforms

| NodeType | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| `filter` | `collection` (`collection<T>`), `predicate` (`boolean`) | `collection` (`collection<T>`) | Family-level: keep rows where predicate is true |
| `except` | `collection` (`collection<T>`), `exclude` (`collection<T>`) | `collection` (`collection<T>`) | Keep rows from `collection` whose `id` does not appear in `exclude` (set difference by identity). Closer to a future `set` than to generic family semantics; stays on `collection` for now. |
| `sort` | `collection` (`list<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`list<T>`) | `direction` is `"asc"` or `"desc"` |
| `limit` | `collection` (`list<T>`), `count` (`number`) | `collection` (`list<T>`) | Take first `count` rows |
| `offset` | `collection` (`list<T>`), `count` (`number`) | `collection` (`list<T>`) | Skip first `count` rows |
| `project` | `collection` (`collection<T>`), `columns` (`string`) | `collection` (`collection<T>`) | Family-level: `columns` is a comma-separated column name list (v1) |
| `group` | `collection` (`list<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`list<T>`) | Partition rows by `column`; `direction` is `"asc"` or `"desc"`. Rows stay flat; hosts use group order for bands/sections. Translators may emit `ORDER BY` on the group column; partition and enum-weight sort happen after execute. |
| `search` | `collection` (`collection<T>`), `query` (`string`) | `collection` (`collection<T>`) | Family-level declarative text retrieval. Imp defines intent only — ranking, field coverage, fuzzy matching, case rules, empty-query behavior, and preview metadata are **host-defined**. Generic SQL translation does **not** translate `search`; hosts intercept at compile or execute. |

Port **names** such as `collection` on `sort`/`limit` are unchanged; their **signal types** are `list<T>`.

`except` is **declarative** only. Imp Translators must resolve it lazily (e.g. SQL anti-membership over a subquery). They must **not** materialize `exclude` into an in-memory id set and subtract.

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

- Combinators are declarative catalog data so editors and Imp Translators share one vocabulary.
- Host-provided collections (via core `input`) integrate as collection → collection functions without baking table names into Imp.
- Full-word type ids keep graphs readable in UI and docs.

## Behavior / pipeline

This library ships **data only**. Compose with `imp-registry` (`loadLibrary`); Imp Translators and runtime consumers consume the catalog.

Typical pipeline:

1. Core `input` (collection) → `filter` / `except` / `sort` / `group` / `limit` / … → core `output`.
2. Predicates and scalars attach as subgraphs into `filter.predicate` (and similar).
3. `except.exclude` is a second collection-producing subgraph (often sharing the same `input`, e.g. via `traverse`).

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- Root [AGENTS.md](../../../AGENTS.md)
