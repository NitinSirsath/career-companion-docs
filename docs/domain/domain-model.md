# Career Companion — Domain Model

**Version:** 1.3
**Status:** Sprint 6, BYO AI and the MCP feature implemented (local engineering; see the execution reports)
**Last Updated:** 2026-10-02

### Version History / Implementation Notes
- **v1.3 (MCP feature, ADR-0002):** `IntegrationToken` and `ExternalSubmission` (§4.8), the `AUTOMATION_SUBMITTED` event, `submittedVia` on applications, and their ownership triggers. Automatic application creation is allowed for automation submissions only; Gmail never creates applications.
- **v1.2 (Sprint 6):** Canonical status read fields, the `userStatusRevision` manual-correction token and owned set/change/clear semantics (§4.5). Implemented history is recording-ordered processing events with bounded owned source-email evidence; no `occurredAt` exists in the implemented schema (§4.6 "Implemented history boundary").
- **v1.1 (Sprint 2):** Implemented `GmailConnection` and `Email` schemas. `GmailConnection` includes encrypted token fields and sync status. `Email` intentionally omits `snippet` and `body` (privacy decision: only metadata headers `Subject`, `From`, `Date` are ingested); `threadId` is implemented as a dedicated column.

## 1. Purpose

This document defines the core domain entities, attributes, relationships, and invariants for Career Companion MVP.

The model is intentionally production-minded but limited to the current MVP. It distinguishes source evidence, AI interpretation, current application state, historical events, and user actions.

## 2. Domain Principles

1. **Email is evidence.** Gmail data is the source evidence from which job-search information is interpreted.
2. **AI output is not domain truth.** AIProcessingResult records AI interpretation and provenance; domain entities represent the structured job-search state used by the product.
3. **Application represents current state.** Historical occurrences belong to ApplicationEvent.
4. **ApplicationEvent represents what happened.** It should be traceable to source email evidence when available.
5. **Action represents what the user needs to do.** It has its own lifecycle and may become obsolete when later information changes the situation.
6. **User-confirmed state outranks AI-derived state.** AI processing must never silently overwrite an explicit user correction.
7. **Relevance and classification are different concepts.** Email relevance controls whether an email participates in job-search processing; AI classification describes the type of email.
8. **Multi-user isolation is mandatory.** User ownership must be enforced across relationships and queries.
9. **Minimize Gmail data.** Persist structured job-search information and only the email evidence required by the MVP; do not create unnecessary copies of email or AI output.
10. **Avoid premature normalization.** Company, JobPosting, Contact, Interview, and Assessment entities are not required for MVP.

## 3. Entity Overview

### Domain entities

- `User` — authenticated product user and tenant root.
- `GmailConnection` — Gmail authorization and sync state for a user.
- `Email` — minimized Gmail evidence record.
- `AIProcessingResult` — latest persisted AI processing state and validated AI output for an email.
- `Application` — current job application state.
- `ApplicationEvent` — historical recruitment/application occurrence.
- `Action` — user-facing obligation or required action.
- `AIConfiguration` — the user's one active AI provider setup (ADR-0001): provider, sealed key, model choices, access state, consent.
- `IntegrationToken` — a per-user bearer token for the MCP endpoint (ADR-0002); only its hash is stored.
- `ExternalSubmission` — evidence that the user's own automation reported a confirmed submission (ADR-0002).

### Infrastructure boundary

Sync execution/retry mechanics are operational concerns. MVP does not require a rich user-facing `SyncJob` domain entity; sync cursor and current sync state may remain on `GmailConnection` unless a later requirement creates a need for historical sync-run records.

## 4. Entity Definitions

### 4.1 User

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `createdAt` | DateTime | Yes | Creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

User is the root ownership boundary for all Career Companion data.

### 4.2 GmailConnection

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | FK to User; one Gmail connection for MVP. |
| `gmailEmail` | String | Yes | Connected Gmail account identifier. |
| `status` | Enum | Yes | `NOT_CONNECTED`, `CONNECTED`, `REVOKED` or equivalent minimal lifecycle. |
| `lastHistoryId` | String | No | Gmail sync cursor when available. |
| `lastSyncedAt` | DateTime | No | Last successful sync timestamp. |
| `syncStatus` | Enum | Yes | Minimal operational state such as `IDLE`, `SYNCING`, `FAILED`. |
| `createdAt` | DateTime | Yes | Creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

