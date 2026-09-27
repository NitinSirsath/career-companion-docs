# Sprint 5 execution report — 2026-09-27

This report supersedes the planning pack's implementation-pending status. The current execution instruction prioritizes implementation, automated validation and code review; unavailable manual Gmail/AI access does not halt engineering work. Live-provider results remain a separate, explicitly unverified evidence lane.

## Ticket scope and identifier reconciliation

The migrated documentation has contradictory identifiers: the index lists five workstreams, four shortened files merge/renumber them, S5-05 is absent, and historical unmatched-email code calls itself COM-37. No authoritative Linear issue contents or Linear connection are available in this environment. The implementation preserves **all five indexed workstreams**. The following sequential COM mapping is an execution cross-reference inferred from the supplied five-ticket list, pending confirmation; it does not rename historical Linear work or claim remote status changes.

| Ticket / document | Engineering result | External evidence |
| --- | --- | --- |
| COM-37 / S5-01 | Environment/source reconciliation, protected local backup, private owner-scoped baseline capture and comparison tooling, isolated test lanes | Original ~1,620-email dataset and deployed runtime are unavailable here |
| COM-38 / S5-02 | Incremental history, repeat idempotency, preservation, configured lookback, reconciliation and multi-page failure regressions; actual local API/queue/worker/browser proof | Real Gmail history discovery, live token refresh and original-data preservation unverified |
| COM-39 / S5-03 | Request/attempt recovery, transactional fencing, bounded Gmail/OAuth transport, queue interruption/retry proof, safe manual AI retry | No live fault injection performed |
| COM-40 / S5-04 | Bounded cross-route cache refresh, recoverable API timeout/settings/retry errors, real-worker browser harness | Fixture providers; no live AI quota workaround |
| COM-41 / S5-05 | Correlated lifecycle, truthful partial counters, retry/exhaustion and worker registration diagnostics; operational runbook | Production worker health requires deployment logs/database access |

## Starting environment

All migrated source files matched GitHub main byte-for-byte after restoring Git metadata without changing working files:

- Backend: `71d234e560a855699ef37851fddf33cfb478d384`
- Frontend: `ef36eb76d4b407c2346da4173c3945f736f65290`
- Docs: `6e19696e93b25f9955ad209618bc534556208a38`

Implementation files remain in place without Sprint 5 commits, per the user's follow-up instruction. No original `.env`, OAuth secrets or Gmail token configuration was migrated. Read-only inspection of local `career_companion_db` found **0 emails, 0 Gmail connections and the eight original migrations**. This is not the historical preserved dataset. A protected custom-format backup exists outside the repositories under the workspace's `.sprint5-private` directory. No migration, reset, fixture or worker was run against that database.

Dedicated newly created databases: `career_companion_sprint5_test`, `career_companion_sprint5_smoke_test`, and `career_companion_sprint5_crash_test`. The existing 12 migrations were deployed only to these fixture databases. Test configuration remains ignored and contains synthetic credentials. Browser/crash fixtures use exclusive database guards; workers stop before fixture cleanup.

## Implementation decisions

1. **Keep the existing architecture.** No scheduler, push subscription, queue replacement, provider change, outbox, public telemetry endpoint or schema redesign. Sync remains manual, additive ingestion of recent INBOX mail. Preserve main's **1/7/14/30-day lookback with default 1 day**; planning references to a fixed 90 days were stale.
2. **Stable request, unique attempt.** Existing `syncClaim` encodes `queued:<request>:attempt:<attempt>`. A retry can reclaim its own expired attempt; an active same-request delivery retries instead of being acknowledged as success. Obsolete requests finish with a superseded outcome. Exact attempt ownership fences cleanup and completion.
3. **Short transactions only.** Acquisition reads fresh settings/checkpoint under a connection row lock. Metadata persistence and final checkpoint updates lock the connection, check database-clock lease validity and cancellation, and require exact ownership. No Google/queue request runs inside a transaction. Transaction lock/statement limits prevent long-held locks. Already-accepted queue sends remain a separate idempotent boundary.
4. **Bound transport at the actual transport.** Both Gmail requests and OAuth refresh use the bounded transporter, with SDK retries disabled and cancellation/absolute deadline propagated. Queue retry/backoff remains the single transient retry policy. A 403 quota failure does not revoke healthy credentials or force OAuth refresh. Token save/revoke uses credential compare-and-swap and bounded DB transactions.
5. **Preserve paid-effect and user decisions.** Migrated main's manual-retry route deleted all AI operations and reset match/relevance state. It now re-enqueues only safely resumable work under the existing singleton/operation ledger. PROCESSING/UNKNOWN/terminal/exhausted work is held for review; completed results may be adopted without new paid calls. No operation versions or existing results are reset.
6. **Separate ingestion from processing.** `lastSyncedAt` confirms ingestion only. An authenticated-shell observer keeps a bounded refresh window across SPA navigation so delayed matching updates cached application/detail/timeline/action views. Failed browser requests time out and leave a usable retry path. Existing design-system primitives and tokens are preserved.
7. **Counts describe evidence.** Every fetched page registers references before processing. Each unique candidate has one current disposition; repeated references never turn one inserted row into both new and existing. Confirmed inserts count before enqueue. Ambiguous persistence/handoff outcomes remain unknown, not success. Success requires confirmed checkpoint commit; no-change repeats may commit the same history string.

