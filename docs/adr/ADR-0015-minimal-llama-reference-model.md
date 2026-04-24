---
adr-id: ADR-0015
title: 'Minimal LLaMA as the reference model for the v1 kernel set'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/devrel"]
---

# ADR-0015: Minimal LLaMA as the reference model for the v1 kernel set

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, DevRel

## Context

[ADR-0014](ADR-0014-kernel-library-v1-canonical-set.md) froze the v1
kernel set — seven DNN kernels plus three BLAS kernels, all emulated
via numpy. The kernels pass their own unit tests and their own
benchmark-harness cases, but until they compose into something
recognisable no one outside the SDK team can tell whether the *set*
is complete.

Two concrete gaps showed up once downstream teams started trying to
use the kernels:

1. **No demonstration of composition.** DevRel wanted a "hello world"
   that takes token IDs in and produces logits out, using only the v1
   surface. The closest existing reference was
   [`zhilicon/models/decoder_lm.py`](../../src/sdk/python/zhilicon/models/decoder_lm.py)
   — but that depends on the `zhilicon.nn.Module` framework which
   requires the native `_core` extension and is Tensor-based, not
   numpy-based. Neither fits the ADR-0013 emulation backend.
2. **No surface for kernel completeness checks.** "Is the v1 set
   enough to build a transformer?" is a Yes/No question — and the
   honest answer is "we think so, but we have not tried." A runnable
   transformer turns that conjecture into a test.

## Decision

Ship a **minimal LLaMA-style transformer** at
[`src/sdk/python/zhilicon/models/minimal_llama.py`](../../src/sdk/python/zhilicon/models/minimal_llama.py)
that:

- Uses **only** the v1 canonical kernels (ADR-0014) — `rmsnorm`,
  `rope`, `grouped_query_attention`, `linear`. No direct numpy
  arithmetic except SiLU in the SwiGLU MLP (which composes `linear` +
  numpy `exp` + elementwise multiply) and softmax-for-lm-head logits
  when the caller wants probabilities (exposed separately, not inside
  the model).
- Works on **numpy arrays** end-to-end. No dependency on the native
  `_core` extension. No dependency on the `zhilicon.nn.Module`
  framework. `pip install -e src/sdk/python` is sufficient.
- Is **small**: under 300 lines including docstrings. Readers see the
  whole model architecture on one screen.
- Is **runnable** in milliseconds on a laptop at the default config
  (vocab=128, dim=128, 2 layers, 4 heads, 2 KV heads).

Architecture: standard LLaMA pre-norm decoder block.

```
    token_ids ─▶ Embedding ─▶ [block × N] ─▶ RMSNorm ─▶ LM head ─▶ logits

    block =
        x = x + GQA(RoPE(RMSNorm(x)))
        x = x + SwiGLU(RMSNorm(x))
```

GQA is the default attention variant (LLaMA-3, Qwen-2, Mistral, DeepSeek
all default to GQA). Users who want plain MHA pass `num_kv_heads =
num_heads`; the kernel dispatcher handles the degenerate case.

### Scope

**In scope for this ADR:**

- Forward pass only (no backward / training).
- Random-init weights only (no HuggingFace checkpoint loading).
- Causal / non-causal toggle.
- Tied / untied LM head.
- Tests that pin: shape preservation, dtype round-trip, determinism,
  causal-mask behaviour, parameter count.
- A how-to page
  ([`build-minimal-transformer.md`](../src/how-to/build-minimal-transformer.md))
  showing a 30-line example.

**Out of scope:**

- Backward pass / autograd. Requires an optimizer, a loss wrapper, and
  a choice of autograd strategy — a separate ADR.
- HuggingFace weight loading. Requires shape / name mapping and a
  dependency on `huggingface_hub` or an equivalent loader. Separate
  ADR.
- KV cache for incremental decoding. The forward pass is prefill-only
  today. A KV-cache wrapper is a natural follow-up once the native
  attention kernel lands.
- Quantisation. FP8 / INT4 weight packs need their own kernel set and
  a quantisation ADR first.

### Why not extend `zhilicon.nn`

