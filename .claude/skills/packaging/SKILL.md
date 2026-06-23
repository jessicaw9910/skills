---
name: packaging
description: >-
  How my Python distributions are packaged — MolSSI cookiecutter-cms pyproject
  layout (setuptools build backend, dynamic version, optional-dependencies,
  console scripts) and the monorepo pattern: many sub-distributions under one
  umbrella sharing a PEP 420 `mkt` namespace (no top-level `__init__.py`), each
  with its own pyproject.toml and a `graft`-ing MANIFEST.in. Use when adding a
  package or sub-package, wiring entry points/package data, or debugging what
  does/doesn't get installed.
---

# Python packaging (setuptools + pyproject)

Based on the **MolSSI cookiecutter-cms** layout: `setuptools` build backend,
all metadata in `pyproject.toml`, version derived dynamically (see
[[versioning]]). A single repo can ship **one** distribution or, as a monorepo,
**many** sub-distributions that share an import namespace.

## Single distribution — `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=61.0", "versioningit"]
build-backend = "setuptools.build_meta"

[project]
name = "my-dist"                 # distribution name (hyphens)
dynamic = ["version"]            # supplied by versioningit — see [[versioning]]
authors = [{ name="Jess White", email="jessica.white@choderalab.org" }]
description = "..."
readme = "README.md"
requires-python = ">=3.9,<3.13"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: POSIX :: Linux",
    "Operating System :: MacOS",
]
dependencies = ["pydantic>=2.7.4", "..."]

[project.optional-dependencies]
dev  = ["black", "flake8", "sphinx", "sphinx-rtd-theme"]
test = ["pytest", "pytest-cov", "filelock"]

[project.scripts]                # console entry points → import path : callable
my_cmd = "mkt.databases.cli.my_cmd:main"

[project.urls]
"Homepage"   = "https://github.com/choderalab/missense-kinase-toolkit"
"Bug Tracker" = "https://github.com/choderalab/.../issues"

[tool.setuptools.packages.find]
where = ["."]
```

- **Distribution name vs import name** differ: dist `my-dist` (hyphens) can import
  as `my_pkg` or, in the monorepo, `mkt.databases`. PEP 503 normalizes
  `.`/`-`/`_`, so a dependency written `"mkt.schema"` resolves to dist
  `mkt-schema`.
- **`dev`/`test` extras** carry the doc + lint + test tooling so the runtime deps
  list stays clean. Install with `pip install -e ".[dev,test]"`.
- Project-wide tool config (flake8, isort) lives in the repo-root
  `pyproject.toml`; coverage config (`[coverage:run]`) lives in the root
  `setup.cfg`. The root itself is **not** an installable distribution in the
  monorepo — see below.

## Monorepo — many sub-distributions, one shared namespace

Layout (`missense-kinase-toolkit`): an umbrella source dir holds one directory
per installable distribution, and **every** distribution populates the *same*
top-level import package `mkt`:

```
missense_kinase_toolkit/          # umbrella (NOT itself a distribution)
├── schema/                       # dist: mkt-schema
│   ├── pyproject.toml
│   ├── MANIFEST.in
│   └── mkt/                      # ← shared namespace: NO __init__.py here
│       └── schema/
│           └── __init__.py       # ← the real package starts here
├── databases/                    # dist: mkt-databases
│   ├── pyproject.toml
│   ├── MANIFEST.in
│   └── mkt/                      # ← again no __init__.py
│       └── databases/__init__.py
├── ml/        → mkt-ml      → mkt/ml/__init__.py
└── modeling/  → mkt-modeling → mkt/modeling/__init__.py
```

### The namespace rule (the part that trips people up)

`mkt/` is a **PEP 420 namespace package** and must have **no `__init__.py`**.
That absence is load-bearing: it lets several independently pip-installed
distributions each drop their own subpackage into a *single* `mkt/` at runtime,
so `import mkt.schema` and `import mkt.databases` both work even though they came
from different wheels. Put an `__init__.py` in `mkt/` and you turn it into a
regular package owned by one distribution — the others can no longer merge into
it.

The real package (with `__init__.py`) begins one level down, at
`mkt/<subpkg>/`.

### Discovery is namespace-aware — *because* it's in pyproject

```toml
[tool.setuptools.packages.find]
where = ["."]
```

Counter-intuitive but verified: with this declared in **pyproject.toml**,
setuptools discovers namespace packages — equivalent to
`find_namespace_packages`. The legacy `find_packages()` would return `[]` here
(it refuses to recurse past the `__init__.py`-less `mkt/`). So you do **not** add
anything special to opt into namespaces — just declare `packages.find` and omit
the `mkt/__init__.py`.

- **Gotcha — greedy discovery.** Namespace-aware `find` will pick up *any*
  stray top-level dir (e.g. a `data/` folder next to `mkt/`) as its own
  top-level package. If you see unexpected `top_level.txt` entries, constrain it:

  ```toml
  [tool.setuptools.packages.find]
  where = ["."]
  include = ["mkt*"]
  ```

### Per-distribution `MANIFEST.in`

Each sub-distribution carries its own `MANIFEST.in` to pull non-`.py` files
(data, the license) into the sdist/wheel:

```
include LICENSE
include MANIFEST.in
graft mkt
global-exclude *.py[cod] __pycache__ *.so
```

- **`graft mkt`** recursively includes everything under the shared namespace dir
  — this is how package data (CSV/JSON/`.dat`, templates) ships. Without it,
  only `.py` files make it in.
- The repo-**root** `MANIFEST.in` is minimal (e.g. `include CODE_OF_CONDUCT.md`)
  since the root isn't a distribution.

### Inter-package dependencies

Sub-distributions depend on each other by **distribution name** in their
`dependencies` list (e.g. `mkt-databases` lists `"mkt.schema"`). Install in
dependency order, often `--no-deps` against a conda env that already has the
third-party deps — see [[ci]] for the CI install order.

## Adding a new sub-distribution

1. `mkdir <name>/mkt/<name>` and add `mkt/<name>/__init__.py` — **no**
   `mkt/__init__.py`.
2. Copy a sibling's `pyproject.toml`; set `name = "mkt-<name>"`, deps, and
   `[project.scripts]`; keep `dynamic = ["version"]` and the `[tool.versioningit]`
   block (see [[versioning]]).
3. Copy the `MANIFEST.in` verbatim (`graft mkt`).
4. Add a path-filtered CI workflow with its own `--cov` and Codecov `flags` (see
   [[ci]] and the codecov config), and wire docs if it has public API ([[docs]]).
5. `pip install -e "<name>[dev,test]"` and confirm `import mkt.<name>` works.
