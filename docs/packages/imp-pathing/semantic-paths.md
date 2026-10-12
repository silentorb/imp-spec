# Semantic paths

## Summary

**Semantic paths** are the normal Imp authoring surface for navigate and project over a host ontology: an untagged token stream. A bind pass resolves each token to a property or relationship, then desugars to the technical catalog (`traverse`, `column` / `project`). Authors do not choose field vs edge operators, and ordinary identity does not require a trailing `.id` hop.

**Relationship tokens** are typically **global relational role ids** (host-defined ULIDs for each half of a relationship type). **Property tokens** remain type-scoped (e.g. column keys). Hosts may also expose type-scoped convenience tokens that resolve to the same relationship hop.

This document is **normative** for the bind/desugar contract. The lowered operators remain specified in [pathing.md](./pathing.md) and [collection-transforms.md](../imp-collection-transforms/collection-transforms.md).

## When to read this

- Implementing or changing ontology-resolved path bind / desugar
- Defining a host `PathOntology` (token → property or relationship; relationship tokens often global role ids)
- Authoring query helpers that emit Imp DAGs without tagged `traverse` / `column` steps

## Requirements

### Two layers

| Layer | Meaning | Surface |
| --- | --- | --- |
| **Semantic path (authoring)** | Untagged tokens = ontology ids; kind from ontology; each token an endpoint | Bind API / query helpers — not a new Translator catalog node in v1 |
| **Technical lowering** | Joins vs column expressions; association + direction; edge filters | Existing `traverse` / `column` / `project` nodes after desugar |

Imp Translators and runtime consumers continue to see the technical DAG. Semantic bind runs **before** translation or execution.

### Ontology integrity

- Each semantic token resolves to **exactly one** of: property, or relationship (not both) in the active resolve context.
- **Property** tokens are type-scoped (e.g. column keys on the current type).
- **Relationship** tokens are preferably **global relational role ids** (unique across the host ontology). Hosts may also accept type-scoped aliases that lower to the same `association` + `direction`. Tokens are **not** free-form display labels unless those labels are also registered as unique semantic ids.
- Hosts **must not** guess kind at runtime (no try-edge-then-field). Ambiguous or unknown tokens **must** fail at bind time.

### PathOntology (host contract)

A host supplies a `PathOntology` with:

| Operation | Behavior |
| --- | --- |
| `resolve(typeId, token)` | Returns a binding for `token`. `typeId` is the current type context (required for properties; relationship role ids may ignore it). Structured error if unknown / ambiguous. |

Binding shapes:

| Kind | Fields | Endpoint | Next type context |
| --- | --- | --- | --- |
| **property** | property name (logical column id) | Scalar value | None — path must end (no further tokens) |
| **relationship** | `association` (string), `direction` (`0` \| `1`), optional edge filter ports | Node collection (or singleton when cardinality is 1) | `nextType` — type id of related nodes for further tokens (optional when the hop is terminal / opposite endpoint is open) |

- `association` and `direction` match [pathing.md](./pathing.md) `traverse` ports (separate values; no packed delimited string in the Imp DAG). Hosts typically derive them from a relational role id.
- `nextType` is required for relationship bindings when further path tokens may follow; bind **must** error if a subsequent token is present and `nextType` is absent.

### Resolve pass

`resolveSemanticPath(tokens, ontology, startType) → resolved steps`

1. Start with current type context = `startType`.
2. For each token in order:
   - If the current endpoint is scalar (previous step was a property), **error** (no context-descent / no continuing after a scalar).
   - Call `ontology.resolve(currentType, token)`.
   - Property → emit a field step; set current endpoint to scalar.
   - Relationship → emit an edge step (`association`, `direction`, optional edge filters); set current type context to `nextType`.
3. Resolved steps are ordered: zero or more leading edge steps, then zero or more trailing field steps. Edge after field is impossible if this algorithm is followed; desugar still rejects that shape if constructed manually.

### Endpoint and identity default

- Every resolved token is a **first-class endpoint**: relationship → nodes; property → scalar.
- When a **node** endpoint is used where a **scalar identity** is required, bind/desugar **must** treat the endpoint as the node’s `id` without requiring an author-written `id` token (e.g. path ending on `member` scalarizes to `member`’s `id`, not `member.id` in the authoring path).
- Explicit trailing `id` as a property token remains valid when the ontology exposes `id` as a property; hosts may treat promoted identity the same way.
- Continuing a path after a scalar endpoint is **illegal**.

### Desugar pass

`desugarPath(resolvedSteps, options) →` Imp graph fragment (`traverse` / `project` nodes + edges)

Rules (v1, linear paths only):

- Leading edge steps → chained `traverse` nodes (same ports as [pathing.md](./pathing.md)).
- Trailing field steps → one `project` with comma-separated column names (collection-transforms), or equivalent `column` wiring when a single scalar expression is required by the caller.
- When the caller requests **scalar identity** and the path ends on a node endpoint (no trailing fields), desugar **must** project `id`.
- Field then edge → **error** (no context-descent).
- Empty path → passthrough of the source collection port (no new nodes), unless scalar identity is requested (then project `id` on the source).

No new catalog `path` / `step` node type is required in v1. Editors and hosts may call resolve + desugar and splice the fragment into a graph.

### Path roles (this milestone)

| Role | Status |
| --- | --- |
| **Navigate** | Supported — edge-only paths replace the current node collection |
| **Project** | Supported — trailing fields (and identity default) on the reached collection |
| **Predicate / EXISTS** | **Deferred** — keep outer collection when related structure matches a condition (correlated `EXISTS` / semi-join). Same token stream is intended; not part of v1 bind/desugar. See Follow-ups. |

Intrinsic vs extrinsic remain different **resolutions** of a semantic token, not two authoring APIs.

### Refusals

Implementations **must not**:

- Allow the same semantic id for a property and a relationship in one type context
- Guess kind at runtime without ontology
- Treat nested JSON walks as relationships
- Pack association + direction into one author-facing delimited path token
- Claim that `filter` / `traverse` ordering alone covers relational path predicates on the outer collection

## Design rationale

- Normal queries should navigate by semantic ids; tooling that manages relationships explicitly can still author `traverse` / `column` directly.
- Keeping the catalog IR on today’s operators preserves existing Translators and hosts while the authoring layer gains ontology-resolved paths.
- Global relational role ids make relationship hops addressable without packing type + direction into the authoring token; property tokens stay type-scoped.

## Behavior / pipeline

1. Author (or editor) supplies untagged tokens + start type + host `PathOntology`.
2. `resolveSemanticPath` → tagged edge/field steps with association/direction bound.
3. `desugarPath` → `traverse` / `project` fragment; optional scalar-identity → `id`.
4. Merge fragment into Imp graph; `flattenGraph` if needed; Translator or runtime consumes the technical DAG.

## Configuration

None beyond the host-supplied `PathOntology` and desugar options (`prefix`, source port, scalar-identity flag).

## Follow-ups

- **Path-valued predicates:** path as a filter argument (or matches-path op) compiling to `EXISTS` / semi-join without replacing the outer collection.
- Hosts wiring `PathOntology` from their schema files and query UIs.
- Broader host surfaces switching from relationship-type + endpoint references to relational role ids (filters, table schemas, etc.).

## See also

- [pathing.md](./pathing.md) — technical `traverse` catalog
- [collection-transforms.md](../imp-collection-transforms/collection-transforms.md) — `column` / `project` / `filter`
- [universal-pathing-exploration.md](./universal-pathing-exploration.md) — non-normative history (superseded by this doc)
- Root [AGENTS.md](../../../AGENTS.md)
