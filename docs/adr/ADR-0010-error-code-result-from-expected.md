---
adr-id: ADR-0010
title: '`ErrorCode` + `Result<T>` as the canonical SDK error model'
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/sdk-coe", "@zhilicon/opensilicon-coe"]
---

# ADR-0010: `ErrorCode` + `Result<T>` as the canonical SDK error model

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** SDK COE + OpenSilicon COE

## Context

The C++ SDK touches performance-critical paths (kernel launches, DMA
initiation, HAL access) where exception overhead and non-local control
flow are both unwanted. At the same time, the Python and Rust bindings
need to round-trip failures with low cost and clear semantics.

During the Batch-4 portfolio audit, a parallel `Status` class with an
attached message string was briefly introduced, duplicating the
pre-existing `Status` alias in `zhilicon/runtime/error.hpp`. The
duplication caused an API mismatch that was only caught in the audit;
it would have bitten the first integrator.

## Decision

The SDK uses the following canonical error model across all C++, pybind11,
and (mirrored) Python surfaces:

```cpp
enum class ErrorCode : uint32_t {
    kSuccess = 0, kDeviceNotFound = 1, kOutOfMemory = 2, kInvalidArgument = 3,
    kDeviceBusy = 4, kTimeout = 5, kDriverError = 6, kUnsupported = 7,
    kPermissionDenied = 8, kEccError = 9, kThermalThrottle = 10,
    kLinkError = 11, kInternalError = 12,
};

template <typename T>
using Result = std::expected<T, ErrorCode>;
using Status = std::expected<void, ErrorCode>;
```

Three invariants:

1. **No exceptions** in performance-critical code paths. Functions that
   can fail return `Result<T>` or `Status`.
2. **`error_to_string(code)`** is the canonical way to produce a
   human-readable message. No free-form message string is attached to
   the error value; context lives at the callsite.
3. **`ErrorCode`** is **append-only**. New variants are added before
   `kInternalError`; existing numeric values never change.

The Rust edge-runtime mirrors the same discipline with its own
`EdgeError` enum, surfaced through the C ABI in
[`zhilicon_edge.h`](../../src/edge-runtime/include/zhilicon_edge.h).

## Consequences

### Positive

- Zero exception overhead on hot paths.
- Monadic error composition via `std::expected`'s `.and_then` /
  `.transform`.
- Stable wire encoding — `ErrorCode` integer values are an ABI-level
  contract that bindings, SDKs, and remote telemetry can rely on.
- Easy to mirror in Python (`ZhiliconError` with a numeric `code`
  attribute) and Rust (matching enum).

### Negative / Risks

- Requires `std::expected`, a C++23 feature. On toolchains pinned to
  C++20, a `tl::expected` polyfill is permitted by the header.
- Loses free-form message at the error value. Mitigation: logging at
  the callsite carries the context; `error_to_string` gives the
  category name.

### Neutral

- Third-party libraries that raise exceptions are caught at the SDK
  boundary and translated to `ErrorCode::kInternalError` with a logged
  context.

## Alternatives considered

### Option A: Exception-based error model

Rejected. Unacceptable on kernel-launch hot paths. Also forces
Python/Rust bindings to cross an exception boundary.

### Option B: Abseil-style `Status` class with message string

Rejected after the Batch-4 audit. Duplicates `zhilicon::Status` as
declared in `error.hpp`; introduces a parallel error-type paradigm.

### Option C: Error-code-only (no `Result<T>` wrapper)

Rejected. Loses type safety; forces out-parameters.

## References

- [`src/sdk/include/zhilicon/runtime/error.hpp`](../../src/sdk/include/zhilicon/runtime/error.hpp)
- [`src/edge-runtime/src/error.rs`](../../src/edge-runtime/src/error.rs)
- Batch-4 audit reconciliation (session history)
