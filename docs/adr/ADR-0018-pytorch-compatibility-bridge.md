---
adr-id: ADR-0018
title: 'PyTorch-compatibility bridge for v1 kernels'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/devrel"]
---

# ADR-0018: PyTorch-compatibility bridge for v1 kernels

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, DevRel

## Context

PyTorch is the adoption default for ML research and the majority of
production training. Every conversation with a prospective
integration partner asks the same question: *"Can I drop your kernel
into my existing PyTorch training loop?"*

The v1 kernel library dispatches on **numpy arrays** for the
emulation path (ADR-0013). A PyTorch user who wants to call
`rmsnorm` on their `torch.Tensor` has to manually convert:

```python
y = torch.from_numpy(
    rmsnorm(x.detach().cpu().numpy(), weight=w.cpu().numpy())
).to(x.device)
```

That is verbose enough to kill adoption. We need a thin shim that
accepts `torch.Tensor` inputs transparently.

## Decision

Ship `zhilicon.kernels.torch_compat` — a one-to-one mirror of every
v1 kernel that accepts and returns `torch.Tensor` objects.

### Design rules

1. **Signature parity.** Every wrapper keeps the *exact same*
   keyword arguments as its numpy counterpart. Switching between
   `from zhilicon.kernels.norm import rmsnorm` and
   `from zhilicon.kernels.torch_compat import rmsnorm` requires
   zero other code changes.
2. **Lazy torch import.** Torch is imported on first *call*, not at
   module import time. The SDK stays installable without torch as a
   dependency; users who do not install torch can still import the
   module (and use its numpy-only pathways).
3. **Input-type inference.** Wrappers detect whether any positional
   argument is a `torch.Tensor`. If so, they return a
   `torch.Tensor`; otherwise they return numpy, matching the input.
   Mixed (some torch, some numpy) inputs are treated as torch for
   output-type purposes — anything else would be surprising.
4. **Device + dtype preservation.** The output tensor inherits the
   first torch input's `.device` and float `.dtype`. A CUDA input
   returns CUDA output (incurring a host round-trip today; when the
   native backend lands, this path will short-circuit into a direct
   device kernel launch).
5. **Integer dtypes not auto-cast.** Label tensors (for
   `cross_entropy_loss`) stay on their original dtype through the
   conversion — we never silently cast int64 to float.

### What's covered

All v1 kernels (ADR-0014):

| Kernel                      | Torch-compat wrapper                                     |
| --------------------------- | -------------------------------------------------------- |
| `rmsnorm`                   | `torch_compat.rmsnorm`                                   |
| `layernorm`                 | `torch_compat.layernorm`                                 |
| `rope`                      | `torch_compat.rope`                                      |
| `softmax`                   | `torch_compat.softmax`                                   |
| `cross_entropy_loss`        | `torch_compat.cross_entropy_loss`                        |
| `linear`                    | `torch_compat.linear`                                    |
| `fused_gemm_relu`           | `torch_compat.fused_gemm_relu`                           |
| `fused_gemm_silu_gate`      | `torch_compat.fused_gemm_silu_gate`                      |
| `flash_attention`           | `torch_compat.flash_attention`                           |
| `grouped_query_attention`   | `torch_compat.grouped_query_attention`                   |

Plus two conversion helpers: `to_numpy(tensor)` and
`as_torch(array, *, like)`.

### Testing strategy

- `test_torch_compat.py` — skipped at module level when torch is
  absent; cross-checks every wrapper against
  `torch.nn.functional.*` where a direct equivalent exists
  (`softmax`, `cross_entropy`, `linear`, `scaled_dot_product_attention`).
- `test_torch_compat_no_torch.py` — runs *without* torch installed,
  pins: (a) module importability, (b) numpy-in → numpy-out path
  works, (c) `to_numpy` identity on numpy arrays.

Together the two test files cover both torch-present and
torch-absent environments.

## Consequences

### Positive

- A PyTorch user pipes `torch.Tensor` straight into a Zhilicon
  kernel call. No boilerplate, no device ping-pong code in their
  application.
- `torch_compat.softmax` is bit-identical to `F.softmax` (tested).
  `torch_compat.cross_entropy_loss` matches `F.cross_entropy` to
  `1e-4`. `torch_compat.flash_attention` matches
  `F.scaled_dot_product_attention` to `1e-4`. Third-party
  correctness evidence that we don't have to produce ourselves.
- The PyTorch integration is one *file*, 250 lines, no new kernel
  code. Future kernel additions only need a new wrapper here.

### Negative / Risks

- CPU–GPU round-trip on CUDA inputs. Every call today materialises
  the tensor on the host, runs the numpy emulation, and ships back.
  Unacceptable for production; fine for development. When the
  native backend lands, `torch_compat` short-circuits into the
  direct silicon kernel — documented as a release-gate item.
- Autograd. We explicitly `.detach()` torch inputs in `to_numpy`,
  breaking the autograd graph. The wrappers are **forward-only**.
  Calling `.backward()` through them is a silent no-op; integrating
  with autograd is a v2 concern (ADR TBD).

### Neutral

- `torch.from_numpy` shares memory with the source array on CPU, so
  the CPU round-trip is one allocation (for the kernel output)
  rather than two. Good enough for v1.

## Alternatives considered

### Option A: Detect torch tensors inside each wrapper module directly

Rejected. That bloats every wrapper with torch-detection logic and
violates the "the kernel dispatcher is a numpy function" contract.
A separate module is the cleaner fence.

### Option B: Monkey-patch `torch.nn.functional.rms_norm` to call us

Rejected. Attractive ("transparent drop-in!"), but breaks the
principle that third-party namespaces stay untouched. A user who
imports both `torch_compat.rmsnorm` and `F.rms_norm` must be able
to tell which one they are calling.

### Option C: Ship a torch custom-op (`torch.ops.zhilicon.rmsnorm`)

Attractive long-term: makes the kernels visible to `torch.compile`
and the autograd engine. Rejected for v1 because it requires
building a torch C++ extension, which is a separate build path
from our pybind11 kernel extension. Revisit when the native kernel
backend lands — at that point a single extension can serve both
torch and the pybind11 surface.

## References

- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0014: Kernel library v1 canonical set](ADR-0014-kernel-library-v1-canonical-set.md)
- [`src/sdk/python/zhilicon/kernels/torch_compat.py`](../../src/sdk/python/zhilicon/kernels/torch_compat.py)
- [`src/tests/sdk/python/test_torch_compat.py`](../../src/tests/sdk/python/test_torch_compat.py)
- [`src/tests/sdk/python/test_torch_compat_no_torch.py`](../../src/tests/sdk/python/test_torch_compat_no_torch.py)
- Batch-23 session history.
