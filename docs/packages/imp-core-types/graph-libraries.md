# Graph libraries

## Summary

**JSON-serializable catalogs** of graph-backed `NodeDefinition` entries. Hosts ship graph libraries as files containing zero or more reusable subgraph definitions that parent graphs reference by `Node.type`.

## When to read this

- Authoring or loading graph library JSON files
- Implementing `imp-graph-library` parse/validation
- Registering graph-backed node types in a host application

## Requirements

### GraphLibrary

| Field | Type | Required |
| --- | --- | --- |
| `id` | `string` | must — library provenance id (not prefixed onto definition ids) |
| `definitions` | `NodeDefinition[]` | must (may be empty) |

Each entry in `definitions` **must** be graph-backed:

- `body` present
- `bindings` present
- `implementation` absent

Primitive operator definitions belong in `NodeLibrary` packages (e.g. `imp.collection.transforms`), not graph library JSON files.

### JSON file format

Graph libraries are UTF-8 JSON objects. Example:

```json
{
  "id": "example.query-patterns",
  "definitions": [
    {
      "id": "filter-archived",
      "name": "Filter archived nodes",
      "inputs": {
        "rows": {
          "id": "rows",
          "type": { "id": "collection", "args": [{ "param": "T" }] }
        }
      },
      "outputs": {
        "rows": {
          "id": "rows",
          "type": { "id": "collection", "args": [{ "param": "T" }] }
        }
      },
      "typeParams": [{ "id": "T" }],
      "bindings": {
        "inputs": { "rows": "in_rows" },
        "outputs": { "rows": "out_rows" }
      },
      "body": {
        "nodes": {
          "in_rows": {
            "id": "in_rows",
            "type": "input",
            "typeArgs": [{ "id": "collection", "args": [{ "param": "T" }] }],
            "inputs": {}
          },
          "out_rows": { "id": "out_rows", "type": "output", "inputs": {} }
        },
        "edges": {}
      }
    }
  ]
}
```

### Identifier strategy

| Field | Rules |
| --- | --- |
| `GraphLibrary.id` | Plain string; identifies the file/package for provenance |
| `NodeDefinition.id` | Plain string (required). **ULIDs recommended** for library-authored entries to reduce cross-library name collisions |
| `NodeDefinition.name` | Optional human label; editors display `name` when present, else `id` |

Bindings **must not** enforce ULID format in this revision — plain strings remain valid. Hosts may generate ULIDs when creating new library entries.

### Array vs map

`definitions` is an **array**, not a map keyed by id. Primary keys live on each entry's `id` property. This supports ULID ids and decoupled display names without dict-key constraints.

Duplicate `id` values within one file are ill-formed. The registry rejects duplicate ids across loaded libraries.

### Validation (`imp-graph-library`)

Parsing **must** validate:

1. Top-level shape: `id` string, `definitions` array
2. Each definition satisfies graph-backed rules (`body` + `bindings`, no `implementation`)
3. `bindings.inputs` / `bindings.outputs` keys match `inputs` / `outputs` port keys exactly
4. Bound node ids exist in `body.nodes`
5. Bound input nodes have `type: "input"`; bound output nodes have `type: "output"`
6. `body` is a well-formed `Graph` (node keys match node ids)

Structural validation runs at parse time. Type compatibility of boundary ports vs declared ports is checked by `imp-typecheck` (`checkDefinition`).

### Loading pipeline

1. Host reads JSON file
2. `parseGraphLibraryJson(text)` → `GraphLibrary`
3. `loadGraphLibrary(registry, library)` merges definitions into registry (throws on duplicate `NodeDefinition.id`)

Registry itself performs no file I/O.

## Design rationale

- **JSON files** let hosts version and ship reusable graph patterns without code changes.
- **Array `definitions`** keeps ids as data (ULID-friendly) and separates display `name` from identity.
- **Same `NodeDefinition` shape** as code-defined libraries — graph libraries are a serialization profile, not a parallel catalog system.
- **Parse-time structural validation** catches malformed bindings early; semantic checks stay in `imp-typecheck`.

## Behavior / pipeline

See [node-libraries.md](./node-libraries.md), [registry.md](../imp-registry/registry.md), and [resolve.md](../imp-graph-resolve/resolve.md).

## Configuration

None.

## See also

- [node-libraries.md](./node-libraries.md)
- [graph-model.md](./graph-model.md)
- [type-system.md](../imp-typecheck/type-system.md)
- [resolve.md](../imp-graph-resolve/resolve.md)
- Root [AGENTS.md](../../../AGENTS.md)
