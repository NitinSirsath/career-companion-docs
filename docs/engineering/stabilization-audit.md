# Stabilization audit — 2026-09-26

Status: implementation and local verification complete; GitHub integration is blocked by repository permissions. Production deployment/provider verification and the original Sprint 5/6 plans remain external gates. This audit does not replace the roadmap.

## Repository state

The supplied `*-main` folders were snapshots without `.git`. Fresh sibling clones preserved them unchanged and matched their tracked contents. GitHub remains the source of truth; all default branches are `main`.

| Repository | Baseline main | Audit branch |
| --- | --- | --- |
| backend | `b60f4569d4ecd76c960d9426ddfe01a97da426c6` | `fix/stabilization-audit` |
| frontend | `f675abe98dca4c4815bba934fb360882d7d20e97` | `fix/stabilization-audit` |
| docs | `ea7a475341302f93977938981967da659ebd1c1b` | `docs/stabilization-audit` |

At intake, each repository had one merged PR; every merge commit was an ancestor of main. No open/unmerged PRs or local-only/remote-only commits were present. Remaining COM-48 remote branches were ancestors of main (backend two commits behind, frontend three). No prior work was discarded.

Final integration evidence: the backend branch contains four focused commits ending at `c7e33f5`; the frontend branch ends at `11e468d`. Both are clean, reviewed local commits. The backend push returned HTTP 403: the authenticated `nitin-stockypro` account has `push: false` for all three repositories. No PR was created, no remote audit branch was created, and all default branches remain at the baseline hashes above. This report is committed locally on `docs/stabilization-audit`. The intended fixes are **not yet in GitHub main**; an account with write access is required to finish PR review/merge. Prepared PR descriptions are available locally; no fork or alternate publication was created to work around permissions.

## Findings, decisions and implemented corrections

| Priority | First failing boundary / root cause | Correction and invariant |
| --- | --- | --- |
| P0 AI cost | Existing AI results were reset before every run. Row uniqueness did not protect external calls; SDK retries multiplied queue retries. | Durable operation/version claims and validated checkpoints; completed/legacy work reused. Three attempts only for explicit transient rejection; SDK one attempt, daily application budget/cooldown, bounded input/output. Uncertain outcomes require reconciliation, including provider success followed by DB failure. Domain failure cannot replay AI. |
| P0 Gmail | Stored history ID was unused. List scans could rediscover everything; insert/enqueue failure stranded records; SYNCING could survive a crash indefinitely. | History paging, 404 fallback, checkpoint before initial scan and commit after ingestion, pending recovery, atomic lease, bounded worker and 202 API. Refreshed credentials persist conditionally; auth failures and quota errors differ. Mailbox switching is rejected to preserve message identity. |
| P0 auth/isolation | Dev header auth depended only on a flag; production could use fallback secrets. Email-based OAuth linking lacked verified-email/identity checks. Matching changed the email before checking application ownership. | Production fails closed, login rotates/persists session and checks verified identity. Ownership checks precede mutations; DB triggers reject cross-owner links and owner transfers. Global Discord webhook explicitly bound to one owner. |
| P1 domain/jobs | Read-then-create effects raced. Notification was sent before its delivery claim. Queue callers could observe incomplete initialization. | Transactional row locks plus event/action uniqueness; explicit user decisions preserved. Durable notification claim before sending, bounded retries, uncertain delivery held. Shared queue initialization and singleton windows supplement database safety. |
| P1 pagination/API | Service defaults were 50, sublists unbounded, timestamp sorting incomplete. Detail and matching selectors relied on the first list page; completed actions could hide pending work. | All seven lists default/cap 20; strict numeric/status/UUID validation; ID tie-breakers; pending-first action order; scoped detail endpoint; paginated sublists/selectors; shared response contracts and count queries. Existing offset contract retained. |
| P1/P2 reliability | Browser treated sync as synchronous; empty-page returns hid navigation; some mutations lacked invalidation/errors. Test URL guard accepted incidental `test` text. | Background status/processing polling, visible failure/retry states, invalidation, navigable empty pages; exact explicit local test DB identity before destructive fixture setup; sanitized provider/job errors. |

The architecture remains React/Query → Express/services → Prisma/PostgreSQL, with pg-boss and Gemini/Discord adapters. No new infrastructure, providers, billing, teams or product features were added. The existing design system was preserved; unrelated visual cleanup was deferred.

Alternatives rejected: in-memory deduplication, queue uniqueness as the sole cost boundary, blanket retries after uncertain external effects, generic workflow/pagination frameworks, and a broad outbox refactor. PostgreSQL claims and transactions fit the existing system. Exactly-once external delivery cannot be promised without a provider idempotency contract; holding uncertain work is the explicit tradeoff.

## Verification evidence

All database checks used a separately initialized local PostgreSQL 15 cluster on port 55439. No development/production database or real provider credentials were used. The final suite used an empty `career_companion_verified_test`; upgrade/failure fixtures used `career_companion_migration_test`.

