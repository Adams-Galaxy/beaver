---
status: preferred
implementation: none
last-reviewed: 2026-09-02
related:
  - ../STATUS.md
  - vocabulary.md
---

# Beaver Design Map

## Summary

Fourteen conceptual areas that together define what Beaver is. This is not a
requirements list and not a roadmap. It is a **coverage map**: it tells us what
we have thought about seriously, what we have barely touched, and where a new
idea belongs.

Its two jobs are to answer *what haven't we thought about yet?* during design,
and *which part of the architecture does this feature belong to?* during
implementation.

It exists specifically because this project is prone to going very deep on one
area — precedence graphs, extension hosts — and forgetting that whole regions
of the desktop remain unexamined.

## Maturity scale

```text
strong        sustained exploration, direction is stable
developing    real progress, direction still moving
partial       touched, significant gaps remain
early         ideas only
untouched     not yet discussed
```

---

## 1. Product identity and philosophy

**Maturity: strong**

What using Beaver should feel like, who it is for, and how opinionated it should
be.

Current direction:

- Personal-first. When generality conflicts with making one person's desktop
  excellent, the workflow wins — but the architecture must stay clean enough
  that others *can* use it.
- Opinionated defaults so a fresh install is a complete desktop; user
  configuration determines operational personality.
- Quietly powerful. Visually restrained, functionally deep. Sophistication lives
  under the floorboards rather than announcing itself.
- Install Beaver and you get a complete desktop. Install Beaver plus your
  dotfiles and your *universe* comes back — not an approximation of it.

Open:

- How much UI flavouring is sensible versus fully overridable
- Whether the shipped shell should be "reference" or "default"

---

## 2. Desktop mental model

**Maturity: developing**

What the environment actually understands as objects.

Relevant: `../domain/spatial-model.md`

Current direction: Space, Plane, Lane, View, Window, Monitor, plus cross-cutting
contextual overlays. Beaver owns desktop *semantics*; Linux services own system
truth.

Open:

- Whether projects and applications are first-class model objects (area 10)
- What a Space's relationship to a directory or environment really is

---

## 3. Window and workspace philosophy

**Maturity: strong (conceptually), developing (behaviour)**

Relevant: `../domain/spatial-model.md`

Current direction: tiling-first with floating as a first-class citizen. The
spatial graph is `(z, y, x)` — Plane, Lane, View. Monitors are viewports onto
the graph rather than owners of it. Navigation is a transformation of monitor
assignments, chosen by an Action Strategy.

Open:

- Monitor pairing and grouping policy
- Plane semantics on monitor disconnect
- Whether floating windows have spatial identity

---

## 4. Shell interaction model

**Maturity: partial**

The surfaces — bar, command surface, overview, notifications, control centre,
OSDs, lock screen, widgets — and, more importantly, the interaction language
connecting them.

Current direction: default is a top bar, tiled applications, wallpaper. No dock,
because the command surface replaces it. Depth is summoned, not permanently
displayed. Widget support is designed for but not built early.

Open: nearly everything below the command surface. This is one of the largest
remaining gaps.

---

## 5. Profiles and context

**Maturity: developing, terminology unresolved**

Relevant: `../policy/overlays-and-sog.md`

Current direction: a contextual overlay is a named, user-activatable
contribution of desired policy with identity and lifecycle. A Space also
contributes an overlay. Both share the composition machinery rather than
inventing two configuration systems.

Open:

- The name. "Focus" is a working term, not a decision.
- Whether multiple contexts compose (`desk + engineering + focus`)
- Lifecycle: activation, deactivation, expiry, exclusivity

---

## 6. Configuration philosophy

**Maturity: active deep exploration**

Relevant: `../policy/overlays-and-sog.md`, `../runtime/runtime-model.md`

Current direction:

- A typed Rust configuration model is canonical; TOML and Lua are ways of
  producing it
- Reload is transactional — a rejected candidate leaves the running config
  untouched
- Precedence is a graph (SOG); merge behaviour belongs to the option schema
- Configuration describes structure; Lua describes logic

Open: SOG topology, facet routing, merge algebra, ambiguity handling, manual and
transient overrides.

---

## 7. Modularity and extensibility

**Maturity: strong**

Relevant: `../extensions/extension-model.md`

Current direction: "Module" is deliberately **not** a fundamental runtime
abstraction. Beaver exposes specific extension points — actions, strategies,
providers, pipeline stages, event subscriptions, capabilities — and a thing
casually called a module is a bundle of those. Three surfaces: Lua in-process,
D-Bus out-of-process, QML for presentation.

Open: managed extension lifecycle, packaging, permissions.

---

## 8. Daemon authority

**Maturity: strong**

Who owns what:

```text
Linux services    physical and system truth
                  NetworkManager, PipeWire, BlueZ, Hyprland

beaverd           desktop semantics
                  Spaces, navigation, desired policy, actions,
                  overlay composition, persistence intent

Quickshell        presentation state
                  hover, animation, scroll position, selection

Lua               user-defined policy and logic,
                  acting through defined capabilities
```

Open: how far reconciliation reaches into services Beaver does not own.

---

## 9. Actions, events and automation

**Maturity: strong**

Relevant: `../runtime/runtime-model.md`

Current direction: Event (fact), Action (intent), Action Strategy (selectable
implementation), Provider (additive), Pipeline (ordered), Capability
(discoverable facility). Keybinds, CLI, UI and automation all invoke the same
semantic actions.

Open: visibility and exposure metadata, execution classes. Action namespacing is
settled — see `vocabulary.md`.

---

## 10. Applications and projects

**Maturity: early**

Whether Beaver understands only `.desktop` entries, or richer things — a
project, a repository, a dev environment, a toolchain.

Current direction: compose with mature Unix tooling rather than cloning it.
`direnv`, `just`, the user's own shell and Git already express context well. A
Space with a directory gives the command surface real project context for free.

Open: nearly all of it. The constraint for now is simply to avoid architectural
choices that make this awkward later.

---

## 11. System integration and boundaries

**Maturity: strong**

Which wheels we refuse to reinvent: NetworkManager, BlueZ, PipeWire/WirePlumber,
UPower, systemd, xdg-desktop-portal, polkit, Secret Service, MPRIS. Beaver
orchestrates them and provides one coherent UX and policy layer.

---

## 12. Reliability and lifecycle

**Maturity: partial**

What happens when Quickshell crashes, the daemon crashes, Hyprland reloads, an
extension dies mid-strategy, or user Lua enters an infinite loop.

Current direction: the core must remain recoverable when user behaviour code
fails. A failed Lua strategy should fall back to the built-in implementation and
report, not take the desktop with it. `beaverd` is session-scoped — Plasma
remains an untouched rescue boat during the dual-session period.

Open: systemd user service topology, restart policy, what persists across
restarts, "lite core" definition.

---

## 13. Visual identity and design language

**Maturity: untouched**

Density, motion, depth, typography, hierarchy, panel philosophy, how much chrome
exists, how animation communicates state.

Deliberately deferred. Deserves its own document when reached.

---

## 14. Project identity and naming

**Maturity: developing**

Beaver — for how beavers build and reshape their environment to suit how they
work.

Current direction: the project name carries the personality; the basic nouns
stay legible. Space, Plane, Lane, View describe rather than decorate. Nobody
should need a zoology glossary to open Firefox.

Open: whether any Beaver-flavoured vocabulary is worth adopting at the edges,
and the binary and service names (`beaverd`, `beaverctl` are working names).

---

## Related

- `../STATUS.md`
- `vocabulary.md`
