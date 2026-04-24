# Architecture Decision Records

Every significant architectural decision across the Zhilicon portfolio,
platform, SDK, and edge runtime is written down as a numbered ADR under
[`docs/adr/`](../../adr/). ADRs are kept outside the mdBook `src/` tree
because they're engineering records — their format is stable and
independent of the rendered portal's navigation.

## Index

| ADR                                                                     | Title                                                            | Status   |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------- | -------- |
| [ADR-0001](../../adr/ADR-0001-errata-first-spec-hygiene.md)             | Errata-first spec hygiene                                        | Accepted |
| [ADR-0002](../../adr/ADR-0002-discovery-1-mtl-32x32-systolic.md)        | Discovery-1 MTL frozen at 32×32 systolic                         | Accepted |
| [ADR-0003](../../adr/ADR-0003-horizon-1-single-tmr-systolic.md)         | Horizon-1 single TMR-protected systolic (16.4 TOPS honest)       | Accepted |
| [ADR-0004](../../adr/ADR-0004-nexus-1-rev-a-b-rf-chain.md)              | Nexus-1 Rev A / Rev B two-revision RF-chain delivery             | Accepted |
| [ADR-0005](../../adr/ADR-0005-sentinel-1-zkp-multi-size-reporting.md)   | Sentinel-1 multi-constraint-size ZKP reporting policy            | Accepted |
| [ADR-0006](../../adr/ADR-0006-sentinel-1-trng-two-stage.md)             | Sentinel-1 two-stage TRNG (RO → Von Neumann → HMAC-DRBG)         | Accepted |
| [ADR-0007](../../adr/ADR-0007-prometheus-intel-18a-with-sf2-parallel.md)| Prometheus Intel 18A primary with Samsung SF2 parallel track      | Accepted |
| [ADR-0008](../../adr/ADR-0008-three-crd-sovereign-platform-model.md)    | Three-CRD sovereign-platform model                                | Accepted |
| [ADR-0009](../../adr/ADR-0009-attestation-service-per-jurisdiction.md)  | Attestation service is per-jurisdiction                           | Accepted |
| [ADR-0010](../../adr/ADR-0010-error-code-result-from-expected.md)       | `ErrorCode` + `Result<T>` as the SDK error model                  | Accepted |
| [ADR-0011](../../adr/ADR-0011-kernels-as-sibling-extension.md)          | `_kernels` as a sibling pybind11 extension                        | Accepted |
| [ADR-0012](../../adr/ADR-0012-horizon-1-edge-runtime-rust-nostd.md)     | Horizon-1 edge runtime is Rust `no_std`                           | Accepted |
| [ADR-0013](../../adr/ADR-0013-kernels-emulation-backend.md)             | `_kernels` emulation backend for laptop-local development          | Accepted |
| [ADR-0014](../../adr/ADR-0014-kernel-library-v1-canonical-set.md)       | Kernel library v1 canonical set                                     | Accepted |
| [ADR-0015](../../adr/ADR-0015-minimal-llama-reference-model.md)         | Minimal LLaMA as the reference model for the v1 kernel set          | Accepted |
| [ADR-0016](../../adr/ADR-0016-kv-cache-design.md)                       | KV cache design for incremental decoding                             | Accepted |
| [ADR-0017](../../adr/ADR-0017-model-persistence-format.md)              | Model persistence format + HuggingFace config mapping                | Accepted |
| [ADR-0018](../../adr/ADR-0018-pytorch-compatibility-bridge.md)          | PyTorch-compatibility bridge for v1 kernels                          | Accepted |
| [ADR-0019](../../adr/ADR-0019-serving-layer.md)                         | Serving layer — single-tenant FastAPI                                | Accepted |
| [ADR-0020](../../adr/ADR-0020-chat-completions-api.md)                  | OpenAI-compatible chat completions API + ChatML template             | Accepted |
| [ADR-0021](../../adr/ADR-0021-in-process-prometheus-metrics.md)         | In-process Prometheus metrics without prometheus-client              | Accepted |
| [ADR-0022](../../adr/ADR-0022-kubernetes-deployment.md)                 | Kubernetes deployment shape for the serving layer                    | Accepted |
| [ADR-0023](../../adr/ADR-0023-huggingface-weight-loader.md)             | HuggingFace SafeTensors weight loader for MinimalLlama               | Accepted |
| [ADR-0024](../../adr/ADR-0024-sovereign-zone-enforcement.md)            | Sovereign-zone enforcement + attestation receipts in the serving layer | Accepted |
| [ADR-0025](../../adr/ADR-0025-crypto-kernel-library-and-tvla-harness.md) | Cryptographic kernel library + TVLA harness — Sentinel-1 gap mitigation | Accepted |
| [ADR-0026](../../adr/ADR-0026-healthcare-sdk-and-discovery1-mitigation.md) | Healthcare clinical-assistant SDK — Discovery-1 category-credibility mitigation | Accepted |
| [ADR-0027](../../adr/ADR-0027-chiplet-simulator-and-prometheus-mitigation.md) | Chiplet simulator + tensor-parallel dispatch — Prometheus 8-chiplet gap mitigation | Accepted |
| [ADR-0028](../../adr/ADR-0028-rad-hard-sdk-and-horizon1-mitigation.md) | Rad-hard SDK (SEU injection + TMR voter + robustness harness) — Horizon-1 gap mitigation | Accepted |
| [ADR-0029](../../adr/ADR-0029-rf-sdk-and-nexus1-mitigation.md) | RF physics SDK (link budget + MIMO channel + beamforming + OFDM) — Nexus-1 gap mitigation | Accepted |

## Process

See [`docs/adr/README.md`](../../adr/README.md) for:

- The status lifecycle (Proposed / Accepted / Superseded / Deprecated).
- The template to copy when authoring a new ADR.
- The rule that an Accepted ADR never gets edited in place — changes go
  through a new ADR that supersedes it.
- Guidance on when a decision warrants an ADR and when it doesn't.
