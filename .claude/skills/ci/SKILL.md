---
name: ci
description: >-
  Personal GitHub Actions + pre-commit conventions (path-filtered per-package
  workflows, conda/micromamba test env, OS/Python matrix, Codecov flags,
  pre-commit hook set). Use when editing .github/workflows, adding a
  sub-package, changing the test env, or debugging CI failures.
---

# CI/CD conventions

## Workflows (`.github/workflows/`)

- In a mono repo, prefer **one workflow per installable sub-package**,
  **path-filtered** so each runs only when its sub-package changes. Add a
  periodic safety net (e.g. `cron: "0 0 * * 0"`) so live/network tests still run
  weekly even without changes.
- Test across a matrix where it matters:
  `os: [macOS, ubuntu, windows] × python: ["3.10", "3.11"]`.
- For scientific stacks, set up the environment with
  `mamba-org/setup-micromamba@v1` pointing at a checked-in env file
  (e.g. `devtools/conda-envs/test_env.yaml`) rather than ad-hoc pip installs.
- Upload coverage to Codecov with a per-package `flags` value so coverage is
  attributed correctly per sub-package.

## Adding / changing a workflow

- New installable sub-package → copy an existing workflow, set its `paths`
  filter, install order (dependencies first, then the package, often with
  `--no-deps`), `--cov=<package>`, the test dir, and a unique Codecov `flags`.
- New runtime/test dependency → add it to the conda env file (if CI installs the
  package with `--no-deps`, all deps must come from the env file).
- Tests needing a token are skipped unless the env var is set (see the `tests`
  skill); to enable in CI, add a GitHub Actions secret and export it before
  pytest.

## Pre-commit

Keep a `.pre-commit-config.yaml` with at least:
- `black`
- `isort` (profile "black")
- `flake8` (max-line-length 88, ignore E203/E501)
- `pyupgrade` (target the minimum supported Python, e.g. `--py39-plus`)
- the standard whitespace/end-of-file/yaml hygiene hooks

Run `pre-commit run --all-files` before pushing. Scope hooks (via `files:` /
`exclude:`) to the source tree in a mono repo so generated or vendored
directories aren't reformatted.
