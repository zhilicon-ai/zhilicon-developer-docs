# How-to: read and alert on telemetry

Every platform component emits Prometheus-format metrics. The reference
Grafana dashboard ships in the Helm chart.

## Enable the telemetry stack

```bash
helm upgrade zhilicon ./platform/charts/zhilicon-operator \
  --namespace zhilicon-system \
  --reuse-values \
  --set telemetry.enabled=true \
  --set operator.metrics.serviceMonitor.enabled=true
```

This installs the OpenTelemetry collector DaemonSet and registers a
`ServiceMonitor` with an existing Prometheus-Operator install.

## Key metrics

| Metric                                         | Meaning                                               |
| ---------------------------------------------- | ----------------------------------------------------- |
| `zhilicon_device_pool_available_devices`       | Healthy unclaimed devices per pool                    |
| `zhilicon_device_pool_min_devices`             | Minimum required for Eligible                          |
| `zhilicon_device_attestation_ok_total`         | Cumulative successful attestations                     |
| `zhilicon_device_attestation_failed_total`     | Cumulative failed attestations                         |
| `zhilicon_device_tj_celsius`                    | Per-device junction temperature                        |
| `zhilicon_workload_active`                     | Running workloads per namespace                        |
| `zhilicon_workload_failed_total`               | Cumulative failed workloads per namespace              |
| `controller_runtime_reconcile_time_seconds`    | Operator reconcile latency (histogram)                 |
| `controller_runtime_reconcile_errors_total`    | Operator reconcile errors                              |

## Alerts

Pre-built rules in
[`platform/telemetry/prometheus-rules.yaml`](../../../platform/telemetry/prometheus-rules.yaml)
cover:

- Operator down (critical)
- Operator reconcile-error rate > 0.1/s for 10 min (warning)
- DevicePool under min devices for 10 min (warning)
- Attestation failure rate > 0 for 15 min (critical)
- Device Tj > 95 °C for 2 min (critical)
- Workload failure rate > 5 % per namespace for 30 min (warning)

Load them with:

```bash
kubectl -n zhilicon-system apply -f platform/telemetry/prometheus-rules.yaml
```

## Dashboard

Import
[`platform/telemetry/grafana-dashboards/zhilicon-overview.json`](../../../platform/telemetry/grafana-dashboards/zhilicon-overview.json)
into Grafana. It provides:

- Available devices per pool (timeseries)
- Attestation success / failure (stat)
- Active workloads per namespace (timeseries)
- Device Tj distribution (heatmap)
- Reconcile latency p99 (timeseries)
- Sovereign-zone binding table

## Programmatic access

From inside a pod, the operator's `/metrics` endpoint is reachable at
`http://zhilicon-zhilicon-operator-metrics.zhilicon-system.svc.cluster.local:8080/metrics`.
The [demo notebook](../../../demo/sovereign-inference/notebooks/sovereign_inference.ipynb)
shows a minimal scrape example.
