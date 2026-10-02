# S6-C01 — Make the Done claims provable: tests and lanes

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 6 closeout — order 4 of 8 ([closeout plan](README.md)) |
| Repository | career-companion-backend (tests and scripts only); career-companion-docs (one note in AI-18, evidence record) |
| Size / priority | M / prove existing Done claims before acceptance is recorded |
| Depends on | [Phase 0 gate](../../migration-verification/README.md#mv-16--gate-record-results-and-decisions) passed (MV-16; git restored); runs after AI-19, AI-20 and S6-R07 in closeout order (soft; see §8) |
| Blocks | [S6-C02](S6-C02-docs-truth-and-acceptance-records.md) (its acceptance records cite these results) |
| Source | [State audit 2026-10-02](../../state-audit-2026-10-02.md) TEST-08, TEST-11, TEST-05, AIB-10, AIB-09, TEST-03; [roadmap §5](../../README.md#5-byo-ai-what-blocks-production-ready) item 4; [closeout README](README.md) §1 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Turn six Done claims that rest on checks that cannot fail into checks that can. Each new or changed check is shown to fail when the behavior it names is removed. No production code changes.

## 2. Why it exists

- The [Sprint 6 execution report](../execution-report.md) (line 34) lists "manual PATCH versus an AI write holding the application lock" as verified safe. The test behind it never sends the PATCH while the lock is held (TEST-08).
- The [BYO AI execution report](../../byo-ai/execution-report.md) (line 29) says the AI-16 test scans "every table". It scans only the `public` schema with a mocked queue. [AI-16](../../byo-ai/issues.md) scope 1 requires `pgboss.job.data` and `output`. Production gating of the settings API has no API test (AIB-10).
- The owner re-runs the Sprint 6 lane script on the new PC. On failure it prints `FAIL:` and never exits (TEST-11).
- On the new PC the owner will create a developer backend `.env` (the dev server for MV-14 and MCP-09 part B needs one). Some of its values leak into the test process and can fail tests for reasons unrelated to the code (TEST-05).
- The Sprint 6 zero-side-effect test snapshots a table nothing writes any more (AIB-09). A lane comment is stale, and the AI lane will break when AI-18 lands (TEST-03).

The closeout goal is that every Done claim is backed by a test that can fail ([README §1](README.md#1-goal)).

## 3. Current behavior

Line numbers were checked on 2026-10-02 against the backend folder. Re-check them before editing.

**3.1 Lock test (TEST-08).** `src/tests/application-status.test.ts:299-319`, test "serializes a manual write against an AI write holding the application lock". The `patch()` helper (:35-36) returns an unsent supertest request; supertest sends only when `.then()` is called. Line 312 builds the PATCH, :313 sleeps 150 ms, :314 releases the lock, :315 awaits the AI write, and :316 `await manual` is the first send. So the AI write commits first and the PATCH runs after it. The test passes with or without the lock. Production code does lock: `ApplicationService.updateUserStatus` (`src/services/application.ts:233-264`) selects the owned row `FOR UPDATE` at :239-242. Removing that clause would not fail the test.

A second fact matters for the fix. In a changed-value PATCH (`null` → `OFFER`) the `UPDATE` at :249-258 also waits on the row lock, and under PostgreSQL's default READ COMMITTED level it re-reads the row after the AI write commits. So even an eager changed-value test still passes without `FOR UPDATE`. Only a same-value (no-op) PATCH, which does no `UPDATE`, depends on the read-time lock. Without it, that PATCH never waits and returns at once with the old `aiStatus`.

**3.2 Sprint 6 lane exit (TEST-11).** `scripts/verify-migration-preservation.cjs`: `connectEmpty` (:30-37) connects at :33 and can throw at :35. Clients close only at :61 and :158, at the end of a successful lane. The top-level handler (:178-182) sets `process.exitCode = 1` and removes the work folder, but never calls `process.exit()`. An open `pg` client keeps Node running after `FAIL:`. The sibling scripts already have the fix: `verify-ai-migration.cjs:162-165` and `verify-mcp-migration.cjs:222-225` call `process.exit()` in `finally`.

**3.3 Developer `.env` leak (TEST-05).** `src/tests/setup.ts:6` loads `.env.test` with `override: true`. Later, `src/index.ts:11` calls plain `dotenv.config()`, which adds any variable from a backend `.env` that `.env.test` does not set. 17 test files import the app. Variables read by the code but absent from `.env.test`:

- `MCP_ALLOWED_HOSTS`: read once when `index.ts:37` calls `createMcpRouter()`, whose default argument is `mcpConfig()` (`src/mcp/router.ts:70`, `src/mcp/config.ts:20-26`). Tests use Host `127.0.0.1` (`mcp-endpoint.test.ts:78-80`). A dev value without `127.0.0.1` (for example `localhost`) turns almost every app-level `/mcp` test in that file into 403.
- `MCP_ALLOWED_ORIGINS`: read at the same point. A dev value matters only if it allows a name the rejection tests use (`evil.example` at `mcp-endpoint.test.ts:268`, `ide-123.example` at :338).
- `MCP_DAILY_SUBMISSION_LIMIT`: read per call (`src/services/externalSubmission.ts:220-225`, `submissionDailyLimit`). `mcp-endpoint.test.ts:83` and `external-submission.test.ts:62` reset it. `application-submission-evidence.test.ts` and `submission-review.test.ts` never do, so a dev value of `0` (the kill switch) or a small number breaks them.
- `TRUST_PROXY_HOPS` (`index.ts:32`, `src/utils/config.ts:4-10`): a valid value changes nothing tests can see; an invalid one fails loudly at import.
- `RELEVANCE_CONFIDENCE_THRESHOLD` (`src/services/ai/pipeline.ts:50`, default 0.7; note `Number('')` is 0): a value above about 0.92 would flip current pipeline fixtures.

No effect in tests: `PORT` (`index.ts:103` listens only outside `NODE_ENV=test`); `AI_DAILY_CALL_LIMIT` and `GEMINI_*`, checked only for production (`src/utils/config.ts:11`, `:30-34`); `AI_EVAL_API_KEY`, read only by the eval runner (`src/eval/ai/run.ts:56`). Gemini SDK env flags are AIB-11 and belong to AI-19.

Second path: the Prisma 6.19.3 client loads the root `.env` itself (no override) when it was generated while a `.env` existed. Today `node_modules/.prisma/client/index.js:520-522` has `rootEnvPath: null` because no `.env` existed at generate time. It will be set if `npm run db:generate` runs while a backend `.env` exists, which is likely on the new PC. This was verified by reading the runtime, not reproduced. So skipping `dotenv.config()` in `index.ts` alone does not fix the leak.

**3.4 BYO AI claims (AIB-10).**
- `src/tests/ai-redaction.test.ts:26` mocks `../services/queue`, and `storedText()` (:55-62) scans only `schemaname = 'public'`. Nothing reaches `pgboss.job`, so the AI-16 job-table scan is not done. Partial cover exists elsewhere: `email-worker-reliability.test.ts:332-386` runs the real queue and checks `pgboss.job.output` for a private string, and `EmailJobFailure` (`src/jobs/emailProcessingJob.ts:64-70`, thrown at :174) carries only the category.
- Production gating is tested only through `deriveAccess` (`ai-access.test.ts:58-67`). `GET` builds `offeredProviders` with `isOffered` (`src/services/ai/settings.ts:83`; `src/services/ai/access.ts:64-65`). `PUT` returns 400 "That AI provider is not offered." through `offeredProvider` (`settings.ts:151-152`). No API test runs these with `NODE_ENV=production`. All three providers are `hidden` (`src/contracts/aiCatalog.ts:72,127,182`).
- Development auth is refused in production at request time (`src/middleware/auth.ts:6`, :50-56), so a production API test needs a real session cookie. A pattern exists: `signSid` at `mcp-endpoint.test.ts:51-52` and the session row at :235-246.
- The 401 list in `ai-settings.test.ts:54-62` leaves out `POST /api/ai/settings/sample-test`. The route is still protected: `router.use(requireAuth)` at `src/routes/ai.ts:27`.

**3.5 Dead snapshot (AIB-09).** `sideEffectCounts()` in `application-status.test.ts:40-50` snapshots `prisma.aICallBudget.findMany()` (:45), compared at :268 and :271. No runtime code reads or writes `ai_call_budgets`. Calls are counted in `ai_usage_days` by `reserveUserCall` (`src/services/ai/usage.ts:45-64`, upsert at :58). The test's other checks (provider mock, queue mocks, `ai_operations` count) still catch a stray AI call, so this is weak coverage, not a broken test.

**3.6 Lane order and AI-18 (TEST-03).** `verify-migration-preservation.cjs:17` and :125-131 build the "legacy" schema from every migration except `20261002090000_add_user_status_revision`. That now includes `20261002120000_user_provided_ai` and `20261002150000_automation_submissions`, so the Sprint 6 migration is applied after newer ones. The comment at :125, "Pre-Sprint-6 schema", is stale. `verify-ai-migration.cjs` does the same (:17, :117-119), inserts into `ai_call_budgets` (:91) and snapshots it (:101). Once AI-18 adds `DROP TABLE ai_call_budgets`, that drop runs inside the legacy schema and the AI lane fails. The AI-18 grep criterion (`aICallBudget`, [issues.md AI-18](../../byo-ai/issues.md)) does not find this raw SQL. The roadmap banner at the top of `byo-ai/issues.md` (line 8) already says AI-18 must update `verify-ai-migration.cjs`. The AI-18 section itself does not say why (lane order) or how (a cutoff), and its grep still misses the raw name. After [AI-19](../../byo-ai/AI-19-runtime-safety-fixes.md) (closeout order 1, §4 item 1.8), `verify-ai-migration.cjs` excludes a list, `20261002120000_user_provided_ai` and AI-19's `<timestamp>_ai_access_paused`, instead of one name; the AI-18 hazard is unchanged (2026-10-02).

**Facts vs risks.** No production defect is known for any item. The gaps are in evidence. The suspected risks are a false "verified" claim (3.1, 3.4), a hung re-run (3.2), false test failures on the new PC (3.3) and a future AI-18 surprise (3.6).

## 4. Scope

1. **Lock test (TEST-08)** in `application-status.test.ts`:
   - Inside the AI transaction, read its backend pid (`SELECT pg_backend_pid()`) before taking the `FOR UPDATE` lock.
   - Send the PATCH eagerly, for example `const manual = patch(...).then((r) => r)`.
   - Poll `pg_stat_activity` until a backend with `wait_event_type = 'Lock'` has the AI pid in `pg_blocking_pids(pid)`. Bound the poll (about 3 s, short steps) and fail with a clear message if it never sees the wait. Then release.
   - Remove the 150 ms sleep at :313.
   - Apply the eager send and the wait to the existing changed-value case, and keep its assertions. It proves the PATCH waits and returns the committed AI write, but it cannot detect a missing `FOR UPDATE` (3.1).
   - Add a same-value case: the application starts with `userStatus = 'OFFER'` and revision 1, the PATCH sends `OFFER` with revision 1, and the response must show `aiStatus: 'INTERVIEW'`, `userStatus: 'OFFER'`, revision 1.
   - Release the gate in `finally`, so a failed poll never leaves the AI transaction open.
2. **Lane exit (TEST-11).** In `verify-migration-preservation.cjs`, make `finally` remove the work folder and then call `process.exit()`, with the same comment as the sibling scripts.
3. **Test environment pinning (TEST-05).** Implementation choice, no owner decision needed. Recommendation: after `setup.ts` loads `.env.test`, set each name below to the `.env.test` value if it has one, else to the pinned value. Neither loader overrides a key that already exists, so this covers `index.ts:11` and the Prisma loader without touching production code. Skipping `dotenv.config()` in `index.ts` under `NODE_ENV=test` is not enough alone (3.3) and changes production code.

   | Name | Pinned value | Same as code default |
   | --- | --- | --- |
   | `MCP_ALLOWED_HOSTS` | `''` (means localhost, 127.0.0.1, [::1]) | yes |
   | `MCP_ALLOWED_ORIGINS` | `''` (no Origin allowed) | yes |
   | `MCP_DAILY_SUBMISSION_LIMIT` | `500` | yes |
   | `TRUST_PROXY_HOPS` | `''` (acts as unset: `index.ts:32` and `config.ts:5` test truthiness) | yes |
   | `RELEVANCE_CONFIDENCE_THRESHOLD` | `0.7` (not `''`, which would mean 0) | yes |

   The key must exist, even as an empty string, so that neither loader sets it. Deleting it in `setup.ts` would let the Prisma loader put the `.env` value back.

   Keep the list in one small helper used by `setup.ts` and by the regression test. Add a regression test: write a temporary env file with dev-style values for every pinned name, load it with plain `dotenv.config({ path })` (no override, as both leaky loaders do), then import the app dynamically. Static imports run first, so do the load in `beforeAll` and then `await import('../index')`. Assert that every pinned name keeps its test value, that `submissionDailyLimit()` returns 500, and that `POST /mcp` with no token returns 401, not 403. Never create or edit the real backend `.env` in a test.
4. **BYO AI proofs (AIB-10):**
   - (a) In `ai-redaction.test.ts`, replace the full queue mock with the real queue, observed through a wrapper as `application-status.test.ts:22-26` does. Then run at least one email through the real worker:
     - Follow `email-worker-reliability.test.ts:332-386`: clear queued email jobs first (:334), `send` with `retryLimit: 0`, `startEmailProcessingWorker`, bounded poll, `offWork`.
     - Make the job fail with a raw, non-AI error that echoes both sentinels. For example, the mocked `GmailFetcherService.fetchMessageMetadata` rejects with an `Error` whose message holds both. The real Gmail wrapper that would sanitize it (`src/services/gmailClient.ts:39`) is mocked away. pg-boss then stores the failure output.
     - Why a raw error: provider SDK errors are already reduced to a `ProviderFailure` by the adapter (`classifyGeminiError`, `src/services/ai/providers/gemini.ts:77-93`). Only a raw error makes the output mutation in §11.3 detectable.
     - Extend `storedText()` to every table in the `public` and `pgboss` schemas. Assert first that `pgboss.job` holds at least one job for this user with non-null output, so the scan is meaningful.
     - In `afterAll`, delete this user's jobs and stop the queue.
   - (b) In `ai-settings.test.ts`, add an API test that sets `NODE_ENV=production` inside `try/finally` and authenticates with a real session row and signed `cc_session` cookie. `GET /api/ai/settings` returns `offeredProviders: []`. `PUT` with `provider: 'gemini'` returns 400 with `VALIDATION_ERROR` and "That AI provider is not offered.", stores no configuration and builds no provider client.
   - (c) Add `['post', '/api/ai/settings/sample-test']` to the 401 list.
5. **Side-effect snapshot (AIB-09).** In `sideEffectCounts()`, replace `budgets` with `usageDays: prisma.aIUsageDay.findMany({ orderBy: [{ userId: 'asc' }, { day: 'asc' }] })`. After this, no test references `aICallBudget`.
6. **Lane comment and AI-18 note (TEST-03).** Replace the comment at `verify-migration-preservation.cjs:125` with a true one: the legacy schema is every migration except the Sprint 6 one, so it includes the BYO AI and MCP migrations and the Sprint 6 migration is applied last, out of order. Add the AI-18 note in §12.
7. **Prove each check can fail.** First stage this ticket's changes (`git add`). Then run each mutation in §11.3, one at a time. Restore the file with `git checkout -- <file>`, which restores the staged version, and confirm `git diff --exit-code` is empty (byte-for-byte, as S6-R05 did). Staging matters because one mutation edits `setup.ts`, which this ticket also changes. Record the results.
8. **Defect rule.** If a check exposes a real production defect, stop. Record it in the closeout execution report and raise it with the owner before any fix. Do not fix production code in this ticket.

## 5. Out of scope

- TEST-06, TEST-09, TEST-14, TEST-15 and TEST-17 ([roadmap Not now](../../README.md#8-not-now), test hygiene).
- CI ([S7-01](../../sprint-7/S7-01-ci-and-toolchain-pins.md)).
- The rest of AIB-10: OpenAI and Anthropic end-to-end redaction (unit-covered), concurrent reservation at the limit (TEST-17, Not now), remove-during-job (AIB-08, Not now), Gemini env isolation (AIB-11, [AI-19](../../byo-ai/AI-19-runtime-safety-fixes.md)).
- BYO AI frontend test gaps from roadmap §5 item 4 (AI-11/AI-12): [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md).
- A migration cutoff in the lane scripts, a pre-Sprint-6-to-head rehearsal, and the AI-18 migration itself. After AI-19 the AI lane already uses an exclusion list (`20261002120000_user_provided_ai` and AI-19's migration); this ticket does not change `verify-ai-migration.cjs` (2026-10-02).
- The usage comment at `verify-migration-preservation.cjs:7` (no `npm run build`, PLAT-16). S6-C02 owns it.
- Rewording historical reports. [S6-C02](S6-C02-docs-truth-and-acceptance-records.md) writes the acceptance records and any banners.
- Any production code change, frontend change or contract change.

## 6. Likely files and components

Backend (`career-companion-backend`):
- `src/tests/application-status.test.ts`: lock test (§4.1) and `sideEffectCounts` (§4.5).
- `src/tests/setup.ts`, a new helper such as `src/tests/helpers/testEnv.ts`, and a new `src/tests/test-env-isolation.test.ts` (§4.3).
- `src/tests/ai-redaction.test.ts` and `src/tests/ai-settings.test.ts` (§4.4). Optionally a shared `src/tests/helpers/session.ts` for the signed cookie; leave `mcp-endpoint.test.ts` as it is.
- `scripts/verify-migration-preservation.cjs`: `finally` (§4.2) and the :125 comment (§4.6).

Docs (`career-companion-docs`): `docs/planning/byo-ai/issues.md` (AI-18 note) and the closeout execution report.

## 7. Implementation notes

- **Data model, migrations, contracts:** none. No `sync-contracts` change and no frontend impact.
- **Prisma transaction limits:** both transactions in the lock test use Prisma's defaults (timeout 5 s, maxWait 2 s). Keep the hold well under 5 s, or the AI transaction is rolled back and the test fails confusingly. The poll query needs a third pooled connection. `.env.test` sets no `connection_limit`, so Prisma's default pool (CPUs × 2 + 1) has at least three.
- **Background jobs:** the redaction test now starts a real pg-boss and an email worker. Stop the worker with `offWork` before the scan and call `stopQueue` in `afterAll`. Never run the suite against the smoke database while the smoke runs.
- **Production mode in a test:** `NODE_ENV` is read at request time by `isOffered` and `requireAuth`, but at import time by `index.ts` (session cookie `secure`, production config check). Flip it only inside `try/finally` around the requests and restore it.
- **Env pinning precedence:** for the pinned names, `.env.test` wins, then the pinned value; shell exports and backend `.env` are ignored. Write this in a comment in `setup.ts`. When the code starts reading a new variable that `.env.test` does not set, add it to the pinned list. Later tickets that add env vars must add them to this list: [S7-05](../../sprint-7/S7-05-twice-daily-scheduled-sync.md) (`GMAIL_SCHEDULED_SYNC_ENABLED`, `GMAIL_SCHEDULED_SYNC_TZ`) and, only if OD-11 adds it, [MCP-10](../../mcp-feature/MCP-10-post-verification-corrections.md) (`MCP_ACCEPT_NULL_ORIGIN` or its final name) (2026-10-02).
- **Failure and recovery:** a failing lock test must release the gate (`finally`). The lane script must exit with code 1 on every failure path.
- **Compatibility:** test counts rise above the Phase 0 baseline (621 on the old laptop). Record the new totals in the closeout execution report.
- **Rollback:** revert the commit. Only tests, one script and docs change.

## 8. Dependencies

- Phase 0 gate passed: git restored (needed for the restore checks in §11.3), test database migrated, lanes runnable on the new PC.
- Order: after AI-19, AI-20 and S6-R07 (closeout orders 1–3). Not a hard block. Only AI-19 shares a file: it also changes backend `src/tests/ai-settings.test.ts`, so build on its version of that file and its helpers. AI-20 is frontend-only.
- No owner decision (OD-nn) is needed.
- Blocks S6-C02: the Sprint 6 lock claim and the AI-16 claim are recorded as accepted only with this evidence.

## 9. Security and privacy

- Synthetic sentinels and fixture keys only. No real key, email or `.env` value goes into a test, log or evidence record.
- The wider scan (`pgboss` schema) strengthens the AI-16 privacy proof. A sentinel found there is a privacy defect: apply the §4.8 defect rule.
- The production-mode test must restore `NODE_ENV` and delete its session row, so production behavior does not leak into other tests.
- The env regression test writes only a temporary file under the OS temp folder and deletes it. It must never write the backend `.env`.
- The lock poll reads `pg_stat_activity` and `pg_blocking_pids` on the local test server only. It needs no extra privilege, because both backends use the same test role.

## 10. Acceptance criteria

- [ ] The lock test sends the PATCH before release, waits until `pg_stat_activity` shows it blocked by the AI transaction, and has no fixed sleep.
- [ ] With `FOR UPDATE` removed from `application.ts:242`, the same-value lock case fails; restored, it passes.
- [ ] A forced lane failure makes `verify-migration-preservation.cjs` print `FAIL:` and exit with code 1 without Ctrl+C. A normal run still prints `migration_preservation_verified` and exits 0.
- [ ] With a backend `.env` that sets `MCP_ALLOWED_HOSTS=localhost` and `MCP_DAILY_SUBMISSION_LIMIT=0`, `mcp-endpoint`, `submission-review` and `application-submission-evidence` tests pass.
- [ ] The env regression test passes, and fails when the pinning call in `setup.ts` is removed.
- [ ] `ai-redaction.test.ts` scans the `public` and `pgboss` schemas, proves at least one job row with output exists for its user, and fails under the §11.3 output mutation.
- [ ] Under `NODE_ENV=production`, the API returns `offeredProviders: []` and a 400 "not offered" for `PUT` with a hidden provider; the test fails when `isOffered` always returns true.
- [ ] `POST /api/ai/settings/sample-test` is in the 401 list.
- [ ] `sideEffectCounts()` snapshots `ai_usage_days`, and the side-effect test fails when `updateUserStatus` temporarily reserves an AI call.
- [ ] The :125 comment is true, and the AI-18 note (§12) is in the AI-18 section of `byo-ai/issues.md`.
- [ ] After each mutation is restored, `git diff --exit-code` is empty.
- [ ] Full backend suite, typecheck, lint (0 errors) and build pass; the three migration lanes pass.

## 11. Testing

**11.1 Tests to add or change:** `application-status.test.ts` (eager changed-value case, new same-value case, usage-days snapshot); `test-env-isolation.test.ts` (new); `ai-redaction.test.ts` (real queue, worker pass, `pgboss` scan); `ai-settings.test.ts` (production API test, 401 row).

**11.2 Commands** (backend repo root; databases per the Phase 0 lane guide):

```sh
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs   # only if not migrated
npx vitest run src/tests/application-status.test.ts
npx vitest run src/tests/test-env-isolation.test.ts src/tests/mcp-endpoint.test.ts src/tests/submission-review.test.ts src/tests/application-submission-evidence.test.ts
npx vitest run src/tests/ai-settings.test.ts src/tests/ai-redaction.test.ts
npm test
# each lane on two empty career_companion_*test databases; drop and recreate them between scripts
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-mcp-migration.cjs
# forced failure: FRESH points at a throwaway non-empty guarded DB (one table); throws at the empty check, before any write
FRESH_DATABASE_URL=<non-empty DB> UPGRADE_DATABASE_URL=<empty DB> node scripts/verify-migration-preservation.cjs; echo "exit=$?"
```

For the `.env` check: if a backend `.env` exists, copy it aside first. Add `MCP_ALLOWED_HOSTS=localhost` and `MCP_DAILY_SUBMISSION_LIMIT=0` (or create the file with only those lines), run the four-file command above, then restore the original file (or delete the one you created).

**11.3 Mutation checks** (stage this ticket's changes first; one at a time; never committed; restore with `git checkout -- <file>` and confirm `git diff --exit-code`):

| Temporary change | Test that must fail |
| --- | --- |
| Delete `FOR UPDATE` at `src/services/application.ts:242` | Same-value lock case (the PATCH never waits, so the bounded poll fails) |
| Remove the new `process.exit()` from `scripts/verify-migration-preservation.cjs` | The forced-failure run in §11.2 hangs after `FAIL:` (stop it with Ctrl+C) |
| Remove the pinning call from `src/tests/setup.ts` | Env regression test |
| `throw err` instead of `new EmailJobFailure(...)` at `src/jobs/emailProcessingJob.ts:174` | `ai-redaction.test.ts` (sentinel in `pgboss.job.output`) |
| Make `isOffered` return `true` (`src/services/ai/access.ts:64-65`) | Production API test |
| Comment out `router.use(requireAuth)` at `src/routes/ai.ts:27` | The sample-test 401 row (with the others) |
| Import `reserveUserCall` and call `reserveUserCall(tx, userId, new Date())` inside `updateUserStatus` | Zero-side-effect test |

Frontend: no change, no commands needed. Browser smoke: not needed; nothing browser-visible changes.

## 12. Documentation updates

- `docs/planning/byo-ai/issues.md`, AI-18 "Risks / notes": add a short dated note. The S6 and AI lane scripts apply an older migration after newer ones (legacy schema = all migrations but the excluded ones; after AI-19 the AI lane already excludes a list: `20261002120000_user_provided_ai` and AI-19's migration). So AI-18 must update `verify-ai-migration.cjs`, which inserts into `ai_call_budgets` (:91) and snapshots it (:101): add its migration to that list or switch to a cutoff, and either way change that insert and snapshot. Its grep must also search the raw name `ai_call_budgets` (also in backend `STABILIZATION.md:47` and `sprint-5/operations.md:44`). Leave the roadmap banner at the top of the file (line 8) as it is.
- Closeout execution report (this folder; create `execution-report.md` if no earlier closeout ticket has): the §11.3 results, the new test totals, the lane outputs and the forced-failure exit code.
- No edit to the Sprint 6 or BYO AI execution reports here. S6-C02 cites this evidence.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend full suite, typecheck, lint (0 errors) and build are green on the new PC, and in CI once S7-01 exists.
- [ ] Every §11.3 mutation was seen to fail and was restored byte-for-byte.
- [ ] The AI-18 note is added; no other doc is rewritten.
- [ ] One focused commit in the backend repo (tests and script only) and one in the docs repo.
- [ ] Evidence is recorded in the closeout execution report. Any defect found is recorded there and raised with the owner, not fixed here.
