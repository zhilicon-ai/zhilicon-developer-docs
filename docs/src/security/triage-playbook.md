# PSIRT triage playbook

Internal-facing. Describes how Zhilicon's Product Security Incident
Response Team (PSIRT) handles an incoming disclosure from the moment
the webhook fires to the moment the CVE publishes.

> **Audience:** PSIRT members, Security COE, Chief Security Officer.
> **Audience this is NOT for:** customers or the general public — see
> [Disclosure portal](disclosure-portal.md) and [CVE policy](cve-policy.md).

## Timeline targets

| Hour   | Action                                                                            |
| ------ | --------------------------------------------------------------------------------- |
| T + 0  | Webhook fires; ticket auto-created in the PSIRT queue                             |
| T + 4  | On-call acknowledges; initial severity class assigned (P0 / P1 / P2 / P3)         |
| T + 24 | Report-back to the reporter with tracking ID confirmation + initial assessment    |
| T + 48 | (P0 / P1 only) SME engaged from the affected program; reproduction attempted      |
| T + 5d | Full triage complete; remediation plan drafted                                    |
| T + 10d | Remediation plan communicated to reporter (per SECURITY.md SLA)                  |

## Severity quick-reference

| Class | Criterion                                                                          |
| ----- | ---------------------------------------------------------------------------------- |
| P0    | Remote unauthenticated code execution; cryptographic break of a shipped primitive; silicon-level attack that bypasses the TEE boundary or the attestation service |
| P1    | Remote authenticated RCE; SEU-recovery bypass on Horizon-1; side-channel that recovers keys in < 2^40 traces |
| P2    | Local privilege escalation; denial-of-service against the attestation service; memory corruption in the SDK C++ layer |
| P3    | Information disclosure without cryptographic consequences; hardening opportunities; documentation errors with security implications |

## Stage-by-stage

### 1. Intake (T + 0 → T + 4)

- Webhook receives the encrypted payload from the disclosure portal.
- Decrypt in the PSIRT ticketing environment (see
  [PGP key management](pgp-key-management.md) for key location).
- Normalise the report into the internal schema.
- Auto-assign to the on-call PSIRT engineer for the affected component
  or program.

### 2. Acknowledgement (T + 24)

- On-call sends the reporter a short message: tracking ID, initial
  severity, SLA for the remediation plan, and the primary PSIRT contact
  for this ticket.
- Do **not** ask for more reproduction material unless the submitted
  reproduction is clearly insufficient. First we try to reproduce with
  what's given.

### 3. Reproduction (T + 48 for P0 / P1)

- Stand up the affected artefact version in an isolated lab.
- Walk the reproduction. Capture stdout / stderr / logs / core dumps
  into the ticket.
- If reproduction fails, ask the reporter for additional context before
  escalating severity downward.
- For hardware findings: coordinate lab access with the affected chip
  program owner. Bench reservations go through the Verification COE
  queue.

### 4. Root-cause analysis

- Bisect to the introducing commit (SDK / firmware / platform).
- For silicon findings, bisect to the RTL revision and identify the
  affected post-silicon population (die ID range, package revision).
- Identify the shortest credible fix.
- If the fix requires a silicon revision, flag the advisory timeline
  extension per [CVE policy](cve-policy.md).

### 5. Fix development

- Develop the fix on a private security branch. CODEOWNERS review from
  the affected component team + the Security COE is mandatory.
- **DCO sign-off** on every commit (per CONTRIBUTING.md). The DCO
  workflow runs on the private security PR the same as on public PRs.
- Run the full security-scans CI workflow (`gosec`, `cargo-audit`,
  Trivy, CodeQL) against the security branch before merge.
- Merge to the affected release branch. Do **not** merge to `main`
  until disclosure day.

### 6. CVE assignment

- Reserve the CVE through the CNA partner (see
  [CVE policy](cve-policy.md)).
- Draft the advisory (ZSA-YYYY-NNNN) using the Zhilicon template.
- Legal review on the advisory body — specifically on the blast-radius
  and customer-impact paragraphs.

### 7. Disclosure

- Cut the release containing the fix.
- Publish the advisory at the scheduled window.
- Flip the CVE status from RESERVED to PUBLIC.
- Notify the reporter.
- Optionally credit the reporter on the acknowledgements page.

### 8. Post-mortem

- Write the internal post-mortem within 5 business days of disclosure.
- Template: same as the one in
  [operations / incident response](../operations/incident-response.md#postmortem-template).
- Action items land as tickets in the affected component's backlog,
  tagged `psirt-followup`.

## Communication discipline

- Every customer-visible communication goes through a single named
  PSIRT contact per ticket. Parallel-conversation-tree is how details
  leak.
- For multi-customer impact, use the per-zone operations channel to
  pre-notify affected customers under NDA at least 24 hours before
  public disclosure.
- Never speculate publicly about who reported a finding. The
  acknowledgements page is the only place where reporter identity
  is attached.

## Reference templates

- **Initial acknowledgement email**: internal-only, link omitted.
- **Remediation plan email**: internal-only, link omitted.
- **ZSA advisory template**: internal-only, link omitted.

All three templates live in the PSIRT private wiki; this page is the
public-facing overview of the process, not a substitute for them.

## Related

- [Disclosure portal](disclosure-portal.md)
- [CVE policy](cve-policy.md)
- [Safe harbour](safe-harbor.md)
- [PGP key management](pgp-key-management.md)
- [Incident response](../operations/incident-response.md) — the broader
  incident flow that a PSIRT ticket may escalate into.
