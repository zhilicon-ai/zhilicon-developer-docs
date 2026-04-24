# Runbooks

Source: [`platform/runbooks/`](../../../platform/runbooks/) (Markdown,
one file per recurring event).

## Available runbooks

| Trigger                                   | Runbook                                                                           |
| ----------------------------------------- | --------------------------------------------------------------------------------- |
| `ZhiliconDeviceAttestationFailing` alert  | [`attestation-failing.md`](../../../platform/runbooks/attestation-failing.md)      |

## Runbook format

Every runbook uses the same skeleton:

1. **Context** — what triggered the alert, blast radius.
2. **First checks (first 5 minutes)** — minimum diagnostic steps; always
   end with a `zhctl` / `kubectl describe` command that captures state for
   a bug filing.
3. **Likely causes** — short table of symptom → root cause → owner.
4. **Remediation** — specific commands to run per cause.
5. **Postmortem** — when to open one and what the template looks like.

## Adding a runbook

When a new alert rule lands in
[`platform/telemetry/prometheus-rules.yaml`](../../../platform/telemetry/prometheus-rules.yaml),
every `severity: critical` rule must ship a matching runbook in the same
PR. The PR template checklist enforces this.
