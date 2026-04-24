<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/zhilicon-ai/.github/main/profile/assets/zhilicon-logo-dark.png" width="320">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/zhilicon-ai/.github/main/profile/assets/zhilicon-logo-light.png" width="320">
  <img alt="Zhilicon" src="https://raw.githubusercontent.com/zhilicon-ai/.github/main/profile/assets/zhilicon-logo-light.png" width="320">
</picture>

# Zhilicon Developer Documentation

### Public developer documentation for the Zhilicon portfolio — MkDocs Material source + 29 Architecture Decision Records.

[![CI](https://github.com/zhilicon-ai/zhilicon-developer-docs/actions/workflows/ci.yml/badge.svg)](https://github.com/zhilicon-ai/zhilicon-developer-docs/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/zhilicon-ai/zhilicon-developer-docs?include_prereleases&sort=semver&color=0d1117&label=release)](https://github.com/zhilicon-ai/zhilicon-developer-docs/releases/latest)
[![Last Commit](https://img.shields.io/github/last-commit/zhilicon-ai/zhilicon-developer-docs?color=0d1117&label=last%20commit)](https://github.com/zhilicon-ai/zhilicon-developer-docs/commits/main)
[![Portfolio](https://img.shields.io/badge/Zhilicon-v0.2.0-0d1117)](https://github.com/zhilicon-ai)

[![ADRs](https://img.shields.io/badge/ADRs-29_records-0d1117)](docs/adr)
[![MkDocs](https://img.shields.io/badge/mkdocs-material-526CFE?logo=materialformkdocs&logoColor=white)](https://squidfunk.github.io/mkdocs-material/)

</div>

---

<p align="center">
  <a href="https://github.com/zhilicon-ai"><strong>Portfolio</strong></a>&nbsp;·&nbsp;
  <a href="https://github.com/zhilicon-ai/zhilicon-sdk"><strong>SDK</strong></a><sup>🔒</sup>&nbsp;·&nbsp;
  <a href="https://github.com/zhilicon-ai/zhilicon-sdk-examples"><strong>Examples</strong></a>&nbsp;·&nbsp;
  <a href="https://github.com/zhilicon-ai/zhilicon-developer-docs"><strong>Developer Docs</strong></a>&nbsp;·&nbsp;
  <a href="https://github.com/zhilicon-ai/zhilicon-developer-docs/releases"><strong>Releases</strong></a>
</p>

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
