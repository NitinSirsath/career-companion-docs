# Email AI Pipeline Architecture

This document describes the email intelligence pipeline introduced in Sprint 3 (COM-27).

## Two-Stage AI Pipeline

To process emails securely and cost-effectively, Career Companion employs a two-stage pipeline:

1. **Deterministic Pre-filter:** Before any AI call is made, basic Gmail labels (e.g. \`CATEGORY_PROMOTIONS\`, \`CATEGORY_SOCIAL\`, \`SPAM\`) are used to immediately filter out irrelevant emails without fetching the email body.
2. **Relevance Classifier:** the user's *fast* model (from their own AI provider; for example Gemini 2.5 Flash-Lite) is used on metadata (subject, sender, snippet) to classify the relevance of the email.
3. **Email Analyzer:** If an email is classified as \`RELEVANT\` or \`UNCERTAIN\`, the system securely fetches a bounded email body (up to 8,000 characters) and uses the user's *detailed* model to extract structured job application data.

## Relevance Threshold

A deterministic confidence threshold (configurable via \`RELEVANCE_CONFIDENCE_THRESHOLD\`, defaulting to 0.7) determines if the relevance classification is trustworthy.
- If the model is confident (\`>= 0.7\`), the decision (\`RELEVANT\` or \`IRRELEVANT\`) is accepted.
- If the model is not confident (\`< 0.7\`), the decision is forced to \`UNCERTAIN\`, causing the system to proceed with fetching the body and analyzing it for safety.

## Email Categories

When an email is deemed relevant, it is categorized into one of the strongly typed values:
\`RECRUITER\`, \`INTERVIEW\`, \`ASSESSMENT\`, \`OFFER\`, \`REJECTION\`, \`FOLLOW_UP\`, \`NEWSLETTER\`, or \`SPAM\`.

## Structured Extraction

The Email Analyzer extracts structured data corresponding to the job application lifecycle:
- **Company & Role:** \`companyName\`, \`jobTitle\`
- **Contact:** \`recruiterName\`, \`recruiterEmail\`
- **Interview Details:** \`interviewStage\`, \`interviewType\`, \`interviewDate\`, \`interviewTime\`
- **Assessments:** \`assessmentInfo\`, \`assessmentDeadline\`
- **Outcomes:** \`offerInfo\`, \`rejectionInfo\`
- **Next Steps:** \`actionRequired\`, \`requestedAction\`, \`actionDeadline\`, \`followUpRequired\`, \`followUpDate\`

All missing fields are explicitly set to \`null\`. The model must not hallucinate missing values.

## AIProcessingResult & State Separation

The AI processing result is persisted in the \`AIProcessingResult\` table, decoupled from application state matching. It records:
- Provider, model, and contract versions.
- The raw decision, category, and structured extracted data.
- Extraction and relevance confidence.
- Error diagnostics (if failed).

Processing lifecycle (\`PENDING\`, \`PROCESSING\`, \`COMPLETED\`, \`FAILED\`) is distinct from the semantic meaning of the email (Relevant/Irrelevant) or its application match state. This prevents mixing infrastructure states with domain states.

## Privacy Boundary

- Raw email bodies are **never** logged or persisted.
- Prompts containing email content, raw provider responses, provider error text, AI keys and access tokens are strictly excluded from logs and storage.
- Safe structured metadata (e.g., duration, provider, error category) may be logged for diagnostics.

## Idempotency Behavior

> Updated 2026-10-02. The earlier "upsert means safe replay and overwrite" description is superseded by the stabilization/Sprint 5 operation ledger described below.

- `AIProcessingResult` still holds one current result per email (unique `emailId`), but provider calls are governed by the durable `ai_operations` ledger: a unique `(emailId, operation, version)` claim is committed **before** calling Gemini. Completed operations are reused without a new provider call; PROCESSING/UNKNOWN/FAILED or exhausted claims are held for operator reconciliation and are never reset automatically. Completed extraction results are adopted on replay, so a re-delivered job or repeat sync makes no additional paid call.
- A global daily call budget and provider cooldown are enforced in the same claim transaction across workers.
- Manual email retry (`POST /api/emails/:id/retry`) re-offers only safely resumable work. It never deletes operation claims or resets relevance/match state, refuses held claims with 409 `AI_OPERATION_REQUIRES_REVIEW`, reports a singleton-suppressed retry with 409 `RETRY_RECENTLY_QUEUED`, and does not write processing state itself (the worker alone moves the email to PROCESSING and its outcome).

## Worker Delivery (Sprint 6, S6-03)

- The email worker registers with `{ includeMetadata: true, batchSize: 1 }` on installed pg-boss 12.31.0 (whose default batch size is already 1). The callback rejects any unexpected multi-job delivery so pg-boss retries every delivered job; no job is acknowledged without an attempted outcome.
- Each delivered job ends with an attributable outcome logged as `job_completed` or `job_failed` with `outcome` = `completed`, `retry_scheduled`, `failed_terminal` or `failed_exhausted`, plus `retryCount/retryLimit`. A failure at the final permitted delivery marks the email FAILED. Stored error details are application-authored AI messages or a fixed safe text; raw error text is never persisted.
- The historical claim that `jobs[0]` silently drops batches was not reproducible under the installed single-job configuration and is not a demonstrated defect. Final-attempt hard-kill can still leave an email PROCESSING (operator limitation retained).

## User-provided AI (ADR-0001, implemented 2026-10-02)

- **Whose AI:** every call uses the job user's own provider configuration (Gemini, OpenAI or Claude, from the code catalog). There is no Career Companion key and no fallback to another provider.
- **Provider-neutral contracts:** \`classification/v2\`, \`extraction/v2\` and \`relevance-batch/v1\` (prompt, Zod schema, role, input bounds) are owned by Career Companion. Each adapter derives its schema dialect from Zod. Every result is validated with the same Zod schema.
- **Access resolution:**
  - lazy, at most once per job, after completed-result adoption and the deterministic filter;
  - without usable access the email returns to \`PENDING\`, with no error fields, an acknowledged job (its delivery withdrawn), and no attempt used.
- **Per-user limits:** the claim reserves one call against the user's daily safety limit (\`AI_USER_DAILY_CALL_LIMIT\`) and respects the user's cooldown. One user's limit or cooldown never affects another.
- **Refusals** (key rejected, account or billing, model unavailable, rate limit) release the claim and restore the attempt, and the user's access state records them. A rate limit pauses that user: retry-after bounded to 10 s–1 h, or 60 s doubling to 30 min.
- **Unknown outcome:** held as \`UNKNOWN\`. The email is \`FAILED\` at once, and that user is paused for 2 minutes (growing to 30).
- **User-approved retry:** each approval (\`approvedRetries\`) permits exactly one call beyond the attempt limit, for unknown, unusable, stale or exhausted operations.
- **Provenance:** each operation records the provider and model at claim time. \`AIProcessingResult.provider/model\` come from the operation that produced the result, so a result reused after a provider switch keeps its own provenance.
- **Resuming:** \`reofferPendingEmails\` (newest first, at most 100, only when access is ready) runs at the end of every sync and after a successful save or "Check again". There is no scheduler.

## Batched relevance triage (COM-125)

When `AI_TRIAGE_BATCH_ENABLED` is on, initial relevance triage can run in batches. It is off by default.

- `AI_TRIAGE_BATCH_ENABLED`: exactly `true` turns it on. Unset, empty or `false` keeps it off. Any other value fails startup.
- `AI_TRIAGE_BATCH_SIZE`: default 20. Allowed range is 1-25.
- Owner decisions on 2026-10-05:
  - **D1:** send sender, subject, Gmail labels and preview, with preview limited to 1,000 characters.
  - **D2:** use batch size 20, with 1-25 allowed.
  - **D3:** a failed or timed-out batch holds all its emails for user approval.

The batch contract is `relevance-batch/v2` (strict rules, COM-133); emails that started on `relevance-batch/v1` stay on it. It uses the user's fast model and allows at most 4,096 output tokens.

Each item has a key `e1` through `eN`, bounded sender, subject, labels and preview. Sender is limited to 512 characters, subject to 1,000, labels to 30 and preview to 1,000. No Gmail or database IDs are sent. Each item is untrusted content.

Results are matched by key. A missing, duplicate, unknown-key or invalid answer never marks an email `IRRELEVANT`. Missing or duplicate results are undecided. The attempt is used and the email is tried again later. After 3 attempts, the email is `FAILED` and the user can approve one retry.

A whole-batch failure is handled as follows:
- A refusal releases every claim, restores the attempt and leaves emails `PENDING`.
- Invalid output makes all emails `FAILED`. The user can approve a retry.
- Invalid request makes all emails `FAILED` for operator review.
- Timeout or unknown outcome makes all emails `FAILED`. The user can approve a retry.
- Unknown outcomes are never resent automatically.
- An approved retry uses the existing `POST /api/emails/:id/retry` route and runs one batch-of-one call.

The `relevance-triage-job` queue uses pg-boss policy `stately` with key `triage:<userId>`. There can be at most one queued and one running triage job per user. A job starts 10 seconds after it is queued. A run stops starting new batches after 180 seconds and queues a follow-up. Job expiry is 300 seconds and the job has 3 retries.

With the flag on, sync and re-offer route emails with no AI result and no classification record to triage. Other emails stay on the per-email job. The Promotions, Social and Spam filter still runs first with no AI call. If Gmail metadata fails for one email, that email goes to the per-email job. An email already classified by a batch keeps that result if the flag is later turned off.

`RELEVANT` and `UNCERTAIN` emails go to the existing extraction step with no second classification call. Extraction is unchanged and uses one call per job email.

Logs hold IDs and counts only.

## Strict relevance rules (COM-133, 2026-10-06)

**Relevant** means a person or company is communicating with the user about a specific role or the user's application:
- a recruiter, hiring manager or company reaching out;
- interview communication, or an assessment or test invitation;
- an application confirmation or status update, a rejection or an offer;
- a follow-up about the user's application;
- a specific job opportunity sent to the user personally.

**Irrelevant** includes every platform-generated job discovery email: job alerts, recommended jobs ("jobs you may like", "you may be a fit"), sponsored or promoted jobs, generic job listings, and marketing or newsletters from job platforms. Personal and transactional email stays Irrelevant.

**LinkedIn:** a message or InMail where a named person reaches out about a specific role is Relevant. LinkedIn job alerts, recommendations and sponsored jobs are Irrelevant. The model decides from content, not only the sender address.

Contracts: `classification/v3` (per-email) and `relevance-batch/v2` (batch) are current. `classification/v2` and `relevance-batch/v1` are frozen for emails that started on them.

- **Categories:** the strict prompts offer only `RECRUITER`, `INTERVIEW`, `ASSESSMENT`, `OFFER`, `REJECTION` and `FOLLOW_UP`. `NEWSLETTER` and `SPAM` stay in the enum for older results.
- **Pre-filter:** under the strict rules, Gmail's Social label no longer drops mail from a linkedin.com sender; the AI decides. Promotions and Spam are unchanged. Emails on the older rules keep the old label rule.
- **Example models:** Gemini 3.5 Flash-Lite (fast) and Gemini 3.8 Flash (detailed). Google limits Gemini 2.5 models to projects that used them before.

### New mails only

Every automatic sync change applies to newly synced email only. Sync, deploys and re-offers never re-classify, re-process, backfill or reset older email. The one exception is a retry the user approves for a held email; it runs on that email's original contract version.

- Each email keeps the relevance contract version of its first classification record. An email with no classification record yet starts on the current version.
- The older prompts stay byte-for-byte unchanged (fingerprint tests) so emails that started on them finish on them, including after a user-approved retry.
- A batch never mixes versions, and an email is never claimed under a second version.
- Already-classified emails keep their results. Job alerts marked Relevant before COM-133 stay Relevant.
- Backend tests 46–51 in `src/tests/ai-triage.test.ts` enforce this. Do not weaken or remove them.

A future ticket will split Relevant into sections (direct contact, application events, job discovery). Until then, job discovery email is Irrelevant.
