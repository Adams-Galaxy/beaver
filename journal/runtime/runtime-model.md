---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - ../foundations/vocabulary.md
  - ../domain/spatial-model.md
  - ../policy/overlays-and-sog.md
  - ../extensions/extension-model.md
---

# Runtime Model

## Summary

Beaver's runtime is built from six primitives with deliberately different
composition semantics: **Event** (a fact), **Action** (an intent), **Action
Strategy** (a selectable implementation), **Provider** (additive), **Pipeline**
(ordered), and **Capability** (discoverable facility).

The central refusal is that Beaver is **not "a massive event system where user
callbacks mutate whatever they want."** That offers maximum theoretical control
and minimum long-term comprehensibility. Events report; Actions mutate; anything
wanting to change the desktop goes through an Action.

The central performance idea is that the elaborate parts of the architecture —
configuration cascade, overlay resolution, Lua registration, capability
availability — **collapse into an effective runtime ahead of interaction**. The
hot path is a lookup and a call.

## Motivation

Three separate problems drove this model.

Once configuration can be overlaid and behaviour can be replaced, there must be
somewhere principled to put each kind of customisation. Without that, Lua becomes
the universal screwdriver and every extension point becomes a callback.

Different things genuinely compose differently. A theme colour wants one winner.
A command surface wants ten providers at once. A notification handler wants
ordered stages. Forcing all of these through one generic hook mechanism is what
creates callback soup.

And a desktop must feel immediate. Extreme configurability and a snappy
`Super+Up` are only compatible if the complexity is paid somewhere other than the
keypress.

## Model

### The three planes

```text
                         USER
                          │
             TOML         │          Lua
               │          │           │
               ▼          ▼           ▼
┌─────────────────────────────────────────────────┐
│                  POLICY PLANE                   │
│  overlays / SOG        behaviour selections     │
│  contexts              rules / automation       │
└─────────────────────────┬───────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────┐
│                 CONTROL PLANE                   │
│                    beaverd                      │
│  domain state      actions       events         │
│  capabilities      persistence   reconciliation │
└─────────────┬──────────────────────────┬────────┘
              │                          │
              ▼                          ▼
         Hyprland                  Linux services

┌─────────────────────────────────────────────────┐
│               PRESENTATION PLANE                │
│  Quickshell shell, command surface, widgets     │
│         all clients of Beaver capabilities      │
└─────────────────────────────────────────────────┘
```

Configuration, behaviour and presentation stop being the same thing. That
distinction is load-bearing throughout.

### Event — a fact

```text
monitor.connected
window.opened
space.activated
notification.received
configuration.changed
```

An event states that something happened. It is immutable, broadcast, and confers
no authority to change anything.

Lua may observe events freely. To *change* something in response, it invokes an
Action.

```text
Event → Lua observer → Action → Beaver
```

### Action — an intent

An Action is the one controlled path through which canonical state is mutated.
That single path is what lets validation, permission checks, logging, state
transition, reconciliation, D-Bus signalling, persistence and error reporting
live in exactly one place.

```text
ActionDefinition
    id
    parameter schema
    description
    visibility:  internal | api | user
    exposure:    command_surface? dbus? lua?
    default handler
```

Identifiers are rooted at the authority that owns the contract — `beaver.` for
core, an extension's own root otherwise. See
`../foundations/vocabulary.md#naming-and-namespaces`.

Visibility metadata matters. `beaver.core.reconcile_internal_state` should never
appear in a command palette; `beaver.space.activate` is an excellent candidate.
The command surface asks for *discoverable* actions rather than spelunking the
entire API.

### Action Strategy — a selectable implementation

```text
                  Action
                    │
             Strategy lookup
                /        \
        Rust default    Lua / external override
```

The built-in Rust implementation is simply the default registered strategy.
There is no privileged internal path — configurability is inherent rather than
bolted on.

**Not every Action needs a Strategy.** `audio.volume.set(0.5)` means exactly
that; inventing a strategy for it is ceremony. Actions that carry a real policy
question deserve one:

```text
navigation.vertical
monitor.reflow
window.place
space.restore
```

### Ingress versus Event

A keybinding is not an event that happens to trigger an action. It is an **action
ingress**:

```text
keyboard
   │
   ▼
Hyprland / input layer
   │  binding: Super+Up → beaver.navigation.vertical(-1)
   ▼
Action invocation
   │
   ▼
Action dispatcher
   │
   ▼
Effective strategy
   │
   ▼
Domain operation → adapters → state change → events
```

We should not manufacture an event purely so it can immediately become an action.

```text
Ingress   asks Beaver to do something
Event     tells interested parties something happened
```

### Provider — additive

Command surface sources: applications, files, windows, Spaces, actions,
calculator, shell, user-defined. Providers contribute candidates. No provider
overrides another; there is nothing to resolve.

A provider chooses what it is willing to expose. Many internal actions are not
meaningful, or are deliberately private.

### Pipeline — ordered

```text
incoming notification
       │
       ▼
   normalise
       │
   enrich metadata
       │
    classify
       │
  user stage (optional)
       │
  apply effective policy
       │
     route
       │
   ┌───┼────────┐
   ▼   ▼        ▼
 popup history discard
```

