# Edge runtime (Horizon-1)

The edge runtime is the on-chip software layer that executes on the
Horizon-1 Cortex-R82AE cores while the AI tiles crunch data.
Source lives under [`src/edge-runtime/`](../../../src/edge-runtime/).

## What it does

| Module                                         | Role                                                                 |
| ---------------------------------------------- | -------------------------------------------------------------------- |
| [`sovereign_context`](../../../src/edge-runtime/src/sovereign_context.rs) | Seal-once + rotate-only sovereign context; gate every attested op     |
| [`attestation`](../../../src/edge-runtime/src/attestation.rs)             | SpaceFibre-framed attestation handshake with ground service            |
| [`tmr`](../../../src/edge-runtime/src/tmr.rs)                             | Software-side triple-modular-redundancy voter                         |
| [`seu_recovery`](../../../src/edge-runtime/src/seu_recovery.rs)           | SEU-IRQ handler · checkpoint/rollback · SAFE_MODE entry               |
| [`scheduler`](../../../src/edge-runtime/src/scheduler.rs)                 | Rate-monotonic deterministic scheduler with Liu-Layland schedulability |
| [`ota`](../../../src/edge-runtime/src/ota.rs)                             | A/B-slot over-the-air update verifier                                 |
| [`telemetry`](../../../src/edge-runtime/src/telemetry.rs)                 | Lock-free SPSC ring-buffer for on-chip events                          |
| [`hal`](../../../src/edge-runtime/src/hal/mod.rs)                         | Hardware abstraction with `Horizon1Hal` + `StubHal` for host tests     |
| [`cabi`](../../../src/edge-runtime/src/cabi.rs)                           | C ABI surface consumed by the C firmware in [`src/firmware/`](../../../src/firmware/) |

## Design principles

1. **`no_std`-first.** Compiles for `armv7r-none-eabihf` and
   `aarch64-unknown-none` without a `std` shim.
2. **No dynamic allocation after init.** All buffers are sized at compile
   time. Any post-init allocation is a bug.
3. **Bounded worst-case execution time** on every public function.
4. **Errors are values, not panics.** Panics only for unrecoverable
   hardware faults and always through [`seu_recovery::enter_safe_mode`](../../../src/edge-runtime/src/seu_recovery.rs).
5. **Sovereign by construction.** No operation on attested resources
   proceeds without a current attestation proof.

## Build

```bash
cd src/edge-runtime
cargo test --features std                             # host-side tests
cargo build --release --target armv7r-none-eabihf     # Horizon-1 target
make staticlib                                        # rlib + staticlib + C header
```

Or use the reproducible [cross-compile container](../../../src/edge-runtime/Dockerfile.cross).

## See also

- [C ABI reference](../reference/c-abi.md) for calling the runtime from the
  existing C firmware.
- [Horizon-1 chip page](../portfolio/horizon-1.md) for the silicon context.
