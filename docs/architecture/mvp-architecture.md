# Career Companion: MVP Architecture & Known Limitations

This document describes the Sprint 4.5 MVP with the 2026-09-26 stabilization corrections. See the [audit evidence and roadmap gates](../engineering/stabilization-audit.md) and backend [release/recovery guide](https://github.com/NitinSirsath/career-companion-backend/blob/main/STABILIZATION.md). Local verification does not establish live production readiness.

## 1. Authentication & Session Boundary
- **Implementation:** The primary authentication mechanism is Google OAuth via the `google-auth-library`.
- **Session:** Authenticated sessions are managed securely using `express-session` with `connect-pg-simple`. Sessions are stored in the database (`cc_session` cookie).
- **Security:** All authenticated routes are strictly scoped to the `req.session.userId`.
- **Note:** Production startup rejects development authentication, weak/default signing secrets and non-HTTPS configured origins/redirects. The auth middleware independently refuses production header impersonation. Google email claims must be verified; an existing different Google identity cannot be silently replaced. Login rotates and persists the session. TLS-terminating deployments must configure the actual trusted proxy count.

## 2. Gmail Integration & Privacy
- **Metadata vs. Body:** The system stores sender, subject, message/thread IDs and timestamps. Snippets are transient classification inputs. The body is fetched on demand for Relevant or Uncertain mail and bounded to 8000 characters for extraction.
- **Privacy:** Raw email bodies are **never** persisted in the database and are **never** returned to the frontend. They are passed strictly in-memory to the AI provider and then discarded. Gmail access tokens and refresh tokens are encrypted at rest using AES-256-GCM.
- **Token Refresh:** A shared OAuth client wrapper persists refreshed encrypted credentials conditionally; an old request cannot resurrect a disconnected connection. Authentication failures revoke the connection; quota-related 403 responses do not.
- **Sync Architecture:** `POST /api/gmail/sync` returns 202 after a durable queue handoff. A user-scoped database claim/lease excludes concurrent scans and allows stale-sync recovery. Initial sync captures history before scanning the existing 90-day INBOX scope; subsequent syncs consume history pages, falling back to full scan on expired history. Checkpoints advance only after ingestion/enqueueing succeeds. Stored messages are deduplicated; pending insert/enqueue gaps are recovered in bounded batches. The UI polls status and processing state.
- **Identity:** One mailbox is supported per user. Reconnecting that mailbox is allowed; changing mailbox identity is rejected until a separately designed migration exists.
- **Retention:** Raw bodies/snippets are not stored. Metadata, structured AI checkpoints and domain records persist until parent deletion. Disconnect clears tokens but retains existing domain data. A time-based retention/deletion policy remains a product decision.

## 3. AI Pipeline (Gemini Only)
- **Why Gemini Only?** Gemini (specifically 2.5 Flash and Flash-Lite) is the sole implemented AI provider for MVP. This choice minimizes infrastructure complexity while offering excellent multimodal/structured output capabilities out of the box via Google's official GenAI SDK.
- **Cost boundary:** Deterministic filtering precedes classification. Each classification/extraction uses a unique `(emailId, operation, version)` ledger, a committed claim before calling Gemini, and a validated completion checkpoint. A global daily call ceiling and cooldown apply across workers. SDK retries are disabled; only explicit transient rejections permit bounded retries.
- **Uncertainty:** A crash or timeout around a provider call cannot prove whether it was charged. Processing/unknown claims require reconciliation and are never automatically released. This can hold work, but prevents silent repeat charges. Completed legacy work is adopted; downstream matching failure does not replay AI.
- **Provider scope:** The existing abstraction remains, with Gemini only. No additional providers or generic workflow system were added.

## 4. Deterministic Matching & State Inference
- **Matching:** The system deterministically matches processed emails to existing applications based on exact `companyName` string matches (after normalization).
- **Ambiguity:** If an email could belong to multiple applications (or if the company name cannot be securely matched), it enters an `AMBIGUOUS` state. The user manually resolves this on the frontend, enforcing human-in-the-loop accuracy.
- **State Inference:** State transitions (e.g., `APPLIED` → `INTERVIEW`) are strictly monotonic. The system will not automatically downgrade an application's state (e.g., moving from `OFFER` back to `ASSESSMENT`).
- **Integrity:** Email/application row locks serialize domain effects in one transaction. Database constraints reject duplicate email-derived events/actions and cross-owner links. Explicit user choices survive AI replay; ownership cannot be transferred implicitly.

## 5. Action & Follow-up Management
- Actions (e.g., "Schedule Interview", "Sign Offer") are created idempotently alongside `ApplicationEvent` records.
- Actions have explicit deadlines and statuses (`PENDING`, `COMPLETED`, `DISMISSED`) which the user can manage directly on the frontend.

## 6. Discord Notifications & Reliability
- **Why Discord Only?** Discord via webhooks is the easiest and most reliable notification channel for MVP, requiring no complex third-party API approval or app installation flow (unlike Slack or push notifications).
- **Delivery boundary:** A `NotificationDelivery` claim is committed before sending. Duplicate executions and uncertain successes do not automatically resend. Only explicit retryable rejections release the claim. This is not an exactly-once delivery guarantee.
- **Recipient boundary:** The global webhook is bound to `DISCORD_USER_ID`; delivery for every other owner is skipped. No multi-user notification configuration feature was added.
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
- **Queueing:** `pg-boss` runs email processing, Gmail sync and Discord jobs directly in Postgres. Callers share an initialization promise. Queue singleton windows reduce duplicate jobs; database claims protect actual effects.
- **Retries:** Queue retries are bounded and delayed. AI attempts and provider cooldown are separately durable. Terminal errors are acknowledged without replay; uncertain provider work is held for operator reconciliation. Exhausted pending work is not revived automatically.
- **Pagination:** All public lists use the existing offset envelope, default/max 20, validated parameters, and stable ID tie-breakers. Sublists and application selectors are paginated. Details load by owned ID independently of list pages. Query invalidation, empty-page navigation and mutation errors are covered by UI tests; concurrent changes can still shift offset pages.
