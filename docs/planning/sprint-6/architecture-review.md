# Sprint 6 current-state assessment

Reconciled source assessment: 2026-09-27. This supports the [authoritative local contract](README.md); it does not supply a second ticket allocation. Sprint 5 engineering and follow-up regression fixes are implemented but uncommitted. No Sprint 6 runtime test, provider call, migration or implementation was performed for this assessment.

## Repository and evidence boundary

Source prefixes refer to these **active migrated repositories**, including uncommitted Sprint 5 files:

| Prefix | Local root | HEAD before this documentation edit |
| --- | --- | --- |
| backend/ | /Users/spurge-rental/Documents/docs/career-companion-backend-main | 71d234e560a855699ef37851fddf33cfb478d384 |
| frontend/ | /Users/spurge-rental/Documents/docs/career-companion-frontend-main | ef36eb76d4b407c2346da4173c3945f736f65290 |
| docs/ | /Users/spurge-rental/Documents/docs/career-companion-docs-main | 6e19696e93b25f9955ad209618bc534556208a38 |

HEAD alone is not the engineering baseline. Preserve working diffs and untracked Sprint 5 source/tests/harness files and record their identity before later implementation. The [Sprint 5 execution report](../sprint-5/execution-report.md), including its follow-up fixes, records automated validation and review. Its 221 backend/60 frontend suite results and later focused regressions are reported prior evidence, not new runs here. Real Gmail, historical original-data preservation and deployment remain unverified; local fixture success does not fill those gates.

Historical inspection on 2026-09-26 used sibling roots without `-main`: backend `c7e33f58eb27f68aa511800978ad0fabb5057902`, frontend `11e468dc10073f5b85f3a14a60cb40aa3ff8d18d`, docs `7b82b2e606f0ff1c3086633f3cbbe51104d4db5e`. It reported then-remote main as `b60f456` / `f675abe` / `ea7a475`, an unexecuted Sprint 5 draft and a worker-disabled smoke. Those are dated facts retained through this note, the [stabilization audit](../../engineering/stabilization-audit.md) and Git history; they are not the current baseline or a current remote-state check. The migrated `*-main` directories are now active Git worktrees, not disposable snapshots.

The read-only investigation task “Audit Sprint 6 planning readiness” (2026-09-27, local task `01a0e38b-df11-7851-81c1-1e5b0077d64e`) found the `58c426c` allocation mismatch, absent Sprint 6 contracts/editor, stale baseline claims and unsupported batch-loss diagnosis. Relevant source findings were rechecked locally for this reconciliation. No GitHub or Linear ticket state is used.

## Confirmed facts, suspected risks and required verification

| Evidence class | Finding | Contract disposition |
| --- | --- | --- |
| Confirmed source fact | List uses AI-first precedence; detail uses user-first. Derived status/revision fields and status PATCH/editor are absent | S6-01/04 deliver the missing contract/workflow |
| Confirmed source fact | Event/recentEvent responses lack the required recording alias and bounded source metadata | S6-02/04; D1 separately holds new action-response scope |
| Confirmed source fact | Installed pg-boss 12.31.0 defaults batchSize=1; email worker uses jobs[0] with includeMetadata and no batch override | Not evidence of current batch loss; S6-03 makes delivery invariant explicit and tested |
| Suspected risk | A future batch-size change could leave delivered work unattempted; candidate reads before locks and simultaneous distinct emails may expose races | Reproduce/verify actual acknowledgment and matcher interleavings; do not label suspicion a demonstrated defect |
| Confirmed existing protections | Email then owned-application row locks, event/action uniqueness, user-decision guards, durable AI claims and duplicate-concurrency tests exist | Preserve and extend; no unnecessary architecture replacement |
| Required runtime verification | Distinct-email effects, stale automated matching versus user resolution, competing resolutions, manual status versus AI, delivery failures/retries | Explicit S6-03 matrix plus S6-05 integration |
| Reported Sprint 5 evidence | Recovery, transport, diagnostics, real-worker browser and follow-up retry regressions passed in isolated fixtures | Engineering baseline; maintain coverage and distinguish it from live evidence |
| Pending entry evidence | Historical approximately 1,620-email dataset and live Gmail incremental/repeat-sync not available/verified here | Retain the overview gate; documentation rewrite needs no provider access |

## Current capability matrix

Implemented means present in source, not proven live or production-ready.

