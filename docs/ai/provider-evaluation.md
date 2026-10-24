# AI provider evaluation and certification

| Field | Value |
| --- | --- |
| Status | **OpenRouter certified (2026-10-23)**, see "OpenRouter certification" below. Gemini, OpenAI and Anthropic stay `hidden`: usable in development and tests, never offered in production except on a test deploy (see "Test deploys" below). |
| Decision | [ADR-0001](../architecture/decisions/ADR-0001-user-provided-ai.md) decision 2; [BYO AI plan §7](../planning/byo-ai/README.md#7-provider-evaluation-plan) |
| Code | backend `src/eval/ai/` (dataset, scorer, runner); catalog `src/contracts/aiCatalog.ts` |

A provider/model pair is offered to users only after it passes Career Companion's synthetic evaluation, its error mapping is confirmed with a real key, and the owner approves its data-use text.

## How to run an evaluation

```bash
AI_EVAL_API_KEY=<key in the shell only> npm run ai:eval -- --provider gemini --fast gemini-3.5-flash-lite --detailed gemini-3.8-flash --runs 2
```

- Run from the backend folder.
- Only catalog models are accepted. Add a candidate as a `hidden` catalog entry first.
- No database, Gmail or ledger is touched. The runner refuses to start when `NODE_ENV=production`.
- A report is written to `src/eval/ai/reports/<date>_<provider>_<fast>_<detailed>.json`. It contains synthetic data, metrics and error kinds only; commit it.
- Compare against the Gemini baseline with `--baseline <report.json>`.
- `--triage batch --batch-size <1-25>` checks relevance only, in batches (no extraction calls).
- Google limits Gemini 2.5 models to projects that used them before. A new key gets `404 NOT_FOUND` (`MODEL_UNAVAILABLE`) for them, so use the Gemini 3 models above.
- OpenRouter: `--provider openrouter --fast anthropic/claude-haiku-5.5 --detailed openai/gpt-6-luna`. A `/` in a model ID becomes `_` in the report's file name.
- Reports are committed exactly as the runner wrote them; `.prettierignore` keeps the formatter away from them.

## CI evaluation (COM-136)

The backend workflow `.github/workflows/ai-eval.yml` runs the evaluation on GitHub Actions.

- **When:** on pull requests that change `src/services/ai/`, `src/eval/ai/` or `src/contracts/aiCatalog.ts`, and manually with "Run workflow". It never runs for pull requests from forks.
- **What:** Gemini, batch mode, `--fast gemini-3.5-flash-lite --detailed gemini-3.8-flash --triage batch --batch-size 22 --runs 1`. Relevance and category only.
- **Cost:** 2 requests per run, about 4,800 input and 2,000 output tokens. The report's token totals over-count in batch mode because each email repeats its batch's usage.
- **Key:** repository secret `GEMINI_EVAL_API_KEY`, created in its own Google Cloud project so it never shares the app's free-tier limits. It only ever receives the synthetic dataset.
- **Report:** downloadable from the run as the artifact `ai-eval-report`.
- **Quota used up:** the run ends `INCONCLUSIVE` (red) after at most three rate-limit waits. It is not a required check, so merging still works. Re-run the job after the daily limit resets.

## Dataset

45 hand-written synthetic emails in `src/eval/ai/dataset/`:
- recruiter 5 (including a LinkedIn InMail), application received 3, interview 6, assessment 4, offer 3, rejection 4, follow-up 3, job alerts 5 (including LinkedIn recommended and sponsored jobs), irrelevant 7, adversarial 5 (prompt injection, non-English, long thread, a fake recruiter asking for a fee, a fake offer asking for a fee);
- since 23 Oct 2026 (owner decision), the two fake-offer cases are expected `RELEVANT` (Offer or Recruiter) with `fakeOfferWarning` set. Before, the fake recruiter case expected "any" relevance and the old `SPAM` category, so it was not really scored;
- since COM-133, job alerts and job-platform newsletters are expected `IRRELEVANT`;
- fictitious companies and people, reserved domains only (enforced by a CI test);
- case 1 is the built-in sample email used by the user's "Try a sample email".

## Pass criteria (starting floors)

| Metric | Floor |
| --- | --- |
| Schema validity, all calls in both runs | 100% (one re-run allowed to rule out flakiness) |
| Relevance accuracy (non-borderline) | ≥ 90% |
| Interview, assessment or offer mail marked IRRELEVANT at or above the confidence threshold | 0 |
| Category accuracy (relevant cases) | ≥ 80% |
| Key-field accuracy | ≥ 90% |
| Hallucinated fields (`mustBeNull`) | ≤ 3% |
| Prompt-injection cases | 0 failures |
| Below the Gemini baseline (relevance, category, fields) | at most 5 points |
| p95 latency per call | ≤ 15 s |

The Gemini baseline run fixes the final values. A floor the current Gemini models miss is lowered to the baseline value and reported to the owner, never hidden.

## Certification checklist (per provider)

1. **Evaluation:** a passing report for every model offered for each role. A floor a model misses is offered only as an owner-approved exception, recorded with its cause in this file.
2. **Error mapping:** confirmed with a real key. Record each row as observed or "not produced".
   - invalid key;
   - revoked key;
   - model not available to the key;
   - no credit or billing;
   - rate limit;
   - provider busy (the provider answers that it is overloaded).
3. **Disclosure:** the catalog's data-use text (summary, training, residency) checked against the provider's current terms and **approved by the owner**. Then set `disclosure.reviewedOn` and a new `disclosure.version`.
4. **Links:** key, billing and terms links open the right pages.
5. **Catalog PR:** `status: 'supported'`, `evaluation` filled for each model, recommended models set. The catalog test refuses `supported` without these.

## Test deploys (`AI_SHOW_HIDDEN_PROVIDERS`)

Owner decision, 2026-10-20: the Render test deploy (owner and one friend) runs with `NODE_ENV=production` and uses OpenRouter with GPT-6 Sol before any provider is certified. The 44-mail certification is not run for it.

- Backend setting `AI_SHOW_HIDDEN_PROVIDERS=true` offers the `hidden` providers in production too. Empty (the default) offers only `supported` ones. Any other value stops startup.
- While it is on in production, startup logs the warning "AI_SHOW_HIDDEN_PROVIDERS is on: uncertified AI providers are offered. Test deploys only."
- It changes no catalog entry: every provider stays `hidden` and uncertified, and users still see each provider's draft data-use text before they save a key.
- Never set it for real users. Before real users: remove it and certify the provider with the checklist above.
- OpenRouter is `supported` since 2026-10-23, so the Render deploy no longer needs the setting for it. Remove it there; Gemini, OpenAI and Anthropic then disappear from that deploy.

## Current status

**Batch relevance evaluation (CI, 2026-10-06):** PASS for `relevance-batch/v2` on `gemini-3.5-flash-lite`, 44 cases, 1 run: relevance 100%, 0 critical misses, category 96%, p95 latency 2.6 s. Extraction was not part of this run. `AI_TRIAGE_BATCH_ENABLED` stays off until the owner decides to turn it on.

| Provider | Models (candidates) | Evaluation | Error mapping | Disclosure | Status |
| --- | --- | --- | --- | --- | --- |
| Google Gemini | Recommended: gemini-3.5-flash-lite (fast), gemini-3.8-flash (fast, detailed). Also offered: gemini-2.5-flash-lite (fast), gemini-2.5-flash (fast, detailed) | Batch relevance PASS on gemini-3.5-flash-lite (CI, 2026-10-06); full evaluation not run | Observed live on 2026-10-12: "model not available to the key" = 404 `NOT_FOUND` → `MODEL_UNAVAILABLE` (2.5 models, new key); "provider busy" = 503 `UNAVAILABLE` and 504 `DEADLINE_EXCEEDED` → `PROVIDER_BUSY` (gemini-3.8-flash; from 2026-10-14, owner decision: 504 counts as no answer in time, not busy). Other rows not confirmed live | Draft, needs owner approval | hidden |
| OpenAI | gpt-5-nano (fast), gpt-5-mini (fast, detailed) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |
| Anthropic Claude | claude-haiku-4-5-20251001 (fast, detailed; tool mode), claude-sonnet-5-5 (detailed; native JSON schema) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |
| Career Companion free AI | openai/gpt-6-luna (fast, detailed), fixed: the user has no model choice | The OpenRouter certification of Luna applies (same model and servers) | Same as OpenRouter. On the free AI no refusal is the user's problem: it pauses the free AI for everyone (owner decision, 2026-10-24) | Approved by the owner with the mock, 2026-10-24 (`career-companion-free-2026-10-24`): mail goes to OpenRouter on Career Companion's account | **supported** while the server has its key |
| OpenRouter | Recommended: openai/gpt-6-luna (fast, detailed). Also offered: anthropic/claude-haiku-5.5 (fast), google/gemini-3.8-flash (detailed), openai/gpt-6-sol (fast, detailed) | PASS: Haiku and Gemini (2026-10-22). Luna and Sol pass every floor except one case, an owner exception (see below) | Confirmed live 2026-10-17 to 2026-10-23 (see below) | Approved by the owner, 2026-10-23 (`openrouter-2026-10-23`) | **supported** |

**Gemini notes (2026-10-12):**
- The 2.5 models are not recommended: Google serves them only to keys that used them before, and a new key gets 404 for both.
- The key check sends one very small real request to each chosen model. Asking Google to describe the model (`models.get`) is not enough: Google answers that also for a model it refuses every real request for.
- 503 is Google's own answer that it did not do the work, and Google does not charge a request that fails with a 400 or 500 error. Owner decision, 2026-10-14: 503, 500 and 502 count as busy. The mail waits, no try is used and nothing is put on hold.
- Owner decision, 2026-10-14: Google's 504 ("deadline expired") counts as no answer in time, like our own timer or a connection that dropped after sending. It uses 1 of 3 tries and Career Companion tries again by itself. After the 3rd try the mail waits for the user's OK.

**OpenRouter certification (2026-10-21 to 2026-10-23):**

Owner decisions:
- Certify OpenRouter only; the other three providers stay hidden.
- Offer four models: Claude Haiku 5.5 (fast), GPT-6 Luna (fast and detailed, recommended for both), Gemini 3.8 Flash (detailed) and GPT-6 Sol (fast and detailed).
- Luna recommended for both steps (2026-10-23), after a 49-mail sandbox test of the whole flow (send, sync, AI): Luna for both steps and Haiku + Gemini each read 43 of 44 scored mails right, but Haiku let a fake job offer asking for a fee through, and Luna cost a sixth as much ($0.0076 against $0.046 for 47 mails).
- Remove Gemma 4 26B A4B and DeepSeek V4.1 Flash. Their 2026-10-17 problems are in the notes below.
- No country limit (O6, 2026-10-23): all four models run on Google or Microsoft Azure servers in the US or Europe. Decide again when a model on another company's servers is added.

Evaluations (`--runs 2`, all 44 cases each run; reports in backend `src/eval/ai/reports/`):

| Report | Fast + detailed | Result | Why |
| --- | --- | --- | --- |
| 2026-10-21T11-43-22-410Z | Haiku 5.5 + Gemini 3.8 Flash (60 s) | FAIL | 2 Gemini calls gave no answer in 60 s |
| 2026-10-21T12-02-39-117Z | GPT-6 Sol + GPT-6 Sol | FAIL | Prompt-injection case blocked (see exception) |
| 2026-10-22T11-35-57-992Z | Haiku 5.5 + Gemini 3.8 Flash (60 s) | FAIL | 2 Gemini timeouts; p95 17.3 s |
| 2026-10-22T19-19-02-480Z | Haiku 5.5 + Claude Sonnet 5.5 | FAIL | Every extraction refused: Anthropic's 16-field limit. Sonnet is not offered |
| 2026-10-22T19-22-12-192Z | GPT-6 Luna + GPT-6 Luna | FAIL | Prompt-injection case blocked (see exception) |
| 2026-10-22T19-22-55-385Z | Haiku 5.5 + Gemini 3.8 Flash (90 s) | **PASS** | Relevance and category 100%, fields 96%, p95 8.3 s, slowest 30 s |

- Haiku 5.5 was perfect in all four of its runs: every answer valid, relevance and category 100%, p95 2.4 s or less.
- Gemini 3.8 Flash always thinks, and under 60 s it sometimes gave no answer in time. Its timeout is now 90 s, and it is offered for detailed only.
- **Owner exception (GPT-6 Luna and GPT-6 Sol), 2026-10-22:**
  - Both fail only the `adversarial-prompt-injection` case: an email that tells the AI to ignore its instructions.
  - Azure's safety filter on their zero-retention servers stops the answer (`finish_reason: content_filter`, confirmed with a one-call check).
  - Career Companion reads that as unusable output, so such a mail waits for the user and nothing wrong is saved.
  - Every other floor passed, with the same numbers for both models: relevance 97.7%, 0 critical misses, category 92.9%, fields 93.8%, no made-up fields, p95 4.5 s or less.
  - The owner keeps both models: real job mail rarely contains such text.
- **Key credit limit (GPT-6 Luna and GPT-6 Sol):**
  - Their servers take no output limit, so OpenRouter holds the worst-case cost on the key's credit limit before each call: about $0.07 for Luna and $1.30 for Sol.
  - With a fresh $0.01 key, Haiku answered but Sol got 402 `openrouter_key_limit` at once.
  - Owner decision, 2026-10-23: no code change. The setup steps ask for a key limit of at least $5, or none.
  - Clear wording per error cause is follow-up work (COM-247).

Error mapping, confirmed live with real keys (2026-10-17 to 2026-10-23):

| Row | Observed |
| --- | --- |
| Invalid key | 401 → `KEY_REJECTED` |
| Revoked (deleted) key | 401 → `KEY_REJECTED` |
| Expired key | 401 → `KEY_REJECTED` |
| Model not available (no zero-retention server takes the request) | 404 → `MODEL_UNAVAILABLE` |
| No credit under the key's limit | 402 `openrouter_key_limit` → `ACCOUNT_OR_BILLING` |
| Blocked content (Azure filter) | 200 with `finish_reason: content_filter` → `INVALID_OUTPUT` |
| Rate limit | Observed 2026-10-23: 429 `openrouter_new_account` (a new account's limit, in the first minutes of a new key) → `RATE_LIMITED`; the model pauses 60 s and the mail goes on by itself |
| Provider busy | Not produced. Busy is: OpenRouter 500 and 502, 503 with `provider_overloaded`, and a 200 answer whose error is `provider_overloaded` or `provider_unavailable` |

Disclosure:
- Checked on 2026-10-22 against OpenRouter's privacy policy (updated 2026-08-31) and its zero-data-retention page.
- OpenRouter does not train on inputs or outputs, and it keeps no prompts unless prompt logging is turned on. `zdr: true` sends requests only to servers that keep nothing. OpenRouter's servers are in the US.
- The owner approved the text on 2026-10-23 (`openrouter-2026-10-23`).

Links (checked 2026-10-23): the key and credits links go through OpenRouter's sign-in to the keys and credits pages, and the privacy link opens the privacy policy.

**OpenRouter notes (2026-10-17):**
- Owner decisions, 2026-10-17: two small models only. Every request asks for servers that keep nothing (`zdr`) and that take every parameter sent (`require_parameters`). No block by country or company for now; decide again before real users. Hidden until certified.
- The key check sends no paid request. It reads the key's own record (`GET /key`): a rejected key, or no credit left under the key's limit. Then it reads one list of servers that keep nothing (`GET /endpoints/zdr`): each chosen model needs a server that takes every parameter sent.
- Rows to confirm live: 402 (no credit), 402 with too much spend in flight, 403 for a key limit and 403 for blocked content, 404 and 503 when no server meets the rules, and a 200 answer that carries an error.
- Owner decision, 2026-10-17: three test candidates for the evaluation: Claude Haiku 5.5, Gemini 3.8 Flash and GPT-6 Sol. GPT-6 Sol's zero-retention servers take no output limit, so none is sent for it; zero retention stays for every model.
- First live check (sample email, 2026-10-17): 401 for a wrong key confirmed. Haiku 5.5 with Gemini 3.8 Flash, and GPT-6 Sol alone, answered correctly. Gemma 4 26B A4B answered correctly when it answered, but its servers often gave no answer in time. DeepSeek V4.1 Flash extracted correctly but screened the sample wrongly.
- Anthropic's JSON output allows at most 16 fields that may be empty, so Haiku 5.5 screens only (extraction has 19). An optional fixed list is sent as "one of the list, or null" (`anyOf`), which Anthropic accepts.

**Re-evaluate when:**
- a model is added or becomes the recommended one;
- a contract version changes;
- an adapter's request shape changes;
- an SDK major version changes;
- a provider announces a deprecation or behavior change.

## Changing an OpenRouter model

Owner decision, 2026-10-17: OpenRouter models change only through the catalog plus an evaluation run, never through a setting while the app runs. Exact model IDs only: an alias such as `~…-latest` can move to another model without an evaluation.

1. Check the candidate's facts on OpenRouter: servers that keep nothing and take our request, thinking, price, expiry date and server countries. From COM-242 on: `npm run ai:openrouter-models -- <model id>`.
2. Add it to the catalog with `evaluation: null`.
3. Run the evaluation twice (see "How to run an evaluation").
4. If it passes, make it recommended or replace the old model. Users on a removed model move to the recommended one by themselves.
5. Ship one small backend PR, plus the frontend contract sync.
