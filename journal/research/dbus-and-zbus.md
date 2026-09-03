---
status: research
verified: no
last-reviewed: 2026-09-02
sources:
  - https://cgit.freedesktop.org/dbus/dbus/tree/doc/dbus-specification.xml
  - https://docs.rs/crate/zbus/latest
related:
  - ../runtime/runtime-model.md
  - ../extensions/extension-model.md
---

# D-Bus and zbus

> External precedent. Describes D-Bus and zbus, not Beaver's interface design.
> Findings unverified — see `README.md`.

## Why this matters to Beaver

D-Bus is the chosen control plane. This records what it gives us for free, and
what the Rust path looks like.

## Findings

### D-Bus supplies the semantic pieces the control plane wants

```text
methods          properties       signals
objects          interfaces       introspection
typed values     service discovery
lifecycle via bus name ownership
```

Hierarchical object paths and versioned interface names give a reasonable
long-lived service boundary.

This is why the interim JSON-over-Unix-socket protocol was skipped: if D-Bus is
the destination, there is little value in building an IPC layer we expect to
remove. It is also why COBS was rejected — COBS solves framing over raw byte
transports, particularly serial links, which is a problem D-Bus has already
solved here.

### Interfaces can be versioned independently of the daemon

The convention of a version suffix on the interface name
(`org.example.Desktop1.Profiles`) means a breaking change to one interface does
not force a version bump on everything else.

### zbus is the Rust path

zbus 5.x supports asynchronous object servers and proxies and integrates with
Tokio when configured to.

## What Beaver took

**Adopted:** D-Bus as the control plane, and per-interface versioning as the
compatibility strategy.

**Adopted as a constraint:** the public contract is the SDK's semantic API, never
the transport. An extension author should write
`beaver.actions.invoke("space.activate", …)` and never hand-construct a message.
This is what keeps the door open to a private high-throughput transport later
without breaking anyone.

**Held in reserve:** for bulk data, if D-Bus ever proves too heavy — length-framed
CBOR or MessagePack over a Unix socket, `SOCK_SEQPACKET`, or D-Bus FD passing.

## Open

- Beaver's actual bus name, object hierarchy and interface split. Not designed.
- Whether dynamic objects (`/org/beaver/Desktop1/Monitors/DP_1`) are worth it, or
  whether flat interfaces with identifier arguments are simpler.
- How D-Bus service discovery maps onto the capability model — an open question
  carried in `../STATUS.md`.
- ~~Whether the reverse-DNS bus name and the action namespace should relate at
  all.~~ **Settled:** they do not. `org.beaver.*` is transport; `beaver.*` is the
  semantic API. See `../foundations/vocabulary.md`.
