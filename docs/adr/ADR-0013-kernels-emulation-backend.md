---
adr-id: ADR-0013
title: '`_kernels` emulation backend for laptop-local development'
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/sdk-coe", "@zhilicon/opensilicon-coe"]
---

# ADR-0013: `_kernels` emulation backend for laptop-local development

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** SDK COE, OpenSilicon COE

## Context

[ADR-0011](ADR-0011-kernels-as-sibling-extension.md) established the
`zhilicon._kernels` pybind11 extension as the surface for fused DNN /
BLAS kernels. The initial dispatcher bodies were intentional stubs that
raised `NotImplementedError` pointing at the authoritative C++
implementation file under `src/sdk/src/{dnn,blas}/`.

The stubs were acceptable for API-surface tests but blocked everything
downstream that wants to *run* a kernel end-to-end:

- SDK integration tests that call `kernels.rmsnorm(...)` to verify
  Python-side plumbing.
- Documentation examples in [`docs/src/sdk/`](../src/sdk/)
  that claim to be runnable.
- Third-party notebooks and tutorials that the DevRel team publishes.

Two paths to unblock them:

1. **Fully wire the hardware-tuned kernels now.** Pulls in the compiled
   kernel tree under `src/sdk/src/{dnn,blas}/`, the autotune cache, the
   device dispatcher, and the CUDA / HIP / Zhilicon-native backends.
   ~10k LOC of C++ on the critical path of one batch.
2. **Provide a numpy-backed emulation backend.** Reference math in the
   dispatcher itself, gated on `py::isinstance(x, np.ndarray)`. Real
   `zhilicon.Tensor` inputs continue to hit the stub and will light up
   when the hardware path lands.

Path 1 is the destination; it is not one batch of work. Path 2 is a
halfway house that lets the documentation, tutorial, and test surface
come alive today without overpromising "native kernel support". The
tradeoff: the emulation path must be labelled clearly so a user does
not run production workloads through it and file a performance bug.

## Decision

Add an emulation backend to the `_kernels` extension. Structure:

- The extension stamps `__backend_kind__ = "emulation"` at module
  load. Consumers (tests, telemetry, runtime) gate performance-sensitive
  code paths on this stamp.
- Each kernel dispatcher begins with a `all_numpy(args...)` check that
  imports numpy lazily (cached), and returns `false` if numpy is not
  importable.
- If all inputs are numpy arrays, the dispatcher computes the reference
  result via numpy's Python API, accumulating in float64 and casting
  back to the input dtype on return.
- If any input is not a numpy array (typically a `zhilicon.Tensor`),
  the dispatcher falls through to `stub_not_wired(op, impl_file)`,
  which raises a `ValueError` naming the authoritative C++ file.

Kernels covered by the emulation backend (expanded to the ADR-0014 v1
canonical set):

| Kernel                          | Emulated | Reference formula                                       |
| ------------------------------- | -------- | ------------------------------------------------------- |
| `dnn.layernorm`                 | yes      | `(x - mean) / sqrt(var + eps) * weight [+ bias]`        |
| `dnn.rmsnorm`                   | yes      | `x / sqrt(mean(x²) + eps) * weight`                     |
| `dnn.rope`                      | yes      | Half-rotate RoPE (LLaMA / Qwen convention)              |
| `dnn.softmax`                   | yes      | Max-subtract stabilisation + exp + normalise            |
| `dnn.cross_entropy_loss`        | yes      | `-log_softmax(logits)[labels]` via log-sum-exp          |
| `dnn.flash_attention`           | yes      | Naive O(N²) SDPA: `softmax(QKᵀ/√d) · V`                 |
| `dnn.grouped_query_attention`   | yes      | Repeat-K/V along head axis, then naive SDPA             |
| `blas.gemm_epilogue`            | yes      | `activation(a @ b + bias)` — none / relu / gelu / silu  |
| `blas.gemm_silu_gate`           | yes      | `(a @ w_up) * silu(a @ w_gate)` — SwiGLU                |
| `blas.linear`                   | yes      | `activation(a @ w.T + bias)` — PyTorch nn.Linear        |

### Attention in the emulation backend (revised by ADR-0014)

The original ADR-0013 text deferred attention to a later batch on the
grounds that a wrong reference would be worse than no reference. After
shipping the other v1 kernels and verifying the dispatcher pattern, the
attention path was filled in with a **naive O(N²) reference** — exactly
the formula above. The implementation fits in ~40 lines because the
emulation backend is not performance-sensitive (ADR-0013 guard).

