# Documentation

Everything under `./docs` is **primarily for AI agents** — stable, high-level knowledge so agents need not re-analyze implementations for basics on every task. Docs should remain easy for humans to read and edit.

## Specs → code (not the reverse)

Package specs describe the **data model and behavior** in language-agnostic terms. They are authored and maintained in this repo.

**Code type interfaces** in language implementations must be regenerable from those specs. An agent should be able to emit unambiguous interfaces from a spec alone. Do **not** treat specs as disposable output of the code.

## Layout

| Path | Role |
| --- | --- |
| [`overview/`](./overview/) | Cross-package architecture, feature registry, authoring template |
| [`packages/`](./packages/) | Authoritative per-package specs |

Start at [`overview/README.md`](./overview/README.md) for the feature registry.
