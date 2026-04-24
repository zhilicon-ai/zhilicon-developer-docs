---
adr-id: ADR-0009
title: Attestation service is per-jurisdiction, not global
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/security-coe", "@zhilicon/platform-team"]
---

# ADR-0009: Attestation service is per-jurisdiction, not global

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Security COE + Platform team

## Context

A sovereign zone is defined by its jurisdiction (UAE, EU, GCC, USA, …).
Sovereignty requires that the attestation-authority key lives in the
same jurisdiction as the workloads it attests — if the key sits in a
foreign data centre, the sovereignty claim is fictional.

A naïve architecture runs a single global attestation service. That
centralises key material, concentrates blast radius if compromised,
and fails the first customer-regulator review in any serious
jurisdiction.

## Decision

**One attestation service per jurisdiction.** Each deployment is
scoped to a single sovereign zone, carries exactly one signing key,
and is operated by the jurisdiction's key custodian (e.g. the UAE
Central Bank CA for `jurisdiction: ae`).

Consequences baked into the operator:

- A `SovereignZone` carries an `attestationEndpoint` that the operator
  probes during reconciliation. If the endpoint is unreachable or the
  zone descriptor mismatches, the operator marks the zone's
  `AttestationEndpointReachable` condition `False`.
- The attestation service refuses any proof request whose zone field
  does not match its configured jurisdiction (`ZoneMismatch` error).
- The service's signing key lives in the process; key rotation is a
  jurisdiction-policy event with its own runbook. Keys never leave
  the service or cross jurisdictional boundaries.

Operational model in the Helm chart: enabling `attestation.enabled=true`
deploys exactly one attestation service alongside the operator. A
multi-jurisdiction cluster runs multiple Helm releases, one per zone.

## Consequences

### Positive

- Data-residency audit is simple: point at the jurisdiction, point at
  the signing key, show the pair don't leave.
- Blast radius of a single key compromise is bounded to one zone.
- The service's responsibilities are small and auditable (≈ 400 lines
  of Rust).

### Negative / Risks

- Running multiple attestation services scales with the number of
  jurisdictions. Ops overhead grows linearly.
- Cross-zone workloads become impossible by construction. This is the
  right default, but an explicit "multi-zone federation" story needs a
  future ADR if the customer base demands it.

### Neutral

- The wire format (nonce, Ed25519, zone descriptor) is identical across
  services — the only per-deployment difference is the jurisdiction
  tag and the signing key.

## Alternatives considered

### Option A: One global attestation service with per-zone namespaces

Rejected. Concentrates keys cross-jurisdictionally; fails the core
sovereignty claim.

### Option B: Each device performs self-attestation directly to the ground authority

Rejected. Scales badly; takes the operator out of the enforcement
loop; makes attestation-proof rotation a per-device scripting problem.

## References

- [`platform/attestation/`](../../platform/attestation/)
- [`RELIABILITY_SECURITY_SUPPLY_CHAIN.md` §2](../../programs/portfolio-upgrade/RELIABILITY_SECURITY_SUPPLY_CHAIN.md)
- [ADR-0008](ADR-0008-three-crd-sovereign-platform-model.md)
