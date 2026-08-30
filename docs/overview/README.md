# Feature registry

Each file under [`../packages/`](../packages/) is the **authoritative design spec** for one Imp package or capability.

## Registered features

| Package | Spec |
| --- | --- |
| Core graph model | [graph-model.md](../packages/imp-core-types/graph-model.md) |
| Node libraries (`NodeType` / `NodeLibrary`) | [node-libraries.md](../packages/imp-core-types/node-libraries.md) |
| Registry (load / lookup type libraries) | [registry.md](../packages/imp-registry/registry.md) |
| React Flow integration | [react-flow.md](../packages/imp-react-flow/react-flow.md) |
| Collection transforms | [collection-transforms.md](../packages/imp-collection-transforms/collection-transforms.md) |
| Pathing | [pathing.md](../packages/imp-pathing/pathing.md) |
| SQL lowering | [sql.md](../packages/imp-sql/sql.md) |
| Dynamic execution | [execution.md](../packages/imp-execution/execution.md) |

## Split of concerns

- **Package spec** (`docs/packages/`) — *what* and *why* (requirements, rationale, regen-friendly model).
- **Binding `AGENTS.md`** (e.g. in `imp-ts/packages/`) — *how to work in the language binding*.
- **Root `AGENTS.md`** — always-on router; not a feature spec.

See [authoring.md](./authoring.md) for how to add or change specs, and [architecture.md](./architecture.md) for package relationships.
