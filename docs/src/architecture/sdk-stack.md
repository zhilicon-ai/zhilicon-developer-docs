# SDK stack

The Zhilicon SDK exposes the silicon through a **declarative compute API**
rather than a CUDA-shaped imperative kernel launcher. Applications describe
what they want computed, not how to schedule it; the runtime handles tile
choice, pipelining, memory placement, and sovereign-zone enforcement.

## Layers

```text
  Python (user-facing)      zhilicon.compute / .medical / .crypto / .space
                            zhilicon.kernels.{attention,linalg,norm,rope}
                            zhilicon.integrations.{pytorch,model_hub,agents}
  ----------------------------------------------------------------------
  C bindings                zhilicon._core    (pybind11) → Device / Stream / Memory
                            zhilicon._kernels (pybind11) → dnn, blas dispatchers
  ----------------------------------------------------------------------
  Dispatcher                Numpy emulation backend (laptop-local)  ⎫
                            Hardware-tuned native backend (silicon) ⎭ one of
  ----------------------------------------------------------------------
  C++ runtime               OperatorRegistry · Scheduler · AutotuneCache
                            ErrorCode / Result<T> · Stream / Graph
  ----------------------------------------------------------------------
  C++ DNN / BLAS            flash_attention · rmsnorm · layernorm · rope
                            gemm · gemm_epilogue · gemm_silu_gate
  ----------------------------------------------------------------------
  HAL                       Per-chip backends (src/sdk/backends/…)
                            MMIO · DMA · command buffer submission
  ----------------------------------------------------------------------
  Silicon                   5 chip programs + shared IP library
```

## Kernel dispatcher: emulation vs native

The `zhilicon._kernels` extension stamps `__backend_kind__` at module
load time — one of `"emulation"` or `"native"`. Every kernel dispatcher
inspects its inputs:

- **numpy arrays in, emulation backend:** the dispatcher computes a
  reference result via numpy's Python API (FP64 accumulate, cast back
  to input dtype). Correct but not hardware-accelerated. This path
  exists so SDK-layer tests, documentation examples, and prototype
  notebooks run on a developer laptop with no silicon attached.
- **numpy arrays in, native backend:** the dispatcher launches the
  hardware-tuned kernel on the active `DispatchContext`'s device.
- **anything-else in (real `zhilicon.Tensor`, torch tensors, duck
  types):** the emulation backend falls through to `stub_not_wired`,
  raising a `ValueError` that names the authoritative C++ file under
  `src/sdk/src/{dnn,blas}/`. The native backend launches the hardware
  kernel against the tensor's device pointer.

The two-mode dispatcher is the load-bearing mechanism behind
[ADR-0013](../adr/ADR-0013-kernels-emulation-backend.md). Its partner
is the benchmark harness guard, which refuses to record numbers
against any backend other than `"native"` — keeping laptop-speed
measurements out of the production performance dashboard.

## Key entry points

- [`zhilicon/compute.py`](../../../src/sdk/python/zhilicon/compute.py) — declarative compute.
- [`zhilicon/kernels/`](../../../src/sdk/python/zhilicon/kernels/) — fused Python kernels
  (attention, fused-GEMM, norms, RoPE). Each resolves a backend via
  [`_backend.py`](../../../src/sdk/python/zhilicon/kernels/_backend.py) with a
  clear-error placeholder if the compiled extension is absent.
- [`zhilicon/runtime/operator_registry.hpp`](../../../src/sdk/include/zhilicon/runtime/operator_registry.hpp) — C++ operator registry.
- [`zhilicon/runtime/error.hpp`](../../../src/sdk/include/zhilicon/runtime/error.hpp) — error convention (`ErrorCode` + `Result<T>`).

## Sovereign gating

Every operator dispatch receives a
[`DispatchContext`](../../../src/sdk/include/zhilicon/runtime/operator_registry.hpp)
that carries the `sovereign_zone` tag. The runtime refuses to launch against
a device that is not attested in that zone.

## Reference docs

- [Python SDK reference](../reference/python-sdk.md)
- [C ABI reference](../reference/c-abi.md) (edge-runtime only)
- [Formal-property library](../reference/formal-library.md) (for RTL-level guarantees the SDK depends on)
