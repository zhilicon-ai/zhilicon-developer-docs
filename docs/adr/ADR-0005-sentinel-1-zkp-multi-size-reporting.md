---
adr-id: ADR-0005
title: Sentinel-1 multi-constraint-size ZKP reporting policy
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/sentinel-1-leads", "@zhilicon/benchmark-review-board", "@zhilicon/chief-architect"]
---

# ADR-0005: Sentinel-1 multi-constraint-size ZKP reporting policy

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Sentinel-1 Security lead + Benchmark Review Board + Chief architect

## Context

The V4.0 upgrade claimed "8 400 Groth16 proofs/s" for Sentinel-1 without
specifying the constraint count. ZKP proof time scales as
O(n log n) for the NTT and O(n) for the MSM, where n is the number of
R1CS constraints. The industry standard varies by vendor:

- Ingonyama publishes at 2^16 (64 K) constraints.
- Ulvetanna publishes at 2^16.
- Supranational publishes at 2^22 (4 M).

At 2^16 Sentinel-1 hits ≈ 16 600 / s (both engines); at 2^20 it's
≈ 142 / s; at 2^22 it's ≈ 36 / s. A single-number "8 400 / s" claim is
meaningless without the size. A customer running a zkEVM rollup at 2^22
constraints expecting 8 400 / s would see 36 / s — a 230× shortfall — and
rightly conclude the vendor was deceptive.

## Decision

Establish a policy, enforced by the Benchmark Review Board:

1. **Every ZKP throughput figure quoted in any Zhilicon material** —
   internal, customer-facing, investor-facing, or marketing — carries
   an explicit constraint count.
2. The canonical reporting table enumerates five points: **2^14, 2^16,
   2^18, 2^20, 2^22**. Both engines enabled, BLS12-381, Groth16 baseline.
3. **Competitor comparisons** use the competitor's published constraint
   size. If no matched-size data exists, the comparison is not published;
   BRB refuses to sign ratios that mix sizes.
4. **No headline "X proofs/s" single number** is permitted externally.

The policy is documented in
[`MEASURED_BENCHMARKS.md`](../../programs/portfolio-upgrade/MEASURED_BENCHMARKS.md)
§5.3 and implemented in
[`s1_pkg.sv`](../../src/rtl/packages/s1_pkg.sv) as the five explicit
per-size target parameters.

## Consequences

### Positive

- Vendor honesty — customers see the curve, not a cherry-picked point.
- Regulator / auditor credibility — ZKP claims survive independent
  inspection.
- Product selection accuracy — customers pick the engine configuration
  that matches their real constraint size.

### Negative / Risks

- Headline becomes a table instead of a number. Harder to fit in a
  single tweet; easier to fit in a regulatory filing.
- Competitors who publish single-number claims look superficially
  faster. BRB + GTM messaging must consistently reframe the comparison.

### Neutral

- The rule applies to any succinct-proof system in the Zhilicon
  portfolio (Groth16, PLONK, Halo2, STARK, Nova). Future ZKP engine
  additions inherit the same convention.

## Alternatives considered

### Option A: Publish at a single vendor-chosen size

Rejected. Reproduces the behaviour we are criticising in competitors.

### Option B: Publish only at the size used by the largest customer

Rejected. Couples the external benchmark to a specific deployment,
creates incentive to favour one customer's workload over another's.

## References

- [E-S1-001](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`MEASURED_BENCHMARKS.md` §5.3](../../programs/portfolio-upgrade/MEASURED_BENCHMARKS.md)
- [`src/rtl/packages/s1_pkg.sv`](../../src/rtl/packages/s1_pkg.sv)
