---
name: git
description: >-
  Personal git and pull-request conventions (branching, scoped imperative
  commits, Claude attribution, PR template). Use when creating commits,
  branches, or pull requests.
---

# Git & pull-request conventions

## Branching

- Don't commit directly to the default branch (`main`). Branch first.
- Only commit or push when explicitly asked.

## Commits

- Keep commits scoped and message subjects in the imperative
  (e.g. "Add transparency regression test", not "Added…").
- In a mono repo, scope each commit to one sub-package where practical
  (CI is often path-filtered per sub-package).
- Use the `gh` CLI for GitHub operations (PRs, issues, API).
- End commit messages with:

  ```
  Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
  ```

## Pull requests

- **If `.github/PULL_REQUEST_TEMPLATE.md` exists**, use it: fill in every
  section and leave the Status box reflecting the true state.
- **If it does not exist**, create `.github/PULL_REQUEST_TEMPLATE.md` with the
  body below, then use that as the PR body.

```markdown
## Description
Provide a brief description of the PR's purpose here.

## Todos
Notable points that this PR has either accomplished or will accomplish.
  - [ ] TODO 1

## Questions
- [ ] Question1

## Status
- [ ] Ready to go
```

The sections are:
- **Description** — the purpose of the PR.
- **Todos** — checklist of what was accomplished.
- **Questions** — open items / things to revisit.
- **Status** — readiness checkbox (only check when actually ready).

- End PR bodies with:

  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  ```
