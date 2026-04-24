# Attestation REST API

The attestation service exposes four HTTP endpoints. The authoritative
OpenAPI-style description lives in
[`platform/attestation/README.md`](../../../platform/attestation/README.md);
the Rust source of the handlers is in
[`platform/attestation/src/routes.rs`](../../../platform/attestation/src/routes.rs).

## Base URL

Per-jurisdiction. Example for the UAE zone: `https://attest.uae.zhilicon.cloud`.

## Endpoints

### `GET /healthz`

Liveness. Returns `200 ok\n` when the process is alive and the signing key
has loaded successfully.

### `GET /v1/zone`

Returns the zone descriptor — jurisdiction, policy SHA-256, authority
public key (base64 Ed25519).

```json
{
  "jurisdiction": "ae",
  "policy_sha256": "8f1a…b3",
  "authority_pubkey_b64": "…"
}
```

### `POST /v1/attest`

Issue a signed attestation proof.

**Request**

```json
{
  "device": {
    "chip_family": "sentinel-1",
    "die_id": "0x1a2b3c4d",
    "firmware_hash": "…sha256…",
    "device_pubkey_b64": "…"
  },
  "workload": {
    "pod_uid": "8c4f…",
    "image_digest": "sha256:…",
    "command_sha256": "7e9…",
    "namespace": "bank-a"
  },
  "nonce": "…",
  "validity_seconds": 3600
}
```

**Response**

```json
{
  "proof": {
    "proof_id": "…",
    "issued_at": "2026-04-17T16:45:00Z",
    "expires_at": "2026-04-17T17:45:00Z",
    "nonce": "…",
    "device": { … },
    "workload": { … },
    "zone": { "jurisdiction": "ae", "policy_sha256": "…", "authority_pubkey_b64": "…" },
    "signature_b64": "…"
  },
  "fingerprint": "…"
}
```

### `POST /v1/verify`

Verify a proof against the service's zone and expiry.

**Request**: `{ "proof": <AttestationProof> }`
**Response**: `{ "ok": true, "fingerprint": "…" }` or an error with
`zone_mismatch`, `expired`, `not_yet_valid`, or `bad_signature`.

## Wire format

Proofs are serialised as JSON. The canonical byte sequence used for
signing is the JSON encoding of every field **except** `signature_b64`, in
the struct's declared order. See
[`platform/attestation/src/attestation.rs`](../../../platform/attestation/src/attestation.rs)
for the exact layout.

## Security notes

- Signing keys are Ed25519, 32-byte seed material.
- Proof validity is capped by `default_validity_seconds` (max 7 days).
- A zone-mismatch between the proof and the verifying service is a
  `400 Bad Request` with `zone_mismatch` — never silently accepted.
- The public key on the proof lets an offline verifier re-check
  signatures without calling the service.
