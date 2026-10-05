# AI provider evaluation and certification

| Field | Value |
| --- | --- |
| Status | **No provider certified yet (2026-10-02).** All three catalog providers are `hidden`: usable in development and tests, never offered in production. |
| Decision | [ADR-0001](../architecture/decisions/ADR-0001-user-provided-ai.md) decision 2; [BYO AI plan §7](../planning/byo-ai/README.md#7-provider-evaluation-plan) |
| Code | backend `src/eval/ai/` (dataset, scorer, runner); catalog `src/contracts/aiCatalog.ts` |

A provider/model pair is offered to users only after it passes Career Companion's synthetic evaluation, its error mapping is confirmed with a real key, and the owner approves its data-use text.

## How to run an evaluation

```bash
AI_EVAL_API_KEY=<key in the shell only> npm run ai:eval -- --provider gemini --fast gemini-2.5-flash-lite --detailed gemini-2.5-flash --runs 2
```

- Run from the backend folder.
- Only catalog models are accepted. Add a candidate as a `hidden` catalog entry first.
- No database, Gmail or ledger is touched. The runner refuses to start when `NODE_ENV=production`.
- A report is written to `src/eval/ai/reports/<date>_<provider>_<fast>_<detailed>.json`. It contains synthetic data, metrics and error kinds only; commit it.
- Compare against the Gemini baseline with `--baseline <report.json>`.

## Dataset

41 hand-written synthetic emails in `src/eval/ai/dataset/`:
- recruiter 4, application received 3, interview 6, assessment 4, offer 3, rejection 4, follow-up 3, job alerts 3, irrelevant 7, adversarial 4 (prompt injection, non-English, long thread, spam posing as a recruiter);
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

A batch triage evaluation is **NOT RUN**. `AI_TRIAGE_BATCH_ENABLED` stays off until a PASS is recorded here.

| Provider | Models (candidates) | Evaluation | Error mapping | Disclosure | Status |
| --- | --- | --- | --- | --- | --- |
| Google Gemini | gemini-2.5-flash-lite (fast), gemini-2.5-flash (fast, detailed) | Not run (baseline pending a real key) | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |
| OpenAI | gpt-5-nano (fast), gpt-5-mini (fast, detailed) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |
| Anthropic Claude | claude-haiku-4-5-20251001 (fast, detailed; tool mode), claude-sonnet-5-5 (detailed; native JSON schema) | Not run | Starting table, unit-tested; not confirmed live | Draft, needs owner approval | hidden |

**Re-evaluate when:**
- a model is added or becomes the recommended one;
- a contract version changes;
- an adapter's request shape changes;
- an SDK major version changes;
- a provider announces a deprecation or behavior change.
