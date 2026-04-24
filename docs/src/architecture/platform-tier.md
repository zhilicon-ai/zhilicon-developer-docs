# Platform tier

The platform tier is the operational skin around the silicon. Source lives
under [`platform/`](../../../platform/).

## Components

| Component                                   | Language     | Role                                                                 |
| ------------------------------------------- | ------------ | -------------------------------------------------------------------- |
| [Operator](../../../platform/operator/)     | Go (kubebuilder) | Reconciles `DevicePool`, `SovereignZone`, `ZhiliconWorkload` CRDs     |
| [Device plugin](../../../platform/device-plugin/) | Go           | Registers `zhilicon.io/device` with the kubelet on each node          |
| [Attestation](../../../platform/attestation/) | Rust (axum)  | Issues and verifies Ed25519-signed attestation proofs per jurisdiction |
| [Fleet CLI](../../../platform/fleet-cli/)   | Python (click) | `zhctl` — every platform primitive driven from the command line       |
| [Telemetry](../../../platform/telemetry/)   | YAML + JSON  | OTel collector config · Prometheus alert rules · Grafana dashboards  |
| [Runbooks](../../../platform/runbooks/)     | Markdown     | Operational playbooks for recurring events                            |
| [Helm chart](../../../platform/charts/zhilicon-operator/) | Helm    | One-shot installer with sensible defaults + opt-in extensions         |

## Architecture

```text
  +---------+      CRDs       +---------------------+
  |  User   | -------------->  |  zhctl (CLI)         |
  +---------+                  +----------+----------+
                                          |
                                          v
                              +------------------------+
                              |  kube-apiserver        |
                              +------------+-----------+
                                           |
                                           v
        +-------------------------------------------------------------+
        |  Zhilicon Operator (Go)                                     |
        |   - DevicePoolReconciler                                    |
        |   - SovereignZoneReconciler                                 |
        |   - ZhiliconWorkloadReconciler                              |
        +-----+--------------------------+----------------------------+
              |                          |
              v                          v
    +----------------+          +---------------------------+
    | Device plugin  |          |  Attestation service       |
    | DaemonSet      |          |  (Rust, HTTP)              |
    +----------------+          +-------------+--------------+
              |                               |
              v                               v
    +----------------+             +---------------------------+
    | Zhilicon       |             |  OpenTelemetry collector   |
    | silicon        |             |  + Prometheus + Grafana    |
    +----------------+             +---------------------------+
```

## Sovereign by construction

A workload cannot run unless:

1. the `SovereignZone` it binds to is valid,
2. the `DevicePool` it targets is `Eligible` and bound to that zone,
3. a fresh attestation proof has been issued for the pod,
4. the node the pod lands on advertises the right chip family,
5. the kubelet allocated an actual device via the device plugin.

The reconciler refuses any request that skips a step.

## Install

The one-shot install path is the [demo walkthrough](../getting-started/install.md).
Full Helm reference: [`platform/charts/zhilicon-operator/README.md`](../../../platform/charts/zhilicon-operator/README.md).