The native-backend story is unchanged: tiled online softmax with
log-sum-exp accumulation, causal / sliding-window masking, and FP32
softmax accumulate. The native kernel replaces the naive path when it
lands; the Python surface stays identical. See
[ADR-0014](ADR-0014-kernel-library-v1-canonical-set.md) for the full
v1 kernel-set decision.

`return_lse=True` remains emulation-unsupported. The log-sum-exp
return is the signature of the tiled algorithm and is only meaningful
in the native path.

## Consequences

### Positive

- Documentation examples that use `rmsnorm` / `layernorm` / `rope` /
  SwiGLU are runnable on a developer laptop with a pip install of
  `zhilicon` + numpy. Previously they raised.
- SDK-layer integration tests get real numerical signal without
  standing up a full hardware stack.
- The `__backend_kind__ = "emulation"` stamp makes it trivial for
  telemetry to refuse to record performance numbers from a laptop run,
  so we do not poison the dashboards.
- The stub-with-file-pointer path is preserved for attention and for
  non-numpy tensor types, so the hardware-wiring work remains
  straightforward.

### Negative / Risks

- A caller might benchmark the emulation backend and misreport the
  number as Zhilicon-silicon performance. Mitigations:
  1. `__backend_kind__` stamp exposed prominently.
  2. Benchmark harness ([`src/tools/bench/`](../../src/tools/bench/))
     asserts `__backend_kind__ == "native"` before recording.
  3. Documentation pages for each kernel call out the emulation
     caveat in a note block.
- Attention tests that want to exercise the Python plumbing still hit
  the stub. Mitigation: the stub message names the C++ file, and the
  resolver test in
  [`test_kernels_backend.py`](../../src/tests/sdk/python/test_kernels_backend.py)
  pins this contract.

### Neutral

- Numpy becomes a soft runtime dependency of `_kernels`. The extension
  still imports cleanly without numpy — only the emulation path is
  unavailable. Since numpy is already a transitive dependency of most
  of the Python SDK, this is a no-op in practice.

## Alternatives considered

### Option A: Wire the real kernels in this batch

Rejected. The hardware-tuned kernel tree is ~10k LOC across multiple
translation units and has its own autotune, device-dispatch, and build
plumbing. Landing it in a single batch would be either (a) untested or
(b) this one batch for the rest of the quarter.

### Option B: Pure-Python reference shipped alongside `_kernels`

Rejected. A Python-level reference would require the resolver to
decide between `_kernels` and a `_kernels_py` fallback, adding a code
path without adding information. Keeping the emulation inside the
extension — same entry point, same dispatcher — means one code path
for callers and a clean "flip a flag" migration to native when the
hardware kernel is wired.

### Option C: Torch-backed reference

Rejected. Adding torch as a hard dependency of `_kernels` is a much
larger ask than numpy. The SDK's torch interop lives in a separate
package ([`src/sdk/python/zhilicon/interop/torch.py`](../../src/sdk/python/zhilicon/interop/torch.py))
that itself imports `zhilicon._kernels`, and introducing a circular
dependency here would be a footgun.

## Migration path to native

When the hardware-tuned kernel for a given operation lands:

1. The dispatcher body for that operation changes from
   "emulation-branch + stub-fallthrough" to a direct call into
   `zhilicon::dnn::layernorm_launch(...)` (or equivalent).
2. If **all** kernels in the extension are wired, flip
   `__backend_kind__` from `"emulation"` to `"native"`.
3. Migration is per-kernel: partial emulation (e.g. rmsnorm native,
   rope still emulated) is a valid state. The `__backend_kind__` stamp
   is module-level; per-kernel status is queryable via
   `zhilicon.kernels.backend_info()` (scaffolded but not yet
   populated — deferred to the native wiring batch).

## References

- [ADR-0011](ADR-0011-kernels-as-sibling-extension.md) — established
  `_kernels` as a sibling extension.
- [`src/sdk/python/zhilicon/_kernels_dnn.cpp`](../../src/sdk/python/zhilicon/_kernels_dnn.cpp)
- [`src/sdk/python/zhilicon/_kernels_blas.cpp`](../../src/sdk/python/zhilicon/_kernels_blas.cpp)
- [`src/tests/sdk/python/test_kernels_numerics.py`](../../src/tests/sdk/python/test_kernels_numerics.py)
- Batch-14 session history.
