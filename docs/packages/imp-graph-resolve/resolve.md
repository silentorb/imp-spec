# Graph resolution

## Summary

**`imp-graph-resolve`** performs **late** expansion of graph-backed `NodeDefinition` references. Transmitted and stored graphs keep composite nodes as `Node.type` lookups; resolution runs at pipeline boundaries before translation or execution.

## When to read this

- Implementing nested graph inlining or execution programs
- Wiring Imp Translators or runtime consumers to support graph-backed definitions
- Understanding when graphs stay referential vs flattened

## Requirements

### Referential vs resolved

| Stage | Composite handling |
| --- | --- |
| Storage / transmission | Reference only (`Node.type` → registry lookup) |
| `checkGraph` (static) | Type-check against catalog ports; **do not** inline |
| Translation | `flattenGraph` immediately before translation |
| Runtime execution | `buildExecutionProgram` → shared subgraph table |

### flattenGraph

`flattenGraph(graph, registry) → Graph`

Recursively expands graph-backed node instances into a single flat graph suitable for existing per-operator translation:

1. Detect composite instances: `getNodeDefinition(registry, node.type)` has `body`
2. Topologically expand composites (instance-level cycle → error)
3. For each composite instance:
   - Load `NodeDefinition.body`
   - Apply `instantiateNodeDefinition` with instance `typeArgs`
   - Rename inner node/edge ids with prefix `${instanceId}::`
   - Rewire: parent inbound edges → bound input boundary ports; parent outbound edges ← bound output boundary ports
4. Remove the composite instance node after splicing
5. Repeat until no composite instances remain

Definition-reference cycles (A's body references B, B's body references A) are detected separately by `checkGraphDefinitions` in `imp-typecheck`.

### buildExecutionProgram

`buildExecutionProgram(graph, registry) → ExecutionProgram`

Preferred path for **runtime consumers** — avoids duplicating shared subgraph bodies:

| Field | Type | Role |
| --- | --- | --- |
| `root` | `Graph` | May still contain composite node instances |
| `subgraphs` | map `SubgraphKey` → `Graph` | Shared instantiated bodies |

`SubgraphKey` = `${definitionId}::${serializedTypeArgs}` (stable JSON serialization of concrete `typeArgs`).

Rules:

- One subgraph entry per unique `(definitionId, typeArgs)` pair
- Multiple composite instances with the same definition + typeArgs **share** one subgraph entry
- Subgraph bodies are instantiated (`typeArgs` applied) but not renamed per call site
- Runtime enters a subgraph at composite nodes, wiring parent edges to boundary nodes via `bindings`

### Boundary wiring (both APIs)

For composite instance `C` with definition `D` and bindings `B`:

- Edge `X.out → C.inPort` becomes wired to inner node `B.inputs[inPort]` at its `value` output
- Edge `C.outPort → Y.in` becomes wired from inner node `B.outputs[outPort]` at its `value` input

Inner literal values on composite instance `C.inputs` map to inner boundary `input` node literals when present.

### Error shape

`flattenGraph` and `buildExecutionProgram` return structured errors (not plain strings):

| Field | Type | Role |
| --- | --- | --- |
| `instanceId` | `NodeId` | may — composite node instance being expanded |
| `definitionId` | `NodeTypeId` | may — catalog definition (`Node.type`) |
| `portId` | `PortId` | may — declared port when binding wiring fails |
| `message` | `string` | must |

## Design rationale

- **Late explosion** keeps transmission format stable and avoids prematurely resolved embedded graphs.
- **Flattening for translation** reuses existing per-operator translation without composite-aware switches.
- **Execution program** preserves shared subgraph resources for functional-style reuse across multiple call sites in one run.

## Behavior / pipeline

### Imp Translators

1. `flattenGraph(graph, registry)`
2. Translate the flat graph to the target format (e.g. SQL)

### Runtime consumers

1. `buildExecutionProgram(graph, registry)`
2. Execute composite nodes via `subgraphs` lookup

## Configuration

None.

## See also

- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-libraries.md](../imp-core-types/graph-libraries.md)
- [type-system.md](../imp-typecheck/type-system.md)
- Root [AGENTS.md](../../AGENTS.md)
