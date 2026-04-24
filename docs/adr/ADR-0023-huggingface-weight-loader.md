---
adr-id: ADR-0023
title: 'HuggingFace SafeTensors weight loader for MinimalLlama'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/devrel", "@zhilicon/security-coe"]
---

# ADR-0023: HuggingFace SafeTensors weight loader for MinimalLlama

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, DevRel, Security COE

## Context

[ADR-0015](ADR-0015-minimal-llama-reference-model.md) shipped
`MinimalLlama` with random-init weights — enough to demonstrate the
kernel library composes into a transformer, not enough to demonstrate
anything useful to a prospective sovereign buyer. Every demo currently
produces byte-level gibberish.

[ADR-0017](ADR-0017-model-persistence-format.md) chose our own `.npz`
format for saving and loading MinimalLlama checkpoints. That format
is clean and safe, but no one else produces models in it. The
universe of trained decoder-only LMs ships as HuggingFace
checkpoints: `config.json` + one or more `model.safetensors` shards.
Without a loader, a sovereign buyer asking "can we run LLaMA-3-8B
through your kernels" gets a "not yet" — the worst possible answer
for a product that otherwise demonstrates an end-to-end sovereign
inference stack.

## Decision

Ship `zhilicon.models.hf_loader` — a read-only loader for
HuggingFace LLaMA-family checkpoints. Uses **safetensors exclusively**;
no support for legacy `.bin` files. Attaches
`MinimalLlama.from_huggingface(path)` as a classmethod so the
ergonomic API matches the rest of our model class.

### What the loader does

1. Reads `config.json` → `MinimalLlamaConfig.from_huggingface_dict`.
2. Validates `model_type ∈ {llama, qwen2, mistral}`. Exotic
   variants (Mixtral MoE, Gemma-specific scaling) raise `ValueError`
   — they need a separate loader with architecture-specific
   logic.
3. Discovers safetensors shard(s) — single `model.safetensors` or
   multi-shard via `model.safetensors.index.json`.
4. Reads each shard via the `safetensors` package if installed,
   otherwise via a 30-line built-in reader that parses the
   documented safetensors header + raw tensor bytes. No
   object-deserialisation path anywhere.
5. Remaps HuggingFace weight names to our dotted names — mostly
   stripping the `model.` prefix. Weight *layouts* (e.g.
   `[out_features, in_features]`) match exactly because we adopted
   the PyTorch convention from ADR-0015.
6. Coerces float dtype to float32 (our MinimalLlama storage dtype).
   BFloat16 is widened via uint16 → high-16 reinterpret cast (no
   ml_dtypes dependency).
7. Constructs a `MinimalLlama` with the checkpoint's config and
   calls `load_state_dict` with `strict=not tie_word_embeddings`
   (tied models omit `lm_head.weight` from the HF dump).

### What the loader deliberately does NOT do

1. **Load HF tokenizers.** Our `ByteTokenizer` has vocab=259. Real
   LLaMA has 32k (LLaMA-2) or 128k (LLaMA-3). The loader warns on
   mismatch; `strict_vocab=True` raises. End-to-end text generation
   with a loaded HF model requires the caller to supply a real BPE
   tokenizer (`sentencepiece`, HF `tokenizers` library, or a
   vendored GGUF tokenizer spec).
2. **Load legacy `.bin` files.** They use an unsafe object-
   deserialisation format that is a code-execution surface for
   untrusted checkpoints. We will not support it; callers with a
   `.bin`-only checkpoint convert via HF's
   `transformers-cli convert-safetensors` or a short Python script.
3. **Load quantised weights.** GGUF Q4_K_M, bitsandbytes Int8,
   GPTQ — all out of scope. A future kernel-library batch that
   introduces INT4/INT8 native kernels makes quantised loading
   meaningful; until then the HF loader accepts F32 / F16 / BF16
   only.
4. **Write HF-format checkpoints.** Symmetry would be nice; it is
   not the priority bottleneck. `MinimalLlama.save()` produces our
   `.npz` format. A future `save_as_huggingface()` is a trivial
   extension.

### Safety posture

- No unsafe object-deserialisation anywhere in the load path.
- Safetensors is the only accepted on-disk format.
- The built-in safetensors reader is 50 lines of numpy — auditable.
- `allow_pickle` is never set on any `np.load` call anywhere in the
  loader.

