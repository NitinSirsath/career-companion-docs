# Career Companion: MVP Architecture & Known Limitations

This document describes the Sprint 4.5 MVP with the 2026-09-26 stabilization corrections, updated for the Sprint 6 status/evidence boundary (§9, 2026-10-02) and automation submissions through MCP (§10, ADR-0002, 2026-10-02). See the [audit evidence and roadmap gates](../engineering/stabilization-audit.md) and backend [release/recovery guide](https://github.com/NitinSirsath/career-companion-backend/blob/main/STABILIZATION.md). Local verification does not establish live production readiness.

## 1. Authentication & Session Boundary
- **Implementation:** The primary authentication mechanism is Google OAuth via the `google-auth-library`.
- **Session:** Authenticated sessions are managed securely using `express-session` with `connect-pg-simple`. Sessions are stored in the database (`cc_session` cookie).
- **Security:** All authenticated routes are strictly scoped to the `req.session.userId`.
- **Note:** Production startup rejects development authentication, weak/default signing secrets and non-HTTPS configured origins/redirects. The auth middleware independently refuses production header impersonation. Google email claims must be verified; an existing different Google identity cannot be silently replaced. Login rotates and persists the session. TLS-terminating deployments must configure the actual trusted proxy count.

## 2. Gmail Integration & Privacy
- **Metadata vs. Body:** The system stores sender, subject, message/thread IDs and timestamps. Snippets are transient classification inputs. The body is fetched on demand for Relevant or Uncertain mail and bounded to 8000 characters for extraction.
- **Privacy:** Raw email bodies are **never** persisted in the database and are **never** returned to the frontend. They are passed strictly in-memory to the AI provider and then discarded. Gmail access tokens and refresh tokens are encrypted at rest using AES-256-GCM.
- **Token Refresh:** A shared OAuth client wrapper persists refreshed encrypted credentials conditionally; an old request cannot resurrect a disconnected connection. Authentication failures revoke the connection; quota-related 403 responses do not.
- **Sync Architecture:** `POST /api/gmail/sync` returns 202 after a durable queue handoff. A user-scoped database claim/lease excludes concurrent scans and allows stale-sync recovery. Initial sync captures history before scanning the configured INBOX lookback; later scans cover since the previous successful sync with a one-hour overlap, capped at 30 days. A capped scan persists an unscanned-gap notice; subsequent syncs consume history pages, falling back to full scan on expired history. Checkpoints advance only after ingestion/enqueueing succeeds. Stored messages are deduplicated; pending insert/enqueue gaps are recovered in bounded batches. The UI polls status and processing state.
- **Identity:** One mailbox is supported per user. Reconnecting that mailbox is allowed; changing mailbox identity is rejected until a separately designed migration exists.
- **Retention:** Raw bodies/snippets are not stored. Metadata, structured AI checkpoints and domain records persist until parent deletion. Disconnect clears tokens but retains existing domain data. A time-based retention/deletion policy remains a product decision.

## 3. AI Pipeline (User-Provided AI, ADR-0001)
- **Whose AI:** each user brings their own AI account from a curated, code-defined catalog: Google Gemini, OpenAI or Anthropic Claude. There is no Career Companion key and no automatic fallback. Each provider is offered only after it passes Career Companion's synthetic evaluation and its data-use text is approved ([provider evaluation](../ai/provider-evaluation.md)).
- **Provider-neutral contracts:** Career Companion owns the prompts and Zod schemas (`classification/v2`, `extraction/v2`). One adapter per API protocol translates them, and every result is validated with the same schema.
- **Cost boundary:**
  - Deterministic filtering precedes classification.
  - Each classification/extraction uses a unique `(emailId, operation, version)` ledger, a claim committed before the call (recording provider and model), and a validated completion checkpoint.
  - A **per-user** daily safety limit and a **per-user** provider cooldown apply. One user never affects another.
  - SDK retries are disabled. Provider refusals never use attempts.
- **Uncertainty:** a crash or timeout around a provider call cannot prove whether it was charged. Unknown claims are held and never released automatically; the user may approve exactly one more attempt ("Retry anyway"). Completed work is adopted; downstream matching failure does not replay AI.
- **Waiting:** without usable AI access, emails wait as `PENDING` and resume on the next sync or after the user fixes access.

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
- AI providers beyond the V1 catalog (Gemini, OpenAI, Claude); further OpenAI-compatible providers are catalog plus evaluation work (ADR-0001).
- Gmail Push Notifications (Pub/Sub) — Polling/sync is sufficient for V1.
- Redis/Kafka — PostgreSQL via `pg-boss` handles all queuing needs.
- Transactional Outbox pattern — Simple fire-and-forget job enqueueing is accepted for non-critical notifications.
- Microservices, event sourcing, or CQRS.
- Billing, teams, or enterprise SSO.

## 8. Reliability & Job Processing
- **Queueing:** `pg-boss` runs email processing, Gmail sync and Discord jobs directly in Postgres. Callers share an initialization promise. Queue singleton windows reduce duplicate jobs; database claims protect actual effects.
- **Retries:** Queue retries are bounded and delayed. AI attempts and provider cooldown are separately durable. Terminal errors are acknowledged without replay; uncertain provider work is held for operator reconciliation. Exhausted pending work is not revived automatically.
- **Pagination:** All public lists use the existing offset envelope, default/max 20, validated parameters, and stable ID tie-breakers. Sublists and application selectors are paginated. Details load by owned ID independently of list pages. Query invalidation, empty-page navigation and mutation errors are covered by UI tests; concurrent changes can still shift offset pages.

## 9. Application Status and Evidence Boundary (Sprint 6)
- **One read rule:** `ApplicationService` has a single response mapper used by create, list, detail and the status PATCH. It returns the stored `aiStatus`/`userStatus` plus derived `effectiveStatus` (`userStatus ?? aiStatus`), `statusSource` (`USER`/`AI`/`UNKNOWN`), `hasStatusConflict` and the stored `userStatusRevision`. Derived fields are not persisted.
- **One write path for manual state:** `PATCH /api/applications/:id/status` with a strict body `{ userStatus, expectedUserStatusRevision }`. One short transaction locks only the owned application row (`id` + authenticated `userId`), compares the revision before no-op detection, writes only manual fields and returns the canonical response from the same transaction. Errors: 400 `VALIDATION_ERROR`, 401, 404 `NOT_FOUND` (missing and foreign are identical), 409 `STATUS_CONFLICT`, sanitized 500. No email lock, external call, job, event, action or AI-ledger change. Matcher keeps email → application locking and writes only `aiStatus`.
- **Evidence selection:** events and each list/detail `recentEvent` add `recordedAt` and a bounded `sourceEmail` (`id`, `subject`, `sender`, `receivedAt`) selected with the page's query, never per event. Source ownership is checked in addition to the parent; a foreign source (only possible through inconsistent legacy data) is returned as null with its `emailId` nulled and a sanitized `evidence_ownership_mismatch` log. No body, snippet, extraction payload or prompt is returned, and reads never call Gmail or AI.
- **Runtime contracts:** the frontend parses create/list/detail/PATCH and events responses with the synchronized backend Zod schemas. Missing required keys or inconsistent derived fields are contract errors, never defaults. A newer client against an older backend shows a recoverable contract error.
- **Remaining limits:** no event occurrence time or agenda (email dates are unrestricted and may come from a Date header), no rematching/merging, no action source evidence (D1 deferred), no correction audit history, monotonic AI inference can still hold a wrong terminal AI state (manual correction is the remedy).

## 10. Automation Submissions through MCP (ADR-0002)
- **Boundary:** the backend hosts one Streamable HTTP MCP server at `POST /mcp` (`backend/src/mcp/`), a thin adapter over the domain service `services/externalSubmission.ts`. One write-only, idempotent tool, `record_application_submission`; no resources, prompts or read tools. The user's own automation agent is the client and pushes one call per confirmed submission. Career Companion never reads the laptop and never applies to jobs.
- **Middleware order:** `/mcp` is mounted before CORS, the global JSON parser, cookies and session. Per request: one log line → POST only (405) → `Host` allowlist (`MCP_ALLOWED_HOSTS`, 403) → `Origin` allowlist (`MCP_ALLOWED_ORIGINS`, default empty; no `Origin` passes) → Bearer integration token (401 with `WWW-Authenticate`) → 32 KB JSON (413/400 as JSON-RPC errors, no stack) → SDK handler.
- **SDK wiring:** `@modelcontextprotocol/server` 2.2.0 and `/node` 2.1.0, pinned. The Express add-on is not used because it re-types `req.auth` app-wide; the runtime-neutral helpers (`validateHostHeader`, `validateOriginHeader`, `verifyBearerToken`, `bearerAuthChallengeResponse`, `createMcpHandler`, `toNodeHandler`) are used directly, and the verified `AuthInfo` is passed to the SDK on a request view, so the session `req.auth` is never touched. Input is validated in the tool handler with the strict schema, so every failure is a tool error with code `invalid_input` and field names only.
- **Token auth:** per-user `ccmcp_` tokens (SHA-256 hashed, single scope, expiring, revocable) are accepted only on `/mcp`; sessions and the development header are never accepted there, and tokens never work on `/api`. The user always comes from the token.
- **Intake:** one transaction per call under `pg_advisory_xact_lock(namespace, hashtext(userId))`: ref check (repeat → `already_recorded`), daily cap (`MCP_DAILY_SUBMISSION_LIMIT`, default 500), insert, conservative match, create or link with the `AUTOMATION_SUBMITTED` event. User review (`/api/submissions/pending`, `/api/submissions/:id/resolve`) takes the same lock. No AI call, no queue, no new infrastructure.
- **Audit:** one `mcp_request` log line per request (token ID, user ID, method, tool, outcome, duration, whether a replay differed); no payload values and no token material.
- **Remaining limits:** until deployment the endpoint only works with Career Companion on the same machine as the agent; production needs edge routing for `/mcp` and `MCP_ALLOWED_HOSTS` set to the `Host` the backend receives. There is no per-IP rate limiter (deployment work). Antigravity compatibility is verified by the owner (MCP-09 part B).

