# Universal pathing exploration

## Summary

**Superseded** by the normative [semantic-paths.md](./semantic-paths.md) (navigate / project bind + desugar; identity default). This file is **non-normative history** from the exploration that led to that design. Path-valued predicates (`EXISTS`) remain deferred — see semantic-paths Follow-ups.

Normative catalogs: [pathing.md](./pathing.md), [semantic-paths.md](./semantic-paths.md), [collection-transforms.md](../imp-collection-transforms/collection-transforms.md).

Historical spike notes: [spike-a-naming](./spikes/spike-a-naming.md), [spike-b-sql](./spikes/spike-b-sql.md), [spike-c-catalog](./spikes/spike-c-catalog.md), [spike-d-types](./spikes/spike-d-types.md).

## When to read this

- Background on why untagged ontology-resolved paths were chosen over forever-tagged sugar
- Historical notes on relational path predicates (still deferred)
- Spike A–D lowering constraints (direction, SQL shapes, endpoint kinds)

## Problem domain

### Semantic integrity (deeper than path IR)

Field-vs-edge ambiguity is not only an Imp operator concern. For a design corpus (e.g. Tome), **ontology integrity** means a single semantic namespace: a node type must not expose a **property** and a **relationship** under the same semantic id. If that invariant holds, “is this token a field or an edge?” is answered by the ontology, not by the query author tagging kinds.

Today’s Imp split (`traverse` vs `column`) and packed association ports force authors to know the **technical** shape of the model. That is appropriate for tooling that manages relationships explicitly. It is the wrong default for a **normal query layer** (including Imp graphs used as queries): authors should navigate by semantic ids only.

**Illustrative authoring target:** `node.member` — not `node.parent.id`. Each path token is a **valid endpoint**:

- Relationship token → related node(s); when a node is the endpoint, its **id** is the default scalarization (no extra `.id` step required for identity).
- Property token → that property’s value endpoint.
- Further tokens continue from whatever endpoint the previous token produced.

So universal pathing is: one token stream, ontology-resolved step kinds, every step a first-class endpoint—used for navigate, project, and relational predicate alike.

**Status check (illustrative corpus):** association *storage* ids (ULIDs) do not collide with property keys, but human perspective labels already overlap many property-ish names. Integrity at the **semantic id** layer is therefore a modelling rule to enforce (and migrate), not something the current corpus fully guarantees for author-facing names.

### Path roles (navigate / project / predicate)

In graph query systems, a path is a shared vocabulary for graph position:

| Role | Intent | Typical SQL shape |
| --- | --- | --- |
| **Navigate** | Replace current node collection with hop targets | `JOIN` edges → targets |
| **Project** | Read a field (or edge prop) at the end of a path from the current row | join + select / scalar subquery |
| **Predicate** | Keep **current** rows when related structure matches a condition | correlated `EXISTS` / semi-join / `NOT EXISTS` |
| **Quantify** (often with predicate) | exists / for-all / count along a path | same, with aggregation |

Intrinsic vs extrinsic are different **resolutions of a semantic token** (via ontology)—not two APIs the query author must pick.

Imp v1 covers **navigate** (`traverse`) and **project/predicate on the current row only** (`column` + `filter`). Authors must already know which tokens are associations vs properties. It also lacks path-in-predicate (EXISTS / semi-join) without replacing the outer collection.

### Why “sugar-only” / “always tag kinds in the query” underfit

An early recommendation (“tagged path → DAG desugar”) and Spike A’s “tagged steps required” answered Imp-as-host-agnostic IR without an integrity rule. That understates the product vision:

- If the ontology guarantees a unique semantic id per property-or-relationship, **authoring paths are untagged semantic tokens**; kind is bound at compile/resolve time from the ontology.
- Tags or separate operators may still appear in a lowered Imp DAG or in hosts without that invariant—but they are not the normal query surface.
- Linear desugar still does not invent relational path predicates.

Spikes A–D remain useful for lowering constraints (direction/edge filters, SQL shapes, no runtime try-both). They are not a verdict against ontology-resolved untagged paths.

## Current Imp split (primitives)

| Primitive | Library | Operator | Role |
| --- | --- | --- | --- |
| Extrinsic hop | `imp.pathing` | `traverse` | Change which **node collection** is current (one association + direction) |
| Intrinsic field | `imp.collection.transforms` | `column` (+ `project` / `sort` / `group` / compares) | Scalar field on the **current** row |

Translators lower each node separately (join vs column expression).

### What placement of `filter` can and cannot do

`filter` + `column` apply to the collection wired into that `filter`. Order relative to `traverse` only chooses **which collection is current**:

