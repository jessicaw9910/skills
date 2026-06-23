---
name: readme
description: >-
  My README skeleton for a MolSSI cookiecutter-cms project — the badge block
  (DOI, codecov, docs, pre-commit.ci, one CI badge per sub-package, app), an
  intro, a monorepo sub-package table, install instructions (git-subdirectory
  until PyPI), and the copyright/acknowledgements/cookiecutter footer. Use when
  creating a README or adding badges/rows for a new sub-package.
---

# README skeleton

Mirrors the MolSSI cookiecutter-cms README, extended for the monorepo. Badges
come from the other skills: codecov ([[codecov]]), docs ([[docs]]), and one CI
badge per sub-package ([[ci]]).

## Badge block

Group with `<br />` so related badges share a line. Order: identity/DOI →
quality (codecov, docs, pre-commit) → per-package CI → app/demo.

```markdown
[//]: # (Badges)
[![DOI](https://zenodo.org/badge/<id>.svg)](https://zenodo.org/doi/10.5281/zenodo.<id>)
[![codecov](https://codecov.io/gh/<org>/<repo>/branch/main/graph/badge.svg)](https://codecov.io/gh/<org>/<repo>/branch/main)
[![Documentation Status](https://readthedocs.org/projects/<repo>/badge/?version=latest)](https://<repo>.readthedocs.io/en/latest/?badge=latest)<br />
[![pre-commit.ci status](https://results.pre-commit.ci/badge/github/<org>/<repo>/main.svg)](https://results.pre-commit.ci/latest/github/<org>/<repo>/main)
[![<pkg>-ci](https://github.com/<org>/<repo>/actions/workflows/<pkg>-ci.yaml/badge.svg)](https://github.com/<org>/<repo>/actions/workflows/<pkg>-ci.yaml)<br />
[![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://<app>.streamlit.app)
```

- **One CI badge per sub-package** — each path-filtered workflow has its own
  badge so a red package is visible at a glance ([[ci]]).
- Codecov badge is repo-wide; per-flag coverage shows on Codecov itself ([[codecov]]).

## Body structure

```markdown
<repo-name> (`<short-name>`)
==============================
<badge block>

## Intro
<one paragraph: what it does, who it's for, link to RTD docs>

## Getting started
<repo> is structured as a monorepo with the sub-packages below.

| Subpackages | Description                                  |
| :-:         | :-                                           |
| `app`       | Streamlit app to visualize ...               |
| `schema`    | Pydantic models + harmonized data ...        |
| `databases` | API clients/scrapers to collect data ...     |
| `ml`        | *In-progress* ML models ...                  |

Sub-packages can be installed directly from GitHub via `pip`:
​```
pip install "git+https://github.com/<org>/<repo>.git#subdirectory=missense_kinase_toolkit/<sub-package>"
​```

### Copyright
Copyright (c) <year>, <author>

#### Acknowledgements
+ <resource links the project relies on>

Project based on the
[Computational Molecular Science Python Cookiecutter](https://github.com/molssi/cookiecutter-cms) version 1.1.
```

- Keep the **monorepo sub-package table** in sync with what's actually
  installable; mark unfinished ones *In-progress* ([[packaging]]).
- Show the **git-subdirectory install** until a package is on PyPI; swap to
  `pip install <dist>` once [[release]] publishing is live.
- Keep the **cookiecutter attribution** footer — it's the provenance line.

## Adding a sub-package

Add a row to the table, add its `<pkg>-ci` badge, and (if it has public API) make
sure it's in the docs ([[docs]]).
