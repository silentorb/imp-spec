# Feature registry

Each file under [`../packages/`](../packages/) is the **authoritative design spec** for one Imp package or capability covered by imp-spec.

## Registered features

| Package | Spec |
| --- | --- |
| Core graph model | [graph-model.md](../packages/imp-core-types/graph-model.md) |
| Node libraries (`NodeDefinition` / `NodeLibrary`) | [node-libraries.md](../packages/imp-core-types/node-libraries.md) |
| JSON graph libraries | [graph-libraries.md](../packages/imp-core-types/graph-libraries.md) |
| Type constraints | [type-constraints.md](../packages/imp-core-types/type-constraints.md) |
| Registry (load / lookup type libraries) | [registry.md](../packages/imp-registry/registry.md) |
| Static type checking | [type-system.md](../packages/imp-typecheck/type-system.md) |
| Late nested graph resolution | [resolve.md](../packages/imp-graph-resolve/resolve.md) |
| Collection transforms | [collection-transforms.md](../packages/imp-collection-transforms/collection-transforms.md) |
| Pathing | [pathing.md](../packages/imp-pathing/pathing.md) |
| Semantic paths | [semantic-paths.md](../packages/imp-pathing/semantic-paths.md) |

**Imp Translators and runtime** (SQL, React Flow, execution) are documented in [imp-ts `docs/features/`](https://github.com/silentorb/imp-ts/tree/main/docs/features/).

## Split of concerns

- **Package spec** (`docs/packages/`) — *what* and *why* (requirements, rationale, regen-friendly model).
- **Imp Translators / runtime** (`imp-ts/docs/features/`) — format translation and in-place execution specs.
- **Implementation `AGENTS.md`** (e.g. in `imp-ts/packages/`) — *how to work in the language implementation*.
- **Root `AGENTS.md`** — always-on router; not a feature spec.

See [authoring.md](./authoring.md) for how to add or change specs, and [architecture.md](./architecture.md) for package relationships.