OAuth access/refresh credentials are sensitive infrastructure secrets and must not be exposed through API DTOs or ordinary domain responses.

### 4.3 Email

`Email` stores minimized source evidence from Gmail.

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | Tenant ownership. |
| `gmailMessageId` | String | Yes | Stable Gmail message identifier; unique within the user connection. |
| `subject` | String | No | Persist only when required by the MVP UX/evidence model. |
| `sender` | String | No | Persist only as needed for source evidence. |
| `receivedAt` | DateTime | No | Gmail message timestamp when available. |
| `relevanceState` | Enum | Yes | `UNPROCESSED`, `RELEVANT`, `IRRELEVANT`. Routing state, not classification. |
| `applicationId` | UUID | No | Nullable FK to Application. |
| `matchState` | Enum | Yes | `UNMATCHED`, `MATCHED`, `AMBIGUOUS`, `IGNORED`. Makes application matching explicit. |
| `matchConfirmedBy` | Enum | No | `AI_AUTO`, `USER_CONFIRMED`. Tracks how the match was established. |
| `createdAt` | DateTime | Yes | Creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

Full email bodies should not be retained by default. Raw Gmail content should be treated as source material for processing and minimized according to the project's privacy/retention policy.

### 4.4 AIProcessingResult

This entity records the **latest AI processing state and validated output** for an Email. It is not the domain source of truth and is not intended to become a second job-search data store.

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | Tenant ownership. |
| `emailId` | UUID | Yes | FK to Email; unique with `userId` for MVP. |
| `aiProvider` | String | Yes | Provider identifier, e.g. `gemini`; string rather than DB enum. |
| `model` | String | Yes | Full canonical model identifier. |
| `status` | Enum | Yes | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`. |
| `classification` | Enum | No | AI email category: `RECRUITER`, `INTERVIEW`, `ASSESSMENT`, `OFFER`, `REJECTION`, `FOLLOW_UP`, `NEWSLETTER`, `SPAM`. |
| `confidence` | Decimal 0.0–1.0 | No | One overall AI confidence score; no per-field confidence for MVP. |
| `extractedData` | JSON/JSONB | No | Validated structured AI extraction used by domain promotion. Must not contain raw model output or duplicate first-class provenance fields. |
| `summary` | String | No | Brief factual AI summary, maximum 280 characters for MVP. Not a replacement for the email body. |
| `errorMessage` | String | No | Processing error description only; must not contain raw model responses or email content. |
| `processedAt` | DateTime | No | Set when processing completes or fails. |
| `createdAt` | DateTime | Yes | Set on insert. |
| `updatedAt` | DateTime | Yes | Tracks processing state changes. |

#### AI processing rules

- MVP keeps one current `AIProcessingResult` per Email.
- A retry resets the current result to `PENDING`; MVP does not retain complete attempt history.
- `AIProcessingResult` therefore represents the **latest processing state**, not an audit log of every AI attempt.
- Raw provider responses and provider error text are never persisted (any provider).
- `provider` and `model` record which of the user's provider and models produced the result (from the operation ledger), so results stay explainable across provider switches.
- `extractedData` is validated staging output for domain promotion, not a UI-facing query model.
- Prompt/version tracking and full AI observability are deferred beyond MVP.

### 4.4a AIConfiguration and AI usage (ADR-0001)

One active AI setup per user (`ai_configurations`, keyed by `userId`, deleted with the user).

| Field | Meaning |
| --- | --- |
| `provider` | Catalog provider ID (`gemini`, `openai`, `anthropic`). The catalog is code, not data. |
| `encryptedApiKey` | The user's provider key as AES-256-GCM ciphertext bound to the user. Write-only: never returned, logged or sent anywhere except to the catalog's endpoint for that provider. |
| `fastModel`, `detailedModel` | `null` follows the recommended model; otherwise a tested catalog model for that role. |
| `accessIssue`, `accessIssueModel` | Why access cannot be used: needs attention (key rejected, account/billing, model unavailable, key unreadable) or limited (rate limited, provider unavailable). |
| `cooldownUntil`, `consecutiveFailures` | Per-user provider cooldown. |
| `verifiedAt`, `lastCheckedAt` | Content-free verification results. |
| `consentDisclosure`, `consentedAt` | Which data-use text the user agreed to, and when. |
| `revision` | Incremented on every save. Job-side state writes apply only to the revision they resolved. |

`ai_usage_days` (user, UTC day) holds Career Companion's own counts: calls (the safety-limit counter), input and output tokens, and verifications. These are not provider billing data.

Access is one of four states, derived from these facts: **Not set up**, **Ready**, **Needs attention**, **Limited**. While access is not Ready, emails needing AI wait as `PENDING`; this is not a processing failure.

### 4.5 Application

`Application` is the central job-search entity and represents the current state of one hiring/application process.

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | Tenant ownership. |
| `companyName` | String | Yes | Company name associated with the application. |
| `jobTitle` | String | No | Role title when reliably known. |
| `location` | String | No | Job location when reliably known. |
| `aiStatus` | Enum | No | Latest AI-derived application state. |
| `userStatus` | Enum | No | Explicit user-confirmed status. |
| `userStatusSetAt` | DateTime | No | Timestamp for the current explicit user confirmation. |
| `userStatusRevision` | Int | Yes | Manual-correction revision, default 0. Changed only by a user set/change/clear; never by AI. |
| `appliedAt` | DateTime | No | Application date when reliable evidence exists; distinct from `createdAt`. |
| `createdAt` | DateTime | Yes | Record creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

#### Application status rules

- Effective status is `userStatus ?? aiStatus`. Every application response carries derived (not persisted) `effectiveStatus`, `statusSource` (`USER` whenever `userStatus` is set, even when it equals AI; `AI` when only `aiStatus` is set; `UNKNOWN` when both are null) and `hasStatusConflict` (both set and different). A null effective status is a valid unknown, never defaulted to Applied.
- AI may update `aiStatus` but must never write `userStatus`, `userStatusSetAt` or `userStatusRevision`. AI/user disagreement is informational only.
- Manual correction (`PATCH /api/applications/:id/status`, approved D2 2026-10-02): one transaction locks the owned application row, compares `expectedUserStatusRevision` **before** no-op detection (a stale request is 409 even if its value equals the latest), then: a current same-value request changes nothing (revision, `userStatusSetAt`, `updatedAt` kept); a changed value sets `userStatus`, a server-generated `userStatusSetAt` and increments the revision once; clearing sets `userStatus` and `userStatusSetAt` to null and increments once, revealing the persisted AI status (or unknown) without rerunning AI.
- Provenance is latest-only: the Application holds the current correction and its confirmation time. Clearing removes that time. A full correction audit log is not kept. A legacy user status with a null timestamp stays "confirmation time unknown"; no date is invented.
- Manual corrections never modify `aiStatus`, create recruitment events, resolve actions, enqueue jobs, call providers or touch AI operation/budget records. Locking is application-only, compatible with matcher's email → application order.
- A single business uniqueness constraint such as `(userId, companyName, jobTitle)` is intentionally **not** enforced; users may have multiple applications for the same company/role.
- Company and JobPosting entities are deferred until a concrete MVP requirement requires them.
- Application status represents broad/current state; detailed occurrences belong to ApplicationEvent.

The exact final status vocabulary should remain the smallest set required by the MVP and be reconciled with ApplicationEvent semantics during implementation. Candidate MVP lifecycle concepts include `APPLIED`, `RECRUITER_CONTACT`, `ASSESSMENT`, `INTERVIEW`, `OFFER`, `REJECTED`, and `CLOSED`, with detailed occurrences represented as events.

### 4.6 ApplicationEvent

`ApplicationEvent` represents a historical recruitment/application occurrence.

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | Tenant ownership. |
| `applicationId` | UUID | Yes | FK to Application. |
| `emailId` | UUID | No | Optional source email/evidence. |
| `type` | Enum | Yes | `APPLICATION_SUBMITTED`, `RECRUITER_CONTACT`, `INTERVIEW_INVITE`, `INTERVIEW_COMPLETED`, `ASSESSMENT_ASSIGNED`, `OFFER_RECEIVED`, `REJECTION_RECEIVED`. |
| `occurredAt` | DateTime | Yes | When the event actually happened. |
| `scheduledAt` | DateTime | No | Used when an event carries a known interview/scheduled time; null otherwise. |
| `deadline` | DateTime | No | Used when an event carries a known assessment/action deadline; null otherwise. |
| `notes` | String | No | Concise structured/contextual detail when needed; not a raw email copy. |
| `provenance` | String | No | Identifies what caused this event (e.g. AI-extracted evidence). |
| `createdAt` | DateTime | Yes | When Career Companion recorded the event. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

#### Event rules

- `occurredAt` is distinct from `createdAt`. An email received today may describe an event that occurred earlier.
- One Email may support multiple ApplicationEvents.
- An ApplicationEvent may exist without an Email, allowing future/manual domain events without requiring artificial evidence.
- No `title` or AI-generated `description` is required. Timeline presentation should derive from event type plus structured fields/evidence.
- `scheduledAt` is meaningful for interview scheduling events; `deadline` is meaningful for assessment/deadline-bearing events. They remain null for unrelated event types.
- `STATUS_CORRECTED` is not treated as a recruitment event. User status provenance is represented on Application through `userStatus` and `userStatusSetAt` for MVP.
- Event records are historical; Application stores the materialized current state needed by the dashboard.

#### Implemented history boundary (Sprint 6)

The implemented `application_events` table records processing events (for example `EMAIL_PROCESSED`) with `oldState/newState` describing **AI status only**, optional `emailId`, `description`/`provenance` (AI interpretation, not verified source text) and `createdAt`. It has no `occurredAt`, `scheduledAt` or `deadline` columns, and none are inferred. Responses add `recordedAt` (same instant as `createdAt`: when the app recorded it) and a bounded `sourceEmail` `{ id, subject, sender, receivedAt }` that is returned only when the email belongs to the requesting owner; otherwise it is null (and a foreign `emailId` is also nulled). `receivedAt` is shown as "Email date", which may come from a Date header and is not proof of when a recruitment event happened. History is ordered by recording time (`createdAt`, then `id`), not reconstructed chronology. Action responses keep their existing `emailId` link; action source evidence is deferred (D1, 2026-10-02).

**`AUTOMATION_SUBMITTED` (ADR-0002):** one event per linked or created `ExternalSubmission`, keyed by a unique nullable `externalSubmissionId` (SetNull). It leaves `emailId`, `oldState`, `newState`, `description` and `provenance` null. Responses add a bounded, owner-checked `sourceSubmission` `{ platform, destinationHost, submittedAt, confirmationText }` for this type only (null on every other event and `recentEvent`), and `analyzedBy` is null. It is shown as "Submitted via automation", never as AI interpretation or missing email evidence. A backfilled submission keeps recording order and may appear after later emails.

### 4.7 Action

`Action` represents a user-facing obligation resulting from job-search communication or events.

| Attribute | Type | Required | Notes |
|---|---|---:|---|
| `id` | UUID | Yes | Primary identifier. |
| `userId` | UUID | Yes | Tenant ownership. |
| `applicationId` | UUID | Yes | FK to Application. |
| `emailId` | UUID | No | Optional source email/evidence. |
| `eventId` | UUID | No | Optional source ApplicationEvent for stronger provenance. |
| `type` | Enum | Yes | `RESPOND_TO_RECRUITER`, `COMPLETE_ASSESSMENT`, `SCHEDULE_INTERVIEW`, `FOLLOW_UP`, `REVIEW_OFFER`. |
| `context` | String | No | Short disambiguating context; not a long AI narrative. |
| `status` | Enum | Yes | `PENDING`, `COMPLETED`, `DISMISSED`, `OBSOLETE`. |
| `dueAt` | DateTime | No | Deadline when known. |
| `createdAt` | DateTime | Yes | Creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

#### Action rules

- `FOLLOW_UP` remains a generic MVP action type; its context identifies what the user is following up about.
- `OVERDUE` is derived from `dueAt` and current time; it is not stored as a separate status.
- `DISMISSED` means the user intentionally chose not to act. `OBSOLETE` means later information made the action no longer applicable.
- Users can complete or dismiss actions. System/AI-driven domain logic may mark actions obsolete when later events resolve or replace them.
- `dueAt` may duplicate an event's `deadline` intentionally: the event records the historical deadline evidence, while the Action stores the operational deadline used for action management.
- MVP deduplicates actions at the application/domain-service layer. No broad `(applicationId, type)` uniqueness constraint is required because multiple legitimate actions of the same type may exist over time.
- `Action` requires an Application in MVP. Unmatched/ambiguous email processing must be resolved before creating an application-bound action.

### 4.8 IntegrationToken and ExternalSubmission (ADR-0002)

**IntegrationToken** (`integration_tokens`, deleted with the user)

| Field | Meaning |
| --- | --- |
| `name` | 1–100 characters, chosen by the user. |
| `tokenHash` | SHA-256 of the plaintext (unique). The plaintext `ccmcp_` + 43 base64url characters is shown once at creation and never stored. |
| `displayPrefix` | The first 12 characters, for recognising a token. |
| `scope` | `submissions:write`, the only scope in v1. |
| `expiresAt`, `lastUsedAt`, `revokedAt` | Expiry (default 90 days, at most 365), last successful use, and immediate revocation. Revoked rows are kept for the audit trail. At most 5 active tokens per user. |

**ExternalSubmission** (`external_submissions`, deleted with the user)

| Field | Meaning |
| --- | --- |
| `source`, `sourceRecordRef` | `AUTOMATION` and `YYYY-MM-DD/HH:MM:SS` from the automation's daily file. Unique per `(userId, source, sourceRecordRef)`: the idempotency key. |
| `platform`, `company`, `jobTitle`, `submittedAt`, `jobUrl`, `portalJobId`, `destinationHost`, `discoverySource`, `location`, `workMode`, `confirmationText` | The ADR-0002 contract fields, trimmed and canonical (job URL without fragment, credentials or non-job-ID query parameters; confirmation truncated to 300 characters). Never submitted answers, resume data or credentials. |
| `receivedAt`, `tokenId` | When Career Companion recorded it, and the token used (SetNull). |
| `matchState` | `NEEDS_REVIEW`, `LINKED`, `CREATED` or `IGNORED`. |
| `resolvedBy`, `resolvedAt` | `AUTOMATIC` (rule-based, never AI) or `USER`, and when. Both null exactly while `NEEDS_REVIEW` (CHECK). |
| `applicationId` | The linked or created application (SetNull). |

Rules:
- A submission is evidence, not a replica: a later call with the same ref never updates it. A user resolution is final in v1.
- Matching (ADR-0002 §7) uses a company key (the existing normalization after dropping trailing legal-suffix words) and the existing title key. No same-company application → `CREATED`; exactly one same-title application and no untitled one → `LINKED`; anything else, or an empty key → `NEEDS_REVIEW`. The Gmail matcher is unchanged.
- Created applications get the submitted company, title and location, and `appliedAt = submittedAt`. Linking sets `appliedAt` only when it is empty. `aiStatus`, `userStatus` and `deriveStatus` are never touched.
- `ApplicationResponse.submittedVia` is `AUTOMATION` when a linked or created submission exists. The UI shows "Applied · via automation" only while `statusSource` is `UNKNOWN`; a user or AI status always wins.
- Ownership triggers reject: a submission linked to another owner's application or token; a change of a submission's or token's owner; an event whose submission belongs to someone other than the application's owner. Links are checked when set or changed, so SetNull deletions never fail.

## 5. Relationships

```text
User
├── 1:1 GmailConnection
├── 1:N Email
│   ├── 1:1 AIProcessingResult (current MVP result)
│   └── N:1 Application (optional; via matchState)
├── 1:N IntegrationToken
├── 1:N ExternalSubmission
│   ├── N:1 IntegrationToken (optional; SetNull)
│   └── N:1 Application (optional; via matchState)
└── 1:N Application
    ├── 1:N ApplicationEvent
    │   ├── N:1 Email (optional evidence)
    │   └── 1:1 ExternalSubmission (optional evidence, AUTOMATION_SUBMITTED)
    └── 1:N Action
        ├── N:1 Email (optional evidence)
        └── N:1 ApplicationEvent (optional provenance)
