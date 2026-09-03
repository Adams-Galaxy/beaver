---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://github.com/AvengeMedia/DankMaterialShell
  - https://github.com/caelestia-dots/shell
  - https://code.visualstudio.com/api/advanced-topics/extension-host
related:
  - ../extensions/extension-model.md
---

# Desktop Shell and Extension-Host Precedent

> External precedent. Describes other projects, not Beaver. Findings unverified —
> see `README.md`.

## Why this matters to Beaver

Two Quickshell-based shells already split a backend from the frontend, which is
the arrangement Beaver is adopting. VS Code is the reference for what a managed
extension host adds over plain processes.

## Findings

### DankMaterialShell — Go core plus Quickshell frontend

DMS separates `quickshell/` from `core/` in a single repository. The backend
handles IPC, system integration and lifecycle. Search was moved into the backend
as `dsearch`.

That last detail was read as significant: a project that started with search in
the frontend moved it to the backend, which is corroborating evidence for putting
search infrastructure in the core rather than the shell.

### Caelestia — CLI as a controller layer

Caelestia's CLI is more of a controller and utility layer around the Quickshell
environment, with shell functions externally accessible via `caelestia shell …`.

A different balance: less a separate desktop core, more a control surface over
the shell.

### VS Code — what an extension host is for

The extension host exists primarily to run extensions **outside the UI process**.
VS Code runs local, web and remote hosts depending on where extension code needs
to execute.

## What Beaver took

**Leaned toward DMS architecturally** — a genuine backend owning IPC, system
integration and lifecycle, with the Quickshell frontend as a client. Beaver goes
further in one respect: the frontend is explicitly replaceable, and the CLI is a
peer client of the core rather than a wrapper around the shell.

**Took the isolation lesson from VS Code**, and then generalised past it. Beaver's
eventual supervisor would not be a language-specific host; it would launch each
extension's own configured runtime in its own process, avoiding the shared-host
problem where extension A needs dependency version X and extension B needs Y.

**Took the ordering lesson:** Beaver does not need a managed extension system to
have extensions. An ordinary program plus the SDK plus D-Bus already works. The
supervisor earns its existence only when we repeatedly want discovery, restart,
logs and debugging.

## Open

- How DMS handles configuration precedence, if at all. Unexamined, and directly
  relevant to the SOG work.
- Whether either project has hit problems with the Quickshell API instability
  that `quickshell-capabilities.md` records.
- What DMS's core/frontend protocol looks like, and whether it is D-Bus.
