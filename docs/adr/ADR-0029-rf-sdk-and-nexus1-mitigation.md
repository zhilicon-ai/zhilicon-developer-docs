---
adr-id: ADR-0029
title: 'RF physics SDK (link budget + MIMO channel + beamforming + OFDM) — Nexus-1 gap mitigation'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/rf-coe", "@zhilicon/nexus-1-leads"]
---

# ADR-0029: RF physics SDK (link budget + MIMO channel + beamforming + OFDM) — Nexus-1 gap mitigation

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, RF COE, Nexus-1 silicon leads

## Context

The portfolio production-readiness audit (2026-04-18) flagged three
Nexus-1 gaps where software-side tooling could close carrier-
credibility concerns ahead of silicon bring-up (Q4 2026 Rev A,
hybrid-bond qual on critical path):

1. **No pre-silicon RF link-budget tooling.** Sovereign carriers
   evaluating Nexus-1 for sub-THz 6G deployments ask "what SNR do
   I get at 140 GHz across 100 m under 5 mm/hr rain?" — and prior
   to this batch the answer was "run a MATLAB script a vendor
   emailed us". For an OEM selling a chip whose RF performance is
   the differentiator, that was a credibility gap.
2. **AI-assisted beamforming claim not demonstrated in software.**
   Nexus-1's on-die AI block does learned beam selection in one
   forward pass instead of the classic O(N) exhaustive codebook
   sweep. The claim was in the datasheet but had no customer-
   runnable reference. An RFCE at a tier-1 carrier asking "what
   does 'learned beamforming' actually buy me over my current
   array?" had no pre-silicon answer.
3. **OFDM waveform reference for UVM missing.** The Nexus-1 RF /
   AI-RAN UVM teams need byte-reproducible I/Q sample streams at
   each numerology to drive their testbench. Without an SDK-
   sourced generator the reference could drift between team
   scripts.

Parallel to these audit items, the SDK had an RF-adjacent need:
- ``zhilicon.telecom`` (existing) provides a high-level API
  surface (``RadioUnit``, ``BeamformingEngine``, ``LinkQuality``)
  but with hard-coded defaults. It needs a physics-backed
  numerical substrate to compute its values from, not just return
  placeholder constants.
- The competitive matrix (``COMPETITIVE_POSITION_MATRIX.md``)
  claims sub-THz advantages over Qualcomm and Samsung in the 6G
  plane. Those claims need a software artefact a customer can
  actually invoke.

Closing the N1 gap and the SDK need with one package is the same
dual-use pattern as ADR-0025 / ADR-0026 / ADR-0027 / ADR-0028
(Sentinel-1, Discovery-1, Prometheus, Horizon-1 respectively).

## Decision

Ship ``zhilicon.rf`` — a radio-frequency physics SDK with a
deliberate dual role:

- **SDK numerical substrate** for any carrier or deployment
  engineer doing pre-silicon link-budget / beamforming analysis.
- **Nexus-1 pre-silicon evidence package** — the validator
  produces the exact "learned beamforming buys you X dB over
  exhaustive search" comparison table the carrier RFCE asked for.

### Module layout

| Module | Role | Backing |
|---|---|---|
| ``zhilicon.rf.link_budget`` | Friis FSPL + kTB noise floor + NF cascade + Shannon capacity | Pure-Python ``math`` |
| ``zhilicon.rf.channel`` | MIMO channel matrix generators (AWGN / Rayleigh / Rician) + CDL cluster data structure (3GPP TR 38.901 §7.7) | Pure numpy |
| ``zhilicon.rf.beamforming`` | DFT codebook, SVD optimal precoder, exhaustive beam selection, pluggable ``BeamPredictor`` interface for AI-assisted | Pure numpy |
| ``zhilicon.rf.ofdm`` | OFDM modulate / demodulate with cyclic prefix, 5G + candidate-6G numerology table, EVM measurement | Pure numpy + ``numpy.fft`` |
| ``zhilicon.rf.validator`` | End-to-end link simulator tying the above together | Composes all four |

### Physics fidelity contract

The top priority for this package is that the numbers it emits are
*correct*, not just that the code runs. Three classes of reference:

1. **Pinned textbook identities.** Friis FSPL at 1 GHz / 1 km =
   92.45 dB — the canonical reference in every RF textbook.
   Thermal noise at T₀ = 290 K = −174 dBm/Hz. The test suite pins
   these to within ≤ 0.05 dB.
