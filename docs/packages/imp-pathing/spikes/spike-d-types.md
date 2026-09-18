# Spike D — Mid-path type rules (sketch)

> **Historical.** Endpoint kinds (node vs scalar) and no context-descent are normative in [semantic-paths.md](../semantic-paths.md).

## Domains

| Domain | Signal | Typical operators |
| --- | --- | --- |
| **Collection** | `collection<T>` (node rows with `id`) | `traverse`, `filter`, `sort`, `project`, `group`, `limit`, `offset`, `except`, `search` |
| **Scalar** | `any` / primitives / `boolean` | `column`, `literal`, `parameter`, comparisons, boolean combinators |

## Step effects (tagged dual-kind, no context-descent)

| Step kind | Input domain | Output domain | Notes |
| --- | --- | --- | --- |
| `edge` | Collection | Collection | Same as `traverse`. Carries `association`, `direction` (0\|1), optional `edge_property` / `edge_equals`. |
| `field` (in projection / sort / group / predicate) | Collection context (row) | Scalar | Same as `column`. Does **not** change the collection flowing through transforms. |
| `field` as a collection-path step | — | **Illegal** mid-path before another `edge` | Without nested-object context descent, a field does not yield a new node collection. |

## Legal sequences

- `edge*` — zero or more edge hops; result remains `collection<T>`.
- `edge* then project(field+)` — hops then intrinsic projection (DAG: `traverse*` → `project`).
- Intrinsic-only — no edge steps; `column` / `filter` / `project` as today.

## Illegal sequences (v1, no context-descent)

- `field` then `edge` — scalar / nested value is not a node collection source.
- `edge` after a scalar sink — cannot wire `traverse.collection` from `column.value`.
- Untagged name used as both association and property — rejected (see Spike A).
- Relationship hops inside `project`/`sort`/`group` column-name strings — those ports stay **flat field names**, not mixed paths.

## Where extrinsic-only ports live

On **edge** steps (or `traverse` nodes), not on field steps:

- `direction`: `0` \| `1`
- `edge_property` + `edge_equals`: both null or both set

## Verdict

A “universal path” is honest for Imp’s type system only as **authoring sugar** over tagged steps that still desugar into today’s collection vs scalar split. Full semantic unification (field hops changing focus like edges) is a **different** model and is out of scope for v1.
