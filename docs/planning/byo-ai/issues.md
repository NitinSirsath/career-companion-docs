# User-provided AI — Linear-ready issue breakdown

> **Roadmap update (2026-10-02):**
> - **AI-15 is widened to Gemini first.** The new body is [AI-15 — Certify and enable Gemini first](AI-15-certify-gemini-first.md) (Sprint 8). It supersedes the AI-15 section below. Gemini certification had no owning ticket, and the plan rule "Gemini supported before AI-09" was not met.
> - **New tickets:** [AI-19 — runtime safety fixes](AI-19-runtime-safety-fixes.md) and [AI-20 — recovery path and status UI](AI-20-recovery-path-and-status-ui.md), both in the [Sprint 6 closeout](../sprint-6/closeout/README.md).
> - **AI-17 is not "Docs done":** several docs still describe the hosted Gemini key ([S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md)).
> - **AI-00** criteria that need git (O7 repository identity) and O8 are recorded after the [migration](../migration-verification/README.md).
> - **AI-18** must also update `scripts/verify-ai-migration.cjs`, which still inserts into and snapshots `ai_call_budgets`.
> - All BYO AI evidence was measured on the old laptop; it is re-run in Phase 0.

**Status:** local planning, 2026-10-02. IDs AI-00 to AI-18 are local; no Linear identity, state or estimate is assumed.

**Source of requirements:** [implementation plan](README.md) (section numbers below refer to it), [ADR-0001](../../architecture/decisions/ADR-0001-user-provided-ai.md) and [AI capability architecture](../../architecture/ai-capability-architecture.md).

**Conventions for every issue**
- **Paths:** `backend/` is `career-companion-backend`; `frontend/` is `career-companion-frontend`.
- **Migrations:** only through `node scripts/guarded-migrate.cjs`.
- **Tests:** only against guarded `career_companion_*test` databases.
- **Keys:** no real provider keys in any committed file or automated test.
- **Every issue's definition of done includes:**
  - backend and frontend typecheck, lint, tests and build pass;
  - `npm run sync-contracts` when contracts change;
  - no key or email content in logs, responses or stored errors.

## Status (2026-10-02)

| Issue | Status |
| --- | --- |
| AI-00 Gate | **Done.** S6-R06 implemented and verified as part of the gate. No git, per the owner's instruction; the current filesystem is the baseline. |
| AI-01 Seam | **Done** (`verifyModels` moved into AI-04) |
| AI-02 Evaluation | **Done in code** (dataset, scorer, runner, CI tests). Gemini baseline run **pending a real key.** |
| AI-03 Catalog | **Done.** All providers `hidden` until certified. |
| AI-04 Failures | **Done in code.** Live Gemini mapping confirmation **pending a real key.** |
| AI-05 Schema and credentials | **Done**, with fresh and upgrade lanes verified |
| AI-06 Per-user ledger | **Done**, including access from the user's configuration (folded in from AI-09) |
| AI-07 Settings API | **Done** |
| AI-08 Sample test | **Done** |
| AI-09 Cutover | **Done** |
| AI-10 Approved retry | **Done** |
| AI-11 AI page | **Done** |
| AI-12 Status across app | **Done** |
| AI-13 OpenAI adapter | **Done** (hidden) |
| AI-14 Anthropic adapter | **Done** (hidden) |
| AI-15 Certify | **Pending:** real keys, live runs, owner approval of the data-use text |
| AI-16 Security and smoke | **Done** |
| AI-17 Docs and release | **Docs done.** Release run pending AI-15. |
| AI-18 Cleanup | After one stable release |

## Milestones

| Milestone | Issues | Outcome |
| --- | --- | --- |
| M0 Gate | AI-00 | Safe starting point |
| M1 Provider-neutral seam | AI-01, AI-02 | Same behavior through an adapter; Gemini baseline measured |
| M2 Foundations | AI-03, AI-04, AI-05 | Catalog, failure vocabulary, schema and encryption ready |
| M3 Per-user AI access (backend) | AI-06 to AI-10 | Users' own keys drive processing; waiting, refusal and approval rules work |
| M4 Frontend | AI-11, AI-12 | Users can set up, fix and understand their AI |
| M5 Providers | AI-13, AI-14, AI-15 | OpenAI and Claude adapters, certified and enabled |
| M6 Verify and release | AI-16, AI-17, AI-18 | End-to-end evidence, documentation, release, cleanup |

---

## AI-00 — Confirm the BYO AI entry gate and record the baseline

**Goal:** start implementation from a known, committed baseline with the privacy prerequisite in place.

**Technical context**
- The working copies inspected for this plan have no `.git`.
- S6-R06 (raw worker errors stored in `pgboss.job.output`) is open.
- ADR-0001 requires both before user keys exist.

**Scope**
1. Identify the authoritative backend, frontend and docs repositories and default branches (O7).
2. Confirm Sprint 6 is committed. Record commit hashes for all three repositories.
3. Confirm S6-R06 is merged and its acceptance criteria hold:
   - `pgboss.job.output` holds only the safe category;
   - an outcome-unknown error is stored as not retryable.
4. Record which production users rely on the hosted Gemini key today (O8), and how they will be told.
5. Record the owner's decision on ordering against the MCP feature (plan §15).

**Out of scope:** any code change, including S6-R06 itself (it is a Sprint 6 ticket).

**Dependencies:** Sprint 6 completion; S6-R06.

**Acceptance criteria**
- [ ] Repository URLs and baseline hashes are recorded in this file's header or the plan's status table.
- [ ] S6-R06 is shown as complete, with a link to its verification.
- [ ] O8 answer and the MCP-feature ordering are recorded.

**Testing requirements:** none new. Link S6-R06's real-queue test result.

**Relevant documentation:** plan §10, §13, §15; [S6-R06](../sprint-6/review-2026-10-02/S6-R06-worker-error-hygiene.md).

