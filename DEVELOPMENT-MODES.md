# Development Modes

## Purpose

Career Companion is developed across different machines and development environments. This document defines the rules for each environment so that ownership, GitHub access, repository state, AI collaboration, and database boundaries remain clear.

The goal is to avoid accidental changes from the wrong GitHub account, duplicated local repositories, unclear ownership, or unsafe assumptions about infrastructure.

---

## Global GitHub Rules

These rules apply to all development modes. Where an older document conflicts with these environment, account, or integration rules, the precedence defined below applies.

1. **Never develop directly on `main`.**
   - Every feature, bug fix, refactor, documentation change, test change, or maintenance task that changes repository state must start from a separate branch.
   - Branch from the current intended base branch before making changes. The default base is the current `main`, unless a documented stacked-branch or release workflow says otherwise.
   - Use the project's conventional branch prefixes: `feat/`, `fix/`, `refactor/`, `docs/`, `test/`, or `chore/`.

2. **All repository changes must reach `main` through a reviewed pull request.**
   - Direct commits, pushes, API writes, or force-pushes to `main` are prohibited by this workflow.
   - Office-local work may defer PR creation until the work is transferred to an authorized personal environment, but it may not bypass the integration rule.
   - The PR must identify what changed, why it changed, the exact verification performed, and any remaining unverified checks.

3. **Do not mix unrelated work in one branch or PR.**
   - Keep changes focused and traceable to the relevant Linear issue or documented work item.

4. **Do not overwrite or force-push shared branches unless explicitly required, authorized, and understood.**

5. **AI-generated changes require verification and appropriate review regardless of tool or model.**
   - This applies to ChatGPT, Codex, Fable, Gemini, Antigravity, Claude, and future AI tools.
   - The developer remains responsible for the final merge decision.

6. **Never commit secrets or sensitive runtime data.**
   - No API keys, OAuth tokens, Gmail tokens, database credentials, session secrets, private MCP tokens, or real user data.

7. **GitHub represents published repository state, not local environment state.**
   - Local databases, local environment variables, credentials, and machine-specific runtime state are environment-owned.
   - Generated files are tracked or ignored according to the repository's own policy; do not assume that all generated artifacts are excluded.

### Policy precedence

For conflicts between project documents:

1. **Project Constitution** governs project-wide accountability, architecture, security/privacy boundaries, and completion standards.
2. **Engineering Workflow** governs the general engineering lifecycle and verification expectations.
3. **Development Modes** governs environment selection, account boundaries, handoffs, and mode-specific implementation responsibilities.
4. **GitHub/Linear AI references** provide operational guidance within those boundaries.

When a conflict is discovered, update the conflicting documents rather than choosing the rule that gives an agent more access or less verification.

---

# 1. Personal — Full Development Mode

### Environment

The personal development machine is the primary authorized development environment for Career Companion. It is the preferred environment for full local development, integration, debugging, and final verification.

### Rules

- The developer owns and controls the environment.
- Gemini + Antigravity remains the primary implementation environment under the project constitution; other approved tools may assist with implementation, review, or orchestration.
- Access to GitHub, a terminal, a database, or external providers must be verified rather than assumed.
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
- **Company credentials must never perform any remote mutation of Career Companion repositories**, including pushes, API commits, branch creation/deletion, pull-request creation/update/merge, review mutation, or force-pushes, regardless of whether GitHub technically permits the action.
- Do **not** change GitHub remotes or authentication merely to bypass company restrictions.
- Do **not** install personal GitHub credentials, tokens, SSH keys, or equivalent authentication on the office machine to bypass those restrictions.
- Office development must also comply with applicable company policy.
- Use Codex, Fable, or other approved local development tools as appropriate.
- Work must have a Linear issue before it starts. If Linear is temporarily unavailable or the work is genuinely offline, use a clearly marked temporary local work identifier only as an exception and reconcile it to the real Linear issue before completion.
- A local branch or local commit is not published project state until it is transferred and pushed from an authorized environment.
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

### Capability and identity preflight

Before any repository mutation or claim of verification, record or confirm:

