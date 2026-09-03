---
status: workbench
authoritative: false
last-reviewed: 2026-09-02
absorbed-into:
  - ../../foundations/design-map.md
  - ../../foundations/vocabulary.md
  - ../../domain/spatial-model.md
  - ../../runtime/runtime-model.md
  - ../../policy/overlays-and-sog.md
  - ../../extensions/extension-model.md
---

# Conceptual Design Notes — September 2026

> **Not authoritative.** This is preserved reasoning from Beaver's founding
> design conversation, including ideas that were rejected and ideas that remain
> merely interesting. Current design lives in the living journal documents listed
> in `absorbed-into`. Do not derive architecture from this file.

This is a thematic distillation, not a transcript.

External findings cited during this conversation — Nix, CSS, systemd, Hyprland,
Quickshell, D-Bus, mlua, PyO3, DMS, Caelestia, VS Code — have been extracted into
`../../research/`, where they are marked unverified and carry their sources.

---

## Product framing

The project began as "a Hyprland and Quickshell setup with maybe a Rust daemon"
and was deliberately reframed upward almost immediately:

```text
Hyprland       → compositor
Quickshell     → renderer / shell frontend
Rust daemon    → desktop core
Linux services → platform
The project    → desktop environment
```

The reframing matters because it changes what the repository is allowed to
become. "My Hyprland config" accumulates scripts. A desktop environment
accumulates architecture.

The intended lifespan is on the order of a decade — through an engineering
career. Every decision was evaluated against *"will this still be workable in
2032?"* rather than *"can I ship a bar this weekend?"*

### The dual-session migration

```text
SDDM
├── Plasma          known-good rescue boat, untouched
└── Beaver session  progressively more habitable
```

Plasma stays until, one morning, it has not been used for three months. This is
why `beaverd` is session-scoped rather than a permanently running user daemon.

### The three-layer separation

```text
Your environment    dotfiles, personal theme, bindings, Spaces, machine config
Beaver              core, compositor integration, shell, overlay machinery,
                    actions, configuration system, sensible defaults
Linux               Hyprland, PipeWire, BlueZ, NetworkManager, systemd, ...
```

The standard this sets:

> Installing Beaver gives you a complete desktop. Installing Beaver plus your
> dotfiles gives you *your* desktop.

Not approximately your desktop. Your universe comes back.

This exists to prevent a specific, very common failure mode: the author's
personal configuration slowly fossilising into the application's architecture.
Eventually this may be physical — a `beaver/` product repository and a separate
`dotfiles/` repository.

### Product character

Quietly powerful. Visually restrained, functionally deep. The default is a top
bar, tiled applications, wallpaper — no dock, because a command surface replaces
it. Depth is summoned.

