---
adr-id: ADR-0016
title: 'KV cache design for incremental decoding'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/opensilicon-coe"]
---

# ADR-0016: KV cache design for incremental decoding

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, OpenSilicon COE

## Context

[ADR-0015](ADR-0015-minimal-llama-reference-model.md) shipped a
prefill-only forward pass for `MinimalLlama`. Generation requires
incremental decoding — producing one new token at a time, reusing
already-computed K/V from the attention layers instead of
recomputing. Three things need resolving:

1. **Who owns the cache?** Per-model, per-layer, or global singleton?
2. **How is the cache stored?** Growing list of numpy arrays, or a
   preallocated ring buffer with a write cursor?
3. **What does the attention kernel do with it?** In particular, the
   causal mask needs to handle `s_q < s_k` (the decode case), which
   the original kernel did not.

Getting the RoPE positional offset right turned out to be the subtle
bit: the naive approach ("ask the cache how many tokens it has")
reads `keys[0].shape[1]` at a given instant, but within a single
forward pass layer `k+1` is called *after* layer `k` has already
appended — so `keys[0]` length is off-by-``new_tokens`` for every
layer past the first.

## Decision

### 1. Cache is owned by the caller, not the model

The `MinimalLlama` model is stateless — no fields mutate during a
forward pass. The KV cache is a separate :class:`KVCache` object that
the caller passes into `model(..., kv_cache=cache)` (or via the
`prefill()` / `decode()` convenience methods).

Rationale: a model instance represents architecture + weights, not a
conversation. Two simultaneous conversations must not clobber each
other; creating two models would duplicate weights. The cache is the
right place for "what has this session seen so far."

### 2. Cache stores per-layer numpy arrays, grown by `np.concatenate`

```python
@dataclass
class KVCache:
    keys:   List[Optional[np.ndarray]]  # per-layer [b, cache_seq, h_kv, head_dim]
    values: List[Optional[np.ndarray]]
```

Each layer owns its own array. Appending an N-token step to a
cache of length L allocates a new array of length L+N and copies
the old one — `O(L+N)` time and space.

This is the **reference implementation** choice, not the production
one. A native silicon kernel will instead:

- Preallocate a cache of size `max_seq_len` at session start.
- Maintain a write cursor that advances by `N` per `decode` call.
- Never realloc.

The reference-implementation choice is justified because:

- It is 10 lines of numpy vs ~100 lines of ring-buffer bookkeeping.
- Correctness is obvious — the cache state after `N` appends is
  literally a concatenation, so the math matches "run on the whole
  sequence from scratch" by construction.
- Emulation-backend throughput is not what anyone measures (see
  [ADR-0013](ADR-0013-kernels-emulation-backend.md)); the native
  kernel gets the ring buffer when it lands.

### 3. RoPE position is read per-layer, not globally

The cache exposes two length accessors:

```python
@property
def seq_len(self) -> int:                 # reads layer 0
def layer_seq_len(self, layer_idx) -> int:  # reads the named layer
```

Attention modules **MUST** use `layer_seq_len(layer_idx)` for the
RoPE offset. Using `seq_len` is a bug: by the time layer 1's
attention runs, layer 0 has already appended, and the global `seq_len`
overshoots by ``new_tokens`` for layers past the first.

The decision to document BOTH accessors (and keep `seq_len` as the
caller-facing property) is deliberate — an external caller wants the
overall conversation length; it is the attention module that needs
the per-layer reading.

### 4. Causal-mask offset in the SDPA kernel

The C++ attention dispatcher previously built the causal mask as
`np.tri(s_q, s_k, k=0)`, which assumes `s_q == s_k`. For incremental
decode (`s_q` = new tokens, `s_k` = cached + new), this masks out
every cached position. The fix:

```cpp
int diag_offset = s_k - s_q;
auto mask = np.attr("tri")(s_q, s_k,
                            py::arg("k") = diag_offset,
                            py::arg("dtype") = np.attr("bool_"));
```

For `s_q == s_k` the offset is 0 and the mask is identical to before
(backward-compatible for every existing test). For `s_q < s_k` the
diagonal shifts right so each query sees the cached keys plus its
causally-allowed new keys.

