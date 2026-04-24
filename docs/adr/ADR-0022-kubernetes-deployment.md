---
adr-id: ADR-0022
title: 'Kubernetes deployment shape for the serving layer'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/platform-team", "@zhilicon/sre-coe", "@zhilicon/security-coe"]
---

# ADR-0022: Kubernetes deployment shape for the serving layer

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Platform team, SRE COE, Security COE

## Context

[ADR-0019](ADR-0019-serving-layer.md) shipped a single-tenant
FastAPI serving process. [ADR-0021](ADR-0021-in-process-prometheus-metrics.md)
added the metrics surface. The missing production piece: how does a
sovereign operator actually *deploy* this?

"Here's a Dockerfile, good luck" is unacceptable. Every prospective
customer asks the same sequence of questions:

1. Is this Kubernetes-native?
2. How does it scale?
3. How is it secured in a zero-trust cluster?
4. How does monitoring plug in?
5. What does an upgrade / rollback look like?

Answering each with a live artefact — not a doc fragment — is the
credibility threshold.

## Decision

Ship a reference set of Kubernetes manifests at
[`platform/serving/k8s/`](../../platform/serving/k8s/) covering the
full production surface. Each file is a single resource, human-
readable, diff-friendly, and under 130 lines. No Helm, no Kustomize,
no templating engine.

### Shape

| Resource           | File                       | Purpose                                                    |
| ------------------ | -------------------------- | ---------------------------------------------------------- |
| Namespace          | *(operator-provided)*      | One namespace per SovereignZone.                           |
| ServiceAccount     | `serviceaccount.yaml`      | Dedicated identity, no auto-mounted token, no RoleBindings. |
| ConfigMap          | `configmap.yaml`           | Non-sensitive config (model_id, sampling defaults).        |
| Deployment         | `deployment.yaml`          | The serving pods. Resource requests + probes + security context. |
| Service            | `service.yaml`             | ClusterIP, port 8080.                                      |
| HPA                | `hpa.yaml`                 | Scale on P99 generate latency; fallback to CPU.            |
| ServiceMonitor     | `service-monitor.yaml`     | Prometheus Operator CRD.                                   |
| NetworkPolicy      | `networkpolicy.yaml`       | Zero-trust ingress (ingress-system + monitoring only); egress restricted to DNS. |

Model checkpoints and license-restricted artefacts are
**not** included. Operators mount them via PersistentVolumeClaim or
Secret — the ConfigMap carries only non-sensitive metadata.

### Security posture: restricted Pod Security Standard

Every pod meets the Kubernetes `restricted` profile:

- `runAsNonRoot: true` with a fixed UID 10001
- `readOnlyRootFilesystem: true` with writable `/tmp` via `emptyDir`
- `allowPrivilegeEscalation: false`
- `capabilities: drop: ["ALL"]`
- `seccompProfile: RuntimeDefault`
- `automountServiceAccountToken: false` — the serving layer does not
  call the Kubernetes API

This matches the default Pod Security Admission level in sovereign
clusters — the manifests apply cleanly on day one, no exemptions
required.

### Scaling policy

HPA scales on `zhilicon_generate_latency_seconds_p99` (target: 2 s)
with CPU as a fallback for clusters without the Prometheus adapter.

Scale-up is aggressive: 100% replica growth per 30 s with a 30 s
stabilisation window. Scale-down is conservative: 25% per 60 s with
a 5-minute stabilisation window. Reasoning: LLM serving is
latency-sensitive (fast scale-up) and KV-cache warmup is expensive
to discard (slow scale-down).

`minReplicas: 2` (not zero). Scale-to-zero is possible via KEDA
`ScaledObject` but requires an explicit opt-in — vanilla HPA does
not handle it correctly for stateful LLM processes.

### Network posture

`NetworkPolicy` implements zero-trust ingress:

