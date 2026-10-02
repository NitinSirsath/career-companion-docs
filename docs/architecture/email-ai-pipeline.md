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
- **Provider-neutral contracts:** \`classification/v2\` and \`extraction/v2\` (prompt, Zod schema, role, input bounds) are owned by Career Companion. Each adapter derives its schema dialect from Zod. Every result is validated with the same Zod schema.
- **Access resolution:**
  - lazy, at most once per job, after completed-result adoption and the deterministic filter;
  - without usable access the email returns to \`PENDING\`, with no error fields, an acknowledged job (its delivery withdrawn), and no attempt used.
- **Per-user limits:** the claim reserves one call against the user's daily safety limit (\`AI_USER_DAILY_CALL_LIMIT\`) and respects the user's cooldown. One user's limit or cooldown never affects another.
- **Refusals** (key rejected, account or billing, model unavailable, rate limit) release the claim and restore the attempt, and the user's access state records them. A rate limit pauses that user: retry-after bounded to 10 s–1 h, or 60 s doubling to 30 min.
- **Unknown outcome:** held as \`UNKNOWN\`. The email is \`FAILED\` at once, and that user is paused for 2 minutes (growing to 30).
- **User-approved retry:** each approval (\`approvedRetries\`) permits exactly one call beyond the attempt limit, for unknown, unusable, stale or exhausted operations.
- **Provenance:** each operation records the provider and model at claim time. \`AIProcessingResult.provider/model\` come from the operation that produced the result, so a result reused after a provider switch keeps its own provenance.
- **Resuming:** \`reofferPendingEmails\` (newest first, at most 100, only when access is ready) runs at the end of every sync and after a successful save or "Check again". There is no scheduler.
