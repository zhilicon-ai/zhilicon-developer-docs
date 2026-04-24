# Run the kernel library on a laptop (emulation backend)

This page walks you end-to-end through running the Zhilicon Python
kernel library on a developer laptop — no silicon, no CUDA, no special
drivers. You will end with a real, numerically-correct `rmsnorm`
computed against a numpy array.

> **Who this is for.** You want to evaluate the SDK surface, try the
> kernel library, build a prototype, or run a tutorial — and you do not
> want to wait for hardware. This page is the shortest path from a
> clean Python install to a green pytest run.
>
> **What this is not.** The emulation backend is not a substitute for
> silicon. Performance numbers from it are 100×–1000× slower than real
> hardware and are refused by the benchmark harness
> ([ADR-0013](../../adr/ADR-0013-kernels-emulation-backend.md)). Do not
> report laptop numbers in release notes or dashboards.

## Prerequisites

- Python 3.10, 3.11, or 3.12 (3.13 will compile but is not in the CI
  matrix yet).
- A C++17-capable toolchain on `PATH`. On macOS the Xcode Command Line
  Tools `clang++` is enough; on Linux, `g++-13` or newer.
- numpy 1.24 or newer (the emulation backend routes every kernel
  through numpy's Python API).

## Install the SDK from source

The project is source-only today — no pre-built wheels on PyPI.

```sh
git clone <your fork URL>
cd ZhiliconMVP

python -m pip install --upgrade "setuptools>=64" wheel "pybind11>=2.12" numpy

# Editable install. Compiles zhilicon._kernels from the three C++
# translation units under src/sdk/python/zhilicon/_kernels_*.cpp.
pip install -e src/sdk/python -v
```

The install step ends with a line like:

```text
Successfully installed zhilicon-0.1.0
```

Note: only `zhilicon._kernels` is produced from `pip install`. The
sibling extension `zhilicon._core` (device / memory / stream bindings)
requires the CMake-driven C++ runtime library; it is **not** built from
`pip install`. The Python package degrades gracefully in its absence
(see [`zhilicon/device.py`](../../../src/sdk/python/zhilicon/device.py)).

## Verify the extension loaded

```sh
python - <<'PY'
import zhilicon._kernels as ext
print("extension tag:     ", ext.__zhilicon_extension__)
print("ABI version:       ", ext.__abi_version__)
print("backend kind:      ", ext.__backend_kind__)
print("dnn ops:           ", sorted(n for n in dir(ext.dnn)  if not n.startswith("_")))
print("blas ops:          ", sorted(n for n in dir(ext.blas) if not n.startswith("_")))
PY
```

Expected output:

```text
extension tag:      _kernels
ABI version:        1
backend kind:       emulation
dnn ops:            ['flash_attention', 'grouped_query_attention', 'layernorm', 'rmsnorm', 'rope']
blas ops:           ['gemm_epilogue', 'gemm_silu_gate']
```

If `backend kind` reads `emulation`, the numpy reference path is
active. `native` means a hardware-tuned backend is linked; everything
below still works.

## Run a real kernel

```python
import numpy as np
from zhilicon.kernels import norm

rng = np.random.default_rng(0)
x      = rng.standard_normal((1, 128, 4096), dtype=np.float32)
weight = np.ones((4096,),                    dtype=np.float32)

y = norm.rmsnorm(x, weight=weight, eps=1e-6)

print("input  shape:", x.shape, "dtype:", x.dtype)
print("output shape:", y.shape, "dtype:", y.dtype)
print("mean of squared output (~1 for RMS-normed):",
      float(np.mean(np.square(y))))
```

Expected:

```text
input  shape: (1, 128, 4096) dtype: float32
output shape: (1, 128, 4096) dtype: float32
mean of squared output (~1 for RMS-normed): 0.9997…
```

The output is numerically correct to within one float32 ULP of the
numpy reference — accumulation is in FP64 inside the dispatcher and
cast back to the input dtype at the end.

## Run the test suite

```sh
pip install pytest
pytest src/tests/sdk/python/ -v
```

You should see `30 passed, 1 skipped`. The one skip is `test_device.py`
— it exercises the native `_core` extension which, per the note above,
is not built by `pip install`.

## Which kernels are real in the emulation backend?

All v1 canonical kernels ([ADR-0014](../../adr/ADR-0014-kernel-library-v1-canonical-set.md))
are fully computed on numpy inputs:

| Kernel                          | Emulation backend | Note                                                |
| ------------------------------- | ----------------- | --------------------------------------------------- |
| `dnn.rmsnorm`                   | Computes          | Matches numpy reference within 1 ULP                |
| `dnn.layernorm`                 | Computes          | With or without bias                                |
| `dnn.rope`                      | Computes          | Half-rotate convention (LLaMA / Qwen)               |
| `dnn.softmax`                   | Computes          | Numerically stable (max-subtract)                   |
| `dnn.cross_entropy_loss`        | Computes          | log-sum-exp fused; `mean` / `sum` / `none`          |
| `dnn.flash_attention`           | Computes (naive)  | O(N²) SDPA reference. `return_lse=True` unsupported. |
| `dnn.grouped_query_attention`   | Computes (naive)  | Expands KV heads via `np.repeat`, then SDPA.        |
| `blas.gemm_epilogue`            | Computes          | `none` / `relu` / `gelu` (tanh) / `silu`            |
| `blas.gemm_silu_gate`           | Computes          | SwiGLU-style gated projection                       |
| `blas.linear`                   | Computes          | PyTorch `nn.Linear` convention (fused transpose)    |

Every kernel falls through to a `ValueError` that names the
authoritative C++ implementation file under `src/sdk/src/{dnn,blas}/`
when the input is *not* a numpy array (a `zhilicon.Tensor`, a torch
tensor, or any duck type). Convert to numpy first for laptop-local
work.

## Attention example

```python
import numpy as np
from zhilicon.kernels import attention

rng = np.random.default_rng(0)
b, s, h, d = 1, 128, 8, 64

q = rng.standard_normal((b, s, h, d), dtype=np.float32)
k = rng.standard_normal((b, s, h, d), dtype=np.float32)
v = rng.standard_normal((b, s, h, d), dtype=np.float32)

# Causal attention (decoder self-attention).
out = attention.flash_attention(q, k, v)
print("attention output shape:", out.shape)  # (1, 128, 8, 64)

# Grouped-query attention: q has 8 heads, kv has 2 (4× compression).
h_kv = 2
k_gqa = rng.standard_normal((b, s, h_kv, d), dtype=np.float32)
v_gqa = rng.standard_normal((b, s, h_kv, d), dtype=np.float32)
out_gqa = attention.grouped_query_attention(
    q, k_gqa, v_gqa, num_kv_heads=h_kv
)
print("GQA output shape:", out_gqa.shape)  # (1, 128, 8, 64)
```

The emulation reference is an explicit `O(N²)` `softmax(QKᵀ/√d)·V` —
no tiling. For sequences above ~4096 it will OOM on a laptop; this is
by design, see ADR-0014.

## Common pitfalls

### I passed a `zhilicon.Tensor` and got `ValueError: … not yet wired`

The emulation backend is gated on numpy arrays. Any other type
— `zhilicon.Tensor`, torch tensors, custom duck types — falls through
to the `stub_not_wired` path and raises with a pointer to the C++
implementation file. For laptop-local work, convert to numpy first:

```python
arr = my_tensor.numpy()
y   = norm.rmsnorm(arr, weight=w, eps=1e-6)
```

### `fatal error: 'pybind11/pybind11.h' file not found` at install time

setup.py uses `pybind11.setup_helpers.Pybind11Extension` to resolve the
pybind11 include path at build time. Reinstall pybind11 in the same
Python environment you are using for `pip install -e`:

```sh
pip install --upgrade "pybind11>=2.12"
```

### Shape mismatch in `rope`

The `cos` and `sin` tables must broadcast over the head axis of
`x [batch, seq, head, head_dim]`. The standard convention is to shape
them as `[seq, 1, head_dim/2]`. See
[`test_kernels_numerics.py`](../../../src/tests/sdk/python/test_kernels_numerics.py)
for a worked example.

## Next steps

- [Integrate the Python SDK](integrate-sdk.md) — wiring the kernel
  library into an existing training loop.
- [Run the benchmark harness](../../../src/tools/bench/README.md) — how
  the ADR-0013 guard refuses emulation numbers on the dashboard path.
- [ADR-0013: `_kernels` emulation backend](../../adr/ADR-0013-kernels-emulation-backend.md)
  — the full design record behind this backend.
