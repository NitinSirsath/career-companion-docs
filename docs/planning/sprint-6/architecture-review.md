# Sprint 6 current-state assessment

Inspected baseline: 2026-09-26. This is a stabilized-code assessment with an unexecuted Sprint 5 dependency, not a claim to have inspected a completed Sprint 5 deployment. The local Sprint 5 draft and its later planning review are available and incorporated.

## Repository and evidence boundary

Source prefixes in this pack identify repositories, not newly invented paths:

| Prefix | Local root | Inspected HEAD |
| --- | --- | --- |
| backend/ | /Users/spurge-rental/Documents/docs/career-companion-backend | c7e33f58eb27f68aa511800978ad0fabb5057902 |
| frontend/ | /Users/spurge-rental/Documents/docs/career-companion-frontend | 11e468dc10073f5b85f3a14a60cb40aa3ff8d18d |
| docs/ | /Users/spurge-rental/Documents/docs/career-companion-docs | 7b82b2e606f0ff1c3086633f3cbbe51104d4db5e plus local planning drafts |

The earlier planning inspection verified remote main at backend `b60f456`, frontend `f675abe`, docs `ea7a475`, without the stabilization commits. That is dated evidence; verify again at execution. The older sibling `*-main` directories are snapshots, not the inspected Git baseline. Existing Sprint 5/README edits are separate local work and are preserved.

The available Sprint 1–4 documents and Git history establish the implemented foundations; user-supplied completion status is not a substitute for live Linear verification. The audit reports 177 backend and 51 frontend tests, builds/typechecks/lints and an isolated browser smoke passing. Those historical results are not new runs. The original browser smoke disables workers and writes completion directly; Sprint 5 plans to replace that limitation. No provider/database access or runtime suite was used to save this pack.

Read the [Sprint 5 architecture findings](../sprint-5/architecture-review.md) and [resolved planning review](../sprint-5/planning-review.md). In particular, queued/active sync claims can diverge on hard crash; final checkpoint update count is ignored; transport/refresh bounds and stale writes require targeted correction. Keep these in Sprint 5 rather than duplicating them as Sprint 6 tickets.

## Current capability matrix

Implemented means present in source, not proven live or production-ready.

| Capability | State | Current behavior / limitation |
| --- | --- | --- |
| Google auth/session | Implemented; needs live verification | Identity checks, rotation, persisted sessions and production guards exist |
| Gmail OAuth/token isolation | Implemented; needs live verification | Readonly scope, encrypted tokens, same-mailbox reconnect and conditional credential writes |
| Initial/incremental ingestion | Partially implemented reliability | History paging, bounded reconciliation and dedup exist; Sprint 5 recovery/fencing findings remain open |
| Automatic ongoing trigger | Missing | No scheduler/watch; Sprint 5 explicitly verifies repeatable manual sync |
| Ingestion/processing progress | Partial | Ingestion status and page-local processing polling; Sprint 5 handles bounded domain refresh |
| Relevance/classification/extraction | Implemented | Deterministic filtering, Gemini structured responses and Zod validation |
| AI idempotency/cost boundary | Implemented locally | Operation/version claims, checkpoints, bounded rejected-call retries and global daily call limit |
| AI semantic accuracy | Needs verification | No supplied measured real-mail accuracy set; dates are nullable unrestricted strings |
| AI recovery/observability | Partial | Logs and durable claims exist; uncertain/exhausted work needs operator review |
| Application create/list/detail | Implemented | Manual creation, paginated list and owned direct detail |
| Automatic application creation | Missing | Matcher searches existing apps; unmatched mail waits for linking |
| Matching/resolution | Implemented with limits | Thread then company/role matching; ambiguous/unmatched linking; ignore only for ambiguous |
| Correct existing wrong match | Missing public workflow | General rematching is not exposed by the current resolution endpoint |
| AI state derivation | Partial | Fixed monotonic ranking; ignores chronology when deciding terminal/reopened conflicts |
| Effective status | Technically unsafe semantics | List uses AI first; detail uses user first; list test asserts incorrect precedence |
| Manual status correction | Missing | Fields exist, but no mutation API or editor |
| Application history | Partial | Generic processing events, recording-time order, no source-email metadata in response |
| Actions/explicit follow-up | Implemented with limits | Pending/completed/dismissed; current promotion prioritizes one action over simultaneous follow-up |
| Precise deadlines | Technically unsafe for agenda use | Unrestricted extracted strings parsed by new Date; unknown timezone/date precision |
| Upcoming interviews | Missing usable view | Extracted strings exist, no normalized schedule or upcoming view |
| Silent applications | Missing | No inactivity policy or derived view |
| Consolidated dashboard | Partial | Action Center and matching queues; no complete activity/interview overview |
| Search/filter | Partial | Action status filter; no application search/status filters |
| Pagination | Implemented with limits | Seven lists default/cap 20; stable ties, offset shifts under concurrent changes |
| Discord | Partial | Owner-bound webhook and claims; enqueue crash window; one configured owner |
| Ownership/minimized Gmail data | Implemented locally; verify rollout | Service checks/DB triggers, transient bodies, encrypted OAuth tokens |
| Retention/deletion | Missing policy/workflow | Disconnect retains records; policy required before broader release |

## Concrete source evidence for the selected scope

