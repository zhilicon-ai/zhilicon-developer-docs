# Build a minimal transformer with the kernel library

This page walks you through running a minimal LLaMA-style transformer
forward pass on a developer laptop, using only numpy + the v1 kernel
library. The whole model is ~30 lines of user-facing code; everything
else is inside
[`zhilicon/models/minimal_llama.py`](../../../src/sdk/python/zhilicon/models/minimal_llama.py).

> **Who this is for.** You have the kernel library working (see
> [Run the kernel library on a laptop](run-kernel-emulation.md)) and
> want to see the kernels compose into a real model.
>
> **What this is not.** The model ships with random weights — there
> is no training, no HuggingFace checkpoint loading, no KV cache, no
> backward pass. See [ADR-0015](../../adr/ADR-0015-minimal-llama-reference-model.md)
> for the scope decision. Logits from a random-init model are
> random-init garbage; this page demonstrates architecture, not
> intelligence.

## Prerequisites

- `pip install -e src/sdk/python -v` completed successfully (the
  emulation walk-through is a strict prerequisite).
- numpy ≥ 1.24.

## Minimal example

```python
import numpy as np
from zhilicon.models import MinimalLlama, MinimalLlamaConfig

# 2 layers × 4 heads × 2 KV heads × 128-dim = ~100K parameters.
cfg = MinimalLlamaConfig(
    vocab_size   = 128,
    dim          = 128,
    num_layers   = 2,
    num_heads    = 4,
    num_kv_heads = 2,
    hidden_dim   = 256,
    max_seq_len  = 64,
)

model = MinimalLlama(cfg, seed=0)
print(f"parameters: {model.parameter_count():,}")

token_ids = np.array([[1, 2, 3, 4, 5, 6, 7, 8]], dtype=np.int64)
logits = model(token_ids)
print(f"logits shape: {logits.shape}")
print(f"logits dtype: {logits.dtype}")
```

Expected output:

```text
parameters: 99,712
logits shape: (1, 8, 128)
logits dtype: float32
```

## What the kernels are doing

Each call to `model(token_ids)` runs:

1. **Embedding lookup** — plain numpy indexing into the
   `[vocab_size, dim]` weight table.
