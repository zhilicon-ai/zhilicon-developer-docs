---
adr-id: ADR-0024
title: 'Sovereign-zone enforcement + attestation receipts in the serving layer'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/security-coe", "@zhilicon/platform-team", "@zhilicon/ceo-office"]
---

# ADR-0024: Sovereign-zone enforcement + attestation receipts in the serving layer

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Security COE, Platform team, CEO Office

## Context

The competitive-position matrix ([`docs/strategy/COMPETITIVE_POSITION_MATRIX.md`](../strategy/COMPETITIVE_POSITION_MATRIX.md))
identified "sovereign attestation at silicon level" as the single
structural axis where Zhilicon beats every incumbent (NVIDIA, AMD,
Apple, Intel, Cerebras, Qualcomm, Samsung). The strategic synthesis
memo called this out as the most important differentiator to make
*demonstrable before silicon arrives*.

Today the claim was supported by ADRs ([ADR-0009](ADR-0009-attestation-service-per-jurisdiction.md))
and matrix rows, but the live stack had no visible enforcement
surface. A sovereign prospect running the serving layer could not
*see* the sovereign feature. That is the worst possible state for a
category-defining claim — the claim is technically true but not
tangibly demonstrable.

Two decisions forced:

1. **Software-first implementation or silicon-gated?** Silicon-rooted
   attestation is where the story fully lands (Sentinel-1 TEE +
   attestation service). But silicon is 12–18 months out. Waiting
   ignores the strategic instruction to use the SDK as a pre-silicon
   sales lever.
2. **What's the contract between today's software attestation and
   tomorrow's silicon attestation?** Unless we design the v1 API to
   survive the v2 silicon transition, we get a painful migration.

## Decision

Ship a software-first sovereign-zone enforcement + attestation
receipt subsystem in the serving layer **now**, with an HTTP/header
surface that survives unchanged into the v2 silicon-rooted world.

### What ships in v1 (today)

`zhilicon.serving.sovereign` module providing:

- **`SovereignZone`** — dataclass carrying `zone_id`,
  `data_classification`, signing key, deployment epoch, and a
  `ZonePolicy` describing what the zone enforces.
- **`ZonePolicy`** — conservative defaults: require the
  `X-Zhilicon-Zone` request header, reject mismatches with HTTP 403,
  attach attestation receipts to every success response. Exemptions
  for named User-Agent prefixes (e.g. `Prometheus/`).
- **Zone-enforcement middleware** — attached via
  `create_app(..., sovereign_zone=SovereignZone(...))`. Zero effect
  when the kwarg is omitted — existing deployments work unchanged.
- **Attestation receipts** — HMAC-SHA256 signed receipts attached to
  every success response via `X-Zhilicon-Attestation-*` headers:
  `Receipt-Id`, `Zone`, `Classification`, `Model-Id`, `Backend-Kind`,
  `Root`, `Timestamp`, `Signature`.
- **`/v1/sovereign/attestation`** probe endpoint — returns the pod's
  zone + a fresh probe receipt + the policy state. Clients call this
  before routing real traffic to verify the pod's identity.
- **`verify_receipt(receipt, signing_key=...)`** — constant-time
  HMAC verification. Clients can log-and-verify off-line.
- **Prometheus metric** — `zhilicon_sovereign_zone_requests_total`
  labelled by `{zone, outcome}`, outcome in `{match, mismatch,
  missing, exempt}`. Lets dashboards alert on mismatch spikes
  (possible misrouting) or missing-header traffic (possible
  unauthenticated clients).

### What comes in v2 (with silicon, Sentinel-1 + Discovery-1)

Same HTTP surface, same header names, same receipt schema. What
changes is the *signing root*:

- Receipt signatures flip from HMAC-SHA256 to ECDSA-P256.
- Signing key is held inside the Sentinel-1 TEE; never leaves the
  enclave.
- Attestation root becomes a measured-boot quote (same shape as AWS
  Nitro / NVIDIA CC quotes), verifiable against Zhilicon's
  attestation service PKI.
- Backend kind flips from `emulation` to `native` — which flows
  through to the attestation root hash (so a silicon-signed receipt
  has a *different* root than an emulation-signed receipt, even for
  the same zone × model combination). Clients pin the root; the
  pin is expected to change on the silicon upgrade.

The v1→v2 migration is therefore a *zero-HTTP-change* event. A
client that verifies v1 receipts with HMAC gains one code path for
ECDSA; both coexist during the transition. The strategic value: we
can tell a sovereign prospect *today* "the integration you build
against our v1 software attestation is the integration you will run
against v2 silicon attestation" — with truth.

### What is NOT in scope for this ADR

- **Authentication / identity.** Zone enforcement is an orthogonal
  concern to authentication. The `X-Zhilicon-Zone` header is
  *advisory* in v1 — it must match the pod's zone, but we do not
  *prove* the client belongs in that zone. Authentication belongs at
  the ingress (mTLS, JWT). A future ADR may fold auth into the zone
  claim (zone-bound JWT).
