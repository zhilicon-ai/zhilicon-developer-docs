---
adr-id: ADR-0006
title: Sentinel-1 two-stage TRNG (RO → Von Neumann → HMAC-DRBG)
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/sentinel-1-leads", "@zhilicon/security-coe"]
---

# ADR-0006: Sentinel-1 two-stage TRNG (RO → Von Neumann → HMAC-DRBG)

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Sentinel-1 Security lead + Security COE

## Context

The pre-V4.0 Sentinel-1 architecture described the TRNG as "2 × RO, 1 Gb/s,
Von Neumann extractor" — implying that a Von Neumann debiaser alone
serves as the conditioning component
([E-S1-003](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)).

FIPS 140-3 + NIST SP 800-90A / SP 800-90B require:

- An **entropy source** satisfying SP 800-90B, with continuous health
  tests.
- A **conditioning component** — either (a) a vetted conditioning
  construction per SP 800-90B §3.1.5, or (b) an SP 800-90A approved
  DRBG seeded from raw entropy.

A Von Neumann debiaser removes first-order bias. It is *not* an approved
conditioning component. Shipping with only Von Neumann would fail CMVP
on the first review.

## Decision

Implement a **two-stage architecture**:

1. **Entropy source** — 2 independent 11-stage ring oscillators, XORed,
   sampled at 100 MHz. Minimum entropy ≥ 0.85 bits/sample per RO per
   SP 800-90B IID-track characterisation.
2. **Conditioning + DRBG** — Von Neumann debiaser (pre-stage bias
   removal) feeding **HMAC-DRBG** (SP 800-90A approved) as the
   authoritative conditioning and generation mechanism.

Continuous health tests per SP 800-90B §4.4.1 (repetition count) and
§4.4.2 (adaptive proportion). Startup test per §4.5 (1 024 samples
burn-in).

Coq correctness proof for the full RO-XOR → Von Neumann → HMAC-DRBG
chain is scheduled Weeks 14–18 of the V4.0 upgrade programme (see
[`CRYPTO_FORMAL_VERIFICATION_PLAN.md`](../../programs/sentinel-1/plans/CRYPTO_FORMAL_VERIFICATION_PLAN.md)).

## Consequences

### Positive

- FIPS 140-3 submission evidence is coherent and complete.
- Von Neumann remains in the chain as a sensible pre-stage; removing it
  would give up a cheap bias-reduction step for no reason.
- Offline verification of the DRBG is deterministic given the same seed.

### Negative / Risks

- Adds HMAC-DRBG area (modest — SHA-256 core is already shared with the
  hash farm).
- Raw-entropy throughput drops from 200 Mbps to ≈ 50 Mbps after Von
  Neumann, then the HMAC-DRBG re-expands to 1 Gbps — the seed rate is
  what matters for FIPS, and 50 Mbps of min-entropy is ample to reseed
  HMAC-DRBG at any practical cadence.

### Neutral

- Any consumer of the TRNG (Kyber keygen, ECDSA signing, nonce
  generation) calls the same API surface whether backed by raw entropy
  or HMAC-DRBG output. FIPS-compliance status does not leak into the
  application API.

## Alternatives considered

### Option A: Von Neumann alone

Rejected. Fails FIPS 140-3 CMVP.

### Option B: Vetted conditioning component (e.g. CBC-MAC per SP 800-90B §3.1.5)

Considered. Equivalent FIPS-compliance outcome; HMAC-DRBG chosen because
the SHA-256 core is already present for the hash farm, so marginal area
is near-zero.

### Option C: Raw-entropy direct (no conditioning)

Rejected. Raw RO output carries first-order bias; direct use would fail
SP 800-90B health tests on first characterisation.

## References

- [E-S1-003](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`programs/sentinel-1/upgrade/DEEP_TECHNICAL_SPEC.md` §4](../../programs/sentinel-1/upgrade/DEEP_TECHNICAL_SPEC.md)
- [`CRYPTO_FORMAL_VERIFICATION_PLAN.md`](../../programs/sentinel-1/plans/CRYPTO_FORMAL_VERIFICATION_PLAN.md)
- NIST SP 800-90A, SP 800-90B
