# Career Companion — Domain Model

**Version:** 1.1
**Status:** Sprint 2 Implemented  
**Last Updated:** 2026-09-12

### Version History / Implementation Notes
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
- Raw Gemini responses are never persisted.
- `extractedData` is validated staging output for domain promotion, not a UI-facing query model.
- Prompt/version tracking and full AI observability are deferred beyond MVP.

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
| `appliedAt` | DateTime | No | Application date when reliable evidence exists; distinct from `createdAt`. |
| `createdAt` | DateTime | Yes | Record creation timestamp. |
| `updatedAt` | DateTime | Yes | Last modification timestamp. |

#### Application status rules

- Effective status is conceptually `userStatus ?? aiStatus`.
- AI may update `aiStatus` but must not overwrite `userStatus`.
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

## 5. Relationships

```text
User
├── 1:1 GmailConnection
├── 1:N Email
│   ├── 1:1 AIProcessingResult (current MVP result)
│   └── N:1 Application (optional; via matchState)
└── 1:N Application
    ├── 1:N ApplicationEvent
    │   └── N:1 Email (optional evidence)
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
