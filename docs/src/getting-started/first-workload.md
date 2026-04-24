# Your first sovereign workload

With the platform [installed](install.md), submit your first
sovereign-gated workload. We'll use the UAE-fintech scenario from the
demo — a `SovereignZone` with jurisdiction `ae`, a `DevicePool` of
Sentinel-1 devices bound to it, and a `ZhiliconWorkload` that runs a
batch ECDSA-signing task.

## 1. Create the zone and pool

```bash
kubectl apply -f demo/sovereign-inference/manifests/namespace.yaml
kubectl apply -f demo/sovereign-inference/manifests/sovereignzone.yaml
kubectl apply -f demo/sovereign-inference/manifests/devicepool.yaml
```

Wait for `SpecValid` and `Eligible` to flip to True:

```bash
kubectl get sovereignzone uae-fintech \
  -o jsonpath='{.status.conditions[?(@.type=="SpecValid")].status}' ; echo
kubectl get devicepool sentinel-uae \
  -o jsonpath='{.status.conditions[?(@.type=="Eligible")].status}' ; echo
```

Both should print `True` within a minute.

## 2. Submit the workload

```bash
kubectl apply -f demo/sovereign-inference/manifests/workload.yaml
kubectl -n bank-a get zhiliconworkload -w
```

Expect the phase to progress `Pending → Attesting → Running → Succeeded`.

## 3. Watch it run

```bash
pod=$(kubectl -n bank-a get pod -l zhilicon.io/workload=ecdsa-batch-0001 \
       -o jsonpath='{.items[0].metadata.name}')
kubectl -n bank-a logs -f "$pod"
```

You should see the sovereign environment variables printed, followed by
the ECDSA batch completing in a few seconds.

## 4. Exercise the sovereign gate

The pod environment carries the attested zone tag. Try running a shell
in the pod and exporting it:

```bash
kubectl -n bank-a exec -it "$pod" -- env | grep ZHILICON
```

Any Python or C++ application running in this pod and calling the SDK
will route through that tag; requests to devices outside the zone are
refused by the runtime.

## See also

- [zhctl in 5 minutes](cli.md)
- [Deploy a SovereignZone](../how-to/deploy-sovereign-zone.md) — the how-to
  for production zone setup (HSM-backed keys, real jurisdictions, TLS).
