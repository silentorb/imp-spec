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
| `collection` | **Container family** over an element type — not itself ordered or unordered; subtypes carry those properties. Future kinds include `map` (not implemented here). Not a filterable catalog interface by itself. |
| `list` | Ordered collection of single values; **duplicates allowed**. Subtype of `collection` (`list` <: `collection`). |
| `set` | Unordered collection of single values; **duplicates disallowed** (row identity / `id` for Imp rows). Subtype of `collection` (`set` <: `collection`). |
| `boolean` | Predicate results |
| `string` | Column names, sort direction, projected column lists |
| `number` | Limits, offsets, numeric literals |
| `any` | Top-type scalar / literal payload; unifies with any signal type in static checking |

### Overloads (kind-specific)

`filter`, `project`, `search`, and `except` are **overloaded symbols**: the library ships **two** `NodeDefinition` entries per id — one for `list<T>` and one for `set<T>` — with uniform shapes (`Kind<T> → Kind<T>`). There is **no** general `collection<T>` version.

Typecheck selects the overload from wired edge types — see [type-system.md](../imp-typecheck/type-system.md). Each overload uses a distinct universal **implementation id** (`filter_list` / `filter_set`, etc.) so hosts can bind kind-specific lowering.

Order/position transforms (`sort`, `limit`, `offset`, `group`) remain **list-only** (single definition each). Predicate and scalar helper nodes use concrete primitive types only.

### No table source node

There is **no** `from` / table-source node. The incoming collection arrives via a core boundary **`input`** node (`imp.core`). The host wires the upstream collection; the subgraph is a collection → collection function.

### Node types

#### Transforms

| NodeType | Overloads | Inputs | Outputs | Implementation ids | Notes |
| --- | --- | --- | --- | --- | --- |
| `filter` | `list`, `set` | `collection` (`list<T>` or `set<T>`), `predicate` (`boolean`) | `collection` (same kind) | `filter_list`, `filter_set` | Keep rows where predicate is true |
| `except` | `list`, `set` | `collection` (kind), `exclude` (same kind) | `collection` (same kind) | `except_list`, `except_set` | Keep rows from `collection` whose `id` does not appear in `exclude`. List overload preserves encounter order of the keep side. |
| `sort` | `list` only | `collection` (`list<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`list<T>`) | `sort` | `direction` is `"asc"` or `"desc"` |
| `limit` | `list` only | `collection` (`list<T>`), `count` (`number`) | `collection` (`list<T>`) | `limit` | Take first `count` rows |
| `offset` | `list` only | `collection` (`list<T>`), `count` (`number`) | `collection` (`list<T>`) | `offset` | Skip first `count` rows |
| `project` | `list`, `set` | `collection` (kind), `columns` (`string`) | `collection` (same kind) | `project_list`, `project_set` | `columns` is a comma-separated column name list (v1) |
| `group` | `list` only | `collection` (`list<T>`), `column` (`string`), `direction` (`string`, default `"asc"`) | `collection` (`list<T>`) | `group` | Partition rows by `column`; `direction` is `"asc"` or `"desc"`. Rows stay flat; hosts use group order for bands/sections. Translators may emit `ORDER BY` on the group column; partition and enum-weight sort happen after execute. |
| `search` | `list`, `set` | `collection` (kind), `query` (`string`) | `collection` (same kind) | `search_list`, `search_set` | Declarative text retrieval. Imp defines intent only — ranking, field coverage, fuzzy matching, case rules, empty-query behavior, and preview metadata are **host-defined**. Generic SQL translation does **not** translate `search`; hosts intercept at compile or execute. |

Port **names** such as `collection` on transforms are unchanged; **signal types** carry the concrete kind.

`except` is **declarative** only. Imp Translators must resolve it lazily (e.g. SQL anti-membership over a subquery). They must **not** materialize `exclude` into an in-memory id set and subtract.

#### Predicates

| NodeType | Inputs | Outputs |
| --- | --- | --- |
| `contains` | `haystack` (`any`), `needle` (`string`) | `value` (`boolean`) | Precise substring match when both operands coerce to strings; adapters define case rules. **Not** a declarative search — use `search` for open-ended retrieval. |
| `equals` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `not_equals` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `less_than` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `greater_than` | `left` (`any`), `right` (`any`) | `value` (`boolean`) |
| `and` | `operands` (`boolean`, **variadic**) | `value` (`boolean`) | N-ary conjunction; inbound edges must set unique `Edge.order`; fold in ascending `order`. Unary (≥1 operand) is the identity. |
| `or` | `operands` (`boolean`, **variadic**) | `value` (`boolean`) | N-ary disjunction; same order rules as `and`. |
| `not` | `value` (`boolean`) | `value` (`boolean`) |

#### Scalars

| NodeType | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| `column` | `name` (`string`) | `value` (`any`) | Column reference by name |
| `literal` | `value` (`any`) | `value` (`any`) | Instance literal on `inputs.value`; passthrough out |

## Design rationale

- Combinators are declarative catalog data so editors and Imp Translators share one vocabulary.
- Kind-specific overloads under one symbol avoid a fake general `collection` filter while preserving `list → filter → limit` when the list overload is selected.
- Host-provided collections (via core `input`) integrate as collection → collection functions without baking table names into Imp.
- Full-word type ids keep graphs readable in UI and docs.

## Behavior / pipeline

This library ships **data only**. Compose with `imp-registry` (`loadLibrary`); Imp Translators and runtime consumers consume the catalog. Hosts should resolve kind-specific **implementation ids** (via `resolveImplementation` after typecheck) when lowering or evaluating.

Typical pipeline:

1. Core `input` (`list` or `set`) → `filter` / `except` / `sort` / `group` / `limit` / … → core `output`.
2. Predicates and scalars attach as subgraphs into `filter.predicate` (and similar).
3. `except.exclude` is a second collection-producing subgraph (often sharing the same `input`, e.g. via `traverse`).

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [type-system.md](../imp-typecheck/type-system.md)
- Root [AGENTS.md](../../../AGENTS.md)
