---
adr-id: ADR-0001
title: Errata-first spec hygiene
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/portfolio-stewards", "@zhilicon/chief-architect"]
---

# ADR-0001: Errata-first spec hygiene

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Portfolio stewards + Chief architect

## Context

The portfolio upgrade produced 21 numerical / architectural inconsistencies
across five chip programs — from the Discovery-1 MTL systolic size dispute
to the Sentinel-1 ZKP-throughput-per-constraint-count issue. Left unchecked,
a number written into one spec would propagate into slides, customer briefs,
and BOM models; a later correction in the underlying spec would not
automatically reach those copies.

Worse, the initial `GAP_ANALYSIS_MATRIX.md` contained rows marked `✅ Closed`
that directly contradicted open entries in
`SPEC_CORRECTIONS_AND_ERRATA.md` — the same number declared both corrected
and in-dispute, depending on which document the reader opened first.

## Decision

Adopt errata-first hygiene: the errata document
([`SPEC_CORRECTIONS_AND_ERRATA.md`](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md))
is the single authoritative source for the status of any disputed number or
decision across the portfolio. Every other document quoting that number
must either:

1. Link to the erratum; or
2. State the post-errata value verbatim and carry the erratum ID in a
   visible marker (e.g. "per E-D1-002").

`GAP_ANALYSIS_MATRIX.md` entries that contradict an open erratum get a
`⚠️ Open — see errata` marker, never `✅ Closed`. The legend (✅ / 🔄 / ⚠️)
is explicit at the top of the matrix.

Any new disputed number added to any spec spawns a new erratum entry in
the same PR. CODEOWNERS for the affected chip and for portfolio-stewards
are both required reviewers.

## Consequences

### Positive

- One document tells the truth about every disputed number; everything
  else defers to it.
- Board, customer, and regulator materials trace back through a single
  audit chain.
- Engineers know that a quoted number in a spec is either erratum-tracked
  or stable.

### Negative / Risks

- Slight overhead when adding new numbers: contributors must decide
  whether the number warrants an erratum or is settled.
- Rigor depends on reviewer discipline — CODEOWNERS + PR template
  checklist enforce it.

### Neutral

- The convention is the same one used at the JEDEC / MIL-STD level for
  chip-program errata — engineers coming from tier-1 chip companies will
  recognise it.

## Alternatives considered

### Option A: Author numbers directly in each spec with no cross-reference

Rejected. Produces the very drift that motivated this ADR.

### Option B: Use a machine-parsed "numbers registry" (YAML / JSON)

Rejected for now. The overhead of maintaining a registry that tracks
every measured number in prose documents is higher than the value it
delivers at this stage. Revisit if the portfolio grows past the current
5 chips.

## References

- [`SPEC_CORRECTIONS_AND_ERRATA.md`](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
- [`GAP_ANALYSIS_MATRIX.md`](../../programs/portfolio-upgrade/GAP_ANALYSIS_MATRIX.md)
