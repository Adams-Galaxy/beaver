---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - ../foundations/vocabulary.md
  - ../runtime/runtime-model.md
---

# Overlays and the System Overlay Graph

> **This document is an active design exploration, not a specification.**
>
> SOG is not ready to implement. Several load-bearing questions in *Open
> questions* are genuinely unanswered. Anyone reading this and concluding
> "great, I'll build this tomorrow" has misread it.

## Summary

Many parts of Beaver want to contribute desired policy: the system defaults, the
user's dotfiles, the machine, the active Space, an active context, an
application, a temporary manual override. Beaver must compose those
contributions **deterministically and explainably**.

The current model has three separable pieces. An **Overlay** is a coherent named
contribution, divided into per-domain **Facets**. The **SOG** is a directed
acyclic precedence graph that decides *which contributions dominate*. The option
**schema** decides *how surviving contributions combine*.

That last separation is the most important idea here, and the one most likely to
survive whatever else changes.

## Motivation

The naive approach — later file wins — produces configuration archaeology. Three
years in, nobody can say why a value is what it is.

The next approach — numeric priorities — is worse in a subtle way. `priority = 862`
tells you nothing. It carries no semantics, invites escalation wars, and every
value in the system eventually acquires a number chosen by vibes.

What we actually want is precedence with *meaning*: `focus` outranks
`application` for notifications because do-not-disturb should beat an app's
preference, and that sentence should be visible in the model.

## Model

### Overlay

A named contribution of desired policy.

```text
Overlay: deep-work
    notifications.mode = silent
    appearance.accent  = ...
    power.profile      = performance
```

An overlay **describes state**. It does not run scripts, dispatch compositor
commands or perform actions:

```text
overlay says:      notifications.mode = "silent"
overlay never:     run hyprctl keyword ...
```

Automation is a separate mechanism — an event observer that invokes an action, or
activates an overlay. Keeping description and execution apart avoids an enormous
amount of spaghetti.

### Facet

An overlay may span domains. A Facet is its portion within one domain.

```text
Overlay: deep-work
├── appearance facet
├── notification facet
└── power facet
```

The overlay stays one semantic object — `beaverctl overlay inspect deep-work`
shows the whole bundle — while each facet participates in its own domain's
precedence.

### Facet placement should be mostly automatic

Users must not write this constantly:

```text
deep-work.appearance    → appearance.focus.normal
deep-work.notifications → notifications.focus.normal
deep-work.power         → power.focus.normal
```

Instead an overlay declares a **role**, and each domain schema knows where a
contribution of that role enters its limb:

```text
Overlay role  +  Option domain  →  default attachment point
```

So this:

```toml
[deep-work]
role = "focus"
appearance.accent = "..."
notifications.mode = "silent"
```

routes itself. The schema already knows `appearance.accent` is an appearance
option and a `focus`-role contribution attaches at `appearance.focus`.

The escape hatch is explicit and structural:

```text
deep-work                → focus.normal          (default)
deep-work.notifications  → notifications.focus.strong
```

The exceptional choice is visible as a deliberate act, not hidden beside an
individual value.

### SOG — the precedence graph

```text
A ─────► B
```

means a contribution at B outranks a contribution at A. Not "B appears later in
a file" — actual semantic precedence.

```text
       B

A             C
```

No path between B and C means they are genuinely **incomparable**. Beaver must
not secretly pick one.

A **ladder** is simply a chain-shaped SOG:

```text
system → user → space → focus → application → manual
```

Most users think in ladders. That should remain the ergonomic default syntax
while the model underneath is a graph.

### Domain limbs

The graph branches because different domains legitimately want different
orderings:

```text
                         system
                            │
                            ▼
                          user
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
     appearance       notifications          power
          │                 │                  │
        space             space              space
          │                 │                  │
        focus           application          focus
          │                 │
    application           focus
          │                 │
        manual            manual
```

Illustrative only. Note that `application` and `focus` are deliberately swapped
between appearance and notifications — an app may reasonably influence its own
appearance, but do-not-disturb should beat an app's notification preference.

Users must be able to modify this: insert a `meeting` node, add a domain, add a
`battery-emergency` node that dominates almost everything under `power`, or make
two nodes deliberately incomparable. The validator rejects cycles.

### Resolution

```text
            active context
                  │
                  ▼
        applicable contributions
                  │
                  ▼
              SOG precedence
                  │
                  ▼
      maximal applicable definitions
                  │
                  ▼
       schema merge semantics
                  │
                  ▼
          effective policy
```

A definition is non-maximal if another applicable definition transitively
outranks it. What remains is the maximal set, which the option's own merge
semantics then combine.

### Merge semantics belong to the option

```text
Scalar<T>       one winner required from the maximal set
Set<T>          union
OrderedSet<T>   merge, with separate ordering rules
Map<K,V>        resolve recursively per key
Custom<T>       domain-defined operation
```

Configuration stops being "TOML values smashed together" and becomes a typed
composition system.

This is why equal precedence is not automatically an error. The rule is not
*"equal-rung contributions are invalid"*; it is **"equal-rung contributions must
be resolvable by the option's merge semantics."** Two contexts both setting
`notifications.mode` at equal precedence is ambiguous. Two overlays both
contributing command providers at equal precedence is perfectly normal.

### Merge ordering is a third, separate dimension

```text
A contributes:  ["git", "files"]
B contributes:  ["shell"]
```

SOG may say both survive. It should not also decide whether `git` comes before
`shell`. Contributions can carry ordering constraints — `before` / `after`
anchors — independent of whether they survive.

Three orthogonal questions, and conflating any two of them is how these systems
rot:

```text
SOG              which contributions dominate?
Schema type      how do survivors combine?
Merge ordering   in what order?
```

