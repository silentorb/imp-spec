# Imp files

## Summary

An **Imp file** is the one JSON shape for transmitting and storing Imp content. Like a source file, it may hold any mix of **elements**: type definitions, node definitions, nodes and edges. It replaces both the serialized graph document (`{ nodes, edges }`) and the graph-library file (`{ id, definitions }`). Nodes and edges in a file are not "a graph" by construction: they become a [graph source](../imp-core-types/graph-sources.md) that may be combined with other files and providers.

## When to read this

- Reading, writing or storing Imp JSON (files, record values, messages)
- Implementing `imp-file` parse / validate / serialize
- Shipping type definitions or reusable node definitions as data

## Requirements

### ImpFile

| Field | Type | Required |
| --- | --- | --- |
| `format` | `"imp"` | may — identifies Imp content; recommended for standalone files |
| `id` | `string` | may — provenance (e.g. a library id); not prefixed onto element ids |
| `types` | `TypeDefinition[]` | may |
| `definitions` | `NodeDefinition[]` | may |
| `nodes` | `Node[]` | may |
| `edges` | `ImpFileEdge[]` | may |

Every section is optional; an absent section is the same as an empty one. A file with no sections is valid (an empty file).

- **`format`**, when present, must be the string `"imp"`. Hosts use it to recognize Imp content among other JSON documents. It replaces host-invented tags such as `"type": "imp-graph"`.
- **Element ids.** Every element carries its own `id`. Within one file, ids must be unique **per section** (two `nodes` may not share an id; a node and an edge may). Sections are arrays, not id-keyed maps, so no element's identity depends on where it sits.
- **Order.** Element order is preserved as written. Serializers must not reorder elements, so stored files diff stably. Order carries no meaning except where a field says so (`Edge.order`).

| Section | Element shape | Spec |
| --- | --- | --- |
| `types` | `TypeDefinition` | [type-definitions.md](../imp-core-types/type-definitions.md) |
| `definitions` | `NodeDefinition` — primitive **or** graph-backed | [node-libraries.md](../imp-core-types/node-libraries.md) |
| `nodes` | `Node` | [graph-model.md](../imp-core-types/graph-model.md) |
| `edges` | `ImpFileEdge` | below |

### ImpFileEdge

An `Edge` plus its identity:

| Field | Type | Required |
| --- | --- | --- |
| `id` | `EdgeId` | must |
| `from` | `PortReference` | must |
| `to` | `PortReference` | must |
| `order` | integer | may — see [graph-model.md](../imp-core-types/graph-model.md) |

### Example

```json
{
  "format": "imp",
  "id": "example.audio",
  "types": [
    { "id": "Audio", "type": { "union": [{ "id": "audio_mono" }, { "id": "audio_stereo" }] } }
  ],
  "definitions": [
    {
      "id": "echo",
      "typeParams": [{ "id": "A", "bounds": [{ "id": "Audio" }] }],
      "inputs": { "audio": { "id": "audio", "type": { "param": "A" } } },
      "outputs": { "audio": { "id": "audio", "type": { "param": "A" } } },
      "bindings": { "inputs": { "audio": "in" }, "outputs": { "audio": "out" } },
      "body": {
        "nodes": {
          "in": { "id": "in", "type": "input", "inputs": {} },
          "out": { "id": "out", "type": "output", "inputs": {} }
        },
        "edges": { "e1": { "from": { "node": "in", "port": "value" }, "to": { "node": "out", "port": "value" } } }
      }
    }
  ],
  "nodes": [
    { "id": "src", "type": "sine", "inputs": {} },
    { "id": "fx", "type": "echo", "inputs": {} }
  ],
  "edges": [
    { "id": "e1", "from": { "node": "src", "port": "audio" }, "to": { "node": "fx", "port": "audio" } }
  ]
}
```

A graph-backed definition's `body` is an in-memory `Graph` (id-keyed maps), as in [node-libraries.md](../imp-core-types/node-libraries.md); only top-level sections are arrays.

### Validation (`imp-file`)

Parsing **must** validate:

