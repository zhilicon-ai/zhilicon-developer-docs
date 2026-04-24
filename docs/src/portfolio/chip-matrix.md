# Chip matrix

Definitive reference — kept in lock-step with
[`README.md` at the repo root](../../../README.md) and with
[`programs/portfolio-upgrade/GAP_ANALYSIS_MATRIX.md`](../../../programs/portfolio-upgrade/GAP_ANALYSIS_MATRIX.md).

## Headline specs

| Program       | Target market                 | Process node           | Headline spec                                       | Package            | Target tape-out | Target launch | Status   |
| ------------- | ----------------------------- | ---------------------- | --------------------------------------------------- | ------------------ | --------------- | ------------- | -------- |
| **Discovery-1** | Healthcare sovereign AI       | TSMC N3P               | 1 840 TOPS INT8 · 192 GB HBM3E · 75 W sustained     | CoWoS-L            | Q3 2026         | Q3 2027 pilot | Pre-PD   |
| **Horizon-1**   | Radiation-hardened space AI   | GF 22FDX SOI           | 16.4 TOPS INT8 sustained TMR · LPDDR5-rad · 300 krad TID | AEC-Q100 equiv. | Q1 2027         | Q3 2028 flight | RTL       |
| **Nexus-1**     | 6G RF + AI chiplet            | GaN 0.1 µm + N5P CMOS  | 256-element D-band phased array · 40 Gbps / link    | SoIC hybrid bond   | Q4 2026         | Q3 2027       | RTL + RF  |
| **Sentinel-1**  | Crypto / fintech / ZKP        | TSMC N3P               | ZKP + PQC + FIPS 140-3 · 62 M ECDSA/s · 144 GB HBM3E | CoWoS-S          | Q2 2026         | Q3 2027       | Pre-tape-out |
| **Prometheus**  | Datacenter AI accelerator     | Intel 18A / Samsung SF2 | 11 PFLOPS FP8 · 384 GB HBM4 · 8-chiplet · 1 kW    | X-Cube SoIC-X      | Q3 2026         | Q3 2027       | Multi-chiplet integration |

## Per-chip deep dives

See the per-program pages:

- [Discovery-1 — Healthcare AI](discovery-1.md)
- [Horizon-1 — Rad-hard space](horizon-1.md)
- [Nexus-1 — 6G RF + AI](nexus-1.md)
- [Sentinel-1 — Crypto / ZKP](sentinel-1.md)
- [Prometheus — Datacenter AI](prometheus.md)

## Errata status

All 21 portfolio errata are resolved as of 2026-04-17. The authoritative
tracker is
[`programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md`](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md).
Each per-chip deep-dive page below links to the specific errata that gated
its current status.
