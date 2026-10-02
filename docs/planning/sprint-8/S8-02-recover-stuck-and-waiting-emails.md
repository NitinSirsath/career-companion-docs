# S8-02 — Recover stuck and waiting emails without an operator

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 8 (provisional) — order 2 of 4 |
| Repository | career-companion-frontend (main part: row label, row action, bounded polling, tests); career-companion-backend (small: one computed field on `GET /api/gmail/messages`, one constant, shared contract, tests). No migration. |
| Size / priority | S / the owner can recover a stuck email without editing the database |
| Depends on | [AI-20](../byo-ai/AI-20-recovery-path-and-status-ui.md) (visible Manual Retry row action; bounded polling for PENDING rows); Sprint 8 entry conditions ([Sprint 8 README](README.md) §5). No owner decision (OD-nn) is needed. |
| Blocks | — |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md#s56--sprint-5-and-sprint-6-leftovers) S56-14; [PLAT-18](../state-audit-2026-10-02.md#plat--platform-security-and-migration-readiness) (stale PROCESSING part only); [Sprint 8 README](README.md) §2, §3, §7; [AI-20](../byo-ai/AI-20-recovery-path-and-status-ui.md) §5 (hands stuck PROCESSING to this ticket) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

An email left PROCESSING with no live job (no write to the row for 20 minutes) says that it stopped, and offers Manual Retry in its row. A row that waits for an automatic retry is labelled honestly. Polling on the Gmail page always ends. The backend tells the page which rows are stuck; it never moves them itself.

## 2. Why it exists

- If the backend dies during an email's final delivery, the email stays PROCESSING. Nothing in the backend moves it again (S56-14). The documented answer is "operator review" (`career-companion-backend-main/STABILIZATION.md:21`). When the row has no error details, the owner's only way out is editing the database.
- A normal stop can do the same. On SIGTERM or Ctrl-C, pg-boss waits only 30 s for the running job, and model timeouts go up to 60 s (PLAT-18, correction a).
- The page has no button in the rare case without earlier error details. It labels a stuck row "Retrying" when nothing will retry. It polls such rows every 2 s or 15 s with no end (S56-14).
- The backend already has a safe path. The retry route accepts PROCESSING emails, and an interrupted AI call becomes approvable after 15 minutes (§3 fact 8; [ADR-0001](../../architecture/decisions/ADR-0001-user-provided-ai.md) decision 10). Only the page is missing.
- "Waiting" in the title means a row that waits for an automatic retry that may never come. PENDING rows that wait for AI or a sync belong to AI-20 (its §4 items 12–13).

## 3. Current behavior

Checked on 2026-10-02. Paths start with `backend/` (career-companion-backend-main) or `frontend/` (career-companion-frontend-main). AI-20 rewrites parts of `frontend/src/routes/gmail.tsx` first, so re-check those line numbers after it lands.

**3.1 Facts (verified in code)**

