# Observability

See the task-oriented [How-to → Read and alert on telemetry](../how-to/read-telemetry.md)
for install steps, metric names, and alert rules.

## Architecture

```text
  +-----------------+        +---------------------+
  | Operator + DP   |  --->  | OTel Collector       |  --->  Customer
  | metrics (8080)  |        | DaemonSet            |        backend
  +-----------------+        |  - prometheus scrape |        (OTLP)
                             |  - resource attrs    |
                             |  - batch + memlimit  |
                             +---------+-----------+
                                        |
                                        v
                             +---------------------+
                             |  Prometheus          |
                             |  (alert rules)       |
                             +---------+-----------+
                                        |
                                        v
                             +---------------------+
                             |  Grafana             |
                             |  (overview dashboard)|
                             +---------------------+
```

## What to watch

| Signal                        | Dashboard panel                   | Alert rule                              |
| ----------------------------- | --------------------------------- | --------------------------------------- |
| Pool availability             | Available devices by pool         | `ZhiliconDevicePoolUnderMin`             |
| Attestation health            | Attestation success / failure     | `ZhiliconDeviceAttestationFailing`       |
| Reconcile latency             | Operator reconcile latency (p99)   | `ZhiliconOperatorReconcileErrors`        |
| Device thermal                | Device Tj distribution (heatmap)  | `ZhiliconDeviceThermalCritical`          |
| Workload throughput           | Active workloads per namespace    | `ZhiliconWorkloadFailedRate`             |

## Benchmark ingest — the ADR-0013 guard

Performance numbers on the kernel dashboard come from the harness at
[`src/tools/bench/kernels_bench.py`](../../../src/tools/bench/kernels_bench.py).
The harness refuses to emit records unless the loaded
`zhilicon._kernels` extension advertises `__backend_kind__ == "native"`
— the contract established by
[ADR-0013](../../adr/ADR-0013-kernels-emulation-backend.md).

The dashboard's ingest job therefore relies on two things:

1. **Harness-side refusal.** Exit code `2` from `kernels_bench.py`
   means "emulation backend detected; no file written". The ingest job
   treats exit `2` as a *configuration error* (someone ran the bench on
   the wrong host), distinct from exit `1` (harness crashed).
2. **Record-side stamping.** Every JSONL record carries
   `"backend_kind": "native"` as a first-class field. Grafana queries
   filter on this so that even if a rogue file slipped through, it
   would fail to join against the expected backend-kind label.

Both checks together keep laptop-speed emulation numbers — which are
100×–1000× slower than silicon — off the performance dashboards. If
you see a Grafana panel that looks suspiciously slow, check the
`backend_kind` label before opening an incident.

## Troubleshooting

- **Metrics not flowing:** confirm the operator's Service is reachable and
  `/metrics` returns Prometheus-format text. Then check the collector logs
  for scrape errors.
- **Wrong zone tags on metrics:** the OTel collector injects
  `sovereign.zone` and `silicon.chip_family` via the `attributes/sovereign`
  processor. If zone tags come back empty, check that the scraped node
  labels are present.
- **Bench ingest reported 0 records:** the harness exited with code 2 —
  the target host is running the emulation backend. See the
  [bench harness README](../../../src/tools/bench/README.md).
