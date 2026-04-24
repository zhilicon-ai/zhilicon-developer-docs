---
adr-id: ADR-0004
title: Nexus-1 Rev A / Rev B two-revision RF-chain delivery
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/nexus-1-leads", "@zhilicon/chief-architect", "@zhilicon/program-management-office"]
---

# ADR-0004: Nexus-1 Rev A / Rev B two-revision RF-chain delivery

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Nexus-1 leads + Chief architect + PMO

## Context

[E-N1-002](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
established that the honest Friis-cascade noise figure of the Skylink RF
chain is **5.98 dB** at V3.0 — not the 4.8 dB quoted in the upgrade
document. With other V3.0 gains folded in (+6 dB PA Psat, +6 dB array gain,
+2.3 dB learned beamforming) the LEO D-band worst-case margin is only
**+0.3 dB** at 64-QAM, below the ≥ 3 dB portfolio design floor. Shipping
a +0.3 dB margin is not acceptable.

A single-revision fix (push NF to 4.8 dB in the current V3.0) would
require a new GaN SPDT (0.8 dB IL) and a revised LNA1 (3.6 dB NF) — both
pending foundry characterisation. Delaying the Q4 2026 tape-out a year
to land these in a single revision would slip the entire portfolio
coordinated launch.

## Decision

Ship Nexus-1 Skylink in **two revisions**:

- **Rev A** (Q4 2026 tape-out, current V3.0 components): system NF
  5.98 dB; LEO support limited to QPSK and 16-QAM; terrestrial backhaul
  covers all modulations with ≥ 3 dB margin. Datasheet quotes
  5.98 dB typ — the 4.8 dB figure is withdrawn.
- **Rev B** (Q2 2027 tape-out target): new eGaN-on-Si SPDT (0.8 dB IL)
  + revised LNA1 (3.6 dB NF). System NF ≤ 4.8 dB target with ≥ +3 dB
  margin at 64-QAM. Gated by Macom + Qorvo shuttle-wafer breakouts
  Q3–Q4 2026.

Fallback: if neither foundry hits the shuttle gate, a two-stage LNA
topology improves NF by 0.6–0.8 dB at the cost of +1.2 mW per element
(0.3 W over 256 elements, inside the 18 W envelope).

## Consequences

### Positive

- Q3 2027 coordinated portfolio launch stays on track.
- Customers with terrestrial-backhaul-only workloads get the full
  feature set in Rev A.
- LEO customers get QPSK / 16-QAM in Rev A immediately, 64-QAM in Rev B.

### Negative / Risks

- Two tape-outs, two test-vehicle runs, two package qualifications.
  NRE delta ≈ $3.2 M.
- A customer expecting 64-QAM LEO from day one must either wait for
  Rev B or accept QPSK / 16-QAM on Rev A.

### Neutral

- Rev A and Rev B share the digital chiplet and AI4PHY. The difference
  is confined to the RF front-end.

## Alternatives considered

### Option A: Ship Rev A at +0.3 dB margin, market it as 4.8 dB NF

Rejected. A +0.3 dB margin is not production-credible, and the 4.8 dB
claim is unsupported by the Friis cascade.

### Option B: Single revision — delay V3.0 tape-out to Q2 2027

Rejected. Slips the portfolio coordinated launch by 6–9 months with no
customer benefit over the two-revision plan.

### Option C: Drop LEO support, ship terrestrial-only Rev A

Rejected. LEO is the single most differentiated segment for the Skylink
product; ceding it to Rev B only is acceptable but ceding it forever is
not.

## References

- [E-N1-002](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`programs/nexus-1/upgrade/NEXUS-1_WORLD_CLASS_UPGRADE.md`](../../programs/nexus-1/upgrade/NEXUS-1_WORLD_CLASS_UPGRADE.md) §8.5
- [`src/rtl/packages/n1_pkg.sv`](../../src/rtl/packages/n1_pkg.sv)
- RISK-010 in [`UPGRADE_RISK_REGISTER.md`](../../programs/portfolio-upgrade/UPGRADE_RISK_REGISTER.md)
