---
name: ci
description: >-
  Personal GitHub Actions + pre-commit conventions (path-filtered per-package
  workflows, uv-based test envs with conda only where wheels are missing,
  OS/Python matrix, Codecov flags, pre-commit hook set). Use when editing
  .github/workflows, adding a sub-package, changing the test env, or debugging
  CI failures.
---

# CI/CD conventions

## Workflows (`.github/workflows/`)

- In a mono repo, prefer **one workflow per installable sub-package**,
  **path-filtered** so each runs only when its sub-package changes. Add a
  periodic safety net (e.g. `cron: "0 0 * * 0"`) so live/network tests still run
  weekly even without changes. On `pull_request` events the `paths` filter is
  evaluated against the **whole PR diff**, so every push to an open PR reruns
  every workflow whose paths the PR touches.
- Test across a matrix where it matters:
  `os: [macOS, ubuntu, windows] × python: ["3.10", "3.11"]`.
- Add a `concurrency` group with `cancel-in-progress: true` so a new push
  cancels the superseded run.
- Upload coverage to Codecov with a per-package `flags` value (and
  `files: ./coverage.xml` — `file` is not a valid `codecov-action@v5` input).
- Run pytest with `--durations=20` so slow tests are visible in every log.

## Test environment: uv by default

- Set up Python with `astral-sh/setup-uv` and install from the packages' own
  `pyproject.toml` (`uv pip install ./pkg-a "./pkg-b[test]"`). One install
  resolves everything, local paths satisfy internal dependencies, and CI tests
  exactly the dependencies users get — no separate env file to drift.
- **Pin setup-uv to an exact tag** (e.g. `@v10.2.0`): since v8 it publishes no
  floating major tags, so `@v10` does not resolve.
- Use `activate-environment: true`, `python-version: ${{ matrix.python-version }}`,
  and `cache-dependency-glob` pointing at the `pyproject.toml` files.
- Before adding a dependency, confirm it ships wheels for every matrix
  platform:
  `uv pip compile --python-platform x86_64-pc-windows-msvc --python-version 3.10 --only-binary :all: ...`
  (repeat for `aarch64-apple-darwin` and `x86_64-manylinux_2_28`). Pure-Python
  sdists are fine; compiled sdists will try to build from source in CI.
- Wrap install commands in a short retry-with-backoff loop so transient
  network failures (DNS, PyPI hiccups) don't trip the matrix's `fail-fast`;
  real resolution errors still fail every attempt.

## Conda/micromamba: only where PyPI falls short

- Use `mamba-org/setup-micromamba` only for dependencies without usable PyPI
  wheels on a platform (e.g. a conda-forge build that works where the wheel
  doesn't). Keep that job narrow: a tiny `create-args` env (python + the
  conda-only package), then `uv pip install` the rest on top (uv targets the
  active env via `CONDA_PREFIX`).
- Pass the Python pin through `create-args` (`extra-specs` is silently ignored
  by v1) and set `cache-environment: true`.
- A checked-in conda env file (e.g. `devtools/conda-envs/test_env.yaml`) can
  remain for local conda users, but CI should not be its source of truth.

## Adding / changing a workflow

- New installable sub-package → copy an existing workflow, set its `paths`
  filter, add the package to the `uv pip install` line (dependencies first),
  `--cov=<package>`, the test dir, and a unique Codecov `flags`.
- New runtime dependency → the package's `pyproject.toml` `dependencies`; new
  test-only dependency → its `[test]` extra. CI installs from these.
- Tests needing a token are skipped unless the env var is set (see the `tests`
  skill); to enable in CI, add a GitHub Actions secret and export it before
  pytest.
- macOS runners have much less RAM than Linux/Windows ones; if a session
  fixture is large, each xdist worker holds its own copy, so lower `-n` on
  macOS (e.g. `-n ${{ runner.os == 'macOS' && '0' || '2' }}`).

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
