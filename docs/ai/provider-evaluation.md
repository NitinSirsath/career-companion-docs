# AI provider evaluation and certification

| Field | Value |
| --- | --- |
| Status | **No provider certified yet (2026-10-02).** All three catalog providers are `hidden`: usable in development and tests, never offered in production. |
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

## CI evaluation (COM-136)

The backend workflow `.github/workflows/ai-eval.yml` runs the evaluation on GitHub Actions.

- **When:** on pull requests that change `src/services/ai/`, `src/eval/ai/` or `src/contracts/aiCatalog.ts`, and manually with "Run workflow". It never runs for pull requests from forks.
- **What:** Gemini, batch mode, `--fast gemini-3.5-flash-lite --detailed gemini-3.8-flash --triage batch --batch-size 22 --runs 1`. Relevance and category only.
- **Cost:** 2 requests per run, about 4,800 input and 2,000 output tokens. The report's token totals over-count in batch mode because each email repeats its batch's usage.
- **Key:** repository secret `GEMINI_EVAL_API_KEY`, created in its own Google Cloud project so it never shares the app's free-tier limits. It only ever receives the synthetic dataset.
- **Report:** downloadable from the run as the artifact `ai-eval-report`.
- **Quota used up:** the run ends `INCONCLUSIVE` (red) after at most three rate-limit waits. It is not a required check, so merging still works. Re-run the job after the daily limit resets.

## Dataset

44 hand-written synthetic emails in `src/eval/ai/dataset/`:
- recruiter 5 (including a LinkedIn InMail), application received 3, interview 6, assessment 4, offer 3, rejection 4, follow-up 3, job alerts 5 (including LinkedIn recommended and sponsored jobs), irrelevant 7, adversarial 4 (prompt injection, non-English, long thread, spam posing as a recruiter);
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

1. **Evaluation:** a passing report for every model offered for each role.
2. **Error mapping:** confirmed with a real key. Record each row as observed or "not produced".
   - invalid key;
   - revoked key;
   - model not available to the key;
   - no credit or billing;
   - rate limit.
3. **Disclosure:** the catalog's data-use text (summary, training, residency) checked against the provider's current terms and **approved by the owner**. Then set `disclosure.reviewedOn` and a new `disclosure.version`.
4. **Links:** key, billing and terms links open the right pages.
5. **Catalog PR:** `status: 'supported'`, `evaluation` filled for each model, recommended models set. The catalog test refuses `supported` without these.

## Current status

**Batch relevance evaluation (CI, 2026-10-06):** PASS for `relevance-batch/v2` on `gemini-3.5-flash-lite`, 44 cases, 1 run: relevance 100%, 0 critical misses, category 96%, p95 latency 2.6 s. Extraction was not part of this run. `AI_TRIAGE_BATCH_ENABLED` stays off until the owner decides to turn it on.

| Provider | Models (candidates) | Evaluation | Error mapping | Disclosure | Status |
| --- | --- | --- | --- | --- | --- |
| Google Gemini | gemini-2.5-flash-lite (fast), gemini-2.5-flash (fast, detailed), gemini-3.5-flash-lite (fast), gemini-3.8-flash (fast, detailed) | Batch relevance PASS on gemini-3.5-flash-lite (CI, 2026-10-06); full evaluation not run | "Model not available to the key" observed live: 404 `NOT_FOUND` → `MODEL_UNAVAILABLE` (2.5 models, new key). Other rows not confirmed live | Draft, needs owner approval | hidden |
| OpenAI | gpt-5-nano (fast), gpt-5-mini (fast, detailed) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |
| Anthropic Claude | claude-haiku-4-5-20251001 (fast, detailed; tool mode), claude-sonnet-5-5 (detailed; native JSON schema) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |

**Re-evaluate when:**
- a model is added or becomes the recommended one;
- a contract version changes;
- an adapter's request shape changes;
- an SDK major version changes;
- a provider announces a deprecation or behavior change.
