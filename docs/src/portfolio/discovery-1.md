# Discovery-1 — Healthcare AI

**Authoritative spec:**
[`programs/discovery-1/upgrade/DISCOVERY-1_WORLD_CLASS_UPGRADE.md`](../../../programs/discovery-1/upgrade/DISCOVERY-1_WORLD_CLASS_UPGRADE.md),
[`DEEP_TECHNICAL_SPEC.md`](../../../programs/discovery-1/upgrade/DEEP_TECHNICAL_SPEC.md).

## What it is

A healthcare-AI SoC purpose-built for medical imaging (CT / MRI), federated
learning across hospital networks, and privacy-preserving inference under
FDA 510(k) and IEC 60601-1 constraints.

## Key numbers

| Metric                  | Value (post-errata)                                   |
| ----------------------- | ----------------------------------------------------- |
| Process                 | TSMC N3P                                              |
| CPU                     | 16 × Cortex-A720AE @ 3.6 GHz                          |
| AI throughput           | 1 840 TOPS INT8 (32×32 MTL × 640 tiles — frozen per [E-D1-002](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| Memory                  | HBM3E 192 GB, 3.5 TB/s raw / 2.8 TB/s sustained (per [E-D1-003](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| Package                 | CoWoS-L with direct-Cu lid + micro-jet thermal rescue |
| Sustained TDP           | 75 W                                                  |
| Certifications path     | FDA 510(k), IEC 60601-1, HIPAA, CE MDR Class IIa      |

## Distinctive features

- **Spatial Intelligence Tiles** — fixed-function 3D / 4D convolution for
  volumetric medical imaging (Radon transforms, Monte Carlo dose, 4D flow).
  Not emulated on a generic tensor core.
- **Privacy Enclave** — PQC (Kyber-1024, Dilithium-5, Falcon-1024) at
  10 GB/s line rate, ARM CCA realm mode, FHE NTT unit for homomorphic
  inference.
- **DICOM / HL7 integration** — hardware-level DICOM frame parsing and
  HL7 v2 / FHIR routing.

## Current status

Pre-Physical-Design. RTL decision to freeze the MTL v2 systolic at 32×32
is in the action log of [E-D1-002](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md);
downstream floorplan area re-estimation is in flight.