## Significant components

Backend: Gmail sync/state/telemetry services, shared OAuth/Gmail client, Gmail/email job handlers, manual email retry route, focused reliability tests, baseline snapshot tool, crash harness and test-only smoke provider adapter. Frontend: authenticated-shell refresh observer, API timeout handling, Gmail mutation/error presentation, component regressions and Puppeteer real-worker smoke. Documentation: this report, corrected five-ticket index/files, operational runbook and repository verification instructions.

## Validation evidence

Final consolidated results and commits are recorded in the validation closeout below. Historical audit test counts are not reused as current evidence.

- Isolated real pg-boss crash regression: worker acquired an attempt, process was SIGKILLed, only fixture lease/job clocks were aged, pg-boss supervision expired/redelivered the original job, retryCount became 1, a fresh worker committed history and one email, and a repeat sync retained one row.
- Representative fixture: 1,620 existing completed emails plus one new tail item across 17 pages; one metadata fetch/new row, no reprocessing of completed records, within the four-minute budget.
- Baseline capture/comparison: private file safety, optional legacy schema and preservation/replay detection tests.
- Local HTTP transport: real installed Google OAuth/Gaxios stack with loopback fixtures; reactive refresh, quota/no-hidden-retry, hung refresh deadline, cancellation and credential replacement/disconnect safeguards.
- Browser: real authenticated API/queues/workers with deterministic Gmail/AI adapters; durable operation budget, delayed application updates, repeat preservation, pagination, action persistence and multi-user isolation.

## Regression review

Independent review found and corrected incomplete counters after a mid-page failure, mismatched failure categories between service and worker logs, and missing exclusive crash-harness locking. Review also required a browser-visible provider failure/recovery scenario in addition to successful smoke and component timeout coverage. Transport/email review found no blocking paid-effect, privacy or ownership issue. The missing browser failure/recovery case was added and passed; the final cross-review found no remaining blocking product-code finding.

## Database and migrations

No Sprint 5 schema migration is introduced. Existing claim/lease, uniqueness, operation-ledger and error fields are sufficient. All 12 pre-existing migrations are required for the current build. The preserved local database remains at its original eight migrations. A real deployment must separately verify migration state and backup before normal rollout; do not migrate merely to make a baseline snapshot query succeed.

## Known limitations and post-sprint pass

- Real Gmail discovery/token refresh, the historical ~1,620-row baseline, live Gemini and live Discord are unverified. This environment has no connected mailbox. Automated fixture success is not a live-provider claim.
- Linear mapping/status remains unconfirmed; no issue was created, overwritten or transitioned.
- Unknown paid AI outcomes require reconciliation; safe retry deliberately does not erase claims. Email queue retries can exhaust while an operation remains resumable after budget recovery; operators can use the guarded retry path when appropriate.
- Pending insert/enqueue-gap recovery remains capped at 100 oldest pending records per sync. Very large/slow provider scans can still require repeated sync; no durable page cursor was added without evidence of starvation in the representative fixture.
- Retention/account-deletion product policy and the existing notification transaction-to-queue gap remain outside Sprint 5.
- If a process is killed during the final email-worker attempt, its email can remain PROCESSING because no catch executes. A failed-queue reconciliation consumer is not present; inspect queue/operation evidence before safe recovery. This is a post-sprint reliability follow-up, never permission to erase an uncertain paid claim.
- Existing Prisma/deepmerge-ts dependency advisory and baseline lint/build warnings need a separate focused maintenance pass; no forced dependency downgrade or broad upgrade was added.


