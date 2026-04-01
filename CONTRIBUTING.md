# Contributing to zhilicon-developer-docs

Documentation contributions are among the most impactful ways to help the Zhilicon developer community. Accurate, clear documentation directly reduces friction for every engineer integrating with ZHI-1.

---

## Who Can Contribute

Everyone — Zhilicon engineers, partners, SDK users, and the broader community. External contributors are welcome and valued.

---

## Three Contribution Paths

### Path 1: Quick Edit on GitHub (no local setup)

For typo fixes, factual corrections, and small wording improvements:

1. Find the page on [developers.zhilicon.ai](https://developers.zhilicon.ai)
2. Click the edit icon (links to the source file in this repo)
3. Make your change directly on GitHub
4. Open a pull request with a one-sentence description of the fix

No local environment needed. This is the fastest path for small changes.

### Path 2: Local Build (for medium changes)

For changes to one or a few pages where you want to preview locally:

```bash
git clone https://github.com/zhilicon-ai/zhilicon-developer-docs
cd zhilicon-developer-docs

pip install -r requirements.txt

# Preview with hot-reload
mkdocs serve
# → Open http://localhost:8000
```

Edit files under `docs/`, verify they look correct in the preview, then open a PR.

### Path 3: Large Contribution (new sections, new guides)

For new documentation sections, new integration guides, or structural changes:

1. **Open an issue first** describing what you want to add and why. This avoids duplicate effort and gets early feedback on scope.
2. Get a thumbs-up from `@zhilicon-ai/docs` before writing.
3. Follow the full local build path above.
4. Ensure all code examples are tested (see Content Guidelines below).
5. Reference the open issue in your PR description.

---

## DCO Sign-Off Requirement

All contributions require a Developer Certificate of Origin sign-off. This certifies you wrote the contribution or have the right to submit it.

```bash
git commit -s -m "docs: fix Device.allocate() parameter description"
```

This adds `Signed-off-by: Your Name <email@example.com>` to the commit. PRs with unsigned commits will not be merged.

---

## Content Guidelines

### Accuracy first

- If you are not certain something is correct, say so in the PR and ask for review rather than guessing
- Do not document APIs, behaviors, or configurations you haven't personally verified
- Do not copy content from internal sources that have not been cleared for public disclosure

### Code example requirements

- Every code example must be tested against the current SDK
- Mark untested examples with a `# Untested` comment at the top of the block
- Examples must be complete — no `...` placeholders, no unexplained prerequisites
- Show realistic inputs and outputs, not `foo`/`bar` placeholders

### Style

Follow [`docs/STYLE_GUIDE.md`](docs/STYLE_GUIDE.md) for voice, terminology, heading case, diagram format, and link conventions. Key points:

- Direct, technical voice — no marketing language
- Sentence-case headings
- Second person ("you") in tutorials; third person or imperative in reference docs

### What not to include

- Internal architecture details not cleared for public disclosure
- Performance numbers not published in [zhilicon-benchmarks](https://github.com/zhilicon-ai/zhilicon-benchmarks)
- Anything received under NDA
- Internal tooling, infrastructure, or build system documentation
- Anything that references internal GitHub repositories external users cannot access

---

## Pull Request Guidelines

- **One topic per PR** — keep PRs focused; it makes review faster and cleaner
- **Link to the live page** if you are fixing an existing page (helps reviewers find it)
- **Run `mkdocs build --strict`** locally before opening the PR — this catches broken links and invalid syntax
- **Describe the change** in the PR body: what was wrong or missing, and why your change is correct

---

## Review Process

| Change Type | Reviewers |
|-------------|-----------|
| Typos, broken links, formatting | `@zhilicon-ai/docs` (one reviewer sufficient) |
| Conceptual explanations, new guides | `@zhilicon-ai/docs` (two reviewers) |
| API reference changes | `@zhilicon-ai/docs` + relevant engineering team |
| Style guide changes | `@zhilicon-ai/docs` + discussion issue required first |

API reference PRs require engineering co-review to ensure technical accuracy. Allow up to 5 business days for review of larger contributions.

---

## Filing Documentation Issues

Not ready to write a fix yourself? File an issue:

- Use the "Documentation Issue" template
- Describe what is wrong or missing
- Include a link to the affected page on developers.zhilicon.ai

---

## Code of Conduct

All contributors are expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md).
