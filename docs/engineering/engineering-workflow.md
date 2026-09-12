# Engineering Workflow

**Status:** Locked  
**Scope:** COM-7 — Define Engineering Workflow

## 1. Purpose

This document defines how Career Companion is developed, reviewed, tested, documented, and shipped. It establishes a lightweight production-minded workflow without adding process that is unnecessary for a solo project.

The workflow complements the architecture defined in COM-6; it does not redefine product, domain, architecture, API, AI, or infrastructure decisions.

## 2. Source-of-Truth Responsibilities

- **Linear** — execution source of truth for issues, sprints, status, acceptance criteria, and implementation notes.
- **GitHub** — source control and authoritative committed technical documentation.
- **Notion** — project knowledge and durable documentation/reference; it does not replace Linear or GitHub.
- **ChatGPT** — product, architecture, technical decision, and mentoring partner.
- **Antigravity** — primary implementation environment and AI-assisted coding environment.

## 3. Git Workflow

### Main branch

`main` is the primary integration branch and should remain in a releasable state.

### Working branches

Create a short-lived branch from `main` for meaningful work.

Recommended naming:

- `feat/<short-description>`
- `fix/<short-description>`
- `refactor/<short-description>`
- `docs/<short-description>`
- `test/<short-description>`
- `chore/<short-description>`

Branches should correspond to a Linear issue when the work is issue-driven.

### Branch lifecycle

1. Start from the latest `main`.
2. Implement the scoped change.
3. Validate locally.
4. Push the branch.
5. Open a pull request when the change is meaningful.
6. Review the change.
7. Merge to `main`.
8. Record completion/implementation notes in Linear.
9. Delete the short-lived branch when no longer needed.

Small, low-risk documentation-only changes may be committed directly to `main` when a PR would add unnecessary overhead.

## 4. Commit Conventions

Use Conventional Commits:

- `feat:` — new functionality
- `fix:` — bug fix
- `refactor:` — code restructuring without behavior change
- `docs:` — documentation changes
- `test:` — test changes
- `chore:` — tooling, configuration, or maintenance

Commits should be focused and describe the actual change. Avoid mixing unrelated work into one commit.

Example:

`feat: add gmail connection callback`

## 5. Linear ↔ GitHub Workflow

The normal implementation path is:

**Linear issue → branch → commits → pull request → merge → Linear completion notes → issue completion**

Linear remains the execution record. GitHub provides the code history and review history.

### Linear synchronization during implementation

Linear should not only be updated when an issue is closed. When a commit or implementation step introduces **substantial changes to the scope, architecture, behavior, acceptance criteria, implementation approach, or technical decisions of a Linear issue**, the issue should be updated promptly with a meaningful implementation comment.

The comment should capture the relevant change, why it was made, and any important consequences or follow-up work. The issue status should also be updated when the work materially changes its execution state.

Small, routine commits that do not materially change the issue do not require a separate Linear comment.

The goal is to keep Linear useful as the execution record without duplicating every Git commit.

Implementation notes in Linear should capture useful context such as:

- what was implemented
- important technical decisions
- deviations from the original approach
- notable limitations
- testing performed
- follow-up work, if any

These notes should make the issue understandable later without depending on chat history.

## 6. Pull Request Workflow

A pull request should be used for meaningful changes that benefit from explicit review.

PR descriptions should include, as appropriate:

- what changed
- why it changed
- relevant Linear issue
- important implementation details
- testing/validation performed
- known limitations or follow-ups

Review should verify:

- acceptance criteria are satisfied
- behavior matches the intended design
- architecture boundaries are respected
- security and privacy implications are considered
- tests are appropriate
- documentation is updated when the change creates durable knowledge

AI-assisted review is encouraged, but the developer remains responsible for the final decision and merge. A solo project does not require pretending that an AI review is a second human reviewer.

## 7. Definition of Done

A change is considered done when applicable requirements are satisfied and:

- acceptance criteria are met
- TypeScript type checking passes
- linting passes
- relevant tests pass
- the affected application/package builds successfully
- security/privacy implications have been considered
- required documentation is updated
- the change has been reviewed appropriately
- the change is merged into `main`
- useful implementation notes are recorded in Linear

Not every item requires a separate ceremony. The level of validation should match the risk and scope of the change.

## 8. CI/CD Workflow

The intended CI/CD flow is:

**Install → Typecheck → Lint → Tests → Build → Review/Merge → Deploy**

The deployment architecture is defined by COM-6. COM-7 defines only the engineering workflow around it.

The planned GitHub Actions deployment process is:

