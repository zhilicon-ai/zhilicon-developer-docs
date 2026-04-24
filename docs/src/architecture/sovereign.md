# Sovereign by construction

"Sovereign-ready" means the software can be configured to respect a
jurisdiction. **Sovereign by construction** means the software *refuses* to
operate outside it. Zhilicon is the latter.

## Invariants

```admonish important
These invariants hold across every layer. Each one is enforced by a
specific component — if any component were absent, the invariant would fail.
```

1. **No attested operation proceeds without a current, verified proof.**
   - Platform: `ZhiliconWorkloadReconciler` blocks scheduling until
     `AttestationProofRef` is populated.
   - Edge runtime: [`sovereign_context::require`](../../../src/edge-runtime/src/sovereign_context.rs)
     gates every attested call.
   - SDK runtime: `DispatchContext.sovereign_zone` flows through every
     kernel launch; the HAL refuses a mismatch.

2. **A device is bound to exactly one sovereign zone at a time.**
   - Platform: `DevicePool.spec.sovereignZoneRef` is required; the zone
     cannot change after pool creation without deletion.
   - Edge runtime: the sovereign context is seal-once, rotate-only — no
     zone migration without a power cycle.

3. **Zone policy and attestation-authority key live in-jurisdiction.**
   - Attestation service runs per-zone; its signing key never leaves the
     service process.
   - Key provisioning chain documented in
     [`programs/portfolio-upgrade/RELIABILITY_SECURITY_SUPPLY_CHAIN.md`](../../../programs/portfolio-upgrade/RELIABILITY_SECURITY_SUPPLY_CHAIN.md) §2.

4. **Attestation proofs are short-lived and bound to a nonce.**
   - Default validity ≤ 1 hour; service-enforced maximum is 7 days.
   - Nonce is device-generated fresh randomness.

5. **Every sovereign-relevant event is observable.**
   - Operator emits metrics per reconciliation; telemetry pipeline ships in
     the Helm chart.
   - Edge runtime emits fixed-size records into a lock-free ring drained by
     the SpaceFibre telemetry task.

## What this buys you

- **Regulatory credibility.** Regulators can audit that data physically
  cannot leave the jurisdiction because the silicon refuses to compute
  against it.
- **Multi-tenancy safety.** Two zones on the same device cluster cannot
  interleave — the scheduler supports a `kSovereignIsolation` policy that
  refuses to schedule across zones on the same partition.
- **Incident blast-radius control.** If an attestation signing key is
  compromised, only workloads in that zone are affected; the rest of the
  fleet keeps running.

## See also

- [Platform tier](platform-tier.md)
- [Attestation REST API](../reference/rest-apis.md)
- [Runbook: attestation failing](../../../platform/runbooks/attestation-failing.md)
