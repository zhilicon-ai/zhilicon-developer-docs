# Release process

## Cadence and versioning

Semantic versioning 2.0.0. Major / minor / patch determined by the labels on
the PRs included in the release (see the `version-resolver` section of
[`release-drafter.yml`](../../../.github/release-drafter.yml)).

## Single source of truth

Every version string lives once: in the top-level [`VERSION`](../../../VERSION)
file. Every secondary manifest (Helm chart, Cargo crates, Python package
version) is bumped from there by
[`scripts/version_bump.py`](../../../scripts/version_bump.py).

## Standard cut

```bash
# 1. Verify everything agrees with the current VERSION.
python3 scripts/version_bump.py --check

# 2. Bump (patch example).
python3 scripts/version_bump.py --bump patch
# Review the diff.
git diff
git commit -s -am "chore(release): v0.1.1"
git push

# 3. Merge the release PR on main. The release-drafter draft now reflects
#    every PR that landed since the previous tag.

# 4. Publish the release:
#    - Edit the draft's "Highlights" paragraph.
#    - Tag the commit:
git tag -s v0.1.1 -m "Zhilicon v0.1.1"
git push origin v0.1.1
```

The [`release` workflow](../../../.github/workflows/release.yml) fires on the
tag. It:

1. Verifies the tag matches `VERSION`.
2. Builds + pushes `zhilicon/{operator,device-plugin,attestation}:X.Y.Z` to GHCR.
3. Packages the Helm chart `zhilicon-operator-X.Y.Z.tgz`.
4. Cross-compiles the Horizon-1 edge-runtime staticlib for `armv7r-none-eabihf`
   and bundles the C header.
5. Generates CycloneDX SBOMs for every image + the source tree.
6. Publishes a GitHub Release with every artefact attached.

## Pre-release tags

Semver pre-release suffixes (`-rc.1`, `-beta.2`) are supported. The release
workflow marks the GitHub Release as a pre-release automatically if the
version contains a hyphen.

## Emergency patch releases

If a security fix must ship outside the normal cadence:

1. Cut from the affected release tag, not `main`:
   `git switch -c hotfix/v0.1.2 v0.1.1`
2. Apply the fix, bump to patch version, tag and push.
3. Cherry-pick the fix into `main` with `git cherry-pick -x`.

## Checklist

Every release uses the standard checklist at
[`scripts/release-checklist.md`](../../../scripts/release-checklist.md).
Copy it into the release PR and tick every box before merging.

## Release notes

The [release-drafter workflow](../../../.github/workflows/release-drafter.yml)
maintains a draft release on every push to `main` and every PR event,
categorising entries by Conventional-Commits scope label. The maintainer
fills in a short **Highlights** paragraph before publishing.

## What does not change between releases

- The C ABI at `src/edge-runtime/include/zhilicon_edge.h` — append-only
  through patch / minor, breaking changes only at major.
- The CRD schemas in `platform/operator/config/crd/bases/` — additive
  changes only within a minor; breaking changes go through a CRD version
  bump (e.g. `v1alpha1 → v1alpha2`) with a deprecation window.
- The attestation wire format documented in
  [REST API reference](../reference/rest-apis.md) — versioned by the
  `version` field inside the payload; minor changes add new optional
  fields, never rename existing ones.
