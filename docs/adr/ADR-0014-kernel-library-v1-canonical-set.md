---
adr-id: ADR-0014
title: 'Kernel library v1 canonical set'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/opensilicon-coe"]
---

# ADR-0014: Kernel library v1 canonical set

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, OpenSilicon COE

## Context

[ADR-0011](ADR-0011-kernels-as-sibling-extension.md) established the
`_kernels` sibling extension. [ADR-0013](ADR-0013-kernels-emulation-backend.md)
shipped the emulation backend for a subset of kernels and explicitly
deferred attention to a follow-up batch. The SDK-layer examples, the
DevRel tutorials, and downstream training-loop integrations all need a
*named* set of kernels that is stable enough to build documentation
against but small enough to finish in one pass.

Every week the question "is X in the kernel library?" comes up with a
different X. Without a written set it gets answered case-by-case, and
the SDK team ends up shipping ad-hoc one-offs that the compile team then
has to port to hardware. The v1 set below is the answer: these kernels
are in; everything else is out until v2.

## Decision

The canonical v1 set — what ships today in the emulation backend and
what the native backend is committed to deliver before v1.0 — is:

### DNN kernels (`zhilicon._kernels.dnn`)

| Kernel                      | Emulation (numpy) | Native (silicon) | Rationale                                                           |
| --------------------------- | ----------------- | ---------------- | ------------------------------------------------------------------- |
| `flash_attention`           | Computes (naive)  | Planned (tiled)  | Every decoder-only transformer.                                     |
| `grouped_query_attention`   | Computes (naive)  | Planned (tiled)  | The modern LLM default (LLaMA-3, Qwen-2, Mistral, DeepSeek).        |
| `layernorm`                 | Computes          | Planned          | Classical transformer norm. GPT-2 / original BERT layouts.          |
| `rmsnorm`                   | Computes          | Planned          | Modern decoder-only norm. LLaMA family.                             |
| `rope`                      | Computes          | Planned          | Position embedding for every modern LM.                             |
| `softmax`                   | Computes          | Planned          | Needed outside attention too (classifier heads, MoE routing).       |
| `cross_entropy_loss`        | Computes          | Planned          | Training inner loop. log-sum-exp fused.                             |

### BLAS kernels (`zhilicon._kernels.blas`)

| Kernel               | Emulation (numpy) | Native (silicon) | Rationale                                                            |
| -------------------- | ----------------- | ---------------- | -------------------------------------------------------------------- |
| `gemm_epilogue`      | Computes          | Planned          | `activation(a @ b + bias)` — a.k.a. the backbone of every MLP.       |
| `gemm_silu_gate`     | Computes          | Planned          | SwiGLU / GeGLU. LLaMA-family gated MLPs.                             |
| `linear`             | Computes          | Planned          | PyTorch `nn.Linear` convention (fused transpose). Projection layers. |

### Explicit non-goals for v1

These are *not* in the library; a v2 ADR will revisit as downstream
workloads surface the need:

- **Fused MoE routing + gating.** Belongs with the MoE-specific
  expert-choice kernels; scheduling-heavy, too chip-specific for v1.
- **Quantisation / dequantisation kernels.** (FP8 weight pack,
  INT4 group-quantised GEMM, etc.) Requires an ADR on quantisation
  formats first.
- **Rotary-embedding cache.** Precomputed cos/sin tables across a
  model; needs a lifecycle story (who owns / invalidates the cache)
  that v1 is too small to carry.
- **Batched GEMV / SpMV.** Sparse and ragged-batch variants —
  legitimate v2 material, no urgent v1 user.
- **Reduction primitives** (sum / mean / variance as standalone
  kernels). The current norm kernels fold their own reductions; a
  standalone reduction surface duplicates that without adding
  operator-level throughput.
- **Dropout.** Trainer-side concern; the native attention kernel will
  accept a dropout_p internally when the training pipeline is ready.

### Attention in the emulation backend

The prior ADR (0013) left `flash_attention` and
`grouped_query_attention` stubbed on the emulation path. v1 flips them
to a **naive O(N²) reference**:

```text
scores = einsum("bihd,bjhd->bhij", q, k) / sqrt(d_k)
if causal: scores = where(lower_triangular, scores, -inf)
attn   = softmax(scores, axis=-1)
out    = einsum("bhij,bjhd->bihd", attn, v)
```

