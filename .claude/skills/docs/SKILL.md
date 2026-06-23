---
name: docs
description: >-
  Building and regenerating Sphinx API docs in MolSSI cookiecutter-cms Python
  projects — autosummary + autodoc + napoleon (NumPy docstrings), recursive
  custom templates, `make html`, and Read the Docs. Use when adding or removing
  public API, fixing docstrings, or refreshing the docs/generated stubs.
---

# Building Sphinx docs (MolSSI cookiecutter-cms)

Projects scaffolded from the
[MolSSI cookiecutter-cms](https://github.com/molssi/cookiecutter-cms) ship a
`docs/` Sphinx skeleton whose **API reference is generated from docstrings**,
not written by hand. You edit code and NumPy-style docstrings; Sphinx
regenerates the per-object `.rst` stubs on build.

## Layout

```
docs/
  conf.py            # Sphinx config (extensions, napoleon, autosummary, theme)
  index.rst          # master toctree (getting_started, api)
  api.rst            # the autosummary entry point — lists top-level packages
  getting_started.rst
  _templates/        # custom-module-template.rst, custom-class-template.rst
  generated/         # AUTO-GENERATED per-object stubs — do not hand-edit
  _build/            # build output (make html -> _build/html/index.html)
  Makefile / make.bat
  requirements.txt   # pip docs deps (sphinx, sphinx-rtd-theme, -e .)
  requirements.yaml  # conda "docs" env (adds myst-parser, nbsphinx, pandoc)
readthedocs.yml      # (repo root) Read the Docs build config
```

## How generation works

- `docs/api.rst` holds a single `.. autosummary::` block with
  `:toctree: generated`, `:template: custom-module-template.rst`, and
  `:recursive:`, listing only the **top-level packages** (e.g. `mkt.schema`,
  `mkt.databases`). Recursion walks submodules/classes from there.
- `conf.py` sets `autosummary_generate = True`, so each build (re)writes the
  stub `.rst` files under `docs/generated/` from the `_templates/` jinja
  templates — one per module/class, each listing members via `autosummary`.
- `sphinx.ext.napoleon` is configured for **NumPy-style** docstrings
  (`napoleon_numpy_docstring = True`), so well-formed docstrings render
  directly — no `.rst` authoring needed.
- An `autodoc-skip-member` hook in `conf.py` prunes Pydantic/`builtins` noise
  (`model_dump`, `__repr__`, etc.) from the member tables.

## Build locally

Create/activate the docs environment, then build:

```bash
# conda (preferred — matches RTD): from docs/requirements.yaml
conda env create -f docs/requirements.yaml   # creates env "docs" (first time)
conda activate docs
# or pip: pip install -r docs/requirements.txt  (installs sphinx + -e .)

cd docs
make clean   # do this whenever the public API changed (see rule below)
make html
# open _build/html/index.html
```

## The rule: never hand-edit `docs/generated/*.rst`

Those stubs are regenerated every build. To **add, remove, or rename** an API
member in the docs, change the code/docstring and rebuild — do not edit the
stub. Because autosummary caches the generated stubs, a plain `make html` may
not drop a removed method or pick up a new one: run **`make clean && make
html`** after any public-API change (added/removed/renamed module, class,
method, or attribute) so the member tables refresh.

## Common tasks

- **New top-level module/package** → add its dotted name to the `autosummary`
  list in `docs/api.rst`; recursion handles the rest.
- **New/removed method on an existing class** → just rebuild with `make clean`;
  the class stub's `Methods` table refreshes from the live class.
- **Docstring not rendering** → confirm it's NumPy-style (napoleon), and that
  the member isn't being dropped by the `skip_member` hook in `conf.py`.

## Read the Docs

Hosting is configured by `readthedocs.yml` at the repo root; RTD installs the
doc dependencies from `docs/requirements.yaml` (autodoc imports the package, so
runtime deps must be available there). After connecting the repo on
readthedocs.org, set the default branch to `main` under Advanced Settings.
