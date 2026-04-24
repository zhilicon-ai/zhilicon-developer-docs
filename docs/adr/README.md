# Architecture Decision Records

This directory is the authoritative record of significant architectural
decisions in the Zhilicon portfolio, platform, SDK, and edge runtime.

## Why ADRs

- **Institutional memory.** Six months from now, no-one remembers why we
  froze the Discovery-1 MTL at 32×32 instead of 16×16. The ADR is the
  answer, preserved.
- **Onboarding leverage.** A new engineer reads the numbered ADR set and
  understands the *why* in an afternoon, not the six months of discussion
  that produced each decision.
- **Disagreement discipline.** When two engineers disagree, they argue
  with the written ADR, not with each other's memory of a meeting.

## Format

Every ADR uses the template at [`template.md`](template.md), which matches
the layout already established in
[`github-org-setup/zhilicon-architecture/templates/adr-template.md`](../../github-org-setup/zhilicon-architecture/templates/adr-template.md).

Files are numbered sequentially with a 4-digit prefix:
`ADR-NNNN-<short-kebab-title>.md`. Once assigned, the number never changes.

## Status lifecycle

| Status       | Meaning                                                                |
| ------------ | ---------------------------------------------------------------------- |
| `Proposed`   | Under discussion. Not yet binding.                                     |
| `Accepted`   | Binding — implementations must conform.                                |
| `Superseded` | A later ADR replaced this one. The record stays; its status points to the successor. |
| `Deprecated` | The subject of the decision no longer exists, but the record stays for historical context. |

An ADR never gets edited in place after it reaches `Accepted` — changes go
through a new ADR that supersedes the old one. This is the same discipline
as errata: the document of record doesn't rewrite itself under you.

## Index

| ADR | Title | Status |
| --- | ----- | ------ |
| [ADR-0001](ADR-0001-errata-first-spec-hygiene.md)           | Errata-first spec hygiene                                       | Accepted |
| [ADR-0002](ADR-0002-discovery-1-mtl-32x32-systolic.md)      | Discovery-1 MTL frozen at 32×32 systolic                        | Accepted |
| [ADR-0003](ADR-0003-horizon-1-single-tmr-systolic.md)       | Horizon-1 single TMR-protected systolic (16.4 TOPS honest)      | Accepted |
| [ADR-0004](ADR-0004-nexus-1-rev-a-b-rf-chain.md)            | Nexus-1 Rev A / Rev B two-revision RF-chain delivery            | Accepted |
| [ADR-0005](ADR-0005-sentinel-1-zkp-multi-size-reporting.md) | Sentinel-1 multi-constraint-size ZKP reporting policy           | Accepted |
| [ADR-0006](ADR-0006-sentinel-1-trng-two-stage.md)           | Sentinel-1 two-stage TRNG (RO → Von Neumann → HMAC-DRBG)         | Accepted |
| [ADR-0007](ADR-0007-prometheus-intel-18a-with-sf2-parallel.md) | Prometheus Intel 18A primary with Samsung SF2 parallel track | Accepted |
| [ADR-0008](ADR-0008-three-crd-sovereign-platform-model.md)  | Three-CRD sovereign-platform model (Zone / Pool / Workload)      | Accepted |
| [ADR-0009](ADR-0009-attestation-service-per-jurisdiction.md)| Attestation service is per-jurisdiction, not global              | Accepted |
| [ADR-0010](ADR-0010-error-code-result-from-expected.md)     | `ErrorCode` + `Result<T>` as the SDK error model                 | Accepted |
| [ADR-0011](ADR-0011-kernels-as-sibling-extension.md)        | `_kernels` as a sibling pybind11 extension of `_core`            | Accepted |
| [ADR-0012](ADR-0012-horizon-1-edge-runtime-rust-nostd.md)   | Horizon-1 edge runtime is Rust `no_std`                          | Accepted |
| [ADR-0013](ADR-0013-kernels-emulation-backend.md)           | `_kernels` emulation backend for laptop-local development        | Accepted |
| [ADR-0014](ADR-0014-kernel-library-v1-canonical-set.md)     | Kernel library v1 canonical set                                  | Accepted |
| [ADR-0015](ADR-0015-minimal-llama-reference-model.md)       | Minimal LLaMA as the reference model for the v1 kernel set       | Accepted |
| [ADR-0016](ADR-0016-kv-cache-design.md)                     | KV cache design for incremental decoding                          | Accepted |
| [ADR-0017](ADR-0017-model-persistence-format.md)            | Model persistence format + HuggingFace config mapping             | Accepted |
| [ADR-0018](ADR-0018-pytorch-compatibility-bridge.md)        | PyTorch-compatibility bridge for v1 kernels                       | Accepted |
| [ADR-0019](ADR-0019-serving-layer.md)                       | Serving layer — single-tenant FastAPI                             | Accepted |
| [ADR-0020](ADR-0020-chat-completions-api.md)                | OpenAI-compatible chat completions API + ChatML template          | Accepted |
| [ADR-0021](ADR-0021-in-process-prometheus-metrics.md)       | In-process Prometheus metrics without prometheus-client           | Accepted |
| [ADR-0022](ADR-0022-kubernetes-deployment.md)               | Kubernetes deployment shape for the serving layer                 | Accepted |
| [ADR-0023](ADR-0023-huggingface-weight-loader.md)           | HuggingFace SafeTensors weight loader for MinimalLlama            | Accepted |
| [ADR-0024](ADR-0024-sovereign-zone-enforcement.md)          | Sovereign-zone enforcement + attestation receipts in the serving layer | Accepted |
| [ADR-0025](ADR-0025-crypto-kernel-library-and-tvla-harness.md) | Cryptographic kernel library + TVLA harness — Sentinel-1 gap mitigation | Accepted |
| [ADR-0026](ADR-0026-healthcare-sdk-and-discovery1-mitigation.md) | Healthcare clinical-assistant SDK — Discovery-1 category-credibility mitigation | Accepted |
| [ADR-0027](ADR-0027-chiplet-simulator-and-prometheus-mitigation.md) | Chiplet simulator + tensor-parallel dispatch — Prometheus 8-chiplet gap mitigation | Accepted |
| [ADR-0028](ADR-0028-rad-hard-sdk-and-horizon1-mitigation.md) | Rad-hard SDK (SEU injection + TMR voter + robustness harness) — Horizon-1 gap mitigation | Accepted |
| [ADR-0029](ADR-0029-rf-sdk-and-nexus1-mitigation.md) | RF physics SDK (link budget + MIMO channel + beamforming + OFDM) — Nexus-1 gap mitigation | Accepted |

## How to propose a new ADR

1. Pick the next unused number. Check `git log -- docs/adr/` if you're
   worried about a race.
2. Copy [`template.md`](template.md) to
   `docs/adr/ADR-NNNN-<kebab-title>.md`.
3. Fill it in. Keep each section tight — an ADR is a decision record, not
   an essay.
4. Open a PR. The relevant CODEOWNERS (per the subject area) are the
   deciders.
5. Once the PR merges with approvals from every listed decider, update
   the status from `Proposed` to `Accepted` in a follow-up PR (or in the
   same PR after review).

## When is an ADR warranted

- The decision has cross-cutting consequences or touches multiple teams.
- The decision reverses a prior ADR.
- The decision will be second-guessed in six months.
- An errata entry in
  [`programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md`](../../programs/portfolio-upgrade/SPEC_CORRECTIONS_AND_ERRATA.md)
  is resolved by a real design change rather than a correction.

Low-consequence, reversible, or routinely-made decisions don't need an ADR.