2. **Per-block × N:**
   - `RMSNorm` → wraps
     [`zhilicon.kernels.norm.rmsnorm`](../reference/python-sdk.md#zhiliconkernels).
   - Q/K/V projections → each is a
     [`linear`](../reference/python-sdk.md#zhiliconkernels) call.
   - `RotaryEmbedding` on Q and K → wraps
     [`rope`](../reference/python-sdk.md#zhiliconkernels) with
     precomputed cos/sin tables in `(seq, 1, head_dim/2)` layout.
   - `grouped_query_attention` → the v1 attention kernel with
     `num_kv_heads` < `num_heads`.
   - Output projection → another `linear`.
   - Second `RMSNorm`.
   - `SwiGLUMLP` → two `linear` projections + numpy SiLU + a third
     `linear`.
3. **Final `RMSNorm`** → prepares hidden states for the LM head.
4. **LM head** → either another `linear` (when
   `tie_word_embeddings=False`) or a direct `linear(x, embedding.weight)`
   using the tied embedding matrix.

Every one of those kernel calls goes through the emulation backend
because the inputs are numpy arrays (see ADR-0013 dispatcher logic).
On silicon the same Python code hits the native backend — no changes
required.

## Sampling from the logits

The model returns raw logits. To get probabilities:

```python
from zhilicon.kernels.activation import softmax

probs = softmax(logits[:, -1, :], axis=-1)   # last-position probability
next_token = int(np.argmax(probs, axis=-1)[0])
```

Greedy sampling on random weights will produce the token with the
highest random logit — useful for verifying the pipeline, useless for
generating text. Replace random-init with real weights (HuggingFace
loader, future batch) and the same sampling code produces real
completions.

## Autoregressive generation

For a full generation loop, use the built-in `generate()` method — it
wraps a prefill over the prompt and a series of incremental-decode
steps, each reusing the KV cache:

```python
prompt = np.array([[1, 2, 3, 4, 5, 6, 7, 8]], dtype=np.int64)

output = model.generate(prompt, max_new_tokens=10)
print(f"output shape: {output.shape}")    # (1, 18)
print(f"generated:   {output[0, 8:].tolist()}")
```

With an optional end-of-sequence token:

```python
output = model.generate(prompt, max_new_tokens=100, eos_token_id=42)
# Generation stops early if the model emits token 42.
```

Under the hood, `generate()` calls:

1. `logits, cache = model.prefill(prompt)` — one forward pass over the
   prompt, populating a fresh :class:`KVCache` for every layer.
2. `logits = model.decode(new_token, cache)` — one forward pass over
   the newest token, appending its K / V to the cache and running
   attention against the accumulated sequence. Repeat.

For sampling beyond greedy, pass a `SamplingConfig` into `generate()`:

```python
from zhilicon.models import SamplingConfig

# Temperature sampling — higher temperature = flatter distribution.
output = model.generate(
    prompt, max_new_tokens=20,
    sampling=SamplingConfig(temperature=0.8),
    seed=42,
)

# Top-k: restrict candidates to the k most probable tokens.
output = model.generate(
    prompt, max_new_tokens=20,
    sampling=SamplingConfig(temperature=0.9, top_k=40),
    seed=42,
)

# Top-p (nucleus): keep the smallest set of tokens whose cumulative
# probability >= top_p.
output = model.generate(
    prompt, max_new_tokens=20,
    sampling=SamplingConfig(temperature=0.9, top_p=0.95),
    seed=42,
)

# All three combined — a typical "creative-but-coherent" LLM config.
output = model.generate(
    prompt, max_new_tokens=20,
    sampling=SamplingConfig(temperature=0.8, top_k=40, top_p=0.95),
    seed=42,
)
```

The `seed` kwarg controls reproducibility — same seed → same output.
Omit it to get fresh tokens each call.

### Sampling correctness guarantees

- `SamplingConfig.greedy()` (default when `sampling=None`) always
  returns `argmax`.
- `temperature=0.0` short-circuits to `argmax` regardless of top-k /
  top-p values — avoids dividing by zero inside the softmax.
- `top_p < 1.0` always keeps at least the head token, so the nucleus
  is never empty and the renormalised probabilities never contain NaN.
- The RNG state is seeded once per `generate()` call, so each step in
  a single call draws from a consistent sequence.

### Correctness guarantee

The two paths `model(prompt + extra)` and
`prefill(prompt) + decode(extra)` are **bit-identical**. The test
suite pins this with `atol=1e-5`; locally the max absolute diff is
zero. If you see divergence, the KV cache has desynced — open an
issue.

### Performance caveat

The emulation backend uses `np.concatenate` to grow the cache, which
is `O(cache_length)` per step — generating `N` tokens is quadratic in
`N`. This is fine for laptop demos (generating 100 tokens on a
default-sized model takes < 1 second) but not a realistic measure of
silicon throughput. On silicon, the native kernel uses a preallocated
ring buffer; see [ADR-0016](../../adr/ADR-0016-kv-cache-design.md).

## Varying the architecture

The config is the only knob. Common variants:

```python
# Multi-head attention (no GQA).
cfg = MinimalLlamaConfig(num_heads=8, num_kv_heads=8, ...)

# Larger hidden dim (wider MLP).
cfg = MinimalLlamaConfig(dim=256, hidden_dim=512, num_heads=8, num_kv_heads=2, ...)

# Deeper model.
cfg = MinimalLlamaConfig(num_layers=8, ...)

# Untied LM head.
cfg = MinimalLlamaConfig(tie_word_embeddings=False, ...)
```

Check
[`MinimalLlamaConfig`](../../../src/sdk/python/zhilicon/models/minimal_llama.py)
for the full set of knobs + their validation rules.

## Causal vs non-causal attention

The default is causal (decoder-only LM). For sanity-check diagnostics
only:

```python
logits = model(token_ids, causal=False)
```

The non-causal mode lets every position see every other position —
useful for comparing against bidirectional encoders but not for
generation.

## Scaling caveat

The emulation backend uses a naive O(N²) scaled-dot-product attention.
Sequence lengths above ~4096 will OOM on a laptop. This is by design
(ADR-0013 + ADR-0014): the emulation backend prioritises correctness
over throughput. On silicon, `flash_attention` runs in O(N) memory via
the tiled kernel.

## Loading a real HuggingFace model

`MinimalLlama` can load a HuggingFace LLaMA-family checkpoint
directly — no `transformers` dependency needed. Point the loader at
a directory containing `config.json` + `model.safetensors`:

```python
from zhilicon.models import MinimalLlama

# Any LLaMA / Qwen-2 / Mistral checkpoint directory.
model = MinimalLlama.from_huggingface("/path/to/Llama-3.2-1B")
print(f"parameters: {model.parameter_count():,}")
```

The loader supports:
- Single-file `model.safetensors` and multi-shard
  `model.safetensors.index.json` layouts
- Float32, float16, and bfloat16 weight dtypes (auto-widened to float32)
- Tied or untied embeddings (config flag)
- LLaMA, Qwen-2, and Mistral `model_type` values

See [ADR-0023](../../adr/ADR-0023-huggingface-weight-loader.md) for
the full design and the list of deliberately-unsupported variants
(quantised weights, legacy `.bin` format, Mixtral MoE).

### Tokenizer caveat

Our `ByteTokenizer` has vocab=259. Real LLaMA-3 has 128,256. The
loader **warns** on vocab mismatch and **loads the architecture
anyway** — but `ByteTokenizer.encode("Hello")` no longer maps to the
tokens the model expects. The model generates gibberish unless you
supply the real BPE tokenizer.

For sales / demo scenarios the mismatch is accepted — the value is
demonstrating the kernels running real weights, not producing
grammatical text. For end-to-end text generation, vendor the
tokenizer from the HF checkpoint:

```python
# If you have `tokenizers` or `sentencepiece` installed:
from tokenizers import Tokenizer
tokenizer = Tokenizer.from_file("/path/to/Llama-3.2-1B/tokenizer.json")

prompt = "Once upon a time"
prompt_ids = np.array([tokenizer.encode(prompt).ids], dtype=np.int64)
out_ids = model.generate(prompt_ids, max_new_tokens=50)
completion = tokenizer.decode(out_ids[0].tolist())
print(completion)
```

### Memory caveat

LLaMA-3-8B at float32 is ~32 GB. Loading succeeds on a machine with
enough RAM; a forward pass on the emulation backend is slow (minutes
per token for the 8B model). Recommended demo targets:

| Model | Size (FP16) | Suitable for |
|---|---|---|
| `Llama-3.2-1B` | 2.4 GB | Laptop emulation forward pass in ~seconds |
| `Qwen2.5-0.5B` | 1 GB | Laptop emulation forward pass in <1 second |
| `Llama-3.2-3B` | 6.4 GB | Workstation emulation; minutes per forward pass |
| `Llama-3-8B` | 16 GB | Silicon target; emulation impractical |

## Next steps

- [Kernel reference](../reference/python-sdk.md) — the v1 kernel API
- [Run the benchmark harness](../../../src/tools/bench/README.md) — how
  the ADR-0013 guard refuses emulation numbers on the dashboard path
- [ADR-0014](../../adr/ADR-0014-kernel-library-v1-canonical-set.md)
  — what's in the v1 kernel set and why
- [ADR-0015](../../adr/ADR-0015-minimal-llama-reference-model.md) — why
  the minimal LLaMA exists and what's explicitly deferred
