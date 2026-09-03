---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://wiki.hypr.land/IPC/
  - https://wiki.hypr.land/0.55.0/Configuring/Advanced-and-Cool/Expanding-functionality/
related:
  - ../domain/spatial-model.md
  - ../runtime/runtime-model.md
---

# Hyprland IPC and Configuration

> External precedent. Describes Hyprland, not Beaver. Findings unverified — see
> `README.md`. **Hyprland moves quickly; re-check before building.**

## Why this matters to Beaver

Hyprland is the compositor adapter's integration surface. Two findings shaped
early architecture: the daemon can subscribe to compositor events rather than
polling, and Hyprland's own configuration story has shifted toward Lua.

## Findings

### Two Unix sockets

Hyprland exposes one socket for issuing commands and a second streaming socket
for events, including workspace and window changes.

This is what makes an observed-state model practical. The daemon can maintain a
coherent model of monitors, workspaces, windows and focus by subscribing, rather
than repeatedly shelling out to `hyprctl`.

### hyprlang deprecated in favour of Lua (0.55)

Reported: since Hyprland 0.55, hyprlang has been deprecated in favour of
first-class Lua configuration and scripting, with Hyprland exposing Lua
callbacks, runtime configuration, timers and dispatchers directly.

**This is the single claim here most worth verifying.** It was significant to the
design conversation — it made the choice of Lua as Beaver's extension language
look convergent with the wider ecosystem rather than idiosyncratic.

If accurate, it also raises a boundary question that is not yet designed: Beaver
embeds Lua and Hyprland now runs Lua. Two Lua environments in one session, with
overlapping concerns, needs an explicit story about which owns what.

## What Beaver took

**Adopted:** event subscription over polling, as the basis for the observed-state
half of the reconciliation model.

**Confirmed:** Lua as a mainstream, ecosystem-consistent choice for desktop
configuration and scripting.

**Deliberately not adopted:** Hyprland's data model. `spatial-model.md` treats a
Hyprland workspace as an implementation detail behind a Beaver View precisely so
that Hyprland's rapid movement stays contained inside the adapter.

## Open

- Verify the 0.55 hyprlang/Lua claim against current documentation.
- Where does the Hyprland-Lua / Beaver-Lua boundary sit? Does Beaver own the
  Hyprland config entirely, generate it, or coexist alongside a user-authored one?
- What exactly does the event socket emit, and is it sufficient to maintain
  observed state without periodic reconciliation against `hyprctl`?