**Risks / notes:** if S6-R06 is not done, stop. Multiple provider SDKs would put raw errors in the database.

---

## AI-01 — Provider-neutral AI contracts and Gemini adapter seam (no behavior change)

**Goal:** the pipeline gets its AI capabilities from a provider-neutral layer, with today's Gemini behavior unchanged.

**Technical context**
- `backend/src/services/ai/gemini/GeminiProvider.ts` holds the prompts, a hand-written Gemini schema, env key and model handling, and error mapping.
- `pipeline.ts` calls `GeminiProvider.getInstance()`.
- Tests (`ai.test.ts`, `ai-idempotency.test.ts`, `email-worker-reliability.test.ts`, `application-status.test.ts`) and `frontend/scripts/smoke-stabilization.mjs` patch `getInstance`.

**Scope**
1. **`contracts.ts`:** add `AI_CONTRACTS` (plan §3.2).
   - Move both prompts **byte-identically**.
   - Keep `classification/v2` and `extraction/v2` and the Zod schemas.
   - Move input bounds (512 / 1000 / 30 / 1000 / 8000) and `maxOutputTokens` 2048 into the contract.
2. **`providers/types.ts`:** `ProviderClient`, `StructuredRequest`, `StructuredResponse`, `VerifyResult`, `ProviderFailure` (plan §3.3). For now `ProviderFailure` maps onto the existing error classes so behavior is identical.
3. **`providers/jsonSchema.ts`:** Zod → JSON Schema with `z.toJSONSchema`, plus the Gemini dialect.
4. **`providers/gemini.ts`:**
   - built from `(provider, apiKey)`;
   - same SDK options: timeout 30 s, retry attempts 1, temperature 0.1, `thinkingBudget: 0`, `maxOutputTokens` 2048;
   - same error mapping as today;
   - returns token usage;
   - `verifyModels` via `models.get`.
5. **`providers/index.ts`:** `createProviderClient(provider, apiKey)`. This is the test seam.
6. **`capabilities.ts`:** `bindCapabilities(client, provider, models)` implements `RelevanceClassifier` and `EmailAnalyzer`, returning `{ version, data, model, usage }`.
7. **Transitional `access.ts`:** `resolveAIAccess(userId)` returns Gemini capabilities from `GEMINI_API_KEY` and the env models. The pipeline uses it instead of the singleton. Provenance values are unchanged.
8. Delete `GeminiProvider.ts` and the hand-written schema. Update `services/ai/index.ts` exports.
9. **Tests:** move every `GeminiProvider.getInstance` spy or mock to `createProviderClient`. Add `src/tests/helpers/fakeProviderClient.ts`.
10. **Smoke:** replace the `getInstance` patch with a patched `createProviderClient` that returns the in-script fake client.

**Out of scope:** catalog, new failure kinds, database changes, per-user anything, other providers.

**Dependencies:** AI-00.

**Acceptance criteria**
- [ ] Prompt strings are byte-identical to the pre-change text (test with the old strings as literals).
- [ ] A contract fingerprint fixture exists for both versions.
- [ ] The derived Gemini schema is equivalent to the deleted hand-written schema (same properties, types, enums, required and nullable). Any keyword difference is listed in the test and stripped by the adapter.
- [ ] All existing backend tests pass with only seam changes. `ai-idempotency` cases keep their assertions.
- [ ] The smoke passes with the new seam. Outbound blocking still proves no provider call.
- [ ] No production code references `GeminiProvider`.

**Testing requirements**
- New `ai-contracts.test.ts`.
- New `ai-providers-gemini.test.ts`: request options, usage, error mapping parity with today's table.
- Existing suites; smoke.

**Relevant documentation:** plan §3.1–§3.4; architecture §5.1–§5.2, §14 step 1.

**Risks / notes**
- Gemini schema keywords such as `minimum`/`maximum` from Zod may not be accepted. Strip them in the dialect and keep validation in Zod.
- Do not change models, prompts or limits here.

---

## AI-02 — Synthetic evaluation set, runner and Gemini baseline

**Goal:** a repeatable, synthetic way to measure a provider/model pair, and a recorded baseline for today's Gemini models.

**Technical context:** the evaluation gate (architecture §5.4, plan §7). No real Gmail data. Live runs are manual.

**Scope**
1. **`backend/eval/ai/dataset/*.json`:** about 40 cases covering the plan §7.1 table.
   - Fictitious content.
   - Reserved domains only (`example.com`, `example.org`, `example.net`, `*.test`).
   - Case 1 is the same email as `sampleEmail.ts` (create that file here).
2. **`eval/ai/score.ts`:** relevance, category, key-field, hallucination, injection and schema-validity metrics, with normalized date and text matching.
3. **`eval/ai/run.ts`** and the npm script `ai:eval`:
   - flags `--provider --fast --detailed --runs`;
   - key from `AI_EVAL_API_KEY`;
   - calls `createProviderClient` and `bindCapabilities` directly;
   - no database, no ledger;
   - refuses to start under `NODE_ENV=production`;
   - writes `eval/ai/reports/<date>_<provider>_<fast>_<detailed>.json` and prints a summary.
4. **Supervised baseline run:** `gemini-2.5-flash-lite` (fast) and `gemini-2.5-flash` (detailed), 2 runs.
5. **Set final thresholds** from the baseline under the plan §7.3 rules. Create `docs/ai/provider-evaluation.md` with the method, thresholds and the Gemini result.

**Out of scope:** prompt changes, CI network calls, dashboards, automatic scheduling of evaluations.

**Dependencies:** AI-01.

