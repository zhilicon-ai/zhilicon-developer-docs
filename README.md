# zhilicon-developer-docs

[![CI](https://github.com/zhilicon-ai/zhilicon-developer-docs/actions/workflows/ci.yml/badge.svg)](https://github.com/zhilicon-ai/zhilicon-developer-docs/actions/workflows/ci.yml)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](LICENSE)
[![Docs](https://img.shields.io/badge/docs-developers.zhilicon.ai-brightgreen)](https://developers.zhilicon.ai)

Source repository for [developers.zhilicon.ai](https://developers.zhilicon.ai) — the public developer documentation for the Zhilicon AI Chip SDK.

---

## About This Repository

This repository contains the Markdown source files that are built into the developer documentation site. It covers:

- **Getting started** — installation, first inference, device enumeration
- **Programming model** — execution model, memory model, concurrency
- **API reference** — Python SDK (`zhilicon`) and C API (`zhilicon.h`)
- **Compiler guide** — supported ops, optimization hints, graph analysis
- **Runtime guide** — configuration, scheduling, diagnostics
- **Integration guides** — PyTorch, ONNX Runtime, TensorFlow, Triton
- **Performance guide** — profiling, roofline analysis, tuning cookbook
- **Release notes** — SDK version history

---

## Contributing Documentation

Documentation contributions are among the most impactful ways to help the Zhilicon developer community. Everyone — internal engineers, partners, and external contributors — can submit improvements.

### Quick Contribution Path

For typo fixes or small corrections, edit the file directly on GitHub and open a pull request. No local setup needed.

For larger contributions, see [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Building Locally

```bash
# Clone
git clone https://github.com/zhilicon-ai/zhilicon-developer-docs
cd zhilicon-developer-docs

# Install MkDocs and theme
pip install mkdocs mkdocs-material mkdocs-minify-plugin

# Preview locally (hot-reload)
mkdocs serve
# → Open http://localhost:8000

# Build static site
mkdocs build --strict
# → Output in site/
```

---

## Repository Structure

```
zhilicon-developer-docs/
├── docs/
│   ├── index.md               # Home page
│   ├── getting-started/       # Installation, quick start, first inference
│   ├── programming-model/     # Execution model, memory model, concurrency
│   ├── api-reference/         # Python and C API reference
│   ├── compiler/              # Graph compiler guide
│   ├── runtime/               # Runtime configuration
│   ├── integration/           # Framework bridge guides
│   ├── performance/           # Profiling and tuning
│   ├── hardware/              # Public chip overview
│   └── release-notes/         # SDK release notes
├── assets/
│   ├── images/                # Diagrams and screenshots
│   └── css/                   # Custom CSS overrides
├── mkdocs.yml                 # Site configuration
└── requirements.txt           # Python dependencies (mkdocs + plugins)
```

---

## Documentation Standards

All documentation follows the standards in [`docs/STYLE_GUIDE.md`](docs/STYLE_GUIDE.md). Key points:

- **Voice:** Direct, technical, no marketing language
- **Code examples:** Always tested, always complete (no `...` placeholders)
- **Headings:** Sentence case
- **Accuracy:** If you're not sure, ask in the PR — don't guess

---

## Style Guide

See [`docs/STYLE_GUIDE.md`](docs/STYLE_GUIDE.md) for complete writing conventions, terminology, diagram standards, and code example requirements.

---

## Filing Issues

- **Documentation errors:** [Open an issue](https://github.com/zhilicon-ai/zhilicon-developer-docs/issues/new/choose) using the "Documentation Issue" template
- **SDK bugs:** Use [zhilicon-sdk-examples](https://github.com/zhilicon-ai/zhilicon-sdk-examples/issues)
- **Security vulnerabilities:** Email security@zhilicon.ai — do NOT open a public issue

---

## License

Documentation content is licensed under [Creative Commons Attribution 4.0 International](LICENSE) (CC-BY-4.0).

Code samples embedded in documentation are licensed under Apache License 2.0.

Zhilicon trademarks and logos are not covered by these licenses.
