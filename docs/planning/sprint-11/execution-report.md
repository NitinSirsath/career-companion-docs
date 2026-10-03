# Sprint 11 engineering execution — 2026-10-03

Status: local engineering implemented and verified. Sprint 10 local verification preceded Sprint 11 implementation. Existing tickets retained. Real-world owner/Gmail/MCP acceptance remains deferred and is not a development blocker.

## Delivered

- S11-01: USER-origin follow-ups extend Action with a durable unique creation receipt and initial normalized payload hash. Replay returns the same identity after edits; changed payload conflicts and foreign identity stays private. Owned receipt lookup reconciles response loss. Captured revisions protect personal edit/status and email evidence stays read-only.
- S11-02: snooze preserves the original deadline and PENDING state. SQL buckets/counts partition the full active dataset, with future snoozes separate from All. Global refresh considers earliest expiry; wake is derived from reads without a job or send. Correction carries prior decisions to new targets; target decisions win on reactivation; personal work does not move with email.
- S11-03: archivedAt/archiveRevision provide reversible visibility independent of status. Default lists/workspace/agenda omit archived applications; explicit filters and detail retain access. Existing linked mail continues tracking, new company/role matches exclude archived targets, new archived-only MCP submissions enter review, and old receipts retain their identity. Explicit new links and user edits require restore.
- S11-04: follow-up/edit/snooze/archive controls, keyboard interaction, narrow layout, canonical revision-aware cache updates and archive membership. An uncertain creation keeps the exact draft/request across closing and reopening; receipt lookup never resubmits. A deliberate retry after missing receipt uses the same request and warns that absence is not proof of failure.

OD-19/20 are accepted for this bounded implementation; ADR-0002 records the archive intake extension. No new notification scheduler, sending feature, MCP tool, provider promotion, checkout or remote workflow.

## Verification

| Check | Actual result |
| --- | --- |
| Backend suite | 58 files / 806 tests passed |
| Frontend suite | 19 files / 209 tests passed |
| Builds/typechecks | Both passed |
| Lint | Both passed; existing warnings only (backend 7 warnings) |
| Shared contracts | All 13 files match byte-for-byte: original 11 plus agenda and temporal |
| Fresh + upgrade migrations | Follow-through, status, AI and MCP preservation suites passed on eight fresh guarded fixture databases |
| Browser | Passed with real local API/DB/queue/workers, synthetic external adapters and outbound blocked; receipt recovery, snooze, archive/restore, unsnooze/completion, agenda and inherited recovery/MCP scenarios |
| Teardown | HTTP and worker drains proven before cleanup; users/jobs/budgets/MCP residue 0/0/0/0 |
| Git | No staging/commit/branch change/push/PR; initial HEADs retained |

Actual outputs and fixture-only screenshots are in [evidence](evidence/). [Sprint 10 report](../sprint-10/execution-report.md) retains the preceding verification stage. Final file inventory and review: [engineering review](engineering-review.md).

## Tests and review findings

Backend follow-through tests cover concurrent same-key creation, replay after edit, changed payload/foreign identity, stale/no-op intent, email evidence protection, 25-row snooze pagination and read-time wake, durable notification suppression, archive preservation, linked-thread intake, MCP receipt replay and archived-only review, action correction, and observed row-lock interleavings for MCP/archive and personal edit/archive. Notification tests cover both suppression through restore and archive after claim before send. Agenda tests preserve date/DST uncertainty, immutable evidence, v2/v3 adoption and correction serialization. The evaluator rejects partial/refused qualification and unsafe invented timing/event mutation output.

Frontend regressions cover uncertain draft receipt recovery and same-key deliberate retry, conflict preservation, unsnooze revision, independent archive/status cache guards, agenda uncertainty and canonical cache membership. Browser review found a real archive-filter regression in uncertain application creation: reconciliation refreshed an obsolete list key. It now restores the visible active/unfiltered list and has a focused regression test. Archived detail no longer shows the misleading load-error copy from the disabled status editor.

Smoke-harness fixes: the combined fixture already has 25 older undated actions, so the new follow-up is on page 2 before/after snooze. The inherited MCP revocation check now awaits the actual DELETE response before testing rejection; matching generic Revoked text alone could race the server write. These are test synchronization corrections, not relaxed product assertions.

## Notification boundary

Archive/snooze create or terminally suppress unclaimed eligible delivery receipts; late queued work cannot send after restore/expiry. Already claimed work is checked again immediately before send. An external send already in flight cannot be retracted. Delivered/uncertain history remains intact; restoring or unsnoozing never queues a catch-up delivery. No exactly-once delivery claim is made.

## Reproduce locally

Use only the existing sibling repositories. Backend: `TEST_ENV_FILE=.env.sprint10.test npm test`, `npm run build`, `npm run lint`. Frontend: `npm test`, `npm run build`, `npm run lint`; `npm run sync-contracts` reads the local sibling backend. Fixture env files are ignored, local and contain no real-provider keys; existing `.env.test` is unchanged.

For new empty guarded disposable databases, run `scripts/verify-sprint-migrations.cjs` with FRESH_DATABASE_URL, UPGRADE_DATABASE_URL and FIRST_NEW_MIGRATION=20261003110000_personal_follow_through. It snapshots every pre-existing column across all tables, including confirmed agenda rows, before the additive upgrade. Reuse the existing status/AI/MCP migration scripts with their explicit fresh/upgrade fixture URLs; they refuse unsafe targets. Never point these at personal data.

The browser runner is frontend `scripts/smoke-stabilization.mjs`, with SMOKE_DATABASE_URL pointing to an exclusive migrated `career_companion_*smoke_test` database, BACKEND_DIR set to the existing backend if needed, and SMOKE_SCREENSHOTS pointing here. It uses the built UI, real local API/PostgreSQL/pg-boss/workers and synthetic Gmail/Gemini boundaries; all other outbound calls are blocked. This is not the deferred real-world Gmail/MCP acceptance.

## Deferred and rollback

Extraction/v3 remains default-off. Two complete real model-specific qualification runs, existing provider prerequisites/disclosure, activation decision, owner usability trial, original-data preservation, actual scheduled days and real Gmail/automation-client acceptance remain pending. No personal database was migrated. Remote CI/deployment is unverified and outside scope.

Rollback retains both additive migrations and receipt/revision/version-aware readers. Disable new v3 selection rather than downgrade to an old binary that can lose selected/held operation identity. Do not drop agenda, intent or receipt columns to roll back UI exposure. Existing S9 work and previous migrations are preserved.