**Acceptance criteria**
- [ ] Dataset passes the shape test and the reserved-domain test in CI.
- [ ] Scorer unit tests cover each metric, including a false negative on an interview case.
- [ ] Baseline report committed. `provider-evaluation.md` records the thresholds and the Gemini result, with SDK version and contract versions.
- [ ] Any metric where Gemini misses a starting floor is reported to the owner.

**Testing requirements:** `eval/ai/*.test.ts` (no network); one supervised live run.

**Relevant documentation:** plan §7; architecture §5.4, O1.

**Risks / notes**
- Keep the dataset small and readable. Every case should teach something.
- The live key stays in the shell only and is never written to an `.env` file.

---

## AI-03 — Code-defined provider and model catalog

**Goal:** a single code catalog of supported providers and models, available to backend and frontend.

**Technical context:** contract files in `backend/src/contracts/*.ts` are copied byte-for-byte to the frontend by `npm run sync-contracts` (top-level files only; it does not delete stale files).

**Scope**
1. **`backend/src/contracts/aiCatalog.ts`** with the plan §3.5 types and helpers:
   - `getProvider(id)`;
   - `supportedProviders()`;
   - `recommendedModel(provider, role)`;
   - `resolveModel(provider, role, configuredId, now)`, which returns `{ model, source }` with the retired fallback.
2. **Gemini entry:**
   - protocol `gemini`, fixed base URL, `FREE_TIER_AVAILABLE`;
   - links, setup steps and a draft disclosure;
   - the two baseline models with roles, recommended flags, temperature 0.1, reasoning `thinkingBudget: 0`, timeout 30 s and `retiresOn` if announced;
   - `status: 'supported'` only after AI-02 and AI-04 evidence exists, otherwise `hidden`.
3. Export from `contracts/index.ts`.
4. Make the transitional `resolveAIAccess` take models from the catalog. Keep the env overrides until AI-09, to preserve behavior.

**Out of scope:** OpenAI and Anthropic entries (AI-13, AI-14), API endpoints, database.

**Dependencies:** AI-01 (AI-02 and AI-04 for the `supported` flag).

**Acceptance criteria**
- [ ] Catalog tests (plan §9) pass:
  - unique IDs;
  - exactly one recommended model per role;
  - recommended models support their role;
  - supported entries have `evaluation`;
  - https links;
  - no supported model past `retiresOn`.
- [ ] The frontend copy is identical after sync.
- [ ] No catalog value is read from the environment.

**Testing requirements:** `ai-catalog.test.ts`.

**Relevant documentation:** plan §3.5; architecture §5.3.

**Risks / notes:** the disclosure text is a draft until the owner signs it off (plan §15).

---

## AI-04 — Provider failure normalization and Gemini error mapping

**Goal:** every provider failure becomes one of Career Companion's failure kinds, with a confirmed Gemini mapping.

**Technical context**
- Today: 429/503 → `RetryableAIError`; other 4xx → `TerminalAIError`; everything else → outcome unknown.
- The new vocabulary is plan §3.3 and §3.6.

**Scope**
1. Final `FailureKind` set and `ProviderFailure`:
   - fixed message per kind;
   - optional `status`, allowlisted `providerCode` and `retryAfterMs`;
   - **no `cause`**.
2. `classifyGeminiError(err)` from the plan §3.6 Gemini column, including `retry-after` / `RetryInfo` parsing.
3. Response-level `INVALID_OUTPUT` detection: empty text, non-JSON, schema-invalid.
4. **errors.ts:**
   - `AIAccessError(reason, resumesAt?)`;
   - `AIOutcomeUnknownError extends TerminalAIError`.

   They are not thrown by the ledger yet; AI-06 wires them.
5. **Supervised live confirmation with a real Gemini key and no email data:**
   - invalid key;
   - key without access to a model (or a non-existent model ID);
   - rate limit by burst on the free tier;
   - region or billing error if one can be produced.

   Record status and codes in `docs/ai/provider-evaluation.md`.

**Out of scope:** ledger behavior changes, other providers.

**Dependencies:** AI-01.

**Acceptance criteria**
- [ ] A table-driven test covers every Gemini row, including unknown → `OUTCOME_UNKNOWN`.
- [ ] A thrown `ProviderFailure` passed through `JSON.stringify` and `util.inspect` contains no text from the original error (sentinel test).
- [ ] Live confirmation is recorded. Any row that could not be produced is marked "not confirmed".

**Testing requirements:** `ai-providers-gemini.test.ts` extended; `ai-errors.test.ts`.

**Relevant documentation:** plan §3.3, §3.6; architecture §9.3, O2.

**Risks / notes:** unconfirmed rows default to `OUTCOME_UNKNOWN`, which is the conservative choice.

---

## AI-05 — AI configuration schema and credential encryption

**Goal:** persistence for per-user configuration and usage, and a sealed credential format.

**Technical context**
- `utils/gmailTokenEncryption.ts` already has AES-256-GCM `encrypt` / `decrypt(…, key)`.
- Migrations are additive and guarded.
- Plan §4 lists every column and why it exists.

