---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://quickshell.outfoxxed.me/docs/about
  - https://quickshell.outfoxxed.me/docs/guide/faq
  - https://quickshell.outfoxxed.me/docs/configuration/getting-started/
related:
  - ../extensions/extension-model.md
---

# Quickshell Capabilities

> External precedent. Describes Quickshell, not Beaver. Findings unverified — see
> `README.md`. **Quickshell explicitly warns about API breakage.**

## Why this matters to Beaver

Quickshell is the frontend. Its native capabilities determine how much Beaver
should duplicate in Rust, and one specific gap determines how the frontend talks
to the core.

## Findings

### Substantial native integration

Quickshell directly supports Hyprland, PipeWire, MPRIS, UPower, system tray,
notifications, PAM and greetd, alongside general Wayland functionality. Proper
QtQuick UI, animations, Wayland surfaces and hot reload.

The implication is that duplicating all of this in Rust immediately would create
work without producing much. It is a direct argument for the boundary in
`journal/README.md`'s architecture: Quickshell owns presentation and immediate UI
state; the core owns policy, persistent state and orchestration.

### `IpcHandler` is Quickshell's own IPC, not a D-Bus client

`IpcHandler` exposes *Quickshell's instance IPC* — the mechanism behind
`qs ipc call launcher toggle`. It is not a general-purpose client for talking to
an external D-Bus service.

Quickshell links `Qt::DBus` internally and uses it for DBusMenu and its own
service integrations, but no documented generic D-Bus QML client was found in its
public API.

**This is the finding that produced an architectural rule.** Without a QML D-Bus
client, the obvious workaround is:

```qml
Process { command: ["beaverctl", "space", "activate", "robocup"] }
```

scattered throughout the shell — named in the design conversation as precisely
the architectural scar to avoid. The intended destination is a small native QML
bridge that is a typed D-Bus client containing no desktop policy.

### The API is explicitly unstable

Quickshell warns about future API breaks.

This is a second, independent argument for not letting Quickshell's object model
become the canonical model of the desktop. The UI can evolve violently while the
core concepts stay stable.

## What Beaver took

**Adopted:** Quickshell as the frontend, committed to. Presentation state stays
there and is not mirrored in the core.

**Produced a design requirement:** the native QML bridge, so the frontend is a
typed client of the core rather than a shell-out harness.

**Reinforced:** the replaceable-frontend principle. A frontend built on an
explicitly unstable API should not be load-bearing for the project's identity.

## Open

- Verify whether a generic D-Bus QML client has since appeared. It would remove
  the need for a custom bridge.
- What building a Quickshell-loadable native QML module actually involves.
- How Quickshell's own hot reload interacts with the core's transactional
  configuration reload — two independent reload mechanisms in one session.