1. Validate the repository.
2. Build the application and Docker images.
3. Publish images to ECR.
4. Run database migrations using the deployment migration task.
5. Deploy the API service to ECS Fargate.
6. Deploy the Worker service to ECS Fargate.
7. Sync the frontend build to S3.
8. Invalidate CloudFront when required.

Migrations must remain backward-compatible with the application deployment strategy, especially when changes are additive or require staged rollout.

Actual CI/CD implementation is deferred until the application exists and there is something meaningful to validate and deploy.

## 9. Testing Expectations

Testing should follow risk rather than arbitrary coverage targets.

Priorities:

- unit tests for important domain logic
- integration tests for persistence and service boundaries
- API tests for critical request/response behavior
- AI workflow tests using deterministic fixtures/mocks where practical
- end-to-end tests for critical user flows once the application is sufficiently developed

External services such as Gmail and Gemini should not make the normal test suite dependent on live network calls.

## 10. Documentation Rules

Document durable knowledge, not every implementation detail.

- **Architecture change** → update the relevant architecture documentation.
- **Product behavior/user flow change** → update product/user-flow documentation.
- **Durable technical decision** → document the decision and rationale.
- **Normal implementation detail** → keep it in code, PR context, or Linear implementation notes; do not create unnecessary documentation.

Not every Linear issue needs a Notion page. Not every GitHub commit needs a Notion update.

Notion should be updated when a decision or piece of knowledge is useful beyond the immediate implementation task.

## 11. AI-Assisted Development

AI tools may be used for implementation, review, exploration, and documentation, but generated output is not automatically considered correct.

The developer remains responsible for:

- understanding the resulting code
- validating behavior
- checking architectural consistency
- reviewing security/privacy implications
- running the required validation
- making the final merge decision

AI should accelerate engineering work, not replace engineering judgment.

## 12. Scope and Process Discipline

Career Companion follows a production-minded but incremental process.

Do not introduce process, infrastructure, libraries, or abstractions solely because they appear impressive.

Prefer:

- clear boundaries
- small changes
- explicit decisions
- automated validation where valuable
- reversible implementation choices
- documentation for durable knowledge

Avoid:

- unnecessary ceremonies
- duplicate tracking systems
- speculative infrastructure
- process that does not reduce meaningful risk

## 13. Development Lifecycle

The overall product development lifecycle is:

**Problem → Requirements → User Flows → Architecture → Data Model → API/AI/Security Design → Linear Implementation Planning → GitHub Implementation → Review → Linear Completion Notes → Notion Knowledge Update when warranted**

This workflow is intentionally lightweight and should evolve as the project grows.
## 14. Gmail Integration Decisions (Sprint 2)

*   **Scope:** `gmail.readonly` is the exclusively selected scope. Other broader scopes (like full access) were rejected to adhere to least-privilege principles, while narrower scopes (like metadata-only) were rejected because we will eventually need to fetch full bodies for AI processing in Sprint 3.
*   **Encryption:** Per-user OAuth tokens (access and refresh) are encrypted using AES-256-GCM before database storage. They are never logged or exposed in API payloads.
*   **Data Minimization:** Raw email bodies and snippets are explicitly NOT persisted in the database, reducing privacy risks.

## Background Queue (pg-boss) Reset

The backend uses `pg-boss` for background job processing. It automatically provisions a `pgboss` schema in PostgreSQL.

If you encounter stuck jobs, schema mismatch errors, or wish to completely purge the queue state in local development, you can drop the schema. `pg-boss` will automatically recreate it on the next backend startup.

```sql
DROP SCHEMA pgboss CASCADE;
```

## 15. Ticket Quality Standard

Starting from Sprint 3.6, all Linear execution tickets must adhere to a new minimum quality standard to ensure they are actionable and clear.

A ticket should be executable from Linear without requiring the developer to reconstruct decisions from ChatGPT history. 

Execution tickets must include:
*   **Objective/Problem:** Clear statement of what needs to be solved.
*   **Technical Context/Current Architecture:** How this fits into the existing system.
*   **Explicit Implementation Scope:** What is being built.
*   **Data/API/Schema Contracts:** Defined inputs, outputs, and schema changes.
*   **Likely Files/Modules:** Where the changes are expected to occur.
*   **Security/Privacy Constraints:** Rules for handling sensitive data, logs, and isolation.
*   **Dependency Chain:** Prerequisites or downstream effects.
*   **Tests and Verification Strategy:** How the change will be validated.
*   **Idempotency/Failure Behavior:** How the system handles retries and failures, where applicable.
*   **Acceptance Criteria:** A clear checklist defining when the task is complete.
*   **Explicit Non-Goals:** What is intentionally deferred or out of scope.
