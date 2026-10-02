# AI-19 — BYO AI runtime safety fixes for daily use

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 6 closeout — order 1 of 8 |
| Repository | backend (main); frontend (client deadline and contract sync only); shared contracts (`src/contracts/aiCatalog.ts`); docs (one-line corrections, §12) |
| Size / priority | M / highest closeout risk once a real key is used daily |
| Depends on | [Phase 0 gate](../migration-verification/README.md) passed (MV-16), git restored (MV-03). Constraint from OD-09: wording only here, no disclosure version bump |
| Blocks | [AI-15](AI-15-certify-gemini-first.md) (the owner approves the data-use text only after this ticket corrects it, OD-09); [AI-20](AI-20-recovery-path-and-status-ui.md) item 22 (per-user pause copy follows the storage choice in §4 item 1, OD-14); [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) acceptance record |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) AIB-05, AIB-11, AIB-03, PLAT-15, TEST-07, AIB-07, AIF-04; [roadmap §5 item 2](../README.md#5-byo-ai-what-blocks-production-ready); [BYO plan](README.md) §3.4, §3.8; [issues.md](issues.md) AI-07 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Make BYO AI safe to use every day with a real key. Six small fixes:

1. A provider that rejects every request (HTTP 400) stops after a few emails, not after hundreds.
2. Server environment flags cannot switch the Gemini SDK into Vertex mode.
3. `@google/genai` is pinned exactly, and one test runs the SDK's real error path.
4. Two overlapping first-time saves both succeed instead of one returning 500.
5. Save and "Check again" get a client deadline longer than the server's worst case.
6. The draft data-use text says truthfully when the email text is sent.

## 2. Why it exists

After Phase 0 the owner uses BYO AI daily with a real key (hidden providers are offered outside production). These gaps show up only then. All six were confirmed or partly confirmed by the audit verifier. Item 6 must land before the owner approves the data-use text in AI-15 (OD-09). See the [roadmap §5](../README.md#5-byo-ai-what-blocks-production-ready) and the [closeout plan](../sprint-6/closeout/README.md).

## 3. Current behavior

Line numbers are from 2026-10-02. Installed-SDK references are to `node_modules/@google/genai` 2.22.0 (the backend is CommonJS, `package.json:29`, so `dist/node/index.cjs` is the build that runs).

### 3.1 INVALID_REQUEST has no damage bound (AIB-05)

Facts:
- `recordFailure` (`src/services/ai/operations.ts:200-207`) sets the operation `FAILED` with `errorCode = INVALID_REQUEST` and returns a `TerminalAIError`, which the caller throws (`:136`). It writes nothing to the user's configuration. Refusals and `OUTCOME_UNKNOWN` do: they go through `noteProviderFailure` (`src/services/ai/usage.ts:104-131`). `OUTCOME_UNKNOWN` in particular pauses the user so an outage holds one call per window (`operations.ts:214-217`).
- The worker marks the email `FAILED` (`src/jobs/emailProcessingJob.ts:142-158`) and acknowledges the job, because a terminal error is not re-thrown (`:174`). The next email sends the same request. The only bound is the per-user safety limit, default 500 calls a day (`usage.ts:14`, `reserveUserCall` at `usage.ts:45-64`).
- `holdOf` (`src/services/ai/heldOperations.ts:29-40`, `FAILED` at `:32-33`) makes these rows operator-only. `POST /api/emails/:id/retry` answers 409 `AI_OPERATION_REQUIRES_REVIEW` (`src/routes/email.ts:191-198`). It reads held operations of every contract version (`email.ts:164-169`), so a version bump does not release them. `STABILIZATION.md:138` says no reset command exists.
- Not every 400 lands here. OpenAI `model_not_found` and Anthropic `not_found_error` already map to `MODEL_UNAVAILABLE`, a released refusal (`openai.ts:22`, `anthropic.ts:34`). Any other status below 500 that no earlier rule matches (for example 400 or 413) falls through to `INVALID_REQUEST` (`classifyOpenAIError`, `openai.ts:25`; `classifyAnthropicError`, `anthropic.ts:38`; `classifyGeminiError`, `gemini.ts:92`).
- The email worker takes one job per callback (`EMAIL_WORKER_OPTIONS`, `emailProcessingJob.ts:184`, registered at `:203`). pg-boss 12.31 defaults `localConcurrency` to 1, so one process runs one email job at a time.
- `PAUSED` already exists as an access reason (`src/services/ai/errors.ts:111-121`; `AIAccessReasonSchema`, `src/contracts/ai.ts:14-25`). `stateOf` maps it to `LIMITED` (`src/services/ai/access.ts:73-78`). The frontend has copy for it (`src/lib/aiLabels.ts:84-85`: "Career Companion has paused AI processing for now."). Today it comes only from `AI_USER_DAILY_CALL_LIMIT=0` (`deriveAccess`, `access.ts:93`). The stored enum `AIAccessIssue` (`prisma/schema.prisma:364-371`) has no `PAUSED`.

Suspected risk (not demonstrated): the likely triggers are request-shape rejections, for example `reasoning_effort: 'minimal'` (`aiCatalog.ts:157,169`, sent at `openai.ts:49-51`) or a Gemini response-schema 400. Latent today because no provider is used with a real key yet.

### 3.2 Gemini client follows Vertex env flags (AIB-11)

- `createGeminiClient` (`src/services/ai/providers/gemini.ts:95-100`) passes `{ apiKey, httpOptions: { baseUrl, retryOptions: { attempts: 1 } } }` and no `vertexai` or `enterprise` option.
- In the SDK, `resolveCloudFlag` (`index.cjs:26125-26152`) reads `GOOGLE_GENAI_USE_ENTERPRISE`, then `GOOGLE_GENAI_USE_VERTEXAI`, only when both options are undefined. Explicit options win. Equal values are allowed; different values throw (`:26129-26133`). Both options exist in 2.22.0 (`dist/node/node.d.ts:6809`, `:6822`).
- Nothing leaks: `getBaseUrl` returns `httpOptions.baseUrl` (`index.cjs:79-90`). But Vertex mode uses API version `v1beta1` (`VERTEX_AI_API_DEFAULT_VERSION`, `:13444`) and `publishers/google/models/<id>` paths (`tModel`, `:3295`), so every Gemini call and verify would fail. The exact HTTP status is unverified.
- OpenAI and Anthropic already pin the options their SDKs would read from the environment (`openai.ts:31`: `organization: null, project: null`; `anthropic.ts:43`: `authToken: null`). `src/utils/config.ts:30-32` rejects only `GEMINI_*` variables. The test `src/tests/ai-providers-gemini.test.ts:89-92` asserts the exact constructor options.

### 3.3 Gemini SDK not pinned; error format untested (AIB-03, PLAT-15)

- `package.json:36` has `"@google/genai": "^2.22.0"`; `:35` and `:48` pin the other SDKs exactly. `package-lock.json:13` (root entry) repeats the caret; `:247-248` resolves 2.22.0. BYO plan §3.4 (`README.md:190`) and E1 (`:888`) require exact pins; `README.md:876` and `execution-report.md:28` list only `openai` and `@anthropic-ai/sdk`.
- `googleErrorBody` (`gemini.ts:52-70`) JSON-parses `err.message`; `classifyGeminiError` maps a 400 to `KEY_REJECTED` only through reason `API_KEY_INVALID` (`gemini.ts:86`). This works because `throwErrorIfNotOK` (`index.cjs:14090-14120`) sets the message to `JSON.stringify(errorBody)` (`:14110`).
- The tests mock `GoogleGenAI` (`ai-providers-gemini.test.ts:13-21`) and build `ApiError` messages by hand (`:24-26`), so the real error path never runs.
- Correction from the verifier: plain `npm install` keeps the locked 2.22.0. Only `npm update`, a regenerated lockfile or an explicit upgrade would move it. If the message format changed, a bad key would become `INVALID_REQUEST` (`gemini.ts:92`), verify would return `INCONCLUSIVE`, and the key would be saved unverified (`settings.ts:194-231`).

### 3.4 Concurrent first-time save returns 500 (TEST-07)

- `saveSettings` (`src/services/ai/settings.ts:146-237`) reads the row outside a transaction (`:153-156`), reserves a verification and calls the provider (`:181-182`, up to about 20 s), then creates the row when none was read (`:220-231`). `userId` is the primary key (`schema.prisma:376`).
- The slower of two first-time saves throws Prisma `P2002`. It is not a `SettingsError`, so `handle` passes it on (`src/routes/ai.ts:31-45`, `:42`) and `errorHandler` returns 500 (`src/middleware/error.ts:40-45`). The AI-07 plan says "upsert with `revision + 1`" (`issues.md:393`). The only `P2002` handling is `src/services/externalSubmission.ts:298`. No concurrent PUT test exists.
- Impact is small: the form blocks a double submit (`ProviderSetupForm.tsx:90`, `:243`). Real triggers are two tabs, or pasting the key again after the 15 s client timeout. The client treats 5xx as outcome-uncertain (`src/api/client.ts:67-70`) and refetches. The first save wins.

### 3.5 Save and check outlast the client deadline (AIB-07)

- `VERIFY_TIMEOUT_MS = 10_000` (`src/services/ai/providers/types.ts:35`). Each adapter looks up each distinct model one after another (`gemini.ts:137-149`, `openai.ts:75-87`, `anthropic.ts:97-109`). Gemini and OpenAI defaults use two models; Anthropic recommends one model for both roles (`aiCatalog.ts:210`).
- Save then writes the row, awaits `reofferPendingEmails` (`settings.ts:235`; up to `REOFFER_LIMIT = 100` sequential queue sends, `gmailSync.ts:22`, `:31-41`) and `readSettings` (`:236`). `checkSettings` does the same: verify at `:260-262`, re-offer only on `VERIFIED` at `:280`, `readSettings` at `:289`. No server code stops when the client disconnects (grep only; unverified beyond that).
- Frontend: `REQUEST_TIMEOUT_MS = 15_000` (`client.ts:46-47`) is the default (`:94`) for `saveAISettings` (`:310-316`) and `checkAISettings` (`:318-324`). The sample test has its own 75 s (`:77-78`, `:331`). On timeout, save shows "We could not confirm whether this was saved…" and refetches (`ProviderSetupForm.tsx:27-28`, `:116`); check shows "The check could not be completed." (`AIStatusPanel.tsx:59-62`).
- A single hung lookup ends at about 10 s with `INCONCLUSIVE`. Going past 15 s needs two slow but successful lookups, so this is rare.

### 3.6 Data-use summary understates when the email text is sent (AIF-04)

- `AI_DATA_SENT_SUMMARY` (`src/contracts/aiCatalog.ts:62-64`, identical in the frontend copy) says the email text is sent "For job-related emails". It is shown before consent (`ProviderSetupForm.tsx:208`).
- The pipeline forces the decision to `UNCERTAIN` below the confidence threshold (`pipeline.ts:73`; default 0.7 at `:50`) and fetches and sends the body whenever the decision is not `IRRELEVANT` (`:93-102`; limit 8,000 at `contracts.ts:126`). So the text also goes out for `UNCERTAIN` and for low-confidence emails the model called `IRRELEVANT`. `src/tests/ai-idempotency.test.ts:161-183` locks this in.
- Emails labelled Promotions, Social or Spam are never sent (`pipeline.ts:47-49`), so "for each new email" overstates. Overstating is safe; it is left as is.
- Consent records only the provider's `disclosure.version` (checked at `settings.ts:163-166`, stored at `:206`, compared at `:107`). Gemini's is `gemini-draft-2026-10` with `reviewedOn: null` (`aiCatalog.ts:87`, `:93`). The same wording is in `ai-capability-architecture.md:132` and `byo-ai/README.md:550`; `email-ai-pipeline.md:11,17` is correct.

## 4. Scope

1. **Pause a user after a systematic INVALID_REQUEST (AIB-05).**
   1. When an operation ends `INVALID_REQUEST`, `recordFailure` counts, in the same transaction, the user's `FAILED` operations with `errorCode = INVALID_REQUEST` on **distinct emails**, for the **same provider and model**, with `startedAt` in the last **30 minutes** (the current one included). At **3 or more** it pauses the user. One or two per-email 400s (for example one oversized email) never pause anyone.
   2. The pause write is guarded by `revision`, like every job-side write (plan §3.8). It logs `ai_requests_paused { userId, provider, model, failures }` with no content.
   3. While paused: `deriveAccess` returns `LIMITED` / `PAUSED` with `resumesAt: null`, checked after needs-attention issues and before the cooldown. `reserveUserCall` refuses with `AIAccessError('PAUSED')`, so a job cannot claim its second call. Emails wait as `PENDING` through the existing path (`emailProcessingJob.ts:114-141`); nothing is re-offered (`gmailSync.ts:32`); approved retries and the sample test get the existing 409 `AI_ACCESS_UNAVAILABLE`.
   4. Nothing lifts the pause by accident: `noteProviderSuccess` already clears only `RATE_LIMITED`/`PROVIDER_UNAVAILABLE` (`usage.ts:142`); the cooldown branch of `noteProviderFailure` (`usage.ts:119-130`) must skip a paused row; "Check again" with `VERIFIED` does not clear it (`settings.ts:273-280` clears needs-attention issues only). A definitive rejection may replace it, as today: on check (`settings.ts:281-285`) or from a call already in flight (`usage.ts:112-117`). Both need the user anyway.
   5. **User resume:** a successful save (`VERIFIED` or `INCONCLUSIVE`) clears the pause, for example after choosing another model. A save with a new key already clears `accessIssue` (`settings.ts:200`). A save that keeps the key does not today (`settings.ts:201-205` clears needs-attention issues only on `VERIFIED`), so that branch must also clear a stored `PAUSED`. If the problem remains, the pause returns at the next failure while the earlier ones are inside the window, and after at most three more otherwise.
   6. **Operator resume:** the documented procedure in §7.
   7. Constants next to the cooldown constants in `usage.ts`: `INVALID_REQUEST_PAUSE_COUNT = 3`, `INVALID_REQUEST_PAUSE_WINDOW_MS = 30 * 60_000`.
   8. **Keep the AI migration lane working (2026-10-02).** Without this, the new migration breaks `scripts/verify-ai-migration.cjs` (§7 "Lane hazard"). Replace `NEW_MIGRATION` (`:17`) with an excluded list holding `20261002120000_user_provided_ai` and this ticket's `<timestamp>_ai_access_paused`, and skip every listed name in the copy loop (`:117-118`). The upgrade step (`guardedMigrate(url)`, `:125`) then applies both, in name order. After the upgrade, assert that `PAUSED` is a value of `AIAccessIssue` (for example `enum_range(NULL::"AIAccessIssue")`). Update the header comment (`:1`) to name both migrations. The MCP and Sprint 6 lanes need no change (§7). If OD-14 picks a ledger-only pause, there is no migration and no lane change.

   **Decision OD-14 (owner, at ticket start): how the pause is stored.** Recommendation: add `PAUSED` to the Prisma enum `AIAccessIssue` (one additive migration), store it in `accessIssue` (model in `accessIssueModel` for diagnostics only), and show it as the existing access reason `PAUSED`. No API, shared-contract or frontend label change. Why: `PROVIDER_UNAVAILABLE` would say "did not respond" and resume by itself when the cooldown ends; `MODEL_UNAVAILABLE` would say the key cannot use the model, and "Check again" would clear it and restart the failures; a pause derived only from the ledger ends when the window passes, so it limits the rate but never stops. Record the choice in the execution report.

2. **Pin the Gemini client to the Gemini API (AIB-11).** Pass `vertexai: false` and `enterprise: false` in `createGeminiClient`, with a comment like `openai.ts:31`. Update the assertion at `ai-providers-gemini.test.ts:89-92`. Prove it with the real-SDK test in item 3.
3. **Pin `@google/genai` exactly and test the real error path (AIB-03, PLAT-15).** Set `"2.22.0"` in `package.json:36` and in the lockfile root entry (`package-lock.json:13`); the resolved entry stays 2.22.0. Add one test file that drives errors through the SDK's own `throwErrorIfNotOK` with a mocked `fetch` `Response` (§11).
4. **Concurrent first-time save (TEST-07).** In the create path, catch `Prisma.PrismaClientKnownRequestError` with code `P2002` and apply the same save as an update (`revision + 1`, key sealed for this user, consent recorded). The result equals two saves in sequence: both answer 200, the last writer wins. Recommendation: catch-and-update, not `upsert`, because Prisma may run `upsert` as read-then-write and still raise `P2002` (unverified for this schema); the catch matches `externalSubmission.ts:298`. A rejected key still saves nothing.
5. **Client deadline for save and check (AIB-07).** In `src/api/client.ts`, add `AI_SETTINGS_TIMEOUT_MS = 30_000` next to `SAMPLE_TEST_TIMEOUT_MS` and pass it from `saveAISettings` and `checkAISettings`. 30 s covers two lookups of 10 s plus 10 s for the write, the re-offer and `readSettings`. Update the comment at `client.ts:46`; add a comment at `providers/types.ts:35` naming the frontend constant. Keep the existing outcome-uncertain handling unchanged.
6. **Correct the draft data-use summary (AIF-04).** Recommended text for `AI_DATA_SENT_SUMMARY`: "For each new email: sender, subject, Gmail labels and a short preview (up to 1,000 characters). Unless the first check is confident the email is not job-related, also the email text (up to 8,000 characters). Nothing else from your mailbox is sent." Then run the frontend `npm run sync-contracts`. Do **not** change any `disclosure.version` or `reviewedOn`: the owner approves the text and the version is bumped in AI-15 (OD-09).

## 5. Out of scope

- AIB-08: remove or switch taking effect only at job end; revision reset on re-create.
- AIF-16: sample-test deadline edge cases (routed to the roadmap's "Not now").
- Server-side abort, parallel lookups, or moving the re-offer after the response (optional extras in AIB-07).
- Any catalog status change, `disclosure.version` bump, `reviewedOn`, live error-mapping confirmation or certification (AI-15, OD-09).
- Zero-provider empty state on `/ai` (release track).
- UI recovery and status copy (AI-20), including a cause-specific message for the per-user pause. That needs no contract change later: `safetyLimit.callsPerDay` is 0 only for the global pause.
- A bound for `INVALID_OUTPUT` (adjacent point in AIB-05; it is user-approvable one email at a time).
- A delete that races a save on an existing row (`P2025` → 500, noted in TEST-07). It needs a remove during an in-flight save and loses no data.
- New endpoints, buttons or scripts for resuming (closeout non-goal: no new endpoints).

## 6. Likely files and components

- Backend: `prisma/schema.prisma` (`AIAccessIssue`); new `prisma/migrations/<timestamp after 20261002150000>_ai_access_paused/migration.sql`; `src/services/ai/operations.ts` (`recordFailure`); `src/services/ai/usage.ts` (constants, `reserveUserCall`, `noteProviderFailure`); `src/services/ai/access.ts` (`deriveAccess`); `src/services/ai/settings.ts` (`saveSettings`: `P2002` catch, clearing a pause); `src/services/ai/providers/gemini.ts` (`createGeminiClient`); `src/services/ai/providers/types.ts` (comment); `src/contracts/aiCatalog.ts`; `package.json`; `package-lock.json`; `STABILIZATION.md`; `scripts/verify-ai-migration.cjs` (excluded list, §4 item 1.8; enum option of OD-14 only).
- Backend tests: `src/tests/ai-user-limits.test.ts`, `ai-access.test.ts`, `ai-settings.test.ts`, `ai-providers-gemini.test.ts`, new `ai-providers-gemini-sdk.test.ts`.
- Frontend: `src/api/client.ts`; `src/contracts/aiCatalog.ts` (by sync only); `src/tests/processing-refresh.test.tsx`.
- Docs: `docs/planning/byo-ai/README.md`, `docs/architecture/ai-capability-architecture.md`.

## 7. Implementation notes

- **Data model and migration.** One statement: `ALTER TYPE "AIAccessIssue" ADD VALUE 'PAUSED';` (the type was created at `20261002120000_user_provided_ai/migration.sql:6`). PostgreSQL 15 allows this inside the migration transaction because the value is not used there. Write the file by hand; never run `prisma migrate dev` or `db:reset` against a test database. Apply only with `guarded-migrate.cjs`. The migration count goes from 15 to 16. No new table or column. Update the enum's doc comment (`schema.prisma:361-363`) to list `PAUSED` as limited, with no cooldown.
- **Lane hazard (2026-10-02).** `upgradeLane` in `verify-ai-migration.cjs` copies every migration except `NEW_MIGRATION` (`:17`, `20261002120000_user_provided_ai`) into a legacy folder and deploys it (`:114-119`). This ticket's migration sorts after `20261002150000`, so it lands in that folder. But the excluded migration creates `AIAccessIssue` (`migration.sql:6`). So the legacy deploy would run `ALTER TYPE` on a type that does not exist yet, and the lane fails before seeding. Fix: §4 item 1.8. The other two lanes keep `user_provided_ai` in their legacy set, before this migration, so the `ALTER TYPE` succeeds there:
  - `verify-mcp-migration.cjs` excludes only `20261002150000_automation_submissions` (`:18`, `:174-175`). After this ticket that is no longer the newest migration, so its upgrade applies it after a newer one. That is harmless: it does not touch `AIAccessIssue` or `ai_configurations`, and the `ai_configurations` snapshot (`:158`) is taken after the legacy deploy. No change.
  - `verify-migration-preservation.cjs` excludes only `20261002090000_add_user_status_revision` (`:17`, `:129-130`) and snapshots no AI configuration table. No change. Its fresh lane will report `migrationsApplied: 16` (`:60-62`; reported, not asserted).
  - With a ledger-only pause (OD-14), there is no migration and none of this applies.
- **Counting query.** `ai_operations` has no `userId`; filter through the email relation: `status: 'FAILED'`, `errorCode: 'INVALID_REQUEST'`, `provider`, `model`, `startedAt >= now − window`, `email: { userId }`, `distinct: ['emailId']`. Volume per user is small; no new index (fine for one user, unverified at scale).
- **Prisma NULL pitfall.** `accessIssue: { not: 'PAUSED' }` also excludes `NULL` rows. Use `OR: [{ accessIssue: null }, { accessIssue: { not: 'PAUSED' } }]` in the cooldown guard.
- **API and shared contracts.** `AIAccessReasonSchema` already has `PAUSED` (`src/contracts/ai.ts:24`), so the API shape is unchanged. The only shared-contract change is the summary string; frontend `npm run sync-contracts` reports 1 file changed, then 0. No new error code: the concurrent save now answers 200.
- **Background jobs.** No queue change. A paused user's jobs end as waiting (email `PENDING`, delivery withdrawn, no attempt used).
- **Operator resume procedure** (goes into `STABILIZATION.md` "Recovery rules"; one user, one transaction, after the code fix is deployed): (a) select that user's `ai_operations` with `status = 'FAILED'` and `errorCode = 'INVALID_REQUEST'` (join `emails` on `emailId`); (b) set them to `RETRYABLE`, `errorCode` and `retryAfter` to null, `attempts − 1` (as a refusal releases a claim, plan §3.8); (c) set those emails from `FAILED` to `PENDING` and clear `processingErrorCategory`, `processingErrorDetails`, `processingErrorStage`, `processingRetryable`, `processingFailedAt` (`schema.prisma:97-102`); (d) set `accessIssue` and `accessIssueModel` to null where `accessIssue = 'PAUSED'`; (e) the user runs a Gmail sync (`gmailSync.ts:240`) or "Check again" (re-offers only on `VERIFIED`, `settings.ts:280`) to re-offer up to 100 waiting emails. Per `STABILIZATION.md:138`, record that duplicate cost was considered: the provider rejected these requests, so no result exists; whether a provider bills a rejected 400 is unverified.
- **Compatibility.** An older frontend already handles `PAUSED`. The new client deadline is only longer. Existing dev consents keep `gemini-draft-2026-10` and stay "current" until AI-15 bumps the version. No production user can have consented: all providers are `hidden`, so none is offered in production.
- **Rollback.** Before reverting code, clear every stored pause (`accessIssue = 'PAUSED'` → null). An older Prisma client is expected to fail reading an unknown enum value (unverified). The enum value can stay in the database unused. The SDK pin, Gemini options, deadline and wording revert as single-line changes (re-run `sync-contracts`).

## 8. Dependencies

- Phase 0 gate passed and recorded (MV-16), with git restored (MV-03) so this lands as a focused commit.
- OD-09: wording only here; approval and version bump in AI-15.
- No dependency on AI-20 (next in order). AI-20 depends on this ticket: its item 22 follows the pause storage chosen in §4 item 1.

## 9. Security and privacy

- The pause is per user: counts filter by the email's `userId`, and writes are scoped by `userId` and `revision`. One user's 400s never pause another user.
- The new log event and every test assertion carry no key, email content or provider text. The real-SDK test puts a sentinel in the error body and asserts it never appears in the `ProviderFailure` (string, inspect, stack) or logs.
- The real-SDK test stubs global `fetch` (`index.cjs:13899` calls it) and uses a fixture key. No test reaches the network.
- Pinning Vertex mode off removes a path where host configuration changes request routing. It is a robustness fix, not a leak fix: the base URL already holds.
- The concurrent-save fix still seals the request's key for its own user and never logs it; the replaced ciphertext is overwritten, not kept.
- The corrected summary must not understate what is sent. It stays a draft until the owner approves it (OD-09).
- The operator procedure is scoped to one user and one error code; it is not a blanket reset.

## 10. Acceptance criteria

- [ ] One `INVALID_REQUEST` fails only that email; access stays `READY`; the next email is processed.
- [ ] Three `INVALID_REQUEST` outcomes on distinct emails for the same user, provider and model within 30 minutes make `GET /api/ai/settings` return `LIMITED` / `PAUSED` with `resumesAt: null`; after that no call is claimed for that user and new emails stay `PENDING` with no attempt used.
- [ ] Failures on different models, outside the window, or for another user do not pause anyone.
- [ ] A later success, rate limit, unknown outcome or "Check again" does not lift the pause; a job holding an older revision cannot write it.
- [ ] A successful save lifts the pause and re-offers waiting emails.
- [ ] The operator procedure in `STABILIZATION.md` lifts the pause and re-opens the user's `INVALID_REQUEST` emails. A test runs the same steps; with the provider fake now succeeding, each re-opened email completes, and another user's rows are unchanged.
- [ ] `src/contracts/ai.ts` and frontend labels are unchanged.
- [ ] The Gemini client is built with `vertexai: false` and `enterprise: false`; with both env flags set to `true`, the real-SDK test still sees Gemini API request paths.
- [ ] `package.json` and the `package-lock.json` root entry say `"2.22.0"`; `npm ci` succeeds; `npm ls @google/genai` shows 2.22.0.
- [ ] Through the SDK's own error path, 400 `API_KEY_INVALID` maps to `KEY_REJECTED`, 429 with `RetryInfo` to `RATE_LIMITED` with `retryAfterMs`, other 400 to `INVALID_REQUEST`, with exactly one `fetch` per call.
- [ ] Two overlapping first-time `PUT /api/ai/settings` both return 200; one row exists with `revision` 1; the test fails on the pre-fix code.
- [ ] Save and "Check again" time out at 30 s; other requests keep 15 s; a timeout still shows the existing outcome-uncertain messages.
- [ ] `AI_DATA_SENT_SUMMARY` states that the text is sent unless the first check is confident the email is not job-related; backend and frontend contracts are identical; no `disclosure.version` or `reviewedOn` changed.
- [ ] The migration applies with `guarded-migrate.cjs`, and the three migration lanes pass.
- [ ] (2026-10-02) `verify-ai-migration.cjs` excludes both `20261002120000_user_provided_ai` and this ticket's migration from its legacy schema; its upgrade applies both and finds `PAUSED` in `AIAccessIssue`. The MCP and Sprint 6 lane scripts are unchanged and pass. Not applicable if OD-14 picks a ledger-only pause.

## 11. Testing

Tests to add or change:
- `src/tests/ai-user-limits.test.ts` ("refusals and unusable outcomes", `:145`): single failure does not pause; three distinct emails pause and the next job makes no provider call; different model, outside window or other user does not pause; stale revision cannot pause; later success, rate limit or unknown outcome keeps the pause; the operator steps (as raw SQL through `prisma.$executeRaw`) resume processing.
- `src/tests/ai-access.test.ts` ("access state precedence", `:26`): `deriveAccess` with a stored `PAUSED` returns `LIMITED`/`PAUSED`, `resumesAt` null; it wins over an active cooldown and the safety limit; `PROVIDER_UNSUPPORTED` still wins over it. `resolveAIAccess` throws `AIAccessError('PAUSED')` without decrypting.
- `src/tests/ai-settings.test.ts`: save clears the pause and re-offers; check keeps it and re-offers nothing; concurrent first-time double PUT with a barrier in `verifyModels` so both reads happen before either create (send both with `Promise.all`; supertest requests are lazy).
- `src/tests/ai-providers-gemini.test.ts:89-92`: expect `vertexai: false, enterprise: false`.
- New `src/tests/ai-providers-gemini-sdk.test.ts` (no `vi.mock('@google/genai')`; `vi.stubGlobal('fetch', …)` returning `Response` objects with `content-type: application/json`; `vi.stubEnv` for the two flags): the mappings in §10, one `fetch` per call, sentinel absent, request URL under `https://generativelanguage.googleapis.com/v1beta/` with no `v1beta1` or `publishers/` (exact path to confirm when writing the test).
- `src/tests/email-worker-reliability.test.ts:219` stays unchanged (one `INVALID_REQUEST` is still operator-only).
- `scripts/verify-ai-migration.cjs` (2026-10-02, enum option only): the excluded list and the `PAUSED` assertion (§4 item 1.8). Run the AI lane once with the new migration but before the list change, to see the legacy deploy fail; recreate both databases; then run it with the change, to see it pass.
- Frontend `src/tests/processing-refresh.test.tsx` ("bounded API client", `:110-122`): save and check are still pending after `REQUEST_TIMEOUT_MS`, then fail with kind `timeout` at `AI_SETTINGS_TIMEOUT_MS`. `src/tests/ai-settings.test.tsx:163-168` must still pass unchanged.

Commands — backend (`career-companion-backend-main`):
```bash
npm install --save-exact @google/genai@2.22.0   # then confirm only the root lockfile line changed
npm ci && npm ls @google/genai
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/ai-user-limits.test.ts src/tests/ai-access.test.ts src/tests/ai-settings.test.ts src/tests/ai-providers-gemini.test.ts src/tests/ai-providers-gemini-sdk.test.ts src/tests/email-worker-reliability.test.ts
npm test
# each with two fresh empty guarded DBs, recreated between scripts
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-mcp-migration.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-migration-preservation.cjs
```

Commands — frontend (`career-companion-frontend-main`):
```bash
npm run sync-contracts   # 1 file changed (aiCatalog.ts); run again: 0 changed
npx vitest run src/tests/processing-refresh.test.tsx src/tests/ai-settings.test.tsx
npm run typecheck && npm run lint && npm test && npm run build
```

Smoke (consent text and save are browser-visible; the smoke saves AI settings at `scripts/smoke-stabilization.mjs:839`): migrate an empty smoke DB with `TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs`, then from the frontend `SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs`. Expect PASS, residue 0. No real key is used; fixture results are not live evidence.

## 12. Documentation updates

- Backend `STABILIZATION.md` "Recovery rules" (`:132-141`): the pause rule, the user resume (save) and the operator procedure from §7.
- `docs/planning/byo-ai/README.md`: one dated line under the §3.8 outcome table (INVALID_REQUEST now pauses the user after 3 distinct emails in 30 minutes); `:550` data-use wording; `:876` add `@google/genai` 2.22.0 to the pinned list.
- `docs/architecture/ai-capability-architecture.md:132`: one-line wording correction.
- The BYO status table is updated by S6-C02; the approval and new `disclosure.version` are recorded by AI-15.

## 13. Definition of done

- Every acceptance criterion in §10 is met, with evidence linked.
- Backend and frontend tests, typecheck, lint (0 errors), build, contract sync and the three migration lanes are green on the new PC (in CI once S7-01 exists).
- The behavior is verified by the tests above and the smoke; nothing is called live evidence.
- The storage decision in §4 item 1 (OD-14) is recorded in the roadmap and the execution report; the docs in §12 are updated.
- One focused commit per repo (or one per item), with evidence recorded in the closeout execution report (`../sprint-6/closeout/execution-report.md`, created when the closeout runs).
