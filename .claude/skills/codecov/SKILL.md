---
name: codecov
description: >-
  How I configure Codecov for path-filtered monorepo CI — the .codecov.yml file:
  per-flag carryforward (so packages that didn't run this push don't report as
  dropped coverage), project/patch status targets, per-flag target overrides,
  ignore paths, and the PR comment layout. Complements the upload step in the ci
  skill. Use when adding/tuning .codecov.yml, fixing red coverage on untouched
  packages, or wiring a new sub-package's flag.
---

# Codecov configuration (`.codecov.yml`)

The [[ci]] skill covers the **upload** half (the `codecov/codecov-action` step
with a per-package `flags:` value). This skill covers the **config** half — the
repo-root `.codecov.yml` that decides what's required and how flags behave.

## The carryforward rule (why this file exists)

With path-filtered per-package workflows ([[ci]]), most pushes run **only one**
sub-package's tests and upload **only that flag**. Without carryforward, Codecov
treats every other flag as 0 / dropped and paints the PR red for packages you
never touched. Fix it once, globally:

```yaml
flag_management:
  default_rules:
    carryforward: true            # reuse last known coverage for un-uploaded flags
  individual_flags_rules:
    - name: schema
      target: 90%                 # mature package — hold a hard floor
    - name: databases
      target: auto                # young package — compare against parent commit
```

`carryforward: true` is the single most important line here. `show_carryforward_flags: true` (below) makes the PR comment label which flags were carried vs freshly measured.

## Status checks — keep them un-noisy

```yaml
coverage:
  status:
    patch: false                  # don't gate on patch coverage (too noisy early)
    project:
      default:
        threshold: 50%            # allow this much drop before failing
```

- `project` = coverage of the whole package; `patch` = coverage of just the
  changed lines. Early in a project, `patch: false` + a loose `project.threshold`
  keeps CI from blocking on every PR; tighten as the suite matures.
- Per-flag `target: auto` compares to the parent commit (no fixed bar);
  `target: 90%` sets a fixed floor for a package you want to keep honest.

## Flag → path mapping

Flags are defined implicitly by the `flags:` value uploaded from CI. If you need
Codecov to know which files belong to a flag (for component views / fixed
carryforward), declare paths explicitly:

```yaml
flags:
  schema:
    paths:
      - missense_kinase_toolkit/schema
    carryforward: true
  databases:
    paths:
      - missense_kinase_toolkit/databases
    carryforward: true
```

`flag_management` (above) and an explicit `flags:` block are two ways to say
much the same thing — use `flag_management.default_rules` to set carryforward
once for all flags, and reach for explicit `flags:` only when you need per-flag
`paths`.

## Ignore + comment

```yaml
ignore:
  - missense_kinase_toolkit/databases/mkt/databases/ncbi.py   # e.g. dead/blocked code

comment:
  layout: "header, flags"
  require_changes: false
  show_carryforward_flags: true   # mark which flags were carried forward
```

Also `ignore:` the usual non-product code — tests, `docs/`, generated version
files — so coverage reflects shipped code.

## Adding a new sub-package's flag

1. In its CI workflow, set `flags: <name>` on the upload step and `--cov=mkt.<name>` ([[ci]]).
2. Add an `individual_flags_rules` entry here with a `target` (`auto` to start).
3. Carryforward is already inherited from `default_rules` — nothing else needed.
4. Add the per-package Codecov badge to the README ([[readme]]).
