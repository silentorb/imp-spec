# Authoring Imp specs

## Spec vs code

| Layer | Location | Regenerated? |
| --- | --- | --- |
| Language-neutral specs | `imp-spec/docs/packages/` | **No** — authored source of truth |
| Binding interfaces / implementation | `imp-ts/packages/*/src`, etc. | **Yes** — implement or regenerate from spec |

When the model changes, update the spec first, then update bindings to match.

## Dependency rule

Imp specs describe Imp only. Do **not** mention downstream hosts, applications, or binding-specific libraries (Kysely, `@xyflow/react`, etc.) in requirements — those belong in binding `AGENTS.md` files.

## Adding a new package spec

1. Create `docs/packages/<package>/<name>.md` using the template below.
2. Add a routing row to root [`AGENTS.md`](../../AGENTS.md) and [`overview/README.md`](./README.md).
3. Link from the binding package `AGENTS.md` in the relevant language repo.

## Feature doc template

```markdown
# <Feature name>

## Summary

## When to read this

## Requirements

## Design rationale

## Behavior / pipeline

## Configuration

## See also
```

Binding-specific sections (`Quick start`, `Verification`, `Implementation pointers`) belong in language binding repos, not in imp-spec.

**Imp Translator and runtime specs** (SQL translation, React Flow, dynamic execution) belong in `imp-ts/docs/features/`, not in imp-spec.
