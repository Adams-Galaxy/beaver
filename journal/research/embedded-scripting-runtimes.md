---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://docs.rs/mlua/latest/mlua/
  - https://docs.rs/crate/mlua/latest/builds
  - https://pyo3.rs/main/building-and-distribution
  - https://doc.rust-lang.org/stable/embedded-book/interoperability/rust-with-c.html
related:
  - ../extensions/extension-model.md
---

# Embedded Scripting Runtimes and the Rust ABI

> External precedent. Findings unverified — see `README.md`.

## Why this matters to Beaver

Three separate questions converge here: what can Lua actually do in-process, is
embedding Python viable, and can Rust extensions be dynamically loaded. Together
they determined the three-tier extension model.

## Findings

### mlua is more capable than "config scripting"

Reported support spans Lua 5.1 through 5.5 plus LuaJIT and Luau, with:

- async Rust function integration
- **separate Lua VM instances**
- memory limits
- instruction hooks
- sandboxing, specifically under Luau
- clean mapping of serde-compatible Rust types into Lua values

The last four matter disproportionately. Fault isolation was an early concern —
`while true do end` must not cost the user their desktop — and memory limits plus
instruction hooks are the mechanisms that make an execution budget possible.
Separate VM instances mean per-extension isolation is available if wanted.

Lua's small standard library is arguably an *advantage*: it means Beaver decides
what the extension environment feels like, and can make it far more
batteries-included than bare Lua without it becoming Python-sized.

### PyO3 can embed CPython

Embedding is supported both dynamically and statically. On Fedora, dynamic
embedding relies on the Python development and shared-library environment.
Distributing an embedded interpreter introduces version and linking
considerations that Lua largely avoids.

So embedding is *feasible*. It was rejected on architectural rather than
technical grounds — see `../extensions/extension-model.md`.

### Rust has no stable ecosystem ABI

Rust's compilation and linkage model does not provide a stable ABI for arbitrary
structs and traits crossing a dynamic library boundary the way a fixed C ABI
does. The documented interoperability path is a C-compatible API, with `cdylib`
as the relevant crate output.

A versioned C ABI for Beaver extensions is therefore possible, at the cost of:

```text
ABI versioning          unsafe FFI
memory ownership        panic boundaries
threading rules         lifetime rules
reload semantics        extension crash = core crash
```

## What Beaver took

**Adopted:** Lua in-process as the intimate behavioural language, with the
isolation primitives above as the intended basis for fault containment.

**Rejected:** native Rust `dylib` plugins. The last item on that list — an
extension crash taking down the desktop core — is disqualifying on its own for a
system whose stated invariant is that the core stays recoverable when user code
fails. Out-of-process extensions give the same capability with fault isolation.

**Rejected:** embedded CPython, despite being viable, because the natural
expression is a second daemon binary that inevitably diverges in lifecycle,
packaging, testing and failure behaviour.

## Open

- Which Lua flavour. Luau's sandboxing is attractive and its language differences
  from stock Lua are not currently assessed. LuaJIT's performance may be
  irrelevant at desktop-policy workloads.
- Whether Beaver runs one VM with isolated environments or several VMs.
- What an instruction budget should actually be, and what happens when it is
  exceeded mid-strategy.
- Whether registered Lua strategies survive a configuration reload, or are torn
  down and re-registered.
