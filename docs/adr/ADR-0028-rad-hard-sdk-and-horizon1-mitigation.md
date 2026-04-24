---
adr-id: ADR-0028
title: 'Rad-hard SDK (SEU injection + TMR voter + robustness harness) — Horizon-1 gap mitigation'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/rad-hard-coe", "@zhilicon/horizon-1-leads"]
---

# ADR-0028: Rad-hard SDK (SEU injection + TMR voter + robustness harness) — Horizon-1 gap mitigation

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Rad-Hard COE, Horizon-1 silicon leads

## Context

The portfolio production-readiness audit (2026-04-18) identified
three Horizon-1 rad-hard gaps that the SDK can close *today* in
software, months ahead of silicon tape-out:

1. **No software-side fault-injection evidence.** The Horizon-1
   datasheet claims 300 krad TID, TMR-protected systolic, and a
   15 ms SEU recovery FSM (ADR-0003). These are hardware claims.
   But a defense prime performing due diligence asked "show me
   fault-injection test evidence on a real model" and the answer
   was "the silicon will handle it" — which is not an auditable
   answer before silicon exists.
2. **DO-254 DAL-B software life-cycle artefacts missing.** DO-254
   DAL-B requires traceable evidence that the *software* running
   on the rad-hard silicon has been exercised against upset events,
   with reproducible seeds, documented patterns, and a published
   FIT-rate estimate. None of that existed as a package.
3. **No pre-silicon way to quantify model robustness.** Customers
   running LEO-orbit inference workloads (Earth-observation CNNs,
   on-orbit LLM assistants, debris-avoidance policy networks) need
   a way to measure their specific model's degradation curve under
   upset events, ideally before committing to Horizon-1 silicon.

Meanwhile, the SDK already had rad-hard-adjacent needs:
- ``_kernels`` emulation (ADR-0013) benefits from a fault-injection
  hook for regression testing against numerical-stability bugs.
- The Prometheus chiplet simulator (ADR-0027) uses an all-reduce
  that would benefit from TMR-voter support for demonstrating
  redundancy-in-software.
- The rad-hard claims in the competitive matrix
  (``COMPETITIVE_POSITION_MATRIX.md``) need a concrete software
  counterpart callers can actually invoke.

Closing the H1 gap and the SDK need with one package is a dual-use
win — same pattern used for the Sentinel-1 crypto library (ADR-0025)
and the Discovery-1 medical stack (ADR-0026).

## Decision

Ship ``zhilicon.rad_hard`` — a radiation-tolerance SDK with a
deliberate dual role:

- **SDK robustness tooling** for any user running inference in a
  harsh environment (LEO / GEO orbit, high-altitude UAV, nuclear-
  plant floor, particle accelerator).
- **Horizon-1 pre-silicon evidence package** — the SEU injector,
  TMR voter, and robustness harness produce the exact artefacts the
  DO-254 DAL-B life-cycle requires.

### Module layout

| Module | Role | Backing |
|---|---|---|
| ``zhilicon.rad_hard.seu`` | SEU bit-flip injection (SBU / MCU / distributed / BER-driven patterns) | Pure numpy with uint reinterpret views for bit-level mutation |
| ``zhilicon.rad_hard.tmr`` | Triple Modular Redundancy majority voter | Pure numpy; bit-identical comparison via matching-uint view |
| ``zhilicon.rad_hard.validator`` | ``ModelRobustnessTest`` — end-to-end SEU campaign harness | Pure numpy + ``seu`` + ``tmr`` |

### SEU injector design

The injector models the four SEU event classes enumerated in the
Horizon-1 DEEP_TECHNICAL_SPEC §4.1 (heavy-ion LET envelope):

- **``SINGLE_BIT``** — single-bit upset (SBU). ~95% of LEO events.
  The default pattern for robustness testing.
- **``MULTI_BIT``** — multi-cell upset (MCU): N adjacent bits in
  one word. Models the high-LET heavy-ion case where a single
  particle strike traverses multiple storage cells. ECC that
  tolerates 1-bit-per-word still fails here; Chipkill / BCH 72,64
  may recover.
- **``DISTRIBUTED``** — N random bits scattered across the array.
  Models the accumulated-upset case (long missions between scrubs).
- **``BER_DRIVEN``** — each bit flipped i.i.d. at probability
  ``ber``. Used for Monte Carlo estimation of model FIT rate at
  a target environmental BER.

All four patterns accept an explicit ``numpy.random.Generator``.
**Determinism under seed is load-bearing** — a regression found
via fault injection must be reproducible tomorrow from its seed
alone. The test suite (``test_inject_is_deterministic_under_seed``)
guarantees this property.

