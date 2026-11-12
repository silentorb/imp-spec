# imp-spec

This repository contains the specification for Imp.

## What is Imp?

**Imp** is a universal transmission format for directed acyclic graphs (DAGs).

Imp defines a portable graph data model: nodes with typed ports, edges that connect ports, and a graph record keyed by ids. Packages in this repo are mostly specifications and converters/translators between Imp and other graph representations.

Imp is the successor to [imp-kotlin](https://github.com/silentorb/imp-kotlin), narrowed to the **graph data layer** only (no text/code language layer).

## Ecosystem

This repository is for agent and human authors of Imp capabilities. Implementations live in separate repos:

| Repo | Role |
| --- | --- |
| [`imp-ts`](https://github.com/silentorb/imp-ts) | TypeScript implementation (primary reference) |
| [`imp-rs`](https://github.com/silentorb/imp-rs) | Rust implementation |

## Layout

| Path | Role |
| --- | --- |
| [`AGENTS.md`](./AGENTS.md) | Agent router — start here |
| [`docs/overview/`](./docs/overview/) | Cross-package architecture and authoring |
| [`docs/packages/`](./docs/packages/) | One directory per Imp package; authoritative specs |

## Agent docs

Start with [`AGENTS.md`](./AGENTS.md). Package specs are under [`docs/packages/`](./docs/packages/).
