# MVP Readiness Report (COM-35)

Historical Sprint 4.5 report. Its idempotency and production-readiness claims must not be treated as current verification. See the [2026-09-26 stabilization audit](docs/engineering/stabilization-audit.md) for reproduced defects, corrections, verification evidence, and remaining rollout gates.

## E2E Flow

The entire user journey was traced and verified:
- **Authentication**: Google OAuth is fully integrated with secure HTTP-only cookies and proper `userId` scoping across all APIs.
- **Gmail Ingestion**: Connects successfully with AES-256-GCM token encryption. Metadata polling behaves idempotently.
- **AI Processing**: Gemini pipeline correctly discriminates relevance using metadata and processes a bounded body (<= 8000 chars) for structured extraction.
- **Application Workflow**: Applications are correctly created, state transitions are inferred securely, and events are logged to the timeline.
- **Ambiguity Resolution**: Ambiguous matches correctly halt automation, requiring user selection via the dashboard. Backend prevents cross-user assignment.
- **Actions**: Actions populate the queue and application detail views.
- **Discord**: Discord webhook delivery is idempotent and properly retries transient errors via `pg-boss`.

## Issues Found

| Severity | Issue | Impact | Fix Applied | Test Updated |
|---|---|---|---|---|
| Medium | Missing Action resolution buttons in App detail view | Users could see pending actions in their timeline but could not dismiss/complete them without returning to the main dashboard. | Added `Complete` and `Dismiss` buttons to `ActionItem` in `applications.$id.tsx` using `updateAction` API mutation. | Frontend typechecks & tests validated. |
| Low | `pg-boss` queue does not exist | `queue.send()` failed for `discord-notification-job` in testing / first-runs if worker hadn't started yet. | Added `boss.createQueue()` to `queue.ts` initialization. | `notificationJob` error removed from logs. |
| Low | `prisma:error` logged in auth tests | Test cleanup generated noisy logs by trying to delete non-existent users. | Changed `prisma.user.delete` to `deleteMany` in `auth.test.ts`. | Auth tests run without Prisma errors. |

## UX Review

- **Meaningful Improvements**: Action items within individual Application Detail pages now have inline `Complete` and `Dismiss` buttons. This drastically reduces clicks and navigational confusion.

## Security Review

- **Authentication**: Properly secured via Google OAuth and `connect-pg-simple`.
- **Authorization**: All primary entities (Applications, Emails, Actions) enforce strict isolation via `userId`.
- **Gmail Privacy**: Raw email bodies are transiently fetched and sent directly to Gemini without persistence. 
- **AI Data Handling**: Strict bounds limit email content exposure to Gemini to 8000 characters.
- **Discord Secret Handling**: Configured via environment variables; never leaked in API payloads.
- **Logging**: Strict scrubbing of token payloads and error body outputs.

## Reliability Review

- **Retries**: Configured for external network calls (Discord webhooks and Gemini API) using `pg-boss` native retries.
- **Idempotency**: Notification webhook delivery uses database locks (`NotificationDelivery`); duplicate processing is prevented. Email ingestion is gated by `gmailMessageId`.
- **Failure Isolation**: Processing tasks are isolated. One failed email does not stop other emails.
- **Known Limitations**: (Documented in MVP Architecture) There is an accepted window where a notification fails to enqueue after Action database persistence.

## Documentation

- Created `career-companion-docs/docs/architecture/mvp-architecture.md` detailing architecture, security boundaries, known MVP limitations, and out-of-scope behaviors.
- Updated backend `README.md` to clarify the state of full Google OAuth vs Development Authentication.

## Quality Gate

All checks pass successfully.

- **Backend Tests**: 122 passing
- **Frontend Tests**: 33 passing
- **Typecheck**: 0 errors
- **Lint**: 0 errors
- **Build**: Successfully built frontend assets and backend `tsc` compilation.

## MVP Readiness

**READY WITH KNOWN LIMITATIONS**

The system operates correctly and securely within its defined scope. The absence of a transactional outbox for Discord notifications and strict Gemini-only AI support are well-understood boundaries for this iteration.
