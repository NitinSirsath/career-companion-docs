# Email AI Pipeline Architecture

This document describes the email intelligence pipeline introduced in Sprint 3 (COM-27).

## Two-Stage AI Pipeline

To process emails securely and cost-effectively, Career Companion employs a two-stage pipeline:

1. **Deterministic Pre-filter:** Before any AI call is made, basic Gmail labels (e.g. \`CATEGORY_PROMOTIONS\`, \`CATEGORY_SOCIAL\`, \`SPAM\`) are used to immediately filter out irrelevant emails without fetching the email body.
2. **Relevance Classifier:** A lightweight AI model (Gemini 2.5 Flash Lite) is used on metadata (subject, sender, snippet) to classify the relevance of the email.
3. **Email Analyzer:** If an email is classified as \`RELEVANT\` or \`UNCERTAIN\`, the system securely fetches a bounded email body (up to 8,000 characters) and uses a more capable model (Gemini 2.5 Flash) to extract structured job application data.

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
- Prompts containing email content, Gemini raw responses, and access tokens are strictly excluded from logs.
- Safe structured metadata (e.g., duration, provider, error category) may be logged for diagnostics.

## Idempotency Behavior

The pipeline uses the \`emailId\` as a unique constraint in \`AIProcessingResult\`. Re-processing an email results in an \`upsert\` operation, preserving exactly one "current" AI result per email. This ensures safe retries and allows future schema/prompt upgrades to cleanly overwrite past results without duplication.
