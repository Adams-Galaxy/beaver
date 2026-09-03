---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Cascade/Introduction
related:
  - ../policy/overlays-and-sog.md
---

# The CSS Cascade

> External precedent. Describes CSS, not Beaver. Findings unverified — see
> `README.md`.

## Why this matters to Beaver

CSS is the most widely deployed configuration cascade in existence, and the
closest thing to a natural experiment in what happens when millions of people
compose conflicting declarations for decades.

It was borrowed for its **model**. Its syntax was never on the table.

## Findings

### The cascade is staged, not a single comparison

CSS does not answer collisions with one priority number. It proceeds through
stages:

```text
relevance
    ↓
origin and layer precedence
    ↓
specificity and scoping
    ↓
document order
```

Each stage only runs on what survives the previous one.

### Cascade layers can be nested

Layers are an explicit, author-controlled precedence mechanism, and they nest —
precedence has structure rather than being flat.

### `!important` inverts origin precedence

Worth noting mainly as a cautionary tale. It exists because the layered model
occasionally cannot express what an author needs, and it is widely regarded as
the escape hatch that ate the system.

## What Beaver took

**Adopted:** the idea that precedence should be *staged and structured* rather
than a single numeric comparison, and that layers are a legitimate
author-controlled concept with internal structure.

**The cautionary lesson:** `!important` is what a cascade produces when the model
cannot express a needed relationship. This is a direct argument for two things in
Beaver — letting users restructure the precedence graph itself, and making facet
attachment an explicit structural act rather than a per-value annotation.
Per-property priority annotations were rejected partly on these grounds.

**Not adopted:** specificity. CSS derives precedence partly from how a selector
was *written*, which is implicit and famously surprising. Beaver's precedence is
declared, never inferred.

## Open

- Cascade layers are relatively recent and their interaction with `!important` is
  subtle (important declarations reverse layer order). If Beaver ever grows an
  override-of-last-resort, this interaction is the prior art to study first.
