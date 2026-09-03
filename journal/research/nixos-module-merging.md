---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://nixos.org/manual/nixos/stable/
related:
  - ../policy/overlays-and-sog.md
---

# NixOS Module Merging

> External precedent. Describes NixOS, not Beaver. Findings unverified — see
> `README.md`.

## Why this matters to Beaver

This is the single most influential source behind the System Overlay Graph. Two
of its ideas were adopted more or less directly; one was deliberately rejected.

## Findings

### Override priority and merge ordering are separate concepts

NixOS keeps two distinct dimensions:

- **Override priority** decides *which definitions survive*. `mkOverride`,
  `mkForce` and `mkDefault` operate here.
- **Merge ordering** decides *the order of values being merged*, without
  affecting whether a definition survives. `mkOrder`, `mkBefore` and `mkAfter`
  operate here.

A definition can be reordered without being promoted, and promoted without being
reordered.

### Option types define merging

Merge behaviour belongs to the **option type**, not to whoever contributed the
value. Some types require all definitions to be equal, some combine them, lists
merge, and custom types may supply their own merge function.

The contributor says *what*; the type says *how it combines*.

### Override priority is numeric

`mkOverride` takes a number; lower wins. `mkForce` and `mkDefault` are named
wrappers around particular values.

## What Beaver took

**Adopted:** the separation of override priority from merge ordering, and the
principle that merge semantics belong to the option schema. Together these are
why `overlays-and-sog.md` treats precedence, merge algebra and merge ordering as
three orthogonal concerns rather than one mechanism.

**Rejected:** numeric override priority. `priority = 862` carries no semantics,
invites escalation, and every value eventually acquires a number chosen by
instinct. Beaver replaces the scalar with a node in a semantic precedence DAG:

```text
Nix:     definition → numeric override priority
Beaver:  definition → node in precedence graph
```

The trade is legibility for a harder resolution problem — and the acceptance that
two definitions may be genuinely incomparable, which a numeric scheme can never
express.

## Open

- How Nix handles the equivalent of Beaver's ambiguity case. A total numeric
  order means ties are possible but incomparability is not; it would be worth
  knowing what Nix does on an exact tie.
- Whether Nix's custom merge functions have an analogue worth copying for
  Beaver's `Custom<T>`.
