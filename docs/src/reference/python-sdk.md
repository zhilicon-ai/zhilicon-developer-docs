# Python SDK reference

Source: [`src/sdk/python/zhilicon/`](../../../src/sdk/python/zhilicon/).

## Top-level

```python
import zhilicon as zh

zh.__version__                  # package version
zh.SovereignContext(...)        # context manager
zh.compute(expr, inputs, constraints)
zh.infer(model, prompt)         # short-form inference helper
```

## `zhilicon.kernels`

Fused tensor primitives with lazy backend resolution. The Python
surface is in
[`zhilicon/kernels/`](../../../src/sdk/python/zhilicon/kernels/) and
dispatches through `zhilicon._kernels` (a pybind11 extension compiled
from `_kernels_*.cpp`). See
[ADR-0011](../../adr/ADR-0011-kernels-as-sibling-extension.md) and
[ADR-0013](../../adr/ADR-0013-kernels-emulation-backend.md) for the
extension layout + emulation-vs-native dispatcher design.

The v1 canonical set is frozen by
[ADR-0014](../../adr/ADR-0014-kernel-library-v1-canonical-set.md); the
table below is the user-facing index.

| Kernel                                                                | Python entry point                                     | Emulation backend | Native backend |
| --------------------------------------------------------------------- | ------------------------------------------------------ | ----------------- | -------------- |
| `flash_attention(q, k, v, *, config, device)`                         | `zhilicon.kernels.attention.flash_attention`           | Computes (naive)   | Planned (tiled) |
| `grouped_query_attention(q, k, v, *, num_kv_heads, config, device)`   | `zhilicon.kernels.attention.grouped_query_attention`   | Computes (naive)   | Planned (tiled) |
| `fused_gemm_relu(a, b, *, bias, precision, accumulate_precision)`     | `zhilicon.kernels.linalg.fused_gemm_relu`              | Computes           | Planned        |
| `fused_gemm_silu_gate(a, w_up, w_gate, *, precision, …)`              | `zhilicon.kernels.linalg.fused_gemm_silu_gate`         | Computes           | Planned        |
| `linear(a, w, *, bias, activation, precision, device)`                | `zhilicon.kernels.linalg.linear`                       | Computes           | Planned        |
| `layernorm(x, *, weight, bias=None, eps=1e-5, precision, device)`     | `zhilicon.kernels.norm.layernorm`                      | Computes           | Planned        |
| `rmsnorm(x, *, weight, eps=1e-6, precision, device)`                  | `zhilicon.kernels.norm.rmsnorm`                        | Computes           | Planned        |
| `rope(x, *, cos, sin, scaling, ntk_alpha, device)`                    | `zhilicon.kernels.rope.rope`                           | Computes           | Planned        |
| `softmax(x, *, axis=-1, precision, device)`                           | `zhilicon.kernels.activation.softmax`                  | Computes           | Planned        |
| `cross_entropy_loss(logits, labels, *, reduction, ignore_index, …)`   | `zhilicon.kernels.loss.cross_entropy_loss`             | Computes           | Planned        |

See [How-to: run the kernel library on a laptop (emulation backend)](../how-to/run-kernel-emulation.md)
for a runnable walkthrough.

## `zhilicon.models`

Reference models built on top of the kernel library.

| Class                      | Purpose                                                                  |
| -------------------------- | ------------------------------------------------------------------------ |
| `MinimalLlama`             | Tiny LLaMA-style decoder-only transformer (numpy-native, emulation-only). Forward pass + `prefill` / `decode` / greedy `generate()`. See [ADR-0015](../../adr/ADR-0015-minimal-llama-reference-model.md) and [ADR-0016](../../adr/ADR-0016-kv-cache-design.md). |
| `MinimalLlamaConfig`       | Architecture config for `MinimalLlama` — vocab_size, dim, num_layers, num_heads, num_kv_heads, hidden_dim, max_seq_len, rope_base, init_scale, tie_word_embeddings. |
| `KVCache`                  | Per-layer key/value buffers for incremental decoding. Owned by the caller, passed into `model.decode(...)`. See [ADR-0016](../../adr/ADR-0016-kv-cache-design.md). |
| `LlamaForCausalLM`         | Full PyTorch-Tensor-based decoder-only LM. Production path for trained weights. Requires the `_core` extension. |
| `ModelRegistry`            | Registry for discovering installed models by name.                       |

For a runnable MinimalLlama walkthrough, see
[How-to: Build a minimal transformer](../how-to/build-minimal-transformer.md).

## `zhilicon.integrations`

See [How-to → Integrate the Python SDK](../how-to/integrate-sdk.md).

## `zhilicon.medical` / `.crypto` / `.space` / `.telecom`

Domain SDKs. Each exposes an API shaped for the vertical (e.g. DICOM
parsing for medical, signing batches for crypto). See the per-package
`__init__.py` docstrings in the repo for the authoritative signatures.

## `zhilicon.sovereign*`

The `sovereign_*` module family implements data residency, attestation,
audit logging, and other sovereign-stack primitives. Entry point:
[`SovereignContext`](../../../src/sdk/python/zhilicon/sovereign.py).

## Error handling

The SDK follows the C++ runtime's convention of `Result<T>` at the
C-binding boundary. At the Python surface these translate to:

- Success → return value.
- Domain error → raise `zhilicon.ZhiliconError` with the numeric error
  code attached.
- Missing compiled extension → raise `NotImplementedError` with a link to
  the install docs.

## Version compatibility

| SDK minor      | `__backend_kind__` available             | Supported chip revisions                     |
| -------------- | ---------------------------------------- | -------------------------------------------- |
| 0.1.x          | `emulation` (numpy)                      | Pre-silicon reference; numpy-backed kernels  |
| 0.2.x (future) | `emulation` + `native` per-kernel switch | A0 silicon for Sentinel-1                    |

The `__backend_kind__` module attribute on `zhilicon._kernels` is the
authoritative signal for which backend is linked in a given build.
Consumers that care about performance fidelity (benchmark harness,
dashboard ingest, release-gate checks) must inspect it before
recording numbers — see
[ADR-0013](../../adr/ADR-0013-kernels-emulation-backend.md).