**Scope**
1. **Prisma and migration** `<timestamp>_user_provided_ai` (after the MCP feature's migrations if it lands first):
   - `ai_configurations` (plan §4.1) and enum `AIAccessIssue`;
   - `ai_usage_days` (§4.2);
   - `ai_operations.provider`, `.model`, `.approvedRetries` (§4.3);
   - CHECK constraints;
   - backfill `provider = 'gemini' WHERE attempts > 0`;
   - `User` relations.
2. **`gmailTokenEncryption.ts`:** optional `aad` parameter on `encrypt` and `decrypt`. Existing call sites are unchanged.
3. **`services/ai/credentials.ts`:**
   - `loadAICredentialKey()` (from `AI_CREDENTIAL_ENCRYPTION_KEY`, 64 hex);
   - `sealApiKey(userId, key)` → `v1:<iv>:<ct>`;
   - `openApiKey(userId, sealed)`;
   - AAD `ai-credential:v1:<userId>`.
4. **`utils/config.ts` (production):** require the key, valid, and different from `GMAIL_TOKEN_ENCRYPTION_KEY`.
5. Add fixture keys to `.env.test`, `.env.smoke.test` and `.env.example` (empty in the example).

**Out of scope:** reading or writing these tables from features (AI-06, AI-07), dropping `ai_call_budgets` (AI-18).

**Dependencies:** AI-00. Can run alongside AI-01 to AI-04.

**Acceptance criteria**
- [ ] Fresh and upgrade migration lanes pass `verify-migration-preservation.cjs`. Existing rows are unchanged apart from the documented `provider` backfill.
- [ ] Deleting a user cascades to the configuration and usage rows.
- [ ] Credential tests:
  - round trip;
  - wrong user ID (AAD) fails;
  - tampered or truncated input fails;
  - missing or invalid env key gives a clear error with no value;
  - ciphertext does not contain the plaintext.
- [ ] Production config tests cover the new checks.

**Testing requirements:** `ai-credentials.test.ts`; `db.test.ts` (constraints, cascade); config tests.

**Relevant documentation:** plan §4, §8; architecture §7.

**Risks / notes:** losing the AI key makes stored keys unreadable. Document a backup before release (AI-17).

---

## AI-06 — Per-user safety limit, usage counts, cooldown and refusal rule in the operation ledger

**Goal:** the ledger enforces per-user limits and cooldowns, does not charge refusals, and records provenance and tokens.

**Technical context:** `services/ai/operations.ts` `runOperation`:
- claim → global `AICallBudget` budget and cooldown → call → checkpoint;
- `MAX_ATTEMPTS = 3`;
- 429/503 consume attempts and set a global cooldown.

**Scope** (plan §3.8, §3.9):
1. **`usage.ts`:**
   - `reserveUserCall(tx, userId, now)`: configuration cooldown check, kill switch, upsert usage day, conditional increment;
   - `recordTokens(userId, day, usage)`;
   - access-state writers guarded by `revision`.
2. **`runOperation({ userId, emailId, contract, access, invoke })`:**
   - compare-and-set claim on `attempts`;
   - held check `attempts ≥ MAX_ATTEMPTS + approvedRetries`;
   - `provider` / `model` written at claim;
   - outcome handling per the plan §3.8 table (release and attempt restore on refusal; cooldown growth and reset; `UNKNOWN` plus short cooldown; `FAILED` with `errorCode` `INVALID_OUTPUT` or `INVALID_REQUEST`);
   - returns `{ data, provider, model }`.
3. **`AI_USER_DAILY_CALL_LIMIT`** (default 500, 0–5000). The production config rejects `AI_DAILY_CALL_LIMIT`.
4. **Pipeline:** passes `access` and the contract. `AIProcessingResult.provider` / `model` come from the operation that produced each part (classification on upsert; extraction on update).
5. Stop reading and writing `ai_call_budgets` (the table stays until AI-18).
6. **Interim access:** env-key access still resolves (AI-09 switches it), with a synthetic `revision` and per-user rows keyed to the job's user.

**Out of scope:** settings API, worker `PENDING` handling (AI-09), approval endpoint (AI-10).

**Dependencies:** AI-04, AI-05.

**Acceptance criteria**
- [ ] User A at the limit does not block user B. A cooldown for A does not affect B.
- [ ] A refusal leaves the operation claimable with its attempts unchanged.
- [ ] `RATE_LIMITED` uses `retry-after` when given, bounded to 10 s–1 h. Otherwise it grows 60 s → 30 min and resets after a success.
- [ ] An unknown outcome is held (`UNKNOWN`) and sets a 2-minute cooldown for that user only.
- [ ] A stale revision cannot change configuration state.
- [ ] Tokens are recorded for successful and invalid-output responses.
- [ ] Every existing `ai-idempotency` guarantee still holds: reuse, one concurrent call, no replay after checkpoint failure, no replay of unknown, ownership check, legacy adoption, deterministic filter.
- [ ] Limit 0 makes no call.

**Testing requirements:**
- `ai-user-limits.test.ts` (new, database);
- `ai-idempotency.test.ts` adapted;
- concurrency tests with two users and with two claims on one operation.

**Relevant documentation:** plan §3.8, §3.9; architecture §9.

**Risks / notes:** keep the checkpoint write outside the catch. That is the existing protection against replay after a persistence failure.

---

## AI-07 — AI settings API: read, save with verification, check again, remove

**Goal:** the session user can create, update, switch, verify and remove their AI configuration through the API.

**Technical context**
- Existing conventions: `/api/<resource>` routers, `requireAuth`, zod contracts in `src/contracts`, the `{ error: { code, message } }` envelope, explicit `select` for secrets (see `routes/gmail.ts`).

**Scope** (plan §5):
1. **`src/contracts/ai.ts`:** `AISettingsResponseSchema`, `SaveAISettingsRequestSchema` (strict), `AccessState` and `AccessReason` enums, error detail schemas.
2. **`services/ai/access.ts`:** `getAccessState(userId, now)` (no decryption) with the plan §3.7 precedence.
3. **`services/ai/settings.ts`:**
   - `read`;
   - `save` (verification cap → verify → upsert with `revision + 1`, consent, cleared issue and cooldown);
   - `check`;
   - `remove`.

   Uses `createProviderClient(...).verifyModels` with a 10 s timeout.
4. **`routes/ai.ts`:** `GET`, `PUT`, `DELETE /api/ai/settings`; `POST /api/ai/settings/check`. Mount in `index.ts`.
5. **`reofferPendingEmails(userId)`:**
   - extracted from `gmailSync.ts`;
   - newest first;
   - skipped when not READY;
   - used by sync and by successful save and check.
6. Logging events `ai_settings_saved`, `ai_settings_removed`, `ai_access_checked` (plan §3.15).

**Out of scope:** sample test (AI-08), using user configurations in the worker (AI-09), frontend.

**Dependencies:** AI-03, AI-05, AI-06.

**Acceptance criteria**
- [ ] 401 without a session on every route.
- [ ] **Save:**
  - unknown keys → 400;
  - hidden or unknown provider → 400;
  - model not in the catalog for that role → 400;
  - key missing on a new provider → 400;
  - consent missing or outdated on a new provider → 400;
  - a blank key on the same provider keeps the stored key.
- [ ] Rejected verification → 422 `AI_ACCESS_REJECTED` with reason (and model). The existing configuration is byte-for-byte unchanged.
- [ ] Inconclusive → saved, `verifiedAt` null, state READY.
- [ ] Check:
  - inconclusive never overwrites a known state;
  - verified clears needs-attention but not an active rate-limit cooldown.
- [ ] 21st verification in a UTC day → 429 `AI_VERIFY_RATE_LIMITED`.
- [ ] `DELETE` is idempotent; the row is gone.
- [ ] No response body, log line or validation error contains the sentinel key.
- [ ] Re-offer is newest first, at most 100, and enqueues nothing when access is not READY.

**Testing requirements:**
- `ai-settings.test.ts` (supertest, dev-header auth, fake client);
- `ai-access.test.ts`;
- `gmailSync.test.ts` updated for the re-offer order.

**Relevant documentation:** plan §3.7, §3.11, §3.12, §5; architecture §6, §7.5, §10.1.

**Risks / notes:**
- The save response must never be built by spreading the database row.
- Use an explicit response mapper.

---

## AI-08 — Sample-email test endpoint

**Goal:** the user can prove structured output end to end with their stored configuration, without using their own mail.

**Scope** (plan §3.13):
1. `POST /api/ai/settings/sample-test` with body `{}`.
2. Resolve access. If not READY → 409 `AI_ACCESS_UNAVAILABLE`.
3. Run classification then extraction on `sampleEmail.ts` through `bindCapabilities`. Each call goes through `reserveUserCall` and records tokens.
4. Access refusals update the configuration like real calls → 422 `AI_ACCESS_REJECTED`. Other failures → 502 `AI_SAMPLE_FAILED { kind }`.
5. Return the validated result subset and usage. Persist nothing else; no ledger row.
6. Log `ai_sample_test` (outcome, tokens).

**Out of scope:** testing an unsaved key; user-supplied sample text.

**Dependencies:** AI-07.

**Acceptance criteria**
- [ ] Success adds 2 to `calls` and records tokens. No `ai_operations` or `ai_processing_results` row is created.
- [ ] At the safety limit → 409 with `SAFETY_LIMIT`, and no call.
- [ ] A fake `ACCOUNT_OR_BILLING` failure → 422 and the state becomes NEEDS_ATTENTION.
- [ ] The sentinel key is absent from responses and logs.

**Testing requirements:** extend `ai-settings.test.ts`.

**Relevant documentation:** plan §3.13, §5; architecture §10.2.

**Risks / notes:** the sample costs the user a few tokens. The UI must say so (AI-11).

---

## AI-09 — Cutover to user-provided AI: waiting as PENDING, refusal rule in the worker, hosted key removed

**Goal:** every AI call uses the job user's own configuration. Without usable access, emails wait as `PENDING` without using retries.

**Technical context**
- `processEmailJob` sets `PROCESSING`, runs the pipeline, and on error writes sanitized fields.
- After S6-R06 it throws a sanitized error for non-terminal failures.

**Scope** (plan §3.7, §3.10, §3.16):
1. `resolveAIAccess(userId)` reads `ai_configurations` only:
   - decrypt via `openApiKey`;
   - decryption failure → `KEY_UNREADABLE` plus an `ai_credential_unreadable` log;
   - catalog model resolution.

   Remove the env-key path.
2. **Pipeline:** resolve lazily once per job, after completed-result adoption and the deterministic filter. Unavailable → throw `AIAccessError`.
3. **Worker:**
   - `AIAccessError` → email `PENDING` (unless `COMPLETED`), clear error fields, log `job_waiting_for_ai`, acknowledge;
   - `AIOutcomeUnknownError` → `FAILED`, not retryable, acknowledged.
4. Remove `GEMINI_API_KEY`, `GEMINI_RELEVANCE_MODEL` and `GEMINI_EXTRACTION_MODEL` from code and `.env.*`. Production config rejects `GEMINI_API_KEY`.
5. **Smoke harness:**
   - create sealed fixture configurations for the fixture users;
   - add scenarios: unconfigured user waits; rate-limited user waits while another user processes.

**Out of scope:** approval endpoint (AI-10), frontend.

**Dependencies:** AI-02 and AI-04 (Gemini certified), AI-06, AI-07.

**Acceptance criteria**
- [ ] User without a configuration:
  - email ends `PENDING`;
  - the job is acknowledged (no pg-boss retry);
  - no `ai_operations` row;
  - no body fetch.
- [ ] A promotional email completes without a configuration.
- [ ] A refusal during extraction:
  - email `PENDING`;
  - classification stays `COMPLETED`;
  - extraction attempts unchanged;
  - after a fix and re-offer, only extraction is called.
- [ ] Unknown outcome → `FAILED` with no queue retry. `pgboss.job.output` holds only the safe category.
- [ ] Job payload keys are exactly `userId` and `emailId`.
- [ ] Switching provider mid-job does not change the new configuration's state (revision guard), and the next job uses the new provider.
- [ ] Production startup fails with `GEMINI_API_KEY` set.
- [ ] Smoke passes, including the new scenarios.

**Testing requirements:** `email-worker-reliability.test.ts` (unit and real-queue lanes), `ai-pipeline-integration.test.ts` (new, three users), smoke.

**Relevant documentation:** plan §3.7, §3.10, §3.11; architecture §4.2, §6, §9.4, §9.6.

**Risks / notes:**
- Do not deploy this to production before AI-11 and AI-12. Users would have no way to set up AI.
- Emails flip `PENDING` → `PROCESSING` → `PENDING` briefly. That is acceptable.

---

## AI-10 — User-approved retry for held AI operations

**Goal:** the user can approve exactly one more attempt for an operation whose outcome was uncertain or unusable.

**Technical context:** `POST /api/emails/:id/retry` refuses held operations with 409 `AI_OPERATION_REQUIRES_REVIEW` and never resets claims.

**Scope** (plan §3.14, §5):
1. Approvability rules: which statuses and `errorCode`s, plus `PROCESSING` older than 15 minutes.
2. **Request:** optional body `{ acceptPossibleDuplicateCharge: true }` (strict).
3. **Responses:**
   - approvable without the flag → 409 `AI_RETRY_NEEDS_APPROVAL` with operations (operation, reason, provider, model, attemptedAt) and `currentProvider`;
   - not approvable → existing 409;
   - access not READY → 409 `AI_ACCESS_UNAVAILABLE`.
4. **Approval transaction:**
   - compare-and-set on status and attempts;
   - `RETRYABLE`, `retryAfter` null, `approvedRetries + 1`;
   - log `ai_retry_approved`;
   - enqueue (keep `RETRY_RECENTLY_QUEUED`).
5. **Contract:** add the request and 409 details schemas to `contracts/email.ts` or `contracts/ai.ts`.

**Out of scope:** bulk approval, automatic replay, approving `INVALID_REQUEST` or legacy partial results.

**Dependencies:** AI-09.

**Acceptance criteria**
- [ ] One approval → exactly one more provider call. A second unknown outcome needs a new approval.
- [ ] Two concurrent approvals of the same hold increment `approvedRetries` once.
- [ ] An approval racing a claim cannot produce two calls.
- [ ] A recent `PROCESSING` (under 15 minutes) is not approvable.
- [ ] Owner-only: a foreign email → 404.
- [ ] Completed results and match decisions are never reset (existing guarantees).

**Testing requirements:** `email-worker-reliability.test.ts` retry section; `ai-user-limits.test.ts` approval cases.

**Relevant documentation:** plan §3.14; architecture §9.5; ADR-0001 decision 10.

**Risks / notes:** if the provider changed since the held attempt, the dialog must show both providers (AI-12).

---

## AI-11 — AI provider page: setup, switch, models, status, sample test, remove

**Goal:** a user can choose a provider, paste a key, pick models, consent, save and verify, and see and fix their AI status, all on one page.

**Technical context**
- **Routing:** file-based TanStack Router; the root route guards auth.
- **Navigation:** hard-coded in `Sidebar.tsx` and `MobileNav.tsx`.
- **API client:** `api` singleton in `src/api/client.ts` with `ApiError` (`code`, `outcomeUncertain`).
- **Forms:** react-hook-form with zod.
- **Primitives:** button, badge, dialog, input, label, native-select, tooltip.
- **Design rules:** `DESIGN_SYSTEM.md` and `AI_UI_RULES.md`.

**Scope** (plan §6.1, §6.3, §6.4):
1. Sync contracts. New `src/routes/ai.tsx`. Nav item "AI provider" in both nav components.
2. **Components:**
   - `ProviderPicker` (supported providers only, plus the subscription note);
   - `ProviderSetupForm` (steps and links, paid warning, password key field, recommended models plus Advanced `<details>`, safety-limit note, data-use disclosure plus consent checkbox, Save and verify);
   - `AIStatusPanel` (state, reason, one fix, last checked, waiting count, today's counts labelled as ours, limit; actions);
   - `SampleTestPanel`;
   - `RemoveAIDialog`;
   - `src/lib/aiLabels.ts`.
3. **API methods:** `getAISettings`, `saveAISettings`, `checkAISettings`, `runAISampleTest`, `removeAISettings`.
4. **Key handling** (plan §6.3):
   - direct call, not `useMutation`;
   - clear the field right after reading it and on unmount;
   - never put the key in a query key, query data, URL or storage.
5. **Save outcomes:** verified, inconclusive and rejected messages as in plan §6.1. A rejection leaves the current status shown.

**Out of scope:** notices on other pages, retry dialog, provenance (AI-12), onboarding wizard, a settings area.

**Dependencies:** AI-07, AI-08.

**Acceptance criteria**
- [ ] Not configured → provider cards. Configured → status panel with the correct state and fix for every `AccessReason`.
- [ ] The key input is `type="password"`, never prefilled, empty after success, rejection and unmount.
- [ ] The sentinel key is absent from the TanStack query and mutation caches, `localStorage`, `sessionStorage` and `location`.
- [ ] Consent is required for a new provider. Switching shows the current status until the new one is saved.
- [ ] The Advanced section lists only catalog models for the role. Default = recommended.
- [ ] The sample test shows the result and says it uses a built-in example and costs tokens.
- [ ] Keyboard-only and mobile layout checks pass. Labels are associated. Focus is visible.

**Testing requirements:** `src/tests/ai-settings.test.tsx` (mocked `api`, memory router); update tests that render the nav if snapshots or expectations change.

**Relevant documentation:** plan §6; architecture §3.1–§3.4; `DESIGN_SYSTEM.md`, `AI_UI_RULES.md`.

**Risks / notes:** final wording is a design task (O9). Keep it plain and short.

---

## AI-12 — AI status across the app: waiting notice, Retry anyway, provenance labels

**Goal:** users understand why emails wait, can approve a held retry knowingly, and can see which provider produced each AI result.

**Technical context**
- The Gmail table (`routes/gmail.tsx`) shows raw `processingState` and a "Manual Retry" tooltip button. Every error shows "Retry could not be confirmed".
- The dashboard has unmatched and ambiguous panels.
- The application detail shows "AI interpretation (not verified source text)".

**Scope** (plan §6.2):
1. **`AIAccessNotice`** on the dashboard and the Gmail page (query `['aiSettings']`). Add `aiSettings` to the processing-refresh keys.
2. **Gmail table:**
   - "Waiting for AI" for `PENDING` rows when not READY;
   - plain text for error categories;
   - branch on retry error codes (definitive 409s get their own message).
3. **`RetryAnywayDialog`** on 409 `AI_RETRY_NEEDS_APPROVAL`:
   - earlier provider and model, reason, possible duplicate charge, current provider (highlighted when different);
   - confirm resends with the flag;
   - an uncertain result re-reads and never resends.
4. **Provenance:**
   - **Backend:** add `provider` and `model` to the `aiProcessingResult` subset in the ambiguous and unmatched responses, and `analyzedBy: { provider, model } | null` to `SourceEmailSchema`.
   - **Frontend:** `AnalyzedBy` component; `deterministic` → "Rule-based filter".
5. Update `gmail.test.tsx` mocks (`retryEmail`, `isApiError`) and the dashboard tests.

**Out of scope:** new email states, bulk retry, timeline redesign.

**Dependencies:** AI-09, AI-10, AI-11.

**Acceptance criteria**
- [ ] The notice appears for NOT_SET_UP, NEEDS_ATTENTION and LIMITED with the right text, waiting count and link. It is hidden when READY.
- [ ] The retry flow works: 409 → dialog → confirm sends `acceptPossibleDuplicateCharge: true` → list refreshes. Cancel sends nothing.
- [ ] A network failure on confirm does not resend.
- [ ] Provenance shows on the ambiguous and unmatched cards and on timeline AI interpretations.
- [ ] Contract changes are additive (nullable fields). Existing contract tests pass.

**Testing requirements:** `src/tests/ai-status.test.tsx`; updated `gmail.test.tsx`, dashboard and application detail tests; backend contract tests for the new fields.

**Relevant documentation:** plan §6.2; architecture §3.3, §3.5.

**Risks / notes:** S6-R08 (open) touches the same retry message. If it is not done first, this issue covers the retry-message part for the new codes only.

---

## AI-13 — OpenAI-compatible adapter and OpenAI catalog entry (hidden)

**Goal:** OpenAI can be used through the provider-neutral layer, ready for certification.

**Scope**
1. Add the `openai` SDK, exact pinned version.
2. **`providers/openai.ts`:**
   - client with `baseURL` from the catalog, `maxRetries: 0`, catalog timeout;
   - Chat Completions with `response_format` `json_schema`, `strict: true`, using the strict dialect from `jsonSchema.ts` (all keys required, optional → nullable, `additionalProperties: false`, unsupported keywords stripped);
   - null→absent normalization for optional keys;
   - `max_completion_tokens`;
   - temperature and `reasoning_effort` from the catalog;
   - usage from `usage.prompt_tokens` / `completion_tokens`;
   - `finish_reason: 'length'` and `refusal` → `INVALID_OUTPUT`;
   - `verifyModels` via `models.retrieve`.
3. **`classifyOpenAIError`** from plan §3.6, including 429 `insufficient_quota` → `ACCOUNT_OR_BILLING` and `retry-after` parsing.
4. **Catalog:**
   - OpenAI entry `status: 'hidden'`, `PAID_ONLY`, links, setup steps, draft disclosure;
   - candidate models for each role (chosen with evaluation in AI-15).

**Out of scope:** Kimi, DeepSeek and Mistral entries; the Responses API; enabling the provider.

**Dependencies:** AI-09 (stable seam and ledger); AI-03, AI-04.

**Acceptance criteria**
- [ ] Recorded-response tests: request shape (`baseURL`, strict schema for both contracts, token parameter, no temperature when unsupported, retries 0, timeout), parsing, usage, every error row, every `verifyModels` result.
- [ ] The strict schema for `classification/v2` makes `category` nullable. A `null` from the model validates as absent.
- [ ] The base URL cannot be overridden by any input (test).
- [ ] The provider stays hidden: the API rejects it and the UI does not show it.

**Testing requirements:** `ai-providers-openai.test.ts`; sentinel test for errors.

**Relevant documentation:** plan §3.4, §3.6; architecture §2.2.

**Risks / notes:** the SDK default is 2 retries. The test must assert 0.

---

## AI-14 — Anthropic adapter and Claude catalog entry (hidden)

**Goal:** Claude can be used through the provider-neutral layer, ready for certification.

**Scope**
1. Add `@anthropic-ai/sdk`, exact pinned version.
2. **`providers/anthropic.ts`:**
   - `maxRetries: 0`, catalog timeout;
   - instructions as `system`; input as one user message;
   - `max_tokens`;
   - structured output: native JSON-schema output where the catalog model says `anthropic_native`, otherwise a forced tool call (`tool_choice` = the contract tool, `input_schema` = derived schema), reading the tool input;
   - thinking off;
   - usage from `usage.input_tokens` / `output_tokens`;
   - `stop_reason` `max_tokens` or `refusal` → `INVALID_OUTPUT`;
   - `verifyModels` via `models.retrieve`.
3. **`classifyAnthropicError`** from plan §3.6, including 529 → `RATE_LIMITED`. Billing and credit errors are confirmed in AI-15.
4. **Catalog:** Claude entry `status: 'hidden'`, `PAID_ONLY`, links, setup steps, draft disclosure, candidate models.

**Out of scope:** enabling, prompt caching, batch APIs.

**Dependencies:** AI-13 (shared dialect helpers).

**Acceptance criteria:** the same set as AI-13 for the Anthropic protocol, including both structured-output modes.

**Testing requirements:** `ai-providers-anthropic.test.ts`; sentinel test.

**Relevant documentation:** plan §3.4, §3.6; architecture §2.2.

**Risks / notes:** structured-output support differs by model. The catalog field decides the mode per model, and the evaluation confirms it.

---

## AI-15 — Certify and enable OpenAI and Claude

**Goal:** OpenAI and Claude become officially supported only if they pass the evaluation gate.

**Scope** (plan §7.4), for each provider:
1. **Choose candidate models.** For each role, start from the provider's current smallest model with reliable structured output; for detailed, the next tier if needed.
2. **Run `ai:eval`** (2 runs) for each candidate. Commit the reports.
3. **Supervised live error mapping with real keys:**
   - invalid key;
   - revoked key;
   - model not available to the key;
   - no credit or billing (an account without credit);
   - rate limit where it can be produced.

   Update the mapping table and tests to the observed responses.
4. **Disclosure:** write it from current terms (training use, retention, residency, paid versus free), and get owner sign-off (plan §15).
5. **Enable:** catalog `status: 'supported'`, recommended models, `evaluation` filled. Update `docs/ai/provider-evaluation.md`.

**Out of scope:** V1.x providers.

**Dependencies:** AI-02, AI-13, AI-14.

**Acceptance criteria**
- [ ] For each provider either:
  - reports pass all §7.3 criteria for the recommended models, mapping confirmed, disclosure signed off, entry enabled; or
  - the provider stays hidden and the reason is recorded. V1 ships without it (ADR release gate).
- [ ] Each model offered in Advanced has its own passing report.

**Testing requirements:** updated adapter mapping tests; catalog tests.

**Relevant documentation:** plan §7; architecture §2.3, §5.4, O2, O3, O9.

**Risks / notes:** live keys stay in the shell. Delete test keys at the provider after the run.

---

## AI-16 — Cross-cutting security, isolation and smoke verification

**Goal:** end-to-end evidence that keys and content never leak and users never affect each other.

**Scope** (plan §8 test list, §9):
1. **`ai-redaction.test.ts`:**
   - a sentinel key through save, check, sample, processing and every failure kind;
   - fake clients throw errors containing the sentinel;
   - scan captured console output, all HTTP responses, and every text/JSON column of every table including `pgboss.job.data` and `output`;
   - the content sentinel the same way.
2. **`ai-boundaries.test.ts`:** `openApiKey` is imported only by `access.ts` and `settings.ts`. `encryptedApiKey` is selected only there.
3. Cross-user tests: API, worker and wrong-AAD.
4. Production config tests (all new rules).
5. **Smoke:** final scenarios (configured, unconfigured, limited while another processes, `/ai` page status, Retry anyway sends the flag) with outbound blocking and teardown checks unchanged.

**Out of scope:** penetration testing, external scanners.

**Dependencies:** AI-12, AI-14.

**Acceptance criteria**
- [ ] All of the above pass in the full suite and the smoke, twice in a row.
- [ ] The smoke records zero residue.

**Testing requirements:** as scoped.

**Relevant documentation:** plan §8, §9; backend `STABILIZATION.md` smoke section.

**Risks / notes:** scanning every column must include JSON columns (`ai_operations.result`, `session.sess`).

---

## AI-17 — Documentation, operations runbook and supervised release

**Goal:** ship V1 safely and leave accurate documentation.

**Scope**
1. **Documentation:** every row of plan §12.
2. **Runbook** in backend `STABILIZATION.md`:
   - add `AI_CREDENTIAL_ENCRYPTION_KEY` (backed up) and `AI_USER_DAILY_CALL_LIMIT`;
   - remove `GEMINI_*` and `AI_DAILY_CALL_LIMIT`;
   - back up the database;
   - drain all old workers;
   - migrate;
   - deploy backend then frontend;
   - start with `AI_USER_DAILY_CALL_LIMIT=0`, then raise it;
   - tell hosted-key users (O8);
   - rollback rules: no return to the hosted key; keep the tables;
   - support diagnosis by events.
3. **Supervised live run per supported provider:** test Gmail account with synthetic emails. Steps:
   - setup;
   - verify (valid and invalid key);
   - sample test;
   - process 3 to 5 emails;
   - a refusal or limit where it can be produced;
   - approve one retry on a forced unknown (fake timeout via a short catalog timeout in a non-production build, or an observed one).

   Record evidence in `docs/planning/byo-ai/release-report.md`.
4. Measure the per-job duration per provider, to confirm the single-worker risk (plan §13).
5. Confirm or adjust the safety-limit default from observed volume.

**Out of scope:** new features found during the run (log them as follow-ups).

**Dependencies:** AI-15, AI-16.

**Acceptance criteria**
- [ ] Docs updated. ADR-0001 status shows implemented with commits.
- [ ] The release report covers every supported provider. Anything not produced live is marked "not verified".
- [ ] Production starts with the new configuration checks passing.

**Testing requirements:** full suites, smoke, live run.

**Relevant documentation:** plan §10, §12, §13.

**Risks / notes:** do not call fixture or synthetic results "live evidence".

---

## AI-18 — Post-release cleanup: drop the global budget table

**Goal:** remove the unused global budget after one stable release.

**Scope**
- Migration `DROP TABLE ai_call_budgets`.
- Remove the Prisma model and any remaining test references.

**Dependencies:** AI-17 plus one stable release with no rollback.

**Acceptance criteria**
- [ ] Fresh and upgrade lanes pass. No code references `aICallBudget`.

**Testing requirements:** migration lanes; full suite.

**Relevant documentation:** plan §4.5.

**Risks / notes:** destructive. Run only after confirming nothing reads the table.
