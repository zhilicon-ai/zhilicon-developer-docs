# `zhctl` in 5 minutes

`zhctl` is the fleet-management CLI. Source lives under
[`platform/fleet-cli/`](../../../platform/fleet-cli/).

## Install

```bash
pip install --editable 'platform/fleet-cli[dev]'
zhctl --version
```

## Command groups

```bash
zhctl device     # inspect individual devices
zhctl pool       # manage DevicePools
zhctl zone       # manage SovereignZones
zhctl workload   # submit and monitor ZhiliconWorkloads
zhctl attest     # request + verify attestation proofs
```

## Everyday examples

```bash
# Inspect every Sentinel-1 device visible to the cluster
zhctl device list --chip sentinel-1

# Show every DevicePool and which zone it binds to
zhctl pool list

# List all SovereignZones with pool and workload counts
zhctl zone list

# Submit a workload from a manifest
zhctl workload submit demo/sovereign-inference/manifests/workload.yaml

# Tail logs for a running workload
zhctl workload logs ecdsa-batch-0001 -n bank-a --follow

# Fetch the zone descriptor from the attestation service
zhctl --attestation-endpoint https://zhilicon-zhilicon-operator-attestation.zhilicon-system.svc.cluster.local:8443 \
      attest zone-info
```

## Configuration

| Flag                     | Env                          | Purpose                                |
| ------------------------ | ---------------------------- | -------------------------------------- |
| `--context`              | `ZHCTL_CONTEXT`              | Kubernetes context to target           |
| `--attestation-endpoint` | `ZHCTL_ATTESTATION_ENDPOINT` | Attestation service URL for the zone   |

## Full reference

See [CLI reference](../reference/cli.md) for every subcommand.