For GQA, `k` and `v` are first expanded along the head axis via
`np.repeat` by the group ratio (`h_q // h_kv`), then the identical SDPA
math runs.

This is deliberately naive — no tiling, no online softmax, no KV
re-materialisation. Correctness first; the emulation backend is not
performance-sensitive by policy (ADR-0013 guard).

`return_lse=True` is accepted by the Python surface but raises in the
emulation backend — the log-sum-exp return value is the signature of
the *tiled* FlashAttention v2 algorithm, which lives in native code.

### Signature contracts

Every kernel in v1 respects:

1. **Python surface is stable.** Keyword-only arguments after the first
   2–3 positional parameters. A new kernel option in v2 becomes a new
   kwarg; argument reordering would be a breaking change that requires
   an ADR supersede.
2. **Dtype round-trip.** The dispatcher casts to FP64 internally,
   computes, then casts back to the input dtype. Callers see stable
   dtypes in, stable dtypes out — even when the numerical accumulation
   matches the "FP32 accumulate" native-kernel contract.
3. **Shape validation before compute.** Every kernel validates shapes
   and argument ranges on any object with a `.shape` attribute before
   branching into emulation vs stub — so a shape bug raises the same
   error on native and emulation backends.

## Consequences

### Positive

- The SDK-layer docs (`docs/src/reference/python-sdk.md`), the how-to
  (`docs/src/how-to/run-kernel-emulation.md`), and the benchmark
  harness all have a fixed v1 surface to name. No more "is X in?"
  churn.
- Downstream integration shims (PyTorch adapter, model-hub loaders)
  know what they can lean on.
- The native-kernel team has a concrete, ranked, small list to
  implement. v1 is seven DNN kernels + three BLAS kernels — nine
  distinct native kernels counting attention's shared implementation.

### Negative / Risks

- Someone will need a kernel that's not in v1 before v2 lands.
  Mitigation: the Python surface is still pybind11 — adding a kernel
  is a PR, not a redesign. The ADR is about commitment, not
  obstruction.
- The naive attention reference is `O(N²)` in compute and `O(N²)` in
  memory. Sequence lengths above ~4096 in the emulation backend will
  OOM on a laptop. Mitigation: documented in
  [`docs/src/how-to/run-kernel-emulation.md`](../src/how-to/run-kernel-emulation.md)
  and the benchmark harness cases cap at seq=256 for attention.

### Neutral

- The set aligns with what a production LLaMA-3 or Qwen-2 inference
  pass needs. Decoder-only transformer training needs the same set
  plus dropout; the native kernel will pick up dropout when the
  trainer pipeline lands.

## Alternatives considered

### Option A: Ship nothing — wait for native kernels

Rejected. The SDK-layer docs and tutorials need a runnable surface
now. The emulation backend cost is bounded (naive numpy reference per
kernel), and the ADR-0013 guard keeps laptop-speed numbers off the
dashboard.

### Option B: Ship *every* kernel a transformer model might touch

Rejected. Grows the surface uncontrollably: embedding-lookup,
positional-encoding-cache, prefix-caching, flash-decoding,
continuous-batching dispatch... the list is unbounded. v1 is the
smallest credible closure.

### Option C: Keep attention stubbed indefinitely

Rejected. The SDK examples, tutorials, and downstream LLM integrations
all bottleneck on attention. Shipping emulation-path SDPA unblocks
every one of them without committing to a native release date.

## References

- [ADR-0011: `_kernels` as a sibling pybind11 extension](ADR-0011-kernels-as-sibling-extension.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [`src/sdk/python/zhilicon/_kernels_dnn.cpp`](../../src/sdk/python/zhilicon/_kernels_dnn.cpp)
- [`src/sdk/python/zhilicon/_kernels_blas.cpp`](../../src/sdk/python/zhilicon/_kernels_blas.cpp)
- [`src/tests/sdk/python/test_kernels_numerics.py`](../../src/tests/sdk/python/test_kernels_numerics.py)
- [`src/tools/bench/kernels_bench.py`](../../src/tools/bench/kernels_bench.py)
- Batch-17 session history.
