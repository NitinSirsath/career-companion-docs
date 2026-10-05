# Development Modes

## Purpose

Career Companion is developed across different machines and development environments. This document defines the rules for each environment so that ownership, GitHub access, repository state, AI collaboration, and database boundaries remain clear.

The goal is to avoid accidental changes from the wrong GitHub account, duplicated local repositories, unclear ownership, or unsafe assumptions about infrastructure.

---

## Global GitHub Rules

These rules apply to all development modes unless explicitly stated otherwise.

1. **Never develop directly on `main`.**
   - Every feature, bug fix, refactor, documentation change, or significant maintenance task must start from a separate branch.
   - Branch from the current intended base branch before making changes.
   - Use a descriptive branch name, for example:
     - `feature/<name>`
     - `fix/<name>`
     - `docs/<name>`
     - `chore/<name>`

2. **Pull requests are the normal integration path.**
   - Changes should be reviewed before merging.
   - The PR should explain what changed, why it changed, and how it was verified.

3. **Do not mix unrelated work in one branch or PR.**
   - Keep changes focused and traceable to the relevant task or issue.

4. **Do not overwrite or force-push shared branches unless explicitly required and understood.**

5. **Never commit secrets or sensitive runtime data.**
   - No API keys, OAuth tokens, Gmail tokens, database credentials, session secrets, private MCP tokens, or real user data.

6. **GitHub represents repository state, not local environment state.**
   - Local databases, local environment variables, generated files, machine-specific configuration, and credentials must not be treated as repository state.

---

# 1. Personal — Full Development Mode

### Environment

The personal development machine is the primary unrestricted development environment for Career Companion.

### Rules

- The developer owns and controls the environment.
- Antigravity and other approved development tools may be used normally.
- The repositories may be cloned and worked on locally.
- GitHub access may be used normally with the personal GitHub account.
- Local development databases may be used for implementation and testing.
- Changes should still follow the global branch and PR rules.
- This environment is the preferred place for full development, integration, debugging, and final verification.

### Responsibility

The personal environment is responsible for maintaining a trustworthy working copy of Career Companion and for safely synchronizing completed work with GitHub.

---

# 2. Office Laptop — Local Repository Mode

### Environment

The office laptop is used for development while authenticated to the company's GitHub environment.

Career Companion repositories may be downloaded locally for development, but this machine does **not** have permission to push Career Companion changes to the personal Career Companion GitHub repositories.

### Rules

- Clone/download the required Career Companion repositories locally.
- Work against the local repository normally.
- Do **not** push Career Companion changes using the company's GitHub credentials.
- Do **not** change GitHub remotes or authentication merely to bypass the company's access restrictions.
- Use Codex, Fable, or other approved local development tools as appropriate.
- Local tickets/issues may be used when work exists in GitHub but has not been synchronized to Linear.
- Keep implementation work in a separate local branch.
- When work is complete, transfer/synchronize the work to the personal development environment using an intentional and auditable process.
- The personal environment remains responsible for the final GitHub push/PR when the office account cannot perform it.

### Important distinction

The office laptop may contain a working copy of the repository, but **the office laptop is not the authority for Career Companion's GitHub history**.

Do not attempt to work around company GitHub permissions.

---

# 3. Office Laptop — Prompt-Only / Remote Repository Mode

### Environment

This is the preferred mode when working on Career Companion from the office without downloading the repository locally.

The office laptop does not contain a local Career Companion repository.

The personal ChatGPT account acts as the coordination layer between the developer and repository-aware coding agents such as Codex or Fable.

### Workflow

The standard workflow is:

```
Developer
   ↓
Personal ChatGPT
   ↓
Codex / Fable
   ↓
Direct analysis of Career Companion GitHub repositories
   ↓
Detailed investigation + implementation plan/prompt
   ↓
Personal ChatGPT
   ↓
New GitHub feature branch
   ↓
Implementation
   ↓
Verification
   ↓
Pull Request
   ↓
Codex / Fable review
   ↓
Corrections if required
   ↓
Final review
```

### Step 1 — Understand the problem

The developer explains the requested feature, bug, improvement, or question to personal ChatGPT.

Personal ChatGPT should not immediately instruct itself to modify code when the repository state or technical cause is unclear.

### Step 2 — Repository investigation

Codex/Fable should directly inspect the relevant Career Companion repositories on GitHub.

They should determine:

- current implementation;
- relevant frontend/backend modules;
- database models and relationships;
- API contracts;
- existing tests;
- migrations;
- configuration boundaries;
- related documentation;
- existing implementation patterns;
- likely root cause;
- risks and regression points.

### Step 3 — Produce an implementation specification

Codex/Fable should provide a detailed solution prompt/specification for personal ChatGPT.

The specification should describe:

