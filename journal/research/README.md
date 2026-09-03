---
status: research
last-reviewed: 2026-09-02
---

# Research

External precedent. **These documents describe what *other* systems do.** They
never describe how Beaver works.

That boundary matters: an agent asked to "implement our D-Bus interface" must not
confuse `dbus-and-zbus.md` with Beaver's D-Bus design. Research informs decisions;
it does not record them. The decision lives in the living journal document, and
its *reasoning* usually lives in that document's **Alternatives considered**
section, which links back here.

## Provenance and verification

Most of what is currently here was gathered during Beaver's founding design
conversation (September 2026) and has **not been independently verified**. Each
document carries:

```yaml
verified: no
sources:
  - https://...
```

`verified: no` means the finding shaped a design discussion but should be
re-checked against primary sources before anything is built on it. Upstream
projects move — particularly Hyprland and Quickshell, both of which are actively
changing and one of which explicitly warns about API breakage.

Set `verified: yes` with a date only after checking the primary source yourself.

## Metadata

Research documents use `status: research` rather than the living-document
lifecycle (`exploratory` / `preferred` / `committed` / …), which does not apply
to them. They are never `committed`; a research document has no authority to
commit anything.

## Contents

**Configuration precedence and merging** — the three systems that shaped the
System Overlay Graph:

- `nixos-module-merging.md` — the most influential single source
- `css-cascade.md` — the cascade model, borrowed conceptually, never its syntax
- `systemd-drop-ins.md` — the understandable-but-primitive end of the spectrum

**Platform integration:**

- `hyprland-ipc.md`
- `quickshell-capabilities.md`
- `dbus-and-zbus.md`

**Extension mechanics:**

- `embedded-scripting-runtimes.md` — mlua, PyO3, Rust ABI constraints
- `desktop-shell-precedent.md` — DankMaterialShell, Caelestia, VS Code
