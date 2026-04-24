# `zhctl` CLI reference

Source: [`platform/fleet-cli/`](../../../platform/fleet-cli/).

## `zhctl device`

### `zhctl device list [--chip <family>] [--zone <name>]`

List every Zhilicon device visible to the cluster.

### `zhctl device describe <node-name>`

Detailed JSON dump for a single device-hosting node.

## `zhctl pool`

### `zhctl pool list [--zone <name>]`

### `zhctl pool apply <manifest.yaml>`

Apply a DevicePool manifest.

### `zhctl pool delete <name>`

## `zhctl zone`

### `zhctl zone list`

## `zhctl workload`

### `zhctl workload submit <manifest.yaml>`

### `zhctl workload status <name> [-n <namespace>]`

### `zhctl workload logs <name> [-n <namespace>] [--follow]`

## `zhctl attest`

Requires `--attestation-endpoint` (or `ZHCTL_ATTESTATION_ENDPOINT`).

### `zhctl attest zone-info`

Returns the attestation service's zone descriptor.

### `zhctl attest issue --device-die-id … --device-chip … …`

Request a signed proof. Full flag list in
[`platform/fleet-cli/zhctl/commands/attest.py`](../../../platform/fleet-cli/zhctl/commands/attest.py).

### `zhctl attest verify <proof.json>`

Verify an offline-signed proof.

## Global flags

| Flag                        | Env                          | Purpose                          |
| --------------------------- | ---------------------------- | -------------------------------- |
| `--context`                 | `ZHCTL_CONTEXT`              | Kubernetes context                |
| `--attestation-endpoint`    | `ZHCTL_ATTESTATION_ENDPOINT` | Attestation service URL           |
| `--help / -h`               |                              | Per-group help                    |
| `--version`                 |                              | Print zhctl version               |
