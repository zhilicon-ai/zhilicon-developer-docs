# System overview

Zhilicon's architecture is a **vertical stack** across four tiers:

```text
  ┌──────────────────────────────────────────────────────────────┐
  │  Tier 4 — Platform                                           │
  │  Kubernetes operator · device plugin · attestation service   │
  │  fleet CLI · OTel / Prometheus / Grafana                     │
  └────────────────────────────┬─────────────────────────────────┘
                               │
  ┌────────────────────────────▼─────────────────────────────────┐
  │  Tier 3 — SDK + Runtime                                      │
  │  Python (kernels, domain APIs) · C++ runtime · operator      │
  │  registry · scheduler · autotune cache · HAL                 │
  └────────────────────────────┬─────────────────────────────────┘
                               │
  ┌────────────────────────────▼─────────────────────────────────┐
  │  Tier 2 — Firmware + Edge Runtime                            │
  │  Boot ROM · BMC · drivers (HBM, ZLink, PCIe, SpaceFibre)     │
  │  Zhilicon edge runtime (Horizon-1 on-chip)                   │
  └────────────────────────────┬─────────────────────────────────┘
                               │
  ┌────────────────────────────▼─────────────────────────────────┐
  │  Tier 1 — Silicon                                            │
  │  5 chip programs · shared-IP library · formal property lib   │
  │  RTL + physical design + packaging                           │
  └──────────────────────────────────────────────────────────────┘
```

Each tier is documented separately:

- [Platform tier](platform-tier.md) — fleet management, attestation, telemetry.
- [Edge runtime](edge-runtime.md) — Horizon-1 on-chip sovereign runtime.
- [SDK stack](sdk-stack.md) — how applications talk to silicon.
- [Sovereign-by-construction](sovereign.md) — the design invariants that
  make the stack sovereign, not just sovereign-ready.

## Cross-cutting principles

1. **Sovereign by construction.** Every operation on attested resources
   walks through the sovereign-context gate — at the Kubernetes tier, at the
   SDK tier, and at the on-chip runtime tier.
2. **Attestable end-to-end.** A single proof chain binds device identity →
   workload identity → zone policy → freshness nonce.
3. **Observable by default.** Every platform component and the edge runtime
   emit OpenTelemetry-compatible traces and metrics; the reference Grafana
   dashboard ships in the Helm chart.
4. **Shared rigor.** The same shared-IP RTL library, the same formal
   property methodology, and the same verification sign-off gates apply to
   every chip program.
