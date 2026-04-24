---
adr-id: ADR-0021
title: 'In-process Prometheus metrics without prometheus-client'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/platform-team", "@zhilicon/sre-coe"]
---

# ADR-0021: In-process Prometheus metrics without prometheus-client

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Platform team, SRE COE

## Context

The serving layer ([ADR-0019](ADR-0019-serving-layer.md)) handles
HTTP requests and model generation. Production deployment requires a
scrape endpoint that Prometheus can poll for:

- Request rate + latency + error rate (the "three golden signals").
- Token throughput — the single most-asked question in LLM serving.
- Backend kind — so dashboards can filter emulation vs. native
  (an ADR-0013 guard at the dashboard level, not just at the
  benchmark harness).

Two ways to add this:

1. **Add `prometheus-client` dependency.** The canonical Python
   library. Well-maintained, threadsafe, covers counter / gauge /
   histogram / summary. ~200 KB install.
2. **Ship a minimal in-process registry.** Prometheus exposition
   format is plain text — a faithful serializer fits in ~50 lines.

The SDK already has a "minimise dependencies" principle ([ADR-0017](ADR-0017-model-persistence-format.md)
rejected a heavier serializer for the same reason). Applying it
here: option 2.

## Decision

Ship a minimal :class:`MetricsRegistry` at
[`src/sdk/python/zhilicon/serving/metrics.py`](../../src/sdk/python/zhilicon/serving/metrics.py)
with three primitives — `Counter`, `Gauge`, `Histogram` — and a
`render()` method that emits Prometheus 0.0.4 exposition format.

Middleware auto-instrumentation captures HTTP request count +
duration for every endpoint. Generation-level counters (tokens,
per-generate latency) are recorded inside the `/generate` handler.

### Metrics exposed

| Name                                              | Type      | Labels                      | Purpose                                 |
| ------------------------------------------------- | --------- | --------------------------- | --------------------------------------- |
| `zhilicon_http_requests_total`                    | Counter   | method, path, status        | Request rate, error rate.               |
| `zhilicon_http_request_duration_seconds`          | Histogram | method, path                | Request latency distribution.           |
| `zhilicon_generate_latency_seconds`               | Histogram | stream                      | Model-call latency (excludes HTTP).     |
| `zhilicon_tokens_generated_total`                 | Counter   | —                           | Token throughput counter.               |
| `zhilicon_backend_kind`                           | Gauge     | kind=emulation\|native      | Dashboard filter label.                 |

Histogram bucket upper bounds span 1 ms → 60 s (14 buckets), chosen
for LLM serving — short health probes at the low end, long
prefill-heavy chat requests at the high end.

### `/metrics` scrape endpoint

`GET /metrics` returns `text/plain; version=0.0.4; charset=utf-8` —
the Prometheus exposition format. Middleware-based request counting
explicitly skips the `/metrics` path so a scrape does not perpetually
increment its own counter.

### Thread safety

Every primitive is guarded by `threading.Lock`. FastAPI under uvicorn
with asyncio is single-threaded, but uvicorn workers with the
`--workers N` flag run multiple processes (each has its own registry,
which is fine). Synchronization cost is negligible — one lock
acquisition per observation.

### Label escaping

Label values are escaped per the Prometheus format spec: backslash
→ `\\`, double-quote → `\"`, newline → `\n`. A test
(`test_label_escaping_handles_special_chars`) pins this so a URL
path containing quotes does not corrupt the exposition output.

## Consequences

### Positive

- Zero new pip dependencies. The serving image is ~200 KB smaller.
- Full control over the surface. We emit exactly the metrics we
  want, named the way our dashboards expect, without layering
  adapter code on top of prometheus-client's default labels.
- Test surface is flat. 12 tests cover primitives, rendering,
  middleware integration, and the self-scrape-exclusion guard.
- The registry is accessible via `app.state.metrics` — generation
  wrappers, chat endpoints, and any future feature can record to
  the same registry without a module-level singleton.

### Negative / Risks

- We reimplement something that exists. If Prometheus 2.0 exposition
  format ships a breaking change, we must update our serializer.
  Mitigation: the format is stable (current version 0.0.4 dates
  from 2017); Prometheus has strong backward-compat discipline.
- No OpenMetrics support. The next-generation format has nicer
  features (exemplars, unit inference, `# UNIT` headers). Not a v1
  priority; OpenMetrics is largely a superset that Prometheus
  itself reads from the classic format fine.
- No Summary metric type. Summaries compute quantiles client-side
  which is expensive and rarely worth it over histograms.
  Deliberately not implemented.

### Neutral

- Middleware skips `/metrics` from its own counters. A minor
  accuracy loss (the scrape path is invisible to itself) but
  standard practice — `prometheus-client` itself does the same.

## Alternatives considered

### Option A: Use `prometheus-client`

Canonical Python library; battle-tested. Rejected for dep weight and
because the minimal format serializer is 50 lines of code. If we
ever need summaries, exemplars, or multi-process registries, revisit.

### Option B: OpenTelemetry metrics SDK

Attractive — would let us export to OTLP instead of Prometheus
scrapes. Rejected for v1 scope: the OTel metrics SDK is large, its
API is still evolving, and every sovereign deployment target today
speaks Prometheus scrape natively.

### Option C: No metrics; rely on sidecar / ingress instrumentation

Rejected. Sidecar instrumentation (Istio, Envoy) captures HTTP-level
signals but cannot see tokens-per-second or backend-kind. Those are
model-specific metrics that must come from the serving process
itself.

## References

- [ADR-0013: `_kernels` emulation backend](ADR-0013-kernels-emulation-backend.md)
  — the original backend-kind guard at the benchmark layer
- [ADR-0019: Serving layer](ADR-0019-serving-layer.md)
- [`src/sdk/python/zhilicon/serving/metrics.py`](../../src/sdk/python/zhilicon/serving/metrics.py)
- [`src/tests/sdk/python/test_serving_metrics.py`](../../src/tests/sdk/python/test_serving_metrics.py)
- Prometheus exposition format spec:
  <https://prometheus.io/docs/instrumenting/exposition_formats/>
- Batch-30 session history.