- `backend/src/services/application.ts`: create/list/detail response mapper, latest event summary, event queries ordered `createdAt ASC, id ASC`.
- `backend/src/contracts/application.ts`: separate status fields; no effective-status/revision fields; event `createdAt` and `emailId`, no source metadata.
- `backend/src/routes/application.ts`: POST create, GET list/detail/events/actions; no status mutation.
- `backend/prisma/schema.prisma`: `aiStatus`, `userStatus`, `userStatusSetAt`; no manual revision. Events preserve old/new AI states and recording time.
- `backend/src/services/matcher.ts`: email then application locks, AI-only status writes, event/action uniqueness, monotonic state ranking and notifications after commit.
- `frontend/src/routes/applications.tsx`: `aiStatus ?? userStatus`; `frontend/src/routes/applications.$id.tsx`: `userStatus ?? aiStatus`.
- `frontend/src/tests/applications.test.tsx`: an explicit AI-precedence assertion must be corrected.
- `frontend/src/api/client.ts`: typed JSON without runtime response validation, structured error codes discarded, no application-query AbortSignal parameter.
- `backend/src/utils/testDatabase.ts` and `src/tests/setup.ts`: guard exists and `.env.test` overrides process values in Vitest. Direct Prisma CLI invocation does not call this guard.
- `frontend/scripts/smoke-stabilization.mjs`: current worker-disabled baseline; use the final Sprint 5 real-worker successor, not its current limitations, for Sprint 6.

## Architecture risks

| Priority | Risk | Disposition |
| --- | --- | --- |
| P0 / blocking gate | Integrated baseline and Sprint 5 closure are not established | Verify before approval/implementation; this label is a readiness gate, not an exploit claim |
| P0 / blocking gate | Original dataset/live incremental evidence absent | Link measured preservation and final live Gmail proof; do not substitute synthetic fixtures |
| P1 | Known sync crash/finalization/fencing gaps | Sprint 5 fixes/tests, not new Sprint 6 scope |
| P1 | Status precedence divergence and no correction path | S6-01/02/04 |
| P1 | Background refresh can silently rebase a new editor | Freeze draft revision in S6-04; test in S6-05 |
| P1 | Event recording time/state can look like recruitment chronology | Source metadata and explicit labels in S6-03/04; do not claim full event-time reconstruction |
| P1 | Monotonic AI inference can preserve wrong terminal state | Keep as an explicit limit; manual correction is a remedy, not a replacement inference policy |
| P1 | Date precision/timezone undefined | Must resolve before trustworthy agenda/reminder features |
| P1 / broader-release gate | Retention/deletion unresolved | Product decision before wider public launch |
| P2 | Stale reads, invalid contracts, uncertain save results | Bounded cancellation, runtime validation and explicit recovery in S6-01/04 |
| P2 | Uncertain/exhausted AI work requires operators | Preserve recovery guide and diagnostic ownership |
| P2 | Matching normalization, absent rematching, stale actions | Document; follow-on scope |
| P2 / accepted | Offset shifts and notification enqueue window | Keep MVP trade-offs explicit |
| Future | Other providers/infrastructure/advanced analytics | Out of scope |

## Smallest material product gaps

1. Consistent, correctable application state.
2. Source evidence and honest history semantics.
3. Reliable interview/deadline interpretation before an agenda.
4. Daily orientation across activity, pending processing, missing data and failures.
5. A defined data lifecycle for wider release.

Select the first two for this proposal. Wrong-match correction, missing agenda, and retention remain unresolved after it; no claim of complete MVP readiness is warranted.

## Technical debt disposition

| Category | Work |
| --- | --- |
| Must fix for selected sprint | Precedence divergence/test; safe correction boundary; draft concurrency; contract validation; misleading history labels; guarded migration procedure |
| Should fix soon | Date normalization; inference conflict/reopen policy; obsolete actions; rematching; complete processing visibility; affected stale docs |
| Acceptable for controlled MVP | Bounded offset pagination, operator reconciliation, one Discord owner, noncritical enqueue window, existing INBOX scope subject to Sprint 5 acceptance |
| Not now | Event sourcing, generic workflows, outbox overhaul, restyling, dependency churn, extra providers/platforms |

## Candidate themes and recommendation

| Theme | Goal / user value | Technical value | Dependencies | Risks | Rough scope |
| --- | --- | --- | --- | --- | --- |
| User-controlled status and explainable history | Understand and correct application state | One read rule, safe writes, clear evidence boundary | Sprint 5 closure | Concurrency, legacy provenance, confusing AI with confirmed state | 19 points / five tickets |
| Reliable interview/deadline agenda | Know what is next and when | Date precision, timezone and supersession rules | Sprint 5, temporal contract and legacy strategy | Invented times, outdated invites, accidental AI replay | 20–30 points |
| Discovery and daily overview | Find apps and see recent activity | Server filters and bounded summaries | Consistent status/activity/freshness semantics | Summarizing misleading state/dates | 13–21 points |

These are alternatives, not a ranking or known team velocity. Recommend the first because it closes documented UF-09 and a demonstrated domain-rule violation; status-based filters and summaries depend on that rule. It adds no AI calls. Clearing overrides and revision semantics are new proposed decisions, not claimed historical approvals.