| Capability | State | Current behavior / limitation |
| --- | --- | --- |
| Google auth/session | Implemented; needs live verification | Identity checks, rotation, persisted sessions and production guards exist |
| Gmail OAuth/token isolation | Implemented; needs live verification | Readonly scope, encrypted tokens, same-mailbox reconnect and conditional credential writes |
| Initial/incremental ingestion | Implemented; fixture-verified in Sprint 5 | History paging, reconciliation, dedup, attempt fencing and bounded transport; live Gmail evidence remains pending |
| Automatic ongoing trigger | Missing | No scheduler/watch; Sprint 5 explicitly verifies repeatable manual sync |
| Ingestion/processing progress | Implemented with limits | Ingestion completion is distinct from AI completion; Sprint 5 adds bounded cross-route domain refresh |
| Relevance/classification/extraction | Implemented | Deterministic filtering, Gemini structured responses and Zod validation |
| AI idempotency/cost boundary | Implemented locally | Operation/version claims, checkpoints, bounded rejected-call retries and global daily call limit |
| AI semantic accuracy | Needs verification | No supplied measured real-mail accuracy set; dates are nullable unrestricted strings |
| AI recovery/observability | Partial | Logs and durable claims exist; uncertain/exhausted work needs operator review |
| Application create/list/detail | Implemented | Manual creation, paginated list and owned direct detail |
| Automatic application creation | Missing | Matcher searches existing apps; unmatched mail waits for linking |
| Matching/resolution | Implemented with limits | Thread then company/role matching; ambiguous/unmatched linking; ignore only for ambiguous; S6-03 verifies remaining concurrency cases |
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
- `frontend/src/api/client.ts`: Sprint 5 deadline/internal signal forwarding exists; application JSON remains unvalidated, structured error codes are discarded and application-query signal parameters are missing.
- `backend/src/utils/testDatabase.ts` and `src/tests/setup.ts`: guard exists and `.env.test` overrides process values in Vitest. Direct Prisma CLI invocation does not call this guard.
- `frontend/scripts/smoke-stabilization.mjs`: current Sprint 5 real-worker harness with deterministic providers, dedicated SMOKE_DATABASE_URL and exclusive lane guards; extend rather than replace.

## Architecture risks and gates

| Priority | Risk | Disposition |
| --- | --- | --- |
| P0 / blocking gate | Sprint 5 working-tree identity and retained entry approvals/evidence need execution sign-off | Preserve implemented files and record evidence before implementation; this label is a readiness gate, not an exploit claim |
| P0 / blocking gate | Original dataset/live incremental evidence absent | Link measured preservation and final live Gmail proof; do not substitute synthetic fixtures |
| Regression boundary | Sprint 5 sync/retry/fencing fixes are uncommitted | Preserve implemented fixes/tests; do not reopen them as new Sprint 6 scope |
| Verification requirement | Delivery and matching interleavings lack complete S6-03 evidence | Verify actual configuration and races; no demonstrated batch-loss claim |
| P1 | Status precedence divergence and no correction path | S6-01/04 |
| P1 | Background refresh can silently rebase a new editor | Freeze draft revision in S6-04; test in S6-05 |
| P1 | Event recording time/state can look like recruitment chronology | Source metadata and explicit labels in S6-02/04; do not claim full event-time reconstruction |
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

The reconciled sprint selects the first two plus the worker/matching reliability scope introduced by `58c426c`. Wrong-match correction, missing agenda, and retention remain unresolved after it; no claim of complete MVP readiness is warranted.

## Technical debt disposition

| Category | Work |
| --- | --- |
| Must fix for selected sprint | Precedence divergence/test; safe correction boundary; draft concurrency; contract validation; misleading history labels; guarded migration procedure; explicit worker delivery and matching-concurrency verification |
| Should fix soon | Date normalization; inference conflict/reopen policy; obsolete actions; rematching; complete processing visibility; affected stale docs |
| Acceptable for controlled MVP | Bounded offset pagination, operator reconciliation, one Discord owner, noncritical enqueue window, existing INBOX scope subject to Sprint 5 acceptance |
| Not now | Event sourcing, generic workflows, outbox overhaul, restyling, dependency churn, extra providers/platforms |

## Historical candidate themes — 2026-09-26

| Theme | Goal / user value | Technical value | Dependencies | Risks | Rough scope |
| --- | --- | --- | --- | --- | --- |
| User-controlled status and explainable history | Understand and correct application state | One read rule, safe writes, clear evidence boundary | Sprint 5 closure | Concurrency, legacy provenance, confusing AI with confirmed state | 19 points / five tickets |
| Reliable interview/deadline agenda | Know what is next and when | Date precision, timezone and supersession rules | Sprint 5, temporal contract and legacy strategy | Invented times, outdated invites, accidental AI replay | 20–30 points |
| Discovery and daily overview | Find apps and see recent activity | Server filters and bounded summaries | Consistent status/activity/freshness semantics | Summarizing misleading state/dates | 13–21 points |

These dated alternatives explain the original theme choice; they are not competing execution plans or current estimates. The authoritative five-ticket mapping now also preserves `58c426c` worker/matcher scope. Manual correction and evidence reads add no AI calls; positive worker fixtures need a nonzero isolated budget. The old 19-point estimate is withdrawn as a current total. D1/D2 in the overview retain unresolved action scope and inherited sign-off.
