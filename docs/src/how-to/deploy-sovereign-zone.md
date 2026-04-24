# How-to: deploy a production SovereignZone

```admonish danger title="Production vs. demo"
The [demo](../getting-started/install.md) uses an all-zeros Ed25519 key
stored in a Kubernetes Secret. **Do not copy this into production.** The
steps below show the production path — HSM-backed keys, real jurisdictions,
TLS, and per-zone attestation service deployment.
```

## Goal

Deploy a SovereignZone bound to a jurisdiction-specific attestation
service with HSM-backed keys, such that:

- No workload in the zone can run without a fresh proof.
- The zone's signing key never leaves the HSM.
- Zone policy, authority public key, and jurisdiction tag are all audit-logged.

## Step 1 — Provision the zone key in an HSM

Use the CA hierarchy defined in
[`programs/portfolio-upgrade/RELIABILITY_SECURITY_SUPPLY_CHAIN.md`](../../../programs/portfolio-upgrade/RELIABILITY_SECURITY_SUPPLY_CHAIN.md) §2.

The custodian for the zone's jurisdiction issues:

- An Ed25519 signing key stored in a FIPS 140-2 L3 HSM.
- The matching authority public key (non-secret).
- A policy manifest SHA-256.

## Step 2 — Deploy the attestation service in-zone

```bash
helm install zhilicon-attestation-ae ./platform/charts/zhilicon-operator \
  --namespace zhilicon-system \
  --create-namespace \
  --set operator.enabled=false \
  --set devicePlugin.enabled=false \
  --set attestation.enabled=true \
  --set attestation.jurisdiction=ae \
  --set attestation.keyCustodian=uae-cba-ca \
  --set attestation.policySha256="<SHA256 from step 1>" \
  --set attestation.signingKeySecretRef=hsm-signing-secret \
  --set attestation.authorityPubkeySecretRef=authority-pubkey-secret
```

Note: `hsm-signing-secret` must be a Kubernetes Secret that routes to an
HSM-backed store (e.g. Vault / Azure Key Vault) rather than raw key bytes.

## Step 3 — Declare the SovereignZone

```yaml
apiVersion: platform.zhilicon.io/v1alpha1
kind: SovereignZone
metadata:
  name: uae-fintech
spec:
  jurisdiction: ae
  encryption:
    - aes-256-gcm
    - kyber-1024-hybrid
  attestationEndpoint: https://attest.uae.zhilicon.cloud
  keyCustodian: uae-cba-ca
  dataResidencyPolicy: fintech-ae-cbuae
  allowedChipFamilies:
    - sentinel-1
```

The `attestationEndpoint` must use HTTPS — the reconciler rejects `http://`.

## Step 4 — Verify the zone is healthy

```bash
kubectl get sovereignzone uae-fintech \
  -o jsonpath='{.status.conditions[?(@.type=="AttestationEndpointReachable")]}'
```

Expect `status: True, reason: ProbeSucceeded`. If `False`, check the
attestation service's own logs first, then the operator logs for
`AttestationEndpointReachable` context.

## Step 5 — Bind a DevicePool and ZhiliconWorkload

Identical to the [first-workload walkthrough](../getting-started/first-workload.md).
The only change is the `sovereignZoneRef` on the DevicePool points at this
zone's name.
