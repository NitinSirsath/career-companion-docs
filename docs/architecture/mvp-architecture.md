# Career Companion: MVP Architecture & Known Limitations

This document reflects the finalized end-to-end (E2E) architecture of the Career Companion MVP upon completion of Sprint 4.5. It documents major design decisions, trade-offs, security boundaries, and explicit out-of-scope capabilities for the V1 MVP.

## 1. Authentication & Session Boundary
- **Implementation:** The primary authentication mechanism is Google OAuth via the `google-auth-library`.
- **Session:** Authenticated sessions are managed securely using `express-session` with `connect-pg-simple`. Sessions are stored in the database (`cc_session` cookie).
- **Security:** All authenticated routes are strictly scoped to the `req.session.userId`.
- **Note:** The `ENABLE_DEV_AUTH` flag exists solely for local testing convenience. It is disabled in production, enforcing the full OAuth boundary.

## 2. Gmail Integration & Privacy
- **Metadata vs. Body:** The system ingests basic metadata (sender, subject, snippet) via background jobs to detect job-related emails. The full email body is **only** fetched on-the-fly when the AI relevance classifier deems it necessary (Relevant or Uncertain).
- **Privacy:** Raw email bodies are **never** persisted in the database and are **never** returned to the frontend. They are passed strictly in-memory to the AI provider and then discarded. Gmail access tokens are encrypted at rest using AES-256-GCM.

## 3. AI Pipeline (Gemini Only)
- **Why Gemini Only?** Gemini (specifically 2.5 Flash and Flash-Lite) is the sole implemented AI provider for MVP. This choice minimizes infrastructure complexity while offering excellent multimodal/structured output capabilities out of the box via Google's official GenAI SDK.
- **Data Minimization:** Email bodies sent to Gemini are truncated (e.g., max 8000 characters) to avoid context bloat and minimize token exposure. The AI pipeline is heavily abstracted, meaning it is technically possible to support Anthropic or OpenAI later if needed.

## 4. Deterministic Matching & State Inference
- **Matching:** The system deterministically matches processed emails to existing applications based on exact `companyName` string matches (after normalization).
- **Ambiguity:** If an email could belong to multiple applications (or if the company name cannot be securely matched), it enters an `AMBIGUOUS` state. The user manually resolves this on the frontend, enforcing human-in-the-loop accuracy.
- **State Inference:** State transitions (e.g., `APPLIED` → `INTERVIEW`) are strictly monotonic. The system will not automatically downgrade an application's state (e.g., moving from `OFFER` back to `ASSESSMENT`).

## 5. Action & Follow-up Management
- Actions (e.g., "Schedule Interview", "Sign Offer") are created idempotently alongside `ApplicationEvent` records.
- Actions have explicit deadlines and statuses (`PENDING`, `COMPLETED`, `DISMISSED`) which the user can manage directly on the frontend.

## 6. Discord Notifications & Reliability
- **Why Discord Only?** Discord via webhooks is the easiest and most reliable notification channel for MVP, requiring no complex third-party API approval or app installation flow (unlike Slack or push notifications).
- **Idempotency:** A `NotificationDelivery` table ensures that an action is never notified to Discord more than once.
- **Known Limitation (Action → Queue Enqueue Window):** The system persists an Action to the PostgreSQL database inside a transaction, and *then* enqueues the notification job to `pg-boss`. If the process crashes immediately between DB persistence and `pg-boss` enqueueing, the notification will be lost. This lack of a transactional outbox is an accepted trade-off for the MVP to reduce architectural complexity.

## 7. Explicitly Deferred / Out of Scope for MVP
- Other Email providers (Outlook) or platforms (LinkedIn).
- Other Notification providers (Slack, WhatsApp, SMS).
- Other AI Providers (OpenAI, Anthropic).
- Gmail Push Notifications (Pub/Sub) — Polling/sync is sufficient for V1.
- Redis/Kafka — PostgreSQL via `pg-boss` handles all queuing needs.
- Transactional Outbox pattern — Simple fire-and-forget job enqueueing is accepted for non-critical notifications.
- Microservices, event sourcing, or CQRS.
- Billing, teams, or enterprise SSO.

## 8. Reliability & Job Processing
- **Queueing:** `pg-boss` manages all background tasks (`email-processing-job`, `discord-notification-job`) directly within Postgres, eliminating the need for Redis. Queues are initialized idempotently on startup.
- **Retries:** Exponential backoff is applied for transient errors (e.g., Discord rate limits or AI provider downtime). Terminal errors (e.g., invalid tokens) fail the job immediately.
