# Beaver Journal

This directory is Beaver's long-term memory. It exists to answer two different
questions quickly, for both humans and coding agents:

> **What does Beaver currently believe?**
>
> **Why did Beaver end up believing that?**

Those are different retrieval problems, and the journal is structured so that
neither one has to be reconstructed by crawling the repository.

---

## Start here

Read in this order when entering the project cold:

1. `STATUS.md` — what phase we are in and what is currently settled
2. `foundations/design-map.md` — the fourteen conceptual areas and their maturity
3. `foundations/vocabulary.md` — canonical terminology and its status
4. Then the relevant subsystem documents below

---

## Authority model

Not all documentation here carries the same weight. This is deliberate.

| Source | Authority | Meaning |
| --- | --- | --- |
| `foundations/` | Product intent | What Beaver is trying to be |
| Living journal docs | Current conceptual design | What we currently believe |
| `adr/` (future, repo root) | Committed decisions | What we have deliberately locked in |
| Code + tests | Implemented behaviour | What actually exists |
| `workbench/` | Historical reasoning | **Not authoritative** |
| `research/` | External precedent | What *other* systems do |

If these disagree, that is a **drift condition**. It should be surfaced, not
silently resolved. An implementation that contradicts a `committed` design or an
ADR is an architectural regression worth reporting.

`research/` is distinct from `workbench/`. Research records what another system
does and what Beaver might learn from it. It never describes how Beaver works.
Its findings are largely **unverified** — see `research/README.md` before relying
on any of them.

---

## Document metadata

Every living document carries a small YAML header:

```yaml
---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - runtime/runtime-model.md
---
```

`status` and `implementation` are **independent dimensions**:

```text
status:                     implementation:
    exploratory                 none
    preferred                   prototype
    committed                   partial
    deferred                    implemented
    superseded
```

Documents under `research/` use `status: research` instead — the living-document
lifecycle does not apply to them, and they are never `committed`. They carry
`verified:` and `sources:` fields as well.

This separation matters. `status: committed` with `implementation: none` means
"we have decided, we have not built it." `status: exploratory` with
`implementation: prototype` means "code exists specifically so we can evaluate
this idea." Both are legitimate and mean very different things to anyone picking
up the work.

Once ADRs exist, any architecturally significant `committed` document must
reference the ADR that commits it:

```yaml
status: committed
adr:
  - ../../adr/0012-action-strategy-model.md
```

---

## Living document structure

Architectural documents follow a familiar flow so that a reader can stop early
once they have what they need:

```text
# Title

## Summary            — three paragraphs maximum
## Motivation         — what problem this solves
## Model              — the actual conceptual architecture
## Examples           — concrete walkthroughs
## Invariants         — what the model must preserve
## Open questions     — what is explicitly unresolved
## Alternatives considered
## Related
```

**Summary + Invariants is usually enough context to begin implementation work.**
The prose explains *why*; the invariants state *what must not be violated*.

---

## Superseded documents do not move

When a living document becomes obsolete, it stays exactly where it is. We change
its header instead:

```yaml
status: superseded
superseded-by:
  - extension-model.md
```

and add a line immediately below the title stating that it is historical and
must not be used for new implementation.

Stable paths are worth more than a cosmetically tidy directory tree. Moving files
to an `archive/` folder breaks links, issues, agent prompts and years of
references for no real gain.

---

## Do not duplicate truth

Each concept has exactly one owning document. Others link to it.

```text
foundations/vocabulary.md     brief canonical definition
runtime/runtime-model.md      the detailed model
```

```text
domain/spatial-model.md       Beaver's spatial semantics
integrations/hyprland.md      how those semantics map onto Hyprland
```

A document that independently redefines a concept it does not own is
documentation drift in progress.

---

## Directory layout

```text
journal/
├── README.md              this file — how to read the documentation
├── STATUS.md              where the project currently is
│
├── foundations/           what Beaver is and what words mean
├── domain/                the desktop model Beaver understands
├── runtime/               how Beaver behaves and is extended at runtime
├── policy/                configuration, overlays, precedence, persistence
├── shell/                 (future) interaction and visual language
├── integrations/          (future) Hyprland, Quickshell, D-Bus, Linux platform
├── extensions/            the extension surfaces and their contracts
├── research/              external precedent — unverified, cited
└── workbench/             dated design exploration, non-authoritative
```

Directories appear when they have something meaningful to contain. We do not
generate placeholder files to satisfy the shape.

Two conventions worth stating:

- **No numeric directory prefixes.** `00-foundations/` looks ordered on day one
  and becomes a miniature Dewey Decimal System by year five. Ordering lives in
  this README, where changing it costs nothing.
- **Two levels of nesting, occasionally three.** Deep trees feel architecturally
  satisfying right up until you cannot remember where you put something.

---

## Context paths

Manual semantic retrieval for the repository. Read these sets, not the whole
journal.

**Working on spatial / compositor behaviour**
```text
foundations/vocabulary.md
domain/spatial-model.md
runtime/runtime-model.md
research/hyprland-ipc.md
```

**Working on configuration, policy or precedence**
```text
foundations/vocabulary.md
runtime/runtime-model.md
policy/overlays-and-sog.md
research/nixos-module-merging.md
research/css-cascade.md
research/systemd-drop-ins.md
```

**Working on extensions or the SDK**
```text
foundations/vocabulary.md
runtime/runtime-model.md
extensions/extension-model.md
research/embedded-scripting-runtimes.md
research/dbus-and-zbus.md
```

**Orienting on the project as a whole**
```text
STATUS.md
foundations/design-map.md
foundations/vocabulary.md
```

---

## Information flow

```text
        brainstorming
              │
              ▼
          workbench/
              │
         distillation
              │
              ▼
      living journal doc
              │
         confidence
              │
              ▼
             ADR
              │
        implementation
              │
              ▼
         code / tests
```

The journal continues living after an ADR is written. They are not duplicates:

- The **ADR** answers *why we committed to this decision at that point in time.*
- The **journal** answers *what this subsystem looks like today.*

Git provides revision history. The journal does not need changelogs.

---

## Notes for coding agents

- Read `STATUS.md` before assuming any design is settled.
- **Never** derive current architecture from `workbench/`. It is preserved
  reasoning, including ideas we rejected.
- Do not introduce public terminology marked `exploratory` into APIs, type names
  or user-facing surfaces without discussion.
- Where a document records an open question, that question is the current state
  of the design. Do not resolve it silently in code.
- If two documents conflict, surface the conflict rather than picking one.
