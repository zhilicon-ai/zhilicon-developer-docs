# Launch Zhilicon serving in production

End-to-end path from `git clone` to a signed, observable, autoscaling
Zhilicon serving deployment on a sovereign Kubernetes cluster. Every
step here ties back to an ADR and a tested artefact — no
unreferenced steps, no hand-wavy configuration.

> **Target audience.** A platform engineer landing a first
> deployment into a sovereign-zone cluster.
>
> **Prerequisites.** Kubernetes 1.28+, a container registry you can
> push to, and either the Prometheus Operator or vanilla Prometheus
> with `kubernetes_sd_configs`.

## 1. Build the image

```sh
# From repo root.
docker build -f src/sdk/python/zhilicon/serving/Dockerfile \
    -t your-registry.example.com/zhilicon/serving:0.1.0 .
docker push your-registry.example.com/zhilicon/serving:0.1.0
```

The Dockerfile ([ADR-0019](../../adr/ADR-0019-serving-layer.md))
bakes in the kernel extension, the serving app, and the byte
tokenizer. The image boots in under 5 seconds on a typical cluster
node.

If you have not built the kernel extension before, verify locally
first:

```sh
python -m pip install --upgrade "setuptools>=64" wheel "pybind11>=2.12" numpy
pip install -e src/sdk/python -v
python -m zhilicon doctor
```

## 2. Prepare a signed checkpoint

The serving image ships with a random-init model for CI smoke tests.
Production deploys a real checkpoint — either via a
PersistentVolumeClaim (for multi-GB models) or a Secret (for small
license-restricted weights).

```sh
# From inside a trusted build environment:
python -m zhilicon generate --prompt "sanity-check" --max-new-tokens 4 \
    --model /path/to/your/checkpoint.npz
```

Checkpoint format is documented in
[ADR-0017](../../adr/ADR-0017-model-persistence-format.md) — a single
`.npz` archive carrying weights + JSON config, loadable via numpy's
safe loader path (no code-execution surface).

## 3. Adjust the manifests

```sh
cp -r platform/serving/k8s /tmp/my-deployment
```

Edit `/tmp/my-deployment/deployment.yaml`:

- Replace the image reference with your registry push.
- Swap the `emptyDir` model volume for a PVC or Secret mount.
- Bump resource requests for your target pod size.

See [ADR-0022](../../adr/ADR-0022-kubernetes-deployment.md) for the
design decisions behind each resource.

## 4. Apply

```sh
kubectl create namespace zhilicon
kubectl apply -f /tmp/my-deployment/
```

Wait for readiness:

```sh
kubectl -n zhilicon rollout status deployment/zhilicon-serving
```

Sanity-check the endpoints:

```sh
kubectl -n zhilicon port-forward svc/zhilicon-serving 8080:8080 &
curl -s localhost:8080/healthz
curl -s localhost:8080/readyz
curl -s localhost:8080/v1/models | jq .
```

## 5. Wire up observability

If you use the Prometheus Operator:

```sh
kubectl apply -f platform/serving/observability/alerts/
```

The `ServiceMonitor` already applied in step 4 starts a 30 s scrape.
Within a minute, the `zhilicon_*` metrics appear in Prometheus.

Then import the Grafana dashboard:

```sh
# Grafana → Dashboards → Import → Upload JSON file.
# Or via API:
curl -X POST -H "content-type: application/json" \
    -d @platform/serving/observability/dashboards/zhilicon-serving.json \
    http://your-grafana/api/dashboards/db
```

The dashboard ([ADR-0021](../../adr/ADR-0021-in-process-prometheus-metrics.md))
has six panels: request rate, latency P50/P99, tokens/sec, error
rate, generation latency, and backend kind.

## 6. Expose via ingress

Every sovereign cluster fronts its services differently. At a
minimum, every ingress route to the serving layer should:

- Terminate TLS at the edge (cluster-managed cert).
- Enforce authentication (mTLS, JWT bearer, or whatever your zone
  dictates).
- Set reasonable request size limits (1 MB is plenty for chat
  completions; SSE responses are unbounded but byte-rate-limited by
  the model).
- Forward the `Accept` header so clients that set
  `Accept: text/event-stream` get streaming.

Reference Istio `VirtualService` and Gateway API `HTTPRoute`
templates live under
`platform/serving/ingress-examples/` *(coming soon — ADR pending)*.

## 7. Verify end-to-end

```sh
# From an authorised client inside the cluster:
curl -X POST https://your-zone.zhilicon.example/v1/chat/completions \
    -H "authorization: Bearer $TOKEN" \
    -H "content-type: application/json" \
    -d '{"messages":[{"role":"user","content":"hello"}],"max_tokens":20}'
```

You should get an OpenAI-shaped JSON response. Backend kind should
read `native` in Grafana (emulation means the image was built
without silicon support — see ADR-0013 and ADR-0021 on that guard).

## 8. Ship a runbook

Every deployment needs a runbook for the alerts we ship
([ADR-0021](../../adr/ADR-0021-in-process-prometheus-metrics.md)):

- `ZhiliconServingHighErrorRate`
- `ZhiliconServingHighLatency`
- `ZhiliconServingNoTokens`
- `ZhiliconServingEmulationBackend`
- `ZhiliconServingPodDown`

Runbook templates live under `docs/src/operations/runbooks/`. Fork
them per zone; the `runbook_url` annotation in the alert YAML
already points at the conventional location.

## Failure modes, triaged

| Symptom                                      | Most likely cause                                         | First check                                        |
| -------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------- |
| Pods CrashLoopBackOff                        | Model checkpoint vocab mismatch with ByteTokenizer        | `kubectl logs -n zhilicon <pod>`                   |
| /readyz returns 503                          | Forward-pass failure during startup probe                 | Same — logs carry the ValueError                   |
| All requests 5xx                             | Kernel extension import failure                           | `kubectl exec ... -- python -m zhilicon doctor`    |
| `backend_kind=emulation` in prod             | Image built without silicon toolchain                     | Rebuild with the native kernel build variant       |
| Prometheus scrape returns empty              | ServiceMonitor label mismatch with your Prometheus CR     | Verify `release:` label on the ServiceMonitor      |
| Tokens counter stuck at 0                    | Chat requests routing but model short-circuits            | Check logs for `ValueError`; check `max_tokens`    |

## Upgrade / rollback

`deployment.yaml` uses rolling updates with `maxSurge: 1`,
`maxUnavailable: 0`. Upgrades are zero-downtime as long as at least
two replicas are healthy. Rollback via
`kubectl rollout undo deployment/zhilicon-serving -n zhilicon` takes
~30 seconds.

## References

- [ADR-0019: Serving layer](../../adr/ADR-0019-serving-layer.md)
- [ADR-0020: Chat completions API](../../adr/ADR-0020-chat-completions-api.md)
- [ADR-0021: In-process Prometheus metrics](../../adr/ADR-0021-in-process-prometheus-metrics.md)
- [ADR-0022: Kubernetes deployment](../../adr/ADR-0022-kubernetes-deployment.md)
- [`platform/serving/k8s/`](../../../platform/serving/k8s/)
- [`platform/serving/observability/`](../../../platform/serving/observability/)
- [`src/sdk/python/zhilicon/serving/README.md`](../../../src/sdk/python/zhilicon/serving/README.md)
- [`demo/chat-ui/`](../../../demo/chat-ui/)
