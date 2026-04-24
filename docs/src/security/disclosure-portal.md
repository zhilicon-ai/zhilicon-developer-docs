# Disclosure portal

The public disclosure portal is at
<https://zhilicon.com/security/disclosure>. This page is the operational
specification for what that portal exposes, the data it collects, and
how the submission is routed inside Zhilicon.

## What the portal is

A single HTML form + a signed webhook into the Zhilicon Product Security
Incident Response Team (PSIRT) ticketing queue. The portal:

- Accepts a disclosure **without requiring an account** — reporters
  never hand Zhilicon credentials before a finding is triaged.
- Offers **end-to-end encryption** by pre-fetching the current PGP
  public key (see [PGP key management](pgp-key-management.md)) and
  encrypting the submission in-browser before POST.
- Issues a **tracking ID** synchronously on submit. The reporter keeps
  the ID to reference their report in any follow-up.
- Never publishes any submitted content — the form is write-only from
  the outside world.

## Data collected

The form has six required fields and three optional ones. Every required
field maps 1:1 to a field in the RFC 9116 spirit and to the internal
PSIRT ticket schema.

| Field                  | Required | Purpose                                               |
| ---------------------- | -------- | ----------------------------------------------------- |
| `component`            | ✓        | SDK / firmware / RTL / compiler / toolchain / platform |
| `chip_program`         | ✓        | discovery-1 / horizon-1 / nexus-1 / sentinel-1 / prometheus / shared |
| `severity_estimate`    | ✓        | critical / high / medium / low                        |
| `reproduction`         | ✓        | Free-form; encrypted-in-browser before POST            |
| `expected_vs_observed` | ✓        | Two free-form fields, same encryption                  |
| `cvss_vector_v31`      |          | Optional; PSIRT re-scores on receipt                   |
| `cve_assignment`       |          | "Not assigned" / "Assigned: CVE-YYYY-NNNNN"            |
| `contact`              | ✓        | Email or Signal handle for PSIRT to reach back         |
| `safe_harbor_opt_in`   | ✓        | Mandatory acknowledgement of the safe-harbor policy    |

For hardware-specific findings, four optional sub-forms extend the
schema:

- **Side-channel** — channel, leakage primitive, measurement setup,
  trace count to recover secret.
- **Fault injection** — voltage / clock / laser / EM parameters,
  success rate across attacked population.
- **Supply-chain** — observed deviation from the provisioning
  chain-of-custody.
- **Key-management / TEE** — attack model (local / privileged /
  physical), primitive, extractive vs. observational.

## Submission flow

```text
  +---------------+         +-----------------+         +------------+
  |  Reporter's   |  HTTPS  |  disclosure     |  HMAC   |  PSIRT     |
  |   browser     | ------> |  portal (CDN)   | ------> |  webhook   |
  +---------------+         +-----------------+         +-----+------+
         |                          ^                         |
         |                          |                         v
     encrypt submission        fetch current        +--------------------+
     against PGP pubkey          PGP pubkey          |  PSIRT tracking    |
                                                    |  queue (per-zone)  |
                                                    +---------+----------+
                                                              |
                                                              v
                                                  Triage (see triage-playbook.md)
```

1. Browser fetches the current PGP public key from
   `https://zhilicon.com/.well-known/security-pgp-key.asc`.
2. Form contents are encrypted in-browser (OpenPGP.js).
3. Ciphertext + metadata (component, chip_program, severity_estimate,
   contact) is POSTed over HTTPS to the portal's CDN endpoint.
4. CDN edge forwards the payload, HMAC-signed with a shared secret, to
   the internal PSIRT webhook.
5. PSIRT webhook decrypts, registers a ticket, returns a tracking ID.
6. Reporter sees the tracking ID in the browser immediately.

## SLAs

See [SECURITY.md](../../../SECURITY.md) for the authoritative list. The
portal displays them on the landing page:

- Acknowledgement within **2 business days**.
- Remediation plan within **10 business days**.
- Coordinated disclosure **90 days** default, extendable by mutual
  agreement for hardware-side or supply-chain issues that need a
  silicon revision.

## Zero-account commitment

Zhilicon will never ask a reporter for an account, an API key, a GitHub
handle, or proof-of-identity before triaging a report. Attribution on
the acknowledgements page is always opt-in and happens only after the
report is closed.

## What the portal does NOT do

- It never *publishes* the submitted content. Publication goes through
  the CVE process and a written advisory (see
  [CVE policy](cve-policy.md)).
- It does not issue bug-bounty payouts directly. Payment routing lives
  in the PSIRT internal workflow; eligible reports receive payment
  information via the contact address on file after resolution.
- It is not a support channel. Support questions get redirected to
  `https://support.zhilicon.com`.

## Accessibility and regulatory

The portal is WCAG 2.1 AA compliant. The encrypted-submission path works
with assistive-technology screen readers — no captcha. GDPR /
PIPL / DIFC-compliant notices and retention policies are published on
the same page.

## Related

- [Safe harbor](safe-harbor.md) — the legal assurance reporters rely on.
- [Triage playbook](triage-playbook.md) — how PSIRT handles a report
  once it lands.
- [PGP key management](pgp-key-management.md) — how the encryption key
  is published and rotated.
- [CVE policy](cve-policy.md) — CNA partner + assignment rules.
