# S6-R06 — Worker error hygiene

| Field | Value |
| --- | --- |
| Status | **Fixed — verified 2026-10-02** (done as the BYO AI entry gate, AI-00) |
| Severity / priority | Low / follow-up |
| Source | [Review 2026-10-02](README.md) L2, L3; S5-05 |
| Repository | career-companion-backend (`src/jobs/emailProcessingJob.ts`) |

## Problem
1. **Raw errors stored in pg-boss.** `if (!terminal) throw err;` (~line 114) rethrows the original error. pg-boss 12.31.0 (`manager.js` `fail(name, jobIds, err)`) stores the serialized error (message, stack, enumerable properties such as Prisma `meta`) in `pgboss.job.output`. `describeFailure` sanitizes only the `emails` row. Whether real email content reaches this column is unconfirmed, but the boundary is not enforced.
2. **"Outcome unknown" shown as retrying.** `retryable: !(err instanceof TerminalAIError)` marks `AIProviderError(..., false)` ("outcome unknown", `GeminiProvider.ts`) as `processingRetryable: true`. The UI briefly shows "Retrying (Rate Limited)" until the next attempt hits the held UNKNOWN operation and marks the email FAILED.

## Fix
Throw a sanitized error to pg-boss (for example `new Error(failure.category)`) and keep the original only in memory. Use `err.isRetryable` for AI provider errors.

## Acceptance criteria
- [x] `pgboss.job.output` for a failed email job contains only the safe category.
- [x] An outcome-unknown error is stored as not retryable, while retry semantics and final-attempt FAILED behaviour are unchanged.

## Verification
`email-worker-reliability.test.ts` (including the real-queue lane, asserting job output), full backend suite, smoke.

## Resolution (2026-10-02)

- `emailProcessingJob.ts`: non-terminal failures now throw `EmailJobFailure(category)` to pg-boss. Its message is the safe category and its stack is `EmailJobFailure: <category>`, so `pgboss.job.output` holds `{ name, message, stack }` with no original text, stack or properties. The original error stays in memory only.
- `describeFailure` uses `err.isRetryable` for AI provider errors, so "outcome unknown" is stored as `processingRetryable: false`. Retry scheduling and the final-attempt `FAILED` rule are unchanged.
- Tests (`email-worker-reliability.test.ts`): a unit test for the thrown error, a unit test for the outcome-unknown flag, and the real-queue lane now asserts the exact stored output for an exhausted job and for a job that threw private text, and scans `pgboss.job.output` for it. Backend 238/238, typecheck clean, lint 0 errors.
