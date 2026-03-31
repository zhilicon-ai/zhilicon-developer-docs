# zhilicon-developer-docs

> Public developer documentation for the Zhilicon AI Chip SDK — quick-start guides, API reference, integration tutorials, and architecture overviews.

[![License](https://img.shields.io/badge/license-CC--BY--4.0-blue.svg)](LICENSE)
[![Docs](https://img.shields.io/badge/docs-developers.zhilicon.ai-brightgreen.svg)](https://developers.zhilicon.ai)

---

## Overview

This is the source repository for [developers.zhilicon.ai](https://developers.zhilicon.ai) — the public developer documentation for the Zhilicon AI Chip ecosystem.

Documentation is authored in Markdown and built with a static site generator. Contributions via pull request are welcome.

---

## Documentation Structure

```
zhilicon-developer-docs/
├── docs/
│   ├── getting-started/       # Installation, first inference, device setup
│   ├── programming-model/     # Execution model, memory model, concurrency
│   ├── api-reference/         # SDK API reference (Python, C++)
│   ├── compiler/              # Graph compiler guide, supported ops, optimization hints
│   ├── runtime/               # Runtime configuration, scheduling, diagnostics
│   ├── integration/           # PyTorch, ONNX, TensorFlow, Triton bridge guides
│   ├── performance/           # Tuning guide, profiling, roofline analysis
│   ├── hardware/              # Public chip overview, memory architecture, I/O
│   └── release-notes/         # SDK release notes by version
├── assets/                    # Diagrams and images
└── site-config/               # Site builder configuration
```

---

## Contributing

Documentation improvements, typo fixes, and new integration guides are welcome from the community. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

For questions, open a [GitHub Discussion](https://github.com/zhilicon-ai/zhilicon-developer-docs/discussions).

---

## Building Locally

```bash
git clone https://github.com/zhilicon-ai/zhilicon-developer-docs
cd zhilicon-developer-docs
pip install -r requirements.txt
mkdocs serve          # Preview at http://localhost:8000
mkdocs build          # Build static site to site/
```

---

## Filing Issues

- **Documentation errors or outdated content:** [Open an issue](https://github.com/zhilicon-ai/zhilicon-developer-docs/issues/new/choose)
- **SDK bugs or feature requests:** Use [zhilicon-sdk-examples](https://github.com/zhilicon-ai/zhilicon-sdk-examples/issues)
- **Security vulnerabilities:** Follow [SECURITY.md](SECURITY.md) — do NOT open a public issue

---

## License

Documentation content is licensed under [Creative Commons Attribution 4.0](LICENSE) (CC-BY-4.0).

Code samples within the documentation are licensed under Apache 2.0.
