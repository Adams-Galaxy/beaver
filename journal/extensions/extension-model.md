---
status: exploratory
implementation: none
last-reviewed: 2026-09-02
related:
  - ../runtime/runtime-model.md
  - ../foundations/vocabulary.md
---

# Extension Model

## Summary

Beaver has three extension surfaces, each suited to a different kind of work:
**Lua** in-process for intimate behavioural extension, **D-Bus** out-of-process
for rich extensions in any language, and **QML** for presentation.

There is deliberately no `Module` runtime abstraction. Beaver exposes specific
extension points — register an action, a strategy, a provider, a pipeline stage;
subscribe to events; contribute configuration schema; expose a capability — and
a thing casually called a "module" is just a bundle of those.

Critically, **an external extension does not require Beaver to manage its
lifecycle.** An ordinary Python program plus the SDK plus D-Bus already works.
Managed extension infrastructure is a later convenience that changes nothing
about the semantic API.

## Motivation

"Highly modular" invites two bad reflexes. The first is defining a universal
plugin trait everything must implement, which forces unrelated functionality
through one shape. The second is `dlopen`-ing shared libraries, which Rust is
poorly suited to and which makes every extension a potential desktop crash.

We also do not want two languages competing for the same niche. Lua and Python
are both attractive; if they overlap, users face a pointless choice and we
maintain two paths to the same capability.

## Model

### Three surfaces

```text
            In-process            Out-of-process
           ─────────────         ────────────────

Logic          Lua                   Python
                                     Rust
                                     C++
                                     anything on D-Bus

UI             QML

Core           Rust
```

Each language has a clear job:

- **Lua** — Beaver's intimate behavioural language. Hot, in-process, deeply
  integrated, Beaver-specific. Strategies, event observers, small automation.
- **External services** — first-class integration and automation. Isolated,
  language-agnostic, restartable, dependency-free from Beaver's perspective.
- **QML** — presentation extension, owned by the frontend.

Lua is not "the worse Python option." It occupies a niche Python should not:
in-process behaviour on interactive paths.

### Extension points, not modules

```text
register action
register action strategy
register provider
register pipeline stage
subscribe to event
contribute configuration schema and defaults
expose capability
```

A "Git extension" might be:

```text
git-extension/
├── extension.toml
├── git.lua
└── shell/
    └── GitWidget.qml
```

Its Lua registers a command provider, some actions, and a Space-change observer.
Its QML provides a widget. Its TOML provides defaults and schema. At runtime
Beaver sees registered capabilities — no mystical `Module` object required.

### Beaver does not need to know about every QML component

If someone writes `MyClock.qml` that reads public state and renders something,
`beaverd` has no business knowing it exists. Quickshell owns presentation. The
core becomes involved only when a component needs desktop semantics, at which
point it uses the ordinary API.

> **UI composition belongs to the frontend unless it needs desktop semantics.**

This keeps `beaverd` from becoming a registry of visual widgets.

### External extensions are ordinary programs

The baseline model, and the preferred one:

```python
from beaver import Beaver

desktop = await Beaver.connect()

print(await desktop.spaces.current())
print(await desktop.windows.all())

await desktop.actions.invoke(
    "beaver.space.activate", {"name": "robocup"}
)
```

An ordinary Python project with its own `pyproject.toml` and its own venv,
`beaver-sdk` installed like any dependency. Run it with `python -m mytool` or
under `systemd --user`. **No extension host involved.**

This also makes Beaver scriptable in a way that is genuinely attractive: a short
script can pull and inspect the full desktop state programmatically.

Planned SDKs: `beaver-sdk-rs`, `beaver-sdk-python`, `beaver-sdk-cpp`. One
versioned semantic API per language.

### Extensions can register back

D-Bus works both directions. An extension can own its own service:

```text
org.beaver.Extension.AdamRobotics1
```

and register providers, actions, capabilities and state with the core. The
command surface can then query a Python search provider without Python living
anywhere near `beaverd`.

### Managed extensions, if they ever earn it

A supervisor would add discovery, installation metadata, activation, process
lifecycle, restart and backoff, health, logs, debugging, permissions and
dependency declaration.

```toml
[extension]
id = "adam.robotics"
runtime = "python"
entrypoint = "robotics.main"

[extension.activation]
events = ["space.entered"]

[extension.capabilities]
provides = ["adam.robotics.projects"]
```

Two things matter about this design:

**It is not language-specific.** Not a Python host — a supervisor that launches
each extension's configured runtime, each in its own process. This avoids the
classic shared-host problem where extension A needs dependency version X and
extension B needs Y, and it gives clean failure boundaries.

