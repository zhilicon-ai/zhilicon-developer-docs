---
adr-id: ADR-0025
title: 'Cryptographic kernel library + TVLA side-channel harness — Sentinel-1 gap mitigation'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/security-coe", "@zhilicon/sentinel-1-leads"]
---

# ADR-0025: Cryptographic kernel library + TVLA side-channel harness — Sentinel-1 gap mitigation

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Security COE, Sentinel-1 leads

## Context

The portfolio production-readiness audit (2026-04-18) called out
three Sentinel-1 audit gaps that the SDK can directly help close,
even before the silicon tapes out:

1. **Side-channel leakage test infrastructure missing entirely.**
   The Sentinel-1 target is ``|t| < 4.5 @ 100 M traces`` under Test
   Vector Leakage Assessment (TVLA) — the side-channel community's
   standard bar for "no first-order leakage." No harness existed to
   measure it. That absence blocked any credible side-channel test
   plan being written into the tape-out binder.
2. **UVM testbench at 0%** with "no test vectors generated." The
   Sentinel-1 silicon UVM teams need NIST CAVP-format reference
   vectors (the ``.rsp`` files NIST publishes) to drive their
   testbench. Producing vectors on-demand, reproducibly, closes the
   dependency on hand-downloading NIST dumps.
3. **HMAC-DRBG reference for the two-stage TRNG** (ADR-0006). The
   silicon team will implement the DRBG state machine in RTL; they
   need a bit-exact Python reference to cross-validate against,
   otherwise a subtle bug in the spec→RTL mapping leaves them
   stranded at bring-up.

Separately, the SDK itself was accumulating a genuine need for
production-grade crypto primitives — the sovereign attestation
receipts shipped in ADR-0024 were signed with HMAC-SHA-256 via the
stdlib; the same pattern will need AEAD for persistent audit logs,
ECDSA for cross-zone attestation, and a FIPS-appliance deterministic
RNG for reproducible signing. Meeting the SDK need with the same
batch that closes the Sentinel-1 gaps is a dual-use win.

## Decision

Ship ``zhilicon.crypto`` — a small cryptographic kernel library
with a deliberate dual role:

- **SDK production crypto** for sovereign-serving use cases.
- **Sentinel-1 pre-silicon test tooling** — the TVLA harness and
  the NIST CAVP reference-vector generator live in the same package
  so the Sentinel-1 team pulls them via one ``pip install``.

### Module layout

| Module | Role | Backing |
|---|---|---|
| ``zhilicon.crypto.hash`` | SHA-256, SHA3-256, SHAKE128, SHAKE256 | Stdlib ``hashlib`` |
| ``zhilicon.crypto.hmac_drbg`` | NIST SP 800-90A HMAC-DRBG (SHA-256), the ADR-0006 two-stage TRNG back-end | Pure Python — 120 lines — so the Sentinel-1 RTL team can diff against it line by line |
| ``zhilicon.crypto.aead`` | AES-256-GCM | Lazy-imports ``cryptography`` (optional dep). Keeps the SDK installable without it. |
| ``zhilicon.crypto.test_vectors`` | NIST CAVP ``.rsp`` loader + generator | Pure Python + ``hashlib`` |
| ``zhilicon.crypto.tvla`` | TVLA Welch's t-test + constant-time timing probe | Pure numpy |

### TVLA harness design

The harness implements the Goodwill et al. (NIAT 2011) methodology:
collect two populations of per-operation measurements (one with a
fixed key, one with random keys), apply Welch's t-test, compare
``|t|`` against a threshold — 4.5 is the community-standard
confidence bound (``p < 1e-5``).

Two entry points:

- ``welch_t_test(traces_a, traces_b, *, threshold=4.5)`` — the
  statistical primitive. Callers supply whatever "trace value" is
  meaningful for their setup: per-call wall-clock latency,
  point-in-time oscilloscope amplitude, derived features from
  post-alignment traces. Returns a :class:`TVLAResult` with the
  t-statistic, threshold, and pass/fail verdict.
