---
name: gitignore
description: >-
  How to generate a .gitignore for a Python project — MolSSI cookiecutter base
  plus personal add-ons (build/_version, caches, OS junk, analysis outputs,
  and a keep-skills .claude rule). Use when creating or extending a .gitignore.
---

# Generating a `.gitignore`

Start from the **MolSSI cookiecutter** Python `.gitignore` (the standard
GitHub `Python.gitignore` plus the toptal `osx,windows,linux` block), then layer
on the personal add-ons below.

## Base — start from the standard Python ignores

Pull in the standard Python set (or the MolSSI cookiecutter version, which is
the same base): byte-compiled files, distribution/packaging dirs, installer
logs, test/coverage reports, virtualenvs, type-checker caches, Jupyter
checkpoints, Sphinx/mkdocs build output, etc. Append the toptal
`osx,windows,linux` block for OS cruft (`.DS_Store`, `Thumbs.db`, `*~`, …).

Easiest way to fetch a fresh copy:

```bash
curl -sL https://www.toptal.com/developers/gitignore/api/python,osx,windows,linux \
  -o .gitignore
```

## Personal add-ons (append these)

```gitignore
# In-tree generated version files (setuptools-scm etc.)
*/_version.py

# mypy
.mypy_cache/

# local caches / scratch data
requests_cache*
*.sqlite
*params.json

# generated analysis outputs — don't version-control
images/
data/
configs/        # plot/run config files not shipped with the package

# project virtualenv directory (if you name it VE)
VE/

# Claude Code working files, but KEEP version-controlled project skills
.claude/*
# but keep version-controlled project skills so they're shared/reusable
!.claude/skills/
```

## Notes

- The `.claude/*` + `!.claude/skills/` pair is deliberate: ignore Claude's local
  state but keep `.claude/skills/` checked in so skills are shared and reusable
  across the team/repo.
- Tune the output-directory ignores (`images/`, `data/`, `configs/`) to whatever
  the project actually generates; the intent is "don't commit regenerated
  artifacts."
- Keep `*/_version.py` if you use `setuptools-scm` or a cookiecutter that writes
  a generated version file.
