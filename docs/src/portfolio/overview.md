# Portfolio overview

Zhilicon ships five chips. They share the shared-IP library, the platform
tier, the SDK, and the formal-verification methodology. Each one targets a
specific vertical where a sovereign-AI requirement is a hard constraint.

| Chip            | Market                         | Process                | Key spec                                            | Status today                    |
| --------------- | ------------------------------ | ---------------------- | --------------------------------------------------- | ------------------------------- |
| **Discovery-1** | Healthcare / medical AI        | TSMC N3P               | 1 840 TOPS INT8, 192 GB HBM3E, 75 W sustained       | Pre-PD; 32×32 MTL frozen        |
| **Horizon-1**   | Rad-hard space AI              | GF 22FDX SOI           | 16.4 TOPS INT8 TMR-protected, 300 krad TID, Class V  | RTL + qualification plan locked |
| **Nexus-1**     | 6G RF + AI chiplet             | GaN 0.1 µm + N5P CMOS  | 256-el D-band phased array, 40 Gbps / link          | Rev A (5.98 dB NF) shipping Q4 2026; Rev B targeting 4.8 dB NF |
| **Sentinel-1**  | Crypto / fintech / ZKP         | TSMC N3P               | 62 M ECDSA/s, full PQC, ZKP multi-constraint table  | Pre-tapeout (Q2 2026)           |
| **Prometheus**  | Datacenter AI accelerator      | Intel 18A / Samsung SF2| 11 PFLOPS FP8, 384 GB HBM4, 8-chiplet, 1 kW         | Multi-chiplet integration; Intel 18A war-room running |

## Shared infrastructure

```text
                            ┌──────────────────────────┐
                            │   Platform Tier          │
                            │   (operator + attest +   │
                            │    device plugin + CLI)  │
                            └────────────┬─────────────┘
                                         │
 ┌──────────┬──────────┬──────────┬──────┴───┬──────────┐
 │ Discovery-1 │ Horizon-1 │ Nexus-1 │ Sentinel-1 │ Prometheus │
 └──────┬──────┴──────┬───┴─────┬────┴────┬──────┴─────┬──────┘
        │             │         │         │            │
        ▼             ▼         ▼         ▼            ▼
   ┌────────────────────────────────────────────────────────┐
   │   Shared IP library (AXI, NoC, AES, NTT, TMR, ECC, …)  │
   └────────────────────────────────────────────────────────┘
```

Every chip shares:

- the **shared-IP RTL library** (see [shared-IP formal coverage](../reference/formal-library.md));
- the **platform operator** for fleet scheduling under a sovereign zone;
- the **SDK dispatch path** with per-family backends;
- the **compliance / errata discipline** maintained in
  [`programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md`](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md).

## What changes per chip

| Dimension         | Discovery-1 | Horizon-1 | Nexus-1 | Sentinel-1 | Prometheus |
| ----------------- | ----------- | --------- | ------- | ---------- | ---------- |
| CPU               | Cortex-A720AE | Cortex-R82AE (lockstep) | Cortex N5P | Custom TEE-hardened | Internal SM-class |
| AI accelerator    | 640 × MTL 32×32 | 8 × AI tile 32×32 TMR  | AI4PHY | ZKP NTT + MSM | 384 SMs × 16×16×16 TCs |
| Memory            | HBM3E 192 GB | LPDDR5-rad 204 GB/s | LPDDR5 | HBM3E 144 GB | HBM4 384 GB |
| Certifications    | FDA 510(k), IEC 60601-1 | MIL-PRF-38535 Class V, ESCC | FCC, CE, TELEC | FIPS 140-3, CC EAL4+ | Export-control compliant |