## Final validation closeout

| Check | Result |
| --- | --- |
| Backend full Vitest | **221 tests, 23 files passed** |
| Baseline tool Node tests | **15 passed** |
| Frontend full Vitest | **60 tests, 10 files passed** |
| Backend/frontend typecheck and build | Passed |
| Backend lint | 0 errors; 7 pre-existing unused-disable warnings |
| Frontend lint | 0 errors; 20 existing warnings |
| Frontend build size | Existing >500 KB bundle warning; no sprint bundle redesign |
| Prisma validate / migration status | Valid; all 12 existing migrations applied in isolated test lane |
| Fresh fixture databases | Existing migration chain deployed successfully |
| Isolated backup restore + eight→12 migration upgrade | Retained synthetic email and completed legacy AI result; baseline comparison found **0 invariant violations**; two explained deltas are newly available AI ledger/budget tables |
| Actual process crash + queue redelivery | Passed, actual retryCount=1, repeat retained one email |
| Final real-worker browser smoke | Passed delayed cross-route update, repeat preservation and 503 failure→manual sync recovery; AI fixture calls remained exactly 2 |
| Git diff whitespace | Passed in all repositories |
| Dependency audit | Frontend 0 vulnerabilities; backend 3 high entries for one inherited deepmerge-ts stack-exhaustion advisory through Prisma/@prisma/config; forced suggested Prisma downgrade not applied |

One intermediate Chrome run disconnected during a frame transition; subsequent clean reruns, including the final failure/recovery smoke, passed. This transient browser-harness event did not reproduce as an application defect.

The protected source database was backed up and restored only into the dedicated upgrade fixture database. No original mailbox data was present. Local restore/upgrade verification is not proof of the unavailable historical dataset or deployed system.

Current delivery state: all five assistant-created Sprint 5 commits were removed from the local branch histories at the user's request. File hashes were verified unchanged by that operation. Changes remain uncommitted in the three project folders; no push, PR, merge, deployment or Linear mutation occurred. A fresh follow-up review found two issues; both were subsequently fixed at the user's request, as recorded below.

All five indexed engineering workstreams were implemented. Real Gmail and original-data evidence, authoritative COM title mapping, and the listed post-sprint follow-ups remain explicit limitations.


## Follow-up code review — uncommitted delivery

1. **P2 — Retry API can overwrite a new terminal failure.** `backend/src/routes/email.ts:209–214` sends the job, then changes any FAILED email to PENDING. If the new worker finishes with a terminal failure before enqueue acknowledgment returns, the API overwrites that new FAILED state and leaves PENDING with no live job. Confirmed with one isolated API/DB regression reproduction; the temporary test was removed after review. State transition needs a version/attempt fence or safer enqueue ordering, including a fast-failure race test.
2. **P2 — Timed-out manual retry does not reconcile accepted work.** `frontend/src/routes/gmail.tsx:69–73` restarts refresh only on success. An accepted retry with a lost/late response leaves an idle observer and cached FAILED state stale until the user explicitly refreshes. Reconcile uncertain mutation outcomes as the sync mutation already does; add lost-acknowledgment coverage. Source-confirmed, not browser-reproduced in this review.

Independent sync/transport review found no additional actionable P1/P2 issue. No product code was changed during this follow-up review, and no new commits were created. Prior suite passes remain evidence of their covered scenarios, not evidence that these races are fixed.


## Follow-up fixes — 2026-09-27

Both P2 review findings above are resolved in the uncommitted files.

- Retry acknowledgment no longer writes email processing state. The worker alone moves a retried email into PROCESSING and its final outcome; queue acknowledgment cannot overwrite a fast failure or completion. Until the worker starts, the previous failed outcome remains visible. Paid-operation claims and user decisions are unchanged.
- Manual retry uses onSettled to start bounded refresh even when the HTTP response is lost or times out. The error remains visible while queries reconcile the actual server result.
- Added backend regressions for processing/failure before queue acknowledgment, plus frontend component coverage for accepted retry with lost response and the manual-retry transport timeout.
- Validation: 20 focused backend tests and 17 focused frontend tests passed; both typechecks passed; affected backend lint and frontend lint passed (existing frontend warnings remain). No broad suite or live-provider rerun was needed for these two targeted fixes.
- No commits, pushes or deployment were performed.
