# AGENTS Guide — Imp (language-neutral specs)

## Repository purpose

**Imp** is a universal transmission format for directed acyclic graphs (DAGs). This repo holds **language-neutral** design specs for Imp capabilities. Language implementations ([TypeScript: `imp-ts`](https://github.com/silentorb/imp-ts), [Rust: `imp-rs`](https://github.com/silentorb/imp-rs)) implement these specs.

Imp succeeds [imp-kotlin](https://github.com/silentorb/imp-kotlin) but keeps only the **graph data layer** (no text/code language layer).

## Scope boundary

imp-spec covers the **graph IR**, **Imp files**, **type definitions**, **NodeLibrary** operator catalogs, **`NodeImplementation` dispatch rules**, registry, typecheck, and graph-resolve. **Imp Translator** specs (format translation — e.g. Imp → SQL, Imp ↔ React Flow) and **runtime** specs (in-place execution) document themselves in [imp-ts `docs/features/`](https://github.com/silentorb/imp-ts/tree/main/docs/features/).

Use **Imp Translator** for packages that translate Imp graphs to other formats (not Adapter, not Resolver, not lowerer).

| Package | Role | Spec |
| --- | --- | --- |
| `imp-core-types` | Core graph + library interfaces; core boundary `NodeLibrary` | [graph-model](./docs/packages/imp-core-types/graph-model.md), [graph-sources](./docs/packages/imp-core-types/graph-sources.md), [node-libraries](./docs/packages/imp-core-types/node-libraries.md), [type-definitions](./docs/packages/imp-core-types/type-definitions.md) |
| `imp-registry` | Load `NodeLibrary` values and Imp files; look up node and type definitions | [registry](./docs/packages/imp-registry/registry.md) |
| `imp-typecheck` | Static type checking for graphs | [type-system](./docs/packages/imp-typecheck/type-system.md) |
| `imp-file` | Parse, validate and serialize Imp files (types, definitions, nodes, edges) | [imp-files](./docs/packages/imp-file/imp-files.md) |
| `imp-graph-resolve` | Late nested graph resolution (flatten / execution program) | [resolve](./docs/packages/imp-graph-resolve/resolve.md) |
| `imp-collection-transforms` | Collection combinator `NodeLibrary` | [collection-transforms](./docs/packages/imp-collection-transforms/collection-transforms.md) |
| `imp-pathing` | GQL-like path operator `NodeLibrary`; semantic path bind/desugar | [pathing](./docs/packages/imp-pathing/pathing.md), [semantic-paths](./docs/packages/imp-pathing/semantic-paths.md) |

Cross-package architecture and authoring rules: [`docs/overview/`](./docs/overview/).

## Specs vs code (regen rule)

Imp is **agent-spec driven**. Treat package specs under `docs/packages/` as the source of truth for the data model.

| Layer | What | Regenerated? |
| --- | --- | --- |
| Specs | `docs/` in this repo | **No** — authored and maintained |
| Code interfaces | Language implementation packages (`imp-ts/packages/*/src`, etc.) | **Yes** — regenerable from the matching spec |

Specs must be language-neutral enough that an agent could emit equivalent interfaces in other languages without redesigning the model.

When docs and code disagree, update the doc or the code explicitly — do not leave them divergent.

## Feature documentation

| If your task involves… | Read |
| --- | --- |
| Core graph model (`Graph`, `Node`, `Edge`, ports, `InputValues`) | [graph-model.md](./docs/packages/imp-core-types/graph-model.md) |
| Reading graphs through edge / node providers (`GraphSource`, local vs remote nodes) | [graph-sources.md](./docs/packages/imp-core-types/graph-sources.md) |
| Node definition libraries (`NodeDefinition`, `NodeLibrary`) | [node-libraries.md](./docs/packages/imp-core-types/node-libraries.md) |
| Imp files (JSON: types, definitions, nodes, edges) | [imp-files.md](./docs/packages/imp-file/imp-files.md) |
| Late nested graph resolution | [resolve.md](./docs/packages/imp-graph-resolve/resolve.md) |
| Type definitions and union types (`TypeDefinition`, named unions, bounds) | [type-definitions.md](./docs/packages/imp-core-types/type-definitions.md) |
| Static type checking | [type-system.md](./docs/packages/imp-typecheck/type-system.md) |
| Registry load / lookup | [registry.md](./docs/packages/imp-registry/registry.md) |
| Collection transform combinators | [collection-transforms.md](./docs/packages/imp-collection-transforms/collection-transforms.md) |
| Path / hop operators | [pathing.md](./docs/packages/imp-pathing/pathing.md) |
| Ontology-resolved semantic paths (bind / desugar) | [semantic-paths.md](./docs/packages/imp-pathing/semantic-paths.md) |
| Imp Translators (SQL, React Flow) or runtime execution | [imp-ts `docs/features`](https://github.com/silentorb/imp-ts/tree/main/docs/features/) |

See [`docs/overview/authoring.md`](./docs/overview/authoring.md) for the feature-doc template.

## Working conventions

- Focused changes only; avoid unrelated refactors.
- **Prototypal stage — no backwards compatibility.** Delete old shapes; migrate consumers in the same change.
- **One-way dependency:** Imp specs must not mention downstream consumers (hosts, applications). Host docs may link here.
- Package ids use `imp-*` naming (`imp.core`, `imp.collection.transforms`, …).
