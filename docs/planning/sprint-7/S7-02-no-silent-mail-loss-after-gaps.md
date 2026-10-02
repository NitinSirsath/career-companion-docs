# S7-02 — Never skip mail when the gap since the last sync is longer than the lookback

| Field | Value |
| --- | --- |
| Status | Implemented locally — remote/live acceptance pending; [evidence](execution-report.md) (2026-10-03) |
| Phase / sprint | Sprint 7 — order 2 of 6 |
| Repository | career-companion-backend (Gmail sync service, one additive migration, Gmail status route and shared contract); career-companion-frontend (synced contract, Gmail page notice and settings copy) |
| Size / priority | M (a migration, a shared-contract change, a UI change and lane re-runs) / stop silent loss of job mail at the start of the golden path |
| Depends on | OD-04 decided by the owner; Sprint 7 entry (Phase 0 gate passed, Sprint 6 closeout done). Not on S7-01; the two can run in parallel. |
| Blocks | S7-05. S5-FU-01 rebases on it (§7). |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-10 (and S56-05, which shows how one missed run opens a gap); [roadmap §6](../README.md#6-owner-decisions) OD-04; [Sprint 7 README](README.md) §2, §5 (interim step MV-12 in [Phase 0](../migration-verification/README.md)), §7 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

After any gap between successful syncs, the next successful sync scans every INBOX message received since the last successful sync, up to 30 days back. When the 30-day cap cuts mail, the app records the dates it did not scan and the Gmail page says so. The first sync keeps the configured lookback.

## 2. Why it exists

- Sync is manual today and the default lookback is 1 day. A user who waits more than a day between syncs loses the mail in between. It is never ingested, and nothing tells them (S56-10).
- That mail is the product's main input: interview invites and rejections in the gap never reach the Action Center.
- The twice-daily schedule (S7-05) does not remove the risk. One failed or missed run gives a gap of about 24 hours or more (S56-05). S7-05 depends on this ticket for that reason ([Sprint 7 README §6](README.md#6-ordered-tickets)).
- The owner's recommended answer is OD-04: scan since the last successful sync, capped at 30 days, and tell the user when the cap cut mail.
- Until this ships, Phase 0 keeps the lookback at 30 days as an interim step (MV-12, [Sprint 7 README §5](README.md#5-dependencies)).

## 3. Current behavior

Line numbers were checked on 2026-10-02 against the backend and frontend folders. Re-check them before editing.

**3.1 Facts (verified in code).**

1. **Window.** `GmailSyncService.syncUser` reads `syncLookbackDays = connection.syncLookbackDays || 1` (`src/services/gmailSync.ts:84`). The default is 1 (`prisma/schema.prisma:148`, `GmailConnection.syncLookbackDays`). `PATCH /api/gmail/settings` allows only 1, 7, 14 or 30 (`src/routes/gmail.ts:338`).
2. **Full-scan trigger.** `shouldFullSync` (`gmailSync.ts:86-93`) is true when there is no `lastHistoryId`, no `lastSyncedAt`, the days since `lastSyncedAt` exceed the lookback, or the lookback was raised above `lastSyncedLookbackDays`.
3. **Full scan.** `fullSync` (`gmailSync.ts:182-204`) takes the history baseline first (:183-186). It then lists `labelIds: ['INBOX']` with `q: newer_than:${syncLookbackDays}d` (:190-199, query at :194).
4. **Age filter.** The inner `ingest` drops any message whose `internalDate` is older than `Date.now() - syncLookbackDays * 86400_000` (`gmailSync.ts:162-164`). It runs on both the full and the history path. `Date.now()` is read again for each message.
5. **History path.** It runs only when `shouldFullSync` is false (`gmailSync.ts:205-236`). A 404 falls back to `fullSync` (:223).
6. **Checkpoint.** On success, the final `updateMany` (`gmailSync.ts:241-252`) sets `lastHistoryId` to the new baseline, `lastSyncedAt` to the time the sync *finished*, and `lastSyncedLookbackDays`. The failure path (:263-273) changes neither `lastSyncedAt` nor `lastHistoryId`. So the gap is measured from the end of the last successful sync.
7. **Result.** Mail received between `lastSyncedAt` and (now − lookback) is never ingested, and the checkpoint moves past it. `GET /api/gmail/status` (`routes/gmail.ts:68-113`) returns `lastSyncedAt` and nothing about skipped mail. The page shows "Last synced …" (`frontend src/routes/gmail.tsx:216`). The settings copy says only "Control how far back to look for emails during sync." (`gmail.tsx:241-243`).
8. **Reconnect.** Disconnect (`routes/gmail.ts:311-323`) clears `lastHistoryId` but keeps `lastSyncedAt`. The callback upsert (:233-256) touches neither. A different mailbox is refused (:218-221). So the first sync after a reconnect is a full scan, and its gap is measured from the old `lastSyncedAt`.
9. **Existing rows are safe.** `ingest` looks up `userId_gmailMessageId` first and skips stored rows; it re-enqueues only `PENDING` ones (`gmailSync.ts:135-143`). The upsert has `update: {}` (:166-176). Uniqueness is `@@unique([userId, gmailMessageId])` (`schema.prisma:112`).
10. **Time budget.** `heartbeat` throws after 4 minutes (`gmailSync.ts:121-123`). The `gmail-sync-job` has `retryLimit: 3`, `retryDelay: 60`, `retryBackoff: true` and `expireInSeconds: 300` (`src/jobs/gmailSyncJob.ts:31-37`). The catch path restores the queued claim (`gmailSync.ts:269`), so a retry can continue. `STABILIZATION.md:140` already says very large first scans may need more than one sync.
11. **AI cost.** Each new INBOX row is enqueued at once (`gmailSync.ts:177-178`). The per-user safety limit defaults to 500 calls per UTC day (`src/services/ai/usage.ts:14`). `reserveUserCall` throws `SAFETY_LIMIT` when it is reached (:63). The email then waits as `PENDING` (`src/jobs/emailProcessingJob.ts:114-128`). Each later sync re-offers at most `REOFFER_LIMIT = 100` waiting rows, newest first, and only while the user's AI access is `READY` (`gmailSync.ts:22`, `reofferPendingEmails` :31-41, check at :32).
12. **Tests.** In `src/tests/gmail-ingestion.test.ts`, the only tests that set `lastSyncedAt` use `new Date()` with lookback 1 (:102-110, :124-132). The mock dates every message now (:63-70). No test uses a stale `lastSyncedAt` or checks the `q` value.
13. **Trigger.** A sync starts only from `POST /api/gmail/sync` (`routes/gmail.ts:373-417`) through `requestGmailSync`. There is no scheduler.

**3.2 Suspected risks (not demonstrated).**

- `lastSyncedAt` is written when a sync ends (fact 6), while the full-scan baseline is taken when it starts. With a gap just under the lookback, mail that arrived during the previous run can fall outside the age filter. This is an edge case noted by the verifier.
- Gmail's exact meaning of `newer_than:Nd` (rolling 24-hour days or calendar days) is unverified. Every test mocks `googleapis` and ignores `q`.
- How long a 30-day scan of a real mailbox takes is unverified. It may need more than one attempt (fact 10).

## 4. Scope

**4.0 Decision.** OD-04 must be decided before work starts. Recommendation (roadmap): scan since the last successful sync, capped at 30 days; tell the user when the cap cut mail. The rest of this ticket assumes that answer.

1. **Window rule.** Add an exported pure helper in `gmailSync.ts`, for example `syncWindow({ now, lastSyncedAt, lookbackDays })`, and two constants: `MAX_SYNC_WINDOW_DAYS = 30` (the largest allowed lookback) and `SYNC_GAP_MARGIN_MS = 60 * 60_000` (1 hour).
   - No `lastSyncedAt` (first sync): `windowDays = lookbackDays`. No gap is recorded.
   - Otherwise: `gapStart = lastSyncedAt − margin`; `requiredDays = ceil((now − gapStart) / 1 day)`; `windowDays = min(30, max(lookbackDays, requiredDays))`.
   - `windowStart = now − windowDays days`.
   - If `gapStart < windowStart`, the cap cut mail: `unscanned = { from: gapStart, until: windowStart }`. Otherwise `unscanned = null`.
   - The margin covers fact 6: one attempt runs at most about 4 minutes plus pending recovery. The window is never narrower than the configured lookback.
2. **One window per sync.** `syncUser` computes the window once, from the connection row it already reads, with one `now`. The Gmail query uses `newer_than:${windowDays}d`. The age filter on both paths uses the fixed `windowStart` instead of a fresh `Date.now()` per message. Keep the `shouldFullSync` trigger as it is.
3. **Record the cut.** Add two nullable columns to `GmailConnection`: `unscannedFrom` and `unscannedUntil`. Write them in the same final `updateMany` that writes `lastSyncedAt` (`gmailSync.ts:242-252`), and only when `unscanned` is not null. A failed attempt writes neither.
4. **Status response.** `GET /api/gmail/status` selects both columns and returns `unscannedGap: { from, until }` as ISO strings, or `null`. The no-connection branch (`routes/gmail.ts:86-94`) returns `null`.
5. **Shared contract.** In `src/contracts/gmail.ts`, add to `GmailStatusResponseSchema`: `unscannedGap: z.object({ from: IsoDateTime, until: IsoDateTime }).nullable().optional()`. Add a local `const IsoDateTime = z.iso.datetime({ offset: true })` to `gmail.ts`, as in `contracts/ai.ts:9` (S7-05 reuses it). Run the frontend `sync-contracts`.
6. **Gmail page notice.** In the Email Sync card, under "Last synced" (`gmail.tsx:216`), show a short warning when `unscannedGap` is set. Suggested text: "Mail received between {from} and {until} was not checked. More than 30 days passed since the last successful sync, and a sync looks back at most 30 days. Check Gmail directly for job emails from those dates." Format dates with `format(…, 'MMM d, yyyy')`, as the email list already does (`gmail.tsx:395`). Use a plain paragraph, not `role="alert"`: it is a standing fact, not an event.
7. **Settings copy.** Replace the text at `gmail.tsx:241-243` with: "How far back the first sync looks, and how far back a sync looks after you raise this setting. After that, each sync covers everything since the last successful sync, up to 30 days."
8. **Log.** Add `windowDays` and `gapCapped` (boolean) to the existing `gmail_sync_completed` line (`gmailSync.ts:253-261`). Nothing else in logs changes.
9. **Owner choices under OD-04** (recommended defaults; the owner may override at review):
   - **Notice lifetime.** Recommended: keep the recorded gap until a newer capped sync replaces it. The fact stays true, and the product cannot recover mail older than 30 days. Clearing it on the next successful sync risks the S7-05 schedule hiding it before the user sees it. No dismiss control in this ticket.
   - **Reconnect.** Recommended: the same rule applies after a disconnect and reconnect (fact 8). The first sync scans since the old `lastSyncedAt`, capped at 30 days, and records a gap if the cap cut mail.

## 5. Out of scope

- The scheduler, start-up catch-up and last/next sync display (S7-05).
- Claim fencing, crash recovery, success-only-with-one-row checkpoint, and sync events (S5-FU-01).
- Bounded Google requests, timeouts and the time budget itself (S5-FU-02).
- Gmail push (Pub/Sub), non-INBOX labels, and per-user schedules ([Sprint 7 README §4](README.md#4-non-goals)).
- Changing the lookback options, the default, `REOFFER_LIMIT` or the AI safety limit.
- Recovering mail older than 30 days, or counting how many messages the cap cut (no extra Gmail call).
- Resetting, re-processing or re-matching rows that are already stored.

## 6. Likely files and components

Backend (`career-companion-backend-main`):
- `src/services/gmailSync.ts` — `syncWindow`, the constants, `syncUser` (query, age filter, final update, log).
- `prisma/schema.prisma` — `GmailConnection.unscannedFrom`, `unscannedUntil`.
- `prisma/migrations/<timestamp after the newest migration at implementation time>_gmail_unscanned_gap/migration.sql` — new (AI-19 adds one in the closeout).
- `src/routes/gmail.ts` — `GET /status`.
- `src/contracts/gmail.ts` — `GmailStatusResponseSchema`.
- Tests: `src/tests/gmailSync.test.ts`, `src/tests/gmail-ingestion.test.ts`, `src/tests/gmail.test.ts`.

Frontend (`career-companion-frontend-main`):
- `src/contracts/gmail.ts` — synced copy, never edited by hand.
- `src/routes/gmail.tsx` — notice and settings copy.
- `src/tests/gmail.test.tsx`.

## 7. Implementation notes

- **Data model and migration.** Two nullable columns, no default, no backfill, no index: `ALTER TABLE "gmail_connections" ADD COLUMN "unscannedFrom" TIMESTAMP(3), ADD COLUMN "unscannedUntil" TIMESTAMP(3);`. Existing rows read as null, so nothing is shown until a capped sync happens. Write the SQL by hand; do not run `prisma migrate dev` or `db:reset` against a database with real data, because they can offer a reset. The folder must sort after the newest migration at implementation time (AI-19 adds one in the closeout, `<timestamp>_ai_access_paused`, after `20261002150000_automation_submissions`). Apply it to test databases only through `scripts/guarded-migrate.cjs`, and to the dev database with `npx prisma migrate deploy`.
- **Migration lanes.** Each lane builds its legacy schema from every migration except its own (for example `verify-mcp-migration.cjs:174-175`). The new migration is therefore applied before each lane's own migration. The Sprint 6 lane already works this way with three later migrations after the closeout (`user_provided_ai`, `automation_submissions` and AI-19's `ai_access_paused`), so the lanes should pass. AI-19's migration alters `AIAccessIssue`, which the AI lane's own migration (`20261002120000_user_provided_ai`) creates, so the AI lane needs AI-19's lane fix first (TEST-03); build on that version. This is unverified until they run. No new lane script: the change is two nullable columns.
- **API and contract.** The change is additive and optional. Older clients ignore the field. The page treats a missing field as null. Keeping it optional also keeps typed fixtures without the field compiling, such as frontend `src/tests/processing-refresh.test.tsx:29-31`. The backend sends `toISOString()` values. Nothing parses the status response with `GmailStatusResponseSchema` today (the frontend uses the type only, `src/api/client.ts:411-412`); the new backend test does. Merge the backend first, then the frontend sync commit (S7-01 merge order).
- **Background jobs.** No change to queue names, job data or job options. After a long gap, more `email-processing` jobs are queued, and more calls run on the user's key. The per-user safety limit still applies (fact 11). Emails over the limit wait as `PENDING`, and each sync re-offers at most 100 of them. A 30-day backlog can take several days and syncs to drain. That is accepted, and the notice is not about it.
- **Failure and recovery.** Every attempt computes the window again from the unchanged `lastSyncedAt`. An attempt stopped by the time budget or an error writes no gap and does not advance `lastSyncedAt`. The pg-boss retry, or the next manual sync, continues. Stored rows are skipped by a database lookup before any Gmail message fetch (fact 9), so each retry gets further. If failures go on for more than 30 days, the cap applies and the gap is recorded by the first success.
- **Compatibility.** First sync: unchanged. History path: unchanged, except that its age cutoff can be wider than the lookback, never narrower. So a message added to INBOX whose date falls in that extra margin is now kept instead of dropped. Full scans after a gap get wider. With the MV-12 interim lookback of 30, behavior is the same as today for gaps up to 30 days, plus the new record when the cap applies.
- **With S5-FU-01.** FU-1 moves the settings and checkpoint read into a locked transaction at acquisition. The window must be computed from that same read. S7-02 lands first. FU-1 moves the window computation along with the read.
- **Rollback.** Revert the code commit. The columns stay nullable, unused and harmless, so no down migration is needed. The frontend works with or without the field.

## 8. Dependencies

- OD-04 decided (§4.0). The §4 item 9 choices are confirmed at review.
- Sprint 7 entry: Phase 0 gate passed and Sprint 6 closeout exit criteria met ([Sprint 7 README §5](README.md#5-dependencies)).
- No code dependency on S7-01. Once S7-01 exists, CI must be green.
- S7-05 waits for this ticket. S5-FU-01 rebases on it (§7).

## 9. Security and privacy

- Same Gmail scope (`gmail.readonly`), same metadata fields (`format: 'metadata'`, Subject, From and Date at `gmailSync.ts:146-156`), INBOX only. The scan reaches at most 30 days back, the same as the largest lookback a user can already choose.
- After a long gap, more email text can reach the user's own AI provider, on their key. This stays within the per-user safety limit and the existing BYO AI consent. AI-19 corrects the consent wording about when email text is sent.
- The new fields are timestamps only. They are returned only to the owner, through `requireAuth` (`routes/gmail.ts:64`) and `findUnique({ where: { userId } })`. No token field is selected. The token-leak test (`gmail.test.ts:203`) must still pass.
- Logs gain only `windowDays` and a boolean. They contain no message IDs, subjects, senders or dates.
- Owner-scoped uniqueness is kept. Existing rows and user decisions are never reset (fact 9).

## 10. Acceptance criteria

- [ ] With lookback 1, `lastHistoryId` set and `lastSyncedAt` 5 days ago, a sync lists with `q: 'newer_than:6d'`. It ingests an INBOX message received 4 days ago and one received 1 hour ago. The same test fails on the current code.
- [ ] With `lastSyncedAt` 45 days ago, the sync lists with `newer_than:30d`. It ingests a message from 20 days ago and drops one from 40 days ago. It stores `unscannedFrom` = old `lastSyncedAt` − 1 hour and `unscannedUntil` = now − 30 days (within the test's tolerance).
- [ ] A first sync (no `lastSyncedAt`) with lookback 7 lists with `newer_than:7d` and records no gap.
- [ ] A gap shorter than the lookback still uses the history path. Its age cutoff is never narrower than the configured lookback.
- [ ] The wider scan does not change or re-enqueue rows already stored (including a `COMPLETED` row with a user match decision). Another user's row with the same Gmail ID is untouched. There is still one row per (user, Gmail ID).
- [ ] A failed sync leaves `lastSyncedAt`, `lastHistoryId` and both gap columns unchanged. The next successful sync still covers the gap.
- [ ] A later uncapped sync keeps a recorded gap. A newer capped sync replaces it.
- [ ] `GET /api/gmail/status` returns `unscannedGap` with ISO strings when recorded, and `null` otherwise, including when there is no connection. The body parses with `GmailStatusResponseSchema` and has no token fields.
- [ ] Backend and frontend `src/contracts` are identical after `sync-contracts`.
- [ ] The Gmail page shows the notice with both dates when `unscannedGap` is set, and no notice when it is null or missing. The new settings copy is shown.
- [ ] With the age filter changed back to the configured lookback, the 5-day regression test fails (§11.3).
- [ ] The migration applies through `guarded-migrate.cjs`. The three migration lanes pass.
- [ ] Full backend and frontend suites, typecheck, lint (0 errors) and build pass. The smoke passes.

## 11. Testing

**11.1 Tests to add or change.**
- `src/tests/gmailSync.test.ts` (no DB queries; `.env.test` still required): `syncWindow` cases for no `lastSyncedAt`; a gap shorter than the lookback; exactly 1 day (gives 2, because of the margin); a lookback larger than the gap; 29 days 22 hours (30, no gap); 45 days (capped, exact `from` and `until`); a `lastSyncedAt` in the future (gives the lookback).
- `src/tests/gmail-ingestion.test.ts` (DB, mocked `googleapis`): the §10 sync cases. Make `mocks.get` return a per-ID `internalDate`. Assert `mocks.list` with `expect.objectContaining({ q: 'newer_than:6d' })`. For the ownership case, seed a second user's row with the same Gmail ID.
- `src/tests/gmail.test.ts`, in `describe('GET /api/gmail/status')` (:169): `unscannedGap` set, null, and no connection; parse with the schema.
- Frontend `src/tests/gmail.test.tsx`: notice shown with both dates; no notice when null or missing; new settings copy.

**11.2 Commands.**

```sh
# backend repo root
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/gmailSync.test.ts src/tests/gmail-ingestion.test.ts src/tests/gmail.test.ts
npm test
# each lane on two fresh, empty career_companion_*test databases
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-mcp-migration.cjs

# frontend repo root
npm run sync-contracts && npm run typecheck && npm run lint && npm test && npm run build
npx vitest run src/tests/gmail.test.tsx

# smoke (empty migrated smoke DB; both repos built)
TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs   # backend
SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs                              # frontend
```

The smoke is a regression check only. Its fake Gmail ignores `q` and dates every message now (`smoke-stabilization.mjs:110`, :119-120), so it is not evidence for the gap fix.

**11.3 Mutation check** (temporary, never committed): in `syncUser`, use `Date.now() - syncLookbackDays * 86400_000` again as the age cutoff. The 5-day regression test must fail. Restore the file and confirm `git diff --exit-code`.

**11.4 Live evidence.** All results above are fixture results. Live evidence comes from daily use: with lookback 1, after more than a day without a sync, mail from the gap appears. Record it in the execution report when seen. Until then, call the evidence fixture-only.

## 12. Documentation updates

- `career-companion-backend-main/STABILIZATION.md:140` and `docs/architecture/mvp-architecture.md:15`: describe the window rule (first sync uses the lookback; later syncs cover since the last successful sync, up to 30 days; a cut is shown on the Gmail page). S6-C02 fixes the stale "90-day" wording in the same lines; build on its version.
- `docs/planning/migration-verification/README.md`, MV-12: a dated one-line note that the interim 30-day lookback is no longer needed to avoid gaps once S7-02 ships. The owner may keep it or lower it.
- `docs/planning/sprint-7/execution-report.md` (create it if no earlier Sprint 7 ticket has): test counts, lane outputs, the mutation result, and the owner's §4 item 9 choices.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend tests, typecheck, lint (0 errors) and build are green on the new PC, and in CI once S7-01 exists.
- [ ] The migration lanes and the smoke pass. The §11.3 mutation was seen to fail and was restored.
- [ ] Behavior is checked on the Gmail page with a seeded capped gap on the dev database. Live gap evidence is recorded only if seen (§11.4).
- [ ] The §12 docs are updated. No other doc is rewritten.
- [ ] One focused commit per repo (backend, frontend, docs).
- [ ] Evidence is recorded in the Sprint 7 execution report. Any defect found outside this scope is recorded there and raised with the owner, not fixed here.
