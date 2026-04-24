---
adr-id: ADR-0007
title: Prometheus Intel 18A primary with Samsung SF2 parallel track
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/ceo-office", "@zhilicon/prometheus-leads", "@zhilicon/program-management-office"]
---

# ADR-0007: Prometheus Intel 18A primary with Samsung SF2 parallel track

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** CEO + Prometheus leads + PMO + CFO

## Context

Prometheus is the flagship datacenter AI accelerator and Zhilicon's
highest-revenue-per-unit product. It is also Intel Foundry's **first
mass external customer on Intel 18A**. Historical first-customer yield
for new nodes has slipped 6–12 months. A 30% probability × 9/10 impact
risk (`RISK-001`) is an order of magnitude higher than NVIDIA accepts
for its flagship. A single-sourcing error would slip the entire
portfolio coordinated launch by two-plus quarters.

At the same time, moving off Intel 18A entirely loses access to Intel's
transistor-density and power advantages that the P1 business case
assumes. Samsung SF2 is the nearest alternative — first-gen GAA,
comparable density, different yield curve.

## Decision

Run **two foundry tracks in parallel**, with clear primary / secondary
roles and a CEO-authorised switch playbook:

- **Intel 18A** is primary. Helios (the compute chiplet) targets
  tape-out Q3 2026 on 18A. All initial marketing, customer MoUs, and
  supply-chain commitments reference the 18A shipment.
- **Samsung SF2** is a *fully-funded*, *fully-staffed* parallel track.
  The SF2 team maintains a mirror Helios RC2 RTL branch, a working
  synthesis + floorplan + timing-closure pipeline, a shuttle wafer,
  and enough headcount that the gap from Intel handoff to Samsung
  tape-out is bounded at 8 weeks.
- A **weekly war room** tracks 8 Green / Amber / Red indicators;
  the CEO has decision authority to flip the primary to Samsung at
  any time, with a documented 8-week switch playbook.

Full governance in
[`programs/prometheus/governance/INTEL_18A_ESCALATION.md`](../../programs/prometheus/governance/INTEL_18A_ESCALATION.md).

Funding commitment: $15.9 M annual (keep-alive); $18 M pre-authorised
(RC2 tape-out NRE on trigger).

## Consequences

### Positive

- Process-node single-point-of-failure is removed as a program risk.
- Customer briefings can credibly say "foundry-independent bring-up".
- If Intel 18A slips by > 4 weeks, the CEO can make the switch in
  ≤ 7 days and the portfolio ships with a 3-month total slip — bounded
  and modelled.

### Negative / Risks

- Running two tracks costs $15.9 M / year keep-alive. That is real
  opex on a single product line.
- Design work must stay foundry-portable — every ML compiler back-end
  target, timing constraint, and PDN analysis lives in dual-track form.
  Team discipline required.
- Intel Foundry relationship is sensitive. The governance document is
  explicit that the parallel track is risk mitigation, not a threat.

### Neutral

- SKU pricing (P-2000 / P-3000 / P-5000) is set at the foundry-
  independent level. A Samsung ramp would not change customer prices.

## Alternatives considered

### Option A: Single-source Intel 18A

Rejected. 30% probability × 9/10 impact is unacceptable for a
flagship product.

### Option B: Single-source Samsung SF2

Rejected. Loses Intel 18A transistor-density / power advantages; the
commercial pitch depends on them.

### Option C: Pre-build on TSMC N3P as third option

Considered and rejected (for now). TSMC capacity is allocated to NVIDIA
/ Apple on N3P; credible access at the required volume is not on the
table for 2027.

## References

- [`programs/prometheus/governance/INTEL_18A_ESCALATION.md`](../../programs/prometheus/governance/INTEL_18A_ESCALATION.md)
- `RISK-001`, `RISK-008` in [`UPGRADE_RISK_REGISTER.md`](../../programs/portfolio-upgrade/UPGRADE_RISK_REGISTER.md)
- [`src/rtl/packages/p1_pkg.sv`](../../src/rtl/packages/p1_pkg.sv)