### Ambiguity is a feature

```text
notifications.mode

system.default      = normal
space.robocup       = normal
application.teams   = urgent-only
focus.deep-work     = silent
```

If `application → focus` in the notifications limb, `deep-work` dominates Teams
and the answer is `silent`.

But if `space` and `focus` were deliberately incomparable and both are maximal:

```text
AMBIGUOUS POLICY

notifications.mode has two incomparable maximal definitions:
    space.robocup    = normal
    focus.deep-work  = silent
```

This is not Beaver failing. It is Beaver correctly reporting that the user's own
precedence graph does not decide the question. Silent last-write-wins is the
thing we are trying to eliminate.

### Introspection is not optional

The danger of a graph is not computational complexity. It is **human
invisibility**. If we build SOG, provenance tooling ships with it, not after it.

```text
beaverctl policy explain notifications.mode
```

```text
Effective:
    notifications.mode = silent

Provided by:
    deep-work.notifications

Overlay:  deep-work
Facet:    notifications
Attached: notifications.focus

Dominates:
    teams.notifications
    robocup.notifications
    user.notifications

Source:
    ~/.config/beaver/focuses/deep-work.toml:17
```

Plus `beaverctl policy graph [domain] [--dot]` so the graph can be rendered, and
eventually a visual policy inspector in the shell.

## Examples

Deliberately thin. **Stress testing this model against five or six ugly,
realistic configurations is the immediate next design task** — see
`../STATUS.md`. Candidate scenarios:

- Multiple simultaneously active contexts
- Do-not-disturb versus an application exception
- Space appearance versus application appearance
- Monitor-specific behaviour
- A temporary manual override with a lifetime
- An extension contributing defaults

If those keep forcing us to invent exceptions, SOG is wrong. If they reduce
naturally to facets, precedence and merge algebra, the model has earned its keep.

## Invariants

Provisional — these hold only if the model survives stress testing.

- An overlay describes desired state. It never performs actions.
- Precedence is a partial order. Incomparable maximal definitions are reported,
  never silently resolved.
- The graph is acyclic; cycles are a validation failure.
- Precedence, merge semantics and merge ordering are three separate concerns and
  must not be collapsed into one mechanism.
- Merge behaviour belongs to the option's schema, not to the contributor.
- SOG resolves which policy is effective. It does not execute domain logic.
- Every effective value must be explainable — source, overlay, facet, attachment
  point, and what it dominated.

## Open questions

The genuinely unresolved ones. This list *is* the current state of the design.

- **Graph topology.** One universal graph, domain-branched limbs, or independent
  per-domain graphs? Domain limbs currently look best but are unproven.
- **Facet routing.** Is role-based automatic attachment sufficient? What happens
  when an overlay contributes to a domain that has no attachment point for its
  role?
- **Ambiguity: warn or fail?** A warning risks being ignored; a hard failure
  risks an unusable desktop from a small mistake. Possibly it varies by domain,
  or by whether a fallback exists.
- **User-defined graph structure.** How much may a user restructure, and what
  guards against them producing something incomprehensible?
- **Manual and transient overrides.** What does a temporary override mean? Does
  it expire? On what — time, context change, restart? Where does it live given it
  is state rather than configuration?
- **How graph-shaped is the user-facing surface?** We could present ladders and
  convenience constructs while compiling to a DAG internally. Linux users can
  handle a graph, but they should not have to write graph theory to change an
  accent colour.
- **Conditional policy.** The Slack example — *"may interrupt at critical
  urgency"* — is a rule, not a value. Does the notification schema support
  declarative matching, or does SOG resolve a named policy object that a pipeline
  stage evaluates? Currently leaning to the latter.
- **Where does the schema live**, and can extensions extend it?
- **Interaction with the effective runtime.** What triggers recomputation, and is
  it whole-world or incremental?

## Alternatives considered

**Numeric override priorities.** Nix's approach, rejected for Beaver in favour of
a semantic DAG — but the *split* Nix makes between override priority and merge
ordering, and its insistence that **option types define merging**, are both
adopted here. The single most valuable borrowed idea in this document.
→ `../research/nixos-module-merging.md`

**A single universal linear ladder (the original SOL).** Rejected: different
domains legitimately want different orderings, and forcing one order makes either
do-not-disturb or application theming wrong.

**Per-property priority annotations.**

```toml
notifications.mode.priority = 879
appearance.radius.priority  = 813
```

Rejected. This is where the swamp begins. Facet-level attachment keeps the
exceptional case structurally visible. CSS's `!important` is the cautionary
precedent — an escape hatch that exists because the layered model could not
express what authors needed. → `../research/css-cascade.md`

**"Portals" for distributing precedence across domains.** The need was real; facet
routing appears to satisfy it without another primitive. Also collides with
`xdg-desktop-portal`.

**Using SOG for behavioural composition too.** Rejected. A setting wants one
value; a command surface wants ten providers at once; a pipeline wants ordered
stages. Different mathematics. SOG still participates by resolving *which*
strategy is selected — `behavior.navigation.vertical = "paired"` is an ordinary
policy value — after which the behaviour registry defines how it works.

**systemd-style lexicographic drop-ins.** Understandable and predictable, but too
primitive: it cannot express that do-not-disturb beats an application preference.
`10-`/`20-` filename prefixes are also numeric priority wearing a different hat.
→ `../research/systemd-drop-ins.md`

## Related

- `../runtime/runtime-model.md` — SOG chooses *which* strategy; the registry
  defines how it works
- `../foundations/vocabulary.md`
- `../research/nixos-module-merging.md`, `../research/css-cascade.md`,
  `../research/systemd-drop-ins.md` — the three precedents behind this model
- `../workbench/2026/2026-09-conceptual-design.md`
