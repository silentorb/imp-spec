# Spike B — Mixed-path intents vs current DAG SQL

> **Historical.** Normative navigate/project bind is [semantic-paths.md](../semantic-paths.md). Path predicates (`EXISTS`) remain deferred.

## Goal

Compare three intents expressible today as Imp DAGs with what a unified opaque path (`A.B.C`) would need.

Compiled with `imp-sql` against a generic schema (`items` / `edges`, `propertiesColumn` on edges).

## Case 1 — Edge then field (`traverse` → `project`)

**DAG:** `input → traverse(knows,0) → project(title) → output`

**SQL (actual):**

```sql
select distinct "title" from (select * from "items") as "sources"
  inner join "edges" as "path_edges"
    on "path_edges"."source_id" = "sources"."id" and "path_edges"."type" = ?
  inner join "items" as "targets"
    on "targets"."id" = "path_edges"."target_id"
```

Parameters: `["knows"]`

**Path sugar:** `edge(knows,0).field(title)` can expand to the same DAG. No ambiguity if steps are tagged.

## Case 2 — Inter-hop filter (`traverse` → `filter` → `traverse`)

**DAG:** `input → traverse(knows,0) → filter(status = active) → traverse(works_at,0) → output`

**SQL (actual):** nested join subqueries with `where "status" = ?` between hops; edge property bags via `json_patch` when projecting full rows.

Parameters: `["knows","active","works_at"]`

**Opaque path `knows.works_at`:** has **nowhere** to put the mid-path predicate. A path IR must either:

- stay **sugar over a DAG** (filters remain separate nodes), or
- allow **predicates on steps** / path segments (richer than a name list)

## Case 3 — Field-only (`filter` + `project`, no pathing)

**SQL (actual):**

```sql
select "title" from "items" where "status" = ?
```

Parameters: `["active"]`

No `schema.edges` required. A “universal path” that always goes through pathing would over-join if misused; field-only work should remain collection-transforms.

## Verdict

Path sugar **expands cleanly** for pure edge chains and edge-then-project. Case 2 **forces paths to stay sugar-only** (or grow step-local predicates). Do **not** replace the DAG with a single opaque path node as the sole authoring model.
