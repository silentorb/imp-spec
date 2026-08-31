# Graph resolution

## Summary

**`imp-graph-resolve`** performs **late** expansion of graph-backed `NodeDefinition` references. Transmitted and stored graphs keep composite nodes as `Node.type` lookups; resolution runs at pipeline boundaries (SQL compile, execution).

## When to read this

- Implementing nested graph inlining or execution programs
- Wiring `imp-sql` or `imp-execution` to support graph-backed definitions
- Understanding when graphs stay referential vs flattened

## Requirements

### Referential vs resolved

| Stage | Composite handling |
| --- | --- |
| Storage / transmission | Reference only (`Node.type` → registry lookup) |
| `checkGraph` (static) | Type-check against catalog ports; **do not** inline |
| SQL compile | `flattenGraph` immediately before lowering |
| Execute | `buildExecutionProgram` → shared subgraph table |

### flattenGraph

`flattenGraph(graph, registry) → Graph`

Recursively expands graph-backed node instances into a single flat graph suitable for existing SQL lowering:

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

Preferred path for **`imp-execution`** — avoids duplicating shared subgraph bodies:

| Field | Type | Role |
| --- | --- | --- |
| `root` | `Graph` | May still contain composite node instances |
| `subgraphs` | map `SubgraphKey` → `Graph` | Shared instantiated bodies |

`SubgraphKey` = `${definitionId}::${serializedTypeArgs}` (stable JSON serialization of concrete `typeArgs`).

Rules:

- One subgraph entry per unique `(definitionId, typeArgs)` pair
- Multiple composite instances with the same definition + typeArgs **share** one subgraph entry
- Subgraph bodies are instantiated (`typeArgs` applied) but not renamed per call site
- Interpreter enters a subgraph at composite nodes, wiring parent edges to boundary nodes via `bindings`

### Boundary wiring (both APIs)

For composite instance `C` with definition `D` and bindings `B`:

- Edge `X.out → C.inPort` becomes wired to inner node `B.inputs[inPort]` at its `value` output
- Edge `C.outPort → Y.in` becomes wired from inner node `B.outputs[outPort]` at its `value` input

Inner literal values on composite instance `C.inputs` map to inner boundary `input` node literals when present.

## Design rationale

- **Late explosion** keeps transmission format stable and avoids prematurely resolved embedded graphs.
- **SQL flattening** reuses existing per-operator lowering without composite-aware switches.
- **Execution program** preserves shared subgraph resources for functional-style reuse across multiple call sites in one run.

## Behavior / pipeline

### SQL (`imp-sql`)

1. `flattenGraph(graph, registry)`
2. Existing `graphToSql` / `graphToKysely` on flat graph

### Execute (`imp-execution`)

1. `buildExecutionProgram(graph, registry)`
2. Interpreter handles composite nodes via `subgraphs` lookup

## Configuration

None.

## See also

- [node-libraries.md](../imp-core-types/node-libraries.md)
- [graph-libraries.md](../imp-core-types/graph-libraries.md)
- [sql.md](../imp-sql/sql.md)
- [execution.md](../imp-execution/execution.md)
- [type-system.md](../imp-typecheck/type-system.md)
- Root [AGENTS.md](../../AGENTS.md)
