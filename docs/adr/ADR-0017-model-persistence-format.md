---
adr-id: ADR-0017
title: 'Model persistence format + HuggingFace config mapping'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/devrel", "@zhilicon/security-coe"]
---

# ADR-0017: Model persistence format + HuggingFace config mapping

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, DevRel, Security COE

## Context

[ADR-0015](ADR-0015-minimal-llama-reference-model.md) and
[ADR-0016](ADR-0016-kv-cache-design.md) built the minimal LLaMA model
with random-init weights and an in-memory-only API. That unlocks demos
but not real workflows — every adoption story we hear starts with "can
you load LLaMA-3-8B on this?"

Getting from random init to real weights needs two things:

1. **A persistence format.** A way to save a model, reload it, and
   verify the forward pass is identical. Without this, every checkpoint
   conversation is "well, it should work..."
2. **A HuggingFace config mapping.** LLaMA / Qwen / Mistral / Gemma —
   the entire decoder-only family — publishes weights under the HF
   format. Mapping their `config.json` onto our
   :class:`MinimalLlamaConfig` is the gate that lets us say "yes, this
   model fits" before trying to load a 16 GB checkpoint.

## Decision

### Persistence format

**Single-file `.npz` archive.** One archive = one model checkpoint.

Contents:

- Every weight as a named numpy array under its PyTorch-compatible
  dotted name (`embed_tokens.weight`, `layers.0.self_attn.q_proj.weight`,
  `norm.weight`, etc.). Name convention matches the HuggingFace LLaMA
  state dict exactly where possible so a future HF loader has minimal
  name mapping to do.
- Sidecar `__zhilicon_config_json__`: the architecture config, JSON-
  encoded, stored as a `uint8` array of UTF-8 bytes.
- Sidecar `__zhilicon_format_version__`: an `int64` scalar for forward-
  compatibility checks during load.

### Loader runs with the safe-deserialisation flag off

Archives load with `np.load(path, allow_pickle=False)`. Any attempt to
store a Python object via `dtype=object` would require the unsafe
loader flag and is rejected by the security contract.

This is why the config is stored as a uint8 byte array rather than an
object-dtype array — the latter would need the unsafe loader. An
attacker who pushes a modified checkpoint to a shared drive cannot
execute code on `MinimalLlama.load(path)`.

A test (`test_archive_contains_no_object_dtype_arrays`) asserts the
security property so it cannot silently regress.

### state_dict / load_state_dict are first-class

`MinimalLlama.state_dict()` returns a flat `Dict[str, np.ndarray]`.
`MinimalLlama.load_state_dict(sd, *, strict=True)` writes a compatible
dict back.

- `strict=True` (default) raises on any missing or extra key.
- `strict=False` skips unknown keys — useful when loading a superset
  checkpoint that also carries optimiser state or extra modules.
- Shape mismatches always raise regardless of `strict`.
- Dtype coercion happens at load: `float16` inputs cast to `float32`
  for storage. Non-float arrays (absent in v1 but reserved for future
  use) are copied as-is.

### HuggingFace config mapping

`MinimalLlamaConfig.from_huggingface_dict(hf_cfg)` — one-shot converter
from a HF `config.json` dict to our config. Mapping:

| HuggingFace              | MinimalLlama             | Default if absent |
| ------------------------ | ------------------------ | ------------------ |
| `vocab_size`             | `vocab_size`             | 128                |
| `hidden_size`            | `dim`                    | 128                |
| `num_hidden_layers`      | `num_layers`             | 2                  |
| `num_attention_heads`    | `num_heads`              | 4                  |
| `num_key_value_heads`    | `num_kv_heads`           | = num_heads        |
| `intermediate_size`      | `hidden_dim`             | 256                |
| `rms_norm_eps`           | `norm_eps`               | 1e-6               |
| `max_position_embeddings`| `max_seq_len`            | 512                |
| `rope_theta`             | `rope_base`              | 10000.0            |
| `tie_word_embeddings`    | `tie_word_embeddings`    | True               |

Unknown HF keys are silently ignored — the goal is "make the common
case work", not to be a full HF config parser.

## Consequences

### Positive

- `model.save(path)` + `MinimalLlama.load(path)` roundtrip is
  **bit-identical**. Tests pin this at `rtol=0.0, atol=0.0`.
- HF `config.json` maps to our config in one call. LLaMA-2-7B,
  LLaMA-3-8B, Qwen, Mistral all fit without further changes.
- Loader is safe by default. No unsafe deserialisation flag anywhere
  in the code path.
- PyTorch-compatible key names mean the eventual HF weight loader is
  mostly a rename table, not a schema translation.

### Negative / Risks

- `.npz` is uncompressed by default. A 7B model is ~13 GB FP32 — large
  for a single file, but acceptable for v1. A future
  `save_compressed(path)` would use `np.savez_compressed` at the cost
  of load-time CPU. Deferred to a v2 ADR.
- Config is stored as JSON bytes rather than a typed record. A
  malformed archive (hand-edited JSON) raises at load time rather
  than at save time. Acceptable — archives are produced by
  `model.save()`, not hand-typed, and the version field catches
  format-level divergence.

### Neutral

- `int64` for the version field is overkill (we will never have
  `2**63` format versions) but is numpy-native without awkward
  casting, and the archive overhead is 8 bytes.

## Alternatives considered

### Option A: One file per weight + JSON manifest

Rejected. A 32-layer model has hundreds of weights; filesystem ops
scale poorly, and transferring a checkpoint becomes a directory
copy. Single-file is what HF `.safetensors` does for the same reason.

### Option B: `.safetensors` format

Attractive — HF's native format, zero-copy mmap, format-enforced no
code execution. Rejected for v1 only because it adds a `safetensors`
dependency; the numpy `.npz` path is good enough while we are building
the SDK's outer ring. A v2 ADR may promote `.safetensors` once the
trade-off tips.

### Option C: Store config via an object-dtype numpy array

Rejected on security grounds. Loading an object-dtype array requires
an unsafe loader flag which would be a standing invitation for
deserialisation attacks on shared checkpoints. Storing config as a
uint8 byte array uses the safe loader path with no loss of
functionality.

### Option D: Keep config as a separate JSON sidecar

Rejected for single-file ergonomics. A checkpoint should be one file
a user can email, mount, or upload. The uint8 encoding inside the
archive gives the same safety property without the two-file
inconvenience.

## References

- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [ADR-0016: KV cache design](ADR-0016-kv-cache-design.md)
- [`src/sdk/python/zhilicon/models/minimal_llama.py`](../../src/sdk/python/zhilicon/models/minimal_llama.py)
- [`src/tests/sdk/python/test_minimal_llama_persistence.py`](../../src/tests/sdk/python/test_minimal_llama_persistence.py)
- Batch-21 session history.
