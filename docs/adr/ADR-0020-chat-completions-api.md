---
adr-id: ADR-0020
title: 'OpenAI-compatible chat completions API + ChatML template'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/devrel", "@zhilicon/platform-team"]
---

# ADR-0020: OpenAI-compatible chat completions API + ChatML template

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, DevRel, Platform team

## Context

[ADR-0019](ADR-0019-serving-layer.md) shipped `/generate` and
`/generate/stream` — our native HTTP surface, well-shaped for the
kernel library's capabilities. It is *not* the shape the client
ecosystem expects.

Every LLM client library (LangChain, LlamaIndex, Vercel AI SDK,
LiteLLM, OpenAI's own SDK, Instructor, marvin, guidance) defaults to
the OpenAI `/v1/chat/completions` endpoint shape. Users starting with
any of those libraries cannot point them at Zhilicon without an
adapter. "Write your own adapter" is unacceptable adoption friction
— vLLM, Text Generation Inference, llama.cpp, LocalAI, and Ollama
all ship OpenAI-compatible endpoints for this exact reason.

Two decisions forced:

1. **Message → prompt conversion.** Chat messages are role-tagged
   objects; the model consumes a single string. The conversion is
   *model-specific* — LLaMA-2, LLaMA-3, Qwen, Gemma, ChatGLM all use
   different delimiter conventions. We ship one default and leave
   room for others.
2. **Streaming shape.** OpenAI's chat streaming emits a three-phase
   sequence (role, deltas, final + `[DONE]`) wrapped in a full
   chat-completion-chunk envelope per frame. Ours has been a single
   `{"token_id": int, "text": str}` per-frame. They do not overlap.

## Decision

### Ship `/v1/chat/completions` with OpenAI wire compatibility

Request body matches the OpenAI schema at the fields we support:
`messages`, `model`, `max_tokens`, `temperature`, `top_p`, `seed`,
`stream`, `stop`. Unknown fields are accepted via
`model_config = {"extra": "allow"}` and silently ignored — clients
sending `tools`, `tool_choice`, `response_format`, `logprobs`, `n`,
etc. do not get rejected.

Response envelope matches OpenAI verbatim:

```json
{
  "id": "chatcmpl-<hex>",
  "object": "chat.completion",
  "created": <unix_ts>,
  "model": "<model_id>",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "..."},
    "finish_reason": "stop" | "length"
  }],
  "usage": {
    "prompt_tokens": N,
    "completion_tokens": M,
    "total_tokens": N + M
  }
}
```

Streaming emits `chat.completion.chunk` envelopes — three phases:

1. **Role chunk** (once at start):
   `{"delta": {"role": "assistant"}, "finish_reason": null}`
2. **Content chunks** (N × per-token):
   `{"delta": {"content": "<fragment>"}, "finish_reason": null}`
3. **Terminal chunk** (once at end):
   `{"delta": {}, "finish_reason": "stop" | "length"}`
   followed by `data: [DONE]`.

This layered protocol matches what the Vercel AI SDK, LangChain
streaming callbacks, and LiteLLM stream handlers expect. Any single
deviation breaks them. The test suite pins all three phases.

### ChatTemplate class with `chatml` default

The `messages → prompt` conversion lives in
:class:`zhilicon.serving.chat.ChatTemplate`. Two built-in formatters:

- **`chatml`** (default) — `<|im_start|>{role}\n{content}<|im_end|>\n`
  with a trailing `<|im_start|>assistant\n` generation prompt.
  This is the OpenAI / Qwen convention and the closest thing to a
  universal format for open models.
- **`simple`** — `{Role}: {content}\n` with a trailing `Assistant: `
  prompt. Not faithful to any trained model; useful for terminal demos
  and for tests where the prompt string must be human-readable.

Callers needing a model-specific template (LLaMA-3 header-id, Mistral
`[INST]`, Gemma `<start_of_turn>`) pass a custom `ChatTemplate`
subclass — or, eventually, read the template from the tokenizer's
metadata like HuggingFace does.

Intentional omissions for v1:

- **Jinja2-based templates.** HuggingFace embeds the chat template as
  a Jinja2 string in tokenizer config. Supporting that requires a
  Jinja2 dependency and sandboxed execution. For a byte-level
  tokenizer reference demo, a typed Python function is cleaner.
- **Per-tokenizer template selection.** The template is passed
  explicitly at `register_chat_endpoints()` time, not inferred.

### Endpoint mounted only when tokenizer is present

`create_app(model, tokenizer, ...)` conditionally attaches the chat
endpoints — if `tokenizer is None`, `/v1/chat/completions` simply is
not there. Rationale: the chat API takes strings; a server without a
tokenizer cannot serve them. Better to 404 than to return 400 on every
call.

Opt-out knob: `create_app(..., enable_chat=False)` disables the chat
surface even when a tokenizer is attached — useful when stacking a
different chat server in front.

### What we do NOT implement (yet)

| Feature                          | Reason for deferral                                               |
| -------------------------------- | ----------------------------------------------------------------- |
| `stop` sequences                 | Accepts at schema level; enforcement needs per-token match check. |
| `n > 1` multiple completions     | Trivial to add; no user demand yet.                               |
| `tool_calls` / function calling  | Needs tokenizer awareness of `<tool>` markers. Model-specific.    |
| `response_format: "json_object"` | Needs constrained decoding. Separate batch.                       |
| `logprobs` / `top_logprobs`      | We have logprobs per step (streaming path); wiring to response shape is straightforward but deferred. |

All are backward-compatible additions to the envelope.

## Consequences

### Positive

- Every OpenAI-API-compatible client works against us on day one. No
  custom SDK required.
- The static HTML chat UI ([demo/chat-ui](../../demo/chat-ui)) talks
  to the endpoint with vanilla `fetch` + SSE — demonstrable.
- Test surface matches the wire protocol. 13 new tests pin the
  envelope + streaming phases + determinism + schema
  forward-compatibility.
- ChatTemplate is separately testable and can be reused outside
  FastAPI — for example, by the benchmark harness if we want to
  measure prefill latency on realistic prompts.

### Negative / Risks

- The chat streaming format is verbose — a JSON envelope per token
  adds ~200 bytes of overhead per frame. Acceptable for v1;
  binary-framed alternatives (gRPC, MessagePack SSE) are a v2
  consideration when throughput becomes limiting.
- ChatML delimiter bytes (`<|im_start|>`, `<|im_end|>`) are multi-byte
  under our byte-level tokenizer. A real BPE tokenizer with these as
  single special tokens is more efficient. Deferred until a real
  tokenizer replaces ByteTokenizer.
- Clients relying on `n > 1`, `response_format`, or `tool_calls`
  will silently ignore those fields and get a plain completion.
  Mitigation: documented; the fields are schema-allowed for forward
  compat.

### Neutral

- `stop` sequences accepted but not honoured. Clients that depend on
  them will see longer completions than expected; a v2 batch adds
  per-token matching.

## Alternatives considered

### Option A: Only ship our native `/generate` endpoint

Rejected. Forces every adoption path to write an adapter.
`vLLM` / `TGI` / `Ollama` all learned this lesson and shipped
OpenAI-compatible endpoints for the same reason.

### Option B: Ship OpenAI's `/v1/completions` (text-only, no messages)

Rejected. OpenAI deprecated it in favour of chat completions. New
clients do not target it; supporting it means maintaining two code
paths for no benefit.

### Option C: Embed a Jinja2 chat template engine

Attractive — matches HuggingFace's approach. Rejected for v1 on
dependency weight (jinja2 is 200 KB, needs sandboxing for untrusted
templates) and because our tokenizer does not yet consume HF
tokenizer configs. Revisit when real BPE tokenizers land.

### Option D: Expose `/v1/chat/completions` without a tokenizer (raw token passthrough)

Rejected as incoherent. The endpoint's value is "text in, text out";
without a tokenizer it is just a worse `/generate`.

## References

- [ADR-0019: Serving layer](ADR-0019-serving-layer.md)
- [ADR-0017: Model persistence format](ADR-0017-model-persistence-format.md)
- [`src/sdk/python/zhilicon/serving/chat.py`](../../src/sdk/python/zhilicon/serving/chat.py)
- [`src/tests/sdk/python/test_serving_chat.py`](../../src/tests/sdk/python/test_serving_chat.py)
- [`demo/chat-ui/index.html`](../../demo/chat-ui/index.html)
- Batch-29 session history.
