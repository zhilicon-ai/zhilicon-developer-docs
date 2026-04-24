# Incident response

## Severity definitions

| Sev  | Criterion                                                         | Pages      |
| ---- | ----------------------------------------------------------------- | ---------- |
| S1   | Customer-impacting outage or security breach in a sovereign zone   | On-call CSO + VP Engineering + Platform lead |
| S2   | Degraded SLA in a production zone but workloads still running       | On-call Platform + zone owner               |
| S3   | Non-customer-impacting defect observed in staging / canary          | Normal triage                                |

## S1 flow

1. **Acknowledge** the page within 5 minutes.
2. **Declare** the incident in the internal #incidents channel with the
   severity, affected zone(s), and first-observed timestamp.
3. **Contain** — the default containment for a sovereign-zone incident is
   to revoke the zone's signing key and let existing proofs expire. This
   is a CEO-authorised action under
   [`programs/prometheus/governance/INTEL_18A_ESCALATION.md`](../../../programs/prometheus/governance/INTEL_18A_ESCALATION.md)
   (that doc covers Prometheus; the analogous playbook for other chip programs
   lives under each program's `governance/` directory).
4. **Diagnose** — run the relevant [runbook](runbooks.md); collect
   diagnostics via `demo/sovereign-inference/scripts/smoke-test.sh
   --collect-diagnostics`.
5. **Remediate** — narrow to the minimum change that restores SLA.
6. **Communicate** — hourly status to customers until resolved.
7. **Postmortem** — within 5 business days. Template below.

## Postmortem template

```markdown
# Incident — <short-title>

**Date:** YYYY-MM-DD
**Severity:** S1 / S2 / S3
**Duration:** <start> → <resolved>
**Blast radius:** <zones / customers / workloads affected>

## Timeline
- HH:MM — event / observation / action

## Root cause
One paragraph — the underlying engineering cause, not the symptom.

## Detection
How long until we knew? What would have caught it sooner?

## Remediation
What we did during the incident to restore service.

## Follow-ups
- [ ] …
- [ ] …

## What went well
- …

## Approval
- Platform lead: <name, date>
- CSO (if S1 security-adjacent): <name, date>
```

## Where postmortems live

Under the affected program's `governance/` directory (e.g.
`programs/prometheus/governance/postmortems/`). Open an issue against the
repo with the `incident` label and link the postmortem.
