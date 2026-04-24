# Security policy

See the authoritative policy: [`SECURITY.md`](../../SECURITY.md).

Key points:

- **Do not open public GitHub issues for security findings.** Use the
  private disclosure channel at `security@zhilicon.com` or the portal at
  `https://zhilicon.com/security/disclosure`.
- Acknowledgement within **2 business days**; remediation plan within
  **10 business days**.
- **90-day coordinated disclosure**, extendable for hardware or supply-chain
  issues that require a silicon revision.
- Hardware-rooted findings (side-channel, fault injection, errata) are
  coordinated with the affected chip program owner and publish an ERRATA
  advisory alongside the fix.

## Hardware-specific reporting

When filing:

- **Side-channel** — channel, leakage primitive, measurement setup, trace
  count to recover secret.
- **Fault injection** — parameters (voltage / clock / laser / EM), success
  rate across the attacked population.
- **Supply-chain** — observed deviation from the published provisioning
  chain-of-custody.
- **Key-management / TEE** — attack model (local / privileged / physical),
  primitive affected, extractive vs. observational.

## Safe harbour

Researchers acting in good faith within the scope defined in
[`SECURITY.md`](../../SECURITY.md) will not face legal action for testing
against publicly released artefacts or reference designs. Testing against
customer deployments requires the customer's explicit consent.