Backend:
1. **Start write.** `processEmailJob` sets only `processingState: 'PROCESSING'` before any work (`backend/src/jobs/emailProcessingJob.ts:98-101`). It does not clear earlier error fields.
2. **End writes.** Only the worker's own code leaves PROCESSING: the waiting-for-AI write (`:118-128`), the failure write (`:148-158`, FAILED only when terminal or exhausted, else PROCESSING with error details) and `EmailAIPipeline.finish` (`backend/src/services/ai/pipeline.ts:116-131`, COMPLETED). During a job, the only other email write is the matcher's (`pipeline.ts:118`; `backend/src/services/matcher.ts:106`, `:160`). User match actions and S8-01 corrections also write the row (§7). A sync that sees a stored message again calls `prisma.email.upsert` with `update: {}` (`backend/src/services/gmailSync.ts:166-176`, `ingest`); see §3.2.
3. **Job options.** `emailJobOptions` (`emailProcessingJob.ts:15-25`): `retryLimit: 3` (`EMAIL_RETRY_LIMIT`, `:8`), `retryDelay: 60`, `retryBackoff: true`, `expireInSeconds: 300`, `singletonSeconds: 300`.
4. **pg-boss 12.31.0 (read from source, not measured).** A job active past `expireInSeconds` is failed by `failJobsByTimeout` (`node_modules/pg-boss/dist/plans.js:1980-1985`). The supervise loop runs it (`boss.js:128` timer, call at `boss.js:311`), at most every 60 s by default (`superviseIntervalSeconds` and `monitorIntervalSeconds`, `attorney.js:564`, `:570`). A failed job with retries left goes to `retry` with a backoff of `retryDelay × (2^(n+1)/2) × (1 + random)` seconds, where n is the failed delivery's `retry_count` (`plans.js:2079-2088`). The longest wait, before the last retry, is under 60 × 8 = 480 s. At `retry_count = retry_limit` the job becomes `failed` with no retry.
5. **No recovery.** There is no dead-letter queue, failure hook or sweeper (`backend/src/services/queue.ts:8-13`, plain `createQueue`). `reofferPendingEmails` re-offers only PENDING rows (`backend/src/services/gmailSync.ts:31-41`, filter at `:34`).
6. **Shutdown.** `shutdown` (`backend/src/index.ts:119-131`) calls `stopQueue` → `boss.stop()` with defaults (`queue.ts:25-29`). pg-boss waits up to 30 s (`node_modules/pg-boss/dist/index.js:185`), then `failWip` fails active jobs with "pg-boss shut down while active" (`manager.js:532-540`). `process.exit(0)` then ends the handler. On the final delivery the email stays PROCESSING.
7. **Retry route.** `POST /api/emails/:id/retry` (`backend/src/routes/email.ts:148`) refuses only COMPLETED (`:177-180`). A held AI operation that is not approvable gives 409 `AI_OPERATION_REQUIRES_REVIEW` (`:191-199`). An approvable one gives 409 `AI_RETRY_NEEDS_APPROVAL` with details (`:204-221`). A singleton-suppressed enqueue gives 409 `RETRY_RECENTLY_QUEUED` (`:276-285`).
8. **Interrupted AI call.** The claim sets `startedAt` (`backend/src/services/ai/operations.ts:88-105`, `startedAt` at `:100`). A PROCESSING claim is approvable as `OUTCOME_UNKNOWN` only after `STALE_PROCESSING_MS = 15 * 60_000` from `startedAt` (`backend/src/services/ai/heldOperations.ts:13-14`, `holdOf` `:34-38`). Before that the route returns `AI_OPERATION_REQUIRES_REVIEW`. The claim starts after the start write, separated by one or two Gmail calls and at most one model call (`pipeline.ts:45`, `:59`, `:94-95`).
9. **Response has no age.** `GET /api/gmail/messages` (`backend/src/routes/gmail.ts:421-454`) selects no timestamp that says when PROCESSING began (`:431-447`). `processingFailedAt` is null after a first-delivery kill, and stale after a manual retry of an old FAILED row (fact 1). `Email.updatedAt` exists with `@updatedAt` (`backend/prisma/schema.prisma:105`). The strict-keys test asserts it is not returned (`backend/src/tests/gmail.test.ts:582-625`, `not.toContain('updatedAt')` at `:622`).
10. **Status poll is already bounded.** `GET /api/gmail/status` reports a SYNCING row with an expired lease as FAILED (`routes/gmail.ts:101-105`). The lease is 5 minutes (`gmailSync.ts:43-44`).

Frontend:
11. **Polling.** The messages query polls every 2 s while any row is PENDING (not waiting for AI) or PROCESSING with falsy `processingRetryable`, and every 15 s while any row is PROCESSING with `processingRetryable` true. There is no end (`frontend/src/routes/gmail.tsx:81-88`). Polling runs only while the tab is focused (default `QueryClient`, `frontend/src/main.tsx:10`).
12. **Label.** The status header reads "Retrying" whenever `processingRetryable` is true (`gmail.tsx:355`). Otherwise it uses `PROCESSING_ERROR_LABELS`, where `RetryableAIError` also reads "Retrying" (`frontend/src/lib/aiLabels.ts:100`). So a FAILED row whose last error was `RetryableAIError` (thrown when a claim is not ready, `backend/src/services/ai/operations.ts:106`) also says "Retrying". The badge shows the raw state (`:344-345`).
13. **Action.** Manual Retry exists only inside the error tooltip, shown only when `processingErrorDetails` is set (`gmail.tsx:347-387`, button `:371-383`). AI-20 makes it a visible row action for the same rows.
14. **Bounded window.** `frontend/src/lib/processingRefresh.ts` (Sprint 5 S5-04) holds a 120 s window (`:6-7`, `startProcessingRefresh` `:24-27`). Sync, retry and approval open it (`gmail.tsx:109`, `:123`, `:137`), as does a new `lastSyncedAt` (`frontend/src/components/ProcessingRefreshObserver.tsx:28-34`). While open, the observer invalidates `gmailMessages` every 4 s (`:36-47`).
15. **Tests.** No frontend test renders a PROCESSING row. Manual Retry is tested only on FAILED rows with details (`frontend/src/tests/ai-status.test.tsx`, `processing-refresh.test.tsx:82-96`).

