---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - ../domain/spatial-model.md
  - ../runtime/runtime-model.md
  - ../policy/overlays-and-sog.md
  - ../extensions/extension-model.md
---

# Vocabulary

## Summary

The canonical glossary. Beaver invents meaningful concepts, and if their
definitions are scattered across fifteen documents, ambiguity will breed.

Each term carries a **status**. This matters more than the definitions
themselves: it tells you which terminology is safe to build APIs, type names and
user-facing surfaces around.

```text
exploratory    working term, expect it to change
preferred      settled enough to use in documentation
committed      locked in, changing it requires an ADR
```

Terms here are brief and canonical. The owning document holds the detail.

**We do not invent names to fill conceptual holes.** Where we do not yet know
what something should be called, this document says so.

---

## Project and components

### Beaver

**Status: committed**

The project. A personal, modular, long-lived Linux desktop environment built
around a programmable desktop core, a reactive shell frontend, and a tiling
compositor.

Distinguish from: the author's dotfiles, which are a *consumer* of Beaver.

### Core / `beaverd`

**Status: preferred** (name exploratory)

The Rust daemon. Owns the desktop model, desired policy, actions, state and
orchestration. Session-scoped, not a permanently running user daemon.

### `beaverctl`

**Status: preferred** (name exploratory)

The command-line client. Architecturally just another client of the core, with
no privileged path.

### Shell / frontend

**Status: preferred**

The Quickshell presentation layer. A client of the core. The shipped shell is
replaceable in its entirety.

---

## Spatial model

Owned by `../domain/spatial-model.md`.

### Space

**Status: preferred**

A persistent, session-like spatial context comprising one or more Planes. May
carry a directory or environment context, may contribute policy through an
overlay, and may retain persistent state.

Distinguish from: a Hyprland workspace. This distinction is the reason we do not
call it "workspace" — the collision would force us to write things like
`workspace.view.hyprland_workspace_id` forever.

Related: Plane, Lane, View.

### Plane

**Status: exploratory**

The `z` axis of a Space. An optional structural dimension. Its main use is
giving each monitor its own two-dimensional sheet within one Space.

Zero tax until you use it — a user with one Plane never needs to know it exists.

### Lane

**Status: preferred**

The `y` axis. A horizontal sequence of Views, vertically stacked with other
Lanes within a Plane.

### View

**Status: preferred**

The `x` axis leaf, and the actual virtual desktop. Contains windows — tiled,
floating or fullscreen.

A View maps onto one Hyprland workspace. That mapping is an adapter concern; the
Hyprland workspace number is implementation detail.

Coordinate: `(z, y, x)` — Plane, Lane, View.

### Monitor

**Status: preferred**

A physical display. Monitors are **viewports onto** the spatial graph, not
owners of it. A Space holds a presentation assignment of `Monitor → View`.

---

## Runtime model

Owned by `../runtime/runtime-model.md`.

### Event

**Status: preferred**

An immutable fact that happened. `monitor.connected`, `window.opened`,
`space.activated`.

An event says *"this happened."* It does not mean *"everybody gets a chance to
rewrite reality now."*

### Action

**Status: preferred**

Semantic intent — a request that something happen. The single controlled path
through which canonical state is mutated, so that validation, logging, state
transition, reconciliation, signalling and persistence all have one home.

Keybindings, the CLI, UI surfaces and automation all invoke the same actions.

### Action Strategy

**Status: preferred**

A named, selectable implementation of a behavioural contract. The built-in Rust
implementation is simply the default registered strategy — there is no
privileged internal path that extensions cannot reach.

Not every Action needs a Strategy. `audio.volume.set(0.5)` means exactly that.
`navigation.vertical` has a genuine policy question underneath it.

### Provider

**Status: preferred**

An additive source of candidates or data. Command surface sources — applications,
files, windows, actions, calculator, shell — are providers. Providers do not
override one another.

### Pipeline

**Status: preferred**

An ordered transformation or routing chain. Notification handling is the
motivating case: normalise, enrich, classify, apply policy, route.

### Capability

**Status: exploratory**

A discoverable semantic facility that exists right now. `beaver.compositor.views`,
`beaver.audio`. Registered by adapters and extensions; declared as required by
strategies; queried by frontends to decide what UI to show.

Deliberately lightweight. Not a dependency solver, not a package manager, not a
module graph.

Distinguish from Action: a Capability says *"I can provide this class of thing."*
An Action says *"do this specific thing."*

---

## Policy and configuration

Owned by `../policy/overlays-and-sog.md`.

### Overlay

**Status: preferred**

A coherent, named contribution of *desired policy*. An overlay describes state —
`notifications.mode = "silent"` — and never performs actions. It does not run
scripts or dispatch compositor commands.