```

### Relationship invariants

1. `Email.applicationId` is nullable because an email may be unmatched, ambiguous, or ignored.
2. `Email.matchState` explicitly distinguishes `UNMATCHED`, `MATCHED`, `AMBIGUOUS`, and `IGNORED` rather than relying on a nullable FK alone.
3. `ApplicationEvent.applicationId` is required.
4. `ApplicationEvent.emailId` is optional because domain events can exist without a single source email.
5. `Action.applicationId` is required for MVP.
6. `Action.emailId` and `Action.eventId` are optional provenance references.
7. Cross-user references must be rejected: an entity's referenced Email/Application/Event must belong to the same User.

## 6. Application Matching

Application identity is intentionally not defined by a simple `(companyName, jobTitle)` uniqueness constraint.

Email-to-application matching is an application-layer decision based on available evidence. The model must support four outcomes:

```text
UNMATCHED   → no reliable application identified
MATCHED     → one reliable application identified
AMBIGUOUS   → multiple plausible applications; do not silently choose
IGNORED     → email intentionally excluded from application association
```

When matching is ambiguous, the system must not silently attach the email to an arbitrary application.

## 7. Status and Event Boundary

The model separates current state from history:

```text
Application
  current state
       ↑
       │ materialized/updated from domain events
ApplicationEvent
  historical occurrence
       ↑
       │ evidence