The primitive is ``flip_bit(array, flat_index, bit_within_word)``
— one bit, one element, in place. Implementation via
``array.view(matching_uint_dtype)`` is a zero-copy reinterpret
view, so the float / int / bool original array reflects the post-
flip bit pattern directly. Works on fp32, fp16, int32, int64, bool;
other dtypes raise ``TypeError``.

### TMR voter design

The voter implements element-wise majority voting over three
replicas, matching the Horizon-1 Muller C-element hardware voter
(ADR-0003). Semantics:

- **3-way agreement** → output is the agreed value, no event.
- **2-way agreement** → output is the majority value, event logged
  as "corrected" (SEU caught + recovered).
- **0-way agreement** (all three differ) → output is undefined;
  event logged as "undefined" so callers can react (retry, flag
  uncertainty, escalate).

**Equality is bit-identical, not approximate.** Radiation-induced
flips almost always change the bit pattern of a float; the whole
*point* of TMR is to catch exactly that. Using ``np.allclose`` would
silently accept close-but-not-equal corruptions where two replicas
were hit by coincident flips producing near values. The voter
compares via ``array.view(uint_dtype)`` — the same uint reinterpret
trick used by the injector, so a flip injected by ``flip_bit`` is
guaranteed to be detected by ``majority_vote``.

Entry points:
- ``majority_vote(a, b, c)`` — stateless primitive returning a
  ``TMRResult`` with ``output``, ``n_corrected``, ``n_undefined``.
- ``TripleReplica.from_tensor(t)`` — deep-copies the source into
  three independent arrays so SEU injection into one cannot corrupt
  the others.
- ``TMRVoter(fn)`` — wraps a callable, runs it three times, votes
  on the output. Non-deterministic functions are correctly flagged
  as faults (by rad-hard contract, non-determinism IS a fault).

### Model robustness harness

``ModelRobustnessTest`` is the end-to-end artefact the DO-254
submission cites. Given a forward-pass callable + a map of
``{layer_name: weight_tensor}``, it runs ``n_trials`` injection
trials:

1. Snapshot the target tensor (``tensor.copy()``).
2. Inject one SEU via ``SEUInjector.inject``.
3. Run the forward pass.
4. Compare the output to the pre-injection baseline via
   max-absolute-difference (promoted to fp64 to avoid rounding
   noise masking tiny real drifts).
5. Classify: baseline-match (diff == 0), small drift
   (diff ≤ tolerance, default 1e-3), catastrophic (diff > tolerance
   or NaN/inf).
6. Restore the snapshot via ``np.copyto``.

The emitted ``ModelRobustnessReport`` carries:
- ``n_baseline_matches`` / ``n_small_drifts`` / ``n_catastrophic``
  — the three-bucket histogram.
- ``max_observed_diff`` — worst divergence seen.
- ``per_layer_events`` — full audit trail of every SEU injected,
  keyed by layer, with ``(flat_index, bit_within_word, pre, post)``
  for each flip.
- ``fit_rate_estimate_per_mbit`` — scaled by the industry-standard
  LEO SEU FIT reference (1000 failures / 10⁹ device-hours / Mbit
  of non-hardened fp32 DRAM at ~500 km). Indicative only — the
  real FIT calculation is a function of shielding, altitude, and
  solar activity, but the ratio ``n_catastrophic / n_trials`` lets
  customers compare model architectures on a common scale.

### What this is NOT

- **Not a substitute for radiation-beam testing.** The tape-out
  binder will still need actual heavy-ion and proton-beam testing
  at a DOE / Los Alamos / RADEF facility. The simulator is a
  *model* of SEU events; the real event rate depends on cross-
  section curves the tests produce.
- **Not a TMR silicon implementation.** It's a Python reference
  for the silicon voter to diff against. The RTL team can read
  ``majority_vote`` top-to-bottom (30 lines) and check their
  combinational logic matches.
- **Not a FIT-rate oracle.** The 1000 FIT/Mbit reference is an
  industry anchor; deployments cite their own measured rates from
  orbit once on-orbit telemetry is available.

## Consequences

### Positive (closing audit gaps)

- **H1 fault-injection evidence: mitigated.** The harness exists,
  runs today, produces deterministic reports. A defense prime
  asking for "SEU test evidence" receives a reproducible campaign
  artefact (57 tests pass in under a second; a real campaign
  against MinimalLlama runs a 200-trial report in < 2 s).
