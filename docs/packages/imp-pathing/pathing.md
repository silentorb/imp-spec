# Pathing

## Summary

A **`NodeLibrary`** of GQL-like graph path operators (`traverse`, …) over a host-supplied node collection and edges relation. Library id `imp.pathing`. Catalog only — no execution. Imp Translators consume this catalog.

## When to read this

- Adding or changing path / hop node types
- Authoring Imp graphs that walk host graph edges for translation or runtime evaluation

## Requirements

### Library identity

| Field | Value |
| --- | --- |
| Library `id` | `imp.pathing` |
| Depends on | `imp-core-types` only (not `imp-registry`) |

### Naming

Use **full words** for `NodeTypeId`s unless the abbreviation is a widely understood term (`traverse` is fine).

### Type ids (v1)

| Id | Role |
| --- | --- |
| `collection` | **Container family** over an element type (typically host nodes). Same meaning as in [collection-transforms.md](../imp-collection-transforms/collection-transforms.md). |
| `list` | Ordered collection of single values; duplicates allowed; subtype of `collection`. |
| `set` | Unordered collection; duplicates disallowed; subtype of `collection`. |
| `string` | Association ids and other string literals |
| `number` | Direction and other numeric literals |

Reuse the same type ids as [collection-transforms.md](../imp-collection-transforms/collection-transforms.md) so path results compose with `filter` / `sort` / `project`. `traverse` declares `typeParams: [{ id: "T" }]` and uses `collection<T>` on collection input and output ports. Its distinct-by-`id` emission is closer to `set` than to `list`; **this revision keeps `traverse` on `collection`** (set-shaped result / list·set overloads are follow-up). Note that `filter` / `project` / etc. no longer accept bare `collection` — authors should bound path pipelines with concrete `list` or `set` where those transforms are used, or await traverse overloads.

### No table source node

There is **no** `from` / table-source node. The incoming node collection arrives via a core boundary **`input`** node or upstream transforms. Edge storage is host-defined at translation time.

### Node types

| NodeType | Inputs | Outputs | Notes |
| --- | --- | --- | --- |
| `traverse` | `collection` (`collection<T>`), `association` (`string`), `direction` (`number`, default `0`), `edge_property` (`string`, default `null`), `edge_equals` (`any`, default `null`) | `collection` (`collection<T>`) | One hop: for each source row identity (`id`), follow host edges where `source = id` and `type = hostEdgeType(association, direction)`, emit distinct target nodes as the new collection. When the host edges relation has a JSON `propertiesColumn`, overlay hop-edge properties onto each target's `properties` so downstream `project` / `group` can read them. When both `edge_property` and `edge_equals` are non-null, also require the hop edge's JSON property named by `edge_property` to equal `edge_equals`. Hosts may map `edge_equals` through schema hooks at SQL translation time. |

- Chained hops = multiple `traverse` nodes in the Imp DAG (not recursive / variable-length paths in v1).
- `association` and `direction` are **separate** Imp values. Graphs must not pack them into one delimited string.
- `direction` is `0` or `1` (endpoint index on a two-ended association).
- Hosts map `(association, direction)` onto their edges `type` column at the translation boundary.
- Optional edge property filter requires a host edges relation with a properties column; `edge_property` must be a simple identifier when set.
- If exactly one of `edge_property` / `edge_equals` is non-null, translation **must** throw.
- Result remains a **node collection**, so collection transforms keep working.

## Design rationale

- Path operators are declarative catalog data so editors and Imp Translators share one vocabulary.
- Keeping the catalog format-agnostic lets other backends interpret the same graphs.
- Explicit `association` + `direction` ports keep Imp graphs readable without host-specific string tokenization; hosts may still encode a packed type string at the SQL boundary if their storage requires it.

## Behavior / pipeline

This library ships **data only**. Compose with `imp-registry` (`loadLibrary`); Imp Translators and runtime consumers consume the catalog.

Typical pipeline:

1. Core `input` (node collection) → `traverse` → `filter` / `project` → core `output`.
2. Multiple `traverse` nodes chain single hops.

## Configuration

None.

## See also

- [semantic-paths.md](./semantic-paths.md) — ontology-resolved authoring paths (bind/desugar onto `traverse` / `column` / `project`)
- [collection-transforms.md](../imp-collection-transforms/collection-transforms.md)
- [graph-model.md](../imp-core-types/graph-model.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- Root [AGENTS.md](../../../AGENTS.md)
