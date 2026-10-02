# Linear — AI Collaboration Reference

## Purpose

This document defines how Linear is used as the source of truth for project execution within Career Companion. It describes the conventions, workflows, and rules for managing issues, epics, milestones, and sprints in Linear.

---

## Role in the Project

Linear is the **single source of truth for project execution**. All work must be tracked here before it is started. Nothing is built without a corresponding Linear issue.

---

## Structure Conventions

### Epics
> _To be defined. This section will describe how Epics are structured and named._

### Issues
> _To be defined. This section will describe the required fields, labels, and format for all issues._

### Milestones / Cycles
> _To be defined. This section will describe how sprints and cycles are managed._

---

## Issue Lifecycle

All issues must transition through the following states to accurately reflect progress. Do not skip states unless explicitly instructed, and do not leave issues in `Todo` if work has begun or is awaiting verification.

- **Todo / Backlog**: Work has not yet begun. Future tickets and unverified ideas live here.
- **In Progress**: Active development is occurring on the issue. Move the ticket here when you begin investigating or coding.
- **In Review**: Implementation is complete, but pending external verification (e.g., manual smoke tests, QA, stakeholder approval, or blocked Phase 0 tests like BYO AI and MCP). Move tickets here if you are waiting for the owner to perform manual checks.
- **Done**: The code is merged, verification is completely finished, and all acceptance criteria are provably met.
- **Canceled**: The ticket is no longer relevant, was a duplicate, or the decision was made not to pursue it.

*Agent instruction: Always update the issue state in Linear to match reality. If an implementation is blocked awaiting manual user review, move it to `In Review`.*

---

## Workflow Rules

> _To be defined. This section will enumerate the rules for how issues are assigned, prioritized, and updated._

---

## Integration with GitHub

> _To be defined. This section will describe how Linear issues link to GitHub branches and pull requests._

---

## Integration with AI Agents

> _To be defined. This section will describe how Gemini/Antigravity reference and update Linear issues during implementation._
