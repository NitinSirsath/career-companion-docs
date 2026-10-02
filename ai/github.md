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

> **STRICT WARNING (Environment Specific):** 
> The development workflow requires strict adherence to Pull Request rules **ONLY when developing on the personal PC**. When developing on the office PC, do not enforce this PR process unless explicitly directed.

When on the **personal PC**:
- Development must NOT happen directly on the `main` branch.
- Every Linear issue must have its own dedicated feature branch (e.g., `feature/COM-123-short-desc`).
- Once development and local verification are complete, you MUST open a Pull Request (PR) for the branch.
- A PR must be reviewed and pass CI (if configured) before being merged into `main`. Do not push directly to `main`.
- Merge the PR only after the associated Linear ticket is ready to move to `Done` or `In Review`.

---

## Branch Protection Rules

> _To be defined. This section will describe which branches are protected and what rules apply._

---

## Integration with Linear

> _To be defined. This section will describe how GitHub branches and PRs are linked to Linear issues._

---

## CI/CD

> _To be defined. This section will describe the CI/CD pipeline and what runs on each push or PR._
