---
adr-id: ADR-0027
title: 'Chiplet simulator + tensor-parallel dispatch — Prometheus 8-chiplet gap mitigation'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/prometheus-leads", "@zhilicon/platform-team"]
---

# ADR-0027: Chiplet simulator + tensor-parallel dispatch — Prometheus 8-chiplet gap mitigation

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Prometheus leads, Platform team

## Context

The portfolio production-readiness audit (2026-04-18) flagged
Prometheus's largest open execution item as:

> "**8-chiplet (88B-gate) system emulation not yet scheduled** —
> at $150K and 8 weeks, this must land pre-tape-out to validate
> NoC under full load."

And the strategic synthesis memo identified Prometheus's
"dual-foundry datacenter AI with silicon-level sovereign
attestation" as the #1 commercial-leverage axis in the portfolio —
but with the caveat that *no software demo exists today* that
exercises the 8-chiplet architecture. A prospect asking "can you
show it running a 70B-class model?" receives a 12-month-deferred
answer.

Two dependent gaps:

1. **Multi-chiplet coherence contract is untested in software.**
   The NoC, scatter/gather, and all-reduce semantics that
   Prometheus's silicon will implement have no Python reference.
   Silicon teams lose the diff-baseline they need for bring-up.
2. **Tensor-parallel inference has no SDK surface.** Modern LLMs
   (70B, 400B class) do not fit in a single chiplet. The software
   layer needs a TP-split Linear and attention / MLP pattern that
   a customer can run today on emulation and that transitions to
   silicon unchanged.

This batch is the software-first fix for both.

## Decision

Ship a new `zhilicon.chiplet` subpackage with three layers:

### 1. :class:`ChipletFabric` — simulated N-chiplet NoC

- Four collective primitives — `scatter`, `gather`, `all_gather`,
  `all_reduce` — matching what NCCL / Megatron-LM / vLLM use.
- Topology-agnostic for correctness; topology-aware (torus_2x2x2,
  full_mesh, ring) for latency/bandwidth modelling via
  :meth:`ChipletFabric.estimate_bandwidth_cost`.
- :class:`FabricStats` counters: scatter / gather / all_gather /
  all_reduce call counts + cumulative bytes + simulated latency
  in nanoseconds. Used to compare TP vs. PP partition costs
  pre-silicon.
- **Determinism contract:** `all_reduce` uses a left-fold sum,
  `(shards[0] + shards[1]) + shards[2] + ...`, so output is
  byte-identical across runs. Silicon tree-reduction may diverge
  by ~1 ULP on extreme inputs; that divergence is within the
  documented tolerance.

### 2. :class:`TPLinear` — tensor-parallel dense layer

Two Megatron-LM orientations:

- **Column-parallel** (split the `out` dim): per-chiplet matmul,
  gather along the last axis. Bit-identical to single-chip.
- **Row-parallel** (split the `in` dim): scatter input, per-chiplet
  matmul, all-reduce. Numerically equal to single-chip within the
  left-fold determinism bound — tested at `rtol=1e-4, atol=1e-5`.

Input is an un-sharded tensor; the layer handles scatter →
compute → combine internally, so the caller never manages shards
directly. Activation (none / relu / gelu / silu) is applied on the
final combined output — semantics identical to single-chip.

Parametric divisibility check: `out_features % num_chiplets == 0`
for column-parallel; `in_features % num_chiplets == 0` for
row-parallel. Raises `ValueError` on mismatch rather than
silently zero-padding (which would be the #1 source of subtle TP
bugs).

### 3. :class:`MinimalLlamaDistributed` — end-to-end split model

Wraps a :class:`MinimalLlama` and splits every transformer block's
projections across the fabric in the standard Megatron pattern:

- **Attention:** q/k/v_proj column-parallel (head-dim subset per
  chiplet); o_proj row-parallel (all-reduces to combine).
- **SwiGLU MLP:** gate_proj + up_proj column-parallel (hidden-dim
  subset per chiplet); down_proj row-parallel.

Per block this costs **two** all-reduce operations. Column →
row alignment keeps intermediate outputs on-chiplet.

Forward pass produces output within `rtol=1e-4, atol=1e-4` of the
single-chip reference on 1 / 2 / 4 / 8 chiplets. Measured max-abs-
diff on 8 chiplets with vocab=64, dim=64, num_heads=8, num_kv=4 is
`1.2e-7` — essentially bit-identical.

## Consequences

### Positive (Prometheus gap mitigation)

- **8-chiplet emulation gap closed in software.** The audit-flagged
  "88B-gate emulation not yet scheduled" still matters for RTL
  verification, but the *design-validation* use case (does the
  coherence contract hold? does TP-split produce correct output?)
  is now answerable today without waiting 8 weeks for Palladium.
