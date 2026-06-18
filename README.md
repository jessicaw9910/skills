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
| [`gitignore`](.claude/skills/gitignore/SKILL.md) | generating a Python `.gitignore` from the MolSSI cookiecutter base plus personal add-ons |

## Usage

Copy a skill directory into another project's `.claude/skills/` to make it
available there, or reference these as a starting point when setting up a new
repo. Each `SKILL.md` has YAML frontmatter (`name`, `description`) that tells
Claude Code when the skill is relevant.