### 5. prefill / decode / generate API

The model exposes three methods:

- `prefill(token_ids) -> (logits, cache)` — one forward pass over the
  prompt; returns both the logits (for the caller to inspect) and a
  fresh cache.
- `decode(token_ids, cache) -> logits` — run the model with an
  existing cache; mutates the cache in place.
- `generate(prompt_ids, max_new_tokens, eos_token_id) -> token_ids` —
  greedy autoregressive wrapper. Calls `prefill` once, then `decode`
  in a loop, always picking `argmax(logits[:, -1, :])`. Returns the
  full sequence (prompt + generated).

Greedy-only for v1. Temperature / top-k / top-p sampling is a
compose-over-decode pattern the caller can implement without a new
kernel — it stays out of scope for the reference model.

## Consequences

### Positive

- The model is stateless. Two `generate()` calls in parallel don't
  conflict. Checkpointing is a cache-only concern.
- The `prefill + N × decode` math is **bit-identical** to the
  full-sequence forward pass. Tests pin this with
  `atol=1e-5` (verified at `max_abs_diff = 0.0` locally).
- The kernel fix for causal-mask-with-offset is backward-compatible
  — every existing attention test still passes.
- The per-layer `layer_seq_len` accessor makes the subtle
  "layers-advance-in-lockstep" bug impossible to reintroduce silently.

### Negative / Risks

- `np.concatenate` per decode step is `O(L)` memory traffic. Naive
  generation of `N` tokens over a prompt of length `P` does
  `O((P+1)(P+2)/2)` total copies — quadratic in `P+N`. For laptop
  emulation this is fine; for silicon we explicitly plan a preallocated
  ring-buffer cache per the ADR-0014 native-kernel roadmap.
- The cache object is mutable. A caller who wants to replay
  generation must deep-copy the cache at each step — not automatic.
  Mitigation: documented in the `decode` docstring; a future
  `cache.fork()` method is a natural addition when/if branching is
  a real user need.

### Neutral

- `generate()` is greedy-only. Callers who want sampling can
  implement it over `prefill + decode`; the kernel surface supports
  both. Adding sampling as a method on `MinimalLlama` is deliberately
  deferred — temperature / top-k / top-p semantics deserve their own
  ADR.

## Alternatives considered

### Option A: Model owns the cache (stateful model)

Rejected. Two `generate()` calls with different prompts would
clobber each other. A PyTorch-style ``model.reset_cache()`` escape
hatch is ugly and still racy under concurrent use.

### Option B: Preallocated ring buffer now

Rejected for the reference implementation. The extra bookkeeping
(write cursor, wraparound, mask adjustment when the buffer is full)
does not add correctness signal over `np.concatenate`, and it is
production-path code that belongs with the native kernel. The
emulation backend should be as boring as possible to read.

### Option C: Separate cache classes per precision

Rejected. The cache only stores materialised numpy arrays; precision
is a property of the arrays, not the cache. Adding a precision tag to
the cache struct would force every caller to specify it at
construction, which is busy work that the native kernel will handle
transparently when it lands.

### Option D: Handle the RoPE-position bug inside `KVCache` (auto-advance)

Rejected. Encoding "the current layer is about to append N tokens"
into the cache leaks the attention module's control flow into the
cache struct. Reading `layer_seq_len(layer_idx)` is explicit and
obvious at the call site; the bug we had arose because `seq_len` was
*implicit*, and switching to a per-layer accessor made the correct
reading the obvious one.

## References

- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0014: Kernel library v1 canonical set](ADR-0014-kernel-library-v1-canonical-set.md)
- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [`src/sdk/python/zhilicon/models/minimal_llama.py`](../../src/sdk/python/zhilicon/models/minimal_llama.py)
- [`src/tests/sdk/python/test_minimal_llama_generate.py`](../../src/tests/sdk/python/test_minimal_llama_generate.py)
- [`src/sdk/python/zhilicon/_kernels_dnn.cpp`](../../src/sdk/python/zhilicon/_kernels_dnn.cpp) — `sdpa_numpy_fp64`
- Batch-19 session history.
