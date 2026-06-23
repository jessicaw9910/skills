# skills

A personal, repo-agnostic collection of [Claude Code](https://claude.com/claude-code)
skills — broadly applicable conventions extracted from my projects for reuse
across future repositories.

## Contents

Skills live under `.claude/skills/<name>/SKILL.md`:

| Skill | What it covers |
|-------|----------------|
| [`plotting`](.claude/skills/plotting/SKILL.md) | matplotlib/seaborn conventions — Arial font, Nature/NPG palette, fully-opaque colors for PowerPoint/PDF export, spine removal, high-DPI saving |
| [`git`](.claude/skills/git/SKILL.md) | branching, scoped imperative commits, Claude attribution, and PR-template handling |
| [`formatting`](.claude/skills/formatting/SKILL.md) | Python style — import grouping, NumPy-style docstrings, trailing-docstring constants, lowercase comments, double quotes |
| [`tests`](.claude/skills/tests/SKILL.md) | pytest conventions — network marker, token-gated skips, session/module fixtures, xdist safety |
| [`ci`](.claude/skills/ci/SKILL.md) | GitHub Actions + pre-commit — path-filtered per-package workflows, conda/micromamba env, OS/Python matrix, Codecov flags |
| [`docs`](.claude/skills/docs/SKILL.md) | building/regenerating Sphinx API docs in MolSSI cookiecutter-cms projects — autosummary + autodoc + napoleon (NumPy docstrings), recursive templates, `make html`, Read the Docs |
| [`packaging`](.claude/skills/packaging/SKILL.md) | setuptools/pyproject distribution layout — dynamic version, extras, console scripts, and the monorepo pattern: many sub-distributions sharing a PEP 420 `mkt` namespace (no top-level `__init__.py`) with `graft`-ing MANIFEST.in |
| [`packaging-data`](.claude/skills/packaging-data/SKILL.md) | shipping non-`.py` files — `MANIFEST.in graft` (sdist) vs `package-data` (wheel), the dotted-package-name key gotcha, and pruning caches `graft` sweeps in |
| [`codecov`](.claude/skills/codecov/SKILL.md) | `.codecov.yml` for path-filtered monorepo CI — per-flag carryforward, project/patch status targets, per-flag target overrides, ignore paths, PR comment layout |
| [`gitignore`](.claude/skills/gitignore/SKILL.md) | generating a Python `.gitignore` from the MolSSI cookiecutter base plus personal add-ons |
| [`versioning`](.claude/skills/versioning/SKILL.md) | git-tag-derived package versioning with versioningit — dynamic version, `importlib.metadata` at runtime, PEP 440 local versions, releasing by tag, monorepo tag namespaces |

## Usage

Each `SKILL.md` has YAML frontmatter (`name`, `description`) that tells Claude
Code when the skill is relevant. There are two ways to consume these skills in
another project.

### Option A — copy / vendor

Copy a skill directory into the project's `.claude/skills/`. The content is
always present in context when the skill triggers, works offline, and is shared
with collaborators via the repo. The trade-off is that it's a **copy**: it
drifts from this repo until you re-sync it by hand.

### Option B — reference (fetch on demand)

Keep a thin local skill in the project's `.claude/skills/` that holds only the
repo-specific additions and tells Claude to `WebFetch` the canonical version
here, e.g.:

```
https://raw.githubusercontent.com/jessicaw9910/skills/main/.claude/skills/<name>/SKILL.md
```

This keeps a single source of truth. See `missense-kinase-toolkit` and
`mkt_impact` for working examples.

**Caveats (all attendant):**

- **Claude Code does not follow links in a `SKILL.md` automatically.** The local
  file must *explicitly* instruct a `WebFetch`, or the baseline never loads.
- **Fetch-on-demand isn't guaranteed.** Claude may apply only the local
  repo-specific notes and skip the fetch. Have the local skill instruct it to
  **notify you if the fetch fails** rather than silently proceeding.
- **Needs network + the `WebFetch` tool permitted**, and adds some latency.
- **URLs resolve only against what's committed.** The `main` raw URLs 404 until
  the referenced skill is merged to `main` (pin a branch/tag/commit SHA in the
  URL if you need to reference unmerged or versioned content).
- **No version pinning by default.** Referencing `main` means the project tracks
  whatever `main` currently says; use a tag or commit SHA in the URL to pin.
- **Collaborators** get the local thin skill from the repo, but the fetched
  baseline depends on this repo staying public and reachable.
