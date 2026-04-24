---
adr-id: ADR-0003
title: Horizon-1 single TMR-protected systolic, 16.4 TOPS honest
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/horizon-1-leads", "@zhilicon/verification-coe", "@zhilicon/program-management-office"]
---

# ADR-0003: Horizon-1 single TMR-protected systolic, 16.4 TOPS honest

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Horizon-1 leads + Verification COE + Program Management Office

## Context

The V3.0 Horizon-1 upgrade claimed 32 TOPS INT8 from "8 tiles × 32×32
systolic × 1 024 MACs × 1 GHz". The arithmetic only works if each tile
runs *dual-issue* — two independent 32×32 systolic arrays per tile. No
dual-issue mechanism was documented ([E-H1-003](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)).

Horizon-1 is a Class V (QML-V) rad-hard program. Each 32×32 systolic is
already triplicated (TMR) at the circuit level, consuming the die-area
budget that a second array would require. Adding a dual-issue pipeline
would force either dropping TMR (unacceptable for space) or enlarging the
die (yield + cost penalty on 22FDX SOI, with a hard Class V area cap).

## Decision

Retain a **single TMR-protected 32×32 systolic** per AI tile. Eight tiles
× 2.048 TOPS per tile × 1 GHz = **16.4 TOPS INT8 sustained**. The honest
number replaces the 32 TOPS peak claim in every customer-facing artefact.

Rationale in one sentence: the space market values sustained, predictable
throughput with proven radiation hardness over peak TOPS numbers, and
16.4 TOPS is still a 26× improvement over H1 V2.0's 0.62 TOPS.

## Consequences

### Positive

- Honest headline survives customer benchmarking committees.
- TMR stays in place — the rad-hard value proposition is preserved.
- Class V die area stays inside the qualification budget.

### Negative / Risks

- Headline drops from 32 TOPS to 16.4 TOPS. Marketing repositions the
  pitch as "26× V2.0 at Class V, 2× more efficient than Orin AGX on
  target rad-hard workloads" — Orin AGX being excluded from flight.
- A customer who evaluated Horizon-1 on the 32 TOPS number must be
  re-briefed. No published datasheet yet carried that number, so the
  blast radius is limited to internal briefings.

### Neutral

- The single-vs-dual systolic decision is internal to the AI tile
  micro-architecture — no SDK, compiler, or ABI impact.

## Alternatives considered

### Option A: Design a real dual-issue pipeline

Rejected. Would require a second 32×32 systolic per tile (2× area)
without TMR (unacceptable) or 2× TMR (6× area — well outside the
Class V budget).

### Option B: Drop TMR to recover area for dual-issue

Rejected outright. Class V + space market requires TMR. Dropping TMR
would invalidate the program thesis.

## References

- [E-H1-003](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`programs/horizon-1/upgrade/HORIZON-1_WORLD_CLASS_UPGRADE.md`](../../programs/horizon-1/upgrade/HORIZON-1_WORLD_CLASS_UPGRADE.md)
- [`src/rtl/packages/h1_pkg.sv`](../../src/rtl/packages/h1_pkg.sv)