1. **Filter destinations:** `traverse*` → `filter(column…)` — predicate on reached rows; output is destinations.
2. **Filter origins by own fields:** `filter(column…)` → `traverse*` — predicate on sources, then hop.
3. **Filter intermediates:** `traverse` → `filter` → `traverse`.
4. **Filter hop edges:** `traverse` `edge_property` / `edge_equals` (not node `column`).

**Missing — relational predicate on the outer collection:**

> Keep nodes from *C* when related nodes via path *P* satisfy *Q* (output still *C*).

Example: “features that have an active dependent.”  
`filter → traverse` and `traverse → filter` are both the wrong shape. Workarounds via `except` / inverted subgraphs are brittle.

**Needed:** path as an **argument** to filter (or a dedicated matches-path op): outer collection unchanged; path + condition compile to EXISTS / semi-join.

## Two layers (do not conflate)

| Layer | Meaning | Notes |
| --- | --- | --- |
| **Semantic path (authoring)** | Tokens = ontology ids; each token an endpoint; kind from ontology | Normal query surface; requires semantic integrity |
| **Technical lowering** | Joins vs column exprs; direction; edge filters; EXISTS | Tooling / Translator; may still use distinct ops after resolve |

Push back that still holds: runtime try-edge-then-field without ontology; JSON nested walks as fake relationships; packing direction into author-facing delimited strings; **allowing the same semantic id for a property and a relationship**.

Push back that was **too weak / mis-aimed**: “sugar over traverse/column is enough”; “authors must always tag field vs edge in the query.”

## Ambiguity and constraints (revised)

1. **Kind collision** — Without ontology integrity, untagged tokens are ambiguous (Spike A). **With** integrity, resolve kind from the ontology at bind time. Enforce uniqueness across properties and relationships in the corpus model.
2. **Endpoint rule** — Every path token resolves to an endpoint (node set or scalar). Node endpoints default to **id** when a scalar identity is required; do not require a mandatory `.id` hop for ordinary relational use (`member` not `member.id`).
3. **Cardinality** — Relationship tokens yield node collections (or single node when cardinality is 1); property tokens yield scalars. Continuing a path after a scalar needs an explicit rule (usually end of path).
4. **Direction / edge filters** — Technical; supply via ontology defaults (perspective → endpoint index) or optional qualifiers—not as the primary authoring vocabulary.
5. **Path roles** — Same token stream for navigate, project, and relational predicate (`EXISTS`); role is how the path is *used*, not a different path language.
6. **Performance** — Ontology resolve + specialized lowering (join / column / `EXISTS`) can match hand-built DAGs; cost is wrong if kind is guessed at runtime.

## Spike results (short)

| Spike | Verdict (updated reading) |
| --- | --- |
| A Naming | Untagged tokens need a **resolver**. Ontology integrity makes semantic-id paths safe; without it, collisions (e.g. perspective labels vs property names) prove the modelling rule is load-bearing |
| B SQL | Technical lowering for linear navigate/project works today; relational predicate and endpoint-default-id need new shapes |
| C Catalog | Linear desugar ≠ product design; normal surface is ontology-resolved paths |
| D Types | Endpoint kinds (node vs scalar) still matter after resolve; authoring should not require choosing `traverse` vs `column` up front |

## Recommendation (revised)

**Centre the design on semantic paths + ontology integrity**, not on tagged sugar over today’s operators.

1. **Corpus rule:** within an ontology, semantic ids for properties and relationships are unique (full semantic integrity). Migrate / validate corpora accordingly; author-facing names must not collide.
2. **Query surface:** path tokens are those ids; each token is an endpoint; relationship endpoints are nodes (id by default when identity is needed). Use the same paths for navigate, project, and **relational filter** (EXISTS / semi-join).
3. **Lowering:** Imp Translators / hosts resolve token → property column vs association(+direction) using the ontology, then emit joins / column exprs / `EXISTS`. Distinct `traverse`/`column` may remain as lowered forms or advanced/tooling ops.
4. **Still refuse:** same semantic id for field and relationship; runtime guessing kind without ontology; JSON-as-relationship; pretending `filter`/`traverse` order is enough for relational predicates.

**Next work:** (a) integrity validation + id scheme for author-facing relationship tokens vs storage ULIDs; (b) path-valued predicates with endpoint semantics; (c) bind/resolve pass from semantic path → today’s (or evolved) Imp ops / SQL.

## See also

- [pathing.md](./pathing.md)
- [collection-transforms.md](../imp-collection-transforms/collection-transforms.md)
- [architecture.md](../../overview/architecture.md)
