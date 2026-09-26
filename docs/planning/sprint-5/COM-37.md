# COM-37 — Establish the preserved-dataset baseline and Sprint 5 readiness gates

Status: proposed local ticket; not implemented. Repository owner: documentation, with backend/frontend environment evidence. Suggested priority: High. Estimate: 3 points. Scope authority: the supplied Sprint 5 brief; see the COM-37 identifier conflict in [conversion notes](linear-conversion.md).

## 1. Title

Establish the preserved-dataset baseline and Sprint 5 readiness gates.

## 2. Goal

Create a reproducible, non-destructive starting record for the existing approximately 1,620 emails, current stabilized code, and verification environments. An engineer must be able to prove later that new Gmail changes were added without deleting/replacing historical records or replaying completed AI work.

## 3. Context

Sprints 1–4 are complete. The 2026-09-26 local stabilization added Gmail history paging, async sync, leases, AI claims, owner isolation and 20-item pagination. It already recorded isolated test evidence; it did not validate the user's real mailbox or database. The audit report also says its commits were not integrated into remote main. The supplied `*-main` snapshots are not the implementation baseline. Record actual deployment state before assuming local fixes are active.

## 4. Current implementation

- `backend/prisma/schema.prisma`: emails, Gmail connection/history/lease, AI operations/budgets, application/events/actions and delivery state.
- `backend/STABILIZATION.md`: non-destructive rollout, legacy-data preflight, configuration and reconciliation instructions.
- `backend/src/tests/setup.ts` and `src/utils/testDatabase.ts`: tests override from `.env.test`, require exact `DATABASE_URL == TEST_DATABASE_URL`, and permit only a specifically named local test database.
- `backend/src/index.ts`: starting a non-test API also starts all three workers; `/health` only returns HTTP liveness.
- `frontend/scripts/smoke-stabilization.mjs`: isolated fixture creation/deletion and direct sync completion; unsuitable for the preserved dataset.
- `docs/docs/engineering/stabilization-audit.md`, `docs/docs/architecture/mvp-architecture.md`, `docs/PROJECT_CONSTITUTION.md`: current decisions and known evidence limits. See [architecture review](architecture-review.md) for exact repository commits and aliases.

## 5. Problem / gap

There is no current owner-scoped manifest/count, credential/deployment readiness record, or classification of historical pending/failed work for Sprint 5. “Approximately 1,620” alone cannot detect deletion followed by replacement. Old architecture and AI documentation can also mislead an engineer into resetting processing, treating SDK retries as harmless, or expecting separate promotion jobs.

## 6. Proposed implementation

1. Record backend/frontend/docs Git SHAs, clean/dirty state, runtime artifacts and worker instances. Verify the intended stabilization commits are included in the runtime. Any missing integration/deployment is an external prerequisite, not permission to deploy during planning.
2. Identify the actual dataset owner and same Gmail mailbox privately. Inspect migration identities/table-column presence read-only, then select only fields available in that schema; a legacy eight-migration dataset lacks stabilization lease/AI-operation fields. Record measured count `N0`, scope, observation time, connection status, history anchor and last successful sync. Export the baseline identities and immutable metadata as described in the runbook, using an explicit read-only transaction **before** any later authorized schema rollout. Record absent stabilization state as unavailable and retain the rollout gate; never migrate just to collect a baseline.
3. Record distributions for processing/relevance/match states, existing user-confirmed matches/ignores and user application status, completed AI results/operation attempts, and pending/active/retry/failed queue work. Do not process or reset backlog to simplify the record.
4. Verify a protected backup exists and a restore has been checked in an isolated copy; record evidence location and timestamp, not credentials. Review the existing migration preflight on the restored copy. If real data/schema is incompatible, document the exact gate; do not delete duplicates or execute migrations in this ticket.
5. Establish two named verification lanes: a disposable guarded local test DB for fixture/fault tests, and the original preserved database/mailbox for supervised live verification. Prevent test harnesses, fixture cleanup, seed/reset and fault injection from targeting the latter.
6. Record non-secret configuration readiness: OAuth redirect/cookie/proxy behavior, encryption-key continuity (presence only), same-mailbox grant, Gemini budget policy/backlog, Discord disabled for tests, and working worker registration. Do not print environment files or tokens.
7. Reconcile stale docs at the affected boundaries and record the conflict between the unmatched-email COM-37 comment/test and this brief's COM-37. Preserve the five IDs in this pack; do not silently overwrite an unrelated Linear issue later.

Deliver a completed private baseline record plus a sanitized readiness summary using the [runbook evidence template](verification-runbook.md). This ticket is preparation and documentation; no product behavior change is required.

## 7. Architecture impact

None to runtime design. It establishes the exact deployed boundary and documents that API and workers currently share `src/index.ts`, that sync is asynchronous, and that ingestion completion differs from AI/domain completion. Missing prerequisites become explicit gates for COM-38, not new product features.

## 8. Data impact

Read-only inspection of `users`, `gmail_connections` (exclude token columns), `emails`, `applications`, `application_events`, `actions`, `ai_processing_results`, `ai_operations`, `ai_call_budgets`, `notification_deliveries`, migration metadata, and queue metadata. No DML, index, constraint or schema change. Include identity sets and counts; AI output and email subject/sender exports stay private. Migration state must be measured, not inferred from schema files.