- Only pods in `ingress-system`, `monitoring`, and the `zhilicon`
  namespace itself may reach port 8080.
- Egress is restricted to cluster DNS (port 53 on `kube-system`/`kube-dns`).

The serving layer has no other outbound traffic today. A deployment
that integrates the Zhilicon attestation service adds a single
egress rule pointing at that service's labels — explicit and small.

### Not bundled: ingress

Sovereign deployments vary on ingress (Istio IngressGateway, Envoy
Gateway API, NGINX, Cloud Armor). We do not ship a reference because
every cluster picks its own. The Zhilicon operator (separate
component) is expected to generate ingress resources per zone.

## Consequences

### Positive

- A cluster-admin can run `kubectl apply -f platform/serving/k8s/`
  and have a working serving deployment in under a minute (modulo
  image availability).
- Every manifest is readable end-to-end without a templating engine
  — diffs are diffs, reviews are reviews.
- 20 tests pin cross-manifest consistency (port names, selector
  labels, env-var alignment with the entrypoint) — the top class of
  Kubernetes bug (silent mismatches between Service and Deployment)
  is test-covered.
- The observability pack ([ADR-0021](ADR-0021-in-process-prometheus-metrics.md))
  and deployment manifests are cross-referenced by tests: if
  someone renames a metric in code, the dashboard / alert tests go
  red.

### Negative / Risks

- No Helm / Kustomize means multi-environment variations (dev /
  staging / prod) copy+paste the manifests. Mitigation: the Zhilicon
  operator issues per-zone overlays; users running without the
  operator can pick up Kustomize themselves.
- `emptyDir` model volume is fine for CI smoke tests but production
  must swap for a PVC or Secret. Documented in both the manifest
  comment and the launch guide.
- No PodDisruptionBudget. A cluster-wide drain during an upgrade
  can take all replicas offline simultaneously. Add when the real
  HA story matters (recommended: `minAvailable: 50%`).

### Neutral

- Manifest count is 7. Kubernetes-native ops teams find this
  familiar; teams coming from pure-Docker-Compose find it
  intimidating. The Docker option remains available for solo
  deployments.

## Alternatives considered

### Option A: Ship a Helm chart instead

Helm makes variations easier but adds a templating layer that
obscures the manifests. For a reference deployment, clarity wins.
We can generate a Helm chart *from* these manifests if needed later
— the other direction loses information.

### Option B: Kustomize base + overlays

Attractive for dev/staging/prod. Rejected for v1 because the
variations we'd introduce (replica counts, model paths, ingress
hosts) are exactly what the Zhilicon operator generates
automatically from a SovereignZone CR. Kustomize duplicates work the
operator already does.

### Option C: Embed operator-specific CRDs inline

Rejected. The Zhilicon operator's CRDs
(`SovereignZone`, `DevicePool`, `ZhiliconWorkload`) are out of scope
for a static-manifest reference. A user running operator-less should
still be able to deploy serving with pure `kubectl apply`.

### Option D: Bundle a Grafana dashboard ConfigMap

Decided *against* bundling but still ship the dashboard JSON
separately. Reason: ConfigMap-based dashboard discovery is Grafana-
version-specific (`grafana_dashboard: "1"` label, or sidecar
annotation depending on version). Shipping the bare JSON is
portable across Grafana deployments.

## References

- [ADR-0019: Serving layer](ADR-0019-serving-layer.md)
- [ADR-0021: In-process Prometheus metrics](ADR-0021-in-process-prometheus-metrics.md)
- [`platform/serving/k8s/`](../../platform/serving/k8s/) — the manifests
- [`platform/serving/observability/`](../../platform/serving/observability/)
  — Grafana dashboard + alert rules
- [`src/tests/sdk/python/test_k8s_manifests.py`](../../src/tests/sdk/python/test_k8s_manifests.py)
  — cross-consistency tests
- [Pod Security Standards — restricted](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted)
- Batch-33 session history.
