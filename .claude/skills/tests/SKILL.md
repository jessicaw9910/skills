---
name: tests
description: >-
  Personal pytest conventions (layout, the network marker, token-gated skips,
  session/module fixtures, xdist-safe patterns, graceful skips on transient API
  failures). Use when adding or running tests, or debugging test failures.
---

# Testing conventions (pytest)

## Layout

- Keep tests close to the code: one `test_<module>.py` per source module, with
  shared fixtures in a `conftest.py` in the test directory.
- Run with the project's virtual environment, not the system Python.

```bash
pytest -v --cov-report=xml --color=yes --cov=<package> path/to/tests/
```

## Markers and gating

- Mark any test that hits the network with `@pytest.mark.network` (register the
  marker in `conftest.py`). Apply it to the class or function. This lets CI and
  local runs deselect network tests with `-m "not network"`.
- Gate tests that need a secret/token behind a module-level skip rather than
  letting them fail when the credential is absent:

  ```python
  pytestmark = pytest.mark.skipif(
      get_token() is None,
      reason="API token not set; skipping live tests",
  )
  ```

## Fixture conventions (`conftest.py`)

- Query each external API **once** — use `scope="session"` or `scope="module"`
  fixtures, then assert against the shared object. Put imports inside the
  fixture to keep collection cheap.
- Skip gracefully on transient API failures: call `pytest.skip(...)` when a live
  endpoint returns non-200 rather than asserting hard, so flaky networks don't
  fail the suite.
- Set configuration via the project's config setters in fixtures, not by
  mutating `os.environ` directly.

## Parallel (xdist) safety

If the suite runs under `pytest-xdist` (e.g. `-n 2 --dist loadfile`), keep it
parallel-safe:

- Keep network fixtures session/module-scoped so workers don't each re-query.
- Guard shared on-disk work with a `filelock.FileLock` so parallel workers don't
  clobber each other.

## Regression tests for hard-won rules

When a class of bug recurs (e.g. SVG transparency in figures), add a regression
test that asserts the invariant directly, so the fix can't silently regress.
