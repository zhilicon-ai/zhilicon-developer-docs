# Contributing

See the authoritative guide: [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

Summary of what matters for day-one contributors:

- **Read the relevant program README** under `programs/<chip>/` before
  touching anything in that chip's scope.
- **Every commit** carries a DCO `Signed-off-by:` from a real human
  (`git commit -s`). The [DCO workflow](../../.github/workflows/dco.yml)
  enforces this on every PR.
- **Conventional Commits** with silicon-aware scopes: `rtl`, `pd`, `dft`,
  `dv`, `sdk`, `fw`, …
- **CODEOWNERS review** is mandatory for every touched path plus a green
  CI run before merge.
- **No AI-authored attribution** in commits, commit messages, or doc
  front-matter. Authorship is human.

## Running CI locally

Most CI jobs can be reproduced locally:

```bash
# Operator
(cd platform/operator && make test && make lint)

# Attestation service
(cd platform/attestation && cargo fmt --check && cargo clippy -- -D warnings && cargo test)

# Edge runtime
(cd src/edge-runtime && make test && make aarch64)

# Fleet CLI
(cd platform/fleet-cli && ruff check . && mypy zhctl && pytest -q)

# Helm chart
helm lint platform/charts/zhilicon-operator

# Spec tools
npm install && make all
```

## Where to propose larger changes

Open a design-discussion issue before implementation for anything that
touches:

- CRD schemas
- Public SDK API
- Edge-runtime C ABI
- Shared-IP RTL under `src/rtl/ip/common/`
- Cross-chip verification methodology

Small changes go straight to PR.
