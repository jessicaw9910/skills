---
name: versioning
description: >-
  How my Python packages version themselves — versioningit (git-tag-derived,
  build-time) with dynamic version, importlib.metadata at runtime, and PEP 440
  local-version formatting. Use when cutting a release, adding versioning to a
  package, reading a weird +distance.gsha.dirty version, or wiring a monorepo of
  sub-distributions.
---

# Package versioning (versioningit, git-tag-derived)

Versions are **derived from git tags at build/install time** by
[`versioningit`](https://versioningit.readthedocs.io) — never hardcoded in
`pyproject.toml`. This pairs with the MolSSI cookiecutter-cms layout.

## Wiring (per distribution)

```toml
[build-system]
requires = ["setuptools>=61.0", "versioningit"]
build-backend = "setuptools.build_meta"

[project]
name = "my-dist"
dynamic = ["version"]          # versioningit supplies it; do NOT set version =

[tool.versioningit]
default-version = "1+unknown"  # fallback if there's no git at all (stripped sdist)

[tool.versioningit.format]
distance       = "{base_version}+{distance}.{vcs}{rev}"
dirty          = "{base_version}+{distance}.{vcs}{rev}.dirty"
distance-dirty = "{base_version}+{distance}.{vcs}{rev}.dirty"

[tool.versioningit.vcs]
method = "git"
match = ["*"]                  # which tags count as version tags
default-tag = "0.0.0"          # base version when the repo has no matching tag yet
```

Expose it at runtime by reading installed metadata (don't re-derive at import):

```python
# package __init__.py
from importlib.metadata import version

__version__ = version("my-dist")   # the distribution name, with hyphens
```

## What the number means

Computed from `git describe` against tags:

- **On a tag, clean tree** → just the tag, e.g. `0.1.0`.
- **Past a tag** → `{base}+{distance}.g{shortsha}`, e.g. `0.1.0+437.g7b189bd`
  (437 commits since `0.1.0`).
- **Dirty working tree** → same, with `.dirty` appended:
  `0.1.0+437.g7b189bd.dirty`.

Everything after `+` is a PEP 440 **local version label** — fine for local/dev
installs, but PyPI rejects local versions, so only clean tagged builds are
publishable.

## Cutting a release

The version *is* the tag. To bump it, create and push an (annotated) git tag:

```bash
git tag -a 1.2.0 -m "release 1.2.0"
git push origin 1.2.0
```

Then build/publish from that clean, tagged commit. There's no version string to
edit anywhere.

## Gotchas

- **Captured at build/install time, not import time.** `importlib.metadata`
  reads what was baked into the installed dist's metadata, so an *editable*
  install's `__version__` goes stale (and shows `+distance…dirty`) until you
  reinstall/rebuild. The version in an editable checkout is not "live."
- **No git, no derivation.** Building from a tarball without `.git` falls back to
  `default-version` (`1+unknown`) — keep that fallback so builds don't crash.
- **Monorepo of sub-distributions sharing one tag namespace.** With
  `match = ["*"]`, every distribution in the repo derives from the *same* tags,
  so they move in lockstep off one tag (e.g. `mkt-schema` and `mkt-databases`
  both track `0.1.0`). To version sub-packages independently, give each prefixed
  tags (e.g. `schema-1.2.0`) and set per-package `match = ["schema-*"]` plus a
  `tag2version`/`tag-template` so the prefix is stripped from the base version.