```text
┌─────────────────────────────────────────────────────────┐
│ Space / views                             system status │
├─────────────────────────────────────────────────────────┤
│                                                         │
│                    tiled applications                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Contextual awareness was explicitly deferred: the desktop should be *lived in*
for months before deciding how it should understand how its user works. The
requirement is only that we do not architecturally shut ourselves out of it.

A distinction that came up and is worth keeping. A desktop can be smart in an
annoying way — *"I moved your window because I thought you'd like that"* — or
smart in an engineering-tool way — *"here is the state I inferred, here is the
policy being applied, and you can override it."* Beaver wants the latter.

### Vocabulary philosophy

The name comes from how beavers build and reshape their environment to suit how
they work.

Beaver-flavoured nouns were considered — lodge, pond, bank, channel, dam — and
set aside. Nobody should need a zoology glossary to open Firefox. The project
name carries the personality; the basic nouns stay legible.

---

## Spatial exploration

The tmux analogy drove this from the start: sessions and structured panes as
first-class, persistent citizens, but for the desktop.

### Naming

The top-level object was initially "Beaver Workspace" and renamed to **Space**
because the collision with Hyprland's concrete `workspace` would have produced
permanent friction — `workspace.view.hyprland_workspace_id` and a lifetime of
explaining which workspace is meant.

### The 2.5D question

Two monitor models were wanted: monitors sharing one spatial graph, and monitors
each having their own. Rather than two mechanisms, a third axis unified them.

Genuine arbitrary 3D navigation was floated and set aside. 2.5D — a discrete
stack of 2D sheets — gives the capability without the geometry.

The constraint imposed on the idea: **zero tax until you use it.** A user with
one Plane must never encounter `z`. Beaver is not to be advertised as the world's
first desktop hypercube.

### Persistence

Defined as **restorable desktop intent** rather than process resurrection.
Beaver can recreate structure, directories, overlays and which applications
belong where. It cannot checkpoint Firefox's memory or an arbitrary CAD
application's unsaved state, and should not pretend to.

Levels identified:

```text
Space metadata      names, structure, directories, overlays
Launch intent       which applications belong where
App restoration     best-effort, application-specific
Live runtime        current actual windows — not persisted
```

The **definition versus instance** distinction emerged here and remains
unresolved: dotfiles define a Robocup *template*; the running system holds a
Robocup *instance* remembering what happened while it was used.

### Config / state / runtime

```text
~/.config/beaver/       configuration   reproducible, declarative, in dotfiles
~/.local/state/beaver/  state           persistent, machine-local, not config
$XDG_RUNTIME_DIR/beaver/ runtime        ephemeral observations
```

Paths illustrative. The conceptual division is the point, and it is why Space
*state* does not belong in dotfiles while Space *templates* do.

---

## Runtime and behaviour exploration

### The founding refusal

An early framing was "is Beaver just a massive event system where users plug in
their own Lua callbacks?" Rejected as offering maximum theoretical control and
minimum long-term comprehensibility.

The alternative that emerged separates three things that a callback system
conflates:

```text
Event      a fact             "this happened"
Action     an intent          "do this"
Strategy   an implementation  "here is how that is done"
```

The `Super+Up` walkthrough is what made this click. The binding should not mean
`hyprctl dispatch workspace +1`, nor "call Lua callback 37." It means
`navigation.vertical(-1)`, and *how* that behaves is a separately resolvable
question.

### Ingress versus event

A refinement worth recording: a keypress is not an event that becomes an action.
It is an **action ingress**. We should not manufacture an event purely so it can
immediately become an action.

### The extension-point taxonomy

Stress-tested against roughly twenty real desktop behaviours. It held up, with
one important finding: **not every Action needs a Strategy.**

```text
Super+Up                        Action → Strategy
Activate a Space                Action → Strategy → state transition
Monitor connected               Event → automation → Actions
Application search              Providers
"> " shell search               Provider
Notification arrives            Event + Pipeline + policy
Do-not-disturb                  policy value
Window placement                Event + Strategy
Volume up                       Action, no Strategy needed
Screenshot                      Action
Wallpaper                       policy value
Space restore                   Action → Strategy
Ranking search results          Pipeline
Custom status widget            frontend only, no daemon involvement
Bluetooth availability          Capability
Lua reacts to battery < 15%     Event → Lua → Action
Change navigation algorithm     Strategy replacement
```

### Authority model

```text
Linux services   physical and system truth
beaverd          desktop semantics
Quickshell       presentation state
Lua              user-defined policy and logic, via defined capabilities
```

### The performance principle

> **Beaver resolves complexity ahead of interaction.**

Configuration cascades, Lua registrations, active Spaces and contexts, monitor
topology and extension availability may be complicated. Once resolved, `Super+Up`
travels a short, boring path. This is what makes extreme configurability and a
solid-feeling desktop compatible rather than opposed.

### Lua boundaries

Lua must not directly mutate canonical state. `beaver.state.active_space = "foo"`
is not a supported shape. Everything goes through actions, so that validation,
logging, transitions, reconciliation, signalling and persistence have one home.

Lua owns its own state freely, and Beaver may eventually offer namespaced
persistent extension state.

Fault isolation was flagged early — `while true do end` must not cost the user
their desktop. A failed strategy should fall back to the built-in implementation
and report. This may become one of Beaver's better reliability properties.

---

## Overlay and precedence exploration

### From SOL to SOG

The original proposal was a **System Overlay Ladder**: rungs, each containing
either an unordered set of leaves or an ordered set of nested rungs. Its key
insight — that precedence should have *structure and meaning* rather than
arbitrary numeric priorities — survived entirely.

What broke it was cross-domain ordering. Notifications want
`application < focus` so do-not-disturb beats an app preference; appearance
plausibly wants the reverse. There is no single correct global order, so the
ladder became a **graph**, with a ladder as its simplest shape.

### Three separated concerns

The most durable outcome of this thread:

```text
SOG              which contributions dominate?
Schema type      how do survivors combine?
Merge ordering   in what order?
```

Nix supplied two of these directly: override priority and merge ordering are
separate concepts, and *option types define merging*. Beaver's twist is replacing
Nix's numeric override priority with a semantic precedence DAG.

CSS supplied the shape of the cascade — relevance, then origin and layer, then
specificity and order, with nestable layers. Notably, CSS was borrowed for its
*model*, never its syntax.

systemd supplied the understandable-but-primitive end of the spectrum: vendor
config, drop-ins, lexicographic ordering, scalars last-one-wins, lists
accumulate. Predictable, and unable to express what Beaver needs.

### Ambiguity as a feature

Two incomparable maximal definitions for a scalar produce an error report, not a
coin flip. Silent last-write-wins was identified as one of the largest sources of
configuration archaeology.

The rule was later refined: equal precedence is not automatically invalid. It is
invalid only when the option's merge semantics cannot resolve it. Two contexts
setting `notifications.mode` is ambiguous; two overlays contributing command
providers is normal.

### Facets

Solved the cross-domain problem. One overlay, multiple domain-specific
contributions, each participating in its own limb — without pretending
`deep-work` is three unrelated overlays.

Role-based automatic attachment came out of the requirement that users must not
hand-route every facet.

### What SOG must not become

A rule engine. The Slack example — *"may interrupt at critical urgency"* — is
conditional policy, not a value. SOG resolves *which policy object is effective*;
a pipeline stage evaluates it.

Similarly, SOG is not for behavioural composition. It participates by resolving
which strategy is selected; the behaviour registry defines how that strategy
works.

---

## Extension exploration

The three-tier model settled quickly once the goal was stated as *no language
awkwardly fighting another for the same niche*.

### Why not native Rust plugins

Rust offers no stable ecosystem ABI for arbitrary structs and traits across a
`dylib` boundary. A versioned C ABI is possible but brings ABI versioning, unsafe
FFI, ownership contracts, panic boundaries, threading and lifetime rules, reload
semantics, and extension crashes becoming core crashes. "Dragon husbandry for a
feature we do not currently need."

### The Python question

Embedding CPython via PyO3 is feasible and was genuinely wanted — Python is the
language the author knows best, and programmatic desktop-state inspection from a
script is very attractive.

Rejected as an embedded interpreter because the natural expression is a second
binary, `beaverd` and `beaverd-with-python`, which inevitably diverge in
lifecycle, packaging, testing and failure behaviour, and which drag Python's
version and linking requirements into the daemon's.

Resolved instead by making Python a **first-class external extension language**
with an official SDK. It keeps everything wanted — venvs, arbitrary dependencies,
independent restart, no desktop-killing crashes — and loses only in-process
latency, which does not matter for anything Python should be doing.

An intermediate idea, a dedicated `beaver-python-host`, was superseded within the
same discussion by a **language-agnostic supervisor** that launches each
extension's own configured runtime in its own process. This avoids the shared-host
dependency conflict and generalises.

### The ordering insight

> We can answer *what can extensions do?* before *how are they installed?*

Runtime architecture and distribution are separate concerns, and conflating them
would have produced a package manager before there was anything to package.

---

## Command surface

Reframed early from "launcher" to **command surface**, because launcher
undersells it.

The strongest idea in this thread: **compose with mature Unix tooling rather
than cloning it.** `direnv`, `just`, Git and the user's own shell already express
project context well. A Space with a directory gives it to the command surface
for free.

```text
workspace directory = ~/dev/robocup

