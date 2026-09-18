# Spike A — Naming / kind ambiguity

> **Historical.** Normative design is [semantic-paths.md](../semantic-paths.md). Untagged tokens are safe when a type-scoped ontology resolves kind; this spike measured corpus collisions that motivate that integrity rule.

## Goal

Measure whether untagged path step strings can safely mean either a field or an edge without a `kind` tag.

## Method

Against an illustrative design corpus that uses **ULID association ids** and human property / table-schema names:

- Association registry size: **39** ULID keys
- Property-ish name set (promoted columns ∪ table-schema field names, heuristic): **106**
- **Association id ∩ property-ish:** **0** collisions

Human **perspective labels** on associations (not Imp `association` port values today) heavily overlap property-ish names (**19** shared labels such as `Features`, `Dependencies`, `Children`). A hostile host that used those labels as association ids would collide with fields (e.g. `title`, `Dependencies`).

## Verdict

| Scheme | Safe for Imp-generic catalogs? |
| --- | --- |
| Untagged bare strings | **No** — host-dependent; fails when association names are human-readable |
| Tagged steps (`kind: field \| edge`) | **Yes** — kind is explicit; no host name resolver required for dispatch |
| ULID-only association ids (one host) | Practically collision-free **for that host**, but must not be assumed by Imp |

**Required for Imp:** tagged steps (or keep separate operators). Untagged strings are rejected for any universal path IR.