- ``constant_time_check(fn, *, fixed_key, random_keys, ...)`` —
  convenience wrapper that times ``fn(key, message)`` with the
  ``perf_counter_ns`` clock and runs TVLA on the latency
  populations. Good for surfacing gross non-constant-time behaviour
  during Python-level prototyping. **Not** a substitute for the
  post-silicon EM/power bench; that's what we will use when we have
  silicon.

Same primitive serves both phases. Simulation-phase engineers call
``constant_time_check`` on a Python implementation of a crypto
primitive to catch obvious leaks. Post-silicon lab engineers feed
oscilloscope traces directly into ``welch_t_test``. Shared
statistical core, shared pass/fail reporting format.

### NIST CAVP reference-vector generation

``test_vectors.generate_sha256_vectors(lengths=...)`` produces a
list of :class:`NistCAVPVector` objects matching the format NIST
publishes for algorithm validation: ``Len``, ``Msg``, ``MD`` fields
in a ``.rsp`` file. The S1 UVM testbench already consumes ``.rsp``;
the generator + :func:`render_cavp_vectors` emit that format byte-
exactly. Messages are deterministic (``b"\\x42" * n``) so the
generated file is byte-reproducible across runs and machines —
which means S1's UVM testbench can commit the reference ``.rsp``
artefact to git and regenerate it in CI to guarantee drift never
creeps in between the Python reference and the ``.rsp`` file the
testbench reads.

### HMAC-DRBG as reference

The implementation follows SP 800-90A §10.1.2 exactly:

- ``Instantiate`` — derives (K, V) from entropy ‖ nonce ‖
  personalization_string via the ``Update`` function.
- ``Update`` — the 2-pass key/V refresh.
- ``Generate`` — output ``n`` bytes in 32-byte HMAC chunks;
  post-generate ``Update`` call for forward secrecy.
- ``Reseed`` — re-instantiates (K, V) from new entropy + additional
  input.
- Reseed-counter enforcement at the SP 800-90A-specified ``2^48``
  ceiling; single-request ceiling at ``2^19`` bytes.

The Sentinel-1 TRNG architecture (ADR-0006: RO → Von Neumann →
HMAC-DRBG) uses this DRBG as its conditioning stage. Matching
Python + RTL will be verified at bring-up by running the same
(entropy, nonce, personalization) triple through both and diffing
the output byte stream.

## Consequences

### Positive (closing audit gaps)

- **S1 side-channel test infrastructure: mitigated.** The harness
  exists, runs today, and detects synthetic leakage in our own
  test suite (see ``test_leaky_fn_fails_tvla``). S1 team receives
  a ready-to-use package; they supply the cryptographic
  implementation under test, they inherit the statistical core.
- **S1 UVM test-vector gap: mitigated.** Vectors regenerate from
  ``generate_sha256_vectors()``; round-trip via
  ``render_cavp_vectors()`` / ``load_cavp_vectors()``. CI can run
  this path to block any drift.
- **S1 HMAC-DRBG RTL cross-validation: unblocked.** The reference
  implementation is 120 lines, deterministic from (entropy, nonce,
  PS), and matches NIST CAVP vector format.

### Positive (SDK value)

- Attestation receipts (ADR-0024) gain a proper crypto path — AEAD
  for audit-log sealing, HMAC-DRBG for receipt-ID generation.
- Sovereign buyers asking "what crypto comes in the box" see a
  specific module list rather than a hand-wave at "standard
  libraries".
- Zhilicon's competitive matrix claim "FIPS 140-3 L4 + silicon ZKP
  + silicon PQC" gains a concrete software-today counterpart. We
  don't claim this library is FIPS-validated — it isn't. But the
  primitive set it exposes is the set S1 silicon will certify.

### Negative / Risks