> just build     runs in the right place, with the right environment
> git status     does the obvious thing
```

And `>` should mean *enter shell provider mode* using the **configured user
shell**, not *enter Beaver's homemade terminal language*. The user's `.zshrc`,
aliases and functions stay useful. Someone else's fish setup stays useful.

The routing prefixes themselves belong to the command *application*, not the
core. Someone's dotfiles could make `!` the shell prefix and add a Just provider.
Someone else's frontend has no command palette at all.

---

## Ideas deliberately not adopted

**JSON-RPC over a Unix socket as an interim protocol.** Considered first, then
skipped: if we already believe D-Bus is the destination, there is little value in
building an IPC layer we expect to remove.

**COBS framing.** Raised as an alternative to JSON. It solves framing over raw
byte transports, particularly serial links — a problem D-Bus has already solved
for local desktop IPC. Kept in reserve for a hypothetical high-throughput private
channel, alongside length-framed CBOR/MessagePack, `SOCK_SEQPACKET`, or D-Bus FD
passing.

**Quickshell `Process` calls to `beaverctl` as the integration mechanism.**
Explicitly named as the architectural scar to avoid. The intended destination is a
small native QML bridge — a typed D-Bus client containing no desktop policy.

**A universal overlay ladder (SOL).** Superseded by SOG.

**Per-property numeric priority annotations.** "Where the swamp begins."

**"Portal" as a precedence-distribution primitive.** The need was real and is met
by facet routing. Also collides with `xdg-desktop-portal`.

**A `trait BeaverModule`.** Rejected in favour of specific extension points.

**Arbitrary volumetric 3D desktop navigation.** Set aside in favour of 2.5D.

**Numeric directory prefixes in the journal.** `00-foundations/` looks ordered on
day one and becomes a miniature Dewey Decimal System by year five.

**Moving superseded documents to `archive/`.** Stable paths beat a cosmetically
tidy tree.

---

## Interesting deferred ideas

Good ideas with no current home. Preserved so they are not lost.

- **`beaverctl config explain appearance.radius`** — showing the value, its
  source file and line, and every override it beat. This later generalised into
  the SOG provenance tooling, and remains the strongest argument that a
  complicated cascade can be pleasant to use.
- **Widget support** comparable to macOS and the ricing ecosystem. Architected
  for, deliberately not built early.
- **An authenticated phone bridge** invoking the same actions. Mentioned in
  passing; the point is that the architecture does not forbid it.
- **A "personal local operating context service"** — one program that coherently
  knows what you are doing, what project you are in, what windows exist, what
  devices are connected, what state you prefer and what changed. A traditional
  desktop has all of that scattered across dozens of processes. Explicitly *not*
  an AI feature, and explicitly not an early concern.
- **A visual policy inspector** in the shell, showing why a value is effective.
- **`beaverctl policy graph --dot`** for rendering the precedence graph.
- **Interface-level D-Bus versioning** — `Desktop1.Profiles` becoming
  `Desktop2.Profiles` without breaking everything else.
- **Execution classes** (`hot` / `interactive` / `background`) constraining which
  extension surfaces may implement a given strategy.
- **A "lite core"** — running `beaverd` alone and building an entirely custom
  frontend against it.

---

## The original fourteen questions

Preserved because they poked at the right boundaries and shaped everything since.
Now maintained as a living document at `../../foundations/design-map.md`.

```text
 1  Product identity and philosophy
 2  The desktop mental model
 3  Window and workspace philosophy
 4  Shell interaction model
 5  Profiles and context
 6  Configuration philosophy
 7  Modularity and extensibility
 8  The daemon's authority
 9  Actions, events and automation
10  Applications and projects
11  System integration and boundaries
12  Reliability and lifecycle
13  Visual identity and design language
14  Project identity and naming
```

---

## Unresolved at the close of this session

Carried into `../../STATUS.md`:

- SOG stress testing against realistic ugly configurations
- Runtime lifecycle and ownership
- The name for contextual overlays ("Focus" is a placeholder)
- Action namespacing
- Whether the Plane dimension survives contact with real use
