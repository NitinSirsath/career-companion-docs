# Sprint 9 — Daily job-search workspace

Status: **IMPLEMENTED LOCALLY**, verified 2026-10-03; real-use acceptance pending. OD-16 and OD-17 accepted for this sprint. Sprint 7/8 implementation is complete and is not reopened.

## Goal

On opening Career Companion, the owner can see due work across all stored actions, distinguish incomplete input coverage, resolve existing review needs, and find a known application.

This extends the existing dashboard, action controls and application list. No new AI capability, provider, migration, background job or external integration is planned.

## Tickets and order

| Order | Ticket | Outcome | Size | Dependencies |
| --- | --- | --- | --- | --- |
| 1 | [S9-01](S9-01-workspace-read-model.md) | Complete action buckets/counts and review counts | M | OD-16, OD-17 |
| 2 | [S9-02](S9-02-daily-workspace-ui.md) | Daily workspace with honest coverage and existing controls | M | S9-01 |
| 3 | [S9-03](S9-03-application-discovery.md) | Owned search and effective-status filtering | M | OD-16 |
| 4 | [S9-04](S9-04-workspace-verification.md) | Integrated behavior and user-trial evidence | S | S9-01..03 |

All four tickets are implemented locally. [Execution report](execution-report.md) records verification, uncommitted source state and pending user-trial evidence. [API contract](api-contracts.md) documents requests, counts, time boundaries and coverage limits.

## Entry and exit

Entry met: the owner authorized resolving the review findings and implementing Sprint 9 on 2026-10-03. Browser IANA display timezone with visible Asia/Kolkata fallback is adopted (OD-17); scheduling remains unchanged. The [completed office state](../future-sprints/completed-state-review.md) is the baseline. Remote publication/CI and personal-machine verification are not local engineering gates.

Exit:

- [x] Full-dataset counts, date precision and timezone behavior are verified beyond one page.
- [x] Read failures/incomplete coverage cannot display “all caught up”.
- [x] Existing completion, dismissal, status override and email correction flows still work.
- [x] Search/status filters apply before pagination with owner isolation.
- [x] Runtime checks, focused regressions and integrated fixture smoke pass with recorded evidence.
- [x] Real-use task observations are recorded separately; pending observations stay pending.

See [architecture §2](../future-sprints/architecture.md#2-sprint-9-read-models-over-existing-data) and [acceptance register](../future-sprints/acceptance-and-decisions.md).

## Risks and scope limits

The main risks are timezone disagreement, page-local counts, stale query keys and empty-state overstatement. Address them with one backend read model and meaningful multi-page tests.

No calendar, new follow-up creation, archive, provider certification, historical reprocessing, notification changes or broad visual redesign. Existing AI-19/20 work retains its IDs. Acceptance defects are ticketed only if reproduced.
