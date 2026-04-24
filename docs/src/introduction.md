# Introduction

Welcome to the Zhilicon developer documentation. This portal is the entry
point for engineers, customer-solution architects, and operators working
with the Zhilicon silicon portfolio and its surrounding platform.

## What Zhilicon is

Zhilicon is a vertically integrated sovereign-AI silicon stack:

- **Five chips** targeting five markets — healthcare, space, 6G RF,
  crypto / fintech, and datacenter AI. See [Portfolio → Chip matrix](portfolio/chip-matrix.md).
- **A platform tier** — Kubernetes operator, device plugin, attestation
  service, fleet CLI, telemetry — that makes a deployment sovereign by
  construction rather than by convention. See
  [Architecture → Platform tier](architecture/platform-tier.md).
- **An on-chip edge runtime** for Horizon-1 rad-hard workloads that
  executes under radiation-induced single-event upsets. See
  [Architecture → Edge runtime](architecture/edge-runtime.md).
- **An SDK** — Python + C++ — that exposes the silicon through a
  declarative compute API, rather than as a CUDA-shaped imperative kernel
  launcher. See [Architecture → SDK stack](architecture/sdk-stack.md).

## Reading paths

| If you are a…                     | Start here                                    |
| --------------------------------- | --------------------------------------------- |
| Customer solutions architect      | [Getting started → Install](getting-started/install.md) → [First workload](getting-started/first-workload.md) |
| Platform / SRE engineer           | [Architecture → Platform tier](architecture/platform-tier.md) → [Runbooks](operations/runbooks.md) |
| SDK developer                     | [Architecture → SDK stack](architecture/sdk-stack.md) → [Python SDK reference](reference/python-sdk.md) |
| Space / embedded engineer         | [Edge runtime](architecture/edge-runtime.md) → [C-ABI reference](reference/c-abi.md) |
| Chip / verification engineer      | [Portfolio → Chip matrix](portfolio/chip-matrix.md) → [Formal library](reference/formal-library.md) |

## Authoritative sources

This portal **does not duplicate** the engineering specifications under
`programs/`. It links to them. If a spec in `programs/portfolio-upgrade/` and
a page here disagree, the `programs/` copy wins. Every page that references a
spec lists the source document at its head.

```admonish tip title="Canonical navigation"
The single authoritative index of all 47 spec documents is
[`programs/MASTER_INDEX.md`](../../programs/MASTER_INDEX.md). Bookmark it.
```