2. **Closed-form scalings.** Doubling distance adds 6.02 dB
   (20 log₁₀ 2). Doubling frequency adds 6.02 dB. Doubling
   temperature adds 3.01 dB. All three pinned.
3. **Information-theoretic bounds.** SVD precoder achieves the
   top singular value σ₁ as effective channel gain; DFT codebook
   is sub-optimal; exhaustive codebook search cannot exceed SVD
   on the same channel realisation. The validator test suite
   pins ``SVD.gain_db ≥ DFT.gain_db`` to within 0.2 dB.

### AI-assisted beamforming — pluggable, not prescriptive

The SDK does **not** ship a pre-trained beam-prediction model.
Instead it ships a :class:`BeamPredictor` interface:

```python
predictor = BeamPredictor(
    predict_fn=lambda observation: my_torch_model(observation).argmax(),
    n_beams=32,
    label="my_learned_predictor",
)
result = simulate_mimo_link(
    link_budget=..., beamforming_mode=BeamformingMode.AI_ASSISTED,
    predictor=predictor, ...
)
```

Customers bring their own model (trained offline on their
deployment scenario, quantised for on-chip execution). The SDK's
job is to benchmark it against the DFT and SVD baselines on the
same channel realisations. The :class:`EndToEndLinkResult`
structure produces the `SNR X → Y dB (+Z dB), C ≈ Q Gbps`
comparison line.

This design is deliberate: prescribing a model architecture would
be premature given the field is still converging on beam-selection
DNN designs (CNN-over-angular-domain, graph neural networks,
transformer-based), and would tie the SDK to a torch / JAX
dependency which other SDK modules avoid.

### Noise-floor semantics

