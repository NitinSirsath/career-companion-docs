# Sprint 6 verification runbook

Status: reconciled future execution procedure, 2026-09-27, not an execution report. No database changes, implementation tests, worker runs or live-provider checks were performed while saving this pack. The Node procedure below is a documented composition of existing tools, not a new npm script.

## Entry evidence and execution lanes

Complete the [Sprint 5 entry gate](README.md#b-why-this-sprint-follows-sprint-5). Record backend/frontend/docs HEAD plus preserved working diff/untracked source identity; Sprint 5 implementation and follow-up fixes are uncommitted. Link its [execution report](../sprint-5/execution-report.md), including real-worker and regression evidence, then record final implementation commits when they exist. D1/D2 and retained live Gmail/original-data evidence gates remain unresolved; do not run this procedure as a substitute for their sign-off. No provider access is needed for documentation review.

Use separate disposable database lanes for fresh migration, legacy upgrade and real-worker smoke. All must satisfy the existing backend test guard and contain only approved synthetic fixtures. The database name alone does not prove safe provenance. Do not use the preserved original mailbox database for any of these lanes.

Backend Vitest setup loads `.env.test` with override and tests can delete fixture rows. The smoke has real workers that can consume any eligible jobs in their database. Never run a DB-resetting suite concurrently with the smoke or against a database containing arbitrary jobs/users. Complete each lane and stop its processes before changing target configuration.

## 1. Prepare and identify the target

Work from `career-companion-backend-main` in this migrated workspace (source prefix `backend/`). Review the intended local, disposable target and fixture provenance without exposing its credentials. Prepare `.env.test` so DATABASE_URL and TEST_DATABASE_URL identify that exact target. Export TEST_DATABASE_URL in the shell from the reviewed local secret configuration before the procedure below; it is the independent expected value against which `.env.test` is checked. Do not print either URL in logs.

Review network blocking and ensure no normal application/worker process is attached to this database. If any target, ownership or exclusivity check is uncertain, stop before migration or fixture setup. Never treat a later Vitest safety check as authorization for an earlier Prisma command.

## 2. Generate and compile before DB work

From backend, use the installed locked dependencies:

```sh
npm run db:generate
npm run typecheck
npm run lint
npm run build
```

Generation must precede checks that depend on S6-01's new Prisma field. Build produces a current `dist/utils/testDatabase.js` without starting the server. Do not use npm start/dev here: normal startup may start workers.

From frontend, after the backend contract edits are ready, first verify the synchronization source. The current `scripts/sync-contracts.mjs` hardcodes sibling `career-companion-backend`, absent in this migrated workspace; it prints “Skipping sync” and exits 0. That is **not successful synchronization**. As part of S6-01 implementation, make the existing sync path resolve the selected backend (or use a verified equivalent backend-to-frontend contract copy), then check file contents/diff. Do not hand-maintain a divergent frontend schema. No script or contracts are changed during this documentation task.

Only after resolving that prerequisite:

```sh
npm run sync-contracts
npm run typecheck
npm run lint
npm run build
```

Inspect the synchronized contract diff. Do not hand-fork generated/copied frontend contracts or silently accept unrelated schema drift.

## 3. Guard before invoking the migration CLI

The following procedure must run from backend after successful fresh compilation. It loads the same overriding `.env.test` used by tests, compares it to the independently exported expected target, invokes the existing guard, and only then launches installed Prisma CLI commands with that validated environment. A failed guard cannot fall through to migration. Do not run the individual migration command first.

```sh
node <<'NODE'
const { config } = require('dotenv');
const { spawnSync } = require('node:child_process');
const expectedTarget = process.env.TEST_DATABASE_URL;
if (!expectedTarget) {
  throw new Error('Export the reviewed disposable test target before continuing');
}
const loaded = config({ path: '.env.test', override: true, quiet: true });
if (loaded.error) {
  throw new Error('Test configuration could not be loaded; no migration started');
}
const { assertTestDatabase } = require('./dist/utils/testDatabase');
try {
  assertTestDatabase(process.env.DATABASE_URL, process.env.TEST_DATABASE_URL);
  if (process.env.TEST_DATABASE_URL !== expectedTarget) throw new Error();
} catch {
  throw new Error('Test database preflight failed; no migration started');
}
process.env.NODE_ENV = 'test';
const prismaCli = require.resolve('prisma/build/index.js');
for (const args of [['migrate', 'deploy'], ['migrate', 'status'], ['validate']]) {
  const result = spawnSync(process.execPath, [prismaCli, ...args], {
    env: { ...process.env },
    stdio: 'inherit',
  });
  if (result.error || result.status !== 0) process.exit(result.status ?? 1);
}
NODE
```

The existing guard requires matching URLs, a local PostgreSQL host, a `career_companion_*test` database name, and no unsupported connection query parameters. It is necessary but not sufficient: the operator must also establish synthetic provenance and lane exclusivity. Do not change `.env.test` between this preflight and its lane's tests. Recheck it if anything changes. Do not record credential-bearing environment values or unsanitized CLI diagnostics in published evidence.

During implementation, verify rejection ordering without contacting a database: an absent expected target, an unsafe target, a mismatch between shell expectation and `.env.test`, or unequal URL variables must prevent CLI spawning. This is an acceptance requirement; this document does not claim those runtime cases have passed.

## 4. Fresh and upgrade preservation lanes

For a fresh lane, apply all migrations only through the guard and verify a newly created application's revision is zero and ownership constraints exist.

For an upgrade lane, first establish the approved pre-S6 schema in a separate disposable database using that revision's guarded procedure. Seed synthetic legacy records through implementation-time test fixtures: approximately 1,620 emails, multiple owners, existing AI/user statuses including a user status with unknown timestamp, events/actions and completed AI operations. Preserve a before snapshot of IDs and relevant fields. Do not run a generic production seed or copy real Gmail bodies into fixtures.

Switch to the candidate implementation and apply S6-01's additive migration through the guarded procedure. Verify zero revisions on legacy applications; statuses, existing timestamps, IDs, ownership and AI records unchanged; event/action uniqueness and triggers intact. Store sanitized before/after comparisons. Run no-op/change/clear tests separately from the migration snapshot so expected mutations are not mistaken for migration damage.

The original approximately 1,620-email dataset must remain untouched by these lanes. Link Sprint 5's measured original-data preservation and live incremental/repeat-sync records separately. Synthetic count similarity is not proof of original-data preservation.

## 5. Focused and full suites

After target preflight/migration, backend focused checks:

```sh
npm test -- src/tests/application.test.ts src/tests/matcher.test.ts src/tests/ai-idempotency.test.ts src/tests/stabilization-safety.test.ts src/tests/email-worker-reliability.test.ts
```

Frontend focused checks:

```sh
npm test -- src/tests/applications.test.tsx src/tests/stabilization-ui.test.tsx
```

Once focused checks pass on a stable implementation, run `npm test` in each repository for its full suite. Typecheck/lint/build commands in step 2 apply to the final diff; rerun affected checks if subsequent edits changed their inputs. Do not repeatedly run full suites without a new reason. No npm e2e script exists in the inspected baseline.

### S6-03 queue and matcher verification lane

Record installed pg-boss version, registration and effective batch size/concurrency. Current source uses batchSize=1, not a demonstrated batch-loss configuration. Use a separately guarded/exclusive real-queue lane for delivery and deterministic barriers in focused DB tests for matching races. Follow [S6-03 sections 6/14/15](S6-03.md) for the full matrix: multiple queued emails, mixed failure/retry, distinct-email same-application effects, duplicate replay, stale automatic selection versus user link/ignore, competing legal resolutions, and AI/manual-field interleavings. Do not run workers alongside row-resetting Vitest on the same database. Record per-job outcomes and effects; duplicate-only tests do not replace this matrix.

If single-job delivery is retained, verify its explicit invariant; no >1 batch performance goal is implied. Multi-job delivery requires mixed batch acknowledgment/failure evidence. Reproduce a suspected defect before calling it fixed. Preserve Sprint 5's final-attempt crash/operator limit unless separately scoped.

## 6. Extend and run the final Sprint 5 smoke

Start with the implemented Sprint 5 harness and follow-up regression fixes in [S5-04](../sprint-5/S5-04.md) and the [execution report](../sprint-5/execution-report.md). It already runs real local workers with deterministic provider adapters. The worker-disabled smoke belongs only to the 2026-09-26 historical audit.

Before running the extended script, export the reviewed `SMOKE_DATABASE_URL` for a fresh exclusive `career_companion_*_smoke_test` target and repeat the guard/provenance checks. The Sprint 5 harness reads it before loading backend `.env.test`, sets both runtime database URLs to it, rejects non-empty fixture state and takes an advisory lock. `BACKEND_DIR` can identify the active migrated backend; `CHROME_BIN` selects the installed browser when needed. Migrate this lane separately through steps 1–4 with the same reviewed target; the smoke guard does not authorize an earlier unguarded migration. Configure environment before backend imports. Keep NODE_ENV=test for explicit startup control, then register real Gmail/email workers. Use a valid fixture history anchor and encrypted fake OAuth token, deterministic provider responses at external boundaries, and a small nonzero AI budget through the real ledger. Verify the actual history.list call. Block unexpected backend/browser network traffic; allow only the verified local database and local application traffic needed by the harness. Real notification delivery stays disabled.

From frontend, after all Vitest suites using the target have finished:

```sh
node scripts/smoke-stabilization.mjs
```

Required combined scenarios:

| Scenario | Required evidence |
| --- | --- |
| Inherited Sprint 5 path | Sync returns 202; real queue/workers persist domain results; delayed processing refreshes mounted UI; repeat sync has no duplicates/additional completed-AI calls |
| Canonical reads | List/detail agree for user-over-AI and true unknown; an app older than page one is directly accessible |
| Manual set/change | Acknowledged response and refreshed list/detail agree; revision/timestamp obey S6-01 |
| S6-03 delivery | Record pg-boss version/options; prove every delivered job has an attempted outcome under the chosen size-one/all-jobs invariant, including retry/terminal failure |
| S6-03 matching | Distinct emails retain legitimate effects; duplicate work stays unique; stale automatic selection and competing legal resolutions preserve user decisions; no cross-owner writes |
| AI after correction | Real fixture email processing changes AI state, preserving all manual fields/revision |
| Competing editors | Second editor saves; first editor background-refetches; first save still sends its frozen revision and receives 409 |
| Clear | Explicit clear reveals current persisted AI or unknown, without invoking AI |
| Delayed reads | Detail/list GET started before save cannot repaint older manual state after acknowledgement |
| Uncertain save | Drop response after server commit; reconcile by GET; no automatic PATCH; failed GET retains draft and disables Save |
| Uncertain creation | Real POST commits before malformed/dropped `201`; retain draft and show unknown outcome; refresh owned list without blind second POST; failed reads stay blocked, and same-company/role or missing first-page results do not determine the outcome. Require deliberate review before any new creation under S6-01 section 6 |
| Invalid contract | Missing fields/invalid 2xx fail parsing; valid null stays valid; uncertain mutation result is reconciled |
| Evidence/sections | Recording time and email date differ honestly; escaped source/AI text; missing source and failed history/action requests recover locally |
| Navigation/pagination | In-flight save remains bound to original app; 20-item paging and >20 history coverage remain usable |
| Accessibility/layout | Keyboard/focus/pending/error behavior works on desktop and narrow viewport with existing themes |

D1 must resolve additional action-response/UI evidence before that scope is implemented or marked passed. Event/recentEvent evidence above is specified; independent action-section recovery remains required either way.

Do not write successful processing/domain output directly after triggering the positive path. Initial synthetic seeding is allowed; faking completion after the trigger does not verify workers.

## 7. Side-effect accounting and cleanup

For a manual PATCH, first drain earlier fixture activity, take relevant counters, perform the mutation and compare. Alternatively use proven operation correlation. Assert zero provider calls, enqueued jobs, AI-ledger changes, event inserts and action transitions for that manual interval. A whole real-worker run cannot have a blanket zero-AI budget/call requirement.

For replay, assert completed AI operations reuse results without a provider call and event/action rows stay unique. Existing matcher behavior can enqueue notification work on replay; do not invent a zero-queue promise. Keep real notification delivery off and account for fixture notification jobs.

The current Sprint 5 harness stops the queue before cleanup, but closes its HTTP server only after fixture deletion and Prisma disconnection. Do not describe the stronger ordering below as already implemented. S6-05 owns this bounded harness adjustment while preserving Sprint 5 scenarios, provider isolation and exclusive-lane guards.

Required teardown, including a failed run:

1. Stop new browser/API activity, HTTP request intake and worker fetch. Drain/abort active HTTP requests and worker handlers and prove no future writes can occur before starting destructive cleanup. Retain fixture adapters and outbound blocking until those handlers are quiescent; release controlled fixture barriers as needed to let them finish.
2. If drain/quiescence cannot be proven, skip destructive cleanup, retain sanitized diagnostics and quarantine the disposable lane. A timeout is not evidence that a handler stopped. Do not reuse or clean the lane until remaining processes are stopped and no future writes are possible.
3. With handlers quiescent and database access still available, retain fixture action IDs before owner cascade deletion; remove fixture owner jobs, notification jobs identified by those action IDs, fixture owners and exclusively owned fixture daily-budget/operation data safely.
4. Close remaining browser/server/queue/database resources and restore adapters after handlers are stopped; release lane ownership last. Do not restart workers for cleanup.
5. Record teardown ordering and residual records/jobs. Verify successful and injected-failure cases with an in-flight HTTP write and active worker; fixture deletion must never overlap either handler.

Preserve the existing safety invariants and extend the lifecycle to meet this contract; do not copy its current server-close-after-deletion order unchanged. Never use db:reset, broad production deletion, AI operation reset or live delivery as a cleanup shortcut.

### Final architecture and data-preservation review

S6-05 section 7 is the required checklist: existing service/queue/DB boundaries, canonical mapper and owned revision writes, bounded evidence/recording semantics, verified delivery and matching locks, runtime contracts/frozen drafts/recovery, and additive rollout/rollback. Record pass/fail with source/test evidence for each. A generic smoke pass does not close this review. Retain accepted notification, temporal, rematching, retention and operational limits; fixture success does not establish broader MVP readiness.

## 8. Execution record — pending

| Evidence | Current state |
| --- | --- |
| Engineering baseline and prior Sprint 5 evidence | Linked in assessment/execution report; uncommitted files are the baseline, not new S6 results |
| Entry evidence approval and D1/D2 decisions | Pending |
| Actual deployed API/worker versions and integrated baseline approval | Not verified by this documentation task; retained entry gate |
| Final implementation commit IDs / contract sync diff | Pending |
| Disposable target provenance and guard rejection tests | Pending |
| Fresh migration / additive upgrade field comparisons | Pending |
| Focused/full tests, typecheck, lint and builds | Pending |
| Real-worker browser scenarios and sanitized artifacts | Pending |
| S6-03 actual delivery and matching-concurrency matrix | Pending |
| S6-05 final architecture boundary review | Pending |
| Manual/replay provider and job accounting | Pending |
| Teardown and residual fixture/job check | Pending |
| Original-data preservation and live Gmail evidence from Sprint 5 | Unverified/unavailable here; retained gate |
| Rollout/rollback and remaining product limitations documented | Pending |

Record commands, start/end, selected revisions, pass/fail, sanitized diagnostics and artifact paths when executed. A document link/section/syntax check is not a runtime feature test. Saving this pack completes planning corrections only.
