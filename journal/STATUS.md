---
status: preferred
implementation: none
last-reviewed: 2026-09-02
---

# Current Design Status

**Phase: conceptual design.**

No production implementation exists. There is no Rust workspace, no Quickshell
frontend, no D-Bus service and no build system. This is intentional — the
architecture is being designed before the code that would otherwise dictate it.

---

## Working directions

These have survived sustained discussion and are the current basis for further
design. They are not yet committed decisions.

- **Beaver is a desktop environment**, not a Hyprland configuration. Hyprland is
  the compositor, Quickshell is the frontend, a Rust daemon is the desktop core.
- **Personal-first, but never paternalistic.** Beaver ships an opinionated
  default so it works on install, but user configuration determines its
  operational personality. Beaver does not decide how anyone should use their
  desktop.
- **The product and the author's dotfiles are separate things.** Dotfiles are
  Beaver's most demanding consumer, not part of its architecture.
- **D-Bus as the control plane.** Skipping an interim JSON-over-socket protocol
  we already expect to remove.
- **TOML for declarative configuration, Lua for behavioural logic.**
- **Space / Plane / Lane / View** as the spatial model, with a View mapping onto
  a Hyprland workspace.
- **Event / Action / Action Strategy** as the runtime behaviour model.
- **Replaceable frontend.** The shipped Quickshell shell is a client of the core,
  not the core's face. Replacing it entirely is the ultimate escape hatch.
- **Out-of-process rich extensions** over a native Rust plugin ABI.
- **Precedence as a graph (SOG)** rather than a single linear ladder.
- **Identifiers rooted at the authority owning the contract** — `beaver.*` for
  core, an extension's own root otherwise. `system.*` rejected on collision with
  the SOG rung of that name.

---

## Exploratory

Actively being designed. Do not build against these.

- Exact SOG topology — universal graph versus domain-branched limbs
- Facet attachment and routing rules
- Whether ambiguity is a warning or a hard failure
- Terminology for contextual overlays (currently "Focus", unsettled)
- The 2.5D Plane dimension and multi-monitor presentation semantics
- Capability system detail beyond simple feature discovery
- Managed extension supervision (`beaver-extd`)
- Lua execution model — VM count, isolation, budgets, fallback

---

## Not yet substantially designed

Untouched or barely touched. Fair game for the next conceptual rounds.

- Visual language and design system
- Application and project model
- Shell surfaces beyond the command surface
- Reliability, lifecycle and recovery detail
- Persistence — what is state, what is configuration, what is runtime
- Implementation architecture, crate boundaries, build and packaging
- Vision and principles documents (deliberately deferred until the conceptual
  map is complete — we should not write ceremonial documentation before we can
  write it with confidence)

---

## Recently settled

- **Action / event / capability namespacing** (2026-09-02) — `beaver.*` root,
  no short aliases, D-Bus names kept separate. `journal/foundations/vocabulary.md`.
  A strong candidate for the first ADR.
- **Research separated from living design** (2026-09-02) — external precedent
  moved to `journal/research/`, marked unverified with sources.

## Immediate next conceptual work

1. **SOG stress testing.** Five or six deliberately ugly, realistic
   configurations resolved by hand. If we keep inventing exceptions, SOG is
   wrong. If they reduce naturally to facets + precedence DAG + merge algebra,
   we have found something good.
2. **Runtime lifecycle and ownership.** What `beaverd` starts, whether Lua
   environments survive reloads, how strategy replacement is made transactional,
   what happens when an extension vanishes while its strategy is active, how
   D-Bus service discovery maps into capabilities, and what a "lite core"
   actually means.
3. **A full zoom-out** over the fourteen areas in `foundations/design-map.md`.
