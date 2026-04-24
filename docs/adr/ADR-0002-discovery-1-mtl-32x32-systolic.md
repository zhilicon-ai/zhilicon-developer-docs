---
adr-id: ADR-0002
title: Discovery-1 MTL frozen at 32×32 systolic
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/discovery-1-leads", "@zhilicon/chief-architect"]
---

# ADR-0002: Discovery-1 MTL frozen at 32×32 systolic

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Discovery-1 leads + Chief architect

## Context

The V3.0 Discovery-1 upgrade spec claimed 1 840 TOPS INT8. The Batch-4
portfolio audit ([E-D1-002](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md))
showed that the arithmetic assumes a 32×32 systolic per MTL (1 024 MACs per
MTL × 640 MTLs × 1.8 GHz × 78 % utilisation = 1 840 TOPS), while some
companion documents still described a 16×16 systolic (256 MACs per MTL →
590 TOPS peak). A 3× discrepancy on the headline number is not a
recoverable marketing problem after tape-out.

Area budget: 32×32 is 4× the MAC count of 16×16, which translates to
roughly 2.3× the per-MTL area. At 640 MTLs per chip, the die-area
re-check is non-trivial.

## Decision

The Medical Tensor Lane v2 systolic is **32×32**. Every downstream artefact
(block diagram, floorplan, area estimate, timing constraints, MTL v2
register file map, compiler kernel library) must carry this assumption.

The Chief Architect signs off. The action log in E-D1-002 tracks the
propagation: block diagram → area re-estimate (2.3× per MTL) → floorplan
fit at 640 MTLs / 395 mm² → compiler kernel library update. Until every
propagation step lands, no customer-facing TOPS figure may be quoted
externally.

## Consequences

### Positive

- Headline 1 840 TOPS INT8 is defensible against competitive benchmarks
  (B200 tensor core geometry for reference — 16×16×16, i.e. 16 384 MACs
  per core per cycle).
- Eliminates the 3× ambiguity; every downstream doc, tape-out constraint,
  and compiler kernel uses the same systolic geometry.

### Negative / Risks

- Physical Design must revalidate floorplan area against the 2.3× per-MTL
  increase. Worst case: descope to 576 MTLs and re-spec to ≈ 1 660 TOPS.
  RISK-003 in `UPGRADE_RISK_REGISTER.md` covers this.
- Compiler kernel library grows — a 32×32 systolic operator shape is
  coarser, so small-batch workloads may underutilise. Mitigated by
  autotune + multi-tile scheduling.

### Neutral

- No ecosystem compatibility impact — the systolic is internal to the
  MTL; the SDK continues to expose the same high-level primitives.

## Alternatives considered

### Option A: Keep 16×16 and publish 590 TOPS

Rejected. A 3× headline cut a year into a program with customer MoUs
attached is a non-starter.

### Option B: Heterogeneous MTL sizes (some 16×16, some 32×32)

Rejected. Breaks the compiler's uniform-tile assumption and explodes
autotune search space.

## References

- [E-D1-002](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`programs/discovery-1/upgrade/DISCOVERY-1_WORLD_CLASS_UPGRADE.md`](../../programs/discovery-1/upgrade/DISCOVERY-1_WORLD_CLASS_UPGRADE.md)
- [`src/rtl/packages/d1_pkg.sv`](../../src/rtl/packages/d1_pkg.sv)
- [ADR-0001](ADR-0001-errata-first-spec-hygiene.md)
