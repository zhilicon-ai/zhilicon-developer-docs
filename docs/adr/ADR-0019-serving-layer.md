---
adr-id: ADR-0019
title: 'Serving layer — single-tenant FastAPI'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/platform-team", "@zhilicon/devrel"]
---

# ADR-0019: Serving layer — single-tenant FastAPI

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Platform team, DevRel

## Context

With [ADR-0014](ADR-0014-kernel-library-v1-canonical-set.md) freezing
the kernel surface, [ADR-0015](ADR-0015-minimal-llama-reference-model.md)
shipping a reference model, [ADR-0016](ADR-0016-kv-cache-design.md)
enabling incremental decoding, and [ADR-0017](ADR-0017-model-persistence-format.md)
defining the checkpoint format — the last missing piece for a
laptop-demonstrable, sovereign-ready end-to-end story is an HTTP
serving endpoint.

Every prospective customer opens with the same question: *"How do I
expose the model to my application?"* Answering that concretely
requires a reference implementation of the wire protocol, not just a
pointer to "go write your own FastAPI wrapper around the Python API."

Three things force specific decisions:

1. **Single-tenant vs multi-tenant.** We could ship a router that
   switches between multiple loaded models in one process. Attractive
   for laptop development but the wrong shape for production —
   sovereign deployments want one model per process per namespace for
   isolation and cost accountability.
2. **Streaming protocol.** Chat UIs expect token-by-token output.
   Two viable protocols: server-sent events (SSE) and WebSocket.
3. **OpenAI compatibility.** The entire client ecosystem (LangChain,
   LlamaIndex, the Vercel AI SDK, every curl-based demo) speaks
   OpenAI's HTTP convention. Matching even a subset of it is
   overwhelmingly valuable.

## Decision

### Single-tenant FastAPI app factory

`zhilicon.serving.create_app(model, tokenizer=None, *, model_id, ...)`
returns a FastAPI application bound to exactly one model. No router,
no multi-model switching, no model-swap endpoint. To serve two
models, run two processes — Kubernetes handles the fleet layer, not
the Python process.

This aligns with how vLLM, Text Generation Inference, and the
NVIDIA Triton Python backend draw the same line: serving code does
one model; orchestration does many.

### FastAPI + pydantic, lazy-imported

FastAPI is not a hard SDK dependency. The kernel library and the
reference model must remain installable without `pip install
fastapi` — a user running notebooks should not pay the dependency
cost. The serving layer's imports are gated:

- Module import is always safe (no FastAPI import at top level).
- `create_app(...)` call raises a clear `ImportError` pointing at the
  install line if FastAPI or pydantic is absent.
- Pydantic request/response models are defined at module level,
  inside a `try: from pydantic import BaseModel` guard that falls
  back to placeholder classes. This is required because FastAPI's
  body-parameter detection fails on locally-scoped pydantic models
  (returns 422 on every POST).

### Endpoint surface

| Method | Path                | Purpose                                                   |
| ------ | ------------------- | --------------------------------------------------------- |
| GET    | `/healthz`          | Process liveness. Always 200 when up.                     |
| GET    | `/readyz`           | Model readiness. 200 after a 1-token forward pass succeeds. |
| GET    | `/v1/models`        | OpenAI-compat: list the single loaded model.              |
| POST   | `/generate`         | Synchronous completion. JSON in, JSON out.                |
| POST   | `/generate/stream`  | SSE streaming completion.                                 |

Intentional omissions:

- **No `/v1/chat/completions`.** That endpoint requires chat-template
  machinery (system / user / assistant message formatting) which is
  a tokenizer concern and varies per model family. A separate
  wrapper module can layer it on top once the tokenizer surface
  grows a real chat template.
- **No `/v1/completions` path.** OpenAI has already deprecated it in
  favour of chat completions; we skip to the modern surface.
- **No authentication.** Sovereign deployments terminate auth at the
  ingress (mTLS, JWT-bearer at the gateway). Adding it at the
  serving layer duplicates effort and obscures where the policy
  actually lives.

### Streaming is SSE, not WebSocket

Server-sent events are:

- One-directional (server → client), which matches the generation
  semantics exactly.
- Plain HTTP/1.1 with `text/event-stream`, so every existing proxy,
  load balancer, and firewall already handles them.
- The protocol the OpenAI streaming API uses, which means Vercel AI
  SDK, LangChain, and `curl` consume our stream without custom code.

WebSocket would win for bidirectional tool-calling scenarios. Those
are out of scope for v1; the HTTP surface is the right starting
layer.

Per-token payload:

```json
{"token_id": 104, "text": "h"}
```

Stream terminates with the literal `data: [DONE]` sentinel, matching
OpenAI.

