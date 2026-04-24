---
adr-id: ADR-0011
title: '`_kernels` as a sibling pybind11 extension of `_core`'
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/sdk-coe"]
---

# ADR-0011: `_kernels` as a sibling pybind11 extension of `_core`

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** SDK COE

## Context

The Python kernel library (`zhilicon.kernels.{attention, linalg, norm, rope}`)
was originally scaffolded to import from `zhilicon._core.dnn` and
`zhilicon._core.blas`. The existing `_core` extension (built from
`_bindings.cpp`) exposes `Device`, `Stream`, `Event`, `DeviceMemory`,
`DeviceInfo`, `ChipType`, `ErrorCode` — but no DNN / BLAS submodules.
At import time, the kernel library fell back to a `_MissingBackend`
placeholder that raised `NotImplementedError` on any call.

Two options to bring real compiled kernels online:

1. **Extend `_bindings.cpp`** to register DNN + BLAS submodules inside
   `PYBIND11_MODULE(_core, m)`. Touches a 244-line pre-existing file
   owned by another team.
2. **Create a sibling `_kernels` extension** with its own
   `PYBIND11_MODULE(_kernels, m)`. Zero changes to `_bindings.cpp`;
   adds a second entry to `pyproject.toml`'s `[tool.setuptools]
   ext-modules`.

Given the Batch-4 session-learned lesson (overwriting unfamiliar code
causes silent regressions), option 2 is structurally safer.

## Decision

Add `zhilicon._kernels` as a sibling extension module. Structure:

- [`_kernels_bindings.cpp`](../../src/sdk/python/zhilicon/_kernels_bindings.cpp)
  — entry point with `PYBIND11_MODULE(_kernels, m)`, an ABI-version
  stamp, and calls out to subpackage registration functions.
- [`_kernels_dnn.cpp`](../../src/sdk/python/zhilicon/_kernels_dnn.cpp)
  — DNN submodule (`flash_attention`, `grouped_query_attention`,
  `layernorm`, `rmsnorm`, `rope`).
- [`_kernels_blas.cpp`](../../src/sdk/python/zhilicon/_kernels_blas.cpp)
  — BLAS submodule (`gemm_epilogue`, `gemm_silu_gate`).

The Python-side resolver
([`kernels/_backend.py`](../../src/sdk/python/zhilicon/kernels/_backend.py))
tries `_kernels → _core → _MissingBackend` in that order.

Initial kernel entry points validate inputs and raise
`NotImplementedError` with a pointer to the authoritative C++
implementation file. Future batches replace each stub body with a
real call; the Python surface stays fixed.

## Consequences

### Positive

- Zero touch on `_bindings.cpp` — eliminates the Batch-4 class of
  regression.
- Clear separation of concerns: `_core` owns device / memory / stream;
  `_kernels` owns compute primitives.
- The sibling extension can evolve on its own ABI cadence (currently
  `__abi_version__ = 1`). `_core` is unchanged.
- Type stubs in [`_kernels.pyi`](../../src/sdk/python/zhilicon/_kernels.pyi)
  give IDE and mypy full API visibility without running the extension.

### Negative / Risks

- Two `.so` files per package install instead of one. Negligible size,
  imperceptible load-time cost.
- Someone extending the kernel library later might attempt to add a
  function to `_bindings.cpp` instead of `_kernels_dnn.cpp`. CODEOWNERS
  routes the new files to the SDK + OpenSilicon COEs; the ADR is
  linked from the SDK reference page.

### Neutral

- The `_core` legacy path in `_backend.py` is kept for forward
  compatibility if `_bindings.cpp` ever grows DNN / BLAS submodules of
  its own.

## Alternatives considered

### Option A: Add submodules to `_core` directly

Rejected. Touches a 244-line pre-existing file owned by another team;
risk of regression outweighs the small benefit of a single `.so` file.

### Option B: Separate Python package (`zhilicon-kernels`) installed alongside `zhilicon`

Rejected. Adds pip-install coupling and release synchronisation
complexity with no offsetting benefit.

## References

- [`src/sdk/python/zhilicon/_kernels_bindings.cpp`](../../src/sdk/python/zhilicon/_kernels_bindings.cpp)
- [`src/sdk/python/pyproject.toml`](../../src/sdk/python/pyproject.toml)
- [ADR-0010](ADR-0010-error-code-result-from-expected.md)
- Batch-11 session history
