# BYO AI — execution report (2026-10-02)

Implemented directly in the local repositories, per the owner's instruction: no git, branches or worktrees. The baseline is the completed Sprint 6 filesystem.

## Outcome

- Users bring their own Gemini, OpenAI or Claude key. AI runs only on that key, with per-user limits, cooldowns and waiting.
- All three providers are `hidden`: usable in development and tests, **not offered in production** until certified ([provider evaluation](../../ai/provider-evaluation.md)).
- **Evidence:**
  - backend 467 tests;
  - frontend 118 tests;
  - typecheck, lint (0 errors) and build pass on both;
  - the migration's fresh and upgrade lanes pass;
  - the real-worker browser smoke passes with four new BYO AI scenarios.

## Done

| Issue | Result |
| --- | --- |
| AI-00 | S6-R06 implemented: pg-boss stores only `EmailJobFailure(<category>)`; an unknown outcome is stored as not retryable. |
| AI-01 | Prompts moved byte-for-byte into `AI_CONTRACTS`. Contract fingerprints are pinned. The derived Gemini schema equals the old hand-written one. `GeminiProvider` was replaced by adapters behind `createProviderClient`. |
| AI-02 | 41 synthetic cases (reserved domains enforced), scorer, `npm run ai:eval`. |
| AI-03 | Code catalog shared with the frontend. A supported provider requires an evaluation and an approved disclosure (tested). |
| AI-04 | `ProviderFailure` kinds, with no provider text and no `cause`; table-tested mapping per adapter; content-free verification. |
| AI-05 | `ai_configurations`, `ai_usage_days`, `ai_operations.provider/model/approvedRetries`. AES-256-GCM sealing bound to the user, with a dedicated key. |
| AI-06–AI-10 | Per-user safety limit (default 500), counts, cooldown and refusal rule. Access from the user's configuration only. Waiting as `PENDING`. Settings API (verify before save). Sample test. User-approved retry. |
| AI-11–AI-12 | `/ai` page; AI notice; "Waiting for AI" (not polled); "Retry anyway" dialog; provenance labels. |
| AI-13–AI-14 | OpenAI-compatible and Anthropic adapters (strict schema dialect, tool or native output), pinned SDKs `openai@7.27.0` and `@anthropic-ai/sdk@0.131.0`. |
| AI-16 | End-to-end redaction test: a sentinel key and email text pushed through every path, scanning logs, responses and every table. It was shown to catch an injected leak. Credential-boundary test. Smoke scenarios. |

## Changes from the plan (engineering decisions made while implementing)

- **Access from configuration earlier:** reading access from the user's configuration moved into AI-06. Per-user cooldowns and refusals need a configuration row, so there was no interim hosted-key state.
- **Waiting deliveries are withdrawn:** a job that only waited for AI is cancelled in pg-boss. pg-boss keeps a 5-minute singleton slot per email for every non-cancelled job, which would otherwise block re-offering an email right after the user sets up AI or a short cooldown ends. Proven with the real queue.
- **Approved retries get their own job key** (`<user>-<email>-approved-<n>`). The failed delivery still holds the email's slot. Approvals cannot race, and held emails get no other jobs.
- **`KEY_UNREADABLE` access issue:** a stored key that cannot be decrypted (for example a lost encryption key) shows "Enter your API key again" instead of failing emails.
- **`offeredProviders` in the settings response:** the server decides which providers are offered. Hidden providers are refused in production.
- **Locations:**
  - evaluation code is in `backend/src/eval/ai/`, so typecheck, lint and tests cover it;
  - `reofferPendingEmails` lives in `services/gmailSync.ts`;
  - event provenance is an event-level `analyzedBy`, leaving the Sprint 6 `sourceEmail` fields unchanged.
- **Smoke:** the safety limit is 30, and screenshots are opt-in with `SMOKE_SCREENSHOTS=<dir>`.

## Not done / not verified

- **No live provider call has been made.**
  - The Gemini baseline evaluation, live error-mapping confirmation and certification of all three providers need real keys (AI-15).
  - OpenAI and Claude model IDs are candidates until evaluated.
- **Data-use text** for all three providers is a draft and **needs owner approval** before any provider is enabled in production.
- **Provider links** (key, billing, terms pages) must be checked in AI-15.
- **Single email worker:** the per-provider job-duration measurement waits for the live release run (AI-17).
- **AI-18** (drop `ai_call_budgets`) waits for one stable release.
