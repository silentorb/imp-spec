# Graph sources

## Summary

Imp algorithms read graphs through a **graph source**: a pair of small, read-only providers — one for **edges**, one for **nodes**. Whether a node lives in the same document, another document, or an external store is the provider's concern. Code that consumes a graph source (type checking, resolution, execution, editors) never distinguishes local from remote nodes.

The serialized [`Graph`](./graph-model.md#graph) is one implementation of both providers.

## When to read this

- Writing an algorithm that walks a graph
- Supplying nodes from more than one place (multi-document or store-backed graphs)
- Implementing a provider in a language binding

## Requirements

### NodeProvider

| Member | Signature | Required |
| --- | --- | --- |
| `getNode` | `(id: NodeId) → Node \| absent` | must |
| `nodeIds` | `() → NodeId[]` | may |

- `getNode` returns the `Node` for an id, or absent when this provider cannot supply it.
- `nodeIds` lists the ids the provider can **enumerate** cheaply (e.g. a local document). Providers backed by large or external stores omit it.
- A returned `Node` is an ordinary node instance. Its port templates come from the registry via `Node.type`, exactly as for any other node — providers do **not** supply definitions.
- A provider over an external store may derive `Node.type` from that store's own type model (e.g. the id of a type node the record is a member of). The registry must then hold a definition with that id for the node to type-check.

### EdgeProvider

| Member | Signature | Required |
| --- | --- | --- |
| `getEdge` | `(id: EdgeId) → Edge \| absent` | must |
| `edgeEntries` | `() → [EdgeId, Edge][]` | must |

Edges are the graph's topology, so an edge provider is always enumerable. Derived indexes (inbound edges per `(node, port)`, adjacency) are built by shared helpers over `edgeEntries`, not added to the interface.

### GraphSource

| Field | Type | Required |
| --- | --- | --- |
| `nodes` | `NodeProvider` | must |
| `edges` | `EdgeProvider` | must |

The bundle that graph algorithms accept.

### Membership

The **node ids of a graph source**, in stable order, are:

1. every `from.node` / `to.node` across `edgeEntries()` (in entry order, first occurrence), then
2. any `nodes.nodeIds()` not already listed.

The second term keeps unwired nodes (e.g. freshly added in an editor) part of the graph while topology stays edge-first.

A member id is **resolved** when `nodes.getNode(id)` returns a node, otherwise **unresolved**. Unresolved endpoints are validation errors for type checking and execution; they are not a data-shape error.

### Standard constructors and helpers

| Name | Behavior |
| --- | --- |
| `graphSource(graph)` | Graph source over a serialized `Graph`: `getNode` / `nodeIds` read `Graph.nodes`; `getEdge` / `edgeEntries` read `Graph.edges` (key order) |
| `layeredNodes(...providers)` | `NodeProvider` that returns the first hit across `providers` in order. `nodeIds` is the de-duplicated union of providers that implement `nodeIds` (omitted when none do) |
| `nodeMapProvider(nodes)` / `edgeMapProvider(edges)` | Providers over a bare `Graph.nodes` / `Graph.edges` map (building blocks for `graphSource` and layering) |
| `graphNodeIds(source)` | Membership list (above) |
| `resolvedNodes(source)` | Resolved member nodes, in membership order |
| `unresolvedNodeIds(source)` | Member ids with no node from the provider |
| `inboundEdges(source, nodeId, portId)` | `[EdgeId, Edge][]` targeting `(nodeId, portId)`, sorted by `Edge.order` ascending (unset order sorts as `0`, ties by entry order) |
| `materializeGraph(source)` | Serialized `Graph` with every **resolved** member in `nodes` and every edge in `edges`. Unresolved endpoints keep their edges and get no node entry |

### Synchronous snapshots

Providers are **synchronous** and describe a snapshot. Fetching, caching, and change notification for remote nodes belong to the host: a host that resolves nodes lazily re-runs consumers (or re-renders) when new nodes arrive. Imp algorithms stay pure functions of their inputs.

### Consumers

- [`checkGraph`](../imp-typecheck/type-system.md#checkgraph) takes a graph source.
- [`flattenGraph` / `buildExecutionProgram`](../imp-graph-resolve/resolve.md) take a graph source and return serialized graphs.
- Graph-backed definition bodies (`NodeDefinition.body`) remain serialized `Graph` values; consumers wrap them with `graphSource`.

## Design rationale

- **Edges first.** A graph is defined by its wires. Node data is looked up by id, and where it is stored is deferred to the provider.
- **Two small providers, not one interface.** Node and edge storage vary independently (e.g. edges in a document, nodes split between that document and an external store). Each side is one or two methods; the bundle is a plain pair.
- **Nodes are just nodes.** Remote and local nodes share one shape and one definition lookup, so no algorithm grows a "remote" branch.
- **Layering mirrors mixed data sources.** `layeredNodes(local, external)` lets a document override or supplement external nodes without either side knowing about the other.
- **`Graph` unchanged.** The transmission format stays a self-contained document; the provider view is an access layer over it.

## Behavior / pipeline

Access layer only. No serialization format of its own.

## Configuration

None.

## See also

- [graph-model.md](./graph-model.md) — `Graph`, `Node`, `Edge`
- [type-system.md](../imp-typecheck/type-system.md) — endpoint resolution in `checkGraph`
- [resolve.md](../imp-graph-resolve/resolve.md) — flatten / execution program
