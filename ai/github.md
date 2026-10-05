# GitHub — AI Collaboration Reference

## Purpose

This document defines how GitHub is used within Career Companion. It describes the repository structure, branching strategy, pull request conventions, and the relationship between GitHub and Linear.

---

## Repositories

> _To be defined. This section will list all repositories, their purpose, and their relationship to each other._

---

## Branching Strategy

> _To be defined. This section will describe the branching model (e.g. main, develop, feature branches) and naming conventions._

---

## Commit Conventions

> _To be defined. This section will define the commit message format (e.g. Conventional Commits) and enforce consistency._

---

## Pull Request Process

The project-wide rule is that repository changes are integrated into `main` through a reviewed pull request. This applies in every development mode.

- Development must NOT happen directly on the `main` branch.
- Use the project's conventional branch prefixes: `feat/`, `fix/`, `refactor/`, `docs/`, `test/`, or `chore/`.
- Every issue-driven change should have a corresponding Linear issue before work begins.
- Office-local work may be developed on a local branch when remote mutation is prohibited, but the work must be transferred to an authorized environment and integrated through a PR before reaching `main`.
- Company GitHub credentials must never perform Career Companion remote mutations, including API commits, branch mutations, PR mutations, reviews, merges, or force-pushes.
- Never install personal GitHub credentials on an office machine to bypass company restrictions.
- Before a remote mutation, verify the actual authenticated GitHub user/app, repository, target branch, and permitted operation.
- A PR must be reviewed and required verification must pass before merge. Checks that cannot run remain **unverified**; they are not treated as passes.
- The developer owns the final merge decision.

For Prompt-Only / Remote Repository Mode, repository read access, write access, command execution, and database access are separate capabilities. If the required capability is unavailable, stop at investigation/specification or hand off to an approved environment rather than claiming implementation or verification.

## Branch Protection Rules

The repository workflow requires PR-based integration even where GitHub branch protection is not technically enabled. Protection settings should enforce the documented policy when the repository configuration supports them.

---

## Integration with Linear

> _To be defined. This section will describe how GitHub branches and PRs are linked to Linear issues._

---

## CI/CD

> _To be defined. This section will describe the CI/CD pipeline and what runs on each push or PR._
