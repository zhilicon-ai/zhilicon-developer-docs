# Nexus-1 — 6G RF + AI chiplet

**Authoritative spec:**
[`programs/nexus-1/upgrade/NEXUS-1_WORLD_CLASS_UPGRADE.md`](../../../programs/nexus-1/upgrade/NEXUS-1_WORLD_CLASS_UPGRADE.md),
[`DEEP_TECHNICAL_SPEC.md`](../../../programs/nexus-1/upgrade/DEEP_TECHNICAL_SPEC.md).

## What it is

A D-band (130–150 GHz) 6G RF chiplet with a 256-element phased array, a
GaN-on-SiC front end, and a co-integrated CMOS chiplet carrying AI4PHY
(learned beamforming, channel estimation, equalisation).

## Key numbers

| Metric                    | Rev A (Q4 2026 tape-out)                            | Rev B (Q2 2027 target)                              |
| ------------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| Process (RF)              | GaN-on-SiC 0.1 µm                                    | Same + new eGaN SPDT                                 |
| Process (baseband)        | TSMC N5P                                             | Same                                                 |
| System NF                 | 5.98 dB typ. (per [E-N1-002](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) | ≤ 4.8 dB target (eGaN SPDT + revised LNA1)            |
| PA Psat per element       | +31 dBm                                              | Same                                                 |
| PAE                       | 22 % aggressive / 18 % fallback                       | Same                                                 |
| LEO link margin (QPSK)    | + 1.5 dB                                             | ≥ + 4.0 dB                                           |
| LEO link margin (64-QAM)  | 0 dB (not supported on Rev A)                         | ≥ + 3.0 dB (enabled)                                 |

## Modulations supported

- **Rev A:** QPSK, 16-QAM for LEO; up to 256-QAM for terrestrial backhaul
  (clear-sky margin).
- **Rev B:** 64-QAM and 256-QAM on LEO, gated by DPD convergence.

## Current status

Rev A tape-out targets Q4 2026. eGaN SPDT + LNA1 shuttle wafers at Macom and
Qorvo planned Q3–Q4 2026 for Rev B qualification. Gate criterion to lock
Rev B: at least one foundry demonstrates IL ≤ 0.8 dB SPDT + NF ≤ 3.6 dB LNA1
across −40 °C to +85 °C on the breakout.
