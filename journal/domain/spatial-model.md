---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - ../foundations/vocabulary.md
  - ../runtime/runtime-model.md
---

# Spatial Model

## Summary

Beaver organises the desktop as a **Space** containing a three-axis grid of
Views. A View is the actual virtual desktop that holds windows, and maps onto one
Hyprland workspace. Its address is `(z, y, x)` — Plane, Lane, View.

Monitors do not own this structure. They are **viewports onto it**. A Space holds
a separate presentation assignment of `Monitor → View`, and navigation is a
transformation of that assignment rather than a compositor-level workspace
switch.

The hierarchy and the axes are a stable direction. Most of the *behaviour* built
on top of them — monitor grouping, Plane semantics, disconnect handling — is
still exploratory.

## Motivation

Hyprland's workspace abstraction is a flat numbered list. That is a perfectly
good compositor primitive and a poor worldview for a desktop environment.

The model this project actually wants is closer to tmux: sessions and structured
panes as first-class, persistent, named things. A workspace should be able to
mean *"the RoboCup Space, development Lane, firmware View"* rather than
*"workspace 7"*.

Making Hyprland's numbering carry that meaning would weld the entire desktop to
one compositor's data model, which is exactly what
`../foundations/design-map.md` area 8 says we should not do.

## Model

### The hierarchy

```text
Space
  Plane[z]
    Lane[y]
      View[x]
        Window
```

```text
Space: robocup
Plane 0

             horizontal (x)
        ─────────────────────►

Lane A       A1   A2   A3   A4

Lane B       B1   B2   B3

Lane C       C1   C2   C3   C4   C5

  │
  │ vertical (y)
  ▼
```

Lanes need not be equal length. The grid is ragged.

### The compositor mapping

```text
Beaver View  <── adapter ──> Hyprland workspace
```

One View, one Hyprland workspace. The workspace number is an adapter detail that
should not appear in the domain model.

### Monitors as viewports

```text
DP-1  → View (0, 1, 2)
DP-2  → View (0, 1, 3)
DP-3  → View (0, 0, 1)
```

This is the single most important idea in this document. Spatial structure and
monitor presentation are **separate concerns**. The Space's shape does not change
when a monitor is plugged in, unplugged, or reassigned.

### Two monitor presentations

**Shared mode** — all monitors draw Views from the same Plane. The default, and
the case almost everyone will use.

**Independent-plane mode** — each monitor is assigned its own Plane:

```text
DP-1 → Plane 0
DP-2 → Plane 1
DP-3 → Plane 2
```

Each monitor effectively gets its own two-dimensional desktop sheet, while all
sheets remain part of one Space. This gives the second mode without inventing a
second, unrelated data structure.

### The Plane dimension is optional

A user with:

```text
1 Space
1 Plane
N Lanes
N Views
```

never encounters `z` at all. This is a design requirement, not an accident:
**zero tax until you use it.** Beaver is not to be advertised as the world's
first desktop hypercube.

### Navigation

Navigation transforms the `Monitor → View` assignment. Which monitors move
together is decided by an Action Strategy, not by the spatial model.

Working default keybindings, illustrative only:

```text
Super + Left/Right          x: Views within a Lane
Super + Up/Down             y: Lanes within a Plane
Super + Shift + Up/Down     z: Planes
Super + Shift + Left/Right  Spaces
```

Changing Space carries meaningful ripple, so it deserves the deliberate modifier.

## Examples

### Independent vertical navigation (default)

`Super+Down` with DP-1 focused:

```text
before                  after
DP-1: B2                DP-1: C2
DP-2: B3                DP-2: B3
DP-3: A1                DP-3: A1
```

### Grouped vertical navigation

The Space declares DP-1 and DP-2 as a synchronised vertical group:

```text
before                  after
DP-1: B2                DP-1: C2
DP-2: B3                DP-2: C3
DP-3: A1                DP-3: A1
```

The Space model is untouched. Only the navigation strategy differs.

Grouping can differ per axis:

```text
vertical groups:    [DP-1, DP-2]  [DP-3]
horizontal groups:  [DP-1]  [DP-2]  [DP-3]
```

### Monitor disconnect

With DP-1 on Plane 0 and DP-2 on Plane 1, DP-2 vanishes. Plane 1 does not have to
vanish with it — it can remain part of the Space, simply undisplayed, and stay
navigable via `z`.

Or policy can fold its Views into a surviving Plane. Both are legitimate; this is
a `monitor.reflow` strategy decision, not a model decision.

### Where declarative config stops

This is expressible as configuration:

```toml
[navigation.vertical]
strategy = "grouped"
groups = [["DP-1", "DP-2"], ["DP-3"]]
```

This is not:

> Group DP-1 and DP-2 only when both are connected, unless I'm using the laptop
> display while docked, except in the presentation Space.

That is programming. It belongs in Lua as a registered strategy. We should not
invent a declarative language complicated enough to accidentally become a poor
programming language.

## Invariants

- **Spatial structure and monitor presentation are separate concerns.** A Space's
  shape must not change as a consequence of monitor topology changing.
- A View maps to exactly one compositor workspace, and that mapping is owned by
  the adapter. Compositor workspace identifiers must not appear in the domain
  model.
- Monitors are viewports. They do not own Lanes, Planes or Views.
- Navigation is a transformation of the `Monitor → View` assignment, performed by
  a replaceable Action Strategy.
- The Plane dimension must impose no cost on users who do not use it.

## Open questions

- **Persistence.** What survives a restart? The working position is *restorable
  desktop intent* — names, structure, directories, overlays, which applications
  belong where — rather than pretending to checkpoint arbitrary Linux processes.
  Application restoration is best-effort and application-specific.
- **Space definition versus Space instance.** Dotfiles would define a Robocup
  *template*; the running system holds a Robocup *instance* that remembers what
  happened while it was in use. This distinction looks powerful and is not worked
  out.
- Are Lanes meaningful entities or purely geometry? Can they be named, collapsed,
  reordered?
- Do Spaces have an inherent relationship to projects and directories, or is that
  merely a common configuration?
- What does moving a *window* across Lanes, Planes and Spaces mean, and how does
  it interact with monitor assignment?
- Do floating and fullscreen windows have spatial identity, or are they View-local?
- Is `z` traversal ever something a user does deliberately in normal use, or only
  a recovery path when a monitor disappears?
- Can two monitors display the same View? Should they be able to?

## Alternatives considered

**Calling the top-level object a Workspace.** Rejected. Hyprland already owns
that word concretely, and Beaver's object means something substantially larger.
The collision would produce permanent explanatory friction.

**Giving each monitor an entirely independent spatial graph.** Rejected in favour
of Planes. Independent graphs give the same capability while making
cross-monitor movement and reflow into special cases; Planes keep one structure.

**Arbitrary volumetric 3D navigation.** Considered and set aside. 2.5D —
a discrete stack of 2D sheets — gives the useful capability without the
geometry. Recorded in the workbench.

## Related

- `../foundations/vocabulary.md`
- `../runtime/runtime-model.md` — navigation as Action and Action Strategy
- `../research/hyprland-ipc.md` — the compositor integration surface
- `../workbench/2026/2026-09-conceptual-design.md`
