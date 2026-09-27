# Sprint 6 local conversion package

Status: reconciled 2026-09-27; local copy instructions only. No Linear ticket identity, state, capacity, labels or approval is assumed. This reconciliation performs no external ticket or PR action. The office environment's local repository is the source of Sprint 6 requirements.

## Canonical copy index

Copy each complete ticket body rather than a shortened replacement. All five contain sections 1–21, including objective, scope/requirements, architectural constraints, API/data/job/privacy boundaries, verification, acceptance, dependencies, recovery, documentation and definition of done. Keep open checkboxes and unresolved decisions visible.

| Canonical local ID / title | Description and acceptance source |
| --- | --- |
| [S6-01 — Establish application status semantics and manual user correction](S6-01.md) | Entire ticket; acceptance §15, dependencies §17, definition of done §21 |
| [S6-02 — Record source email evidence for timeline events and AI actions](S6-02.md) | Entire ticket; acceptance §15, dependencies §17, definition of done §21 |
| [S6-03 — Verify and harden AI worker delivery and application matching concurrency](S6-03.md) | Entire ticket; acceptance §15, dependencies §17, definition of done §21 |
| [S6-04 — Deliver the application status correction and evidence-review workflow](S6-04.md) | Entire ticket; acceptance §15, dependencies §17, definition of done §21 |
| [S6-05 — Final architecture and data-preservation verification for Sprint 6](S6-05.md) | Entire ticket; acceptance §15, dependencies §17, definition of done §21 |

## Dependencies and shared references

Use the [authoritative overview mapping/graph](README.md#f-ticket-list) and ticket section 17. S6-01 owns both reads and status PATCH/migration; S6-02 owns evidence; S6-03 owns worker/matching verification; S6-04 consumes S6-01/02; S6-05 accepts all four. S6-03 can investigate alongside S6-01 but final manual-revision checks need S6-01. It does not supply a new frontend API.

Attach the [overview](README.md), [assessment](architecture-review.md), [runbook](verification-runbook.md) and [reconciliation/review](planning-review.md). Section 20 of each ticket and overview section J identify implementation documentation updates. The shared runbook is the command/guard source. Do not duplicate API semantics into independently maintained conversion summaries.

## Pending decisions and conversion boundary

D1 action-response scope and inherited D2 sign-off remain explicit in [overview section L](README.md#l-open-decisions-and-readiness). Retain the live Gmail/original-data entry gate while acknowledging implemented, uncommitted Sprint 5 engineering. Do not convert pending approval/evidence into an implementation-ready or Done status.

If external issue creation is separately authorized later, verify the actual team/project, labels, priorities and estimation scale. Copy canonical titles verbatim and all ticket requirements; use verified repository links accessible to the team, not machine-local paths. Resolve D1 in the local contract before publishing any decided action scope. Preserve suggested priorities only as proposals. The historical 19-point total and old ticket estimates do not apply to this allocation; re-estimate without inventing capacity.

Verify Sprint 5 identities before linking dependencies: local COM cross-references are unconfirmed and historical identifiers have collisions. Replace local dependency IDs only with IDs/URLs actually returned or verified. Do not invent Sprint 6 COM mappings, remote ticket state or prior creation status.

## Completion bookkeeping

Link source identity including Sprint 5 working changes, then implementation commits, actual evidence and approval records when available. Every acceptance checkbox remains open until its evidence exists. Documentation reconciliation alone does not implement Sprint 6 or authorize issue creation, messaging, PRs, migrations or deployment.
