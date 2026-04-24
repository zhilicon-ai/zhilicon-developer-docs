# Prometheus — Datacenter AI accelerator

**Authoritative spec:**
[`programs/prometheus/upgrade/PROMETHEUS_WORLD_CLASS_UPGRADE.md`](../../../programs/prometheus/upgrade/PROMETHEUS_WORLD_CLASS_UPGRADE.md),
[`DEEP_TECHNICAL_SPEC.md`](../../../programs/prometheus/upgrade/DEEP_TECHNICAL_SPEC.md),
[`SERVER_TOPOLOGY_AND_SCALING.md`](../../../programs/prometheus/upgrade/SERVER_TOPOLOGY_AND_SCALING.md),
[`FRONTIER_TRAINING_AND_HYPERSCALE.md`](../../../programs/prometheus/upgrade/FRONTIER_TRAINING_AND_HYPERSCALE.md).

## What it is

Zhilicon's flagship datacenter AI accelerator. Eight chiplets — four
Helios (compute), two Iris (IO), two Hestia (HBM4 controller) —
integrated via X-Cube SoIC-X. Targets sovereign AI clusters running
frontier-scale training and inference workloads.

## Key numbers

| Metric                     | Value                                           |
| -------------------------- | ----------------------------------------------- |
| Process (dual-source)      | Intel 18A primary, Samsung SF2 parallel         |
| Chiplets                   | 4 × Helios (compute) + 2 × Iris (IO) + 2 × Hestia (HBM4) |
| SMs per Helios             | 96 (384 total)                                  |
| Tensor core                | 16×16×16 FMA per TC, 1 TC / SM                   |
| AI_CLK                     | 0.875 GHz (per [E-P1-001](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| SM_CLK                     | 2.0 GHz                                         |
| Peak FP8                   | 11 PFLOPS                                       |
| Peak FP4                   | 22 PFLOPS                                       |
| Memory                     | HBM4 384 GB, 14.4 TB/s                          |
| Chip-to-chip (ZLink)       | 1.65 TB/s per direction, 3.3 TB/s aggregate (per [E-P1-002](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| TDP                        | 1 kW (liquid cooling)                           |
| SKUs                       | P-2000 ($58 K) / P-3000 ($72 K) / P-5000 ($95 K) (per [E-P1-003](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |

## Risk posture

Prometheus is **Intel Foundry's first mass external customer for Intel 18A**.
That is the single highest-risk program in the portfolio. The Zhilicon
response:

- A fully-funded Samsung SF2 parallel track maintained at
  second-source-ready state at all times.
- A weekly Monday war room + Friday executive review, with decision
  authority to flip the primary foundry on an 8-week playbook.
- Full detail in
  [`programs/prometheus/governance/INTEL_18A_ESCALATION.md`](../../../programs/prometheus/governance/INTEL_18A_ESCALATION.md).

## Current status

RTL multi-chiplet integration in progress. Helios tape-out targets
Q3 2026. 8-chiplet A0 integration target Q1 2027. GA launch Q3 2027.
