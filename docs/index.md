# Zhilicon Developer Documentation

Welcome to the official Zhilicon developer portal. This documentation covers everything you need to get productive with the ZHI-1 AI chip — from your first inference on the simulator to deploying production-scale ML workloads on silicon.

---

## Quick Start

=== "pip (recommended)"

    ```bash
    pip install zhilicon-sdk[simulator] --index-url https://pypi.zhilicon.ai/simple/
    export ZHILICON_DEVICE=simulator
    python -c "import zhilicon; print(zhilicon.version())"
    ```

=== "conda"

    ```bash
    conda install -c zhilicon zhilicon-sdk
    export ZHILICON_DEVICE=simulator
    python -c "import zhilicon; print(zhilicon.version())"
    ```

Expected output:

```
zhilicon-sdk 1.1.2  (simulator build)
```

Run your first inference:

```python
import zhilicon as zhi

device = zhi.open_device()                      # simulator or ZHI-1 hardware
model  = zhi.load_model("resnet50.zhimodel")    # pre-compiled .zhimodel
input  = zhi.Tensor.from_numpy(image_array)     # NCHW float16 or int8
output = model.run(input)
print(output.to_numpy())
```

---

## Hardware Access

| Path | Latency | Throughput | How to Get Started |
|------|---------|------------|-------------------|
| **Software Simulator** | Functional accuracy only | N/A (not representative) | `pip install zhilicon-sdk[simulator]` — free, no account required |
| **Evaluation Board (ZHI-1 B0)** | Production silicon | 12,400 img/s ResNet-50 FP16 | [Apply at developers.zhilicon.ai/access](https://developers.zhilicon.ai/access) |

---

## SDK Components

```
zhilicon-sdk
├── zhilicon.device      — device discovery, open, close, capabilities
├── zhilicon.model       — model loading, compilation, execution
├── zhilicon.tensor      — tensor allocation, data transfer, format conversion
├── zhilicon.profiler    — latency profiling, throughput measurement, power metrics
└── zhilicon.compiler    — Python-accessible ONNX → .zhimodel compilation
```

---

## SDK Version Support

| SDK Version | Python 3.10 | Python 3.11 | Python 3.12 | Linux x86-64 | macOS Simulator |
|-------------|:-----------:|:-----------:|:-----------:|:------------:|:---------------:|
| 1.1.x | Yes | Yes | Yes | Yes | Simulator only |
| 1.0.x | Yes | Yes | Yes | Yes | Simulator only |

Hardware execution requires Linux x86-64 with PCIe. macOS supports the simulator only.

---

## Where to Go Next

<div class="grid cards" markdown>

- :material-rocket-launch: **[Getting Started](getting-started/index.md)**
  Run your first model in under 5 minutes on the free simulator.

- :material-book-open-variant: **[SDK API Reference](api-reference/index.md)**
  Complete reference for all SDK modules, classes, and methods.

- :material-chip: **[Compiler Guide](compiler/index.md)**
  Compile ONNX models to `.zhimodel` with quantization and graph optimization.

- :material-gauge: **[Performance Guide](performance/index.md)**
  Profile, tune, and maximize throughput on ZHI-1 hardware.

</div>

---

## Community

| Channel | Link |
|---------|------|
| GitHub Discussions | [Discussions](https://github.com/zhilicon-ai/zhilicon-developer-docs/discussions) |
| Discord | [discord.gg/zhilicon](https://discord.gg/zhilicon) |
| Stack Overflow | [`zhilicon` tag](https://stackoverflow.com/questions/tagged/zhilicon) |
| Forum | [developers.zhilicon.ai/forum](https://developers.zhilicon.ai/forum) |

---

## License

Documentation is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
SDK code examples are licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