1. The top level is an object; `format`, when present, is `"imp"`; `id`, when present, is a non-empty string
2. Each present section is an array of objects
3. Every element has a non-empty string `id`, unique within its section
4. `types[]` entries have `type`; `params`, when present, is an array
5. `nodes[]` entries have a string `type` and an object `inputs`
6. `edges[]` entries have `from` and `to` port references
7. Graph-backed `definitions[]` entries (`body` present) satisfy the graph-backed rules: `bindings` present, `implementation` absent, `bindings.inputs` / `bindings.outputs` keys match `inputs` / `outputs` port keys exactly, bound node ids exist in `body.nodes` with type `input` / `output` respectively, and `body` is a well-formed `Graph` (node keys match node ids)
8. Primitive `definitions[]` entries (no `body`) have no `bindings`

Structural validation runs at parse time. Type compatibility (boundary ports vs declared ports, type-definition validity) is checked by `imp-typecheck` (`checkGraphDefinitions`).

Edges may name endpoints that no node in the file defines; resolution is a graph-source concern (see [graph-sources.md](../imp-core-types/graph-sources.md)).

### Loading

Loading has two independent halves:

| Half | Operation | Result |
| --- | --- | --- |
| Catalog | `loadImpFile(registry, file)` — see [registry.md](../imp-registry/registry.md) | `types` and `definitions` merged into the registry's global namespaces (duplicate ids throw) |
| Topology | `impFileGraphSource(file)` | A `GraphSource` over the file's `nodes` and `edges` |

Hosts that need an in-memory `Graph` materialize one from the graph source (`materializeGraph`). Writers turn a `Graph` back into a file with `impFileFromGraph(graph)`, which emits `nodes` / `edges` arrays (edge ids from the graph's edge keys).

### Restricting sections

A host may accept only some sections in a given slot, e.g. a stored record that holds only topology. `assertImpFileSections(file, allowed)` throws when a section outside `allowed` is non-empty. Files stored as the value of a record (for example a Tome node property) omit `id`: the record's own identity identifies them.

### API (`imp-file`)

| Operation | Behavior |
| --- | --- |
| `parseImpFileJson(text)` | Parse UTF-8 JSON text and validate; returns `ImpFile` or throws `ImpFileParseError` |
| `parseImpFile(value)` | Validate an already-parsed JSON value |
| `serializeImpFile(file)` | JSON text with sections in the order `format`, `id`, `types`, `definitions`, `nodes`, `edges`; empty sections omitted; element order preserved |
| `impFileGraph(file)` | In-memory `Graph` (id-keyed maps) over `nodes` / `edges`, in element order |
| `impFileGraphSource(file)` | Graph source over `nodes` / `edges` |
| `impFileFromGraph(graph, extras?)` | Imp file holding a graph's nodes and edges (plus optional `format` / `id` / `types` / `definitions`) |
| `assertImpFileSections(file, allowed)` | Throws when a section outside `allowed` is non-empty |

## Design rationale

- **One shape** replaces graph documents, graph libraries and host-tagged variants, matching Imp's decentralized node / edge model: content is a set of elements, not one graph.
- **Arrays with per-element ids** keep identity in the data (ULID-friendly), decouple display names, and let a host store or move individual elements without rewriting map keys.
- **Primitive definitions allowed.** Extensions already exchange primitive catalogs as JSON; a file may declare node types whose implementations a host supplies.
- **Two loading halves.** Catalog content (types, definitions) is global once loaded; topology (nodes, edges) is a graph source, so several files and remote providers compose.
- **Parse-time structural validation** catches malformed elements early; semantic checks stay in `imp-typecheck`.

## Behavior / pipeline

1. Host reads JSON text (a file, a record value, a message).
2. `parseImpFileJson(text)` → `ImpFile`.
3. `loadImpFile(registry, file)` when the file carries `types` or `definitions`.
4. `impFileGraphSource(file)` (or `materializeGraph`) for its nodes and edges; `checkGraph(source, registry)`.

## Configuration

None.

## See also

- [graph-model.md](../imp-core-types/graph-model.md)
- [graph-sources.md](../imp-core-types/graph-sources.md)
- [node-libraries.md](../imp-core-types/node-libraries.md)
- [type-definitions.md](../imp-core-types/type-definitions.md)
- [registry.md](../imp-registry/registry.md)
- [type-system.md](../imp-typecheck/type-system.md)
- Root [AGENTS.md](../../../AGENTS.md)