| Gate | Actual result |
| --- | --- |
| Backend `npm run typecheck`, `npm run build` | Pass |
| Backend `npm run lint` | Pass, 0 errors; 11 existing unused-disable warnings |
| Backend `npm test` | 19 files, 177 tests pass |
| Frontend `npm run typecheck`, `npm run build` | Pass; existing large-chunk advisory remains |
| Frontend `npm run lint` | Pass, 0 errors; 20 warnings |
| Frontend `npm test` | 8 files, 51 tests pass |
| Prisma validate/status | Valid schema; all 10 migrations applied |
| Fresh migrations | All 10 applied to an empty isolated database |
| Upgrade / failure / recovery | Eight baseline migrations plus legacy AI/duplicate fixtures; integrity migration rejected duplicate data, preserved both rows and rolled back its DDL; explicit fixture reconciliation and failed-migration resolution allowed successful upgrade |
| Migration/schema drift | `prisma migrate diff --from-url … --to-schema-datamodel prisma/schema.prisma --exit-code`: no difference |
| Legacy AI migration | Completed legacy result adopted after upgrade with Gemini replaced by a throwing sentinel; zero new operation records |
| Built browser smoke | Real API/PostgreSQL and built UI: list pages, direct older application, timeline paging, action mutation, queued sync/polling, mobile overflow and no JS errors; external browser requests blocked |
| Diff/config review | Shared contracts synchronized; no generated client/router edits, dependency/lockfile changes, real credentials or environment files included; migrations/configuration reviewed |

Regression coverage includes concurrent AI/notification execution, persistence-after-provider failure, ambiguous network outcomes, retry ceilings and budget rollback; deterministic filtering and uncertain classification; incremental Gmail/deduplication, partial failure, token refresh and disconnect races; direct DB ownership rejection, matching concurrency, user ignore decisions, every list limit and malformed pagination; independent detail loading, older application selection, empty pages and mutation recovery. Real pg-boss initialization/singleton behavior was exercised. Test count is evidence of the executed suite, not a completeness claim.

The auth test fixture collision discovered during focused reruns was fixed by giving the nested development-auth fixture its own identity. The full run passed; after the final conditional identity-link race guard, the 10 authentication tests, typecheck and build passed again. No live Gmail/Gemini/Discord call, real Google login or production migration was performed. The browser sync completion was a controlled fixture, not an external-provider E2E claim.

## Migrations, configuration and recovery

Two new migrations add AI operation/budget state, sync leases, notification claims, domain uniqueness and ownership triggers. Existing duplicate/cross-owner data blocks deployment without deleting it. See the backend [STABILIZATION.md](https://github.com/NitinSirsath/career-companion-backend/blob/main/STABILIZATION.md) for read-only preflight SQL, rollout sequence, recovery and test commands.

Production requires strong signing secrets, HTTPS URLs, development auth disabled and the correct trusted proxy setting. `AI_DAILY_CALL_LIMIT` defaults to 100 calls across users per UTC day (0 disables new calls); it is not a currency budget. `DISCORD_USER_ID` binds the existing webhook to its intended owner. Drain old workers before migration/deployment and release the frontend/backend together for the changed sync and paginated sublist contracts.

Privacy boundaries: IDs/sender/subject/timestamps and encrypted OAuth tokens are stored; snippets/raw bodies are transient. Structured AI checkpoints and application records can contain personal information and persist until parent deletion. Disconnect clears credentials but retains domain records. No retention period or deletion feature was invented.

## Remaining issues and human review gates

- **GitHub integration:** Provide write access, push the reviewed audit branches, create/review the coordinated PRs, merge to default branches, and verify ancestry and clean checkouts. Remote main currently does not contain these fixes.
- **Production/provider evidence:** Back up and inspect real data, review migration preflight, confirm proxy/cookie/OAuth configuration, and supervise a small-budget live Gmail → AI → matching flow plus one intended-owner Discord delivery. Local tests cannot establish these deployment facts.
- **Uncertain/legacy work:** PROCESSING/UNKNOWN AI claims, legacy partial results and claimed uncertain notifications require reconciliation. Completed checkpoints are reusable; resetting claims or changing versions to force replay is unsafe without deliberate review. Budget/queue exhaustion may leave work requiring operator resumption.
- **Notification durability:** A crash after action commit but before queue send can lose a notification. Matching replay can enqueue an existing action; a full transactional outbox remains deferred as the previously accepted noncritical limitation.
- **Bounded ingestion:** Very large first scans may need multiple syncs; pending recovery takes at most 100 records per sync. Existing 90-day INBOX scope remains. Mailbox switching is unsupported.
- **Pagination:** Offset pages can shift under concurrent inserts/status changes. Stable ordering prevents tie ambiguity, not snapshot inconsistency. Search/general filtering and total counts are not part of the existing contract.
- **Privacy:** Agree retention/deletion policy before wider public release. Structured extraction is not anonymous data merely because raw bodies are absent.
- **Roadmap access:** Original Sprint 5 and Sprint 6 acceptance criteria are absent from the repositories; available Linear URL opens a sign-in screen. No sprint scope or Linear status was invented or changed.

## Deferred work and stopping rule

Defer general UI restyling, warning/deprecation cleanup, dependency upgrades, bundle tuning, broader search/filtering, Unicode matching improvements, new providers/channels, notification outbox and retention UX unless an original sprint explicitly requires them. These discoveries are not additions to Sprint 5 or Sprint 6.

The required local P0/P1 safety gates now pass. Stop architectural cleanup and resume product work once the authoritative plan and relevant deployment/verification prerequisites are available:

1. Re-evaluate the **original Sprint 5** against durable AI claims, async sync and paginated APIs. Preserve product intent; change dependencies only with concrete architectural evidence.
2. Implement Sprint 5 only after its relevant gates pass, using **Understand → Inspect → Diagnose → Plan → Implement → Verify → Review**.
3. Re-evaluate **Sprint 6 after Sprint 5** using the same workflow and original intent.
4. Record unrelated improvements separately; do not convert this audit into an endless refactor or a replacement roadmap.

## References

- [Gmail sync/history expiration](https://developers.google.com/workspace/gmail/api/guides/sync).
- [Gmail error classifications](https://developers.google.com/workspace/gmail/api/guides/handle-errors).
- Installed `@google/genai` retry-option documentation describes five attempts by default; this implementation explicitly configures one.
