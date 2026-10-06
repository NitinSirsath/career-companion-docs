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

Since 2026-10-06, `main` in `career-companion-backend`, `career-companion-frontend` and `career-companion-docs` is protected by a GitHub ruleset named **Protect main**:

- No direct pushes to `main`, for anyone, including the owner and AI tools. Every change goes through a pull request.
- No force pushes and no deleting `main`.
- Required approvals: 0. GitHub does not allow approving your own PR, and AI tools open PRs as the owner, so a required approval would block every merge. Only the owner has write access, so only the owner can merge.
- Required checks before merge:
  - backend: `test` and `migration-lanes`;
  - frontend: `check` and `contract-drift`;
  - docs: none (no CI).
- The backend `AI eval` check is deliberately **not** required, so a used-up Gemini quota never blocks a merge.
- The bypass list is empty. In an emergency the owner turns the ruleset off for the shortest possible time.
- Secret push protection is on for all three repositories.

The ruleset only limits changes to `main`. Reading code, PRs and CI results is unaffected for every tool.

AI tools and connectors (ChatGPT, Codex, Claude and others) open **draft** PRs only and never merge. Check any claim an AI tool makes about a commit, branch or merge on GitHub itself before acting on it.

---

## Integration with Linear

> _To be defined. This section will describe how GitHub branches and PRs are linked to Linear issues._

---

## CI/CD

What runs today (no deployment pipeline yet):

| Repository | Check | When | Required |
| --- | --- | --- | --- |
| backend | `test` (typecheck, lint, build, migrations, tests) | every push and PR | yes |
| backend | `migration-lanes` | every push and PR | yes |
| backend | `AI eval` (Gemini, synthetic emails) | PRs that change `src/services/ai/`, `src/eval/ai/` or `src/contracts/aiCatalog.ts`; manual run | no |
| frontend | `check` (typecheck, lint, tests, build) | every push and PR | yes |
| frontend | `contract-drift` (frontend `src/contracts` equals backend `main`) | every push and PR | yes |

CodeRabbit (free for public repositories) posts review comments on new PRs. It is advisory and never a required check.

See [provider evaluation](../docs/ai/provider-evaluation.md#ci-evaluation-com-136) for how the AI eval works and what it costs.
