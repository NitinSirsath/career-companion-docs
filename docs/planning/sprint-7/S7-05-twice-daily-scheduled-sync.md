# S7-05 — Twice-daily scheduled Gmail sync

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 7 — order 6 of 6 (last) |
| Repository | career-companion-backend (main); career-companion-frontend (Gmail page last/next sync, status refetch); shared contracts (`src/contracts/gmail.ts`) |
| Size / priority | M / last in Sprint 7; delivers the locked 2026-10-01 product decision |
| Depends on | S5-FU-01 (FU-1 fencing and minimal sync events); S7-02 (no mail loss after a gap); S7-04 (worker start-up and readiness); OD-05; S7-01 (CI) in place |
| Blocks | The roughly two weeks of real daily use before Sprint 8 ([Sprint 8 README](../sprint-8/README.md) §5) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-09, DOCS-22, AIB-06; OD-05 ([roadmap §6](../README.md#6-owner-decisions)); [Sprint 7 README](README.md) §3, §6, §7; product decision 2026-10-01 ([docs README](../../../README.md), [Sprint 6 README](../sprint-6/README.md) "Product scope decisions"); [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md) §4, §7.3 item 5; [AI-20](../byo-ai/AI-20-recovery-path-and-status-ui.md) §5 (copy handoff) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Gmail sync runs by itself at 00:00 and 18:00 in one configured timezone (default `Asia/Kolkata`) for every user whose Gmail is connected. A user who is already syncing is skipped, not failed. If a slot was missed because the backend was off or the PC was asleep, one catch-up sync runs when the backend is back. Emails waiting for AI resume through the existing re-offer at the end of each sync. The Gmail page shows the last successful sync and the next automatic sync. Manual sync keeps working unchanged.

## 2. Why it exists

- The owner locked a twice-daily scheduled sync on 2026-10-01. It is not built and had no ticket (S56-09, DOCS-22).
- No doc says which timezone "12:00 AM and 6:00 PM" means. pg-boss evaluates schedules in UTC unless told otherwise (DOCS-22, S56-09). OD-05 settles it.
- Emails that wait for AI (cooldown, pause, daily safety limit, operator pause) stay PENDING until the user clicks Sync, saves AI settings or runs a successful check. Nothing resumes them in the background (AIB-06).
- Sprint 7 order puts this last on purpose: a scheduler must not run on a sync that a crash can fake as success (S5-FU-01), on a lookback that drops mail after a gap (S7-02), or on workers that can be silently dead (S7-04) ([Sprint 7 README](README.md) §6).

## 3. Current behavior

Facts (checked in the code on 2026-10-02; backend = `career-companion-backend-main`, frontend = `career-companion-frontend-main`):

- **Manual is the only trigger.** `POST /api/gmail/sync` (`src/routes/gmail.ts:373-417`, `router.post('/sync')`) is the only caller of `requestGmailSync` (:397). It returns 409 `SYNC_IN_PROGRESS` for a live lease (:387-395, :400-401). The frontend starts a sync only from the button (`src/routes/gmail.tsx:103-111`, `syncMutation`). Its status polling (`gmail.tsx:64-68`; `src/components/ProcessingRefreshObserver.tsx:21-25`) never starts one. A grep of backend `src` for schedule/cron/setInterval finds no scheduler; `src/services/gmailSync.ts:28` says "(no scheduler)".
- **Request and claim.** `requestGmailSync(userId)` (`src/jobs/gmailSyncJob.ts:11-53`) claims the row only when `status = 'CONNECTED'` and the row is not SYNCING or its lease expired or is null (:13-24). It stores `syncClaim = queued:<uuid>` and a 5-minute lease (`syncLease`, `gmailSync.ts:43-44`). Zero rows → `SyncInProgressError` (:25), also when the user is not CONNECTED. It sends `gmail-sync-job` with `{ userId, claim }`, `singletonKey: claim`, `retryLimit: 3`, `retryDelay: 60`, `retryBackoff: true`, `expireInSeconds: 300` (:28-38). A send failure sets FAILED with `syncError = 'QUEUE_UNAVAILABLE'` (:41-51).
- **Worker.** `startGmailSyncWorker` (`gmailSyncJob.ts:55-73`) logs `gmail_sync_failed` with `jobId` and the error class only (:62-68) and swallows `GmailAuthError` and `SyncInProgressError` (:69). That swallow is the G1 false-success path S5-FU-01 FU-1 fixes. No log or job field says what started a sync.
- **Sync end.** `GmailSyncService.syncUser` (`gmailSync.ts:79-277`) re-offers waiting mail at the end of every successful scan (:240), then sets `lastSyncedAt` and the checkpoint only on success (:241-252), then logs `gmail_sync_completed` (:253-261). So `lastSyncedAt` means "last successful sync". Among other conditions, a full scan runs when `daysSinceLastSync > syncLookbackDays` (default 1) (:84-93, `shouldFullSync`); slots 6 h and 18 h apart stay on the history path in steady state.
- **Re-offer.** `reofferPendingEmails` (`gmailSync.ts:24-41`) does nothing unless access is READY (:32) and offers at most 100 PENDING emails, newest first (`REOFFER_LIMIT`, :22, :33-38). Other callers: `src/services/ai/settings.ts:235` (`saveSettings`) and :280 (`checkSettings`, after a verified check). Enqueueing is idempotent per email (`singletonKey`, `src/jobs/emailProcessingJob.ts:19`).
- **Why mail waits.** Access is LIMITED/PAUSED when the limit is 0 (`src/services/ai/access.ts:93`), during a cooldown (:98-103), or after the daily safety limit until the next UTC midnight (:104, `deriveAccess`). The email worker then sets the email back to PENDING and cancels the job (`emailProcessingJob.ts:114-129`, `withdrawDelivery` at :79) (AIB-06).
- **Revoked Gmail.** A 401 or `invalid_grant` inside `withGmail` sets `status = 'REVOKED'` (`src/services/gmailClient.ts:33-38`). Disconnect sets `NOT_CONNECTED` (`routes/gmail.ts:314`).
- **Queue.** `getQueue` (`src/services/queue.ts:5-23`) creates `new PgBoss({ connectionString, schema: 'pgboss' })` with default options and creates three queues (:12). It never calls `schedule()`. pg-boss turns scheduling on by default (`node_modules/pg-boss/dist/attorney.js:382`).
- **Start-up.** `src/index.ts:103-113` starts the three workers outside `NODE_ENV=test` with fire-and-forget catches (the S7-04 target). Shutdown stops the queue (:119-131).
- **Status API.** `GET /api/gmail/status` (`routes/gmail.ts:68-113`) returns `connected`, `gmailEmail`, `status`, `syncStatus` (an expired SYNCING lease reads as FAILED), `syncError`, `lastSyncedAt`, `syncLookbackDays`. The no-connection branch is :86-95. Contract: `GmailStatusResponseSchema` (`src/contracts/gmail.ts:22-30`); `lastSyncedAt` is still the loose `z.date().nullable().or(z.string().nullable())` (:28). Newer contracts use `z.iso.datetime({ offset: true })` (for example `src/contracts/ai.ts:9`). The frontend reads status without a runtime parse (`src/api/client.ts:411-412`) and shows "Last synced …" (`gmail.tsx:216`).
- **Data model.** `GmailConnection` (`prisma/schema.prisma:135-156`) has no timezone or schedule field. No user has a timezone anywhere (DOCS-22).
- **Installed pg-boss 12.31.0** (`node_modules/pg-boss/package.json:3`):
  - `schedule(name, cron, data, options)` (`dist/index.d.ts:103`); `ScheduleOptions = SendOptions & { tz, key, missed }` (`dist/types.d.ts:584-596`).
  - `tz` falls back to `'UTC'` (`dist/timekeeper.js:677`, `schedule`). The queue must exist first, or it throws "Queue … not found" (:698-702).
  - `missed`: `'skip'` (default) sends nothing for occurrences that came due while no cron pass ran; `'once'` sends one job for the most recent one (`types.d.ts:587-594`; `timekeeper.js:456-466`, `missedOccurrence`). A gap is measured from the last cron pass recorded in the database, so it covers both a stopped backend and a suspended (asleep) process.
  - Schedules are rows in the `pgboss` schema. `schedule()` upserts; only `unschedule()` removes one. Cron passes run every 30 s by default (`attorney.js:611`) and are claimed through the database, so several processes send one job per occurrence.
  - `cron-parser` 5.10.0 is installed as a pg-boss dependency (`package-lock.json:2107-2108`); pg-boss evaluates cron with it (`timekeeper.js:483-484`). Its `prev()` answers strictly before the reference date (comment at `timekeeper.js:479-482`; also checked locally with `Asia/Kolkata`).
  - `createQueue` uses the `standard` policy by default (`dist/manager.js:1439`). On that policy a `singletonKey` without `singletonSeconds` does not stop a duplicate job: the unique indexes (`dist/plans.js:713-755`) cover other policies, or `singleton_on`, which is set only from `singletonSeconds` (:1885-1889).
- **Docs.** The decision is in the docs [README](../../../README.md):7 and the [Sprint 6 README](../sprint-6/README.md):15, with no timezone. [ai-capability-architecture.md](../../architecture/ai-capability-architecture.md):450 says "every manual or scheduled sync" (S6-C02 corrects it first). [email-ai-pipeline.md](../../architecture/email-ai-pipeline.md):78 says "There is no scheduler." [byo-ai/README.md](../byo-ai/README.md):351, :867, :873 wait for this ticket. [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md):87 (§4) says any scheduler should build on FU-1 fencing. S5-FU-01 §7.3 item 5 already adds the `trigger` parameter and field that this ticket uses.

Suspected risks (not demonstrated):

- **Lease while queued.** The 5-minute lease starts at request time, and the sync worker uses pg-boss's default concurrency of one job. With several users, a queued user's lease can expire while it waits behind another user's sync (up to 4 minutes, `gmailSync.ts:122`). The status then reads FAILED until its job starts. Not an issue for one owner; FU-1 changes the claim model anyway.
- **Sleep and wake.** That `missed: 'once'` fires after the PC wakes is read from the pg-boss source, not observed. Only the real-day check (§10) proves it.
- **Dev restarts.** `ts-node-dev` restarts on file saves. Each start-up runs the catch-up check (§4 item 6). It requests a sync only for users without a successful sync since the latest slot, but a persistently failing sync would retry on every restart.
- **Google test-mode tokens.** If the OAuth app is in Testing status, Gmail becomes REVOKED after 7 days (PLAT-19), and scheduled sync then skips the user until reconnect.

## 4. Scope

1. **OD-05 — owner decision. Recommendation:** one configured timezone, `Asia/Kolkata`, for 00:00 and 18:00. On start-up, run one catch-up if a slot was missed. No per-user timezones. Record the outcome in the execution report before coding. If the owner picks another zone, only the default in item 2 changes.
2. **Configuration** (read once at start-up; validated in every environment, next to the `TRUST_PROXY_HOPS` check in `src/utils/config.ts:4-10`):
   - `GMAIL_SCHEDULED_SYNC_ENABLED`: `true` or `false`, default `true`. Any other value stops start-up with a clear message. `false` is the off switch for emergencies and special lanes.
   - `GMAIL_SCHEDULED_SYNC_TZ`: an IANA zone name, default `Asia/Kolkata`. An invalid name stops start-up (check with `Intl.DateTimeFormat('en-US', { timeZone })`; pg-boss also rejects it).
   - The cron is a code constant, `0 0,18 * * *`. It is not configurable and not user-editable.
3. **Slot math** in one small module (for example `src/services/gmailSchedule.ts`): `latestSlot(now)` (the last 00:00 or 18:00 at or before `now`) and `nextSlot(now)` (the first one after `now`) in the configured zone. **Recommendation:** add `cron-parser` as a direct dependency pinned exactly to the version pg-boss resolves (5.10.0 today), and use `CronExpressionParser.parse(cron, { tz, currentDate }).prev()` / `.next()`. `prev()` excludes its reference date, so `latestSlot` passes `now + 1 ms` (as pg-boss does, `timekeeper.js:483`); otherwise exactly 18:00 would return 00:00. Reason: the same evaluator pg-boss uses, and the status route needs no pg-boss instance. `npm ls cron-parser` must show one copy.
4. **Schedule registration** (new `src/jobs/gmailScheduledSyncJob.ts`). Add queue `gmail-scheduled-sync-job` to the list in `queue.ts:12`. When enabled, call on every start-up:
   `boss.schedule('gmail-scheduled-sync-job', '0 0,18 * * *', { source: 'schedule' }, { tz, missed: 'once', retryLimit: 2, retryDelay: 60, expireInSeconds: 120 })`.
   Re-calling on each boot is safe: it upserts, and pg-boss bounds `missed` by the row's `created_on`, not `updated_on`. When disabled, call `boss.unschedule('gmail-scheduled-sync-job')` instead. A row left behind would keep sending jobs. Register the fan-out worker only when enabled. Do all of this inside S7-04's start-up routine (suggested there as `startWorkers` in `src/jobs/startWorkers.ts`), so it gets the same retry-then-exit rule, and add the fan-out queue to the queues S7-04's `GET /ready` reports when enabled.
5. **Fan-out handler** `runScheduledGmailSync(source, now = new Date())`:
   - `slot = latestSlot(now)`. List `GmailConnection` rows with `status = 'CONNECTED'` (select `userId` and `lastSyncedAt` only).
   - Skip a user whose `lastSyncedAt >= slot` (`skippedRecent`): they already synced since this slot.
   - Otherwise call `requestGmailSync(userId, 'scheduled')`. `SyncInProgressError` → `skippedBusy`, not a failure. This also covers a user who disconnected between the list and the claim. Any other error → `failed`, logged per user, and the loop continues.
   - Users one at a time; the handler only claims and queues, so it is short.
   - At the end, log one summary (item 9). If `failed > 0`, throw so pg-boss retries the run. A retry is safe because of the skip rules.
   - Count REVOKED connections as `skippedRevoked` for the log only. NOT_CONNECTED rows are ignored.
6. **Start-up catch-up (OD-05).** When enabled, after registration, send one fan-out job `{ source: 'startup' }`. Do not rely on a `singletonKey` for it: on the default queue policy it does not deduplicate (§3, pg-boss facts). The same handler then syncs only users without a successful sync since the latest slot. Together with `missed: 'once'` (item 4) this covers: backend stopped over a slot, PC asleep over a slot (no restart), and a slot whose sync failed before a restart. Duplicate runs are harmless: the second one finds users busy or recent.
7. **Same lease as manual sync.** Scheduled syncs go through `requestGmailSync` and the existing claim and lease, as left by S5-FU-01 FU-1. No second claim or lock. A manual click during a scheduled sync gets the existing 409. A slot during a manual sync is a busy skip. If FU-1 renames the request entry point, use the new one.
8. **Trigger field.** S5-FU-01 §7.3 item 5 adds `requestGmailSync(userId, trigger = 'manual')`, puts `trigger` in the job data, and logs it on `gmail_sync_started` and each terminal event (`gmail_sync_completed`, `gmail_sync_failed`, `gmail_sync_superseded`). A job without `trigger` counts as `manual`. This ticket only passes `'scheduled'` from the fan-out and tests that it reaches those events. The manual route keeps passing nothing.
9. **Fan-out log event** `gmail_scheduled_sync_run`: `source` (`schedule` or `startup`), `slot` (ISO), `timezone`, `lateBySeconds` (now minus slot; a large value marks a catch-up), `connected`, `requested`, `skippedRecent`, `skippedBusy`, `skippedRevoked`, `failed`, `durationMs`. Per-user failure: `gmail_scheduled_sync_request_failed` with `userId` and error class only. Never log `gmailEmail`, subjects, senders, tokens, cursors or history IDs (S5-FU-01 FU-3 item 4).
10. **Waiting emails resume** only through the existing sync-end re-offer (`gmailSync.ts:240`). No per-user resume timer, delayed job or new re-offer path (BYO plan E16 and non-goals, [byo-ai/README.md](../byo-ai/README.md):903, :938). Note: the safety limit resets at UTC midnight (05:30 in `Asia/Kolkata`), after the 00:00 slot. So the 18:00 slot is the first automatic resume for safety-limited mail.
11. **Status contract (minimal).** Add `nextScheduledSyncAt: IsoDateTime.nullable().optional()` to `GmailStatusResponseSchema`, reusing the `IsoDateTime` constant (`z.iso.datetime({ offset: true })`) that S7-02 adds to `gmail.ts` next to its `unscannedGap` field, and the same optional style. The backend always sends the field. Value: `nextSlot(now)` when the connection is CONNECTED, the schedule is enabled, and this process registered it (an in-memory flag set by the registration in item 4, the same in-memory style as S7-04's `/ready`; no database query); otherwise `null`. Return `null` in the no-connection branch too. "Last sync" stays the existing `lastSyncedAt`. Run `npm run sync-contracts` in the frontend.
12. **Gmail page.** Under "Last synced …" (`gmail.tsx:216`) show "Next automatic sync: <time>" in the same format (`'MMM d, yyyy h:mm a'`, browser local time), or "Automatic sync is off." when connected and the value is `null`. No other UI change.
13. **Status refresh after a slot.** In `ProcessingRefreshObserver` (mounted app-wide), extend the `['gmailStatus']` `refetchInterval`: 2 s while SYNCING (unchanged); otherwise, when `nextScheduledSyncAt` is set, refetch about 60 s after it. The existing SYNCING poll and the new-`lastSyncedAt` refresh window (`ProcessingRefreshObserver.tsx:27-34`) then show the result. Window-focus refetch (TanStack default) covers sleeping tabs.
14. **"Sync to continue" copy (AI-20 handoff). Recommendation: keep AI-20's wording unchanged.** It stays true (a manual sync resumes waiting mail now), and the new line in item 12 tells the user it will also continue on its own. Record the review in the execution report.
15. **Real-use proof.** Run the backend on the owner's PC for two real days with real Gmail. Record each slot as on time or caught up, with at least one catch-up (backend stopped or PC asleep across a slot). On a personal PC the schedule runs only while the backend runs. This is stated in the docs (§12), not hidden.

## 5. Out of scope

- Gmail push (Pub/Sub), per-user schedules or timezones, a user-editable schedule, and notifying the user of sync results ([Sprint 7 README](README.md) §4).
- Any change to sync fencing, claims, Google request bounds or counters (S5-FU-01, S5-FU-02).
- The lookback-gap rule (S7-02) and worker start-up retry and readiness (S7-04). This ticket uses both.
- Recovering stuck PROCESSING rows ([S8-02](../sprint-8/S8-02-recover-stuck-and-waiting-emails.md)).
- Changing the 100-email re-offer cap (`REOFFER_LIMIT`). A larger PENDING backlog still needs several syncs (AIB-06 correction 2).
- Tightening the old `lastSyncedAt` contract type. No ticket owns it; S6-R08 / OD-12 cover only the application date fields.
- Deployment, process managers or keeping the backend running when the PC is off (OD-08).
- A Prisma migration or new database column.

## 6. Likely files and components

- backend: `src/jobs/gmailScheduledSyncJob.ts` (new), `src/services/gmailSchedule.ts` (new), `src/jobs/gmailSyncJob.ts` (`requestGmailSync` with `trigger`, as left by S5-FU-01; no change expected), `src/services/queue.ts`, the start-up routine and `/ready` from S7-04 (suggested `src/jobs/startWorkers.ts`, `src/routes/health.ts`), `src/routes/gmail.ts`, `src/contracts/gmail.ts`, `src/utils/config.ts`, `package.json` and `package-lock.json` (`cron-parser`), `.env.example`, `README.md`, `STABILIZATION.md`.
- backend tests: `src/tests/gmail-scheduled-sync.test.ts` (new), `src/tests/gmail-schedule-registration.test.ts` (new, real pg-boss), `src/tests/gmail-schedule.test.ts` (new, unit), `src/tests/gmail.test.ts`, `src/tests/gmail-ingestion.test.ts`.
- backend test env (2026-10-02): S6-C01's pinned test-env helper (suggested there as `src/tests/helpers/testEnv.ts`, used by `src/tests/setup.ts`) gains `GMAIL_SCHEDULED_SYNC_ENABLED` = `true` and `GMAIL_SCHEDULED_SYNC_TZ` = `Asia/Kolkata`, both the code defaults (S6-C01 §4 item 3 and its §7 rule for new variables). Pin `true`, not `''`: item 2 stops start-up on any other value. Its isolation test, `src/tests/test-env-isolation.test.ts`, gets one more case (§11).
- frontend: `src/contracts/gmail.ts` (synced only, never hand-edited), `src/routes/gmail.tsx`, `src/components/ProcessingRefreshObserver.tsx`, `src/tests/gmail.test.tsx`, `src/tests/processing-refresh.test.tsx`.
- docs: see §12.

## 7. Implementation notes

- **Data model / migrations:** none in Prisma. pg-boss stores the schedule and the new queue in its own `pgboss` schema at runtime. The three migration lanes are unaffected.
- **API and shared contracts:** one new optional, nullable field on `GET /api/gmail/status`, beside S7-02's `unscannedGap`. Merge backend first, then the frontend sync commit (S7-01 `contract-drift` can be red in between). An older frontend ignores the extra field; the new page treats a missing field like `null`.
- **Background jobs:** one new queue and one schedule row. The fan-out job only claims and queues, then the existing `gmail-sync-job` worker does the work. Scheduled and manual syncs share retries, lease and the 4-minute budget.
- **Tests and the smoke:** registration happens only in the non-test start-up path. The smoke sets `NODE_ENV=test` and registers workers itself (frontend `scripts/smoke-stabilization.mjs:47`, :1009-1010), so it never registers the schedule. A test that registers it must `unschedule` in `afterAll`.
- **Failure and recovery:** an invalid config stops start-up. A failed `schedule()` call follows the S7-04 rule (retry, then exit non-zero) and leaves readiness not ready. A failed run retries twice. A failed user sync shows FAILED on the page as today, and the next slot or start-up retries it.
- **Compatibility:** manual sync, its 202 and 409 responses and the frontend error handling are unchanged. Old queued sync jobs without `trigger` log `manual` (S5-FU-01 rule).
- **Rollback:** set `GMAIL_SCHEDULED_SYNC_ENABLED=false` and restart once. That removes the schedule row. Only then revert the code. If the code is reverted first, delete the row by hand (`SELECT * FROM pgboss.schedule` to find it); otherwise pg-boss keeps sending jobs to a queue with no worker.

## 8. Dependencies

- Phase 0 gate passed and Sprint 6 closeout done ([migration verification](../migration-verification/README.md)).
- OD-05 recorded ([roadmap §6](../README.md#6-owner-decisions)).
- [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md) FU-1 and minimal events done: an unattended run must not be acknowledged as success after a crash.
- [S7-02](S7-02-no-silent-mail-loss-after-gaps.md) done: a catch-up after days off must not drop mail.
- [S7-04](S7-04-worker-startup-and-readiness.md) done: the schedule registers in the same start-up path and shows in readiness.
- [S7-01](S7-01-ci-and-toolchain-pins.md) CI running, so the contract change is guarded.
- [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) has already removed the false "scheduled sync" wording; §12 states the new truth.
- Owner: a real Gmail account connected on the PC, for §4 item 15.

## 9. Security and privacy

- Scheduled sync uses the same `gmail.readonly` grant and code path as a manual click. No new scope, endpoint or credential.
- Only CONNECTED users are synced. Disconnect or revoke stops scheduled sync at once: both the fan-out filter and the claim's `status = 'CONNECTED'` condition (`gmailSyncJob.ts:16`) refuse.
- Each sync can queue new mail for AI on the user's own key. The per-user safety limit (`AI_USER_DAILY_CALL_LIMIT`, default 500) and the 100-email re-offer cap still apply. The data-use text describes what is sent per new email and does not tie it to a click (`src/contracts/aiCatalog.ts:63-64`, `AI_DATA_SENT_SUMMARY`; AI-19 rewords it but keeps it per email), so no consent change is needed here.
- Logs carry user IDs and counts only (§4 item 9).
- The schedule is set by the operator through env, not by users. The status field reveals only the next slot time.

## 10. Acceptance criteria

- [ ] OD-05 outcome recorded in the execution report.
- [ ] Enabled: start-up leaves exactly one pg-boss schedule for `gmail-scheduled-sync-job` with cron `0 0,18 * * *`, timezone `Asia/Kolkata` (or the configured zone) and `missed: 'once'` (`getSchedule` shows it).
- [ ] Disabled: start-up removes the schedule, sends no start-up job, and status returns `nextScheduledSyncAt: null`; the page shows "Automatic sync is off."
- [ ] An invalid `GMAIL_SCHEDULED_SYNC_ENABLED` or `GMAIL_SCHEDULED_SYNC_TZ` stops start-up with a clear message.
- [ ] Fan-out: each CONNECTED user with `lastSyncedAt` before the slot gets exactly one `gmail-sync-job` with `trigger: 'scheduled'`. Users synced since the slot, busy users (live lease) and REVOKED or NOT_CONNECTED users get none, and busy users' rows are unchanged.
- [ ] One user's queue failure does not stop the others, and the run then fails so pg-boss retries it.
- [ ] Two runs for the same slot, or a run during a manual sync, give at most one sync per user. A manual click during a scheduled sync gets 409 `SYNC_IN_PROGRESS`.
- [ ] Start-up catch-up requests a sync only for users without a successful sync since the latest slot.
- [ ] Slot math is correct at the edges in `Asia/Kolkata`: 17:59:59 → next 18:00; exactly 18:00 → latest 18:00; 23:59:59 → next 00:00 of the next day; 18:30 UTC → 00:00 local.
- [ ] A scheduled sync re-offers waiting PENDING emails when access is READY, and none when it is not.
- [ ] A scheduled sync's `gmail_sync_started` and terminal event carry `trigger: 'scheduled'`; a manual sync's carry `manual`. `gmail_scheduled_sync_run` carries the item 9 fields. No log line contains an email address, subject, token or history ID.
- [ ] `GET /api/gmail/status` returns `nextScheduledSyncAt` as an ISO string with offset, or `null`, and parses with the shared schema. Frontend `npm run sync-contracts` copies `gmail.ts`; a second run reports 0 changed.
- [ ] The Gmail page shows "Last synced" and "Next automatic sync"; the status refetches about 60 s after the slot.
- [ ] **Observed on two real days of use** on the owner's PC with real Gmail: log lines for every slot (on time or caught up, with `lateBySeconds`), at least one catch-up, and page screenshots. Waiting-email resume is recorded if a wait happened, or marked "not observed". Nothing synthetic is called live evidence.
- [ ] Docs updated per §12.

## 11. Testing

Tests to add or change:

- `src/tests/gmail-schedule.test.ts` (new, unit): `latestSlot` and `nextSlot` edge cases from §10; config validation for both variables.
- `src/tests/gmail-scheduled-sync.test.ts` (new, DB): call `runScheduledGmailSync` with a fixed `now`. Mock `../services/queue` with `vi.mock` so `getQueue()` returns an object whose `send` is a spy, in the same style as `gmail-ingestion.test.ts:15` mocks the enqueue. Cover: connected and stale → one request with `trigger: 'scheduled'`; recent → skip; live lease → busy skip with the row unchanged; REVOKED and NOT_CONNECTED → none; a send failure for one user → others proceed, the run throws; two runs → one request; the start-up source. Spy on `console.log` for the summary fields and the absence of `gmailEmail`.
- `src/tests/gmail-schedule-registration.test.ts` (new, DB, starts a real pg-boss; a separate file because the file above mocks the queue module): registration creates the row with zone, cron and `missed: 'once'`; the disabled path removes it; `afterAll` unschedules and stops the queue.
- `src/tests/gmail-ingestion.test.ts`: a scheduled-trigger sync re-offers PENDING emails when READY and not when LIMITED.
- `src/tests/gmail.test.ts`: status includes `nextScheduledSyncAt` when connected and active; `null` when disabled, not connected or with no connection; the response parses with `GmailStatusResponseSchema`.
- `src/tests/test-env-isolation.test.ts` (S6-C01) and its pinned list (2026-10-02): add both variables with their defaults (§6). New case: the temporary dev-style env file sets `GMAIL_SCHEDULED_SYNC_ENABLED=false` and `GMAIL_SCHEDULED_SYNC_TZ=America/New_York`; after the dynamic app import both keep `true` and `Asia/Kolkata`, and the config check does not stop the import.
- Frontend `src/tests/gmail.test.tsx`: the "Next automatic sync" line, the "off" line, nothing when not connected. `src/tests/processing-refresh.test.tsx`: with fake timers, the status refetches after the slot. Existing status fixtures need no change (the field is optional).

Commands, backend:

```bash
npm ci
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/gmail-schedule.test.ts src/tests/gmail-scheduled-sync.test.ts src/tests/gmail-schedule-registration.test.ts src/tests/gmail.test.ts src/tests/gmail-ingestion.test.ts src/tests/queue.test.ts
npm test
npm ls cron-parser            # one copy, exact version
```

Frontend:

```bash
npm run sync-contracts        # expect src/contracts/gmail.ts changed; run again: 0 changed
npm run typecheck && npm run lint && npm test && npm run build
SMOKE_DATABASE_URL=<empty migrated smoke DB> node scripts/smoke-stabilization.mjs   # browser-visible change: expect PASS, residue 0
```

Manual: §4 item 15 on the owner's PC. Look for `gmail_scheduled_sync_run` and the per-sync events in the backend output.

## 12. Documentation updates

Line numbers are from 2026-10-02; S6-C02 edits some of these first, so re-find them with `grep -rn -i "schedul"`.

- Backend `.env.example`: both variables with comments (default, off switch, "the schedule runs only while the backend runs").
- Backend `README.md`: a short "Scheduled Gmail sync" paragraph: times, zone, catch-up, off switch, logs, and that a personal PC runs it only while the backend is up.
- Backend `STABILIZATION.md`: Configuration table (:112) rows; Recovery rules, Gmail bullet (:140), for the shared lease and busy skip; "Privacy and diagnostics" (:143-154) for the new events; the rollback order from §7.
- [email-ai-pipeline.md](../../architecture/email-ai-pipeline.md):78 and [ai-capability-architecture.md](../../architecture/ai-capability-architecture.md) §9.6 (:450-451): resume happens at manual and scheduled syncs.
- [byo-ai/README.md](../byo-ai/README.md):351, :867, :873: dated notes that S7-05 delivered the scheduled sync.
- [user-flows.md](../../product/user-flows.md) UF-04 (:154): the mechanism is the twice-daily schedule plus manual sync.
- [MVP-READINESS-REPORT.md](../../../MVP-READINESS-REPORT.md):3 and the docs [README](../../../README.md):7: scheduled sync implemented, zone recorded.
- [Roadmap](../README.md) §1 row "Twice-daily scheduled sync"; the S7-05 box in the [Sprint 7 README](README.md) §7, with evidence links.
- Sprint 7 `execution-report.md`: OD-05 outcome, the copy decision (§4 item 14), test counts, and the two-day log of slots.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend tests, typecheck, lint and build are green, in CI (S7-01); the smoke passes with residue 0.
- [ ] Behavior verified on two real days on the owner's PC, including one catch-up.
- [ ] Docs updated per §12.
- [ ] Focused commits: backend (config and slot math; schedule and fan-out; status contract; docs), then the frontend contract sync and page.
- [ ] Evidence recorded in the Sprint 7 `execution-report.md`. Nothing synthetic is called live evidence.
