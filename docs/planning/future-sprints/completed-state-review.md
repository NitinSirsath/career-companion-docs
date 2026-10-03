# Completed product review — office laptop, 2026-10-03

Status: read-only source review for future planning. No application changes, provider calls, database commands or regression reruns were performed in this review.

## Baseline and evidence

Local repositories are siblings under the office scratchpad identified by the owner's execution report:

`/private/tmp/claude-501/-Users-spurge-rental-Downloads-docs/8472a241-e50b-41f8-9bcf-4a75ed0fd02a/scratchpad/`

| Repository | Locally inspected HEAD | Start state |
| --- | --- | --- |
| career-companion-backend | 6af9cd2d994e4d3d1944a5d0029b6d64f0085519 | Clean |
| career-companion-frontend | 155a1aacd9d7f2e6df56657a128b1cf511b616d6 | Clean |
| career-companion-docs | 1e98ae4b3018d1b68effc88ec790f0d62fd0ec7d | Clean before this documentation pass |

[Sprint 7 execution](../sprint-7/execution-report.md) and [Sprint 8 execution](../sprint-8/execution-report.md) record 767 backend tests, 182 frontend tests, passing builds/typechecks/lint with documented warnings, three fixture migration lanes, browser smoke and real SIGKILL/same-job recovery with synthetic providers. These are prior recorded results, not tests rerun for this document.

The owner's current status is authoritative: **Sprint 7/8 implementation COMPLETE; real-world acceptance PENDING.** Live Gemini certification is not claimed. Downloads copies were not inspected or changed during this planning pass. No remote comparison is needed.

## What exists

| Capability | Source inspected | Product implication |
| --- | --- | --- |
| Bounded, fenced Gmail ingestion and scheduled catch-up | [sync ownership](../../../../career-companion-backend/src/services/gmailSyncOwnership.ts), [schedule job](../../../../career-companion-backend/src/jobs/gmailScheduledSyncJob.ts), execution reports | Preserve these completed foundations. Live scheduled days remain an acceptance item. |
| DATE/DATETIME action deadlines | [schema](../../../../career-companion-backend/prisma/schema.prisma), [deadline parser](../../../../career-companion-backend/src/utils/actionDeadline.ts), [display](../../../../career-companion-frontend/src/lib/deadline.ts) | Reuse precision; do not infer an interview time from an action deadline. |
| Manual application status and source history | [application service](../../../../career-companion-backend/src/services/application.ts), [detail UI](../../../../career-companion-frontend/src/routes/applications.$id.tsx) | User status remains authoritative and separately revisioned. |
| Move/unlink/restore email effects, including retired history | [matcher](../../../../career-companion-backend/src/services/matcher.ts), [dialog](../../../../career-companion-frontend/src/components/MatchCorrectionDialog.tsx), [ADR-0003](../../architecture/decisions/ADR-0003-correcting-email-matches.md) | Future entities must participate in correction without changing its accepted policy. |
| Stopped-processing recovery and bounded polling | [Gmail UI](../../../../career-companion-frontend/src/routes/gmail.tsx), Sprint 8 report | Reuse Manual Retry and paid-call approval; do not plan them again. |
| Action completion/dismissal and contextual Gmail links | [dashboard](../../../../career-companion-frontend/src/routes/index.tsx), [action service](../../../../career-companion-backend/src/services/action.ts) | A daily workspace is an extension of existing behavior. |
| BYO AI, operation ledger and evaluation refusal accounting | [pipeline](../../../../career-companion-backend/src/services/ai/pipeline.ts), [contracts](../../../../career-companion-backend/src/services/ai/contracts.ts), [evaluation](../../../../career-companion-backend/src/eval/ai/run.ts) | No automatic reprocessing on a new release or contract change. Certification is still separate. |
| Manual application entry and MCP submission intake/review | [applications UI](../../../../career-companion-frontend/src/routes/applications.tsx), [submission service](../../../../career-companion-backend/src/services/externalSubmission.ts) | Do not create another ingestion channel or automatic application system. |

## Remaining product gaps

These are capability gaps, not newly demonstrated Sprint 7/8 defects.

| Gap observed in source | Consequence | Destination |
| --- | --- | --- |
| Dashboard groups only the returned 20-action page into overdue/upcoming/undated. API filters only status; no full-dataset time buckets or counts. | Users must browse pages to understand their workload. | S9-01, S9-02 |
| Application list API accepts pagination only; UI has no company/title search or effective-status filter. | Finding a known application gets harder as the list grows. | S9-03 |
| Review queues exist, but no aggregate attention summary; sync and AI state are separate. | Empty work can be mistaken for complete coverage. | S9-01, S9-02 |
| interviewDate/interviewTime/assessmentDeadline are nullable strings in extraction/v2; no scheduled-event entity or agenda route. | Upcoming interviews cannot be presented as trustworthy times. | S10-01..S10-03 |
| Actions expose status changes; no user-created follow-up form or snooze field. | Work known only to the user has no deliberate follow-through path. | S11-01, S11-02, S11-04 |
| Application has no archive state; CLOSED is a status override, not a reversible visibility choice. | Old or accidental applications remain in the working list. | S11-03, S11-04 |

Source evidence supports absence of these capabilities; daily-use evidence has not yet ranked their frequency or severity. The proposed order favors a small, useful next sprint.

## Existing obligations kept under existing IDs

- AI-15: runner delivered; live model reports/error rows and disclosure approval pending.
- AI-19: existing runtime/disclosure prerequisites remain. The current Gemini constructor still lacks explicit mode pinning, and the catalog summary still says body text is sent only for job-related mail while the pipeline also extracts uncertain mail. This review does not reproduce every historical AI-19 issue.
- AI-20: the visible retry entry was delivered as a Sprint 8 prerequisite. Do not count the whole historical UI ticket as complete or recreate its delivered portion.
- MCP-08/MCP-09 part B: real automation-client acceptance is separate from SDK fixture evidence.
- S6-C01/S6-C02/S6-R08: any remaining historic closeout work retains its ID and must be checked against current evidence before scheduling. Not inserted into Sprint 9 by default.
- Original-data preservation, live Gmail, remote CI/publication and deployment evidence remain distinct. Remote work is out of scope for this office workflow.

## Review limits

No end-user trial, live provider certification, security certification or fresh runtime test was performed. Architecture proposals below are reviewable designs, not claims that new APIs, entities or views already exist. The temporary checkout location is recorded for traceability; nothing was copied into Downloads or relocated.
