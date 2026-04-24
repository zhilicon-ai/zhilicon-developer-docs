# Sentinel-1 — Crypto / fintech / ZKP

**Authoritative spec:**
[`programs/sentinel-1/upgrade/SENTINEL-1_WORLD_CLASS_UPGRADE.md`](../../../programs/sentinel-1/upgrade/SENTINEL-1_WORLD_CLASS_UPGRADE.md),
[`DEEP_TECHNICAL_SPEC.md`](../../../programs/sentinel-1/upgrade/DEEP_TECHNICAL_SPEC.md).

## What it is

A cryptographic-workload accelerator for banks, exchanges, CBDC operators,
and zero-knowledge rollup infrastructure. Three unified TEE realms (CCA,
SGX-compat, hyper-enclave), FIPS 140-3 submission in Q4 2026, full
post-quantum coverage, and two ZKP engines.

## Key numbers

| Metric                           | Value                                           |
| -------------------------------- | ----------------------------------------------- |
| Process                          | TSMC N3P                                        |
| Die area                         | 520 mm²                                         |
| ECDSA P-256 verify               | 62 M / s                                        |
| AES-256-GCM                      | 850 GB / s                                      |
| SHA-256                          | 1 240 GHash / s                                 |
| TRNG                             | Two-stage RO → Von Neumann → HMAC-DRBG (per [E-S1-003](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)) |
| PQC                              | Kyber-1024, Dilithium-5, Falcon-1024, SPHINCS+, Classic McEliece, BIKE, HQC |
| Memory                           | HBM3E 144 GB, 5.4 TB/s                          |
| FIPS 140-3 submission            | Q4 2026 (pulled in from Q2 2027)                |

## ZKP throughput per constraint count

```admonish warning
Every ZKP number is reported per constraint count — no single-number claim
is permitted in customer-facing material. See
[E-S1-001 resolution](../../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md).
```

| Constraint count   | Groth16 (both engines, projection)   | Typical application                 |
| ------------------ | ------------------------------------ | ----------------------------------- |
| 2^14 (16 K)        | ≈ 66 000 / s                         | Micro-proofs, onboarding            |
| 2^16 (64 K)        | ≈ 16 600 / s                         | Simple payment / attestation        |
| 2^18 (256 K)       | ≈ 4 200 / s                          | Privacy-preserving identity         |
| 2^20 (1 M)         | ≈ 142 / s                            | Rollups, zkEVM inner proof          |
| 2^22 (4 M)         | ≈ 36 / s                             | zkEVM outer / aggregated proof      |

## Current status

Pre-tape-out, target Q2 2026. Cryptographic formal-verification plan in
[`programs/sentinel-1/plans/CRYPTO_FORMAL_VERIFICATION_PLAN.md`](../../../programs/sentinel-1/plans/CRYPTO_FORMAL_VERIFICATION_PLAN.md)
schedules Coq proofs for AES-GCM, ECDSA, Kyber, Dilithium, the HMAC-DRBG
conditioning chain, secure boot, TEE realm isolation, and key-lifecycle
invariants over 26 weeks.