- **70B-class model demo path unlocked.** Prometheus's commercial
  thesis ("dual-foundry datacenter AI + sovereign attestation")
  now has a concrete "yes you can run a real LLaMA-scale model"
  demo. Slow on numpy emulation — but correct, and the same code
  runs on silicon when it arrives.
- **NoC coherence contract is now testable.** The `all_reduce`
  determinism test + the `MinimalLlamaDistributed` equivalence
  test together pin the functional contract. If Prometheus's
  silicon NoC regresses on either, the software reference catches
  it.
- **Silicon bring-up has a diff-baseline.** Post-tape-out, the
  bring-up team runs the same `MinimalLlamaDistributed` forward
  pass on silicon and compares logits against this reference. Any
  divergence flags a silicon / firmware regression before it hits
  production.

### Positive (SDK value)

- Integrates with :class:`MinimalLlama` from ADR-0015 without
  touching it. Zero changes to the single-chip reference model.
- Works today on the emulation backend. No new C++ extensions
  required; pure numpy + existing kernel library.
- Users who want multi-chiplet inference today get it; users who
  don't pay zero cost (package is opt-in import).

### Negative / Risks

- **Simulation-phase latency numbers are indicative, not
  measured.** `FabricStats.total_latency_ns` uses a simple
  bandwidth + per-hop-latency model. Real silicon performance
  depends on queue contention, sequenced-coherence stalls, and
  memory-system interplay that the simulator does not model. The
  stats exist for relative ranking of partition strategies, not
  absolute perf prediction.
- **Left-fold determinism is a trade-off.** Real silicon tree-
  reduction is log(N) latency; our left-fold is O(N). For large
  chiplet counts this diverges from what silicon will do. The
  functional correctness is preserved; only the latency model is
  conservative.
- **LM head is single-chiplet.** The vocab-dimension output layer
  is not split in v1. For very large vocab (32k+ LLaMA), this is
  a single-chiplet memory hit. TP-splitting the LM head is a
  straightforward extension; deferred to a v2 batch when vocab
  becomes the bottleneck.

### Neutral

- Pipeline-parallelism (assign different layers to different
  chiplets) is a different partition strategy with its own
  trade-offs. Not in this batch; the TP-only split covers the
  Prometheus commercial case (single-socket-per-model) cleanly.
  PP + TP hybrids are the v2 story.

## Alternatives considered

### Option A: Wait for Palladium emulation

Rejected. The audit-flagged 8-week + $150K cost is real; paying
it does not remove the need for a *software* reference the
silicon team can diff against. Software-first is strictly
complementary.

### Option B: Use `deepspeed` or `torch.distributed` for TP

Rejected. Both are excellent for their intended workloads (real
GPU clusters + NCCL) but inappropriate as a reference: they
assume a specific backend (NCCL, MPI, UCX) and a specific
execution context (torch distributed process group). We need a
simulator that runs in a single Python process with numpy, so
the reference is auditable top-to-bottom and runs in CI without
multi-process setup.

### Option C: Build the chiplet surface on `zhilicon.distributed`

Rejected. The existing `zhilicon.distributed` module is multi-
*node* DDP (process groups, remote collectives). Chiplet-level
TP is a different layer with different semantics — a single
process, shared memory, synchronous dispatch. Putting both in
one namespace would conflate very different concerns.

### Option D: Implement as a pybind11 C++ extension

Rejected for v1. A numpy reference ships in an afternoon and
runs in CI without compile steps. When the native backend lands
(silicon), the C++ dispatch path takes over — the Python layer
remains a reference only.

## References

- [ADR-0007: Prometheus Intel 18A primary with Samsung SF2 parallel track](ADR-0007-prometheus-intel-18a-with-sf2-parallel.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [ADR-0016: KV cache design](ADR-0016-kv-cache-design.md)
- [`src/sdk/python/zhilicon/chiplet/`](../../src/sdk/python/zhilicon/chiplet/)
- [`src/tests/sdk/python/test_chiplet.py`](../../src/tests/sdk/python/test_chiplet.py)
- [`docs/strategy/EXECUTION_TO_A_PLUS_ROADMAP.md`](../strategy/EXECUTION_TO_A_PLUS_ROADMAP.md) — Prometheus gates updated to reflect chiplet-SDK mitigation
- Megatron-LM paper: <https://arxiv.org/abs/1909.08053>
- Portfolio audit 2026-04-18 (this session): Prometheus §8-chiplet-emulation gap line item
- Batch-41 session history.