- **H1 DO-254 DAL-B software artefact: mitigated.** The
  combination of ``SEUEvent`` audit trail + ``ModelRobustnessReport``
  + bit-identical determinism under seed provides the DAL-B
  traceable-evidence requirement for the *software* portion of the
  life-cycle submission.
- **H1 model robustness quantification: unblocked.** Any model
  (MinimalLlama, a customer CNN, a policy network) can be
  ``ModelRobustnessTest``-wrapped and reported on, today, without
  silicon.

### Positive (SDK value)

- SDK users running inference in harsh environments gain first-
  class fault-injection testing.
- ``_kernels`` emulation (ADR-0013) can now run SEU-injected
  regression tests for numerical-stability bugs.
- Prometheus chiplet simulator (ADR-0027) can overlay TMR on
  replicated compute for redundancy-in-software demos.
- The "rad-hard claim" in the competitive matrix stops being a
  hardware-only claim — there is now a software deliverable
  sovereign buyers can install and invoke.

### Negative / Risks

- **FIT-rate is indicative, not authoritative.** The 1000 FIT/Mbit
  LEO reference is an industry anchor, not a measurement on
  Zhilicon silicon. We document this explicitly in the report
  docstring and the ADR; we do not claim it as a certified figure.
- **The harness costs O(n_trials × forward_pass_latency).** A
  10 000-trial campaign on a MinimalLlama forward pass takes
  minutes. Acceptable for pre-silicon evidence; silicon will need
  hardware-accelerated fault injection for 10⁸+ trial campaigns.
- **Bit-flips on integer dtypes can produce values outside a
  sensible range** (e.g. flipping the sign bit of an index). This
  is the correct simulation behaviour — silicon SEUs do produce
  such values — but callers should be aware the downstream code
  must handle nonsense inputs.

### Neutral

- **Only numpy dtypes supported today.** Extension to torch
  tensors would be a wrapper around ``tensor.detach().numpy()``;
  deferred until a user asks.
- **No chiplet-parallel injection yet.** The harness injects into
  one tensor at a time. Distributed SEU campaigns across 8
  chiplets (ADR-0027) would layer the harness on top of
  ``ChipletFabric``; deferred.

## Alternatives considered

### Option A: Delegate to a general fault-injection library (e.g. PyTorchFI)

Rejected. Existing libraries are either torch-only (PyTorchFI,
TensorFI) or simulator-coupled (gem5). The Horizon-1 emulation
backend is numpy-native; a torch dep would be pure weight. More
importantly, the DO-254 evidence path wants a reference the
Horizon-1 RTL team can diff against — ``flip_bit`` in 30 lines of
pure numpy is pedagogically transparent in a way a torch library
wrapping CUDA kernels is not.

### Option B: Do the fault injection in the ``_kernels`` emulation backend itself

Rejected. The emulation backend's role is functional correctness;
intertwining it with fault injection would muddy the contract.
Keeping ``rad_hard`` as a separate package means emulation stays
deterministic by default, and rad-hard tests are opt-in.

### Option C: Ship only TMR, defer SEU injection

Rejected. TMR without a way to inject SEUs is not testable pre-
silicon. The pair ships together because they're the minimum
viable evidence pair for the DO-254 artefact.

### Option D: Compare voter outputs via ``np.allclose``

Rejected outright — masks real corruptions. A flip in the mantissa
LSB of an fp32 produces a value ``np.allclose`` would accept. The
whole point of TMR is to catch bit-pattern changes. The test
``test_vote_uses_bit_identity_not_allclose`` pins this contract.

## References

- [ADR-0003: Horizon-1 TMR + Muller C-element voter](ADR-0003-horizon-1-tmr-muller-voter.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0027: Chiplet fabric simulator + Prometheus mitigation](ADR-0027-chiplet-simulator-and-prometheus-mitigation.md)
- [`src/sdk/python/zhilicon/rad_hard/`](../../src/sdk/python/zhilicon/rad_hard/)
- [`src/tests/sdk/python/test_rad_hard.py`](../../src/tests/sdk/python/test_rad_hard.py)
- [`docs/strategy/EXECUTION_TO_A_PLUS_ROADMAP.md`](../strategy/EXECUTION_TO_A_PLUS_ROADMAP.md) — H1 gate updated with mitigation evidence
- Horizon-1 DEEP_TECHNICAL_SPEC §4.1 (heavy-ion LET envelope)
- DO-254 DAL-B software life-cycle artefact list
- Portfolio audit 2026-04-18 (this session): Horizon-1 §fault-injection and §DO-254 gap line items
- Batch-42 session history.
