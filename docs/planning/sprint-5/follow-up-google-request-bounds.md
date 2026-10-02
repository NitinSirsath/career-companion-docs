# S5-FU-02 — Bound every Google and Gmail request

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 8 — order 4 of 4 |
| Repository | career-companion-backend (main). career-companion-frontend: no code change (contract check only). Docs repo: doc updates. |
| Size / priority | M / an unattended sync or email job must fail within a known time, never hang on Google |
| Depends on | S5-FU-01 (Sprint 7; reshapes `syncUser`); Sprint 8 entry conditions; S7-01 CI; the expiry decision in §4.0 (OD-15, owner, at ticket review) |
| Blocks | — |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-06 (verifier's corrected statement); split out of [S5-FU-01](./follow-up-gmail-reliability.md) §3 FU-2 and FU-4.3 and §5 acceptance item 3; [architecture review](architecture-review.md) G3; [Sprint 8 README](../sprint-8/README.md) §2, §3, §7 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Every request the backend sends to Google ends within a known bound, and the Google SDKs retry nothing. pg-boss stays the only retry policy. This covers Gmail data calls, the OAuth token refresh inside `withGmail`, and the token calls in the Gmail and sign-in routes. One sync attempt has one absolute deadline, and both workers stop their Gmail work when pg-boss cancels the job. A quota 403 never revokes the connection and never forces a token refresh. Loopback HTTP tests against the installed Google libraries prove it.

## 2. Why it exists

- **S56-06** (partly confirmed, low). The OAuth refresh, `getToken`, `getProfile` and `revokeToken` have no time limit. A hung Google token endpoint leaves a sync or email job waiting forever. Verifier loopback run: a call with a 1 s timeout was still pending after 12 s.
- **SDK retries are on.** One Gmail data call can take about 62 s (4 × 15 s plus backoff), not 15 s. That can push an attempt past its 4-minute budget and the 5-minute pg-boss expiry.
- **A quota 403 forces a refresh.** Credentials are stored without an expiry, so the library refreshes on every 403 as well as on 401.
- **Workers ignore cancellation.** At expiry pg-boss fails and retries the job while the old handler keeps running.
- **Disconnect can hang.** A hung revoke blocks clearing the stored tokens, and the UI gives up at 15 s.
- **Unattended runs.** After S7-05, sync runs twice a day with nobody watching. A hang must end as a normal failure that pg-boss retries.
- The roadmap moved S5-FU-01 FU-2 and FU-4.3 here; S5-FU-01 keeps fencing in Sprint 7.

Damage today is limited. Claim and lease fencing stops two attempts from owning one sync. A hang leaves a leaked promise and SYNCING for up to 5 minutes, not corrupt data (S56-06). That is why this comes after fencing, in Sprint 8.

## 3. Current behavior

Checked on 2026-10-02 against `career-companion-backend-main` and its installed `node_modules`: googleapis 180.0.0, googleapis-common 9.0.4, google-auth-library 11.0.2, gaxios 7.3.1, pg-boss 12.31.0. S5-FU-01 will move most `gmailSync.ts` lines; re-find them by symbol before editing.

**3.1 Facts (verified in code).**

1. **Gmail data calls pass only `{ timeout: 15_000 }`:** `src/services/gmailSync.ts:154` (`messages.get` in `ingest`), `:184` (`getProfile` in `fullSync`), `:198` (`messages.list`), `:220` (`history.list`); `src/services/gmailFetcher.ts:60` (`fetchMessageMetadata`), `:69` (`fetchMessageBody`). Outside `src/tests/`, nothing sets `retryConfig`, `retry: false`, `transporterOptions`, `google.options`, `expiry_date` or `job.signal` (grep).
2. **SDK retries default on.** `googleapis-common/build/src/apirequest.js:263-264` sets `options.retry = true`. gaxios retries 408, 429 and 5xx up to 3 times (`gaxios/build/cjs/src/retry.js:23-24`, `:50-64`; `shouldRetryRequest` `:93-134`). It does not retry its own abort (`:95-98`). Verifier loopback, 1 s timeout: hang → 1 hit, about 1.0 s; 503 → 4 hits; 429 → 4 hits; slow 503 → 2 hits. gaxios merges a caller's `signal` with the timeout only while that signal is not yet aborted; an already-aborted signal is replaced by the timeout alone (`gaxios/build/cjs/src/gaxios.js:431-440`, `#appendTimeoutToSignal`). So a request started after cancellation still runs until its timeout.
3. **OAuth clients have no transport bound.** All three use the positional constructor: `src/services/gmailClient.ts:21-25` (`withGmail`), `src/routes/gmail.ts:51` (`createOAuth2Client`), `src/routes/auth.ts:43` (`createOAuth2Client`, `OAuth2Client` from google-auth-library). The positional form passes `{}` to the base class (`google-auth-library/build/src/auth/oauth2client.js:75`), so the transporter is `new Gaxios(undefined)` (`authclient.js:68`). The refresh (`refreshTokenNoCache`, `oauth2client.js:238-275`), code exchange (`getTokenAsync`, `:180-212`), revoke (`revokeToken`, `:411-425`) and ID-token cert fetch (`getFederatedSignonCertsAsync`, `:577`) spread `RETRY_CONFIG` (`authclient.js:276-283`: `retry: true`, POST included) and set no timeout. The per-call `{ timeout: 15_000 }` never reaches the refresh.
4. **No expiry is stored.** `withGmail` sets only `access_token` and `refresh_token` (`gmailClient.ts:27-30`). `GmailConnection` has no expiry column (`prisma/schema.prisma:135-156`). The callback upsert drops `tokens.expiry_date` (`routes/gmail.ts:233-256`), although `getTokenAsync` computes it (`oauth2client.js:208`).
5. **When the library refreshes.** Without `expiry_date`, `requestAsync` refreshes once on a 401 **or a 403**, then replays the request (`oauth2client.js:490-507`; `isAuthErr` at `:500`, refresh at `:505`). With `expiry_date`, it refreshes before the request when the token expires within 5 minutes (`isTokenExpiring`, `:812-817`; `DEFAULT_EAGER_REFRESH_THRESHOLD_MILLIS`, `authclient.js:31`), and never on 401/403, because `forceRefreshOnFailure` defaults to false (`authclient.js:49`, `:76`). Verifier loopback with a hung token endpoint: a 401 call still pending at 12 s, a 403 call at 8 s.
6. **Auth versus quota is already right.** `googleAuthFailure` is status 401 or `invalid_grant` (`gmailClient.ts:10-13`). `withGmail` marks REVOKED by compare-and-swap on the old `accessToken` (`:34-38`), replaces every error with `'Gmail request failed'` plus a status (`:39-40`), and saves refreshed tokens by compare-and-swap on `status: 'CONNECTED'` and the old `accessToken` (`:41-52`). A 403 never revokes (`src/tests/gmail-ingestion.test.ts:166-172`).
7. **Request-path calls with no bound:** `getToken` (`routes/gmail.ts:202`), `getProfile` with no options at all (`:211`), `revokeToken` (`:296`); sign-in `getToken` (`routes/auth.ts:107`) and `verifyIdToken` (`:114-117`). Disconnect clears tokens only after the revoke returns or throws (`routes/gmail.ts:293-305`, then `:311-323`). The frontend gives up after 15 s (`career-companion-frontend-main/src/api/client.ts:47`, `REQUEST_TIMEOUT_MS`).
8. **Cooperative budget only.** The 4-minute budget is checked inside `heartbeat` (`gmailSync.ts:121-123`), which runs before each message and page. No Gmail call receives a deadline.
9. **Workers ignore `job.signal`:** `startGmailSyncWorker` (`src/jobs/gmailSyncJob.ts:55-73`) and `processEmailJob` (`src/jobs/emailProcessingJob.ts:87-176`). pg-boss sets `job.signal` for every delivered job (`node_modules/pg-boss/dist/manager.js:411-412`, `#processJobs`; typed at `dist/types.d.ts:848`). It aborts it when the handler passes `expireInSeconds` (`resolveWithinSeconds`, `manager.js:435`, `dist/tools.js:39-50`) and when a stopping boss fails work in progress (`failWip`, `manager.js:532-539`). Sync jobs: `retryLimit: 3`, `retryDelay: 60`, `retryBackoff: true`, `expireInSeconds: 300` (`gmailSyncJob.ts:33-36`). Email jobs: `expireInSeconds: 300` (`emailProcessingJob.ts:23`).
10. **Email worker path.** `EmailAIPipeline.processEmail` calls `GmailFetcherService.fetchMessageMetadata` (`src/services/ai/pipeline.ts:45`) and `fetchMessageBody` (`:94`), both through `withGmail`. Same refresh gap.
11. **Tests.** Every Gmail test mocks the libraries (`gmail-ingestion.test.ts:16`, `gmailFetcher.test.ts:6`, `gmail.test.ts:34`; `auth.test.ts:9` mocks google-auth-library). No test sends real HTTP through the installed stack. The browser smoke replaces `google.gmail` with an in-process fake (`career-companion-frontend-main/scripts/smoke-stabilization.mjs:158`).
12. **Already mitigating:** claim and lease fencing (`gmailSync.ts:96-117`, `:124-128`), the 5-minute lease (`LEASE_MS`, `:43`), and the status route reporting FAILED after the lease (`routes/gmail.ts:101-105`).

**3.2 Suspected risks (not demonstrated).**

- How a real Google endpoint hangs or rate-limits is unknown. Every number above comes from loopback runs, not live calls.
- With SDK retries off, a short 429 or 503 burst fails the job at once instead of a retry within about 2 s. pg-boss retries after 60 s or more. Under a long burst, email jobs may use all attempts and become FAILED (manual retry exists). Not measured.
- How the *new* settings behave (deep merge of `retryConfig`, signal hand-off to the refresh) is read from code only. The loopback tests in §11 settle it.

## 4. Scope

**4.0 Decision OD-15 (owner, at ticket review).** How to stop the refresh on 403 (FU-2.3).

- **Recommendation: store the access-token expiry.** Add a nullable `GmailConnection.accessTokenExpiresAt` and pass it as `expiry_date`. The library then refreshes before expiry and never on 401 or 403 (fact 5). This is the library's normal mode and needs no custom refresh code. Cost: one additive migration. Side effect: with a known expiry, a 401 revokes without a refresh attempt. Today a 401 is refreshed first and revokes only if the refresh or the replay fails. A 401 on an unexpired token normally means Google revoked access, so the outcome rarely differs. S5-FU-01 §3 allows a migration only when a measured failure shows the existing columns are not enough, and only with a data-preservation plan. The verifier's loopback 403 run (fact 5) is that measurement, and §7 is the plan.
- **Fallback (no migration):** keep the reactive refresh, now bounded. A quota 403 still costs one bounded refresh and one replay. Record FU-2.3's "never force refresh" as not met.

The rest of this ticket assumes the recommendation. Items marked (4.0) drop under the fallback.

1. **One factory for every Google OAuth client.** Add `src/services/googleTransport.ts` with:
   - `GMAIL_REQUEST_TIMEOUT_MS = 15_000` (today's value), `GOOGLE_OAUTH_TIMEOUT_MS = 10_000` (refresh, code exchange, cert fetch), `GOOGLE_REVOKE_TIMEOUT_MS = 5_000` (keeps disconnect inside the frontend's 15 s deadline), `SYNC_ATTEMPT_BUDGET_MS = 240_000` (item 6; kept here so tests can shorten it, §7). These are recommendations; change one only with a reason in the execution report.
   - `createGoogleOAuthClient({ clientId, clientSecret, redirectUri, timeoutMs, signal? })`, using the options-object constructor with `transporterOptions: { timeout: timeoutMs, retryConfig: { retry: 0 }, signal }`. Build it with `google.auth.OAuth2` from `googleapis`. In the installed tree that is the same class as google-auth-library's `OAuth2Client` (checked with `node -e`, 2026-10-02), so the existing `googleapis` mocks and the §7 test seam still apply.
   - `gmailCallOptions(signal?)`, returning `{ timeout: GMAIL_REQUEST_TIMEOUT_MS, retryConfig: { retry: 0 }, signal }`.
   - Use `retryConfig: { retry: 0 }`, not only `retry: false`. Each request carries its own `retry: true` (facts 2 and 3), and gaxios deep-merges request options over the defaults (`gaxios.js:300`), so a default `retry: false` is overridden. `retryConfig.retry: 0` survives the merge and stops `shouldRetryRequest` (`retry.js:100-102`).
   - No `google.options()` global, and no environment variable that changes Google URLs.
2. **Gmail data calls.** Every Gmail call passes `gmailCallOptions(signal)`: the four in `gmailSync.ts`, the two in `gmailFetcher.ts`, and the callback `getProfile` (`routes/gmail.ts:211`). Each call site first calls `signal?.throwIfAborted()`, because gaxios ignores a signal that is already aborted (fact 2). Keep building Gmail clients through `google.gmail(...)` from `googleapis`, so the smoke seam (fact 11) still works.
3. **`withGmail`.** Accept an optional `{ signal?: AbortSignal }`. Build the OAuth client through the factory with `GOOGLE_OAUTH_TIMEOUT_MS` and that signal, so the refresh is bounded and cancellable. (4.0) Pass `expiry_date` when `accessTokenExpiresAt` is set. Keep the 401/`invalid_grant` revoke, the sanitized error and both compare-and-swap writes. (4.0) The token save writes the new expiry in the same `updateMany`.
4. **Gmail routes.** The callback builds its client with `GOOGLE_OAUTH_TIMEOUT_MS`. (4.0) Its upsert stores `tokens.expiry_date` (null when absent) on create and update. Disconnect builds its client with `GOOGLE_REVOKE_TIMEOUT_MS`. A timeout is handled like any revoke failure (logged as `gmail_revocation_failed`), and the tokens are still cleared. (4.0) Disconnect also clears the expiry. Responses, redirects and status codes stay the same.
5. **Sign-in route.** `routes/auth.ts` builds its client through the factory with `GOOGLE_OAUTH_TIMEOUT_MS`; this covers `getToken` and the cert fetch inside `verifyIdToken`. S56-06 does not name it; it is here because S5-FU-01 §5 and the Sprint 8 exit criterion say *every* OAuth request. Recommendation: include it (two lines). If the owner drops it, narrow the Sprint 8 exit wording.
6. **One deadline per sync attempt.** At acquisition, `syncUser` creates one attempt signal: `AbortSignal.any([jobSignal, AbortSignal.timeout(SYNC_ATTEMPT_BUDGET_MS)])` (item 1; today's 4 minutes). The job signal is optional: direct calls such as `gmail-ingestion.test.ts:76` pass none and get the budget alone. It passes that signal to `withGmail` and to every Gmail call. `heartbeat` calls `signal.throwIfAborted()` instead of its own `Date.now()` check. Check it again before pending recovery (`reofferPendingEmails`) and before the final checkpoint. Fit this into the attempt structure S5-FU-01 introduced; do not change its fencing.
7. **Deadline and cancel outcomes.** When the attempt signal fired, the attempt ends through the existing failure path (checkpoint unchanged; claim handling as S5-FU-01 defines). It throws `SyncDeadlineError` (budget; reuse S5-FU-01's deadline error if it added one) or `SyncCancelledError` (job signal). Decide by which signal aborted, not by the sanitized Google error. The worker rethrows both, so pg-boss retries, as it does for the budget error today. `syncError` stays `SYNC_FAILED`.
8. **Sync worker.** `startGmailSyncWorker` passes `job.signal` to `syncUser`.
9. **Email worker.** `processEmailJob` passes `job.signal` to `EmailAIPipeline.processEmail(userId, emailId, { signal })`, which passes it only to the two `GmailFetcherService` calls. AI provider calls get no signal and no other change. Failure recording stays `GmailRequestFailed` (`describeFailure`, `emailProcessingJob.ts:41-57`).
10. **Invariant.** A unit test asserts that `SYNC_ATTEMPT_BUDGET_MS` is at least 30 s below the sync job's `expireInSeconds` × 1000, so an attempt ends itself before pg-boss expiry. Export that expiry as a constant; today it is a literal in `requestGmailSync` (`gmailSyncJob.ts:36`).
11. **Log categories.** No new event, counter or log field. S5-FU-01 §7.3 item 5 gives its terminal event a fixed `category` list and leaves four entries to this ticket. Fill them: `CANCELLED` (`SyncCancelledError`), `REQUEST_TIMEOUT` (a Google call or refresh hit its own bound), `NETWORK_ERROR` (no HTTP response, not a timeout), and the 403 split: a 403 whose Google reason is a rate-limit or quota reason becomes `RATE_LIMIT`, any other 403 stays `FORBIDDEN`. `SyncDeadlineError` maps to the existing `DEADLINE_EXCEEDED`. To make this possible, `withGmail`'s sanitized error keeps `status` and gains one allowlisted field (for example `reason: 'timeout' | 'network' | 'rate_limit'`), never Google text. Unverified: the exact Gmail 403 reason strings (confirm them against Google's Gmail API error documentation) and how to tell a timeout from other no-response errors through gaxios (the S56-06 verifier found that node-fetch's `AbortError` has no `code`). The loopback tests settle the second point.
12. **Tests** per §11, including the FU-4.3 loopback suite. S5-FU-01 §7.3 item 3 moved the cancellation case of FU-4.2 here, with FU-2.2. S5-FU-01 keeps its deadline test for the 4-minute budget; this ticket changes it to the new attempt signal (§7, test seam).

## 5. Out of scope

- Sync telemetry beyond §4 item 11: new events, fields, `gmail_sync_queued`, `gmail_sync_job_outcome` and the FU-3 counter equations ([roadmap §8](../README.md#8-not-now)). The category list itself and the queue-send error belong to S5-FU-01 (Sprint 7).
- Snapshot and baseline tooling (FU-5; OD-13).
- The scheduler (S7-05). Claim fencing, attempt identity and finalization (S5-FU-01).
- AI behavior: provider timeouts, SDK settings, a signal for AI calls, the shutdown deadline (release track item 3; AIB-13, PLAT-18).
- Database statement and lock timeouts (S5-FU-01 FU-1.3).
- pg-boss queue options (retry limit, delay, expiry) and any in-app retry or backoff for 429.
- Frontend and shared-contract changes.
- Google OAuth publishing status and 7-day refresh tokens (PLAT-19; release track item 1).
- Gmail push ([roadmap §8](../README.md#8-not-now)) and per-user schedules (OD-05).

## 6. Likely files and components

Backend (`career-companion-backend-main`):

- `src/services/googleTransport.ts` (new): constants, `createGoogleOAuthClient`, `gmailCallOptions`.
- `src/services/gmailClient.ts` (`withGmail`); `src/services/gmailSync.ts` (`syncUser`, `heartbeat`, new error classes); `src/services/gmailFetcher.ts`.
- `src/services/ai/pipeline.ts` (`processEmail`: signal to the two fetches only).
- `src/jobs/gmailSyncJob.ts` (`startGmailSyncWorker`); `src/jobs/emailProcessingJob.ts` (`processEmailJob`).
- `src/routes/gmail.ts` (`createOAuth2Client`, callback, disconnect); `src/routes/auth.ts` (`createOAuth2Client`).
- (4.0) `prisma/schema.prisma` (`GmailConnection.accessTokenExpiresAt`); `prisma/migrations/<timestamp after the newest>_gmail_access_token_expiry/migration.sql`.
- Tests: `src/tests/gmail-transport.test.ts` (new), `gmail-ingestion.test.ts`, `gmailFetcher.test.ts`, `gmail.test.ts`, `gmailSync.test.ts`, `email-worker-reliability.test.ts`, `auth.test.ts` (its mock moves to the factory's import, §11).

Frontend: none. Docs: §12.

## 7. Implementation notes

- **Data model and migration (4.0).** `ALTER TABLE "gmail_connections" ADD COLUMN "accessTokenExpiresAt" TIMESTAMP(3);` Nullable, no default, no backfill, no index. The folder must sort after the newest migration at implementation time. AI-19 adds one in the closeout (`<timestamp>_ai_access_paused`, after `20261002150000_automation_submissions`), and S7-02 and S7-03 may add theirs first; re-check at merge time. Write the SQL by hand, as S7-02 does. Apply it to test databases only through `scripts/guarded-migrate.cjs`, and to the dev database with `npx prisma migrate deploy`; never run `prisma migrate dev` or `db:reset` against real data. Existing rows read null and keep today's reactive refresh until their first refresh records an expiry. Keep the column out of the status route's `select` (`routes/gmail.ts:74-82`).
- **Migration lanes.** Run all three. As in S7-02, the new migration is applied before each lane's own migration. Expected to pass; unverified until run.
- **API and contracts.** No change. `syncError` is a free string (`src/contracts/gmail.ts:27`), and the frontend does not branch on its values (grep, 2026-10-02). `npm run sync-contracts` must report 0 changed.
- **Background jobs.** No queue option changes. Handlers now use `job.signal`. After pg-boss expiry or `failWip`, an old handler stops at its next Gmail call or heartbeat instead of running on. Expected worst case for one Gmail call: 15 s; with a reactive refresh (null expiry) 15 + 10 + 15 s; never past the attempt deadline.
- **Failure and recovery.** Deadline or cancel: the attempt fails, the checkpoint does not move, and the pg-boss retry or the next sync continues. Stored rows are skipped by a database lookup before any fetch (`gmailSync.ts:135-143`). Transient 429/5xx: no SDK retry; the sync job retries after 60 s with backoff; an email job retries, is FAILED at its last attempt, and can be retried by hand. Hung refresh: a bounded error, not an auth failure, so no revoke. `invalid_grant` from the refresh: REVOKED, as today. Hung revoke: disconnect still answers and clears tokens in about 5 s plus database time.
- **Compatibility.** Start after S5-FU-01 is merged and re-find every `gmailSync.ts` line by symbol. The OAuth2 mocks in `gmailFetcher.test.ts` and `gmail.test.ts` have no `credentials` property and ignore constructor arguments, so keep the optional chaining on `oauth.credentials` (`gmailClient.ts:42-43`). `AbortSignal.any` and `AbortSignal.timeout` need Node 20.3 or later; S7-01 pins 24. The library behaviors this relies on (positional constructor drops options, `retryConfig` deep merge, signal copied by reference) are pinned by the loopback tests: an upgrade that changes them turns those tests red. `package.json:46-47` uses caret ranges, so install with `npm ci`.
- **Test seam.** Add no environment variable or production option that redirects Google URLs. In the loopback test, wrap the real library with `vi.mock('googleapis', async (importOriginal) => …)`: a subclass of `google.auth.OAuth2` that adds `endpoints: { oauth2TokenUrl, oauth2RevokeUrl, oauth2FederatedSignonPemCertsUrl }` on the loopback server (the constructor spreads `options.endpoints`, `oauth2client.js:86-95`), and a `google.gmail` wrapper that adds `rootUrl`. Make bounds short by mocking `googleTransport.ts` with `importOriginal`. Override the constants and also `gmailCallOptions`: it reads its constant inside its own module, so a mocked export alone does not change it. Consumers that pass a constant to `createGoogleOAuthClient` pick up the mocked value. The attempt deadline uses `AbortSignal.timeout`, which stubbing `Date.now` does not fire (S5-FU-01's deadline test stubs `Date.now`); shorten `SYNC_ATTEMPT_BUDGET_MS` through the same mock instead. Whether vitest fake timers drive `AbortSignal.timeout` is unverified; do not rely on it.
- **Rollback.** Revert the code commit. The column stays nullable and unused; no down migration.
- **Commits.** Focused backend commits: factory and Gmail bounds; deadline and worker signals; (4.0) expiry column; tests. Then docs.

## 8. Dependencies

- [S5-FU-01](./follow-up-gmail-reliability.md) done (Sprint 7). It reshapes acquisition, attempts and finalization in `syncUser`; this ticket adds the deadline to that shape.
- [Sprint 8](../sprint-8/README.md) entry conditions (§5). No dependency on S8-01, S8-02 or AI-15; it can run in parallel (§6). S8-02 also edits `emailProcessingJob.ts` (it adds `EMAIL_PROCESSING_STUCK_MS` next to `emailJobOptions`, its §4 item 1); rebase whichever lands second.
- [S7-01](../sprint-7/S7-01-ci-and-toolchain-pins.md) CI green. The new tests use loopback only, so they run in CI.
- The §4.0 decision (OD-15) recorded in the roadmap and this file before coding.

## 9. Security and privacy

- Tokens stay out of logs, errors and job output. `withGmail` keeps sanitizing errors. The new error classes are app-authored, with no `cause` and no Google response attached. A `GaxiosError` carries its request config. gaxios's default redactor masks the `Authorization` header, `grant_type` and secret fields, but not `refresh_token` (`gaxios/build/cjs/src/common.js:234-299`, `defaultErrorRedactor`; key list at :270), so a raw refresh error can still hold the refresh token. pg-boss stores thrown errors in `pgboss.job.output` (the S6-R06 rule, `emailProcessingJob.ts:59-70`).
- Disconnect now always clears stored tokens within the bound. Today a hung revoke can leave them stored.
- (4.0) The expiry is a timestamp, not a secret. No route selects or returns it.
- Loopback tests bind `127.0.0.1` on a random port, use fake tokens and make no real Google call. CI needs no Google secret.
- No scope change (`gmail.readonly`), no new Gmail data read, no URL override in production code.

## 10. Acceptance criteria

- [ ] `grep -rnE "new (google\.auth\.OAuth2|OAuth2Client)\(" src | grep -v "^src/tests/"` finds only `googleTransport.ts`, and every Gmail call site passes `gmailCallOptions(...)`.
- [ ] Loopback: a Gmail data call that gets 503, 500 or 429 hits the server exactly once and rejects without delay.
- [ ] Loopback: a hung Gmail data call rejects at its bound (within 1 s, with test bounds) after one hit.
- [ ] Loopback: a 401 with no stored expiry causes one token request and one replay; the call succeeds; the new token (and, with 4.0, its expiry) is saved by compare-and-swap.
- [ ] Loopback (4.0): with a known, unexpired expiry, a quota 403 hits the data endpoint once and the token endpoint zero times; the connection stays CONNECTED with tokens unchanged.
- [ ] Loopback (4.0): with a known, unexpired expiry, a 401 marks the connection REVOKED after one data hit and zero token hits.
- [ ] Loopback (4.0): with an expiry less than 5 minutes away, the refresh happens before the request (one token hit, one data hit).
- [ ] Loopback: a hung token endpoint ends the call within the OAuth bound after one token hit; the connection is not revoked.
- [ ] Loopback: `invalid_grant` from the token endpoint marks the connection REVOKED.
- [ ] Loopback: aborting the signal mid-request rejects within 1 s after one hit, with no retry. A call started with an already-aborted signal rejects with zero hits.
- [ ] Callback: a hung `getToken` or `getProfile` redirects to `?gmailError=server_error` within its bound. Sign-in: a hung `getToken` redirects with `?error=server_error` (unless §4 item 5 is dropped).
- [ ] Disconnect: with a hung revoke, it still returns 200 `{ disconnected: true }` in under 15 s, and the row is NOT_CONNECTED with empty tokens (and, with 4.0, a null expiry).
- [ ] Sync: when the attempt deadline fires during a hung page, the attempt ends with `SyncDeadlineError` before `expireInSeconds`, no later page is requested, and `lastHistoryId` and `lastSyncedAt` are unchanged.
- [ ] Sync worker: an aborted `job.signal` ends the attempt with `SyncCancelledError` before the next Gmail call.
- [ ] Email worker: both Gmail fetches receive `job.signal`; AI calls are unchanged, and the existing AI tests pass unmodified.
- [ ] The budget-versus-expiry invariant test passes.
- [ ] The terminal sync event logs `CANCELLED`, `DEADLINE_EXCEEDED` and `REQUEST_TIMEOUT` in the matching cases above, `RATE_LIMIT` for a quota 403 and `FORBIDDEN` for another 403. No line contains Google error text or a token.
- [ ] (4.0) The migration applies through `guarded-migrate.cjs` and all three migration lanes pass; the callback stores the expiry and disconnect clears it.
- [ ] Existing Gmail and sign-in suites still pass: `gmail-ingestion` (8 tests on 2026-10-02, including the quota-403 case), `gmailFetcher` (including the 401-revokes case), `gmail` (24), `auth` (10). S5-FU-01 may add tests first; compare with the counts after it lands.

## 11. Testing

**New file `src/tests/gmail-transport.test.ts`.** Database-backed, because `withGmail` reads the connection (and, like every file, it needs `.env.test`). A `node:http` server on `127.0.0.1:0` plays the Gmail API and the token, revoke and cert endpoints. It counts hits per path and can answer, fail with a status, or hang. The real googleapis, google-auth-library and gaxios run against it through the §7 test seam, with test bounds of a few hundred milliseconds. Cases: every loopback criterion in §10, the callback, sign-in and disconnect cases through `supertest` (same setup as `gmail.test.ts`), and the category criterion through the sync job handler S5-FU-01 exports (its §7.5). Close the server and any hung sockets in `afterEach`.

**Changes to existing tests:**

- `gmail-ingestion.test.ts` (or S5-FU-01's `gmail-sync-fencing.test.ts`, wherever its deadline test lives): deadline and cancellation, with mocked `messages.list` / `history.list` that wait for the passed `signal`. Shorten the budget through the §7 test seam. Assert the error class, an unchanged checkpoint and no further page call.
- `gmailFetcher.test.ts`: the signal reaches both fetches; the 401-revokes case stays.
- `gmail.test.ts`: (4.0) the callback stores the expiry; disconnect clears it.
- `gmailSync.test.ts`: the budget-versus-expiry invariant.
- `email-worker-reliability.test.ts`: `processEmail` receives `job.signal`.
- `auth.test.ts`: it mocks `google-auth-library` (`auth.test.ts:9`). Once `routes/auth.ts` builds its client through the factory, the client comes from `googleapis`, whose own import of the library is not inlined by vitest (`vitest.config.ts` sets no `server.deps.inline`). So move the mock to `googleapis` (same shape) or to `../services/googleTransport`. Expected; unverified until run.

**Commands (backend):**

```bash
npm ci
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/gmail-transport.test.ts
npx vitest run src/tests/gmail-ingestion.test.ts src/tests/gmailFetcher.test.ts src/tests/gmail.test.ts src/tests/gmailSync.test.ts src/tests/email-worker-reliability.test.ts src/tests/auth.test.ts
npm test
# (4.0) each lane with two new, empty career_companion_*test databases
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-mcp-migration.cjs
# smoke lane, before the smoke below
TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL='<smoke URL>' node scripts/guarded-migrate.cjs
```

**Frontend** (no code change): `npm run sync-contracts` (expect 0 changed), then `npm run typecheck && npm run lint && npm test && npm run build`.

**Smoke.** No browser-visible change, so it is not required by the rule. Run it once anyway, because the smoke replaces `google.gmail` (fact 11) and this ticket changes how Gmail clients are built. From the frontend: `SMOKE_DATABASE_URL=<empty migrated smoke DB> node scripts/smoke-stabilization.mjs`; expect PASS, residue 0.

**Live Gmail.** One manual sync on the owner's account after the change is useful but optional. Call it live evidence only if it really ran. Loopback results are fixture evidence.

## 12. Documentation updates

- [operations.md](operations.md):65 (transport paragraph): state the delivered bounds (15 s Gmail, 10 s OAuth, 5 s revoke), SDK retries off, the one attempt deadline, cancellation, and when a refresh happens (before expiry; reactive only for rows with no stored expiry). If S6-C02 added a "target contract" banner, narrow it so it no longer covers this paragraph.
- [follow-up-gmail-reliability.md](./follow-up-gmail-reliability.md): a dated note that FU-2, FU-4.3, the cancellation case of FU-4.2 and the four categories from §7.3 item 5 were delivered by S5-FU-02, with an evidence link.
- Backend `STABILIZATION.md:140` (the Gmail recovery rule, which says each scan is bounded to "four minutes plus the current request"; S6-C02 and S5-FU-01 also edit it): add the transport bounds, SDK retries off, the one attempt deadline and cancellation.
- [Sprint 8 README](../sprint-8/README.md) §7: tick the S5-FU-02 line with evidence links. Sprint 8 `execution-report.md` (new, dated).
- [Roadmap](../README.md) §1 row "Sprint 5 Gmail reliability": bounded Google calls delivered.
- This file: Status line, plus a dated "Resolution" section that records the §4.0 decision.

## 13. Definition of done

- [ ] Every §10 criterion is met, with evidence linked.
- [ ] Backend typecheck, lint, build, guarded migrate and full suite are green in CI (S7-01); the three lanes pass if the migration was added; frontend checks are green with 0 contract changes.
- [ ] Behavior verified: the loopback suite is green and the smoke passed once. Nothing synthetic is called live evidence.
- [ ] The §4.0 decision is recorded in this file (owner, date).
- [ ] Docs updated per §12.
- [ ] Focused commits in the backend repo (OD-02).
- [ ] Evidence (test counts, lane output, smoke result, decision) is recorded in the Sprint 8 `execution-report.md`.
