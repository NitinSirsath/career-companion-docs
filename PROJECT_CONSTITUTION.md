# Project Constitution — Career Companion

## Purpose

This document defines the governing principles, roles, responsibilities, and processes for the Career Companion project. It is the single source of truth for how the project is run.

---

## Project Overview

Career Companion is a production-oriented AI-powered job-search assistant that connects to Gmail, understands job-related emails, extracts useful information, tracks applications and communication, identifies required actions and follow-ups, and surfaces important events.

The project is built both as a useful product and as a serious engineering learning/portfolio project.

---

## Guiding Principles

- Production-minded architecture + incremental implementation.
- Protect the MVP from feature creep.
- Prefer explicit boundaries over premature abstraction.
- Validate behavior, not only code compilation.
- Diagnose before modifying code.
- Keep durable decisions documented.
- AI accelerates implementation; it does not replace engineering judgment.
- Privacy and security are first-class concerns.

### No Evidence, No Fix

Before modifying code for a bug, identify the first failing boundary using logs, database state, API responses, tests, or a reproducible case.

Do not perform speculative fixes, recursive patching, or unrelated refactoring.

---

## Roles & Responsibilities

### Product & Architecture — ChatGPT

Responsible for product thinking, architecture, technical decisions, trade-off analysis, mentoring, review, and challenging implementation decisions.

### Implementation — Gemini + Antigravity

Responsible for scoped implementation, code exploration, testing, and implementation reporting.

AI-generated output must be verified before it is considered correct.

### Project Execution — Linear

Linear is the execution source of truth for issues, sprints, acceptance criteria, dependencies, implementation notes, and completion state.

### Source Control & Technical Documentation — GitHub

GitHub is the source of truth for code, commits, reviews, and committed technical documentation.

### Project Knowledge — Notion

Notion stores durable project knowledge and reference documentation. It does not replace Linear or GitHub.

---

## Development Workflow

The standard workflow is:

**Define → Understand → Plan → Implement → Verify → Review → Commit → Close**

1. **Define** — requirement, scope, acceptance criteria.
2. **Understand** — read existing architecture and trace the affected boundary.
3. **Plan** — identify data flow, files/modules, contracts, risks, and verification.
4. **Implement** — one Linear issue at a time.
5. **Verify** — typecheck, lint, relevant tests, and actual behavior.
6. **Review** — regression, architecture, security/privacy, and scope checks.
7. **Commit** — focused commit with useful implementation notes.
8. **Close** — update Linear with implementation and verification evidence.

Full workflow reference:
https://app.notion.com/p/3e0a98abb650818097d6d72462c8ea6a?pvs=204

---

## AI-Assisted Development Rules

When using Antigravity/Gemini:

- Diagnose before modifying code.
- Do not guess the root cause from symptoms.
- Do not perform speculative refactors during debugging.
- Do not change multiple architectural boundaries unless required.
- When asked to diagnose, stop after diagnosis.
- Implement only the accepted fix.
- Verify actual behavior, not only compilation.
- If verification fails, diagnose the new failure boundary before making another change.

---

## Golden Path Verification

Critical product flows must have repeatable verification paths.

The primary Gmail/AI path is:

**Gmail → Sync → Email → Queue → Worker → AI → AI Result → Matching → Actions → UI**

Changes affecting this path should preserve or update its verification coverage.

---

## Linear Issue Quality Standard

Execution issues must be understandable without relying on ChatGPT history.

Include, where applicable:

- Objective / problem
- Current behavior
- Desired behavior
- Technical context
- Implementation scope
- Data/API/schema contracts
- Likely files/modules
- Security/privacy constraints
- Dependencies
- Tests and verification strategy
- Failure/idempotency behavior
- Acceptance criteria
- Explicit non-goals

---

## Definition of Done

A task is complete when:

- acceptance criteria are satisfied
- required typechecking/linting/tests pass
- actual behavior is verified
- architecture and security/privacy implications are considered
- required documentation is updated
- focused changes are committed
- useful implementation and verification notes are recorded in Linear

Code compiling alone is not sufficient evidence of completion.

---

## Decision-Making Process

1. Identify the problem and constraints.
2. Review existing architecture and documentation.
3. Evaluate the smallest appropriate solution.
4. Record durable architectural/product decisions.
5. Implement incrementally.
6. Verify behavior.
7. Update the relevant source of truth.

Do not introduce infrastructure, libraries, abstractions, or process solely because they appear impressive.

---

## Change Management

Changes to locked architecture, product scope, security/privacy boundaries, or development process require explicit review before implementation.

A change should identify:

- what is changing
- why it is changing
- affected boundaries
- migration/regression risk
- verification plan

---

## Document Versioning

| Version | Date | Author | Notes |
|---------|------|--------|-------|
| 0.2 | 2026-09-19 | Nitin Sirsath | Added AI-assisted development workflow, evidence-based debugging, golden-path verification, and execution standards |
| 0.1 | _TBD_ | — | Initial skeleton |
