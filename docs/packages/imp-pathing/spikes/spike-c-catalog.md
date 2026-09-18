# Spike C — Catalog shape (C3 desugar)

> **Historical.** Normative design is [semantic-paths.md](../semantic-paths.md): ontology-resolved tokens + desugar helper (no new Translator node). Public API lives in `imp-ts` `imp-pathing` (`resolveSemanticPath` / `desugarPath` / `bindSemanticPath`).

## Choice

Prototype **C3 only**: keep normative `traverse` / `column` / `project`; add a **tagged path → DAG desugar** helper. No new Translator node type.

Skipped for this pass: C1 (`path` node with step list in the catalog), C2 (single `step` operator with `kind` ports).

## Artifact

Language binding (throwaway / parkable):

- `imp-ts/packages/imp-pathing/src/spikes/desugar-path.ts`
- `imp-ts/packages/imp-pathing/src/spikes/desugar-path.test.ts`

Not exported from the package public entry. Expands:

- leading `kind: "edge"` → chained `traverse`
- trailing `kind: "field"` → one `project`
- `field` then `edge` → throw (no context-descent)

Round-trip check: desugared `edge+field` graph compiles under `imp-sql` like a hand-built DAG.

## Verdict

**Prefer C3** if authoring sugar is wanted later. Do not add C1/C2 until sugar proves insufficient for editors.
