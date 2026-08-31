# AGENTS Guide — Imp (language-neutral specs)

## Repository purpose

**Imp** is a universal transmission format for directed acyclic graphs (DAGs). This repo holds **language-neutral** design specs for Imp capabilities. Language bindings (TypeScript: [`imp-ts`](../imp-ts), Rust: [`imp-rust`](../imp-rust)) implement these specs.

Imp succeeds [imp-kotlin](https://github.com/silentorb/imp-kotlin) but keeps only the **graph data layer** (no text/code language layer).

| Package | Role | Spec |
| --- | --- | --- |
| `imp-core-types` | Core graph + library interfaces; core boundary `NodeLibrary` | [graph-model](./docs/packages/imp-core-types/graph-model.md), [node-libraries](./docs/packages/imp-core-types/node-libraries.md), [graph-types](./docs/packages/imp-core-types/graph-types.md) |
| `imp-registry` | Load `NodeLibrary` / `GraphTypeLibrary` values and look up types | [registry](./docs/packages/imp-registry/registry.md) |
| `imp-typecheck` | Static signal-type checking for graphs | [type-system](./docs/packages/imp-typecheck/type-system.md) |
| `imp-react-flow` | Imp ↔ React Flow converters | [react-flow](./docs/packages/imp-react-flow/react-flow.md) |
| `imp-collection-transforms` | Collection combinator `NodeLibrary` | [collection-transforms](./docs/packages/imp-collection-transforms/collection-transforms.md) |
| `imp-pathing` | GQL-like path operator `NodeLibrary` | [pathing](./docs/packages/imp-pathing/pathing.md) |
| `imp-sql` | Imp collection graphs → SQL | [sql](./docs/packages/imp-sql/sql.md) |
| `imp-execution` | Dynamic runtime for collection/path graphs (read-only host) | [execution](./docs/packages/imp-execution/execution.md) |

Cross-package architecture and authoring rules: [`docs/overview/`](./docs/overview/).

## Specs vs code (regen rule)

Imp is **agent-spec driven**. Treat package specs under `docs/packages/` as the source of truth for the data model.

| Layer | What | Regenerated? |
| --- | --- | --- |
| Specs | `docs/` in this repo | **No** — authored and maintained |
| Code interfaces | Language binding packages (`imp-ts/packages/*/src`, etc.) | **Yes** — regenerable from the matching spec |

Specs must be language-neutral enough that an agent could emit equivalent interfaces in other languages without redesigning the model.

When docs and code disagree, update the doc or the code explicitly — do not leave them divergent.

## Feature documentation

| If your task involves… | Read |
| --- | --- |
| Core graph model (`Graph`, `Node`, `Edge`, ports, `InputValues`) | [graph-model.md](./docs/packages/imp-core-types/graph-model.md) |
| Node type libraries (`NodeType`, `NodeLibrary`) | [node-libraries.md](./docs/packages/imp-core-types/node-libraries.md) |
| Graph type libraries (`GraphType`, `GraphTypeLibrary`) | [graph-types.md](./docs/packages/imp-core-types/graph-types.md) |
| Static type checking | [type-system.md](./docs/packages/imp-typecheck/type-system.md) |
| Registry load / lookup | [registry.md](./docs/packages/imp-registry/registry.md) |
| React Flow integration / converters | [react-flow.md](./docs/packages/imp-react-flow/react-flow.md) |
| Collection transform combinators | [collection-transforms.md](./docs/packages/imp-collection-transforms/collection-transforms.md) |
| Path / hop operators | [pathing.md](./docs/packages/imp-pathing/pathing.md) |
| Imp → SQL lowering | [sql.md](./docs/packages/imp-sql/sql.md) |
| Dynamic execution | [execution.md](./docs/packages/imp-execution/execution.md) |

See [`docs/overview/authoring.md`](./docs/overview/authoring.md) for the feature-doc template.

## Working conventions

- Focused changes only; avoid unrelated refactors.
- **Prototypal stage — no backwards compatibility.** Delete old shapes; migrate consumers in the same change.
- **One-way dependency:** Imp specs must not mention downstream consumers (hosts, applications). Host docs may link here.
- Package ids use `imp-*` naming (`imp.core`, `imp.collection.transforms`, …).

## Workbench integration

In **silentorb-workbench**, this repo mounts at `.mnt/imp-spec/` (optional; host default `../imp-spec`, or `IMP_SPEC_REPO`). Bindings: TypeScript `.mnt/imp-ts/`, Rust `.mnt/imp-rust/` (optional; `IMP_RUST_REPO`). See [architecture bindings table](./docs/overview/architecture.md#language-bindings).
