# Career Companion — High-Level Architecture

**Version:** 0.2 (In Review — awaiting final ChatGPT approval)
**Status:** In Review
**Linear Issue:** [COM-6 — Design High-Level Architecture](https://linear.app/welcome-nitin/issue/COM-6/design-high-level-architecture)
**Last Updated:** 2026-09-05

---

## 1. Purpose

This document defines the high-level system architecture for Career Companion MVP. It covers the major system components, their responsibilities, how they communicate, the technology stack with rationale, and the key architectural constraints that guide implementation.

This document is intentionally high-level. It establishes structure and boundaries; implementation-level detail belongs in engineering tickets and technical design documents produced during Sprint 1.

---

## 2. System Overview

Career Companion ingests Gmail recruitment emails, processes them through an AI pipeline, and presents the resulting structured job-search state to the user through a dashboard.

The system has four distinct layers:

```
┌─────────────────────────────────────┐
│           Frontend (SPA)            │  React + Vite + TanStack
└──────────────────┬──────────────────┘
                   │ REST / HTTPS
┌──────────────────▼──────────────────┐
│           Backend API               │  Express + TypeScript
└──────┬───────────────────────┬──────┘
       │                       │
       │ Enqueue jobs    Prisma │ (direct)
       │                       │
┌──────▼──────┐   ┌────────────▼──────┐
│  pg-boss    │   │    PostgreSQL      │
│  Job Queue  │   │    (Supabase)      │
└──────┬──────┘   └───────────────────┘
       │ Dequeue                 ▲
┌──────▼──────────────────┐      │ Prisma (direct)
│    Background Worker    │──────┘
│                         │
│  Gmail Sync Pipeline    │──────▶ Gmail API
│  AI Processing Pipeline │──────▶ Gemini API
│  Domain Promotion       │
└─────────────────────────┘
```

---

## 3. System Components

### 3.1 Frontend

**Technology:** React 19 + Vite + TypeScript
**Routing:** TanStack Router (type-safe, file-based client routing)
**Server state:** TanStack Query (data fetching, caching, background refetch, optimistic updates)
**UI:** Tailwind CSS + shadcn/ui
**Hosting:** Vercel (free tier)

The frontend is a single-page application (SPA). It is entirely behind authentication — no public-facing pages, no SEO requirements, no server-side rendering needed. TanStack Query handles all server-state synchronization, which is the dominant concern for a live job-search dashboard.

### 3.2 Backend API

**Technology:** Node.js + Express + TypeScript
**ORM:** Prisma
**Auth:** Google OAuth 2.0 (Passport.js or custom middleware)
**Session:** HTTP-only cookie sessions
**Hosting:** Railway or Render (persistent process, not serverless)

The Backend API is the single entry point for all frontend requests. It handles authentication, reads/writes domain state, and enqueues background jobs. It does not call Gmail or the Gemini API directly — those are the Worker's responsibility.

### 3.3 Background Worker

**Technology:** Node.js + TypeScript
**Job queue:** pg-boss (PostgreSQL-backed, runs as a sibling process or co-located with API)
**Hosting:** Same deployment unit as the Backend API (Railway / Render), or a separate dyno/service if needed

The Worker owns the entire async pipeline:
- Gmail synchronization (initial and ongoing)
- Email relevance classification (RelevanceClassifier)
- Deep email analysis (EmailAnalyzer)
- Domain promotion (Application, ApplicationEvent, Action creation/update)

The Worker and the Backend API share the same PostgreSQL database via Prisma. They do not communicate with each other directly — the API enqueues jobs; the Worker dequeues and processes them.

### 3.4 Database

**Technology:** PostgreSQL
**Managed by:** Managed PostgreSQL provider (current target: Supabase free tier)
**ORM / schema:** Prisma (shared schema between API and Worker)

The domain model is defined in [COM-5](https://linear.app/welcome-nitin/issue/COM-5). The database is the single source of truth for all job-search state. The Prisma schema is the authoritative implementation of the domain model.

### 3.5 Gmail Integration

**API:** Google Gmail REST API (`@googleapis/gmail`)
**Auth:** Google OAuth 2.0 with `gmail.readonly` scope
**Sync mechanism:** Polling via Gmail History API with `lastHistoryId` cursor (V1). Gmail push notifications via Google Cloud Pub/Sub are a V2 upgrade path.

### 3.6 AI Integration

**Provider:** Google Gemini API
**Client:** `@google/genai` SDK (TypeScript)

The AI integration uses **two named roles**, not specific model version constants. Model selection is implementation configuration; the architecture references roles:

| Role | Responsibility | Current model target |
|---|---|---|
| `RelevanceClassifier` | Fast, low-cost relevance + classification on metadata only | `gemini-2.5-flash-lite` |
| `EmailAnalyzer` | Deep extraction, summary, action detection, match candidates on full content | `gemini-2.5-flash` |

Model names are stored in environment configuration, not hardcoded. This allows model upgrades without architectural changes.

> **Note:** `gemini-2.0-flash-lite` was discontinued on June 1, 2026 and must not be used.

---

## 4. Gmail Ingestion and AI Processing Pipeline

### 4.1 Full Pipeline

```
Gmail
  ↓
[1] Metadata ingestion
    Fields: sender, subject, date, threadId, snippet, labelIds
    Format: messages.get with format=metadata
    Full email body is NOT fetched at this stage
  ↓
[2] Deterministic pre-filter  (zero AI cost)
    - Gmail label CATEGORY_PROMOTIONS → skip
    - Gmail label CATEGORY_SOCIAL → skip
    - Gmail label SPAM → skip
    - Sender domain on known marketing/newsletter blocklist → skip
    Skipped emails: stored with relevanceState=IRRELEVANT, retained in DB (recoverable)
  ↓ (passed pre-filter)
[3] RelevanceClassifier  (fast Gemini model, metadata only, ~200 tokens)
    Output structured JSON:
      {
        "relevance": "RELEVANT" | "IRRELEVANT" | "UNCERTAIN",
        "classification": "RECRUITER" | "INTERVIEW" | "ASSESSMENT" |
                          "OFFER" | "REJECTION" | "FOLLOW_UP" | null,
        "confidence": 0.0–1.0
      }
    → IRRELEVANT: relevanceState=IRRELEVANT, stop (retained, recoverable)
    → RELEVANT or UNCERTAIN: proceed
  ↓
[4] Full email content fetch
    messages.get with format=full
    Body used for AI processing only; raw body is NOT persisted
  ↓
[5] EmailAnalyzer  (stronger Gemini model, full content)
    Output structured JSON:
      {
        "relevance": "RELEVANT" | "IRRELEVANT",
        "classification": "RECRUITER" | "INTERVIEW" | ... | null,
        "extractedData": {
          "companyName", "jobTitle", "location",
          "applicationRef", "scheduledAt", "deadline"
        },
        "summary": string (max 280 chars),
        "actionRequired": boolean,
        "actionType": "RESPOND_TO_RECRUITER" | "COMPLETE_ASSESSMENT" |
                      "SCHEDULE_INTERVIEW" | "FOLLOW_UP" | "REVIEW_OFFER" | null,
        "actionDueAt": datetime | null,
        "matchCandidates": [
          { "applicationId": uuid, "confidence": 0.0–1.0, "signals": [...] }
        ]
      }
  ↓
[6] Persist AIProcessingResult
    Validated structured output stored in extractedData
    Raw Gemini response is NEVER persisted
  ↓
[7] Application matching  (see Section 5)

    ┌─────────────────────────────────────────────────────────┐
    │ Deterministic / High-confidence match (Tiers 1–2)       │
    │   → Email.matchState = MATCHED                          │
    │   → Email.applicationId set                             │
    │   → Proceed directly to [8] Domain promotion            │
    └─────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │ Ambiguous / Low-confidence (Tiers 3–4)                  │
    │   → Email.matchState = AMBIGUOUS                        │
    │   → Ranked candidates surfaced to user via dashboard    │
    │   → User selects application (or creates new)           │
    │   → User confirmation recorded                          │
    │   → Email.matchState = MATCHED                          │
    │   → Proceed to [8] Domain promotion                     │
    └─────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │ No match (Tier 5)                                       │
    │   → Email.matchState = UNMATCHED                        │
    │   → Domain promotion deferred until user links manually │
    └─────────────────────────────────────────────────────────┘
  ↓
[8] Domain promotion  (runs only after Email.matchState = MATCHED)
    - Email.applicationId and Email.matchState already set in [7]
    - Create or update Application (aiStatus only — never userStatus)
    - Create ApplicationEvent (linked to Email as evidence)
    - Create Action if actionRequired=true
```

### 4.2 Pipeline Constraints

- **No open database transactions during Gmail or Gemini API calls.** The pattern is always: read state → close transaction → call external API → open new transaction → persist result → close transaction.
- **AI processing and domain promotion are separately retryable.** Steps 3–6 (AI phase) and step 8 (domain promotion) are independent pg-boss jobs. A retry on the AI phase does not re-trigger domain promotion. A retry on domain promotion is idempotent.
- **Each phase owns its defined state transitions and must not bypass another phase's responsibilities.** The RelevanceClassifier owns `Email.relevanceState`. The EmailAnalyzer owns `AIProcessingResult`. The matching step owns `Email.matchState` and `Email.applicationId`. Domain promotion owns `Application`, `ApplicationEvent`, and `Action`. Phases must not reach into another phase's domain; for example, domain promotion must not bypass the matching step by writing `Email.applicationId` independently.

---

## 5. Application Matching Strategy

> **We are not trying to solve perfect email → application matching in V1. That is a deliberate product decision, not a limitation.**

### 5.1 Human-in-the-Loop Model

```
AI proposes → user confirms → system records association → AI can later flag inconsistencies
```

User confirmation is the authoritative act for ambiguous matches. However, **user confirmation is not permanent truth** — a user can confirm the wrong application. The system must preserve enough evidence to later detect and surface potentially incorrect associations.

### 5.2 Matching Tiers (evaluated cheapest first)

| Tier | Signal | Action | Cost |
|---|---|---|---|
| **1** | `threadId` matches an existing matched email | Auto match — same application | Free (DB lookup) |
| **2** | Explicit application/job reference ID extracted | Strong signal — auto match | Free (EmailAnalyzer already paid) |
| **3** | AI match confidence above threshold, single clear candidate | Propose to user for confirmation | No additional Gemini call |
| **4** | Low confidence or multiple plausible candidates | Show ranked candidates, require user selection | No additional Gemini call |
| **5** | No plausible match | UNMATCHED — user can link manually | — |

Match candidates are returned as part of the `EmailAnalyzer` output. Application matching does not trigger an additional Gemini call.

### 5.3 Sender Domain is Supporting Evidence, Not Identity

A single application may generate emails from a recruiter's personal Gmail, an ATS (Greenhouse, Lever, Workday), the company's HR team, an assessment platform (HackerRank, Codility), or a scheduling tool (Calendly). None of these share a domain. Sender domain is one signal among many — never the primary application identifier.

### 5.4 Association Provenance

For every email-to-application association, the architecture requires that the following be recorded:

- **How the association was made:** automatically (deterministic or high-confidence AI) vs. user-confirmed
- **AI confidence at time of match:** stored in `AIProcessingResult.extractedData`
- **AI-proposed candidate applications:** retained even after user override

This enables a V2 product feature: flagging potentially inconsistent associations when later evidence conflicts with a confirmed match. V1 preserves the data; V1 does not build the flagging UI or logic.

> **⚠️ COM-5 Amendment Required**
>
> The current COM-5 domain model (`Email`) does not include a dedicated field for association provenance (i.e. whether the match was auto-confirmed or user-confirmed). The `Email.matchState` enum (`UNMATCHED`, `MATCHED`, `AMBIGUOUS`, `IGNORED`) covers the current match outcome but not how that outcome was reached.
>
> Before or during Sprint 1 implementation, COM-5 must be amended to add match provenance. The proposed addition is a field such as `matchConfirmedBy: AI_AUTO | USER_CONFIRMED` on the `Email` entity. This must be a tracked COM-5 change, not a silent schema addition.
>
> This COM-6 document records the requirement. The actual domain model change belongs in COM-5.

### 5.5 Application Timelines are Application-Scoped

`ApplicationEvent` records belong to an `Application`. The timeline is always rendered through the application lens — never as a flat email feed or sender-grouped inbox. This is an architectural constraint on both the backend query model and the frontend navigation design.

---

## 6. Technology Stack

### Frontend

| Concern | Technology | Rationale |
|---|---|---|
| Framework | React 19 + Vite | SPA sufficient for a private auth-gated dashboard; no SSR needed |
| Language | TypeScript | End-to-end type safety; aligns with backend/worker |
| Routing | TanStack Router | Type-safe client routing, no framework lock-in |
| Server state | TanStack Query | Built for server-state sync: caching, background refetch, optimistic updates |
| UI primitives | shadcn/ui | Accessible, composable component primitives |
| Styling | Tailwind CSS | Utility-first, consistent with shadcn/ui |
| Hosting | Vercel or equivalent static hosting | CDN delivery of static assets; persistent process not required |

### Backend API

| Concern | Technology | Rationale |
|---|---|---|
| Runtime | Node.js + TypeScript | Consistent language across all layers |
| Framework | Express | Lightweight, no magic; clear separation of concerns |
| ORM | Prisma | Type-safe, code-generates from schema, enforces domain model contracts |
| Auth | Google OAuth 2.0 + Passport.js | Gmail scope grant handled at the backend |
| Sessions | HTTP-only cookies | Secure, simple; JWTs not needed for V1 solo project |
| Hosting | Persistent-process hosting (Railway, Render, or equivalent) | Persistent process required — not serverless |

### Background Worker

| Concern | Technology | Rationale |
|---|---|---|
| Job queue | pg-boss | PostgreSQL-backed; no Redis required; queue/worker architecture fully applicable |
| Gmail client | `@googleapis/gmail` | Official Google client library |
| Gemini client | `@google/genai` | Official Google GenAI SDK |
| Retry / dead-letter | pg-boss built-in | Exponential backoff, visibility timeout, dead-letter queue |

> **pg-boss vs BullMQ:** BullMQ is architecturally equivalent but requires Redis. pg-boss uses the existing PostgreSQL instance. The queue/worker patterns (enqueue, dequeue, retry, dead-letter) are identical. pg-boss is the V1 choice. Redis + BullMQ can be considered if volume requires it.

### Database

| Concern | Technology | Rationale |
|---|---|---|
| Database | PostgreSQL | Relational model fits the domain; strong consistency |
| Managed provider | Managed PostgreSQL (current target: Supabase) | Managed backups, no self-hosting required |
| Schema management | Prisma Migrate | Versioned migrations, safe schema evolution |

### AI

| Concern | Technology | Rationale |
|---|---|---|
| Provider | Google Gemini API | Available on Gemini Pro plan; target models on free tier |
| RelevanceClassifier | `gemini-2.5-flash-lite` (current) | Cheapest, fastest; sufficient for metadata-only classification |
| EmailAnalyzer | `gemini-2.5-flash` (current) | Stronger model for complex extraction from full email content |
| Output format | Gemini structured output (JSON schema) | Parseable, validated output; avoids prompt-engineering fragility |
| Model config | Environment variables | Model identifiers are config, not code; upgrades require no code changes |

---

## 7. Communication Patterns

### Frontend ↔ Backend API
- REST over HTTPS, JSON
- HTTP-only session cookie (Secure, SameSite=Lax)
- All routes require authenticated session except `/auth/*`

### Backend API ↔ Worker
- No direct communication
- API enqueues jobs via pg-boss (PostgreSQL INSERT)
- Worker dequeues via pg-boss (SELECT FOR UPDATE SKIP LOCKED)
- Fully decoupled — neither knows the other's internal state

### Worker ↔ Gmail API
- Gmail REST API via `@googleapis/gmail`
- Polling via `users.history.list` with `startHistoryId` cursor from `GmailConnection.lastHistoryId`
- Initial sync: paginated `users.messages.list` with label/query filters
- OAuth tokens stored encrypted at rest; refresh handled by Worker before each sync

### Worker ↔ Gemini API
- `@google/genai` SDK (TypeScript)
- Structured output mode (JSON schema enforced at API level)
- Non-streaming (structured output requires complete responses)
- Rate limiting handled via SDK retry configuration

### Worker ↔ Database
- Prisma direct connection (same PostgreSQL instance as API)
- Transactions are short-lived; never open during external API calls
- Each processing phase is an independent transaction

---

## 8. Key Architectural Constraints

These are non-negotiable for V1.

1. **No open DB transactions during external API calls.** Pattern: read state → close transaction → call Gmail/Gemini → open transaction → persist result → close transaction.

2. **AI processing and domain promotion are separately retryable.** Two distinct pg-boss jobs. Failed promotion does not re-run Gemini. Failed Gemini does not re-run promotion.

3. **User-confirmed status (`userStatus`) is never overwritten by AI.** Domain promotion writes only to `Application.aiStatus`. `userStatus` is written exclusively by user-initiated actions.

4. **Filtered emails are retained, not deleted.** `relevanceState=IRRELEVANT` is a mutable flag. Emails can be reprocessed if filter logic changes.

5. **Raw email bodies are never persisted.** Bodies are fetched for processing only; discarded after `AIProcessingResult` is written.

6. **Raw Gemini responses are never persisted.** Only validated structured output is stored in `AIProcessingResult.extractedData`.

7. **Application timelines are application-scoped.** No sender-grouped or domain-grouped email view in V1. All timeline queries start from `applicationId`.

8. **AI match evidence is preserved after user confirmation.** AI confidence scores and proposed candidates are stored even after a user-confirmed override, enabling future inconsistency detection.

---

## 9. Deployment Topology

```
Static hosting (e.g. Vercel)
  └── Frontend (React SPA — static build, CDN delivery)

Persistent-process hosting (e.g. Railway, Render)
  ├── Backend API  (Express — persistent process)
  └── Worker       (pg-boss + Gmail sync + Gemini — persistent process)
      Both share the same PostgreSQL connection

Managed PostgreSQL (e.g. Supabase)
  └── PostgreSQL (domain data + pg-boss job tables)

Google APIs
  ├── Gmail API  (OAuth 2.0 — gmail.readonly scope)
  └── Gemini API (RelevanceClassifier + EmailAnalyzer)
```

> **Current implementation targets:** Vercel (frontend), Railway or Render (backend + worker), Supabase (PostgreSQL). These are implementation choices, not architectural dependencies. The architecture requires only: static hosting, a persistent-process host, and a managed PostgreSQL instance.

---

## 10. Deferred / Not V1

| Item | Reason |
|---|---|
| Local / self-hosted LLM | Infrastructure complexity not justified for V1; Gemini API with metadata-only classification is sufficient at personal-project scale |
| Redis + BullMQ | pg-boss eliminates Redis dependency for V1 volume |
| Gmail push notifications (Pub/Sub) | Polling sufficient for V1; Pub/Sub adds GCP setup overhead |
| Inconsistency flagging UI | V2 feature; V1 preserves the data only |
| Automatic application correlation engine | Human-in-the-loop matching is the deliberate V1 decision |
| Per-field AI confidence scoring | Single overall confidence score sufficient for MVP |
| Full AI processing history / audit log | Single current AIProcessingResult per email (COM-5) |
| JWTs / token-based auth | HTTP-only cookie sessions sufficient for V1 |
| React Native mobile app | COM-10 — future roadmap |

---

## Document Versioning

| Version | Date | Notes |
|---|---|---|
| 0.1 | 2026-09-05 | Initial draft. All decisions from COM-6 review session incorporated. |
| 0.2 | 2026-09-05 | ChatGPT review corrections: (1) explicit matching branch in pipeline, (2) AI_AUTO provenance flagged as COM-5 amendment, (3) phase ownership constraint reworded, (4) pricing/free-tier assumptions removed from architecture. Pending final ChatGPT approval. |
