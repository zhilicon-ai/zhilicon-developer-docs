# How-to: integrate the Python SDK

The Zhilicon Python SDK is installable from the repo in editable mode
during development, and from a private PyPI index in production.

## Install (editable)

```bash
pip install --editable src/sdk/python
python -c "import zhilicon; print(zhilicon.__version__)"
```

## Hello world

```python
import zhilicon as zh

with zh.SovereignContext(country="ae", encrypt=True):
    result = zh.compute(
        "C = A @ B",
        inputs={"A": tensor_a, "B": tensor_b},
        constraints=zh.Constraint(latency_ms=10),
    )
```

The `SovereignContext` tag must match the zone the workload is attested in,
otherwise the runtime refuses to launch.

## Kernel library

The [`zhilicon.kernels`](../../../src/sdk/python/zhilicon/kernels/) package
provides fused primitives:

```python
from zhilicon.kernels import flash_attention, rmsnorm, rope

out = flash_attention(q, k, v)
x   = rmsnorm(x, weight=w, eps=1e-6)
x   = rope(x, cos=cos_tbl, sin=sin_tbl)
```

Each kernel resolves its backend lazily via
[`_backend.py`](../../../src/sdk/python/zhilicon/kernels/_backend.py) — if
the compiled extension is not linked, calls raise a clear `NotImplementedError`
rather than failing at import.

## Ecosystem adapters

| Adapter                                                         | Purpose                                                        |
| --------------------------------------------------------------- | -------------------------------------------------------------- |
| [`integrations.pytorch`](../../../src/sdk/python/zhilicon/integrations/pytorch.py)      | ATen backend for `torch.Tensor ↔ zh.Tensor`                    |
| [`integrations.model_hub`](../../../src/sdk/python/zhilicon/integrations/model_hub.py)  | Load checkpoints from the community model-hub protocol          |
| [`integrations.agents`](../../../src/sdk/python/zhilicon/integrations/agents.py)        | Agent / chain orchestration framework adapter                   |
| [`integrations.inference_api`](../../../src/sdk/python/zhilicon/integrations/inference_api.py) | REST chat-completion-compatible inference server                |
| [`integrations.fastapi_sovereign`](../../../src/sdk/python/zhilicon/integrations/fastapi_sovereign.py) | Sovereign API surface for FastAPI apps                    |

## Domain SDKs

| Domain     | Package                                                        | Example                           |
| ---------- | -------------------------------------------------------------- | --------------------------------- |
| Medical    | [`zhilicon.medical`](../../../src/sdk/python/zhilicon/medical/) | `CTAnalyzer().analyze("/path/to/ct")` |
| Crypto     | [`zhilicon.crypto`](../../../src/sdk/python/zhilicon/crypto/)   | `sign_batch(messages)`             |
| Space      | [`zhilicon.space`](../../../src/sdk/python/zhilicon/space/)     | Star-tracker, SLAM                |
| Telecom    | [`zhilicon.telecom`](../../../src/sdk/python/zhilicon/telecom/) | Beamforming, channel estimation   |

## See also

- [Python SDK reference](../reference/python-sdk.md)
- [SDK architecture](../architecture/sdk-stack.md)