### Facet

**Status: exploratory**

The domain-specific portion of an Overlay. One semantic overlay may contribute
appearance, notification and power facets, each participating in the
corresponding domain's precedence.

This is what allows a single overlay to span domains without being forced to
occupy one universal precedence slot.

### SOG — System Overlay Graph

**Status: exploratory**

The directed acyclic precedence graph. An edge `A → B` means a contribution at
B outranks a contribution at A. Two nodes with no path between them are
genuinely **incomparable**, and Beaver must not secretly pick one.

### Ladder

**Status: exploratory**

A chain-shaped SOG. The simple, ergonomic special case — `system → user → space
→ focus → application → manual`. Most users think in ladders; the model is a
graph.

Historical note: the design began as SOL, a System Overlay *Ladder*, and grew
into a graph when it became clear different domains want different orderings.

### Merge semantics

**Status: exploratory**

How surviving contributions combine, owned by the option's schema rather than by
precedence. `Scalar` selects one winner; `Set` unions; `OrderedSet` merges with
ordering; `Map` resolves per key.

The separation is the point: **SOG decides which contributions dominate; the
schema decides how survivors combine.**

### Observed State

**Status: preferred**

What is actually true right now, as reported by the systems that own it.
Hyprland's real windows, the real connected monitors, the real audio devices.

### Desired Policy

**Status: preferred**

What the user's configuration says *should* be true. Survives the temporary
absence of the thing it describes — a profile referring to a disconnected
monitor remains valid.

The daemon reconciles the two. This is what makes `beaverd` a desktop
*controller* rather than a bag of callbacks.

---

## Naming and namespaces

### Action, event, strategy and capability identifiers

**Status: preferred** — a strong ADR candidate once ADRs begin.

Identifiers are dot-separated. **The root segment names the authority that owns
the identifier's contract** — whoever is entitled to define and change its
semantics.

```text
beaver.space.activate           owned by Beaver
beaver.navigation.vertical
beaver.compositor.views         capability, same rule
adam.robocup.index              owned by Adam's extension
```

Core Beaver identifiers are rooted at `beaver.`. Extensions use their own root
and sit on equal footing — there is no reserved unprefixed namespace and no
second-class citizenship for extension identifiers.

This is deliberately *not* vendor branding. It is ownership, for the same reason
D-Bus uses reverse DNS: the architecture guarantees more than one naming
authority will exist, because that is what the extension model is for.

**`system.` was rejected** because it is already taken. `system` is the lowest
rung of the precedence graph in `../policy/overlays-and-sog.md`, where it means
"vendor defaults, lowest authority." Reusing it for actions would make it mean
"built-in core, highest trust" — a direct collision, and worst of all in the
provenance output where clarity matters most.

**No short aliases, for now.** `beaver.space.activate` has exactly one canonical
form. Ergonomics come from CLI subcommands and namespaced Lua proxies, not from
an alias table:

```text
beaverctl space activate robocup            not the raw id
beaver.space.activate("robocup")            proxy sugar
actions.invoke("beaver.space.activate")     general form
```

An alias mechanism is easy to add later and effectively impossible to remove. It
also cuts against the principle that every resolved value has one explainable
identity.

### D-Bus names are a separate namespace

**Status: preferred**

```text
beaver.space.activate           semantic API — the public contract
org.beaver.Desktop1.Space       D-Bus interface — transport
```

These are related but never the same string, and neither is derived from the
other. `../extensions/extension-model.md` holds that the public contract is the
SDK's semantic API and never the transport; that is what keeps a future private
high-throughput channel possible without breaking every extension.

---

## Terms we have not settled

### "Focus"

**Status: exploratory — placeholder**

Working term for a named, user-activatable contextual overlay with identity and
lifecycle. `deep-work`, `presentation`, `do-not-disturb`.

The name is unsatisfying because "focus" already means window focus. It also
overlaps with "profile" and "mode", both of which were used earlier in design
discussion. **This needs a real decision.** Do not build a public API around this
word yet.

### "Module"

**Status: deliberately not adopted**

Used casually to mean "a bundle of extension contributions." It is explicitly
**not** a runtime abstraction. There is no `trait BeaverModule`. See
`../extensions/extension-model.md`.

### "Portal"

**Status: exploratory — not adopted**

Proposed during design as a mechanism for distributing an overlay's precedence
across domains. The need it identified is real; it appears to be satisfied by
facet routing without introducing another primitive. Recorded so the idea is not
lost.

Note the collision with `xdg-desktop-portal`, which is a strong argument against
the word regardless.

---

## Related

- `design-map.md`
- `../README.md` — how document authority works