- **This library is not FIPS-validated.** It's a *reference* for
  RTL cross-validation and a *convenience* for SDK users who don't
  need FIPS today. Deployments that require FIPS-validated software
  crypto use ``cryptography``'s FIPS modes or a validated HSM —
  same posture as most Python projects.
- **AEAD requires the ``cryptography`` dep.** A deliberate dep on
  OpenSSL via the binding. Air-gapped sovereign clusters that can't
  install it fall back to the hashing + HMAC-DRBG surface, which
  covers attestation signing. AEAD is orthogonal; deployments that
  need it install the dep.
- **The constant-time check is a best-effort wall-clock proxy.**
  System noise can produce false positives on a busy CPU. The test
  suite uses a tolerant ``threshold=8.0`` for the positive-case
  probe, and skips (doesn't fail) if the background system is too
  noisy. This is explicit — we don't claim the harness is a
  substitute for the post-silicon lab bench.

### Neutral

- **SHA-256 and SHA-3 are stdlib-backed.** No value in rolling our
  own. The wrappers exist purely to return ``bytes`` (not hex) and
  to be discoverable from ``zhilicon.crypto`` alongside HMAC-DRBG.
- **ML-KEM / ML-DSA not included yet.** Sentinel-1 silicon includes
  both. A future batch adds pure-Python references for cross-
  validation (both are ~500 lines to implement naively). Out of
  scope here.

## Alternatives considered

### Option A: Delegate crypto to the ``cryptography`` package and skip our own reference

Rejected for the S1 gap-mitigation branch — we specifically need
bit-exact references the Sentinel-1 RTL team can diff their
implementation against. ``cryptography`` wraps OpenSSL, which is a
correctness reference but not a pedagogical reference for an RTL
state machine. Our ``HMACDRBG`` class is 120 lines; the silicon
team can walk it top-to-bottom.

### Option B: Embed the full NIST CAVP test vector set

Rejected. The NIST CAVP corpus is hundreds of MB. Git-committing it
would bloat the repo; vendoring it via a submodule is friction
without benefit when we can regenerate vectors deterministically on
demand.

### Option C: Use ``scipy.stats.ttest_ind`` for the Welch's test

Rejected. Adding a scipy dep for a 10-line formula is the wrong
trade. Pure numpy works, keeps the dependency surface small, and
the computation is transparent enough for a security auditor to
verify.

### Option D: Separate package for S1 silicon teams

Attractive at first glance — ``zhilicon-sentinel-crypto`` as a
distinct PyPI name. Rejected because the SDK's own attestation
path needs the same hash / HMAC / DRBG primitives; duplicating
maintenance across two packages is pointless when the single
package serves both audiences.

## References

- [ADR-0006: Sentinel-1 two-stage TRNG](ADR-0006-sentinel-1-trng-two-stage.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
- [ADR-0024: Sovereign-zone enforcement + attestation receipts](ADR-0024-sovereign-zone-enforcement.md)
- [`src/sdk/python/zhilicon/crypto/`](../../src/sdk/python/zhilicon/crypto/)
- [`src/tests/sdk/python/test_crypto.py`](../../src/tests/sdk/python/test_crypto.py)
- [`src/tests/sdk/python/test_crypto_tvla.py`](../../src/tests/sdk/python/test_crypto_tvla.py)
- NIST SP 800-90A Rev. 1: <https://csrc.nist.gov/publications/detail/sp/800-90a/rev-1/final>
- Goodwill et al., "A testing methodology for side channel resistance validation", NIAT 2011
- Portfolio audit 2026-04-18 (this session): Sentinel-1 §side-channel and §UVM gap line items
- [`docs/strategy/EXECUTION_TO_A_PLUS_ROADMAP.md`](../strategy/EXECUTION_TO_A_PLUS_ROADMAP.md) — Sentinel-1 gates updated to reflect these mitigations
- Batch-39 session history.
