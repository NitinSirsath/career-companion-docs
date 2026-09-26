# Sprint 6 verification runbook

Status: proposed execution procedure, not an execution report. No database changes, implementation tests, worker runs or live-provider checks were performed while saving this pack. The Node procedure below is a documented composition of existing tools, not a new npm script.

## Entry evidence and execution lanes

Complete the [Sprint 5 entry gate](README.md#b-why-this-sprint-follows-sprint-5). Record final backend/frontend/docs commit IDs and reconcile contracts, refresh behavior and fixture safety against those revisions. Sprint 5 is currently a draft, not completed evidence.

Use separate disposable database lanes for fresh migration, legacy upgrade and real-worker smoke. All must satisfy the existing backend test guard and contain only approved synthetic fixtures. The database name alone does not prove safe provenance. Do not use the preserved original mailbox database for any of these lanes.

Backend Vitest setup loads `.env.test` with override and tests can delete fixture rows. The smoke has real workers that can consume any eligible jobs in their database. Never run a DB-resetting suite concurrently with the smoke or against a database containing arbitrary jobs/users. Complete each lane and stop its processes before changing target configuration.

## 1. Prepare and identify the target

Work from `career-companion-backend`. Review the intended local, disposable target and fixture provenance without exposing its credentials. Prepare `.env.test` so DATABASE_URL and TEST_DATABASE_URL identify that exact target. Export TEST_DATABASE_URL in the shell from the reviewed local secret configuration before the procedure below; it is the independent expected value against which `.env.test` is checked. Do not print either URL in logs.

Review network blocking and ensure no normal application/worker process is attached to this database. If any target, ownership or exclusivity check is uncertain, stop before migration or fixture setup. Never treat a later Vitest safety check as authorization for an earlier Prisma command.

## 2. Generate and compile before DB work

From backend, use the installed locked dependencies:

```sh
npm run db:generate
npm run typecheck
npm run lint
npm run build
```

Generation must precede checks that depend on S6-02's new Prisma field. Build produces a current `dist/utils/testDatabase.js` without starting the server. Do not use npm start/dev here: normal startup may start workers.

From frontend, after the backend contract edits are ready:

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

Switch to the candidate implementation and apply S6-02's additive migration through the guarded procedure. Verify zero revisions on legacy applications; statuses, existing timestamps, IDs, ownership and AI records unchanged; event/action uniqueness and triggers intact. Store sanitized before/after comparisons. Run no-op/change/clear tests separately from the migration snapshot so expected mutations are not mistaken for migration damage.

The original approximately 1,620-email dataset must remain untouched by these lanes. Link Sprint 5's measured original-data preservation and live incremental/repeat-sync records separately. Synthetic count similarity is not proof of original-data preservation.

## 5. Focused and full suites

After target preflight/migration, backend focused checks:

```sh
npm test -- src/tests/application.test.ts src/tests/matcher.test.ts src/tests/ai-idempotency.test.ts src/tests/stabilization-safety.test.ts
```

Frontend focused checks:

```sh
npm test -- src/tests/applications.test.tsx src/tests/stabilization-ui.test.tsx
```

Once focused checks pass on a stable implementation, run `npm test` in each repository for its full suite. Typecheck/lint/build commands in step 2 apply to the final diff; rerun affected checks if subsequent edits changed their inputs. Do not repeatedly run full suites without a new reason. No npm e2e script exists in the inspected baseline.

## 6. Extend and run the final Sprint 5 smoke

Start with the completed Sprint 5 harness requirements in [COM-40](../sprint-5/COM-40.md) and its [planning review](../sprint-5/planning-review.md). The currently inspected script's worker-disabled/manual-completion behavior is not sufficient and must not be retained as the acceptance path.

Before running the extended script, select its exclusive disposable target and repeat the guard/provenance checks. Configure environment before backend imports. Keep NODE_ENV=test for explicit startup control, then register real Gmail/email workers. Use a valid fixture history anchor and encrypted fake OAuth token, deterministic provider responses at external boundaries, and a small nonzero AI budget through the real ledger. Verify the actual history.list call. Block unexpected backend/browser network traffic; allow only the verified local database and local application traffic needed by the harness. Real notification delivery stays disabled.

From frontend, after all Vitest suites using the target have finished:

```sh
node scripts/smoke-stabilization.mjs
```

Required combined scenarios:

| Scenario | Required evidence |
| --- | --- |
| Inherited Sprint 5 path | Sync returns 202; real queue/workers persist domain results; delayed processing refreshes mounted UI; repeat sync has no duplicates/additional completed-AI calls |
| Canonical reads | List/detail agree for user-over-AI and true unknown; an app older than page one is directly accessible |
| Manual set/change | Acknowledged response and refreshed list/detail agree; revision/timestamp obey S6-02 |
| AI after correction | Real fixture email processing changes AI state, preserving all manual fields/revision |
| Competing editors | Second editor saves; first editor background-refetches; first save still sends its frozen revision and receives 409 |
| Clear | Explicit clear reveals current persisted AI or unknown, without invoking AI |
| Delayed reads | Detail/list GET started before save cannot repaint older manual state after acknowledgement |
| Uncertain save | Drop response after server commit; reconcile by GET; no automatic PATCH; failed GET retains draft and disables Save |
| Invalid contract | Missing fields/invalid 2xx fail parsing; valid null stays valid; uncertain mutation result is reconciled |
| Evidence/sections | Recording time and email date differ honestly; escaped source/AI text; missing source and failed history/action requests recover locally |
| Navigation/pagination | In-flight save remains bound to original app; 20-item paging and >20 history coverage remain usable |
| Accessibility/layout | Keyboard/focus/pending/error behavior works on desktop and narrow viewport with existing themes |

Do not write successful processing/domain output directly after triggering the positive path. Initial synthetic seeding is allowed; faking completion after the trigger does not verify workers.

## 7. Side-effect accounting and cleanup

For a manual PATCH, first drain earlier fixture activity, take relevant counters, perform the mutation and compare. Alternatively use proven operation correlation. Assert zero provider calls, enqueued jobs, AI-ledger changes, event inserts and action transitions for that manual interval. A whole real-worker run cannot have a blanket zero-AI budget/call requirement.

For replay, assert completed AI operations reuse results without a provider call and event/action rows stay unique. Existing matcher behavior can enqueue notification work on replay; do not invent a zero-queue promise. Keep real notification delivery off and account for fixture notification jobs.

Teardown, including a failed run, must:

1. Stop new request intake and worker fetch; drain/abort active handlers and prove no future writes can occur.
2. Retain fixture action IDs before user cascade deletion so notification jobs carrying actionId can be located.
3. Remove fixture owner jobs and notification jobs identified by those action IDs; remove exclusively owned fixture daily-budget/operation data safely.
4. Close queue, database, server and browser resources without restarting workers for cleanup.
5. Report remaining fixture records/jobs and any failed drain. Quarantine the disposable lane if shutdown cannot be proven; do not race deletion against active workers.

Preserve Sprint 5's exact final lifecycle implementation when extending this checklist. Never use db:reset, broad production deletion, AI operation reset or live delivery as a cleanup shortcut.

## 8. Execution record — pending

| Evidence | Current state |
| --- | --- |
| Final approved baseline and Sprint 5 closure links | Pending |
| Final implementation commit IDs / contract sync diff | Pending |
| Disposable target provenance and guard rejection tests | Pending |
| Fresh migration / additive upgrade field comparisons | Pending |
| Focused/full tests, typecheck, lint and builds | Pending |
| Real-worker browser scenarios and sanitized artifacts | Pending |
| Manual/replay provider and job accounting | Pending |
| Teardown and residual fixture/job check | Pending |
| Original-data preservation and live Gmail evidence from Sprint 5 | Pending |
| Rollout/rollback and remaining product limitations documented | Pending |

Record commands, start/end, selected revisions, pass/fail, sanitized diagnostics and artifact paths when executed. A document link/section/syntax check is not a runtime feature test. Saving this pack completes planning corrections only.