- **Fleet-wide attestation root management.** The deployment epoch
  is a per-pod timestamp in v1. v2 adds a fleet-wide attestation
  service issuing per-pod quotes; separate ADR.
- **Multi-tenant zone routing.** One pod serves one zone. Multi-zone
  routing is an ingress concern ([ADR-0019](ADR-0019-serving-layer.md)).

## Consequences

### Positive

- **The sovereign-attestation competitive claim is demonstrable
  today.** A prospect running `curl -H "X-Zhilicon-Zone: ae"`
  against a Zhilicon serving pod sees `X-Zhilicon-Attestation-*`
  headers in every response. A prospect running `curl` without the
  header sees a 403 with `reason_code: zone_missing`. That is a
  *visible product feature*, not a promise.
- **The v1→v2 transition is a signing-root swap, not an API
  rewrite.** Integrations built today are silicon-ready. This is
  the strategic feature — the SDK-ahead-of-silicon anomaly is
  protected.
- **Operators get dashboardable visibility.** The
  `zhilicon_sovereign_zone_requests_total` counter gives a per-zone
  traffic breakdown and exposes mis-routing. Zero rate on `match`
  with non-zero on `mismatch` is a paging-level anomaly.
- **No breaking change to existing deployments.** The
  `sovereign_zone` kwarg on `create_app` defaults to `None` → no
  middleware registered → existing behaviour preserved.

### Negative / Risks

- **HMAC signing key is a shared secret.** In v1 the key is
  per-pod, ephemeral (generated at startup if not provided), and
  disclosed to authorised clients out-of-band for verification. An
  attacker who compromises the pod can forge receipts — but so can
  they via direct inference. The security model is "trust the pod,
  verify the pod identity via the attestation service." v2 fixes
  this properly by moving the key into the TEE.
- **Receipt counter is monotonic per-process.** A pod restart
  resets to 0. The combination of (receipt_id, deployment_epoch)
  remains globally unique, but a naive dedup cache that keys on
  receipt_id alone would incorrectly match across restarts. The
  header carries both; clients MUST key their dedup on the combined
  tuple.
- **The zone header is advisory, not authenticated.** A malicious
  client can set the header to match the pod and bypass the missing-
  header reject. This is deliberate — zone enforcement is not
  authentication. The ingress layer must authenticate the client
  independently.

### Neutral

- **Probe endpoint leaks pod identity.** `/v1/sovereign/attestation`
  intentionally reveals zone ID, model ID, backend kind, and
  policy. This is the whole point — clients use it for discovery.
  In high-security deployments the ingress may add an
  authentication requirement to the probe path; that's an ingress
  config, not a serving-layer concern.

## Alternatives considered

### Option A: Wait for silicon to ship any attestation feature

Rejected. Strategic priority is "demonstrate sovereign attestation
pre-silicon." Waiting concedes the narrative to Cerebras / G42 /
Qualcomm Cloud AI which could all announce software-level sovereign
features faster.

### Option B: Ship silicon-rooted attestation only (no v1 software path)

Rejected. Same timing objection as Option A — silicon is 12–18
months out. Also: engineering the v2 path first means no validation
of the header schema before silicon lands. Building v1 in software
lets us iterate on the HTTP surface with real customers, so v2 lands
with a proven API.

### Option C: Zone-bound JWT instead of a header

Attractive — integrates auth with zone claim. Rejected for v1 because
it pre-supposes an auth ingress decision that not every sovereign
buyer has made yet. The header + probe endpoint is the simplest
possible surface; JWT layering is a v2 ADR when we see the auth
distribution in the field.

### Option D: Roll the receipt into the HTTP body instead of headers

Rejected. Response bodies are schema-owned by each endpoint (chat
completions has OpenAI's shape; generate has our own). Adding a
receipt field would break clients that parse the OpenAI schema
strictly. Headers are the correct channel — they do not affect body
parsing and they are naturally opaquely forwardable through reverse
proxies.

## References

- [ADR-0009: Attestation service per jurisdiction](ADR-0009-attestation-service-per-jurisdiction.md)
- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md) — source of the `backend_kind` distinction in receipts
- [ADR-0019: Serving layer](ADR-0019-serving-layer.md)
- [ADR-0020: Chat completions API](ADR-0020-chat-completions-api.md)
- [ADR-0021: In-process Prometheus metrics](ADR-0021-in-process-prometheus-metrics.md)
- [`docs/strategy/COMPETITIVE_POSITION_MATRIX.md`](../strategy/COMPETITIVE_POSITION_MATRIX.md)
- [`docs/strategy/STRATEGIC_SYNTHESIS.md`](../strategy/STRATEGIC_SYNTHESIS.md)
- [`src/sdk/python/zhilicon/serving/sovereign.py`](../../src/sdk/python/zhilicon/serving/sovereign.py)
- [`src/tests/sdk/python/test_serving_sovereign.py`](../../src/tests/sdk/python/test_serving_sovereign.py)
- Batch-38 session history.