- development mode and execution host;
- repository and intended base revision;
- authenticated GitHub user/app actually used by the operation;
- allowed repository operations;
- terminal/command execution availability;
- dependency installation availability;
- database access, if required;
- available test/build/browser verification capabilities.

Read access, write access, command execution, and database access are separate capabilities. A model or tool name does not establish any of them.

If required capabilities are unavailable, stop at investigation/specification or hand off the unverified work to an approved environment. Do not claim implementation or verification that did not occur.

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

Personal ChatGPT may modify the repository through GitHub only when the capability/identity preflight confirms that the authorized personal GitHub identity/app has the required write access.

**It must always:**

1. start from the intended base branch;
2. create a separate feature/fix/docs branch;
3. make changes only on that branch;
4. verify the changes using the available approved environment and record the exact revision tested;
5. create a pull request instead of modifying `main` directly.
6. If a required check cannot run, record it as **unverified** rather than passing it by implication.

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

If problems are found, personal ChatGPT applies the corrections on the same feature branch and the review cycle repeats. Every material correction must be re-verified and the updated PR head must be reviewed again. Material disagreements that remain unresolved are escalated to the developer, who owns the final merge decision.

### Verification and merge gate

A PR is not ready to merge merely because a review was completed.

The implementation record must identify:

- the exact commit/PR head that was tested;
- commands and tools used;
- test, typecheck, lint, build, and behavioral results that were actually run;
- outstanding checks;
- checks that were unavailable or intentionally skipped, explicitly marked **unverified**;
- any known risks or limitations.

Required verification that cannot be run blocks merge until the developer explicitly accepts the risk under the project's completion rules. Corrections require relevant re-verification and review of the latest PR head.

Merge approval does not authorize deployment, production/live-data changes, paid AI-provider usage, external job submissions, or other separately controlled actions.

### Database boundary

**The database schema is repository-owned and must be understandable from GitHub. Database state/data is environment-owned and must never be assumed to be available from GitHub.**

Repository artifacts describe **intended/reproducible structure and migration history**; they do not prove that a particular environment has applied every migration or contains particular records.

Codex/Fable should understand database structure using repository artifacts such as:

- Prisma schema;
- Prisma migrations and migration history;
- Prisma enums and relations;
- raw SQL constraints, indexes, triggers, and other migration SQL;
- seed definitions;
- backend/domain code;
- API code;
- database-related tests;
- library-managed structures such as pg-boss;
- relevant documentation.

Seeds are initialization definitions, not proof that those records exist in a target environment. Applied migration state and existing records must be verified from the target environment when a task genuinely depends on them.

They must **not** assume that the developer's local PostgreSQL database is accessible merely because the repository is accessible.

Actual database records, local credentials, local environment variables, and machine-specific database state remain environment-specific.

If a task genuinely requires access to a shared database, that database must be explicitly provisioned and access deliberately granted. Use synthetic/non-production data by default. Never expose a personal or production database simply to make repository analysis or testing easier.

Automated tests must continue to use an isolated test database/runner. A shared development database must not be substituted for the existing test database guard or used to weaken its safety checks. Destructive database operations require explicit authorization and appropriate preservation/rollback measures.

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

### Multi-repository coordination

When a change spans frontend, backend, and/or docs repositories:

- record the compatible base/revision of each affected repository;
- link the related PRs and Linear issue;
- synchronize backend-owned API/contracts before frontend verification;
- record integration evidence against the compatible revisions;
- define merge/deployment ordering when one repository depends on another;
- do not treat independently passing repository checks as proof that the combined system is compatible.

### MCP boundary

Career Companion's application MCP interface remains a write-only submission interface as defined by ADR-0002. It must not be expanded into a general repository browser or database-inspection API. Development-tool integrations and their credentials are separate from that application boundary.

### Source-of-truth reconciliation

GitHub is authoritative for published repository revisions. Linear is authoritative for project execution state. Local branches, local tickets, scratchpads, and databases are not authoritative until reconciled to the appropriate source of truth.

The core principle is:

**Think and review deeply → create a branch → implement → verify → review → merge.**

GitHub repository state, project-management state, local development state, and database state must always be treated as separate concerns.