Email
```

The MVP should not implement full event sourcing. Application state is a materialized domain value, while ApplicationEvent provides the historical timeline needed to understand meaningful changes.

Detailed event types such as `INTERVIEW_INVITE` and `INTERVIEW_COMPLETED` should not automatically become separate persistent application statuses. Broad application status and detailed event history serve different purposes.

## 8. AI-to-Domain Promotion

The conceptual processing path is:

```text
Gmail
  ↓
Email (minimized evidence)
  ↓
AIProcessingResult (validated AI interpretation)
  ↓
Domain promotion/service logic
  ├── Application
  ├── ApplicationEvent
  └── Action
```

AIProcessingResult is not mandatory middleware for manual changes:

```text
User
  ↓
Application.userStatus
```

Manual status correction therefore does not require AIProcessingResult.

## 9. Privacy and Retention

- Raw email bodies should not be persisted unless a concrete MVP requirement requires them.
- Raw AI model responses must never be persisted in AIProcessingResult.
- `summary` must remain a brief factual derivative, not a copy of the email body.
- `extractedData` contains validated structured extraction only and must not become a second email/domain store.
- Error messages must not contain raw email content or model response dumps.
- Deleting a User should delete associated domain and processing data according to the eventual database deletion policy.
- AIProcessingResult retention should follow Email retention in MVP; it does not require an independent long-term retention lifecycle.

## 10. Conceptual Database Constraints

The implementation should enforce at minimum:

- UUID primary keys.
- Required foreign keys between related entities.
- Unique `(userId, emailId)` for the single current AIProcessingResult per Email in MVP.
- Appropriate Gmail message identity uniqueness within a user/connection.
- Indexes beginning with `userId` for tenant-scoped query paths.
- Indexes for operational AI states such as `(userId, status)`.
- Indexes supporting application timelines such as `(applicationId, occurredAt)`.
- Indexes supporting pending actions such as `(userId, status, dueAt)`.
- Cross-user references must be prevented through application validation and, where practical, database-level composite foreign-key enforcement.
- No business uniqueness constraint on `(userId, companyName, jobTitle)`.
- No stored `OVERDUE` action status.

Exact Prisma/SQL definitions belong to the implementation phase and are intentionally not specified here.

## 11. Deferred / Not MVP

The following are intentionally excluded from the MVP domain model:

- Company entity
- JobPosting entity
- Recruiter/Contact entity
- Interview entity
- Assessment entity
- StatusHistory entity
- EmailApplicationMatch join entity
- Full AI processing history / experiment tracking
- Prompt/schema version tracking
- Per-field AI confidence
- Generic AI metadata/configuration blobs
- Event sourcing
- Rich SyncJob history entity

These may be reconsidered when a concrete product requirement justifies them.

## 12. Final Domain Model Decision

The MVP domain model is considered stable enough to proceed to high-level architecture.

The model deliberately favors clear boundaries over maximal normalization:

```text
Source evidence       → Email
AI interpretation     → AIProcessingResult
Current job state     → Application
Historical occurrence → ApplicationEvent
User obligation       → Action
Ownership boundary    → User
Gmail integration     → GmailConnection
```

This model is intended to provide a stable foundation for COM-6 High-Level Architecture without prematurely locking implementation-specific database or API details.

### Action deadline precision (Sprint 7 implementation)

Actions now store nullable `deadlinePrecision` (`DATE` or `DATETIME`). DATE is a UTC-midnight calendar date displayed without a time; it becomes overdue after that day in the viewer's local calendar. DATETIME is an explicit-zone instant. Null precision means an unchanged legacy value. New ambiguous deadlines remain null. Missing years are inferred from the email's received UTC date with a one-day allowance, never the processing year; see Sprint 7 S7-03. No existing action is backfilled.
