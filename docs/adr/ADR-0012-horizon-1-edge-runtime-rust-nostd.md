---
adr-id: ADR-0012
title: Horizon-1 edge runtime is Rust `no_std`
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/horizon-1-leads", "@zhilicon/firmware-coe", "@zhilicon/security-coe"]
---

# ADR-0012: Horizon-1 edge runtime is Rust `no_std`

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Horizon-1 leads + Firmware COE + Security COE

## Context

The Horizon-1 on-chip runtime must execute on Cortex-R82AE cores under
the following constraints:

- Radiation tolerance (single-event upsets masked by hardware and
  recovered by software).
- Deterministic timing (rate-monotonic scheduling; bounded worst-case
  execution time on every public function).
- No dynamic allocation after boot.
- Sovereign-context enforcement, attestation handshake, OTA verifier,
  telemetry — all non-trivial state machines.

Language candidates:

1. **C** — industry standard for rad-hard. No memory safety. Every
   buffer-overflow or TOCTOU bug is a potential on-orbit fault.
2. **C++** — more abstraction than C. Exception handling and RTTI are
   not compatible with `no_std` / no-runtime targets without
   significant configuration.
3. **Rust** `no_std` — memory safety at compile time; `#![no_std]` is
   an explicit, well-supported configuration; strong story for
   embedded targets via the `cortex-r`, `cortex-m`, and
   `armv7r-none-eabihf` triples.
4. **SPARK/Ada** — formal verification by construction. Tooling cost
   is high; the hiring pool is narrow.

## Decision

The Horizon-1 edge runtime is implemented as a **Rust `no_std`-capable
crate**. Key properties:

- `#![cfg_attr(not(feature = "std"), no_std)]` — compiles cleanly for
  `armv7r-none-eabihf` without the `std` library.
- `heapless` collections + `spin` locks + `critical-section` primitive
  — no `alloc` after init.
- A C ABI surface ([`zhilicon_edge.h`](../../src/edge-runtime/include/zhilicon_edge.h))
  lets the existing C firmware in [`src/firmware/`](../../src/firmware/)
  call into it without a full Rust rewrite.
- Fallback `StubHal` allows host-side tests with `std` enabled — the
  same test code that runs on a developer laptop validates the runtime's
  state machines.

Error handling follows the same `EdgeError` + `Result<T>` discipline as
[ADR-0010](ADR-0010-error-code-result-from-expected.md).

## Consequences

### Positive

- Memory safety at compile time for the sovereign-critical code path.
- `no_std` footprint is small and predictable — every dependency is
  audited via `cargo-audit` and the tree is bounded.
- Tooling compounds with the Rust attestation service
  ([`platform/attestation/`](../../platform/attestation/)) — the two
  components share an ecosystem, reviewers, and CI patterns.
- `cargo test --features std` runs the full state-machine test suite
  on a developer laptop without touching silicon.

### Negative / Risks

- Rust + rad-hard is a less-travelled path than C. Hiring is narrower
  than for pure-C embedded.
- Cross-compile toolchain requires a `rust-toolchain.toml` pin and a
  cross-gcc for link; reproducibility is managed via the
  [Dockerfile.cross](../../src/edge-runtime/Dockerfile.cross).
- Interop with the existing C firmware requires a stable C ABI. The
  header is append-only at the patch level; breaking changes go
  through a major version bump.

### Neutral

- FFI to C is first-class in Rust; linking the staticlib into firmware
  is a one-line Makefile change.

## Alternatives considered

### Option A: C

Rejected. Memory-safety bugs on orbit are unrecoverable; the cost of
a class-action memory-corruption recall on Class V silicon would
dwarf the tooling savings.

### Option B: C++ with a `no_std`-analogue subset

Rejected. Configuring the compiler / linker to produce a freestanding
binary with exceptions disabled, RTTI disabled, and STL substituted
is fragile. Rust's `no_std` flag is maintained by the language team.

### Option C: SPARK / Ada

Considered seriously. Rejected on hiring-pool grounds and because the
Rust `no_std` story has matured enough that SPARK's historical
formal-by-construction edge is narrower than it was five years ago.

## References

- [`src/edge-runtime/`](../../src/edge-runtime/)
- [`src/edge-runtime/README.md`](../../src/edge-runtime/README.md)
- [ADR-0010](ADR-0010-error-code-result-from-expected.md)
