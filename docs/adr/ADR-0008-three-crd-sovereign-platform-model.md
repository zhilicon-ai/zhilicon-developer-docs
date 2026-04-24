---
adr-id: ADR-0008
title: Three-CRD sovereign-platform model (Zone / Pool / Workload)
status: Accepted
date: 2026-04-17
deciders: ["@zhilicon/platform-team", "@zhilicon/kubernetes-coe", "@zhilicon/security-coe"]
---

# ADR-0008: Three-CRD sovereign-platform model

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Platform team + Kubernetes COE + Security COE

## Context

The Zhilicon platform-tier goal is "sovereign by construction" — the
software refuses to execute outside its declared jurisdiction. Expressing
that in Kubernetes requires choosing a resource model. Candidates:

1. **Single CRD (`ZhiliconJob`)** — one object per workload, carries
   everything. Easy to author; mixes concerns; difficult to enforce
   the many-jobs-one-zone invariant.
2. **Pair of CRDs** — `Zone` + `Job`. Missing the intermediate device-
   pool abstraction means every job must pick individual nodes, which
   fights Kubernetes' scheduler.
3. **Three CRDs** — `SovereignZone` + `DevicePool` + `ZhiliconWorkload`.
   Aligns with existing Kubernetes patterns (`NodePool`, `StorageClass`)
   and separates concerns cleanly.

NVIDIA's GPU Operator, AMD's ROCm Operator, and Intel's GPU Device
Plugin all use analogous two- or three-CRD layouts. Nobody in the
industry ships a single mega-CRD for accelerator fleet management.

## Decision

Three cluster-scoped / namespaced CRDs:

- **`SovereignZone`** (cluster-scoped) — declares a jurisdiction, an
  encryption envelope, an attestation endpoint, and a key custodian.
  Lifecycle: created once per jurisdiction by the platform team;
  rarely changes.
- **`DevicePool`** (cluster-scoped) — groups devices of a single chip
  family, bound to exactly one SovereignZone. Lifecycle: created per
  hardware cluster-of-nodes; moderately stable.
- **`ZhiliconWorkload`** (namespaced) — a Pod template + an `int
  devices` count + a `poolRef`. Lifecycle: short-lived; created and
  destroyed by application teams.

Enforcement invariants baked into the reconcilers:

- A `DevicePool` without a valid `sovereignZoneRef` is ineligible.
- A `ZhiliconWorkload` cannot run unless its `poolRef` is `Eligible`
  *and* the pool's zone is `SpecValid` with a reachable attestation
  endpoint.
- The managed Pod receives 4 environment variables
  (`ZHILICON_SOVEREIGN_ZONE`, `ZHILICON_JURISDICTION`,
  `ZHILICON_ATTESTATION_ENDPOINT`, `ZHILICON_ATTESTATION_PROOF_REF`)
  plus device-plugin resource requests — injected by the operator,
  not trusted from the workload spec.

## Consequences

### Positive

- Separation of concerns: jurisdictions (rare ops event) vs. hardware
  pools (infra ops) vs. workloads (application teams). Each team has
  the minimum scope.
- Existing Kubernetes patterns apply — RBAC, quotas, namespaces,
  ResourceQuota — without custom machinery.
- Sovereign enforcement is centralised in the reconcilers, not
  sprinkled across application code.

### Negative / Risks

- Three CRDs instead of one means three API versions to maintain.
  v1alpha1 today; promotion to v1beta1 gated on a year of production
  feedback.
- New users must learn three concepts. Mitigated by the getting-started
  tutorial and the demo, which walk through all three in under 10 min.

### Neutral

- The model is extensible — additional CRDs (e.g. `SovereignModel`,
  `AttestedConfig`) can slot in without changing the existing three.

## Alternatives considered

### Option A: Single mega-CRD

Rejected. Couples jurisdictions, pools, and workloads into one
lifecycle.

### Option B: Two-CRD (Zone + Job)

Rejected. Loses the device-pool abstraction; scheduling degenerates to
explicit node selection.

### Option C: Kubernetes-native only (use existing `NodePool`, `Namespace` + labels)

Rejected. Cannot express the sovereign-zone enforcement invariants as
a reconciler — it would fall on each application to check them.

## References

- [`platform/operator/`](../../platform/operator/)
- [`platform/operator/config/crd/bases/`](../../platform/operator/config/crd/bases/)
- [ADR-0009](ADR-0009-attestation-service-per-jurisdiction.md)
