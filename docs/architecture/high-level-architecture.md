# Career Companion — High-Level Architecture

**Version:** 0.4 (COM-6 Final — corrections applied)
**Status:** In Review
**Linear Issue:** [COM-6 — Design High-Level Architecture](https://linear.app/welcome-nitin/issue/COM-6/design-high-level-architecture)
**Last Updated:** 2026-09-06

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
                   │ REST / HTTPS (via CloudFront → ALB)
┌──────────────────▼──────────────────┐
│           Backend API               │  Express + TypeScript (ECS Fargate)
└──────┬───────────────────────┬──────┘
       │                       │
       │ Enqueue jobs    Prisma │ (direct)
       │                       │
┌──────▼──────┐   ┌────────────▼──────┐
│  pg-boss    │   │    PostgreSQL      │
│  Job Queue  │   │    (RDS)           │
└──────┬──────┘   └───────────────────┘
       │ Dequeue                 ▲
┌──────▼──────────────────┐      │ Prisma (direct)
│    Background Worker    │──────┘
│    (ECS Fargate)        │
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
**Hosting:** Amazon S3 + CloudFront (static build, CDN delivery, OAC-restricted private bucket)

The frontend is a single-page application (SPA). It is entirely behind authentication — no public-facing pages, no SEO requirements, no server-side rendering needed. TanStack Query handles all server-state synchronization, which is the dominant concern for a live job-search dashboard.

### 3.2 Backend API

**Technology:** Node.js + Express + TypeScript
**ORM:** Prisma
**Auth:** Google OAuth 2.0 (Passport.js or custom middleware)
**Session:** HTTP-only cookie sessions
**Hosting:** ECS Fargate (persistent container behind ALB)

The Backend API is the single entry point for all frontend requests. It handles authentication, reads/writes domain state, and enqueues background jobs. It does not call Gmail or the Gemini API directly — those are the Worker's responsibility. The API does require outbound HTTPS for Google OAuth provider interactions (e.g. token exchange during the OAuth callback flow).

### 3.3 Background Worker

**Technology:** Node.js + TypeScript
**Job queue:** pg-boss (PostgreSQL-backed)
**Hosting:** ECS Fargate (separate service from the API; independently deployable)

The Worker owns the entire async pipeline:
- Gmail synchronization (initial and ongoing)
- Email relevance classification (RelevanceClassifier)
- Deep email analysis (EmailAnalyzer)
- Domain promotion (Application, ApplicationEvent, Action creation/update)

The Worker requires outbound HTTPS access to the Gmail API and Gemini API. The Worker and the Backend API share the same PostgreSQL database via Prisma. They do not communicate with each other directly — the API enqueues jobs; the Worker dequeues and processes them.

### 3.4 Database

**Technology:** PostgreSQL
**Managed by:** Amazon RDS (managed, automated backups, private subnet)
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
| Hosting | Amazon S3 + CloudFront (OAC-restricted private bucket) | CDN delivery of static assets; persistent process not required |

### Backend API

| Concern | Technology | Rationale |
|---|---|---|
| Runtime | Node.js + TypeScript | Consistent language across all layers |
| Framework | Express | Lightweight, no magic; clear separation of concerns |
| ORM | Prisma | Type-safe, code-generates from schema, enforces domain model contracts |
| Auth | Google OAuth 2.0 + Passport.js | Gmail scope grant handled at the backend |
| Sessions | HTTP-only cookies | Secure, simple; JWTs not needed for V1 solo project |
| Hosting | ECS Fargate (persistent container behind ALB) | Persistent process required — not serverless |

### Background Worker

| Concern | Technology | Rationale |
|---|---|---|
| Job queue | pg-boss | PostgreSQL-backed; no Redis required; queue/worker architecture fully applicable |
| Gmail client | `@googleapis/gmail` | Official Google client library |
| Gemini client | `@google/genai` | Official Google GenAI SDK |
| Retry / dead-letter | pg-boss built-in | Exponential backoff, visibility timeout, dead-letter queue |
| Hosting | ECS Fargate (separate service from API; independently deployable) | Persistent background process; independently deployable and restartable |

> **pg-boss vs BullMQ:** BullMQ is architecturally equivalent but requires Redis. pg-boss uses the existing PostgreSQL instance. The queue/worker patterns (enqueue, dequeue, retry, dead-letter) are identical. pg-boss is the V1 choice. Redis + BullMQ can be considered if volume requires it.

### Database

| Concern | Technology | Rationale |
|---|---|---|
| Database | PostgreSQL | Relational model fits the domain; strong consistency |
| Managed provider | Amazon RDS | Managed backups, no self-hosting; private subnet |
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

## 9. Cloud & Deployment Architecture

Career Companion deploys exclusively on AWS. This section defines the complete cloud infrastructure, networking, security, operational, and deployment architecture.

### 9.1 Selected Platform

**Cloud provider:** Amazon Web Services (AWS)  
**Primary region:** To be selected at project start (e.g. `ap-south-1` Mumbai or `us-east-1` N. Virginia)  
**Environments:** Local development and Production only. Staging is explicitly deferred.

### 9.2 Deployment Topology

```
Browser (HTTPS)
    ↓
Amazon CloudFront  (CDN + HTTPS termination)
    ├── /*         → Amazon S3  (React SPA static build)
    │
    └── /api/*     → Application Load Balancer  (public subnets)
                            ↓
                   ECS Fargate API Service  (public subnets)
                            ↓
                   RDS PostgreSQL  (private subnets)
                            ↑  pg-boss job tables (co-located in RDS)
                            ↑
                   ECS Fargate Worker Service  (public subnets)
                        ↙               ↘
                 Gmail API           Gemini API
               (outbound HTTPS)   (outbound HTTPS)

Supporting infrastructure:
    Amazon ECR          — Docker image registry (API + Worker)
    AWS Secrets Manager — all application credentials
    Amazon CloudWatch   — logs and basic monitoring
    IAM + OIDC          — service identity + GitHub Actions deploy role
    ACM                 — managed TLS certificate for custom domain
    VPC                 — network isolation
```

### 9.3 Infrastructure Components

| Component | AWS Service | Notes |
|---|---|---|
| React SPA hosting | S3 + CloudFront + OAC | Private S3 bucket; CloudFront access via OAC only; CDN-distributed; HTTPS via ACM; SPA routing via custom error response |
| API traffic routing | Application Load Balancer | Internet-facing; public subnets; CloudFront custom origin |
| API compute | ECS Fargate (service) | Persistent container; ALB target; independently deployable |
| Worker compute | ECS Fargate (service) | Persistent container; no inbound; independently deployable |
| Database | RDS PostgreSQL | Single-AZ V1; private subnets; pg-boss tables co-located |
| Container registry | Amazon ECR | API and Worker images; image lifecycle policies |
| Secrets | AWS Secrets Manager | All application credentials; no plaintext in environment |
| Logs | Amazon CloudWatch Logs | Log groups per service; 30-day retention |
| TLS certificate | ACM | Free managed certificate; renews automatically |
| IAM | IAM Roles + OIDC | Task roles for runtime; OIDC role for GitHub Actions |

### 9.4 Networking

**VPC CIDR:** 10.0.0.0/16

```
VPC 10.0.0.0/16
│
├── Internet Gateway
│
├── Public Subnet AZ-a   10.0.1.0/24   route: 0.0.0.0/0 → IGW
│   ├── ALB node
│   ├── ECS API tasks       (assignPublicIp: ENABLED)
│   └── ECS Worker tasks    (assignPublicIp: ENABLED)
│
├── Public Subnet AZ-b   10.0.2.0/24   route: 0.0.0.0/0 → IGW
│   ├── ALB node
│   └── (ECS scale-out target)
│
├── Private Subnet AZ-a  10.0.11.0/24  local route only
│   └── RDS DB subnet group member — RDS instance runs here in V1
│
└── Private Subnet AZ-b  10.0.12.0/24  local route only
    └── RDS DB subnet group member — no running instance in V1
```

**RDS DB subnet group** spans Private Subnet AZ-a and AZ-b. AWS requires a DB subnet group to span at least two AZs. In V1, a Single-AZ RDS instance runs in AZ-a. The second subnet satisfies the group requirement and enables a future Multi-AZ upgrade without networking changes. The second subnet does not create a standby or incur RDS charges.

**Outbound internet access — API and Worker:**

Both ECS services require outbound HTTPS to external endpoints:

- **API:** Google OAuth provider endpoints (token exchange during OAuth callback flow). Does not call Gmail or Gemini directly.
- **Worker:** Gmail REST API and Gemini API on every processing cycle.

In V1, both ECS services run in public subnets with `assignPublicIp: ENABLED`. Outbound HTTPS routes through the Internet Gateway via the public subnet route table. This applies to both the API (OAuth) and the Worker (Gmail, Gemini).

> ⚠️ **V1 Trade-off — ECS tasks in public subnets**
>
> Placing ECS tasks in public subnets with public IP addresses is a deliberate V1 cost trade-off. It is not the conventional production posture.
>
> The conventional production posture places ECS tasks in private subnets and routes outbound internet traffic through a NAT Gateway. NAT Gateway incurs a fixed charge of approximately \$32+ per month, which is disproportionate to this project's current scale.
>
> The V1 approach mitigates the security exposure through security groups:
> - `sg-ecs-api` allows inbound only from `sg-alb`. The API task's public IP is not directly reachable from the internet.
> - `sg-ecs-worker` allows no inbound. The Worker task's public IP is not reachable from the internet.
> - RDS is in private subnets with no public accessibility. This is unaffected by the above trade-off.
>
> This trade-off is documented. When the project has real users and justified infrastructure cost, NAT Gateway should be added and ECS tasks should be moved to private subnets.

**Security groups:**

| Security group | Inbound | Outbound |
|---|---|---|
| `sg-alb` | TCP 443 from CloudFront managed prefix list | TCP `{CONTAINER_PORT}` to `sg-ecs-api` |
| `sg-ecs-api` | TCP `{CONTAINER_PORT}` from `sg-alb` | TCP 5432 to `sg-rds`; TCP 443 to 0.0.0.0/0 |
| `sg-ecs-worker` | None | TCP 5432 to `sg-rds`; TCP 443 to 0.0.0.0/0 |
| `sg-rds` | TCP 5432 from `sg-ecs-api`; TCP 5432 from `sg-ecs-worker` | None |

The ALB security group restricts inbound to the AWS-managed CloudFront origin-facing prefix list (`com.amazonaws.global.cloudfront.origin-facing`). This ensures the ALB is not directly reachable from the public internet, bypassing CloudFront.

**NAT Gateway:** Not used in V1. Explicitly deferred. See trade-off note above.  
**ALB for Worker:** No ALB for the Worker. The Worker has no HTTP interface and no inbound traffic.

### 9.5 Compute

**ECS Cluster**  
A single ECS cluster hosts both the API service and the Worker service. The cluster is a logical grouping; Fargate manages all underlying compute.

**API ECS Service**
- Launch type: Fargate (`awsvpc` network mode)
- Task size V1: 256 CPU units (0.25 vCPU) / 512 MB RAM — right-size after observing load
- Desired count: 1 (V1); ECS service auto-scaling added when multiple users justify it
- Target group: registered with ALB; ECS manages registration/deregistration on deploy
- Deployment: rolling update (`minimumHealthyPercent: 100`, `maximumPercent: 200`)
- Health check: `GET /health` → HTTP 200, evaluated by ALB
- Secrets: injected via ECS `secrets` field referencing Secrets Manager ARNs

**Worker ECS Service**
- Launch type: Fargate (`awsvpc` network mode)
- Task size V1: 256 CPU units (0.25 vCPU) / 512 MB RAM
- Desired count: 1 (V1); increased when higher job throughput is needed
- No load balancer; no target group; no inbound traffic
- Deployment: force new deployment; pg-boss visibility timeout recovers in-flight jobs
- Secrets: injected via ECS `secrets` field referencing Secrets Manager ARNs

**Application Load Balancer**
- Scheme: internet-facing
- Subnets: Public AZ-a and AZ-b (required for ALB multi-AZ availability)
- Listener: HTTPS 443 with ACM certificate; HTTP 80 → redirect to HTTPS
- Target group: IP type (Fargate `awsvpc`); health check on `/health`
- CloudFront: ALB DNS name configured as custom origin for the `/api/*` behaviour. This creates a **same-origin deployment** (React SPA and API share the same CloudFront domain), meaning CORS headers and preflight requests are inherently not required. CloudFront → ALB uses HTTPS.

### 9.6 Database

- **Engine:** PostgreSQL (RDS managed)
- **Instance class:** db.t4g.micro (V1 starting point — right-size after observing load)
- **Storage:** 20 GB allocated minimum; storage autoscaling enabled; storage type to be selected at implementation time using the current AWS-recommended option
- **Availability:** Single-AZ (V1). Multi-AZ standby is a V2 reliability upgrade.
- **DB subnet group:** Spans Private Subnet AZ-a and AZ-b. No standby instance in V1.
- **Public accessibility:** Disabled
- **Encryption at rest:** Enabled (AWS-managed KMS key)
- **Automated backups:** 7-day retention
- **In-transit encryption:** SSL enforced via Prisma connection string (`sslmode=require`)
- **pg-boss tables:** Co-located in the same database and schema as domain data

### 9.7 Secrets and Configuration

All sensitive application credentials are stored in AWS Secrets Manager. Non-sensitive configuration (such as the Google OAuth client ID) is stored as environment variables in the ECS task definition. No secrets are stored in Git, in Docker images, or in ECS task definition plaintext fields.

**Secrets Manager — sensitive credentials:**

| Secret | Contents | Consumed by |
|---|---|---|
| `career-companion/production/google-oauth` | `clientSecret` only (the OAuth client secret) | API |
| `career-companion/production/session` | `sessionSecret` (32+ byte random) | API |
| `career-companion/production/gemini` | `apiKey` | Worker |
| `career-companion/production/gmail-token-encryption` | AES-256 key (32 bytes) | API, Worker |
| `career-companion/production/database` | `host`, `port`, `database`, `username`, `password` | API, Worker |

**Non-sensitive configuration (ECS task environment variables — not Secrets Manager):**

| Configuration | Value | Notes |
|---|---|---|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID | Not a secret; safe to store as plaintext config |
| `NODE_ENV` | `production` | Runtime mode |
| `PORT` | Container port | Service configuration |

**Per-user Gmail OAuth tokens** are NOT stored in Secrets Manager. Secrets Manager holds static application credentials, not per-user data. Per-user access tokens and refresh tokens are stored encrypted in `GmailConnection` in RDS, encrypted at the application layer using the `gmail-token-encryption` key. The Worker fetches the key at startup, decrypts the per-user token from RDS when needed, calls Gmail, and discards the decrypted value from memory. Tokens are never logged, never returned in API responses, and never stored in plaintext.

**Local development** uses `.env.local` files (gitignored). No production credentials are accessible in local environments unless the developer has explicit Secrets Manager IAM access.

**S3 and CloudFront configuration:**

The S3 bucket hosting the React SPA is **private** — direct public access is blocked. CloudFront accesses the bucket exclusively through **Origin Access Control (OAC)**. The bucket policy grants `s3:GetObject` only to the specific CloudFront distribution via OAC. No other principal can read the bucket.

**SPA routing:** The React SPA uses client-side routing (TanStack Router). CloudFront must be configured to return the `index.html` for all routes rather than returning a 403/404 from S3 for non-root paths. This is done by configuring a CloudFront custom error response: HTTP 403/404 from S3 origin → return `/index.html` with HTTP 200.

### 9.8 IAM and Security

**No long-lived AWS access keys.** All AWS access uses IAM roles.

**ECS Task Roles (runtime):**
- API Task Role: `secretsmanager:GetSecretValue` on `career-companion/production/*`
- Worker Task Role: `secretsmanager:GetSecretValue` on `career-companion/production/*`
- No other permissions on task roles

**GitHub Actions Deploy Role:**
- Trusted via OIDC federation with GitHub's identity provider — no static credentials in GitHub Secrets
- Permissions: ECR authentication + image push; ECS task definition registration + service update; ECS run-task (migrations); S3 sync; CloudFront invalidation
- All permissions scoped to specific resource ARNs

**HTTPS:**
- Browser → CloudFront: HTTPS (ACM certificate on CloudFront distribution)
- CloudFront → ALB: HTTPS (ACM certificate on ALB listener)
- ALB → ECS API: HTTP within the VPC (HTTPS terminated at ALB; standard pattern)
- ECS → RDS: SSL enforced via Prisma `sslmode=require`

**Logs:** Structured JSON from API and Worker (pino or equivalent). Sensitive fields (tokens, keys, connection strings) explicitly excluded from log output. No raw SQL logging in production.

**Multi-user isolation:** Enforced at the application layer. All domain queries are scoped to `WHERE userId = :authenticatedUserId`. V1 does not use PostgreSQL row-level security (RLS).

### 9.9 Observability

**CloudWatch Log Groups:**

| Log group | Source | Retention |
|---|---|---|
| `/ecs/career-companion-api` | API container stdout/stderr | 30 days |
| `/ecs/career-companion-worker` | Worker container stdout/stderr | 30 days |

Log retention policies must be configured explicitly. The default is no expiry, which results in unbounded storage costs.

**Basic CloudWatch Alarms (V1):**
- ECS API service running task count < 1 → alert (API is down)
- ECS Worker service running task count < 1 → alert (Worker is down)
- RDS FreeStorageSpace < 2 GB → alert

**pg-boss job visibility:** Failed and dead-lettered jobs are queryable via SQL (`SELECT * FROM pgboss.job WHERE state = 'failed'`). This is the primary V1 tool for debugging pipeline failures.

**Not in V1:** Datadog, Grafana, distributed tracing, custom CloudWatch dashboards. CloudWatch is sufficient.

### 9.10 CI/CD Pipeline

**GitHub Actions → ECR → AWS deployment.**

```
On push to main:

1. validate
   — TypeScript check, ESLint, unit tests

2. build-api
   — docker build (career-companion-backend)
   — docker push to ECR (tags: :git-sha, :latest)

3. build-worker
   — docker build (career-companion-worker)
   — docker push to ECR

4. build-frontend
   — vite build (compiles to dist/)
   (S3 deployment happens after ECS deployments — see step 7)

5. migrate
   — ecs run-task (uses API image; command: prisma migrate deploy)
   — Task requirements:
       • Network: runs in a subnet with access to sg-rds (port 5432)
       • IAM: uses the API task role (secretsmanager:GetSecretValue for DB credentials)
       • ECR: pulls the API image — GitHub Actions OIDC role must have ecr:GetAuthorizationToken
       • CloudWatch: awslogs log driver configured (same log group as API service)
   — wait for task exit code 0; halt pipeline on failure

6. deploy-api
   — ecs update-service --force-new-deployment
   — wait for service stability

7. deploy-worker
   — ecs update-service --force-new-deployment
   — wait for service stability

8. deploy-frontend
   — aws s3 sync dist/ s3://career-companion-frontend-prod/
   — cloudfront create-invalidation /*
```

**GitHub Actions credentials:** OIDC token → `assume-role` on `github-actions-deploy`. No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in GitHub Secrets.

**Migration safety:** Migrations (step 5) complete before any new service version starts. All migrations must be backward-compatible: add columns with non-null defaults; never DROP a column in the same release that removes the corresponding code.

**Container registry:** Amazon ECR. Image lifecycle policy retains the last 10 images and removes older untagged images to control storage cost.

### 9.11 Reliability

**API crash:**  
ECS detects the stopped task and starts a replacement. ALB health check marks the unhealthy task out of rotation during replacement. The ALB DNS name and CloudFront URL remain stable. Expected disruption: 30–90 seconds. Sessions persisted in RDS survive the restart.

**Worker crash:**  
ECS detects the stopped task and starts a replacement. pg-boss jobs in-flight when the crash occurred become visible again after their `expireInSeconds` visibility timeout (configurable; default 15 minutes; can be set lower). The replacement Worker picks up the jobs. No jobs are lost. Domain promotion is idempotent — safe to re-run.

**AI job failure (Gemini error, timeout, schema violation):**  
pg-boss catches the exception and marks the job for retry with exponential backoff (`retryBackoff: true`, `retryLimit` configurable). After the retry limit is exceeded, the job enters permanent `failed` state (dead-lettered). Other jobs are unaffected. Dead-lettered jobs can be inspected and re-inserted after the underlying cause is resolved.

**Gmail API failure (rate limit, network error, token expiry):**  
The sync job fails; pg-boss retries with backoff. On 401 (token expiry), the Worker uses the refresh token to obtain a new access token, stores it encrypted in `GmailConnection`, and retries the request. On refresh token revocation, `GmailConnection.status` is set to `REVOKED`; the user must re-authorize.

**In-flight jobs during Worker deployment:**  
ECS stops the old Worker task before starting the new one. In-flight jobs expire via pg-boss visibility timeout and requeue. The new Worker picks them up. Domain promotion idempotency prevents duplicate events.

**Database migrations during deployment:**  
Migrations run as a one-off ECS task before any service deployment. The running service version continues operating against the migrated schema (backward-compatible migration discipline). If migration fails, deployment halts.

### 9.12 Environment Strategy

| Concern | Local development | Production |
|---|---|---|
| Database | Docker Compose PostgreSQL | RDS in private VPC |
| Secrets | `.env.local` (gitignored) | AWS Secrets Manager |
| OAuth callback URL | `http://localhost:3000/auth/callback` | `https://{domain}/auth/callback` |
| HTTPS | None (localhost HTTP) | ACM certificate via CloudFront + ALB |
| API container | `npm run dev` (ts-node watch) | Docker container on ECS Fargate |
| Worker container | `npm run dev:worker` (ts-node watch) | Docker container on ECS Fargate |
| AWS credentials | Developer's local credentials (limited scope) | ECS task role + GitHub Actions OIDC role |

**Staging:** Not created in V1. CI tests (TypeScript, lint, unit) provide the quality gate. Staging doubles infrastructure cost before the product has real users and is explicitly deferred.

### 9.13 Cost Model

> **These are approximate estimates only.** Actual costs depend on: AWS region, account free-tier eligibility and duration, compute utilization, RDS storage type and I/O, traffic volume, public IPv4 address count and uptime, CloudWatch log volume and retention, data transfer, ECR storage, and future AWS pricing changes. Verify all figures against current AWS pricing before committing to a budget.

| Service | V1 configuration | Approximate monthly range |
|---|---|---|
| ECS Fargate — API | 0.25 vCPU / 0.5 GB, 1 task | $9–15 |
| ECS Fargate — Worker | 0.25 vCPU / 0.5 GB, 1 task | $9–15 |
| Application Load Balancer | 1 ALB, low LCUs | $16–22 |
| RDS PostgreSQL | db.t4g.micro, Single-AZ, 20 GB minimum | $15–22 |
| Public IPv4 addresses | ~4 IPs: ALB (×2 AZs) + API task + Worker task | $12–15 |
| Secrets Manager | 5–6 secrets × $0.40/month | $2–3 |
| Amazon ECR | API + Worker images, lifecycle policy | $0–2 |
| CloudWatch Logs | <5 GB/month with 30-day retention | $0–3 |
| S3 + CloudFront | SPA static assets, low traffic | $0–2 |
| ACM, Route 53 | ACM free; Route 53 hosted zone if used | $0–1 |
| **Total estimate** | | **~$63–100/month** |

**Notable cost items:**

- **NAT Gateway** — Not included. If added (V2), this adds approximately \$32/month fixed plus data processing charges. The cost would increase the monthly estimate by roughly 30–50%.
- **Public IPv4 charges** — AWS charges per public IPv4 address per hour (approximately \$0.005/hour as of 2024). ALB requires one IP per AZ; ECS tasks with `assignPublicIp: ENABLED` each carry a charge. Estimated \$12–15/month total for the V1 IP set.
- **ALB** — The largest single infrastructure cost beyond compute. Justified because ECS Fargate task IPs are ephemeral; ALB provides the stable, health-checked DNS target that CloudFront requires.
- **CloudWatch log retention** — Default retention is no expiry. Explicitly set 30-day retention on all log groups to prevent unbounded log storage cost.
- **RDS Multi-AZ** — Not included. Enabling Multi-AZ approximately doubles the RDS line item.

### 9.14 Future Scalability

| Scaling concern | Evolution path |
|---|---|
| More API traffic | ECS auto-scaling (CPU/memory target tracking) increases API task count. ALB distributes traffic across tasks. No code changes. |
| More Worker throughput | Increase ECS Worker desired count (N Workers poll the same pg-boss queue). Increase pg-boss `teamSize` for in-process concurrency within each task. No code changes. |
| Database read load | Add RDS Read Replica. Route read-heavy dashboard queries via Prisma replica routing. |
| Database reliability | Enable RDS Multi-AZ. Automatic failover with ~60–120 second disruption. No application changes. |
| Private networking | Add NAT Gateway. Move ECS tasks from public to private subnets. Update route tables and security groups. No application code changes. |
| Queue throughput | pg-boss is not the expected bottleneck at any realistic Career Companion scale. |
| Extreme database scale | Migrate from RDS to Aurora PostgreSQL. Connection string change; application code unchanged. |
| Observability maturity | Add CloudWatch Alarms, dashboards, and metric filters. Introduce structured log parsing when log volume justifies it. |



## 10. Deferred / Not V1

### Application

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
| PostgreSQL row-level security (RLS) | Application-layer isolation is enforced in V1; RLS adds schema complexity without clear V1 benefit |

### Infrastructure

| Item | Reason |
|---|---|
| NAT Gateway | Fixed cost (~\$32+/month) disproportionate to current project scale; ECS tasks use public subnets as documented V1 trade-off |
| RDS Multi-AZ | Approximately doubles RDS cost; single-AZ with automated backups is acceptable for personal-project V1 |
| Aurora PostgreSQL | 3–5× more expensive than RDS at low utilization; no feature benefit at this scale |
| ECS auto-scaling | Single task sufficient at personal-project scale; add auto-scaling when traffic justifies it |
| ALB for Worker | Worker has no inbound traffic; no load balancer needed or appropriate |
| Staging environment | Doubles infrastructure cost; CI tests provide the quality gate before real users |
| Datadog / external observability | CloudWatch sufficient for V1 |
| AWS WAF | Not justified at personal scale; CloudFront provides basic DDoS protection |
| ElastiCache / Redis | pg-boss uses PostgreSQL for the job queue; no caching requirements in V1 |
| Kubernetes / EKS | Severe over-engineering for two containerized services |
| AWS App Runner | No longer accepting new customers per AWS announcement; ECS Fargate + ALB is the equivalent |
| Kafka / SQS | pg-boss covers all queue requirements; no additional queue infrastructure needed |
| CloudWatch dashboards / metric filters | Basic alarms sufficient for V1; dashboards added when operational need arises |
| RDS Read Replica | Single-instance RDS sufficient at personal-project read volume |

---

## Document Versioning

| Version | Date | Notes |
|---|---|---|
| 0.1 | 2026-09-05 | Initial draft. All decisions from COM-6 review session incorporated. |
| 0.2 | 2026-09-05 | ChatGPT review corrections: (1) explicit matching branch in pipeline, (2) AI_AUTO provenance flagged as COM-5 amendment, (3) phase ownership constraint reworded, (4) pricing/free-tier assumptions removed from architecture. |
| 0.3 | 2026-09-06 | COM-6 amendment: full Cloud & Deployment Architecture added (Section 9). AWS-only deployment with ECS Fargate + ALB (API), ECS Fargate (Worker), RDS PostgreSQL, S3/CloudFront, ECR, Secrets Manager, CloudWatch, IAM/OIDC. V1 public-subnet trade-off documented. Section 10 split into Application and Infrastructure deferred items. |
| 0.4 | 2026-09-06 | Final corrections: (1) API OAuth outbound access documented accurately; (2) stale Railway/Render/Vercel/Supabase references replaced throughout with AWS equivalents; (3) S3 OAC + private bucket + SPA routing documented; (4) Google OAuth clientId moved to config, only clientSecret stored in Secrets Manager; (5) RDS storage type de-hardcoded; (6) migration task network/IAM/ECR/logging requirements documented; (7) cost model disclaimer expanded. |