- root cause;
- files/components/services involved;
- required changes;
- data-model impact;
- API impact;
- frontend impact;
- testing requirements;
- migration requirements, if any;
- security/privacy considerations;
- edge cases;
- acceptance criteria.

### Step 4 — Implementation through GitHub

Personal ChatGPT may then modify the repository through GitHub.

**It must always:**

1. start from the intended base branch;
2. create a separate feature/fix/docs branch;
3. make changes only on that branch;
4. verify the changes as far as the available environment permits;
5. create a pull request instead of modifying `main` directly.

### Step 5 — Independent review

Codex/Fable should review the resulting branch/PR directly from GitHub.

Because the personal ChatGPT Go model may make implementation mistakes, repository-aware review is mandatory for non-trivial changes.

Review should check:

- correctness;
- architecture;
- regressions;
- security;
- privacy;
- database consistency;
- API compatibility;
- frontend/backend integration;
- tests;
- adherence to existing project conventions.

If problems are found, personal ChatGPT applies the corrections on the same feature branch and the review cycle repeats.

### Database boundary

**The database schema is repository-owned and must be understandable from GitHub. Database state/data is environment-owned and must never be assumed to be available from GitHub.**

Codex/Fable should understand the database structure using repository artifacts such as:

- Prisma schema;
- Prisma migrations;
- Prisma enums and relations;
- seed definitions;
- backend/domain code;
- API code;
- database-related tests;
- relevant documentation.

They must **not** assume that the developer's local PostgreSQL database is accessible merely because the repository is accessible.

Actual database records, local credentials, local environment variables, and machine-specific database state remain environment-specific.

If a task genuinely requires access to a shared database, that database must be explicitly provisioned and access must be deliberately granted. Never expose a personal or production database simply to make repository analysis easier.

---

# AI Collaboration Rules

## Repository-aware agents

Codex/Fable should be used for deep repository investigation, architectural reasoning, implementation review, and identifying inconsistencies between code, documentation, and project requirements.

They should inspect the actual repository before making assumptions.

## Personal ChatGPT

Personal ChatGPT acts as the implementation/orchestration layer in Prompt-Only Mode.

It should:

- translate reviewed implementation requirements into repository changes;
- make changes through the approved GitHub workflow;
- keep changes scoped;
- avoid inventing repository structure;
- avoid making architectural changes without sufficient evidence;
- create branches before implementation;
- create/update PRs rather than bypassing review.

## No blind AI-to-AI handoff

A generated prompt is not automatically correct.

When Codex/Fable and personal ChatGPT disagree, the repository state, project requirements, tests, and explicit architectural decisions should be used to resolve the disagreement.

---

# Source-of-Truth Rules

Different systems have different responsibilities.

| System | Primary responsibility |
| --- | --- |
| GitHub | Source control and actual repository/code state |
| Linear | Project execution, planning, sprint work, and issue tracking |
| Local repository | Development workspace and local implementation state |
| Local database | Development data/state for that environment |
| Prisma schema/migrations | Repository-owned database structure |
| Documentation | Architecture, decisions, workflows, and project knowledge |
| Personal ChatGPT | Product/engineering orchestration and implementation assistance |
| Codex/Fable | Repository analysis, implementation assistance, and review |

No single system should be assumed to contain all project state.

---

# Environment Safety Rules

1. Never assume another machine has the same repository state.
2. Never assume a local database exists on another machine.
3. Never assume local environment variables exist outside their originating environment.
4. Never transfer credentials as part of normal development synchronization.
5. Never commit real Gmail data or sensitive personal information.
6. Never bypass company GitHub permissions from an office machine.
7. Always verify the target repository and branch before making remote changes.
8. Prefer small, reviewable changes over large unreviewed AI-generated changes.
9. When repository state is uncertain, investigate before modifying anything.
10. When in doubt about ownership or access, stop and resolve the boundary rather than working around it.

---

# Summary

Career Companion supports three development modes:

1. **Personal — Full Development Mode**
   - Full local development and GitHub access.
   - Primary unrestricted development environment.

2. **Office Laptop — Local Repository Mode**
   - Repository is downloaded locally.
   - Development happens locally.
   - Company GitHub credentials must not be used to push Career Companion changes.
   - Completed work is intentionally synchronized to the personal environment.

3. **Office Laptop — Prompt-Only / Remote Repository Mode**
   - No local repository.
   - Personal ChatGPT coordinates implementation.
   - Codex/Fable inspect GitHub directly and produce detailed technical guidance.
   - Personal ChatGPT implements through a new branch and PR.
   - Codex/Fable review the PR until the change is ready.

The core principle is:

**Think and review deeply → create a branch → implement → verify → review → merge.**

GitHub repository state, project-management state, local development state, and database state must always be treated as separate concerns.