## 9. API impact

No contract change. Capture authenticated `GET /api/gmail/status` (200 with connected/status/syncStatus/lastSyncedAt and optional syncError); 401 indicates session failure. Read email/application lists via `limit/offset`, default/max 20, following `metadata.nextOffset` until null. No public total count exists. Use the private database manifest for exact counts; a moving offset list is not a data-integrity snapshot. Do not trigger sync in this preparation ticket.

## 10. Background-job impact

No job creation, retry or worker restart required. Inventory Gmail/email/Discord queue states and distinguish queued, active, retrying and terminal jobs. Record the baseline worker configuration and whether old workers remain. Avoid starting `npm run dev`/`npm start` merely to inspect configuration: both start workers and can process existing backlog. No queue purge or bulk re-enqueue.

## 11. Security/privacy considerations

Manifest and backup locations must be access-controlled and excluded from Git/Linear. Public evidence contains aggregate counts, hashes, redacted account labels and non-secret runtime versions. Do not export OAuth tokens, session cookies, webhook URLs, raw bodies or structured AI payloads into tickets. Keep other users' data outside the test owner's evidence, while checking their aggregate invariants separately where authorized.

## 12. Testing strategy

No new runtime unit test is required for a documentation baseline. Inspect the existing DB safety guard and fixture cleanup before later tests. Validate the manifest comparison method with synthetic local files or an isolated fixture database during execution. Backend/frontend regression tests and migration fixtures belong to the separate test lane; real-provider proof belongs to COM-38. A clean-install result cannot substitute for the existing-data baseline. Current audit test counts are historical until rerun.

## 13. Acceptance criteria

- [ ] Exact checkout/runtime SHAs and worker versions are recorded; any missing stabilization rollout is explicitly unresolved.
- [ ] The original dataset owner/mailbox is identified, measured `N0` is recorded, and any difference from approximately 1,620 is explained without modifying data.
- [ ] A complete identity/immutable-metadata manifest, protected backup reference and tested-restore evidence exist; count-only evidence is rejected.
- [ ] Existing processing backlog, completed AI checkpoints, operation attempts/budget and explicit user decisions are captured.
- [ ] Test and live databases are distinctly documented; existing destructive fixture scripts cannot be directed at the baseline as part of this plan.
- [ ] All ten migration identities are compared with the actual runtime database in a read-only check; no migration is executed by this ticket.
- [ ] Provider/session/environment readiness and missing inputs are explicit; no credentials are included.
- [ ] Stale touched documentation and the COM-37 identifier conflict have a recorded resolution or conversion gate.

## 14. Dependencies

No Sprint 5 ticket dependency. Requires access to the three inspected repositories, authorized read access to the preserved DB, a protected backup/restore record, deployment metadata, and a mailbox owner able to perform the later controlled change. Stabilization integration/migration rollout is an external gate if absent. Planning can finish with open environment gates; this ticket cannot be accepted as live-ready until those gates are resolved.

## 15. Non-goals

No source edits, sync execution, production migration, seed, database reset, historical cleanup, new authentication system, infrastructure deployment or Linear mutation. Do not expand this into a general documentation rewrite or Sprint 6 work.

## 16. Failure/recovery scenarios

Wrong DB identity: stop the baseline operation before querying/exporting content. Count mismatch: investigate owner/environment and recent legitimate additions, preserving all rows. Missing migration/integrity prerequisites: keep live verification gated and follow the existing release guide in a separate rollout. Missing token/key: document readiness failure without overwriting credentials. Active backlog: record it and agree a controlled live window; do not relabel records as completed. Missing provider access never justifies replacing the dataset with fixtures.

## 17. Verification commands

From each repository: `git status --short --branch` and `git rev-parse HEAD`. From backend: `npm run typecheck`; the repository-documented `npx prisma validate` and `npx prisma migrate status` are read-only schema/status checks when pointed deliberately at the intended environment. Do not run `migrate deploy`, `db:migrate`, `db:seed`, `db:reset` or any test as a shortcut to establish this baseline. See [command matrix](verification-runbook.md) for later isolated-test commands. No commands were executed against a DB during planning.

## 18. Documentation updates

Update `docs/docs/engineering/stabilization-audit.md` with a dated link to this baseline without rewriting historical evidence; `docs/docs/architecture/mvp-architecture.md` with confirmed runtime/reliability limits; `docs/docs/architecture/high-level-architecture.md` where separate worker/promotion and retry statements conflict with code; `docs/docs/architecture/email-ai-pipeline.md` and `docs/docs/domain/domain-model.md` where reset/upsert guidance conflicts with durable AI claims. Add a sanitized readiness record beside this pack; keep raw manifests outside the repo. Update backend `STABILIZATION.md` only if measured rollout facts require clarification.

## 19. Engineering notes

Do not infer deployment readiness from `/health` or local Git state. Prefer a short explicit readiness record over a new inventory tool. The baseline is not an instruction to freeze all legitimate user activity forever; capture a controlled observation window and attribute concurrent additions/processing changes. Source-path aliases and snapshot hashes are defined in this pack so later readers can recover the exact code being planned against.