Users may inject stages, and those stages have explicit ordering semantics rather
than arbitrary registration order.

### Capability — discoverable facility

```text
beaver.compositor.views       registered by the Hyprland adapter
beaver.compositor.monitors
beaver.audio                  registered by the PipeWire adapter
adam.robocup.projects         registered by an external extension
```

A strategy declares what it requires. A frontend asks whether something exists
and hides UI when it does not. That is the whole feature for now — no dependency
solver, no capability marketplace.

### Observed state versus desired policy

```text
ObservedState  +  DesiredPolicy
              │
              ▼
          Reconciler
              │
              ▼
           Actions
```

Hyprland reports that DP-1 is connected at 165 Hz. A Space says DP-1 *should* be
primary. The daemon reconciles them.

This is what makes reconnection, restart and crash recovery ordinary rather than
special: unplug a monitor and the policy stays valid; restart Hyprland and the
daemon rebuilds observed state and reapplies policy.

### Complexity resolves ahead of interaction

```text
TOML + SOG + active overlays + Lua registrations + capabilities
                          │
                    resolve / validate
                          │
                          ▼
              ┌────────────────────────┐
              │  Effective Runtime     │
              │                        │
              │ navigation.vertical  →  grouped_v2
              │ navigation.horizontal → independent
              │ notification.pipeline → [...]
              │ command.providers     → [...]
              │ appearance.*          → resolved values
              └────────────────────────┘
```

Recomputed on configuration reload, context change, Space change, monitor
hotplug — not on every keypress. Then:

```rust
actions.invoke("beaver.navigation.vertical", args)
```

is a lookup into an already-resolved handler.

A Quickshell button never needs to understand SOG. It invokes an action, and
Beaver already knows what that means *here, now*.

## Examples

### Notification arrival

```text
Freedesktop ingress
   → notification.received event
   → notification pipeline
   → effective policy consulted
   → routed to popup / history / discard
```

With `Space = robocup`, `Focus = deep-work`, `App = Slack`, precedence resolves
the notification policy. Note what this exposes: a rule like *"Slack may
interrupt at critical urgency"* is **conditional policy, not a value**. SOG
resolves *which policy is effective*; a pipeline stage evaluates it. SOG must not
become a rule engine.

### Monitor hotplug

```text
Hyprland observation
   → observed state updated
   → monitor.connected event
   → (optional automation observes and invokes actions)
   → desired ≠ observed detected
   → monitor.reconcile action
   → effective strategy
   → assignments and layout applied
```

The built-in reconciliation path works with no Lua present. Automation is
additive.

### Command surface

```text
query
   │
routing:  ">" → shell      "=" → calculator      none → default set
   │
providers queried
   │
aggregation → ranking
   │
   ▼
frontend
```

The meaning of `>` belongs to the command *application*, not to `beaverd`. The
core supplies actions, state, providers and Space context; the application
decides how those become a search experience. Someone else's frontend may have no
command palette at all.

## Invariants

- Events describe facts. They do not directly mutate canonical state.
- Actions express semantic intent and are the only path to canonical state
  mutation.
- Lua must not receive direct write access to canonical state. `beaver.state.x = y`
  is not a supported shape; `actions.invoke(...)` is.
- Action Strategies are replaceable implementations of behavioural contracts. The
  built-in Rust implementation is the default registered strategy and enjoys no
  privileged path.
- Strategy and policy resolution occur outside interaction hot paths wherever
  possible.
- The core must remain recoverable when user behaviour code fails. A failed
  strategy falls back to the built-in implementation and reports the error; it
  does not take the desktop with it.
- Keybindings, CLI, UI surfaces and automation invoke the same semantic actions.
- Presentation state belongs to the frontend. `beaverd` does not track hover,
  scroll position or animation progress.

## Open questions

- **Execution classes.** A concept of `hot` / `interactive` / `background` for
  actions was proposed, where a hot strategy may only be built-in Rust or
  in-process Lua. Useful as a concept; unclear whether it should be enforced.
- **Lua execution model.** One VM with isolated environments, several VMs, or
  per-extension states. Instruction budgets? Preemption? What happens to
  registered strategies when Lua reloads?
- **Transactional strategy replacement.** How a reload swaps strategies without
  the desktop being briefly inconsistent.
- **Dangling strategies.** What happens when an external extension providing an
  active strategy disappears.
- Whether `visibility` and `exposure` need to be separate fields or whether that
  is over-modelling.
- Whether the effective-runtime recomputation is whole-world or incremental, and
  what triggers it.

## Alternatives considered

**A universal event bus with mutating callbacks.** Rejected. It is the most
obvious design and produces a system nobody can reason about after a year.

**Requiring every Action to have a Strategy.** Rejected as ceremony. Strategies
exist where there is a genuine policy question.

**A `trait BeaverModule` that all functionality implements.** Rejected. See
`../extensions/extension-model.md`. Registration of specific capabilities is
cleaner than a universal module object.

## Related

- `../domain/spatial-model.md`
- `../policy/overlays-and-sog.md`
- `../extensions/extension-model.md`
- `../research/dbus-and-zbus.md`, `../research/hyprland-ipc.md`
- `../workbench/2026/2026-09-conceptual-design.md`
