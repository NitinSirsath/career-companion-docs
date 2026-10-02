# S5-FU-01 — Complete the remaining Sprint 5 Gmail sync reliability work

> **Rescoped for Sprint 7 (2026-10-02):** Sprint 7 executes FU-1, FU-4.1, FU-4.2, FU-4.4 and the minimal sync events in [section 7](#7-sprint-7-scope-rescoped-2026-10-02) below. FU-2 and FU-4.3 moved to [S5-FU-02](follow-up-google-request-bounds.md) (Sprint 8). The FU-3 counter equations, FU-4.5 and FU-5 are deferred ([roadmap §8, Not now](../README.md#8-not-now)); FU-5 depends on OD-13. The §4 order "FU-5 first" is superseded: FU-1 goes first. See the [roadmap](../README.md) and [Sprint 7](../sprint-7/README.md). Sections 1–6 are the original record and stay unchanged; where they differ from section 7, section 7 wins.

| Field | Value |
| --- | --- |
| Status | **Proposed follow-up — not started.** Recorded 2026-10-02 at the owner's request. Nothing here is implemented or verified. |
| Origin | Sprint 5 [S5-01](S5-01.md), [S5-03](S5-03.md), [S5-05](S5-05.md); [architecture review](architecture-review.md) gaps G1–G4 and G7 |
| Repository | career-companion-backend (main); career-companion-frontend only for the existing smoke harness |
| Sprint 6 impact | None. This does not change Sprint 6 scope or acceptance. |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed |

## 1. Problem and background

The [Sprint 5 execution report](execution-report.md) describes Gmail reliability work that was implemented in an uncommitted working tree. The downloaded `career-companion-*-main` folders used for Sprint 6 do not contain that work. On 2026-10-02 only the Sprint 5 pieces Sprint 6 depended on were re-implemented: safe email retry, email worker outcomes, client deadline, bounded refresh and the real-worker harness ([Sprint 6 execution report §1](../sprint-6/execution-report.md#1-baseline-and-sprint-5-reconciliation-phase-1)). The Gmail sync path itself is still at its pre-Sprint-5 state.

Current implementation (source-checked 2026-10-02; `gmailSyncJob.ts`, `gmailSync.ts`, `gmailClient.ts`, `gmailFetcher.ts`, `routes/gmail.ts`):

- **Claim mismatch after a crash (G1).** `requestGmailSync` stores `queued:<uuid>` and queues `{ userId, claim }`. `syncUser` swaps it for a random claim. The catch block restores the queued claim, but a killed process never reaches it. The pg-boss retry then carries a claim that no longer matches, `syncUser` throws `SyncInProgressError`, and the worker swallows that error. The job is acknowledged as done although nothing was synced. Recovery waits for the 5-minute lease to expire and a manual sync.
- **Stale writes and false success (G2).** `heartbeat()` and the email `upsert` are separate statements, so a superseded attempt can still insert rows between them. The final checkpoint `updateMany` ignores its affected-row count: a superseded or disconnected attempt still logs `gmail_sync_completed` and returns success. Pending recovery (oldest 100 PENDING rows) runs without heartbeat or lease checks.
- **Unbounded transport (G3).** Gmail calls pass `{ timeout: 15_000 }`, but googleapis/gaxios retries are not disabled, so one call can take several times 15 s. OAuth refresh inside `withGmail` has no explicit timeout. The 4-minute budget is only checked cooperatively in `heartbeat()`, and the worker ignores pg-boss abort signals.
- **Misleading telemetry (G4).** There is no sync start event. `gmail_sync_completed` reports only `messagesIngested`, `messagesSkipped` and duration. `messagesIngested` increments only after a successful enqueue, so a row persisted before an enqueue failure is invisible. 404, non-INBOX and too-old candidates disappear from all counts. Repeated references are not distinguished. `enqueueEmailProcessingJob` now returns the job ID or `null` (Sprint 6), but sync ignores it, so singleton-suppressed offers are not reported. The failure log carries only `jobId` and the error class name: no mode, attempt, retry, counts or duration. A queue send failure stores `QUEUE_UNAVAILABLE` but the API returns a generic 500.
- **Test coverage.** `gmail-ingestion.test.ts` mocks `googleapis` and the enqueue function (8 scenarios: history path, enqueue gap, expired history and paging, later-page failure, lease exclusion, quota 403, refreshed tokens, concurrent disconnect). `gmailSync.test.ts` still tests a copied "401 or 403 means auth" condition instead of the real `googleAuthFailure` helper. There is no process-kill, queue-redelivery, supersession, transport-hang or capacity test. The Sprint 6 smoke covers the happy path, repeat sync and one 503 failure through real workers, not crashes.
- **No baseline tooling (S5-01).** Only `scripts/guarded-migrate.cjs` and `scripts/verify-migration-preservation.cjs` exist. Neither captures or compares a private owner snapshot of a preserved database.

## 2. Why it was deferred

- The owner instructed (2026-10-02) not to rebuild Sprint 5 wholesale. Only Sprint 5 items that Sprint 6 depended on were to be completed.
- Sprint 6 does not change Gmail ingestion. Its tickets consume email/AI/domain outcomes, and its real-worker smoke passed without these items.
- The prior implementation could not be recovered: the working tree was not in the downloaded baseline, and searching for it was out of scope.
- The S5-02 live-evidence gate (real Gmail and original-data preservation) needs an authorized environment that is not available. Work items 1–4 make that later evidence trustworthy; item 5 is a prerequisite for capturing it.

## 3. Exact remaining scope

Keep the existing architecture: manual, additive ingestion of INBOX mail within the configured 1/7/14/30-day lookback (default 1), pg-boss, PostgreSQL claims/leases and current tables. Add no scheduler, push subscription, outbox, new provider, telemetry service or schema redesign. A migration is allowed only if a measured failure proves the existing columns insufficient, and then needs a data-preservation plan.

### FU-1 — Sync fencing and concurrency protection (G1, G2)

1. Keep one logical request identity across pg-boss retries, with a unique per-attempt suffix inside the existing `syncClaim` (for example `queued:<request>:attempt:<attempt>`). An expired attempt of the same request can be reclaimed by its retry. An active same-request delivery must retry, not be acknowledged. An obsolete request (newer request or disconnect) ends with an explicit superseded outcome, never as success.
2. Read fresh settings and checkpoint at acquisition, under a short connection-row-locked transaction with a database-clock lease check.
3. Move each email insert into a short transaction that locks the connection row and checks exact attempt ownership, unexpired lease and connected status before writing. No Google or queue call inside any transaction. Bound lock and statement time.
4. Report success only when the final checkpoint update affects exactly one row owned by the attempt. Otherwise report superseded or failed, and leave the checkpoint unchanged.
5. Give pending recovery the same deadline and ownership checks.
6. Keep: decimal-string history IDs, commit only after all pages and handoffs, 404 history reconciliation without deleting rows, owner-scoped `(userId, gmailMessageId)` uniqueness, and no reset of completed, failed or processing emails.

### FU-2 — Bounded Google/Gmail API requests (G3)

1. Route Gmail calls and OAuth refresh through a transport with an explicit per-request bound and SDK/gaxios retries disabled. pg-boss stays the single transient-retry policy.
2. Propagate one absolute sync deadline and pg-boss cancellation into those calls and between pages.
3. Keep auth versus quota classification: 401 or `invalid_grant` revokes; 403 quota or rate limits never revoke or force refresh. Save or revoke tokens with credential compare-and-swap in bounded transactions.
4. Apply the same bounds to `GmailFetcherService` metadata and body fetches used by the email worker. Do not change AI behavior.

### FU-3 — Sync counters, telemetry and logging (G4)

1. Emit correlated `gmail_sync_queued`, `gmail_sync_started` and a terminal `gmail_sync_completed`, `gmail_sync_failed` or `gmail_sync_superseded`, plus `gmail_sync_job_outcome`. Include user, request, job and attempt IDs, retry count/limit, queue wait, mode (history or reconciliation), duration, `checkpointCommitted` and `checkpointAdvanced`.
2. Record exactly one disposition per unique candidate. These counts must reconcile:
   - `references = uniqueCandidates + duplicateReferences`
   - `uniqueCandidates = existing + persistedNew + notFound + filteredLabel + filteredAge + invalidMetadata + unresolved + persistenceUnknown`
   - `queueOffers = queueAccepted + queueSuppressed + queueUnknown`
   - `persistedNew` counts committed inserts even when the enqueue later fails.
3. Use allowlisted failure categories, for example AUTH_REVOKED, RATE_LIMIT, PROVIDER_UNAVAILABLE, NETWORK_ERROR, REQUEST_TIMEOUT, FORBIDDEN, REQUEST_BUSY, SUPERSEDED, DEADLINE_EXCEEDED, CANCELLED and QUEUE_UNAVAILABLE. Return a distinguishable API error for a queue send failure, coordinated with the existing frontend handling.
4. Never log message bodies, snippets, subjects, addresses, tokens, cursors or history IDs.
5. Add read-only operator SQL and interpretation in [operations.md](operations.md), which already describes this target contract, after verifying it against the new code.

### FU-4 — Gmail sync crash and failure test coverage

1. Add an isolated process-kill regression. A child worker acquires an attempt and is SIGKILLed. Only the fixture job and lease clocks are aged. pg-boss expiry redelivers the original job: `retryCount` becomes 1, a fresh worker commits the checkpoint and the email exactly once, and a repeat sync keeps one row.
2. Add DB-backed tests for: same-request busy, obsolete-request supersession, pause/replace/resume across an email insert, zero-row finalization, cancellation and deadline, partial page failure with counter reconciliation, pending recovery of more than 100 rows, and duplicate references across pages.
3. Add a loopback-HTTP transport test using the installed googleapis/google-auth-library stack: reactive 401 refresh, quota with no hidden retry, a hung refresh hitting its deadline, and cancellation.
4. Replace the copied 401/403 assertion in `gmailSync.test.ts` with tests of the real `googleAuthFailure`.
5. Add a representative capacity fixture: about 1,620 existing completed emails plus a new item across several pages. Only the new item is fetched and inserted, no completed work is re-queued, and the run finishes within the 4-minute budget.
6. Extend the existing smoke only if a browser-visible behavior changes. Keep its guarded teardown.

### FU-5 — Sync snapshot and baseline tooling (S5-01)

1. Add a read-only snapshot tool. It inspects the schema first (applied migrations and optional stabilization or Sprint 6 columns/tables) and runs no migration and starts no worker.
2. It captures, in one REPEATABLE READ, READ ONLY transaction, an owner-scoped manifest of: email identity and immutable metadata, user match decisions, application user status, completed AI results and operations, domain event/action keys, and sync checkpoint state. It excludes secrets.
3. Output goes to a private file (mode 0600) that never overwrites an existing path.
4. Add a compare command that reports missing or changed identities and fields, duplicate `(userId, gmailMessageId)`, changed user decisions and completed-AI deltas, with explained-delta support.
5. Follow the [verification runbook](verification-runbook.md) baseline section. The tool must never reset, seed or migrate the preserved database.

## 4. Dependencies and order

- Starts from the current downloaded baseline with Sprint 6 changes in place: guarded test and smoke lanes, `guarded-migrate.cjs` and the real-worker harness.
- Order: FU-5 first, because it can be built independently and is needed before any live check. Then FU-1, then FU-3 (telemetry reports FU-1's outcomes). FU-2 can run in parallel with FU-1. FU-4 tests are written alongside each change; the crash regression needs FU-1.
- Needs exclusive guarded databases (`career_companion_*test`, plus a separate crash lane) and Chrome only if the smoke is touched.
- Owner prioritization is required before scheduling. The recorded twice-daily scheduled sync decision is separate work, but any future scheduler should build on FU-1 fencing.
- The S5-02 live Gmail and original-dataset evidence remains a separate gate. It needs FU-1 to FU-5 on the same build plus an authorized environment.

## 5. Acceptance criteria

- [ ] A process killed mid-sync is recovered by the same logical request's retry. No busy or obsolete delivery is acknowledged as successful ingestion.
- [ ] A superseded or disconnected attempt cannot insert emails, advance the checkpoint, or log or return success. Success requires exactly one owned checkpoint row update.
- [ ] Every Gmail and OAuth request has an enforced bound, with no hidden SDK retries. One sync attempt has an absolute deadline and honors cancellation. Quota failures never revoke credentials.
- [ ] Lifecycle events correlate request, job and attempt. Counter equations reconcile for full, partial and failed runs. Logs contain no Gmail content, tokens or cursors.
- [ ] A queue-send failure is reported distinctly. The UI still recovers via its existing retry path.
- [ ] Repeat syncs keep one row per Gmail ID and add no AI calls for completed work. Stored rows and user decisions survive narrower lookback and reconciliation.
- [ ] The snapshot and compare tools capture and compare a preserved-style database read-only, without migrating it, and detect injected identity and decision changes.
- [ ] The existing backend and frontend suites, Sprint 6 smoke and migration-preservation lanes still pass. Sprint 6 behavior is unchanged.
- [ ] Documentation (operations.md, backend STABILIZATION/README and this record) states delivered behavior. Live-provider evidence stays explicitly unverified unless actually obtained.

## 6. Verification requirements

- Backend: `npm run db:generate`, `typecheck`, `lint` (0 errors), `build`, guarded migration via `node scripts/guarded-migrate.cjs`, then full `npx vitest run` on the guarded test lane.
- Focused: `gmail-ingestion.test.ts`, `gmailSync.test.ts`, `queue.test.ts`, the new fencing, transport and telemetry tests, `email-worker-reliability.test.ts`, `stabilization-safety.test.ts`.
- Crash regression in its own exclusive empty `career_companion_*_crash_test` lane, with workers stopped before cleanup.
- Snapshot tool tests on a disposable copy, including schema-detection and private-file safety, plus a read-only proof (no writes or migrations recorded).
- Frontend: `npm run sync-contracts` if contracts change, `typecheck`, `lint`, `npm test`, `build`, and `node scripts/smoke-stabilization.mjs` on an exclusive smoke lane.
- `node scripts/verify-migration-preservation.cjs` if any migration is added.
- Record results in a new execution section of this file. Do not claim live Gmail or original-data evidence from fixtures.

## 7. Sprint 7 scope (rescoped 2026-10-02)

This section is the executable Sprint 7 ticket. It can be read on its own.

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 7 — order 5 of 6 |
| Repository | career-companion-backend (main). Frontend: no code change; its checks run as a regression. |
| Size / priority | L / the only L ticket in Sprint 7; must land before the scheduler |
| Depends on | S7-01 (CI first); Sprint 7 entry (Phase 0 gate, Sprint 6 closeout); rebases on S7-02; coordinates with S7-04. No owner decision. |
| Blocks | S7-05; S5-FU-02 (Sprint 8); the pre-release live Gmail/original-data gate that OD-01 moves behind this ticket |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-05, S56-07 (minimal part), S56-24, TEST-12; §3 FU-1 and FU-4 above; [Sprint 7 README](../sprint-7/README.md) §3, §6, §7; [roadmap](../README.md) §3, §8 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

### 7.1 Objective

A sync job killed mid-run is finished by the retry of the same request. No busy, obsolete, superseded or disconnected attempt is ever acknowledged, logged or returned as a successful sync. Each attempt writes one start event and one end event that say what happened, with no Gmail content. The Gmail auth-detection tests call the real code.

### 7.2 Why now

Facts, re-checked in the code on 2026-10-02 (backend = `career-companion-backend-main`):

- **G1 is still present.** `requestGmailSync` (`src/jobs/gmailSyncJob.ts:11-53`) stores `queued:<uuid>` (:12) and sends `{ userId, claim }` with `retryLimit: 3`, `retryDelay: 60`, `retryBackoff: true`, `expireInSeconds: 300` (:28-38). `GmailSyncService.syncUser` (`src/services/gmailSync.ts:79-277`) swaps that claim for a fresh `randomUUID()` (:95-116). Only the catch block (:263-276) puts the queued claim back (:269). After a process kill, pg-boss expires and retries the job with the old claim. The claim filter (:100-101) matches no row, `SyncInProgressError` is thrown (:117), and `startGmailSyncWorker` swallows it (`gmailSyncJob.ts:69`). The job is acknowledged with nothing synced. `GET /api/gmail/status` shows FAILED once the lease expires (`src/routes/gmail.ts:101-105`), with `syncError` null because acquisition cleared it (`gmailSync.ts:114`). Recovery needs a manual click.
- **G2 is still present.** The ownership heartbeat (:165) and the email `upsert` (:166-176) are separate statements. Pending recovery (`reofferPendingEmails`, :31-41, called at :240) has no lease or ownership check. The final checkpoint `updateMany` (:242-252) ignores its row count. The code then logs `gmail_sync_completed` (:253-261) and returns `synced: true` (:262), even when no row was updated.
- **Thin telemetry (S56-07).** Only `gmail_sync_completed` and `gmail_sync_failed` exist. The failure line (`gmailSyncJob.ts:62-68`) carries `jobId` and `category: err.name` only: no user, attempt or retry number. `err.name` is `Error` for most failures, including the supersede and time-budget errors (`gmailSync.ts:123`, :128). A queue-send failure stores `QUEUE_UNAVAILABLE` (`gmailSyncJob.ts:41-51`), but the route calls `next(err)` (`routes/gmail.ts:415`), which answers a generic 500 (`src/middleware/error.ts:40-45`).
- **Circular tests (S56-24, TEST-12).** `src/tests/gmailSync.test.ts:80-123` checks an inline copy, `err.status === 401 || err.status === 403` (:98-99, :119-120), and imports nothing from `gmailClient.ts` (:3-4). The real `googleAuthFailure` (`src/services/gmailClient.ts:10-13`) treats only 401 or `invalid_grant` as auth. No test calls it directly; the `invalid_grant` branch has no test at all. Real behavior is covered only indirectly (`gmail.test.ts:505-566` for 401, `gmail-ingestion.test.ts:166-172` for quota 403).
- **No crash or redelivery test.** `gmail-ingestion.test.ts:147-165` only covers taking over an expired claim.
- **Correction to §1:** pending recovery takes the *newest* 100 PENDING rows (`gmailSync.ts:35`), not the oldest.

S7-05 puts an unattended twice-daily sync on top of this path ([Sprint 7 README](../sprint-7/README.md) §6). With G1, a killed scheduled run is acknowledged and nothing retries it until the next slot, hours later. §4 above already says any scheduler must build on FU-1 fencing.

Suspected risks, not demonstrated defects (S56-05 verifier): G2's practical harm is small today. A stale insert needs ownership to change between two statements; the upsert is idempotent (`update: {}`) and disconnect keeps emails, so the worst case is one extra INBOX row. The main harm is a false success log and an acknowledged job. The worker discards `syncUser`'s return value; only tests read it.

### 7.3 In scope

1. **FU-1 items 1–6, as written in §3.** Concretely:
   - **Request and attempt identity (FU-1.1).** Keep `queued:<uuid>` from `requestGmailSync` as the request identity, and keep `claim` in the job data. The attempt claim becomes `queued:<uuid>:attempt:<retryCount>:<random>` (the §3 example plus a random part, so two deliveries never share a claim). Register the worker with `{ includeMetadata: true }` to get `retryCount` and `retryLimit` (pg-boss 12.31.0 `work(name, options, handler)`, `node_modules/pg-boss/dist/index.d.ts:43`; fields at `types.d.ts:852-856`).
   - **Acquisition (FU-1.2)**, decided in one short transaction that locks the connection row and compares the lease with the database clock: (a) the claim is `queued:<uuid>`, or an attempt of the same request whose lease has expired → acquire, and read settings and checkpoint from this locked row; (b) an attempt of the same request with a live lease → throw a new busy error; the worker rethrows it, so pg-boss retries; (c) any other claim, or status NOT_CONNECTED → obsolete: acknowledge with the superseded outcome. Status REVOKED stays an auth failure: `GmailAuthError`, acknowledged as today (`gmailSync.ts:81-82`). Direct calls without a claim (`syncUser(userId)`, used by tests such as `gmail-ingestion.test.ts:76`) keep today's rule.
   - **Owned inserts (FU-1.3), owned finalization (FU-1.4), guarded pending recovery (FU-1.5) and the FU-1.6 keep-list** exactly as in §3. Details in §7.6.
2. **FU-4.1 crash regression** in its own exclusive `career_companion_*_crash_test` lane (design in §7.10).
3. **FU-4.2 DB-backed tests**, adjusted to this scope: the cancellation case moves to S5-FU-02 with FU-2.2; the deadline case tests the existing 4-minute budget; the partial-page case checks the checkpoint and events, not the deferred counter equations. List in §7.10.
4. **FU-4.4.** Replace the two tautological tests in `gmailSync.test.ts:80-123` with direct tests of `googleAuthFailure` and `googleStatus` (`gmailClient.ts:5-13`): 401 → auth; 403 → not auth; 429 → not auth; `{ response: { data: { error: 'invalid_grant' } } }` → auth; the sanitized error `withGmail` throws (`status: 401`, `gmailClient.ts:39-40`) → auth; a plain `Error` → not auth. Also assert `syncError === 'SYNC_FAILED'` in the quota-403 test (`gmail-ingestion.test.ts:166-172`), as S56-24 suggests.
5. **Minimal FU-3 events.**
   - `gmail_sync_started`, logged only after an attempt acquires. Fields: `trigger` (`manual` or `scheduled`), `userId`, `requestId`, `jobId`, `attemptId`, `retryCount`, `retryLimit`.
   - Exactly one terminal event per delivery the worker handles: `gmail_sync_completed`, `gmail_sync_failed` or `gmail_sync_superseded`. Fields: the same as started, plus `durationMs`, `checkpointCommitted` (`true`; `false`; `null` when the final write's result is unknown, as [operations.md](operations.md):15 defines) and `checkpointAdvanced` (new checkpoint differs from the one read at acquisition; neither value is logged). `failed` and `superseded` add `category`. `completed` keeps `messagesIngested`, `messagesSkipped` and S7-02's `windowDays` and `gapCapped`.
   - `category` comes from a fixed list taken from FU-3.3: `AUTH_REVOKED` (`googleAuthFailure` or `GmailAuthError`), `RATE_LIMIT` (429), `FORBIDDEN` (any 403; the sanitized error keeps only the status, `gmailClient.ts:39-40`, so a quota 403 and a policy 403 look the same), `PROVIDER_UNAVAILABLE` (5xx), `DEADLINE_EXCEEDED` (4-minute budget), `REQUEST_BUSY`, `SUPERSEDED`, `QUEUE_UNAVAILABLE`, and `UNCLASSIFIED` for anything else. `NETWORK_ERROR`, `REQUEST_TIMEOUT` and `CANCELLED` are not emitted yet; those failures log `UNCLASSIFIED`. Whether S5-FU-02 maps its new deadline and cancel errors into this list is its call (its §4 item 11).
   - A busy or obsolete delivery has no started event and one terminal event. A killed attempt has a started event and no terminal event; with an expired lease, that is how an operator sees a crash ([operations.md](operations.md):7).
   - `requestGmailSync(userId, trigger = 'manual')` adds `trigger` to the job data. `POST /api/gmail/sync` passes nothing. S7-05 will pass `'scheduled'`. A job without `trigger` counts as `manual`.
   - No `gmail_sync_queued`, `gmail_sync_job_outcome`, queue wait, mode or counter equations (deferred, [roadmap §8](../README.md#8-not-now)).
6. **Queue-send failure API error. Decision: include it; it is small.** `requestGmailSync` keeps its cleanup (`gmailSyncJob.ts:41-51`), then throws a typed error with code `QUEUE_UNAVAILABLE`. The route maps it next to the `GmailAuthError` mapping (`routes/gmail.ts:407-413`) to `503 { error: { code: 'QUEUE_UNAVAILABLE', message: 'Sync could not be started. Try again in a moment.' } }`, and logs one `gmail_sync_failed` with `category: 'QUEUE_UNAVAILABLE'`, `trigger`, `userId` and `requestId`. It is small because it touches two backend files, changes no contract (error codes are not in `src/contracts`; `SyncResponseSchema` at `src/contracts/gmail.ts:42` covers only the 202 body), and needs no frontend change: the page already shows the server's message (frontend `src/api/client.ts:133-140`, `src/routes/gmail.tsx:217-219`) and the FAILED banner after its refresh (`gmail.tsx:106-108` `onSettled`, `:215`). The state audit row for S56-07 and [S5-FU-02](follow-up-google-request-bounds.md) §5 both place this error here, in S5-FU-01.

### 7.4 Out of scope

- FU-2 (bounded transport, gaxios retries off, OAuth refresh bound, cancellation and abort signals, credential compare-and-swap) and FU-4.3 (loopback transport test) → [S5-FU-02](follow-up-google-request-bounds.md).
- The rest of FU-3: counter equations, `persistedNew`, queue offer counts, `gmail_sync_queued`, `gmail_sync_job_outcome`, queue wait, mode. FU-4.5 capacity fixture. FU-5 snapshot tooling (OD-13). All in [roadmap §8](../README.md#8-not-now).
- The scheduler (S7-05), the lookback gap (S7-02), worker start-up retry and readiness (S7-04).
- Any migration or schema change (except the measured-failure case in §7.6), any `src/contracts` change, any frontend code change, extending the smoke.
- Moving the route pre-check (`routes/gmail.ts:387-395`) or `requestGmailSync` (`gmailSyncJob.ts:19`) to the database clock. On one machine the difference is negligible.
- Running the live Gmail/original-data gate (OD-01).

### 7.5 Likely files and components

- Backend: `src/services/gmailSync.ts` (`syncUser`, `reofferPendingEmails`, new typed errors), `src/jobs/gmailSyncJob.ts` (`requestGmailSync`, `startGmailSyncWorker`; export the per-job handler so tests can call it without pg-boss), `src/routes/gmail.ts` (`POST /sync` mapping), `src/services/gmailClient.ts` (`withGmail`: let the new sync errors through, §7.6). Optionally a small `src/services/gmailSyncEvents.ts` that builds event lines from an allowlisted field set.
- Tests: `src/tests/gmail-sync-fencing.test.ts` (new), `src/tests/gmailSync.test.ts`, `src/tests/gmail-ingestion.test.ts`, `src/tests/gmail.test.ts`.
- Crash lane: `scripts/test-gmail-crash.cjs` (new; the name [operations.md](operations.md):71 already uses) and a local `.env.crash.test` (ignored by `.gitignore:5`, `.env.*`).
- Not touched: `prisma/schema.prisma`, `src/contracts/*`, frontend source.

### 7.6 Implementation notes

- **Data model and migration.** None. `syncClaim` is free text (`prisma/schema.prisma:145`). S7-02's `unscannedFrom`/`unscannedUntil` columns will exist by then; keep writing them in the owned final update. Add a migration only if a measured failure proves the columns insufficient (§3), then with a preservation plan and `verify-migration-preservation.cjs`.
- **Transactions.** Three short kinds: acquisition, one per email insert, final checkpoint. Each starts with `SELECT … FROM gmail_connections WHERE id = … FOR UPDATE`, compares and writes the lease with the database clock, and checks exact attempt claim, unexpired lease and `status = 'CONNECTED'`. Each sets `SET LOCAL lock_timeout` and `SET LOCAL statement_timeout` (suggested 2 s and 5 s) and Prisma `{ maxWait: 5000, timeout: 10000 }`, the values of `TX_OPTIONS` in `src/services/externalSubmission.ts:247`. No `src` file sets a lock or statement timeout today. No Google call and no `queue.send`/`enqueueEmailProcessingJob` inside any transaction: fetch Gmail metadata first, insert in the transaction, enqueue after commit. If the enqueue then fails, the sync fails as today and the next sync re-offers the PENDING row (`gmail-ingestion.test.ts:91-100`).
- **Lock order.** The new transactions lock only the connection row, then write. Every other writer of `gmail_connections` is a single statement outside a transaction (`routes/gmail.ts:233`, :311, :346; `gmailSyncJob.ts:13`, :42; `gmailClient.ts:35`, :44), and that table has no trigger. The `email_ownership` trigger (`prisma/migrations/20260926110000_domain_integrity/migration.sql:16-26`) reads `applications` only when `applicationId` is set, which sync never does. No lock cycle is expected. Disconnect (`routes/gmail.ts:311-323`) waits at most one short transaction.
- **Heartbeat and deadline.** The insert transaction replaces the separate heartbeat at :165. The other heartbeats (:134, :189, :209) become the same ownership check with the database clock. The 4-minute budget (:122-123) stays the attempt deadline; it does not bound a single Google call (S5-FU-02).
- **Errors inside `withGmail`.** Fact: `withGmail` (`gmailClient.ts:31-40`) replaces every error thrown inside its callback with `Error('Gmail request failed')` plus a status. That includes the heartbeat errors (`gmailSync.ts:123`, :128) and Prisma errors, because the page loop and inserts run inside the callback (:131-237). The new busy, superseded and deadline errors raised there would reach the worker as a plain `Error` and log `UNCLASSIFIED`. Let them pass through `withGmail` unchanged (keep sanitizing everything else), or record the outcome before throwing. A test covers each case.
- **Pending recovery.** Give `reofferPendingEmails` an optional guard, called before each enqueue on the sync path. `saveSettings` (`src/services/ai/settings.ts:235`) and `checkSettings` (:280) keep calling it without one. Keep the order "pages, then recovery, then commit" (FU-1.6).
- **Outcomes.** Busy → retryable error (rethrown). Obsolete, or zero rows at finalization → superseded error (acknowledged), connection row untouched. An own claim whose lease has expired counts as lost ownership: stop, insert nothing more and leave the checkpoint unchanged; the retry of the expired pg-boss job reclaims it. Auth failure → `GmailAuthError`, acknowledged as today. Other errors → rethrown for pg-boss retry. The catch block keeps writing FAILED only where the claim equals the attempt claim (:265-266) and restores `queued:<uuid>`. Success keeps `{ synced: true, messagesIngested, messagesSkipped, lastSyncedAt }` (`gmail.test.ts:461-464` reads it) and adds `checkpointAdvanced`. `SyncInProgressError` and its 409 mapping (`routes/gmail.ts:400-402`) stay.
- **Background job.** `gmail-sync-job` keeps its name, `singletonKey` and retry options; data gains `trigger`; the worker gains `includeMetadata: true`. Busy deliveries now retry instead of being acknowledged. With `retryLimit: 3` they can run out of retries during a long busy period; that is safe, because the live attempt of the same request finishes the work. Keep S7-04's `worker_registered` line in `startGmailSyncWorker`.
- **Known limit.** A late redelivery that arrives after its own request already completed (claim cleared) is logged as superseded, not completed. It is never logged or returned as success.
- **API and contract.** `202 { accepted: true }` unchanged; a queue-send failure answers 503 `QUEUE_UNAVAILABLE` instead of 500. `src/contracts` unchanged, so frontend `npm run sync-contracts` reports 0 changed.
- **Compatibility and rollback.** Code only. Before switching builds, let in-flight `gmail-sync-job` jobs finish (query at [operations.md](operations.md):24-29). Old job data `{ userId, claim }` works with the new code (`trigger` = `manual`). To roll back, restore the previous build: the old `requestGmailSync` overwrites a new-format claim once its lease expires, and new job data still carries `claim`, so the old worker can read it (with today's G1 behavior). No data to undo.

### 7.7 Dependencies

- **S7-01 first** ([Sprint 7 README](../sprint-7/README.md) §6): CI must guard this large change. Plus the Sprint 7 entry conditions (§5 there).
- **S7-02** ([ticket](../sprint-7/S7-02-no-silent-mail-loss-after-gaps.md)) lands first and edits `syncUser`. Move its window computation into the locked acquisition read, and keep its final-update columns and log fields.
- **S7-04** ([ticket](../sprint-7/S7-04-worker-startup-and-readiness.md)) also edits `startGmailSyncWorker`; merge its log line with the new `queue.work` options.
- Helpful, not blocking: S6-C01's lock-wait test pattern (`pg_stat_activity` with `pg_blocking_pids`, no fixed sleep); the S6-C02 "target contract" banner on [operations.md](operations.md), which the doc edit here builds on.
- Exclusive guarded databases: the test lane (`career_companion_*test`), a separate crash lane, and the smoke lane with Chrome for the regression run.

### 7.8 Security and privacy

- Event lines use an allowlist. Never log subjects, senders, snippets, bodies, Gmail message or thread IDs, history IDs, page tokens, OAuth tokens, or Google error text. `userId` is an internal UUID and is already logged today (`gmailSync.ts:256`). A test checks this.
- Every new statement is scoped by connection `id` and `userId`. Owner-scoped `(userId, gmailMessageId)` uniqueness is unchanged. A superseded attempt writes nothing.
- The 503 message is fixed text, with no internal detail or stack.
- The crash lane uses a guarded empty database, fixture tokens encrypted with a fixture key, and a fake Gmail. It makes no outbound request and never touches the dev database or a preserved dataset.
- No new production secret or environment variable.

### 7.9 Acceptance criteria

- [ ] The crash lane passes: a worker SIGKILLed after acquisition is recovered by the same request's redelivery (same job ID, `retryCount` 1). The fresh attempt commits the checkpoint and the email exactly once. A repeat sync keeps one row. The lane is empty afterwards.
- [ ] A same-request delivery with a live lease is retried by pg-boss, not acknowledged. An obsolete delivery is acknowledged with `gmail_sync_superseded`. Neither logs `gmail_sync_completed` or returns success.
- [ ] A superseded or disconnected attempt cannot insert an email, advance the checkpoint, or log or return success. Success requires exactly one owned checkpoint row update.
- [ ] Settings and checkpoint are read from the locked row inside the acquisition transaction (code review). A lookback change made after the request is used by the attempt (test; today's code also passes it, so the review is the real check).
- [ ] Every new transaction sets lock and statement timeouts and makes no Google or queue call (code review, noted in the execution report). A held row lock makes the attempt fail within the bound, not hang.
- [ ] Pending recovery offers at most 100 rows and stops when ownership is lost or the deadline passes.
- [ ] Every handled delivery logs exactly one terminal event; every acquired attempt logs one `gmail_sync_started` first. Fields and categories match §7.3 item 5. `trigger` is `manual` by default and `scheduled` when requested. No fixture subject, sender, token, history ID or page token appears in any line.
- [ ] A queue-send failure returns 503 `QUEUE_UNAVAILABLE`, leaves the connection FAILED with `syncError` `QUEUE_UNAVAILABLE`, and logs one `gmail_sync_failed`.
- [ ] `gmailSync.test.ts` calls the real `googleAuthFailure` (401, 403, 429, `invalid_grant`, sanitized 401, plain error). The copied condition and its unused `eslint-disable` lines (:89, :94, :110, :115) are gone.
- [ ] FU-1.6 holds: the eight existing `gmail-ingestion.test.ts` tests pass with no weakened assertion; repeat syncs keep one row per Gmail ID and re-enqueue no completed email.
- [ ] No `src/contracts` change. No migration, or the §7.6 measured-failure case is recorded with its preservation plan and the migration lanes pass.
- [ ] Busy, superseded and deadline outcomes raised inside the Gmail callback log their own category, not `UNCLASSIFIED`.
- [ ] Backend and frontend suites, typecheck, lint (0 errors) and builds pass in CI; the smoke passes. Docs updated (§7.11).

### 7.10 Testing

New `src/tests/gmail-sync-fencing.test.ts` (DB-backed; mock `googleapis` and the enqueue function as `gmail-ingestion.test.ts:15-36` does; call the exported job handler directly with fake job metadata):

- Same-request busy; obsolete request (newer request, and disconnect through the route).
- Pause, replace, resume across an email insert: the fake message fetch pauses, the test replaces ownership, then resumes.
- Lock proof: the test holds the connection row lock on its own connection, starts a sync, proves the insert waits with `pg_stat_activity`/`pg_blocking_pids` (bounded poll, no fixed sleep), changes the claim, commits; the insert must abort as superseded. A second case holds the lock past `lock_timeout`; the attempt must fail, not hang.
- Zero-row finalization; deadline (stub `Date.now` past 4 minutes); later-page failure (checkpoint unchanged, `checkpointCommitted: false`, kept rows reused by the retry).
- Pending recovery with 101 PENDING rows; ownership lost during recovery.
- Duplicate references across pages; fresh lookback read at acquisition.
- Event checks: started/terminal counts, fields, `trigger` values, forbidden strings, and the category of busy, superseded and deadline outcomes raised inside the Gmail callback.

Changed: `gmailSync.test.ts` (item 4), `gmail-ingestion.test.ts` (quota assertion), `gmail.test.ts` (`POST /sync` queue-send failure → 503).

Crash lane, `scripts/test-gmail-crash.cjs`. Runs after `npm run build` and loads `dist/` like `guarded-migrate.cjs:10`. It loads `TEST_ENV_FILE` (default `.env.crash.test`), requires it to match the exported `TEST_DATABASE_URL`, runs `assertTestDatabase` (`src/utils/testDatabase.ts:1-18`), requires a name ending in `_crash_test`, takes its own advisory lock and refuses a lane with any user or `pgboss.job` row. Steps:

1. Create a fixture user and a CONNECTED connection with fixture-encrypted tokens; call `requestGmailSync`.
2. Fork a child that loads the built backend, replaces `google.gmail` with a deterministic fake (as frontend `scripts/smoke-stabilization.mjs:158` does) and runs `startGmailSyncWorker`. The fake's message fetch tells the parent over IPC, then never resolves.
3. Once the attempt claim is in the row, SIGKILL the child. Age only the fixture job's `started_on` and the fixture connection's `syncLeaseUntil`. Call `boss.supervise('gmail-sync-job')` (public, `index.d.ts:78`); expiry tests `started_on + expire_seconds` (`plans.js:1980-1986`, `failJobsByTimeout`). The retry is re-inserted with the same ID and a backoff `start_after` (`plans.js:2079-2088`); age that `start_after` for the fixture job only.
4. Fork a second child with a normal fake, capturing each child's stdout. Assert §7.9 item 1: `retryCount` 1 (incremented on fetch, `plans.js:1682`), one email row, `syncStatus` IDLE, `syncClaim` null, `lastHistoryId` set; two started lines, one `gmail_sync_completed`, no terminal line for the killed attempt. Then request again and check one row.
5. Stop the worker and confirm both children have exited before deleting fixture rows and jobs. If that cannot be confirmed, delete nothing and exit 1.

Recommendation: add the crash lane as its own step in the S7-01 backend workflow, with a second database in the same service, if it runs in under about 2 minutes. Reason: it is the main proof of this ticket. Otherwise record it as a manual lane.

Commands (backend folder):

```sh
npm ci
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<URL from .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/gmail-sync-fencing.test.ts src/tests/gmailSync.test.ts src/tests/gmail-ingestion.test.ts src/tests/gmail.test.ts src/tests/gmailFetcher.test.ts
npx vitest run src/tests/queue.test.ts src/tests/ai-settings.test.ts src/tests/email-worker-reliability.test.ts
npm test
# crash lane: empty career_companion_<x>_crash_test; .env.crash.test has the .env.test names pointing at it
TEST_ENV_FILE=.env.crash.test TEST_DATABASE_URL='<crash URL>' node scripts/guarded-migrate.cjs
TEST_ENV_FILE=.env.crash.test TEST_DATABASE_URL='<crash URL>' node scripts/test-gmail-crash.cjs   # run twice; the second run proves cleanup
# smoke lane (regression only)
TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL='<smoke URL>' node scripts/guarded-migrate.cjs
```

Frontend folder: `npm run sync-contracts` (expect 0 changed), `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`, then `SMOKE_DATABASE_URL='<empty migrated smoke URL>' node scripts/smoke-stabilization.mjs`. The smoke is a regression check of the real sync worker, not crash evidence. Migration lanes are not needed (no migration).

### 7.11 Documentation updates

- [operations.md](operations.md), after S6-C02's banner: mark as delivered `gmail_sync_started`, the three terminal events with their fields, `checkpointCommitted`/`checkpointAdvanced` (:15), the category list, REQUEST_BUSY and SUPERSEDED semantics (:57-58), and the crash command (:71). Keep as target: `gmail_sync_queued`, `gmail_sync_job_outcome`, queue wait and mode (:7), the counters (:9-14), the categories not emitted yet (§7.3 item 5) and snapshot commands (:69); point the transport bound (:65) to S5-FU-02.
- Backend `STABILIZATION.md:140`: describe request/attempt fencing, busy retry, superseded outcome and success only on one owned checkpoint update (build on S6-C02's and S7-02's versions of that line). `:154`: add `gmail_sync_started`, `gmail_sync_completed` and `gmail_sync_superseded`, and the new `gmail_sync_failed` fields.
- This file: update the Status row and add a dated execution section. Sprint 7 `execution-report.md`: evidence and counts. [Roadmap](../README.md) §1 row for S5-FU-01. If the final `requestGmailSync` signature or `trigger` values differ from §7.3, note it in the S7-05 ticket.

### 7.12 Definition of done

- Every §7.9 box is ticked with linked evidence.
- Tests, typecheck, lint (0 errors) and builds are green in CI (S7-01). The crash lane passed twice on one lane, in CI or recorded as a manual run.
- Behavior verified with the guarded tests and the crash lane. This is fixture evidence only; no live Gmail evidence is claimed.
- Docs in §7.11 updated.
- One focused commit (or a short series) in the backend repo; no unrelated changes.
- Evidence recorded in the Sprint 7 `execution-report.md` and in a dated execution section of this file.