```text
beaverd
   └── extension supervisor
          ├── git-extension       (Rust binary)
          ├── robotics-extension  (.venv/bin/python)
          └── fancy-thing         (anything)
```

**The extension stays an ordinary project.** During development you run it
yourself; installed, the supervisor runs it for you. Same code, same SDK, same
D-Bus contract.

None of this is needed initially. It earns its existence when we repeatedly find
ourselves wanting Beaver to discover, start, restart or debug these things.

### Where out-of-process breaks down

```text
Excellent fit
    project indexing, Git integration, task discovery, file search,
    AI integrations, automation, monitor hotplug response,
    long-running workflows, dashboards, state inspection,
    persistence helpers, application integrations

Probably fine
    command search providers, notification classification,
    complex window placement, Space activation hooks

Prefer Lua or Rust
    Super+Up navigation, pointer-following behaviour,
    focus-follow-mouse, rapid window event loops,
    anything invoked constantly,
    anything required for baseline desktop viability
```

The concern on the hot path is not that D-Bus is slow. It is that an interactive
path would then depend on another process, another language runtime's scheduler,
IPC, and that extension's health. That is needless fragility in the one place a
desktop must feel solid.

### Capabilities

```text
beaver.compositor.views
beaver.compositor.monitors
beaver.audio
adam.robocup.projects
```

Adapters and extensions register them. Strategies declare what they require.
Frontends ask what exists and hide UI accordingly — no Bluetooth backend, no
Bluetooth panel.

Deliberately lightweight: semantic feature discovery, nothing more.

## Invariants

- There is no universal `Module` runtime abstraction.
- Extensions do not require Beaver to supervise them. The semantic API is
  identical whether an extension is managed or run by hand.
- An extension crashing must not take down the core.
- Presentation extensions do not register with the core unless they need desktop
  semantics.
- The public contract is the SDK's semantic API, never the transport. A developer
  should never hand-construct a D-Bus message.
- Baseline desktop viability must not depend on any external extension.

## Open questions

- **The Lua standard library.** How batteries-included should it be? The goal is
  Lua feeling closer to Python in power and ergonomics while staying fast. Lua's
  tiny standard library is arguably an advantage here — we get to decide what the
  Beaver environment feels like.
- **Lua execution model.** VM count, isolation between extensions, memory limits,
  instruction budgets, sandboxing. The primitives appear to exist — see
  `../research/embedded-scripting-runtimes.md` — but nothing is chosen, including
  which Lua flavour.
- **Permissions.** Should extensions declare what they may access? Does the core
  enforce it? Not designed at all.
- **Versioning.** How the SDK contract is versioned across three languages, and
  what a breaking change looks like.
- **Discovery.** How Beaver learns an extension exists, before any supervisor.
- **Dangling registrations.** An extension providing an active strategy
  disappears — fallback, error, or refuse to unregister?
- **Distribution and packaging.** Explicitly separated from runtime architecture.
  We can answer *what can extensions do?* before *how are they installed?*
- **Execution classes.** Whether the hot/interactive/background distinction should
  be formally enforced or remain a documented convention.

## Alternatives considered

**Native Rust plugins via `dylib`.** Rejected. Rust has no stable ecosystem ABI
for arbitrary structs and traits crossing a dynamic library boundary. A carefully
versioned C ABI is possible, but brings ABI versioning, unsafe FFI, memory
ownership contracts, panic boundaries, threading rules, lifetime rules, reload
semantics, and a crash in an extension being a crash in the desktop core. That
last one alone contradicts the invariant that the core stays recoverable when
user code fails. Out-of-process extensions give the same power with fault
isolation. → `../research/embedded-scripting-runtimes.md`

**Embedding CPython in `beaverd`.** Technically feasible via PyO3. Rejected
because the natural expression of it is a second binary — `beaverd` and
`beaverd-with-python` — which inevitably develops subtly different lifecycle,
packaging, testing and failure behaviour. Python's version and linking
requirements would also become the daemon's.
→ `../research/embedded-scripting-runtimes.md`

**A dedicated language-specific extension host.** Superseded by the
language-agnostic supervisor above, which avoids shared dependency conflicts and
generalises to every runtime. VS Code's extension host is the precedent for the
isolation benefit; Beaver generalises past its single-runtime shape.
→ `../research/desktop-shell-precedent.md`

**A universal `trait BeaverModule`.** Rejected. Forcing unrelated functionality
through one interface adds ceremony without adding capability.

## Related

- `../runtime/runtime-model.md`
- `../foundations/vocabulary.md`
- `../research/embedded-scripting-runtimes.md`,
  `../research/desktop-shell-precedent.md`, `../research/dbus-and-zbus.md`
- `../workbench/2026/2026-09-conceptual-design.md`