**3.2 Suspected risks (not demonstrated)**

- **Prisma and `updateMany`.** Prisma documents `@updatedAt` as set on every update through the client. That it holds for `updateMany` here is unverified; §4 item 8 adds a test that proves it.
- **Sync upsert.** Whether Prisma moves `updatedAt` on an upsert with `update: {}` (fact 2) is unverified. If it does, a stuck row reads as live for 20 minutes after each sync that sees it again. Polling then runs at 15 s for those 20 minutes and stops; it stays bounded.
- **Hung handler.** Until [S5-FU-02](../sprint-5/follow-up-google-request-bounds.md) bounds the OAuth token refresh and turns off Google SDK retries, a delivery can run past its expiry with no write (S5-FU-02 §2). pg-boss retries it meanwhile. After the threshold the row reads as stuck while the hung handler is still alive. A retry then adds a second delivery. The AI ledger reuses completed results and blocks a second claim, so a second paid call is not expected. Unverified for this exact interleaving.
- **Silent window estimate.** The longest gap between writes for a live job is about 15 minutes: 300 s expiry, up to about 2 minutes until pg-boss notices, up to 480 s backoff. pg-boss fetches by priority, then creation time (`plans.js:1626`, `fetchNextJob`), and a retried job keeps its `created_on`, so a due retry should not wait behind newer jobs. Not measured.

## 4. Scope

**4.0 Defaults (recommended; the owner may override at review).** No OD-nn applies.
- **Threshold: 20 minutes, not 5.** S56-14 starts from the job expiry (300 s). That alone is too short. A live job can be silent for about 15 minutes (§3.2), and an interrupted AI call is approvable only 15 minutes after the claim (fact 8). 20 minutes clears both, so a stuck row lands on the approval dialog (409 `AI_RETRY_NEEDS_APPROVAL`), not on 409 `AI_OPERATION_REQUIRES_REVIEW`.
- **Polling tail: 15 s until rows finish or are stuck.** The page keeps learning whether a live retry finished, and the tail ends on its own. Alternative: no polling outside the window, plus one fetch per row at its threshold.

**Backend**
1. **Threshold constant.** In `emailProcessingJob.ts`, next to `emailJobOptions`, export `EMAIL_PROCESSING_STUCK_MS = 20 * 60_000`. Its comment states the derivation: `expireInSeconds` + about 120 s detection + longest backoff `retryDelay × 2^EMAIL_RETRY_LIMIT`, and at least `STALE_PROCESSING_MS` + 5 minutes.
2. **Computed field.** `GET /api/gmail/messages` also selects `updatedAt`, then maps each row to `processingStuck: processingState === 'PROCESSING' && updatedAt < now − EMAIL_PROCESSING_STUCK_MS`. Use one `now` per request. Do not return `updatedAt`.
3. **Shared contract.** In `backend/src/contracts/gmail.ts` `EmailMessageSchema`, add `processingStuck: z.boolean().optional()` with a one-line comment. Run the frontend `npm run sync-contracts`.

