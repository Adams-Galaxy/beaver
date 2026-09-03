# Agent Instructions

Beaver is in **conceptual design**. There is no implementation yet — no Rust
workspace, no Quickshell frontend, no build system. This is deliberate.

## Before any architecture-sensitive work

1. Read `journal/README.md` — how this project's documentation works and what
   carries authority.
2. Read `journal/STATUS.md` — what is settled, what is exploratory, what is
   untouched.
3. Read the relevant context path listed at the end of `journal/README.md`.

Do not crawl the repository to reconstruct context. The journal has explicit
reading paths for each area of work.

## Rules

- **`journal/workbench/` is not authoritative.** It preserves reasoning,
  including rejected ideas. Never derive current architecture from it.
- **Do not resolve open questions silently.** Where a document records something
  as unresolved, that *is* the current state of the design. Raise it; don't
  decide it in code.
- **Do not introduce `exploratory` terminology** into APIs, type names or
  user-facing surfaces. `journal/foundations/vocabulary.md` marks the status of
  every term.
- **Do not "improve" the architecture** while doing documentation or
  implementation work. If two documents conflict, surface the conflict.
- **Do not duplicate truth.** Each concept has one owning document. Link to it.
- An implementation contradicting a `committed` design or an ADR is an
  architectural regression worth reporting, not silently accommodating.

## Repository layout

```text
journal/    living design documentation and preserved exploration
adr/        (not yet created) committed architectural decisions
tmp/        scratch, not tracked as project material
```