### Request body — prompt OR prompt_ids

Two mutually-exclusive request shapes:

1. `{"prompt": "hello world"}` — text, requires a tokenizer attached
   at `create_app()` time.
2. `{"prompt_ids": [104, 101, 108, ...]}` — integer IDs, works even
   when the server has no tokenizer.

The choice is per-request; servers may advertise either capability.
This lets a tokenizer-free deployment (where the client tokenises)
coexist with a tokenizer-enabled one (where the server does it),
without a configuration flag.

### Dockerfile + entrypoint.py

A `python:3.12-slim`-based container with the SDK installed in
editable mode, plus a default entrypoint that:

- Loads a checkpoint at `/model/model.npz` if present.
- Falls back to a random-init 311K-parameter model for CI smoke-tests.
- Pairs with `ByteTokenizer` by default, validating vocab-size parity.
- Runs as a non-root user for Kubernetes pod-security compatibility.

Environment-variable configuration (`ZHILICON_MODEL_PATH`,
`ZHILICON_MODEL_ID`, `ZHILICON_ENABLE_CORS`) — no config file.
Twelve-factor by default.

## Consequences

### Positive

- A complete path from `git clone` → `uvicorn ... entrypoint:build`
  → `curl http://localhost:8080/generate`. Proof of end-to-end
  working system, not just a library.
- OpenAI-compatible endpoint surface means the entire client
  ecosystem works against us on day one. No custom SDK required.
- Test surface matches the wire protocol. `TestClient` exercises
  every endpoint without a running server — cheap to keep green in
  CI.
- Container is ~200 MB and boots in under 5 seconds on a laptop.

### Negative / Risks

- FastAPI has a non-trivial dependency footprint (starlette,
  pydantic-core, uvicorn). Mitigation: serving is an optional
  extra, not a core SDK dep. `pip install zhilicon` stays light.
- OpenAI-style SSE is text-based (JSON per frame, one frame per
  token). For high-throughput workloads this is overhead vs a
  binary protocol. Accept for v1; a gRPC variant is a v2 ADR when
  the throughput story gets serious.
- Single-tenant means each loaded model spends ~500 MB of Python
  process overhead even for a tiny random-init test model. This is
  fine for production (real LLaMA checkpoints dwarf it) and
  acceptable for dev.

### Neutral

- No authentication in the serving layer itself. Deployments stack
  an ingress (Istio, NGINX, Envoy) in front. This is the same
  trade-off vLLM and TGI make.
- No prometheus metrics endpoint yet. The benchmark harness guards
  (ADR-0013) keep the observability surface clean; a `/metrics`
  endpoint is a v2 addition.

## Alternatives considered

### Option A: Implement a custom minimal HTTP server on the stdlib

Attractive from a dependency-weight perspective (`http.server` is
free). Rejected because:

- No request-body validation — every endpoint would repeat the same
  JSON-schema checks.
- No OpenAPI doc generation — rules out autogenerated client SDKs.
- No streaming primitive for SSE — would need custom chunked
  transfer encoding handling.

FastAPI buys all three for the dependency cost.

### Option B: Ship vLLM / TGI wrapper

Rejected. vLLM and TGI expect CUDA kernels and a PyTorch tensor
model. Our kernels run on numpy today (emulation) or Zhilicon silicon
(native); wedging into a CUDA-first server would force a CUDA
translation that makes no sense.

### Option C: WebSocket-only streaming

Rejected for v1. WebSocket upgrades break through fewer proxies,
require custom framing, and offer no advantage for the one-shot
generation pattern. SSE is the simpler answer for the same use case.

### Option D: Multi-model router in one process

Rejected. The operational cost (hot-reload, memory pressure,
request-routing logic) does not pay off — Kubernetes handles the
routing layer. The serving layer stays a library of N identical
processes, and the operator is the router.

## References

- [ADR-0014: Kernel library v1 canonical set](ADR-0014-kernel-library-v1-canonical-set.md)
- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [ADR-0016: KV cache design](ADR-0016-kv-cache-design.md)
- [ADR-0017: Model persistence format](ADR-0017-model-persistence-format.md)
- [`src/sdk/python/zhilicon/serving/app.py`](../../src/sdk/python/zhilicon/serving/app.py)
- [`src/sdk/python/zhilicon/serving/Dockerfile`](../../src/sdk/python/zhilicon/serving/Dockerfile)
- [`src/sdk/python/zhilicon/serving/README.md`](../../src/sdk/python/zhilicon/serving/README.md)
- [`src/tests/sdk/python/test_serving.py`](../../src/tests/sdk/python/test_serving.py)
- Batch-27 session history.
