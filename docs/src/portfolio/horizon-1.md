# Horizon-1 — Radiation-hardened space AI

**Authoritative spec:**
[`programs/horizon-1/upgrade/HORIZON-1_WORLD_CLASS_UPGRADE.md`](../../../programs/horizon-1/upgrade/HORIZON-1_WORLD_CLASS_UPGRADE.md),
[`DEEP_TECHNICAL_SPEC.md`](../../../programs/horizon-1/upgrade/DEEP_TECHNICAL_SPEC.md).

## What it is

A rad-hard AI processor for LEO, MEO, and deep-space missions. TMR-protected
datapath, lockstep CPUs, SEU recovery FSM, 300 krad(Si) TID target, Class V
(QML-V) qualification path.

## Key numbers

| Metric                     | Value (post-errata)                                |
| -------------------------- | -------------------------------------------------- |
| Process                    | GlobalFoundries 22FDX SOI                          |
| CPU                        | 8 × Cortex-R82AE @ 1.6 GHz (lockstep pairs)         |
| AI throughput              | 16.4 TOPS INT8 sustained, TMR-protected (per [E-H1-003](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| Memory                     | LPDDR5-rad, 204 GB/s                               |
| Interconnect               | SpaceFibre 100 Gbps + SRIO Gen 3 + RFCxCL           |
| TID target                 | 300 krad(Si) (per [E-H1-001](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md))|
| SEL LET                    | > 85 MeV·cm²/mg                                    |
| Mission life               | 20 years                                           |

## On-chip runtime

The [Zhilicon edge runtime](../architecture/edge-runtime.md) — a Rust
`no_std`-capable crate — runs on the Cortex-R82AE cores and implements
sovereign enforcement, SEU recovery, deterministic scheduling, and an OTA
update protocol framed over SpaceFibre.

## Qualification plan

Full test plan in [`DEEP_TECHNICAL_SPEC.md` §4.1](../../../programs/horizon-1/upgrade/DEEP_TECHNICAL_SPEC.md)
covers R1 (TID Co-60), R2 (ELDRS), R3 (proton SEE at IUCF or Brookhaven BLIP),
R4 (heavy-ion SEE at TAMU or Brookhaven Tandem), R5 (neutron), plus HTOL,
thermal cycling, vibration, shock, hermeticity, DPA, burn-in, and ESD.

## Current status

RTL in progress, qualification plan locked, proton-SEE facility slot
reserved at IUCF. Target tape-out Q1 2027, first-flight delivery Q3 2028
(Artemis IX or Starshield V4).