**Frontend** (builds on AI-20's row action and PENDING poll rule)

4. **Stuck row.** When `processingStuck === true`: the badge reads "Stopped" (warning tokens, not destructive). One plain line says what happened, for example "Processing stopped before it finished. Nothing will retry it automatically." The row shows AI-20's visible Manual Retry, with or without `processingErrorDetails`. If details exist, AI-20's disclosure still shows them. The line is plain text, not a live region.
5. **Honest labels.** Remove "Retrying". A stuck row's header reads "Processing stopped". A PROCESSING row with `processingRetryable` true that is not stuck reads "Retry scheduled". Other headers keep the category label from `PROCESSING_ERROR_LABELS`, except that `RetryableAIError` gets a label that does not promise a retry, for example "Temporary error" (fact 12). Final wording is the implementer's, plain and short.
6. **Bounded polling.** Replace the rule at `gmail.tsx:81-88`:
   - Window open: 2 s if any PENDING row that AI-20 polls, or any PROCESSING row, is on the page.
   - Window closed: 15 s only while some PROCESSING row has `processingStuck === false`. A missing field never polls outside the window.
   - Otherwise: no polling.

   Read the window through `useSyncExternalStore(subscribeProcessingRefresh, getProcessingRefreshUntil)` and compare with `Date.now()` inside the function. TanStack re-evaluates it after each fetch, so no extra timer is needed.
7. **Retry from a stuck row.** Use the existing `retryMutation`. It already opens the refresh window and routes a 409 `AI_RETRY_NEEDS_APPROVAL` to the dialog that AI-20 hardens. Other answers keep today's messages (`retryMessage`, `gmail.tsx:26-41`).

**Tests**

8. Backend and frontend tests in §11, including a test that the start write moves `updatedAt` and a test that pins the threshold derivation.

## 5. Out of scope

- A backend sweeper, dead-letter queue, `onComplete` hook or automatic reset of PROCESSING emails. The backend stays as it is; recovery is the owner's click.
- The shutdown deadline, closing the pool and repeated-signal handling (PLAT-18 part 3, AIB-13) — [release track](../README.md#7-release-track-not-scheduled) item 3.
- Notification outbox (PLAT-18 part 1) and offset pagination (part 4) — [roadmap §8](../README.md#8-not-now).
- Changing the retry route, `STALE_PROCESSING_MS`, job options or the 409 codes. Rewording `AI_OPERATION_REQUIRES_REVIEW`: with a 20-minute threshold a stuck row should rarely reach it. Raise it if daily use shows otherwise.
- Hiding Manual Retry on PROCESSING rows a live job still owns. Today's rows keep their action.
- PENDING labels and PENDING polling (AI-20 items 12–13). The status poll, already bounded by the sync lease (fact 10).
- A new `processingStartedAt` column or any migration.
- A real kill harness for email jobs. Seeded state is enough for this ticket.

## 6. Likely files and components

Backend (`career-companion-backend-main`):
- `src/jobs/emailProcessingJob.ts` — `EMAIL_PROCESSING_STUCK_MS`.
- `src/routes/gmail.ts` — `GET /messages` select and mapping.
- `src/contracts/gmail.ts` — `EmailMessageSchema.processingStuck`.
- Tests: `src/tests/gmail.test.ts`, `src/tests/email-worker-reliability.test.ts`.

Frontend (`career-companion-frontend-main`):
- `src/contracts/gmail.ts` — synced copy, never edited by hand.
- `src/routes/gmail.tsx` — badge, header label, stuck line, row action condition, `refetchInterval`.
- `src/lib/aiLabels.ts` — the `RetryableAIError` label in `PROCESSING_ERROR_LABELS`, and the new labels if they sit next to it.
- Tests: new `src/tests/stuck-processing.test.tsx`; `src/tests/fixtures.ts` (add `makeEmailMessage()`).
- Docs: `README.md` (frontend), `STABILIZATION.md` (backend).

## 7. Implementation notes

- **Data model and migrations:** none. `updatedAt` already exists.
- **API and shared contract:** one optional boolean, additive. Backend `src/contracts` is the source; the frontend copy changes only through `npm run sync-contracts`. The backend strict-keys test (`backend/src/tests/gmail.test.ts:582-626`) keeps the key list S8-01 leaves (S8-01 adds `application`, `applicationId` and `matchConfirmedBy` and drops `not.toContain('applicationId')`, S8-01 §11.1) and adds `processingStuck`. It keeps the `not.toContain` checks for `updatedAt`, `createdAt` and `userId`.
- **Why the server decides.** The age is compared on the server, so the browser clock plays no part. The threshold sits next to the job options it comes from.
- **Background jobs:** no change to queues, job data or options. The constant only reads them.
- **Failure and recovery.** A stuck row → Manual Retry. With no held operation, the route enqueues a new job; the original singleton slot (300 s) has long passed. With an interrupted AI call older than 15 minutes, the user gets the approval dialog and may approve one call. The new delivery writes PROCESSING, so `processingStuck` turns false and the label returns to normal.
- **With S8-01.** A match correction writes the email row and resets its age. A stuck MATCHED row then waits another 20 minutes for its action. Accepted.
- **Compatibility.** Backend-first deploy as usual (`STABILIZATION.md:19`). An older backend sends no field: the page shows no stuck state and does not poll outside the window, which is still bounded.
- **Rollback.** Revert the frontend commit, then the backend one. No stored state changes.
- **Design rules.** Semantic tokens, `rounded-none`, `focus-visible` rings ([AI_UI_RULES.md](../../AI_UI_RULES.md) positive rules 3 and 6, negative rule 2). The row action stays reachable in the sideways-scrolling table.

## 8. Dependencies

- AI-20 done (Sprint 6 closeout). This ticket reuses its visible row action, accessible name and PENDING poll rule.
- Sprint 8 entry conditions met ([Sprint 8 README](README.md) §5).
- Related, not required: [S7-04](../sprint-7/S7-04-worker-startup-and-readiness.md) (a missing worker would leave retried jobs queued); S5-FU-02 (removes the hung-handler case in §3.2).
- CI from [S7-01](../sprint-7/S7-01-ci-and-toolchain-pins.md) must be green.

## 9. Security and privacy

- The new field is one boolean per row. It is returned only through the existing `requireAuth` route and `where: { userId }`, so ownership is unchanged.
- `updatedAt` is selected but never returned. No new email content, error text or timestamp leaves the server.
- Retry authorization and the approval rules are unchanged. `acceptPossibleDuplicateCharge` is still sent only from the dialog's confirm button, once per confirmation.
- Fewer requests: polling ends instead of running for as long as the tab is open.

## 10. Acceptance criteria

- [ ] `GET /api/gmail/messages` returns `processingStuck: true` for a PROCESSING email whose `updatedAt` is older than 20 minutes, and `false` for a PROCESSING email updated within 20 minutes and for PENDING, FAILED and COMPLETED rows, even when backdated.
- [ ] The response keys are the ones S8-01 leaves (including `application`, `applicationId` and `matchConfirmedBy`) plus `processingStuck`. The response has no `updatedAt`, `createdAt` or `userId`. Another user's stuck email is not returned.
- [ ] A delivery moves an email's backdated `updatedAt` to the present (proves the `@updatedAt` assumption).
- [ ] A test fails if `EMAIL_PROCESSING_STUCK_MS` drops below `expireInSeconds` + 120 s + `retryDelay × 2^retryLimit`, or below `STALE_PROCESSING_MS` + 5 minutes.
- [ ] Backend and frontend `src/contracts` are identical after `sync-contracts`.
- [ ] A stuck row with no error details shows "Stopped", the line and a visible Manual Retry, with no hover or focus events.
- [ ] A stuck row with retryable error details never shows "Retrying". A non-stuck PROCESSING row with a retryable error shows "Retry scheduled". A FAILED row with category `RetryableAIError` does not show "Retrying".
- [ ] Manual Retry on a stuck row calls `api.retryEmail` once and opens the refresh window. A 409 `AI_RETRY_NEEDS_APPROVAL` opens the approval dialog.
- [ ] Window closed, every PROCESSING row stuck: no message refetch in 10 minutes of fake time.
- [ ] Window closed, one PROCESSING row with `processingStuck: false`: refetch every 15 s. It stops after a response marks the row stuck or COMPLETED.
- [ ] Window open: 2 s. Field missing: no refetch outside the window.
- [ ] A page with every state stops all message polling once the window has closed and the last live row is stuck or done. States: PENDING ready, PENDING waiting, PROCESSING live, PROCESSING stuck, FAILED, COMPLETED.
- [ ] With today's `refetchInterval` restored, the "stops polling" test fails (§11.3).
- [ ] Full suites, typecheck, lint (0 errors) and build pass in both repos. The smoke passes with residue 0.

## 11. Testing

**11.1 Tests to add or change**
- `backend/src/tests/gmail.test.ts`, in `describe('GET /api/gmail/messages (COM-20)')` (`:571`): add `processingStuck` to the key list (`:604-620`, after S8-01 has added its three link fields there; §7). Add cases for stuck, fresh, other states and another owner. Backdate with `prisma.$executeRaw` (`UPDATE "emails" SET "updatedAt" = …`); raw SQL does not touch `@updatedAt`.
- `backend/src/tests/email-worker-reliability.test.ts`: a backdated email gets a fresh `updatedAt` after one delivery. A pure check of the threshold against `emailJobOptions('u', 'e')` and `STALE_PROCESSING_MS`.
- `frontend/src/tests/stuck-processing.test.tsx` (new; same router and mock setup as `processing-refresh.test.tsx`, plus AI-20's `getAISettings` mock resolved with `makeAISettings()`, AI-20 item 20): the frontend cases in §10. Use `vi.useFakeTimers({ shouldAdvanceTime: true })` and count `api.getMessages` calls, as in "expires instead of polling forever" (`processing-refresh.test.tsx:98-107`).
- `frontend/src/tests/fixtures.ts`: `makeEmailMessage(overrides)`.

**11.2 Commands**

```sh
# backend repo root
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs   # no new migration; confirms the test DB is at head
npx vitest run src/tests/gmail.test.ts src/tests/email-worker-reliability.test.ts
npm test

# frontend repo root
npm run sync-contracts        # first run: 1 file changed (gmail.ts); a second run: 0 changed
npm run typecheck && npm run lint
npx vitest run src/tests/stuck-processing.test.tsx src/tests/processing-refresh.test.tsx src/tests/ai-status.test.tsx
npm test && npm run build

# smoke (browser-visible change; empty migrated smoke DB; both repos built)
TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs   # backend
SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs                              # frontend
```

The smoke is a regression check only. It has no stuck-row scenario.

**11.3 Mutation checks** (temporary, never committed): restore today's `refetchInterval` (2 s / 15 s with no end); the "stops polling" test must fail. Set `EMAIL_PROCESSING_STUCK_MS` to 5 minutes; the derivation test must fail. Restore both and confirm `git diff --exit-code`.

**11.4 Manual check.** On the dev database, pick one email, set it to PROCESSING and backdate its `updatedAt` by SQL. Prefer a test user's email. If it must be the owner's own data, pick an email that is FAILED today: the retry is then one the owner could make anyway. The retry runs a real job, which may call Gmail and the owner's AI provider. The Gmail page shows "Stopped" and Manual Retry, and polling stops (browser network tab). Press Manual Retry and see the row move on. This is seeded evidence, not live evidence of a real kill. Record a real stuck email only if daily use produces one.

## 12. Documentation updates

- `career-companion-backend-main/STABILIZATION.md:21`: the known limit stays, but the email is no longer only "for operator review". After 20 minutes the Gmail page marks it stopped and offers Manual Retry; an interrupted AI call goes through the user's approval.
- Frontend `README.md`, "AI provider (ADR-0001)" → "Elsewhere" list (line 47 today; AI-20 edits this section first): stuck PROCESSING rows read "Stopped" with a visible Manual Retry. Message polling ends.
- `docs/planning/sprint-8/execution-report.md`: commands, test counts, mutation results, smoke line, manual check.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend tests, typecheck, lint (0 errors) and build are green on the new PC, and in CI (S7-01).
- [ ] Both §11.3 mutations were seen to fail and were restored. The smoke passes with residue 0.
- [ ] The §11.4 manual check is done and recorded as seeded evidence.
- [ ] The §12 docs are updated. No other doc is rewritten.
- [ ] One focused commit per repo (backend, frontend, docs).
- [ ] Evidence is recorded in the Sprint 8 execution report. Any defect found outside this scope is recorded there and raised with the owner, not fixed here.
