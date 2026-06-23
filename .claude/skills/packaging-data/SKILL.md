---
name: packaging-data
description: >-
  How non-.py files get shipped inside my distributions — MANIFEST.in `graft`
  for the sdist vs `[tool.setuptools.package-data]` for the wheel, the gotcha
  that the package-data key must be the full dotted import package (e.g.
  "mkt.schema", not "schema"), and pruning caches/checkpoints that `graft` would
  otherwise sweep in. Use when a data file (.tar.gz/.cif/.csv/.dat/templates) is
  missing after install, or when adding package data to a sub-distribution.
---

# Shipping package data

Two different mechanisms control two different artifacts, and you usually need
**both** to agree. This is the detail-level companion to [[packaging]].

| File | Controls | Mechanism |
| :- | :- | :- |
| `MANIFEST.in` | what goes in the **sdist** (tarball) | `graft` / `include` / `prune` |
| `pyproject.toml` `package-data` | what goes in the **wheel** (installed) | `[tool.setuptools.package-data]` |

A file present in the sdist but missing from `package-data` won't be installed
from a wheel; the reverse fails to package the source. Make them consistent.

## MANIFEST.in — the sdist side

```
include LICENSE
include MANIFEST.in
graft mkt                       # recursively pull EVERYTHING under the namespace
global-exclude *.py[cod] __pycache__ *.so
```

`graft mkt` is the workhorse — it recursively includes all files (data included)
under the shared `mkt` namespace dir ([[packaging]]). Without it, only modules
discovered as packages ship and your `.tar.gz`/`.cif`/`.csv` are dropped.

- **Gotcha — `graft` is greedy.** It also sweeps in junk that happens to live
  under `mkt/`: `.pytest_cache/`, `.ipynb_checkpoints/`, stray `*.sqlite`. The
  `global-exclude` above only catches bytecode/`__pycache__`/`.so`. Add
  explicit prunes for anything else:

  ```
  prune mkt/**/.pytest_cache
  prune mkt/**/.ipynb_checkpoints
  global-exclude *.sqlite *.ipynb_checkpoints
  ```

  (Mirror these in [[gitignore]] so they're never committed in the first place.)

## package-data — the wheel side

```toml
[tool.setuptools.package-data]
"mkt.schema" = ["KinaseInfo.tar.gz"]    # key = full DOTTED import package
```

- **Gotcha — the key is the dotted package name, not the directory name.** For a
  namespace package the data lives in import package `mkt.schema`, so the table
  key must be `"mkt.schema"`. A bare `schema = [...]` targets a non-existent
  top-level `schema` package and silently ships nothing via the wheel path —
  the file then only survives because `graft mkt` put it in the sdist. If a data
  file is mysteriously present from an sdist install but absent from a wheel
  install, this mismatch is why.
- Alternatively set `include-package-data = true` and let `MANIFEST.in` be the
  single source of truth (setuptools then ships whatever the sdist contains).
  Simpler for the monorepo's "everything under `mkt`" pattern than maintaining
  per-file `package-data` keys.

## Verifying

Build and inspect both artifacts before trusting them:

```bash
python -m build missense_kinase_toolkit/schema --outdir /tmp/d
tar tzf /tmp/d/mkt_schema-*.tar.gz | grep -i kinaseinfo   # in the sdist?
unzip -l /tmp/d/mkt_schema-*.whl   | grep -i kinaseinfo   # in the wheel?
```

If it's in the sdist but not the wheel → fix the `package-data` key (or set
`include-package-data = true`). If in neither → fix `graft`/`include` in
`MANIFEST.in`.

## Reference data outside the package

Large or run-specific inputs (the repo's top-level `data/`, `configs/`,
`*.sqlite` caches) are **not** package data — they're git-ignored generated/
local artifacts ([[gitignore]]). Only ship files the installed code needs at
import/run time; everything else stays out of the distribution.