`zhilicon.nn.Module` exists and is PyTorch-shaped (`Parameter`,
`Linear`, `LayerNorm`, `Embedding`, ...), but every one of its
building blocks operates on `zhilicon.Tensor` objects — which in turn
require the native `_core` extension. That coupling is the wrong
foundation for a reference that must run on a laptop with just numpy
+ the `_kernels` extension.

Rather than either (a) retrofitting `zhilicon.nn.Module` to accept
numpy arrays (invasive, breaks the existing Tensor-based surface) or
(b) waiting until the `_core` extension builds via pip (larger
scope), we ship a **parallel, purpose-built reference**. It
deliberately *doesn't* inherit from `Module` — so that the choice to
depend on the framework is visible at the class level, not implicit.

### Why minimal LLaMA specifically

- **Industry default.** LLaMA-3, Qwen-2, Mistral, DeepSeek-v3, Yi,
  Gemma, Phi — all decoder-only transformers with the same primitives
  (RMSNorm, RoPE, GQA, SwiGLU). Getting one minimal version right
  covers the overwhelming majority of modern LM workloads.
- **No boutique pieces.** The minimal LLaMA has only components that
  already exist in v1. Supporting GPT-style pre-norm + LayerNorm or
  T5-style relative position bias would require kernels not in v1 —
  explicit scope creep.
- **Demonstrable.** Users recognise the name; `MinimalLlama` plus a
  config is a shape a reader can absorb without a primer.

## Consequences

### Positive

- DevRel has a runnable hello-world. The [how-to page](../src/how-to/build-minimal-transformer.md)
  is a 30-line example, not a chapter.
- Implicit v1-completeness check. Any kernel the minimal LLaMA cannot
  implement is a v1 gap — and would surface as a Python error
  immediately, not as a production bug months later.
- New kernel additions (v2) can land tests against the minimal LLaMA
  as an integration point. "Does this kernel slot into a real model
  without breaking anything?" gets a Yes/No answer instead of a theory.

### Negative / Risks

- A user reads "MinimalLlama" and assumes this is a trained model —
  it is not. Mitigation: the how-to page opens with a non-trained
  caveat; the docstring says "random-init weights only"; ADR is
  linked from the reference docs.
- A second reference model for another architecture family (e.g.
  Mixtral-style MoE, Mamba-style SSM) would duplicate the embedding /
  norm / linear boilerplate. Mitigation: defer — if that duplication
  becomes a real problem, a small `zhilicon.models._layers` helper
  emerges at that point, not speculatively.

### Neutral

- The `zhilicon.nn.Module` framework stays the first-class surface
  for Tensor-based workflows. `MinimalLlama` is explicitly a
  laptop-local numpy reference, not a replacement.
- Parameter-storage convention inside `MinimalLlama` matches PyTorch
  (`weight.shape = [out_features, in_features]`), which gives a clean
  migration path when a future batch wires HuggingFace weight loading.

## Alternatives considered

### Option A: Ship only a benchmark-harness example

Rejected. Benchmarks measure latency; they don't demonstrate
composition. A reader of `kernels_bench.py` sees timing numbers but
not how the kernels *fit together* into a usable model.

### Option B: Extend `zhilicon.models.decoder_lm.LlamaForCausalLM`

Rejected. It is 1,000+ lines, depends on `zhilicon.nn.Module`, and
targets the `_core`-extension path. Retrofitting it to work on numpy
would be a larger change than writing a purpose-built reference.

### Option C: Defer until HuggingFace loading + training loop are ready

Rejected. A random-init forward pass is enough to pin the composition
and test the kernels; users who want a trained model can wait for the
follow-up ADRs. Shipping the minimal version now unblocks DevRel
today and doesn't close any future door.

## References

- [ADR-0011: `_kernels` as a sibling pybind11 extension](ADR-0011-kernels-as-sibling-extension.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0014: Kernel library v1 canonical set](ADR-0014-kernel-library-v1-canonical-set.md)
- [`src/sdk/python/zhilicon/models/minimal_llama.py`](../../src/sdk/python/zhilicon/models/minimal_llama.py)
- [`src/tests/sdk/python/test_minimal_llama_transformer.py`](../../src/tests/sdk/python/test_minimal_llama_transformer.py)
- [`docs/src/how-to/build-minimal-transformer.md`](../src/how-to/build-minimal-transformer.md)
- Batch-18 session history.
