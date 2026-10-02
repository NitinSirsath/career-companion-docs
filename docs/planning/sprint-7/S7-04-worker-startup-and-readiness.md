# S7-04 — Background workers recover or fail loudly, and report readiness

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 7 — order 4 of 6 |
| Repository | career-companion-backend (docs repo: doc updates only; no frontend change) |
| Size / priority | S / reliability fix at the start of the golden path; must land before S7-05 |
| Depends on | Nothing inside Sprint 7 ([Sprint 7 README](README.md) §6). Starts after the Phase 0 gate and the Sprint 6 closeout, like all Sprint 7 work. No owner decision. Helpful, not blocking: S7-01 (CI), S6-C01 (TEST-05 test env pinning) |
| Blocks | S7-05 (the schedule must not run on silently dead workers) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) PLAT-07, TEST-16; [Sprint 7 README](README.md) §2, §3, §6, §7; [roadmap](../README.md) §3 and [release track](../README.md#7-release-track-not-scheduled) items 1 and 3 (kept there) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

If the background workers cannot start, the backend retries for about two minutes and then exits with code 1. The failure is loud, and the owner or a supervisor restarts the process. A new `GET /ready` says whether all three workers are registered in this process. `/health` stays a cheap "the HTTP process is up" check. Worker start and queue errors are logged with an error category. The placeholder health test is replaced by real tests.

## 2. Why it exists

- **PLAT-07 (confirmed, medium).** If PostgreSQL is not reachable when the backend boots, all three workers fail once and are never registered again. The API works once the database is up, and `/health` keeps returning 200. Gmail sync, AI email processing and Discord notifications stop with no visible cause. Only a restart fixes it.
- This is a realistic case on the new PC: PostgreSQL runs as a separate local service (native or Docker) and can come up after the backend. Phase 0 only adds a manual interim check (MV-11).
- S7-05 adds a twice-daily sync and a start-up catch-up (OD-05). Jobs queued into a process with dead workers would fail silently every time ([Sprint 7 README](README.md) §6).
- **TEST-16 (info; not re-checked by the audit verifier, confirmed by reading the file).** `src/tests/health.test.ts` is a placeholder, and `/health` is never exercised.
- The planned deployment checks `GET /health` from the load balancer ([high-level-architecture §9.5](../../architecture/high-level-architecture.md)). Today that check would stay green with dead workers.

## 3. Current behavior

Facts (checked in the code on 2026-10-02; backend = `career-companion-backend-main`; pg-boss 12.31.0):

- **Worker start.** `src/index.ts:103-113`: outside `NODE_ENV=test`, `startGmailSyncWorker()`, `startEmailProcessingWorker()` and `startNotificationWorker()` are called once each. Each `.catch(() => console.error(...))` logs only `gmail_worker_start_failed`, `email_worker_start_failed` or `notification_worker_start_failed`. The error object is dropped, nothing retries, and the process goes on to `app.listen` (`:115-117`).
- **One shared queue start.** `src/services/queue.ts:5-23` (`getQueue`) keeps one `starting` promise. `boss.start()` (:11) and `createQueue` for the three names (:12-13) run once. On failure the catch (:15-18) stops the instance and sets `starting = undefined`. So one failed start fails all three workers together, and the next `getQueue()` builds a fresh PgBoss.
- **Registration happens only at boot.** `queue.work` is called only in `startGmailSyncWorker` (`src/jobs/gmailSyncJob.ts:55-73`, `queue.work` at :57, queue `gmail-sync-job`), `startEmailProcessingWorker` (`src/jobs/emailProcessingJob.ts:201-207`, `queue.work` at :203; `EMAIL_PROCESSING_JOB` = `email-processing-job` at :7) and `startNotificationWorker` (`src/jobs/notificationJob.ts:29-162`, `queue.work` at :33; `NOTIFICATION_JOB` = `discord-notification-job` at :6). Only the email worker logs `worker_registered` (`emailProcessingJob.ts:204-206`).
- **Enqueue restarts pg-boss without workers.** `requestGmailSync` (`gmailSyncJob.ts:11-53`, `getQueue` at :27), `enqueueEmailProcessingJob` (`emailProcessingJob.ts:28-35`) and `enqueueNotificationJob` (`notificationJob.ts:12-27`) call `getQueue()`. After a failed boot start, the first enqueue starts a fresh pg-boss. Jobs are accepted and never processed.
- **What the user sees.** `GET /api/gmail/status` (`src/routes/gmail.ts:68`, mapping at :101-105) shows `SYNCING` for the 5-minute lease (`LEASE_MS`, `src/services/gmailSync.ts:43-44`) and then `FAILED` with `syncError` null. New mail is never fetched, because the sync runs inside the dead Gmail worker.
- **Health.** `GET /health` (`index.ts:82-84`) always returns 200 `{ status: 'ok', message: 'Career Companion Backend is healthy.' }` and checks nothing. It is mounted after CORS, the JSON parser, `cookieParser` and the session middleware (`index.ts:41-70`). Nothing in the frontend calls it, and the Vite proxy forwards only `/api` and `/mcp` (frontend `vite.config.ts:19-29`).
- **Queue errors.** `queue.ts:9` logs `{ event: 'queue_error' }` with no category.
- **Shutdown.** `index.ts:119-131` (`shutdown`, on SIGTERM/SIGINT): `server.close`, then `stopQueue()` and `prisma.$disconnect()`, then exit 0. No deadline.
- **Dev runner.** `npm run dev` is `ts-node-dev --respawn` (`package.json:10`). ts-node-dev 2.0.0 does not restart a child that exits; it waits for a file change (`node_modules/ts-node-dev/lib/index.js:156-166`).
- **pg-boss facts used below.** Each connection attempt times out after 10 s by default (`node_modules/pg-boss/dist/db.js:19`, `connectionTimeoutMillis ??= 10000`). `getWipData()` (`dist/index.js:377`; `dist/manager.js:642-648`) lists this instance's workers, minus `stopped` ones. Worker states are `created | active | stopping | stopped` (`dist/types.d.ts:916`). `work()` throws once the instance is stopped (`manager.js:543-545`). A registered worker catches fetch errors and keeps polling (`dist/worker.js:45-87`). `stop()` waits at most 30 s by default (`dist/index.js:185`).
- **Callers that must keep working.** The smoke sets `NODE_ENV=test`, imports `dist/index`, `dist/services/queue` and the start functions, and registers the Gmail and email workers itself (frontend `scripts/smoke-stabilization.mjs:47`, :191-195, :1009-1010). `email-worker-reliability.test.ts:310-361` calls `startEmailProcessingWorker()` on a real pg-boss (:318, :349) and then `offWork` (:324, :361). `notificationJob.test.ts:11-26` mocks the queue module. `ai-redaction.test.ts:26` and `ai-settings.test.ts:16` also mock `../services/queue` with a factory that returns only `getQueue` and `stopQueue`, and both import the app (`:11` and `:3`). Vitest 5.0.0 throws when code reads an export that such a factory does not define (`node_modules/vitest/dist/chunks/index.1_nbEjJY.js:340`).
- **Tests.** `src/tests/health.test.ts:4-6` is `expect(true).toBe(true)`. `src/tests/queue.test.ts:4-15` covers only the happy path. No test covers a worker start failure.

Suspected risks (not demonstrated):

- Jobs queued before an exit stay in `pgboss.job` and should run after a restart (verifier's reading of the code). Not run.
- A refused local connection on Node 24 may surface as `AggregateError` with `code: 'ECONNREFUSED'`, not a plain `Error`. Unverified; the category rule in §4 item 5 handles both.
- A browser with a `cc_session` cookie for `localhost` also sends it to `:3000/health`, so the session middleware may query the session store for that request. Library behavior, not tested here.
- After the workers registered, a database outage only produces repeated `queue_error` lines; the workers keep polling (code read, not run). The `/ready` from this ticket would still say ready, because it does not check the database.

## 4. Scope

1. **Decision: what happens when workers cannot start. Recommendation: retry with bounded backoff, then exit non-zero.** No owner decision is needed; the [Sprint 7 README](README.md) §7 exit criterion already says "retries and then exits non-zero". Reason: a process that serves HTTP with dead workers is the silent failure itself. An exit is visible locally and lets any supervisor restart the process later. Retrying forever was considered and is not recommended: it hides a long outage behind a working API.
2. **Start-up routine.** Add `startWorkers(...)` in a new module (suggested `src/jobs/startWorkers.ts`). `index.ts:103-113` calls it once instead of the three bare calls. It takes a list of `{ queue, start }` pairs; the default is the three real start functions. It returns a promise that resolves when all workers are registered. Rules:
   - Attempt each start function. A worker that registered is never started again in this process. Retry only the failed ones.
   - Within one attempt, call the failed start functions together (`Promise.allSettled`), so they share one `getQueue()` start and one connect timeout.
   - Waits between attempts: 2, 4, 8, 16, 30, 30, 30 s (base 2 s, doubling, cap 30 s; 8 attempts; about 2 minutes of waiting). A refused connection fails at once; an unreachable host can add up to pg-boss's 10 s connect timeout per attempt (§3). These are constants in code. The function also takes them as options, plus an injected `sleep`, so tests run fast. No new environment variable.
   - Each failure logs the existing event name (`gmail_worker_start_failed`, `email_worker_start_failed`, `notification_worker_start_failed`) with `queue`, `attempt`, `maxAttempts`, `retryInMs` (null on the last attempt), `category` and, when allowed, `code` (item 5).
   - When all three are registered, log `{ event: 'workers_ready', queues, attempts }` once.
   - After the last failed attempt, log `{ event: 'worker_start_gave_up', queues: <failed names>, attempts }`, call `stopQueue()` with errors ignored, then exit with code 1. The exit goes through an injected `onGiveUp` (default `process.exit(1)`), so tests do not exit.
   - Once shutdown has begun (`index.ts:119-131`), start no new attempt and do not call `onGiveUp`. Pass an `isShuttingDown()` check or an `AbortSignal`.
   - The HTTP server still starts at once, as today. During the retry window the API works and `/ready` says not ready.
3. **`worker_registered` for every worker.** Add the email worker's log line (`emailProcessingJob.ts:204-206`) to `startGmailSyncWorker` and `startNotificationWorker`, after `queue.work` resolves. Do not change their signatures, queue options or handlers.
4. **Non-starting queue accessor.** In `queue.ts`, keep the started instance in a variable that is set only after `createQueue` succeeds and cleared in the catch (:15-18) and in `stopQueue` (:25-29). Export `getStartedQueue(): PgBoss | undefined`; it never starts pg-boss. Export the three queue names (today a literal list at :12) as one constant, used by `getQueue`, by the default `startWorkers` list and by readiness. Read the new exports only inside functions, never at module load (§7 Compatibility).
5. **Error category in logs.** One small helper (suggested `errorCategory(err)` in `queue.ts` or `src/utils/`) returns `{ category, code? }`. `category` is `err.name` for an `Error`, else `'UnknownError'` (the pattern at `gmailSyncJob.ts:66`). `code` is included only when `err.code` is a string matching `^[A-Z0-9_]{1,40}$`: Node codes such as `ECONNREFUSED`, or PostgreSQL SQLSTATE codes such as `3D000` (no database), `28P01` (bad password), `57P03` (still starting), `42501` (no permission, for example to create the `pgboss` schema). Never log `message` or `stack`. Use the helper in `startWorkers` and in the `queue_error` listener (`queue.ts:9`).
6. **Decision: readiness on `/health` or a new `/ready`. Recommendation: add `GET /ready`; keep `/health` as it is.** Reason: `/health` stays a cheap liveness check with the same 200 body, which is what a load balancer health check needs (§7). Readiness gets its own meaning and status code.
   - `/ready` reads in-memory state only: `getStartedQueue()?.getWipData()`. A queue counts as `registered` when it has a worker in state `created` or `active`. No database query; it never starts pg-boss.
   - `/ready` checks the same queue list that the start-up routine registers (today the three names). Keep that list in one place, so S7-05 can add its queue when its schedule is enabled.
   - All three registered: 200 `{ "status": "ready", "workers": { "gmail-sync-job": "registered", "email-processing-job": "registered", "discord-notification-job": "registered" } }`. Otherwise: 503 with `"status": "not_ready"` and `"not_registered"` for each missing queue. Header `Cache-Control: no-store`. No error text, counts, timestamps or user data.
   - `/health` keeps its status code and body.
   - Mount both right after `/mcp` (`index.ts:37`), before CORS, JSON, cookies and session (`index.ts:41-70`), so neither touches the session store. Suggested file: `src/routes/health.ts`.
7. **Tests** as in §11. They replace the TEST-16 placeholder in `src/tests/health.test.ts`.
8. **Docs** as in §12.

## 5. Out of scope

- A shutdown deadline, draining in-flight work on exit, or closing the session pool ([release track](../README.md#7-release-track-not-scheduled) item 3). The give-up path only calls `stopQueue()` and exits.
- The scheduler and its start-up catch-up (S7-05). This ticket only gives S7-05 something to wait for.
- Separate worker processes, more than one instance, a process supervisor, a Docker image or load balancer settings (OD-08; release track item 1).
- A database check in `/ready` or `/health`, alerting, metrics.
- Sync terminal states, setting `syncError` when the lease expires, sync events and crash recovery of a sync ([S5-FU-01](../sprint-5/follow-up-gmail-reliability.md)).
- The frontend placeholder `src/App.test.tsx` (the other half of TEST-16; frontend, stays noted).
- The real pg-boss start and leftover notification jobs in `matcher.test.ts` (TEST-15, [roadmap §8](../README.md#8-not-now)).

## 6. Likely files and components

- backend: `src/index.ts`, `src/services/queue.ts`, `src/jobs/startWorkers.ts` (new), `src/routes/health.ts` (new), `src/jobs/gmailSyncJob.ts` and `src/jobs/notificationJob.ts` (one log line each), `src/tests/health.test.ts` (rewritten), `src/tests/worker-startup.test.ts` (new), `src/tests/queue.test.ts`, `README.md`, `STABILIZATION.md`.
- docs: see §12.
- frontend: none.

## 7. Implementation notes

- **Data model / migrations:** none. No new queue, no change to the `pgboss` schema.
- **API and shared contracts:** new `GET /ready`. It is an operator endpoint: not under `/api`, not in `src/contracts`, not used by the frontend. `/health` keeps its response and only moves earlier in the middleware order. No contract file changes, so frontend `npm run sync-contracts` stays at 0 changed. In development, call `/ready` on port 3000 directly; the Vite proxy does not forward it.
- **Background jobs:** handlers, queue options, `batchSize` and retry settings are unchanged. Each queue still gets exactly one worker. Jobs queued during the retry window run once the workers register, because `getQueue()` shares one instance between enqueue and `startWorkers`.
- **Failure and recovery:**
  - Database late by less than about 2 minutes: the workers register on their own.
  - Database late by more: exit 1. Locally, `npm run dev` then waits for a file change and `npm start` ends; the owner restarts it. In a deployment, the platform restarts the process (release track). Queued jobs stay in `pgboss.job` (unverified, §3).
  - Database outage after start: workers keep polling; `queue_error` lines now carry a category; `/ready` still says ready (known limit).
  - Partial start: if one worker registered and holds a job when the routine gives up, `stopQueue()` waits up to pg-boss's 30 s stop timeout, and a longer AI call can be cut. This needs a partial start, which is unlikely because all three share one `getQueue()`. The general fix is the release track shutdown deadline.
- **Load balancer (high-level-architecture §9.5, `:521`, `:536`):** in a single API+worker process, keep the load balancer on `/health`. Pointing it at `/ready` would take the only API target out of rotation during the start-up window. Use `/ready` as the deploy-time check ("verify health" in backend `STABILIZATION.md:11`) and for the owner. §9.5 also plans a separate worker service with no HTTP server (`:524-530`); for that layout, the process exit is the signal. ADR-0004 decides; not in scope.
- **S7-05 hook:** S7-05 registers its schedule and fan-out worker inside this routine, so they get the same retry-then-exit rule, and adds its queue to the list `/ready` checks when the schedule is enabled ([S7-05](S7-05-twice-daily-scheduled-sync.md) §4 item 4). Keep the routine open to one more `{ queue, start }` pair; build nothing for S7-05 here.
- **Compatibility:**
  - `NODE_ENV=test` still skips the automatic start (`index.ts:103`). The smoke and the tests that call the start functions directly see no change. Existing log event names stay; only fields are added.
  - `ai-redaction.test.ts` and `ai-settings.test.ts` import the app while mocking `../services/queue` without the new exports (§3). If `index.ts`, `startWorkers.ts` or `routes/health.ts` reads `getStartedQueue` or the queue-name constant at module load, both files fail at import. Read them only inside functions (the handler and `startWorkers`). These two files and `notificationJob.test.ts` must pass without changes.
- **Rollback:** revert the commit. No data, schema or configuration change.

## 8. Dependencies

- Phase 0 gate passed and Sprint 6 closeout exit criteria met ([Sprint 7 README](README.md) §5). Until this ticket ships, the Phase 0 interim check stands: start PostgreSQL first and restart the backend if a `*_worker_start_failed` line appears (MV-11, [Phase 0](../migration-verification/README.md)).
- No owner decision. OD-08 (no deployment) keeps load balancer and supervisor work out.
- Helpful first: [S7-01](S7-01-ci-and-toolchain-pins.md), so the new tests run on every push; [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md) TEST-05 env pinning, because `health.test.ts` imports the app.
- Coordinate with [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md): both touch `gmailSyncJob.ts`. This ticket adds one log line to `startGmailSyncWorker`; the merge conflict, if any, is trivial.
- Blocks [S7-05](S7-05-twice-daily-scheduled-sync.md).

## 9. Security and privacy

- Logs carry only the error `name` and an allowlisted short `code`. PostgreSQL and Node error messages can hold the database host, port, user and database name (for example `connect ECONNREFUSED 127.0.0.1:5432`, or PostgreSQL's "password authentication failed for user …"). Whether a malformed `DATABASE_URL` error can echo the password is unverified; the rule covers it either way. Message and stack are never logged.
- `/ready` has no authentication and needs none: read-only, no database query, fixed queue names and two state words. No user IDs, counts, timings or error text. It cannot start pg-boss or any job. If the backend is ever exposed, the release track can keep `/ready` internal.
- `/health` and `/ready` sit before cookies and session, so they never read or set a session.
- No new environment variable or secret. The give-up path prints no configuration.

## 10. Acceptance criteria

- [ ] `index.ts` calls `startWorkers` once, only outside `NODE_ENV=test`, and no longer calls the three start functions directly.
- [ ] Each failed attempt logs the existing per-worker event with `queue`, `attempt`, `maxAttempts`, `retryInMs`, `category` and, when allowed, `code`; no line contains an error message or stack (unit test with a planted secret).
- [ ] Waits between attempts are 2, 4, 8, 16, 30, 30, 30 s, with at most 8 attempts (unit test).
- [ ] A registered worker is never started again; only failed workers are retried (unit test).
- [ ] After the last failed attempt, `worker_start_gave_up` names the failed queues and the process exits with code 1 (unit test of `onGiveUp`; manual check 1).
- [ ] When PostgreSQL comes up inside the window, all three workers register, `workers_ready` is logged once and `/ready` returns 200 (manual check 2).
- [ ] No attempt starts after shutdown begins, and shutdown still exits 0 (unit test; review of `index.ts`).
- [ ] `startGmailSyncWorker` and `startNotificationWorker` log `worker_registered`; their signatures and queue options are unchanged (health test case 4; review).
- [ ] `queue_error` lines carry `category` and, when allowed, `code` (review; the helper is unit-tested).
- [ ] `GET /health` returns 200 with the unchanged body and is mounted before CORS, cookies and session: neither `/health` nor `/ready` returns an `Access-Control-Allow-Origin` header (test).
- [ ] `GET /ready` returns 503 `not_ready` before workers are registered, including when pg-boss runs with no workers; 200 `ready` when all three are registered; 503 naming the one missing queue after it stops. It never starts pg-boss.
- [ ] A failed `getQueue()` leaves no started instance, and the next call starts a fresh one (test).
- [ ] `src/tests/health.test.ts` has no `expect(true)` placeholder.
- [ ] `email-worker-reliability.test.ts`, `notificationJob.test.ts`, `ai-redaction.test.ts` and `ai-settings.test.ts` pass without changes; frontend `sync-contracts` reports 0 changed.

## 11. Testing

Tests to add or change:

- **`src/tests/worker-startup.test.ts` (new; no database queries).** Fake start functions, a fake `sleep` that records waits, a fake `onGiveUp` and `isShuttingDown`, and a console spy:
  - all three succeed at once: each called once, no wait, one `workers_ready`;
  - a shared failure (`Object.assign(new Error('connect failed postgresql://cc:secret@127.0.0.1/x'), { code: 'ECONNREFUSED' })`) on attempts 1-2, then success: waits `[2000, 4000]`; failure lines carry `category`, `code`, `attempt`, `retryInMs`; no log line contains `secret` or `postgresql://`;
  - one worker always fails: the other two are called once; the failing one 8 times; waits `[2000, 4000, 8000, 16000, 30000, 30000, 30000]`; last `retryInMs` is null; `worker_start_gave_up` lists only that queue; `onGiveUp` called once;
  - shutdown begins during a wait: no further start call, `onGiveUp` not called;
  - `errorCategory`: a non-Error gives `UnknownError`; a lower-case or over-long `code` is dropped; `3D000` is kept.
- **`src/tests/health.test.ts` (rewritten; the last cases use the test database).** In this order:
  1. `GET /health` returns 200 and the unchanged body. `/health` and `/ready` responses have no `Access-Control-Allow-Origin` header. The `cors` middleware sets it on every response when the origin is a fixed string (`node_modules/cors/lib/index.js:47-52`), so its absence proves both routes run before CORS.
  2. `GET /ready` before any queue start returns 503, all three `not_registered`; `getStartedQueue()` is still undefined afterwards.
  3. After `getQueue()` with no workers (the PLAT-07 state): 503, all `not_registered`.
  4. Delete leftover `pgboss.job` rows for the three queue names (TEST-15 leaves notification jobs; same pattern as `email-worker-reliability.test.ts:312`). Call the three real start functions; a console spy sees one `worker_registered` line per queue. `GET /ready` returns 200 `ready` with `Cache-Control: no-store`.
  5. `offWork('discord-notification-job', { wait: true })`: 503, and only that queue is `not_registered`.
  - `afterAll`: `offWork` the rest with `wait: true`, then `stopQueue()`.
- **`src/tests/queue.test.ts` (one test added).** With no instance started (call `stopQueue()` first), set `DATABASE_URL` to `postgresql://cc:x@127.0.0.1:1/career_companion_closed_test?schema=public` (nothing should listen on port 1; the 10 s connect timeout bounds it if not). Expect `getQueue()` to reject and `getStartedQueue()` to be undefined. Record the error's `category` and `code` (expected `ECONNREFUSED`; unverified). Restore the URL in `finally`; then `getQueue()` resolves with a fresh instance.

Commands (backend):

```bash
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/worker-startup.test.ts src/tests/health.test.ts src/tests/queue.test.ts
npx vitest run src/tests/email-worker-reliability.test.ts src/tests/notificationJob.test.ts src/tests/ai-redaction.test.ts src/tests/ai-settings.test.ts
npm test        # previous total minus 1 placeholder plus the new tests; record the count
```

Frontend: no code change. Run `npm run sync-contracts` once and expect 0 files changed.

Smoke: not required; nothing visible in the browser changes, and the smoke's imports keep their names. Run it if any of those exports changed shape.

Manual checks on the new PC (record the output in the execution report):

1. Stop the local PostgreSQL service. Run `npm run build && npm start` with the dev `.env`. Expect failure lines with `category` and `code` and growing `retryInMs`; `curl -i http://localhost:3000/health` gives 200 and `curl -i http://localhost:3000/ready` gives 503. After about 2 minutes, `worker_start_gave_up` is logged and `echo $?` prints 1.
2. Stop PostgreSQL, start the backend, then start PostgreSQL within one minute. Expect three `worker_registered` lines, one `workers_ready`, and `/ready` gives 200.
3. Normal start with PostgreSQL up: `/ready` gives 200 within a few seconds.

## 12. Documentation updates

- Backend `README.md` "Running the Server" (:54-56): start PostgreSQL first; workers retry for about 2 minutes and then the process exits with code 1; after that exit `npm run dev` waits for a file change, so restart it; what `/health` and `/ready` mean, with one `curl` example.
- Backend `STABILIZATION.md`: release step 5 (:11) "verify health" becomes "`GET /ready` returns 200 (all three workers registered); `/health` only shows that the HTTP process is up". In "Privacy and diagnostics" (:143-154), list `worker_registered`, `workers_ready`, `*_worker_start_failed`, `worker_start_gave_up` and `queue_error` with `category`/`code`, and say they never carry error text.
- [operations.md](../sprint-5/operations.md):7 says `worker_registered` "proves registration in that process, not ongoing health". Add that `/ready` reports the same thing for the running process and does not check the database. Edit after the S6-C02 banner on that file.
- [high-level-architecture.md](../../architecture/high-level-architecture.md) §9.5 (:521, :536): one note that `/health` is liveness only, `/ready` reports worker registration (S7-04), and ADR-0004 picks what the load balancer checks.
- [Phase 0](../migration-verification/README.md) MV-11: a dated note that S7-04 replaces the manual "look for `*_worker_start_failed` and restart" check.
- [Sprint 7 README](README.md) §7: tick the S7-04 exit criterion with evidence links. Sprint 7 `execution-report.md`: test counts and the manual check output.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend typecheck, lint, build and the full test suite are green (in CI once S7-01 exists).
- [ ] Behavior verified with the three manual checks in §11 on the new PC.
- [ ] Docs updated per §12.
- [ ] Focused commits: one backend commit (code, tests, backend `README.md` and `STABILIZATION.md`), one docs-repo commit (OD-02).
- [ ] Evidence (test counts, manual check output) recorded in the Sprint 7 `execution-report.md`. Nothing synthetic is called live evidence.