A subtle design point worth recording: ``simulate_mimo_link`` uses
**absolute noise power** (set by the link-budget's kTB + NF), not
signal-relative SNR, when adding AWGN. This is the physics-correct
model: TX-side beamforming concentrates signal power on a matched
direction, while the receiver's thermal noise floor stays fixed.
Using ``add_awgn(signal, snr_db=pre_snr)`` would scale noise to
signal, which would mask any beam gain.

### MRC on receive side

The effective channel after beamforming is ``||H · w||₂``
(maximum-ratio combining magnitude), not ``(H · w).mean()``. The
mean of complex values with random phases cancels coherently to
near zero — which would defeat the whole point of spatial
combining. This is explicit in the validator comments; the tests
pin the gain against published Marchenko-Pastur theory (top
singular value of 4×8 Gaussian matrix ≈ 13.7 dB, captured by the
``SVD.gain_db ≥ DFT.gain_db`` invariant).

### What this is NOT

- **Not a full 3GPP-compliant PHY.** The SDK includes dataclasses
  for CDL profiles but a full CDL-aware channel generator with
  angular + delay spread is out of scope (future batch).
- **Not a pre-silicon replacement for OTA measurement.** The
  validator produces simulation numbers; real deployments need
  anechoic-chamber and field-trial data. The harness does not
  claim to replace either.
- **Not a channel sounding or beam-tracking framework.** Those
  are session-level concerns orthogonal to the physics primitives
  shipped here.

## Consequences

### Positive (closing audit gaps)

- **N1 link-budget gap: mitigated.** A sovereign carrier running
  ``LinkBudget(carrier_hz=140e9, ...)`` gets a
  pinned-to-Friis-physics SNR and capacity estimate in one line.
  The pre-sale engineering-check cycle collapses from "email us a
  MATLAB script" to "pip install zhilicon && python demo.py".
- **N1 AI-beamforming gap: mitigated.** The ``BeamPredictor``
  interface + end-to-end validator produce an auditable "your
  learned model vs. SVD upper bound vs. DFT exhaustive baseline"
  comparison. A tier-1 RFCE asking "what does learned beamforming
  buy me?" gets a concrete number.
- **N1 OFDM reference gap: mitigated.** ``ofdm_modulate`` is
  byte-reproducible given the same ``(symbols, n_fft, n_cp)``
  triple. The UVM team commits reference I/Q traces to git and
  regenerates them in CI. 66 tests pin the round-trip and the
  numerology-table scaling laws.

### Positive (SDK value)

- SDK users running any RF workload — not just 6G — get
  first-class link-budget + channel + beamforming primitives.
- ``zhilicon.telecom`` (existing API surface) gains a physics-
  backed numerical substrate to compute its ``LinkQuality``
  fields from. A follow-up batch can re-back ``telecom`` on top
  of ``rf`` without changing the public API.
- The competitive matrix claim "sub-THz 6G + on-die AI-assisted
  beamforming" gains a software-today counterpart. Customers can
  invoke it before committing to silicon.

### Negative / Risks

- **Simulation numbers can be taken out of context.** A 5.3 dB
  average DFT beamforming gain on Rayleigh is correct physics
  but does not translate directly to field-trial performance
  (which also depends on antenna calibration, RF front-end
  linearity, frequency-selective fading not captured in the
  simplified model). The ADR and the docstrings label the
  outputs as "indicative only"; a tape-out-binder citation
  requires the OTA-measurement track.
- **numpy-only, no torch / JAX.** Consistent with every prior
  SDK batch — keeps dependencies light. Users who want autograd
  for learned beam-selection training supply their own stack and
  plug in via :class:`BeamPredictor`.
- **CDL full implementation deferred.** The data structure ships;
  the generator only supports AWGN / Rayleigh / Rician. A full
  CDL-A / B / C / D / E implementation (with angular + delay
  spread) is ~500 lines and a separate batch.

### Neutral

- **``telecom`` API stays unchanged.** This batch does not re-
  back the existing ``zhilicon.telecom`` module on the new ``rf``
  primitives — that's a follow-up. The two packages coexist:
  ``rf`` = numerical physics, ``telecom`` = high-level O-RAN API.
- **ML-KEM / ML-DSA from ADR-0025 were deferred too.** Consistent
  pattern: ship the minimum reference that closes the audit gap,
  defer completeness to a dedicated follow-up batch.

## Alternatives considered

### Option A: Wrap a third-party RF simulator (Matlab 5G Toolbox, Sionna, NVIDIA Aerial)

Rejected. Each has a large dep footprint (GPU-backed in Sionna's
case) and a strong licensing tie-in (Matlab). The SDK philosophy
is "numpy-only, pedagogically transparent reference". Customers
who want Sionna for training can use Sionna *and* ``zhilicon.rf``
side-by-side.

### Option B: Ship a pre-trained AI beam predictor

Rejected outright. Any pre-trained model would be scenario-
specific, and the field is still converging on architectures. The
pluggable :class:`BeamPredictor` is the right abstraction: ship
the interface, let customers bring their own model.

### Option C: Extend ``zhilicon.telecom`` directly rather than adding ``rf``

Rejected. ``telecom`` is a hardware-abstracted API surface;
``rf`` is a pure-numerical physics layer. Mixing concerns would
make both modules harder to reason about. The right layering is
``telecom → rf`` (future refactor), not ``telecom = rf``.

### Option D: Skip OFDM, only ship link-budget + beamforming

Rejected. The UVM-reference gap called out in the audit requires
byte-reproducible I/Q samples; OFDM modulate/demodulate is the
minimum surface that produces those. The round-trip test plus
EVM utility are direct artefacts the UVM team consumes.

## References

- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0025: Crypto kernel library + TVLA harness (Sentinel-1)](ADR-0025-crypto-kernel-library-and-tvla-harness.md)
- [ADR-0026: Healthcare SDK (Discovery-1)](ADR-0026-healthcare-sdk-and-discovery1-mitigation.md)
- [ADR-0027: Chiplet fabric (Prometheus)](ADR-0027-chiplet-simulator-and-prometheus-mitigation.md)
- [ADR-0028: Rad-hard SDK (Horizon-1)](ADR-0028-rad-hard-sdk-and-horizon1-mitigation.md)
- [`src/sdk/python/zhilicon/rf/`](../../src/sdk/python/zhilicon/rf/)
- [`src/sdk/python/zhilicon/telecom/`](../../src/sdk/python/zhilicon/telecom/) — existing O-RAN API surface; candidate for future re-backing atop ``rf``
- [`src/tests/sdk/python/test_rf.py`](../../src/tests/sdk/python/test_rf.py)
- [`docs/strategy/EXECUTION_TO_A_PLUS_ROADMAP.md`](../strategy/EXECUTION_TO_A_PLUS_ROADMAP.md) — Nexus-1 gate updated with SDK-side mitigation table
- Friis, H.T. (1946), "A Note on a Simple Transmission Formula", Proc. IRE 34 (5): 254–256.
- 3GPP TS 38.211 §4.2 (5G numerologies); TR 38.901 §7.7 (CDL profiles).
- Rappaport et al. (2019), "Wireless Communications and Applications above 100 GHz", IEEE Access 7.
- Portfolio audit 2026-04-18: Nexus-1 §link-budget, §AI-beamforming, §UVM-reference line items.
- Batch-43 session history.