## Consequences

### Positive

- **Sovereign buyer demo unlocks.** A prospective customer can now
  download LLaMA-3.2-1B (the smallest current LLaMA family model) or
  Qwen-2.5-0.5B, point `MinimalLlama.from_huggingface()` at the
  directory, and watch the forward pass go through Zhilicon kernels.
  Tokenizer mismatch means the output is still gibberish without a
  real tokenizer, but **the kernels are running real weights** — a
  dramatically more compelling story than "random init."
- **Pre-silicon sales motion improves.** Combined with ADR-0020's
  OpenAI chat API, a customer can post "explain this CT scan" via
  `/v1/chat/completions`, see tokens stream back through a real
  LLaMA-3.2-1B's attention kernels, and understand that when silicon
  ships, the same HTTP call goes through silicon with no code
  change.
- **Kernel numerical correctness now verifiable against HF
  transformers.** A future batch can compare Zhilicon's forward
  pass against HF `transformers.LlamaForCausalLM(...).forward()` on
  the same weights. Any divergence is a real kernel bug.
- **Weight-sharing tests.** `MinimalLlama.state_dict()` ↔ HF
  checkpoint roundtrip is now bit-exact (tested).

### Negative / Risks

- **Vocab-size mismatch is a confusion tax.** Engineers who load a
  LLaMA-3-8B expect to generate text; our `ByteTokenizer` turns
  every 8-bit token ID 0-255 into a Unicode byte, producing
  gibberish. We warn explicitly, but there will still be confused
  filed issues. Mitigation: the warning message includes the
  workaround.
- **Memory:** real LLaMA-3-8B is 16 GB at FP16. On the emulation
  backend, loading succeeds but a forward pass takes ~hours. We
  document this clearly. A "ready-to-run" demo uses LLaMA-3.2-1B
  (2.4 GB).
- **Dependency on safetensors when available.** The built-in
  fallback is 30 lines but has not been stress-tested against every
  weird tensor layout HF emits. If a corner case trips, the
  error is clear (e.g., `unsupported safetensors dtype F8_E4M3`).

### Neutral

- **Tested with fabricated checkpoints only.** CI does not download
  a real HF model. Fabricating a tiny LLaMA-shaped safetensors dump
  from our own `state_dict()` gives bit-exact round-trip validation
  — that is the strongest correctness signal we can deliver without
  a 2-GB CI dependency.

## Alternatives considered

### Option A: Also support legacy `.bin` checkpoints

Rejected unambiguously. Loading `.bin` files would require flipping
the unsafe deserialisation flag and accepting arbitrary code
execution from any downloaded checkpoint. The security hook on this
repo correctly rejects that pattern; we agree with the hook.
Callers with `.bin`-only checkpoints do a one-line conversion to
safetensors via `transformers-cli`.

### Option B: Wrap the HF `transformers` library

Attractive on day one — `transformers.LlamaForCausalLM.from_pretrained()`
does all of this already. Rejected because:
- `transformers` is a 200-MB dependency that pulls in `torch`,
  `tokenizers`, `safetensors`, and more transitive tree than the
  entire Zhilicon SDK.
- It binds us to HF's release cadence and API changes.
- The one thing we actually need (read safetensors + rename keys)
  is 200 lines of numpy + json.

### Option C: Convert HF checkpoints to our `.npz` format upfront

Doable, but every customer would have to run a converter once per
checkpoint before using MinimalLlama. That is friction. Direct HF
loading is more ergonomic, and the runtime cost is a single pass
over the safetensors bytes — the same cost as any other load.

## References

- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [ADR-0017: Model persistence format](ADR-0017-model-persistence-format.md)
- [ADR-0014: Kernel library v1 canonical set](ADR-0014-kernel-library-v1-canonical-set.md)
- [`src/sdk/python/zhilicon/models/hf_loader.py`](../../src/sdk/python/zhilicon/models/hf_loader.py)
- [`src/tests/sdk/python/test_hf_loader.py`](../../src/tests/sdk/python/test_hf_loader.py)
- SafeTensors format spec: <https://github.com/huggingface/safetensors>
- Batch-37 session history.
