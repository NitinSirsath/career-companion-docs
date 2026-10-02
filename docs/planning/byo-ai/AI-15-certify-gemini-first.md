# AI-15 — Certify and enable Gemini first; OpenAI and Claude optional

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 8 — order 3 of 4 (can run in parallel with S8-01) |
| Repository | backend (eval runner and scorer, Gemini error mapping, catalog, tests, committed eval reports); frontend (contract sync and one test); docs repo (`docs/ai/provider-evaluation.md`, planning records) |
| Size / priority | M / the V1 release gate for BYO AI: Gemini is the minimum provider |
| Depends on | [AI-19](AI-19-runtime-safety-fixes.md) done (corrected data-use wording, exact `@google/genai` pin, `vertexai: false`); OD-09; a real Gemini API key in the owner's shell; [Phase 0 gate](../migration-verification/README.md) passed with git restored |
| Blocks | AI-17 supervised release ([release track](../README.md#7-release-track-not-scheduled)) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) AIB-15, AIB-02, AIF-06, AIB-04, AIF-04 (approval part), AIB-01 (context); [roadmap](../README.md) §5 item 6, §6 OD-09, §8; [Sprint 8 README](../sprint-8/README.md) §6–§7; replaces the AI-15 body in [issues.md](issues.md) (:730-762) and the live Gemini steps of AI-02 (:171-172) and AI-04 (:257-263); [BYO plan](README.md) §7 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Gemini becomes `supported` in the catalog on real evidence, or stays `hidden` with the reason written down. Evidence means: two passing evaluation runs per offered model, a live-confirmed error mapping, a checked retirement status, and an owner-approved data-use text with a new `disclosure.version`. Before any live run, the evaluation runner stops scoring rate limits and refusals as quality failures (step 0). OpenAI and Claude certification stays optional and runs only if the owner asks (OD-09).

## 2. Why it exists

- No provider is certified, so a production build offers no AI at all (AIB-01, AIB-15). Gemini is the V1 minimum ([ai-capability-architecture.md](../../architecture/ai-capability-architecture.md):96; [BYO plan](README.md):813). The owner also uses it daily, so a wrong mapping means wrong cooldowns or refusals ([Sprint 8 README](../sprint-8/README.md) §2).
- No open ticket had acceptance criteria for enabling Gemini (AIB-02, AIF-06). The old AI-15 covered only OpenAI and Claude (issues.md:730). AI-02 and AI-04 are "Done in code" with the live Gemini parts pending; AI-03 is Done; AI-09 was done before its "Gemini certified" dependency (issues.md:498).
- The runner turns a quota problem into a quality FAIL. A baseline full of refusals could also lower the floors for every later provider (AIB-04).
- The owner approves the data-use text only after AI-19 corrects it, and the version is bumped with the approval (AIF-04, OD-09).

## 3. Current behavior

Backend paths are in `career-companion-backend-main`; line numbers are from 2026-10-02.

### 3.1 Catalog and production gate (facts)

- Gemini entry in `src/contracts/aiCatalog.ts`: `status: 'hidden'` (:72); `disclosure.version: 'gemini-draft-2026-10'` (:87); `reviewedOn: null` (:93). Models: `gemini-2.5-flash-lite`, roles `['fast']`, recommended for fast (:96-107); `gemini-2.5-flash`, roles `['fast', 'detailed']`, recommended for detailed (:108-119). Both have `retiresOn: null` and `evaluation: null` (:105-106, :117-118). The frontend copy is identical (checked with `diff`).
- `evaluation` is per model: `{ date, report }` (:32-33). Nothing reads it except the catalog test.
- `isOffered` (`src/services/ai/access.ts:64-65`) allows a hidden provider only outside production; `readSettings` filters `offeredProviders` with it (`src/services/ai/settings.ts:83`), and `saveSettings` refuses a provider that is not offered (`settings.ts:151-152`).
- `src/tests/ai-catalog.test.ts:51-59`: a `supported` provider needs a non-null `reviewedOn`, every model needs a non-null `evaluation`, and no model may be past `retiresOn`. The loop is empty today. `src/tests/ai-access.test.ts:58-67` asserts that Gemini is `hidden` in production.
- `recommendedModel` (`aiCatalog.ts:249-253`) does not check retirement. A retired recommended model keeps being used until a release changes the catalog. `resolveModel` (`:262-273`) replaces only a retired *selected* model.
- `src/eval/ai/reports/` is empty. No live provider call has been made ([roadmap](../README.md) §1).

### 3.2 Evaluation runner and scorer (AIB-04, facts)

- `npm run ai:eval` runs `src/eval/ai/run.ts` (`package.json:20`). The key comes from `AI_EVAL_API_KEY` (`run.ts:56`, `:65`). It refuses `NODE_ENV=production` (`:50`). `--runs` defaults to 2 (`:55`). `--baseline` is optional (`:66-69`).
- The dataset has 41 cases: 7 IRRELEVANT, 16 critical, 4 borderline. One run makes 41 classification and 34 extraction calls, so `--runs 2` makes 150 calls.
- The calls go strictly one after another with no delay, retry or wait (`run.ts:75-114`). SDK retries are off on purpose (`src/services/ai/providers/gemini.ts:96-100`, `attempts: 1`). Every 429, and every Gemini 503, reaches the runner as `RATE_LIMITED` (`gemini.ts:90`). The parsed `retryAfterMs` (`gemini.ts:60-65`) is never used.
- `timed` (`run.ts:39-47`) keeps only the failure kind. A failed call gets `valid: false` (`:89`, `:103`); the kind is kept per call (`:90`, `:104`).
- `score` (`src/eval/ai/score.ts:149-200`) has no refusal category. A refused call lowers schema validity (`:189`, floor exactly 1 at `:209`, FAIL at `:224`). A refused classification on a critical case counts as a critical miss (`:165` counts any invalid classification; `src/eval/ai/eval.test.ts:145` asserts this for a schema-invalid one). A refused extraction counts as missed fields (`:179`). Relevance accuracy drops too.
- stdout prints the metrics table and `FAIL: schema validity …` with no cause (`run.ts:141-142`). The report has `passed` (`:130`) and no SDK version (`:120-133`). Its file name has only the date (`:134-139`), so a same-day re-run of the same pair overwrites the earlier report.
- Plan and code disagree: plan §7.3 allows "at most 1 invalid call across both" runs ([README.md](README.md):663); the code requires 100%. Plan §7.4 says the runner prints models retiring within 60 days (:689); it does not. Plan §7.2 puts the SDK version in the report (:654); it is not there.

### 3.3 Gemini error mapping (AIB-15, facts)

- `classifyGeminiError` (`gemini.ts:77-93`) is the plan §3.6 starting table, marked "must be confirmed with a real key" (:72-76). reason `API_KEY_INVALID` (any status) or 401 → `KEY_REJECTED` (:86); 403 or `FAILED_PRECONDITION` → `ACCOUNT_OR_BILLING` (:87-88); 404 → `MODEL_UNAVAILABLE` (:89); 429 or 503 → `RATE_LIMITED` (:90); 408 or 5xx → `OUTCOME_UNKNOWN` (:91); other → `INVALID_REQUEST` (:92).
- `googleErrorBody` (:52-70) parses `err.message` as JSON and reads only `error.status`, the first `details[].reason` and the first `details[].retryDelay`.
- Two live paths: save and "Check again" call `verifyModels` → `models.get` (:137-149); processing and the sample test call `generateContent` (:102-135). Save logs `ai_access_checked` with the kind (`settings.ts:184-193`, `on: 'check'` at :268). The sample test (`POST /api/ai/settings/sample-test`, `settings.ts:312`) sends only the built-in synthetic email and logs `ai_sample_test` with the kind (:339).
- `cooldownMs` (`src/services/ai/usage.ts:88-93`): with a `retryAfterMs`, the cooldown is that value clamped to 10 s–60 min, with no growth on repeated failures. Without one, it grows from the base, capped at 30 min.
- Adapter tests are table-driven with hand-built bodies (`src/tests/ai-providers-gemini.test.ts:41-77`). AI-19 adds a test through the SDK's real error path (`src/tests/ai-providers-gemini-sdk.test.ts`).

### 3.4 Consent and disclosure (AIF-04, OD-09, facts)

- Consent stores only the provider's `disclosure.version` (checked at `settings.ts:163-166`, stored at `:206` and `:226-227`, compared for `consent.current` at `:107`). A save with an old version is refused with "The data-use summary has changed". The form asks for consent again when `consent.current` is false (frontend `src/components/ai/ProviderSetupForm.tsx:71`) and sends the catalog version (:109).
- `deriveAccess` (`access.ts:81-106`) does not read the consent version, so processing continues with an old consent. Consent-version blocking is on the not-now list ([roadmap](../README.md) §8).
- Tests send the literal `gemini-draft-2026-10` as consent: `src/tests/ai-settings.test.ts:21`, `src/tests/ai-redaction.test.ts:86-90`, frontend `src/tests/ai-settings.test.tsx:135`. These break after a version bump. It is also stored as fixture data, which does not break: `src/tests/helpers/aiAccess.ts:14` (`configureAI`), `src/tests/ai-schema.test.ts:19`, frontend `src/tests/ai-settings.test.tsx:52` (a mocked response) and the frontend smoke seed (`scripts/smoke-stabilization.mjs:363`).

### 3.5 Suspected risks (not demonstrated)

- Free-tier quotas may cause 429s during a run, and may not cover the 300 calls of both pairs in one day (218 of them on `gemini-2.5-flash`, §7). The real limits depend on the key's tier and are unverified.
- A free-tier daily-quota 429 may carry a short `RetryInfo` delay. Each cooldown would then be as short as 10 s, with no growth, so every later attempt meets the same 429 until the quota resets (`usage.ts:88-90`). Whether Google sends it, and whether daily and per-minute 429s can be told apart (for example by a quota ID in a `QuotaFailure` detail, which the adapter does not read), is unverified.
- The response to a revoked (deleted) key is unverified.
- Google may have announced retirement dates for the Gemini 2.5 models by the time this runs. Unverified.

## 4. Scope

0. **Fix the evaluation runner before any live run (AIB-04).** Eval code only (`run.ts`, `score.ts`, plus one small helper module if needed).
   1. `--delay-ms <n>` (default 0, today's behavior): a pause between calls.
   2. On `RATE_LIMITED`, wait `retryAfterMs`, or 30 s if absent, then repeat the same call. At most 3 waits per call (starting value). If the delay asked for is over 120 s, do not wait; the call stays refused. A 429 or 503 means the provider did no work, so a repeat duplicates nothing (provider billing of refused calls is unverified). Never repeat `OUTCOME_UNKNOWN` or `INVALID_OUTPUT`. Latency is measured on the last attempt only.
   3. Refusals are `RATE_LIMITED` plus the access kinds (`ACCESS_FAILURE_KINDS`, `src/services/ai/errors.ts:63`). A refused call is left out of schema validity, latency and every quality metric, and is never a critical miss. A refused classification leaves its case out of relevance, critical, category and the injection-category check. A refused extraction leaves its case out of fields, hallucination and the injected-values check. `OUTCOME_UNKNOWN`, `INVALID_REQUEST` and `INVALID_OUTPUT` still count as invalid calls, as today.
   4. Metrics gain `refusedCalls` and counts per kind. For each refused call the report keeps `status`, `providerCode` and `retryAfterMs` (allowlisted fields of `ProviderFailure`, `errors.ts:82-107`; `timed` keeps only the kind today). The report gets `outcome`: FAIL if any quality floor fails on the answered calls, otherwise INCONCLUSIVE if any call was refused, otherwise PASS. Keep `passed` as `outcome === 'PASS'`. Exit codes: 0 PASS, 1 FAIL, 2 INCONCLUSIVE.
   5. A call still refused after its waits, or any access refusal, ends the run. The runner writes the partial report as INCONCLUSIVE.
   6. stdout names the outcome and the refusal counts by kind, for example `INCONCLUSIVE: 12 calls refused (RATE_LIMITED 12); not a quality result`.
   7. `--baseline` refuses a report that is not PASS, so refusals can never lower another provider's gate.
   8. Unit tests for all of this (§11). No network in tests.
1. **Baseline runs (AI-02 steps 4-5).** With the owner's key, run `--runs 2` for each pair below and commit each report under `src/eval/ai/reports/`.
   - Pair A (recommended): `--fast gemini-2.5-flash-lite --detailed gemini-2.5-flash`.
   - Pair B: `--fast gemini-2.5-flash --detailed gemini-2.5-flash`. Flash is offered for the fast role in Advanced (`aiCatalog.ts:111`). Each offered model needs its own passing report (issues.md:756).
   - **Decision (owner, at ticket start; architecture O9).** Recommendation: run pair B. It costs one more run. The alternative is to remove `'fast'` from flash's roles (a catalog change; `src/tests/ai-catalog.test.ts:67` and `:77` change, and users who chose it fall back to the recommendation through `resolveModel`).
   - Record the SDK version (`npm ls @google/genai`) and contract versions in `provider-evaluation.md` (AI-02 acceptance).
   - **Schema validity floor (decision, at ticket start).** Recommendation: keep the code's 100% over answered calls, and keep "one full re-run allowed" for a one-off flake. Reword plan §7.3 (:663) and `provider-evaluation.md`:34 to match. Reason: once refusals are excluded, an invalid call is a real defect, and the re-run already covers a flake.
   - **If Gemini misses a floor:** stop and report it to the owner (AI-02 acceptance). With the owner's OK, lower that floor in `THRESHOLDS` to the measured value before the run that is committed, and record it ([provider-evaluation.md](../../ai/provider-evaluation.md):44). Recommendation: never allow `criticalMisses` or `injectionFailures` above 0; keep Gemini hidden instead.
2. **Supervised live error mapping (AI-04 step 5, AIB-15).** No email data. Use a separate test key for the destructive rows. Observe both paths (`models.get` and `generateContent`). For each row, the throwaway probe (§7) shows the raw response structure on both paths. The app path shows the mapped kind end to end; the `ai_access_checked` and `ai_sample_test` logs carry only the kind.
   - Invalid key: the probe, a save on the local `/ai` page (backend and frontend `npm run dev`), and one `ai:eval --runs 1` with the bad key.
   - Revoked key: save a valid test key, delete it in Google AI Studio, then use the probe, "Check again" and "Try a sample email".
   - Model not available: a non-existent model ID through the probe. The app accepts only catalog model IDs.
   - Quota 429: a per-minute burst, with the probe in a short loop or `ai:eval --delay-ms 0` (its report keeps only status, code and delay). A free-tier daily exhaustion only if it happens or a throwaway project is used; otherwise "not produced".
   - Region or billing (`FAILED_PRECONDITION`) only if it can be produced.
   - Record per row in `provider-evaluation.md`: date, path, HTTP status, Google `error.status`, detail types, whether `retryAfterMs` was present, and the mapped kind. Rows not produced say "not produced".
   - Update `classifyGeminiError` and the tests to the observed responses. Every change gets a test row.
   - **Daily quota (decision, only if observed).** If the daily 429 can be told apart and carries a short delay, recommendation: map it to `RATE_LIMITED` *without* `retryAfterMs`, so the existing growth applies (capped at 30 min). This is a mapping-only change. If it cannot be told apart, change nothing and record it.
3. **Retirement check (AIB-15 correction a).** Check Google's model and deprecation pages for both models on the day. Set `retiresOn` only if a date is announced (issues.md:209). Record the date and source checked. If a recommended model retires within 60 days (plan §7.4), stop and ask the owner. Replacing a model is a follow-up, not this ticket.
4. **Owner approval of the disclosure (OD-09, AIF-04).** Check the Gemini `summary`, `training` and `residency` text (`aiCatalog.ts:89-92`) and the shared summary corrected by AI-19 against Google's current Gemini API terms. The free-tier wording must say plainly how Google may use free-tier content ([BYO plan](README.md):966). Check that the three links open the right pages (`aiCatalog.ts:75-79`). After the owner approves: set `reviewedOn` to the approval date and a new `version` (recommended form `gemini-YYYY-MM-DD`). Record the approval (owner, date, version) in `provider-evaluation.md`. The bump happens with the approval even if Gemini stays hidden.
5. **Enable, or record why not.** If steps 1-4 pass: Gemini `status: 'supported'`, each model's `evaluation` set to `{ date, report: 'src/eval/ai/reports/<file>.json' }` (flash-lite → pair A; flash → pair B, or pair A if `'fast'` was removed from flash), frontend contract sync, `provider-evaluation.md` status row updated. If not: Gemini stays `hidden`, and the reason is recorded in `provider-evaluation.md` and the Sprint 8 execution report.
6. **Tests that follow the catalog change:** see §11.
7. **Optional, only if the owner asks: OpenAI and Claude** (old AI-15 steps; both stay hidden under OD-09).
   - Pick candidates per role (catalog today: `gpt-5-nano`, `gpt-5-mini`; `claude-haiku-4-5-20251001`, `claude-sonnet-5-5`). Run `ai:eval --runs 2` with `--baseline <Gemini pair A report>`. Commit the reports.
   - Live mapping: invalid key, revoked key, model not available, no credit, rate limit. Open question: an OpenAI restricted key without "Models: Read" may be refused at save, because verify calls `models.retrieve` (`src/services/ai/providers/openai.ts:78`, `verifyModels`) and both 401 (`:19`) and 403 (`:21`, `classifyOpenAIError`) are access kinds that give a 422 (`settings.ts:194`). Record what happens. Anthropic billing and credit errors need a zero-credit key (plan §3.6).
   - Disclosure approval and a version bump per provider, then enable, as in steps 4-5.

## 5. Out of scope

- The AI-17 supervised release run, `release-report.md`, job-duration measurement and the limit-default review (release track).
- Any deployment or production configuration (OD-08).
- AI-18 (drop `ai_call_budgets`).
- New providers or models (V1.x), and replacing a retiring model.
- Prompt or contract changes, or a new contract version (plan §7.3).
- Changes to cooldown, ledger or worker behavior beyond the Gemini mapping. The `INVALID_REQUEST` pause is AI-19.
- Consent-version blocking and the zero-provider empty state (roadmap §8, release track).
- Making the runner print retiring models or add the SDK version to reports; both are recorded by hand here.

## 6. Likely files and components

- Backend: `src/eval/ai/run.ts`, `src/eval/ai/score.ts`, `src/eval/ai/eval.test.ts`, `src/eval/ai/README.md`, new `src/eval/ai/reports/*.json`; `src/services/ai/providers/gemini.ts` (only if a mapping changes); `src/contracts/aiCatalog.ts`.
- Backend tests: `src/tests/ai-catalog.test.ts`, `ai-access.test.ts`, `ai-settings.test.ts`, `ai-redaction.test.ts`, `ai-providers-gemini.test.ts`, `ai-providers-gemini-sdk.test.ts` (from AI-19).
- Frontend: `src/contracts/aiCatalog.ts` (by sync only), `src/tests/ai-settings.test.tsx`.
- Docs: `docs/ai/provider-evaluation.md`, `docs/planning/byo-ai/README.md` (§7.2-§7.4 wording), `docs/planning/byo-ai/issues.md` (AI-15 status row), roadmap §1 and §5.

## 7. Implementation notes

- **Data model and migration.** None. The catalog is code. No guarded migration is needed.
- **Shared contracts.** Only `aiCatalog.ts` changes (status, `evaluation`, `reviewedOn`, `version`, maybe `retiresOn`). Frontend `npm run sync-contracts` reports 1 file changed, then 0. The API shape is unchanged. In production, `offeredProviders` would now include `gemini`, but nothing is deployed (OD-08).
- **Background jobs.** None. Waiting and cooldown rules are unchanged, except a mapping fix from step 2.
- **Throwaway probe (step 2).** A short script outside the repo, or deleted after use, run from the backend with `npx ts-node --transpile-only`. It builds the SDK client with the key from the shell and the same options as `createGeminiClient` after AI-19 (`vertexai: false`, `enterprise: false`, `attempts: 1`), using `models.get` and `generateContent` with the text `ping`. It prints only `classifyGeminiError` fields and the allowlisted body structure (`error.code`, `error.status`, `details[].@type`, `reason`, `retryDelay`, quota IDs). It never prints `error.message`. Not committed.
- **Running.** The runner needs `npm run db:generate` (`src/services/ai/contracts.ts:2` imports the `EmailCategory` enum from `@prisma/client`) but no database. Commit or move the first report before a same-day re-run of the same pair (`run.ts:134-139`). If a run is INCONCLUSIVE, wait for the quota to reset or raise `--delay-ms`, then re-run. INCONCLUSIVE reports need not be committed; list them in the execution report.
- **Free-tier limits.** Pair A plus pair B make about 218 calls on `gemini-2.5-flash`. This may exceed a free-tier daily quota (unverified). Spread the runs over two days or use a billing-enabled key.
- **Compatibility.** After the version bump, the owner's saved Gemini setup shows consent as not current. The next save asks again; processing continues meanwhile (§3.4). Migration-lane scripts insert `gemini-draft-2026-10` as fixture data (`scripts/verify-ai-migration.cjs:56`, `scripts/verify-mcp-migration.cjs:146`); leave them.
- **Rollback.** Revert the catalog commit (status `hidden`, `evaluation` null) and re-sync. Keep the approved `version` and `reviewedOn`, so recorded consents stay current. Reports stay as evidence. The runner fix is its own commit, so it can stay or be reverted separately.

## 8. Dependencies

- AI-19 done: the corrected `AI_DATA_SENT_SUMMARY`, `@google/genai` pinned to 2.22.0, `vertexai: false` and `enterprise: false`, and the real-SDK test file. The owner approves only corrected text (OD-09).
- OD-09 recorded: Gemini only for V1; the owner approves the corrected text and the version is bumped with it.
- A real Gemini key in the owner's shell, plus a throwaway key for the revoked-key row.
- Phase 0 gate passed and git restored, so each step lands as a focused commit. S7-01 CI, if present, runs `eval.test.ts` through `npm test`.

## 9. Security and privacy

- Live keys stay in the shell (`AI_EVAL_API_KEY`). Never in an `.env` file, a report, a log, a ticket or a chat. Read the key with `read -s` so it is not echoed or kept in history; `unset` it after the run.
- Reports hold synthetic data, metrics, failure kinds and allowlisted codes only. Before each commit, check that the key is not in any report (§11).
- The live mapping uses no email data: only the synthetic sample email and the text `ping`.
- A key typed into the local `/ai` page is stored sealed in the dev database. Remove the test configuration after the run.
- Delete test keys at the provider after the run.
- Free tier: under the current draft text, Google may use free-tier content to improve its products, and people may review it (`aiCatalog.ts:90-91`; confirm against current terms in step 4). Synthetic evaluation data is fine on the free tier. Real email on a free-tier key is the owner's informed choice, and the approved text must say so plainly.
- The runner still refuses production (`run.ts:50`).

## 10. Acceptance criteria

- [ ] `eval.test.ts` proves: a refused call is not schema-invalid and not a critical miss; a refused extraction is left out of field metrics; a run with refusals and no quality failure is INCONCLUSIVE; a quality failure with refusals is FAIL; the wait honors `retryAfterMs`, is bounded, and never repeats `OUTCOME_UNKNOWN` or `INVALID_OUTPUT`; `--baseline` refuses a non-PASS report.
- [ ] stdout and the report name the outcome and the refusal counts by kind.
- [ ] Pair A (and pair B, or the recorded alternative) has a committed report with `runs: 2`, outcome PASS and 0 refused calls.
- [ ] `provider-evaluation.md` records the final thresholds, the Gemini results, SDK and contract versions, and any lowered floor with the owner's OK.
- [ ] Each mapping row is recorded as observed or "not produced", with date and path. Every mapping change has a test row. No row is called confirmed without a live observation.
- [ ] The retirement check is recorded with date and source; `retiresOn` is set only if announced.
- [ ] The owner's approval is recorded (date, new `version`); `reviewedOn` is set; the links were checked.
- [ ] Either Gemini is `supported` with `evaluation` set for both models, and backend and frontend contracts are identical; or Gemini is `hidden` and the reason is recorded.
- [ ] OpenAI and Claude stay `hidden` unless the owner asked for §4 item 7.
- [ ] No key appears in any file, report or log; the test keys are deleted at Google.

## 11. Testing

Tests to add or change:
- `src/eval/ai/eval.test.ts`: the §10 cases. Test the wait with an injected `sleep` (no real timers) by moving it into a small exported helper. The existing tests stay, including `:128-146`, where a schema-invalid classification (`error: 'SchemaValidationFailure'`) still counts as invalid and as a critical miss. The live runner records the same case as `INVALID_OUTPUT` (`src/services/ai/capabilities.ts:34`).
- `src/tests/ai-catalog.test.ts`: the loop at `:51-59` becomes active. Recommended: also check that each supported model's `evaluation.report` file exists, has outcome PASS and names that model.
- `src/tests/ai-access.test.ts:58-67`: use a provider that stays hidden (`openai`) for the hidden-in-production check. Add a check that a `supported` Gemini is offered in production.
- `src/tests/ai-settings.test.ts`: the production API test that [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md) adds expects `offeredProviders: []` and a 400 for `PUT` with `gemini`. Change it to a provider that stays hidden (`openai`) and expect `offeredProviders: ['gemini']`.
- `src/tests/ai-settings.test.ts:21`, `src/tests/ai-redaction.test.ts:86-90`, frontend `src/tests/ai-settings.test.tsx:135`: read the version from `getCatalogProvider('gemini')!.disclosure.version` instead of the literal.
- `src/tests/ai-providers-gemini.test.ts:41-64` and `ai-providers-gemini-sdk.test.ts`: add the observed bodies (sanitized, sentinel only).

Commands — backend (`career-companion-backend-main`):
```bash
npm run db:generate && npm run typecheck && npm run lint && npm run build
npx vitest run src/eval/ai/eval.test.ts src/tests/ai-catalog.test.ts src/tests/ai-access.test.ts src/tests/ai-settings.test.ts src/tests/ai-redaction.test.ts src/tests/ai-providers-gemini.test.ts src/tests/ai-providers-gemini-sdk.test.ts
npm test
# No migration in this ticket. Only if the test DB is behind:
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
```

Live runs — owner, backend folder, real network (not part of `npm test`):
```bash
read -s AI_EVAL_API_KEY && export AI_EVAL_API_KEY
npm run ai:eval -- --provider gemini --fast gemini-2.5-flash-lite --detailed gemini-2.5-flash --runs 2 --delay-ms <n>
npm run ai:eval -- --provider gemini --fast gemini-2.5-flash --detailed gemini-2.5-flash --runs 2 --delay-ms <n>
grep -rlF -- "$AI_EVAL_API_KEY" src/eval/ai/reports && echo 'KEY FOUND: do not commit'
unset AI_EVAL_API_KEY
```
Pick `<n>` from the key's quota page on the day.

Commands — frontend (`career-companion-frontend-main`):
```bash
npm run sync-contracts   # 1 file changed (aiCatalog.ts); run again: 0 changed
npx vitest run src/tests/ai-settings.test.tsx
npm run typecheck && npm run lint && npm test && npm run build
```

Smoke: run once, because the consent version changes. The seed stores the old literal (`scripts/smoke-stabilization.mjs:363`), but the BYO AI scenario deletes that setup and saves a new one through the form (`:822-841`), which sends the catalog version. Expect no change; change the literal only if a step fails. Fixture smoke results are not live evidence.

## 12. Documentation updates

- `docs/ai/provider-evaluation.md`: thresholds, Gemini results and report names, SDK and contract versions, the observed mapping table, the retirement check, the approval record, and the status row (`:59-65`). Also the floor wording at `:34`.
- Backend `src/eval/ai/README.md`: `--delay-ms`, the wait rule, refusals and the INCONCLUSIVE outcome.
- `docs/planning/byo-ai/README.md`: §7.3 floor wording (:663); §7.2 (:654) and §7.4 (:689) say the SDK version and the retirement check are recorded by hand.
- `docs/planning/byo-ai/issues.md` status row for AI-15: the result. S6-C02 (closeout) points AI-02 and AI-04 to this ticket.
- [Roadmap](../README.md) §1 BYO AI row and §5 item 6: Gemini supported, or hidden with the reason.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend tests, typecheck, lint and build are green, and the contract sync is clean (in CI once S7-01 exists).
- [ ] Behavior verified: the runner changes by unit tests; the Gemini results and mapping by the live runs above, recorded as live; anything not produced is marked "not produced".
- [ ] Docs updated per §12.
- [ ] Focused commits: runner fix; reports and thresholds; mapping; catalog and approval.
- [ ] Evidence (reports, mapping table, approval, decisions) recorded in the Sprint 8 `execution-report.md`. No fixture or synthetic result is called live evidence.
