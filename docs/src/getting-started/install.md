# Install

This page walks you from zero to a working Zhilicon platform installation
on a local Kubernetes cluster. If you prefer a pre-assembled end-to-end
demo, jump straight to [First workload](first-workload.md) — it calls the
same installer this page uses.

## Prerequisites

| Tool      | Min     | Check                      |
| --------- | ------- | -------------------------- |
| Docker    | 24.x    | `docker version`           |
| kubectl   | 1.28    | `kubectl version --client` |
| kind      | 0.22    | `kind version`             |
| Helm      | 3.15    | `helm version --short`     |
| Python    | 3.11    | `python --version`         |
| jq        | 1.6+    | `jq --version`             |

## 1. Create a kind cluster

```bash
cat > /tmp/zhilicon-kind.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
    labels:
      zhilicon.io/chip-family: sentinel-1
      zhilicon.io/region: ae
  - role: worker
    labels:
      zhilicon.io/chip-family: sentinel-1
      zhilicon.io/region: ae
EOF

kind create cluster --name zhilicon --config /tmp/zhilicon-kind.yaml
```

## 2. Seed simulated devices

In a real deployment, the device plugin discovers devices through sysfs.
On kind, we use its `static` backend and a seed JSON file on each worker:

```bash
for node in $(kind get nodes --name zhilicon | grep worker); do
  docker exec "$node" bash -c 'mkdir -p /etc/zhilicon && cat > /etc/zhilicon/devices.json' <<'DEV_EOF'
  { "devices": [
      { "ID": "zhi0", "ChipFamily": "sentinel-1", "NUMANode": 0, "Healthy": true },
      { "ID": "zhi1", "ChipFamily": "sentinel-1", "NUMANode": 0, "Healthy": true },
      { "ID": "zhi2", "ChipFamily": "sentinel-1", "NUMANode": 1, "Healthy": true },
      { "ID": "zhi3", "ChipFamily": "sentinel-1", "NUMANode": 1, "Healthy": true }
  ] }
  DEV_EOF
done
```

## 3. Install the chart

```bash
bash demo/sovereign-inference/02-install-operator.sh
```

The installer builds every image, loads it into kind, creates the
attestation Secrets (demo keys — not for production), and runs
`helm upgrade --install` with the `static` discovery backend and
attestation enabled.

## 4. Verify

```bash
kubectl -n zhilicon-system get pods
kubectl get crd | grep platform.zhilicon.io
kubectl describe node <worker-name> | grep zhilicon.io/device
```

You should see the operator + device-plugin pods Running, the three CRDs
registered, and each worker advertising `zhilicon.io/device: 4`.

## Next

Install `zhctl` and submit your first workload — see
[First workload](first-workload.md).
