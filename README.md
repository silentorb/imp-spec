# imp-spec

**Language-neutral design specs** for [Imp](https://github.com/silentorb/imp-ts) — a universal transmission format for directed acyclic graphs (DAGs).

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
