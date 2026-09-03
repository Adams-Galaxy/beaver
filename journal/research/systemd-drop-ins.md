---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://www.freedesktop.org/software/systemd/man/latest/
related:
  - ../policy/overlays-and-sog.md
---

# systemd Configuration Drop-ins

> External precedent. Describes systemd, not Beaver. Findings unverified — see
> `README.md`.

## Why this matters to Beaver

systemd represents the understandable-but-primitive end of the configuration
precedence spectrum. It is the model Beaver would get by accident, and it is
worth being explicit about why it is not enough.

## Findings

### Precedence comes from filesystem location

Vendor configuration, local drop-ins and runtime drop-ins form a predictable
hierarchy, with directory location determining which tier a file belongs to.

### Ordering within a tier is lexicographic

Drop-in files are applied in lexicographic filename order — hence the widespread
`10-`, `20-`, `50-` prefix convention.

### Merge behaviour depends loosely on value shape

Scalar settings are generally last-one-wins. List-valued settings accumulate,
with an empty assignment typically used to reset the list.

## What Beaver took

**Adopted:** essentially nothing mechanically, but it confirms two things. Users
genuinely can reason about layered configuration when the layers are few and
named. And the scalar-vs-list distinction arises naturally in any real
configuration system — which is evidence for Beaver's typed merge algebra rather
than against it.

**Rejected:** the model cannot express the relationship Beaver actually needs.
There is no way to say *"do-not-disturb outranks an application's notification
preference, but an application outranks context for appearance."* Precedence
derived from file location and filename is precedence with no semantics, and
`10-`/`20-` prefixes are numeric priority wearing a different hat — the same
problem `nixos-module-merging.md` records rejecting.

It is also the clearest available argument for Beaver's provenance tooling. The
common failure mode with drop-ins is not that resolution is wrong; it is that
nobody can tell *which file won* without going and looking.

## Open

- systemd's list-reset idiom (assigning empty to clear) is a real answer to a
  real problem. Beaver's `Set` and `OrderedSet` types will need an equivalent,
  and it is not currently designed.
