# How-to: author a ZhiliconWorkload

A `ZhiliconWorkload` is the Kubernetes resource your applications submit
to schedule attested work on Zhilicon silicon. It is a namespaced CRD with
a Pod template and a reference to a cluster-scoped `DevicePool`.

## Minimal example

```yaml
apiVersion: platform.zhilicon.io/v1alpha1
kind: ZhiliconWorkload
metadata:
  name: my-workload
  namespace: my-app
spec:
  poolRef:
    name: sentinel-uae     # cluster-scoped DevicePool
  devices: 2               # whole-device granularity
  attestationRequired: true
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: app
          image: my-registry/my-app:1.0.0
          command: ["/app/serve"]
```

## What the operator does for you

When you submit this, the operator:

1. Resolves `sentinel-uae` → its `SovereignZone`.
2. Validates the pool is `Eligible` (has ≥ minDevices healthy).
3. Waits for an attestation proof (if `attestationRequired: true`).
4. Creates a managed Pod named `<workload-name>-zhilicon` in your namespace.
5. Injects four environment variables into every container:

   - `ZHILICON_SOVEREIGN_ZONE` — the bound zone name
   - `ZHILICON_JURISDICTION` — ISO code
   - `ZHILICON_ATTESTATION_ENDPOINT` — the zone's attestation service URL
   - `ZHILICON_ATTESTATION_PROOF_REF` — the issued proof reference

6. Injects `zhilicon.io/device: <devices>` requests / limits into every
   container so the kubelet's scheduler places the Pod on a node with the
   required devices.
7. Sets a node selector so only the bound chip family is eligible.

## Best practices

- **Use `restartPolicy: Never`** for batch workloads; the operator's phase
  machine assumes a non-restarting Pod.
- **Set `maxRuntime`** for long-running inference / training; the operator
  force-terminates once the deadline elapses.
- **Set `priority`** to tune scheduler ordering when multiple workloads
  compete for the same pool.
- **Do not hand-set `zhilicon.io/device` in your container spec** — the
  operator injects it. Duplicate injections would be rejected by the API
  server.
- **Read the sovereign env vars** inside your app and let the Zhilicon SDK
  pick them up automatically — do not pass zone tags as CLI arguments.

## Failure modes

See [Troubleshooting](../../../demo/sovereign-inference/troubleshooting.md).
