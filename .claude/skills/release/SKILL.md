---
name: release
description: >-
  How I cut and publish a release — the version is a git tag (versioningit), so
  a release is: tag the clean commit, let a tag-triggered workflow build the
  sdist+wheel and publish to PyPI via OIDC trusted publishing (no stored token).
  Covers the monorepo case (one tag, build each sub-distribution) and the
  install-from-git fallback when a package isn't on PyPI yet. Use when releasing,
  adding a publish workflow, or deciding how users should install.
---

# Cutting a release

The version **is** the git tag — derived by versioningit at build time, never
edited in a file (see [[versioning]]). So a release is fundamentally: *tag a
clean commit and build from it.* Everything below hangs off that.

## The steps

```bash
# 1. land everything on main, working tree clean (no +dirty in the version)
# 2. create an annotated tag — this IS the version number
git tag -a 1.2.0 -m "release 1.2.0"
git push origin 1.2.0
# 3. the tag push triggers the publish workflow (below)
```

Only a **clean, tagged** commit produces a publishable version. Any distance or
dirtiness yields a PEP 440 local version (`1.2.0+5.gabc123`) that PyPI rejects —
see [[versioning]] for what the suffixes mean.

## Publish workflow (PyPI via OIDC trusted publishing)

Prefer **trusted publishing** (OIDC) over a stored `PYPI_API_TOKEN`: register the
repo+workflow as a trusted publisher on PyPI, then no secret is needed.

```yaml
name: publish
on:
  push:
    tags: ["*"]                 # any version tag
jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write           # required for OIDC trusted publishing
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0, fetch-tags: true }   # versioningit needs tags
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install build
      - run: python -m build      # sdist + wheel into dist/
      - uses: pypa/gh-action-pypi-publish@release/v1   # uses OIDC, no token
```

- **`fetch-depth: 0` / `fetch-tags: true` is mandatory** — a shallow checkout has
  no tags, so versioningit falls back to `default-version` (`1+unknown`) and you
  publish a garbage version. This is the most common release-CI bug.
- Test the path first by publishing to **TestPyPI** (`repository-url`) or running
  `twine check dist/*` before flipping to real PyPI.

## Monorepo — one tag, many distributions

With `match = ["*"]`, all sub-distributions share the repo's tags and move in
lockstep ([[versioning]], [[packaging]]). The publish job builds and uploads each
one from its own directory:

```yaml
      - run: |
          for pkg in schema databases ml; do
            python -m build missense_kinase_toolkit/$pkg --outdir dist/
          done
```

Build in **dependency order** if a later sdist needs an earlier one resolvable;
otherwise order doesn't matter since each is an independent distribution. To
release sub-packages on independent cadences, switch to prefixed tags
(`schema-1.2.0`) and per-package `match` — see the monorepo note in
[[versioning]].

## Install-from-git fallback (not yet on PyPI)

Until a distribution is published, install a sub-package straight from the
monorepo subdirectory:

```bash
pip install "git+https://github.com/choderalab/missense-kinase-toolkit.git#subdirectory=missense_kinase_toolkit/schema"
```

This still runs versioningit against the cloned repo's tags, so the installed
`__version__` is meaningful. Document this form in the [[readme]] until PyPI
publishing is wired up.
