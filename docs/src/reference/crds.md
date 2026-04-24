# CRDs — SovereignZone, DevicePool, ZhiliconWorkload

The authoritative schema lives in
[`platform/operator/config/crd/bases/`](../../../platform/operator/config/crd/bases/).
This page is a human-friendly summary. Use the CRD YAMLs for validation.

## SovereignZone (cluster-scoped)

| Field                          | Type           | Required | Notes                                            |
| ------------------------------ | -------------- | -------- | ------------------------------------------------ |
| `spec.jurisdiction`            | string (2–6)   | ✓        | ISO alpha-2 or region code                        |
| `spec.encryption[]`            | enum list      | ✓        | `aes-256-gcm`, `aes-256-xts`, `chacha20-poly1305`, `kyber-1024-hybrid` |
| `spec.attestationEndpoint`     | https URL      | ✓        | Service in the same jurisdiction                  |
| `spec.keyCustodian`            | string         | ✓        | CA-hierarchy root identifier                      |
| `spec.dataResidencyPolicy`     | string         |          | Free-form regulatory tag                          |
| `spec.allowedChipFamilies[]`   | enum list      |          | Limits which chips can bind                       |
| `status.boundPools`            | int            |          | Observed                                          |
| `status.activeWorkloads`       | int            |          | Observed                                          |
| `status.conditions[]`          | Condition list |          | Includes `SpecValid`, `AttestationEndpointReachable` |

## DevicePool (cluster-scoped)

| Field                          | Type           | Required | Notes                                            |
| ------------------------------ | -------------- | -------- | ------------------------------------------------ |
| `spec.chipFamily`              | enum           | ✓        | One of the five chip programs                    |
| `spec.sovereignZoneRef.name`   | string         | ✓        | Must reference an existing `SovereignZone`        |
| `spec.minDevices`              | int (≥0)       |          | Eligibility threshold                            |
| `spec.nodeSelector`            | map<str, str>  |          | Restricts which nodes contribute devices         |
| `spec.taints[]`                | corev1.Taint   |          | Applied to every device in the pool               |
| `status.availableDevices`      | int            |          | Healthy + unclaimed                              |
| `status.totalDevices`          | int            |          | Registered in pool                               |
| `status.attestationStatus`     | struct         |          | `{ attested, pending, failed, lastChecked }`      |
| `status.conditions[]`          | Condition list |          | Includes `Eligible`                              |

## ZhiliconWorkload (namespaced)

| Field                          | Type              | Required | Notes                                         |
| ------------------------------ | ----------------- | -------- | --------------------------------------------- |
| `spec.poolRef.name`            | string            | ✓        | Cluster-scoped DevicePool                      |
| `spec.devices`                 | int (≥1)          | ✓        | Whole-device granularity                       |
| `spec.template`                | PodTemplate       | ✓        | Operator injects sovereign env + device requests |
| `spec.priority`                | int               |          | Scheduler priority (default 100)               |
| `spec.maxRuntime`              | duration          |          | Force-terminate on expiry                      |
| `spec.attestationRequired`     | bool              |          | Default `true`                                 |
| `status.phase`                 | enum              |          | Pending → Attesting → Running → Succeeded/Failed |
| `status.allocatedDevices`      | int               |          | Observed                                       |
| `status.attestationProofRef`   | string            |          | Proof reference once issued                    |
| `status.startedAt`             | timestamp         |          | Observed                                       |
| `status.completedAt`           | timestamp         |          | Observed                                       |

## Sample manifests

- [SovereignZone — UAE fintech](../../../platform/operator/config/samples/sovereignzone-uae-fintech.yaml)
- [DevicePool — Sentinel-1 pool](../../../platform/operator/config/samples/devicepool-sentinel-uae.yaml)
- [ZhiliconWorkload — ECDSA batch](../../../platform/operator/config/samples/zhiliconworkload-ecdsa-batch.yaml)
