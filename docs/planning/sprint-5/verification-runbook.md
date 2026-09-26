# Sprint 5 verification runbook

This is a future execution procedure, not a verification report. During planning: repositories, migrations, tests, installed transport/queue source and documentation were inspected; no database/provider access, migrations, test fixtures or sync calls were executed. Live fields below are intentionally unmeasured.

## Environment separation

| Lane | Purpose | Allowed state changes during later execution | Forbidden |
| --- | --- | --- | --- |
| Preserved live baseline | COM-37 read-only capture and COM-38 real incremental proof | Normal authenticated sync, controlled synthetic mail received in the owner's mailbox, explicitly designated synthetic application, normal processing of explained new/pending work | Reset, truncate, seed, delete/re-import, bulk state reset, fake provider responses, manually written checkpoints/completion, fault injection or fixture cleanup |
| Isolated local fixture DB | COM-39 unit/integration/queue failures and COM-40 browser smoke | Synthetic fixture setup/cleanup, process interruption, mocked provider/transport faults, real local queue execution | Real mailbox data/credentials, pointing at the original dataset, claiming real-provider proof |
| Isolated restored copy | Backup restore/schema upgrade investigation if deployment prerequisites require it | Explicitly scoped migration rehearsal in a later authorized rollout | Substituting restore/clean-install evidence for incremental proof on the original dataset |

The backend guard requires the exact same `DATABASE_URL` and `TEST_DATABASE_URL`, a local host and a `career_companion_*test` database name. Vitest and the browser harness load `.env.test` with **override enabled**. A matching name alone does not make a database disposable: verify its provenance and contents. Never rename/configure the preserved database to satisfy the guard.

Starting backend `npm run dev` or `npm start` outside test starts Gmail, email and notification workers. Readiness inspection must not accidentally process historical backlog. A backup restoration is only for isolated verification, never a routine rollback over the preserved dataset.

Execution order is COM-37 baseline → COM-39 recovery → COM-41 diagnostics → COM-40 final local worker/browser smoke → COM-38 live Gmail proof on the same build. For the worker-enabled smoke, the existing test DB URL guard is necessary but insufficient: use an exclusively owned smoke DB with no other suite running and no unrelated jobs, because registered workers consume the whole queue. Replace the current harness's plaintext placeholder token, missing history anchor and zero AI allowance with the fixture setup specified in COM-40. Stop handlers before owner/action/job/budget cleanup. None of these fixture operations apply to the preserved lane.

## Baseline capture and invariant comparison

First inspect applied migration identities and table/column presence read-only. The preserved database may still have only the eight baseline migrations; do not migrate it merely to make a snapshot query run. Capture original identities/immutable metadata before any later authorized rollout, then compare them again after rollout. Optional stabilization fields/tables that are absent are recorded as unavailable, not as zero backlog.

Capture a private consistent snapshot under a read-only repeatable-read transaction. Identify the owner explicitly, exclude OAuth/session secrets, and record UTC plus local timezone (`Asia/Kolkata`) timestamps. The core examples below use the eight-migration baseline shape; confirm those columns exist first. They are **parameterized SQL shapes for the later reader**, not an existing repository command; bind `$1` to the verified owner UUID in the chosen database client. Run each parameterized query through that client within the transaction.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY;

SELECT id, "userId", status, "syncStatus", "lastHistoryId", "lastSyncedAt"
FROM gmail_connections WHERE "userId" = $1::uuid;

SELECT count(*) AS email_count FROM emails WHERE "userId" = $1::uuid;

SELECT id, "userId", "gmailMessageId", "threadId", subject, sender,
       "receivedAt", "createdAt"
FROM emails WHERE "userId" = $1::uuid ORDER BY id;

SELECT "processingState", "relevanceState", "matchState", count(*)
FROM emails WHERE "userId" = $1::uuid
GROUP BY "processingState", "relevanceState", "matchState";

SELECT id, "applicationId", "matchState", "matchConfirmedBy"
FROM emails
WHERE "userId" = $1::uuid AND "matchConfirmedBy" = 'USER_CONFIRMED'
ORDER BY id;

SELECT id, "userStatus", "userStatusSetAt"
FROM applications WHERE "userId" = $1::uuid ORDER BY id;

SELECT e."userId", e."gmailMessageId", count(*)
FROM emails e WHERE e."userId" = $1::uuid
GROUP BY e."userId", e."gmailMessageId" HAVING count(*) > 1;

COMMIT;
```

When read-only schema inspection confirms stabilization is applied, include these additional queries **inside the same snapshot, before its COMMIT**. Omit them and record the missing schema gate on a legacy database; do not let an undefined-column/table error abort the original-data capture.

```sql
SELECT id, "syncLeaseUntil", "syncError"
FROM gmail_connections WHERE "userId" = $1::uuid;

SELECT o.id, o."emailId", o.operation, o.version, o.status, o.attempts,
       o."startedAt", o."completedAt", o."retryAfter", o."errorCode"
FROM ai_operations o JOIN emails e ON e.id = o."emailId"
WHERE e."userId" = $1::uuid ORDER BY o.id;
```

Capture result/checkpoint integrity privately as well: current completed `ai_processing_results` and, where present, completed operation `result` values keyed by row/email/version, plus existing email-derived action/event keys and delivery claims. Those structured fields may contain PII; export only to protected local evidence, never the ticket. Where present, query the current UTC-day AI budget and record the global nature of its count. Attribute calls using owner/email operation deltas rather than assuming the global counter belongs solely to this test user. Query queue metadata with the verified installed pg-boss schema, selecting state/attempt/timing and only the minimum payload IDs; do not dump arbitrary payload/output or session tables. Missing stabilization tables remain rollout blockers for live sync, even though the original-data snapshot can be captured safely.

Privately serialize rows in stable field/key/order form and compute a manifest digest using the evidence tooling selected during execution. Retain the full identity manifest: a digest or count alone is insufficient for identifying missing/changed rows later. No extension/migration is needed merely to hash an export.

Let `B0` be the original baseline identity set and `Bpre` the immediate pre-control-change set after any explicitly recorded preparatory reconciliation. Then verify:

- `B0` is a subset of `Bpre` and `Bfinal`, with the same local IDs, owner, Gmail ID and immutable metadata (`threadId`, subject, sender, receivedAt, createdAt). No replacement IDs are acceptable.
- No duplicate `(userId,gmailMessageId)` rows exist. For the controlled Gmail ID, pre-change count is zero and each post-sync count is exactly one with the same local ID.
- `Nfinal = N0 + all explained new identities`. Other real arrivals/fallback additions are enumerated; equality to 1,621 is not assumed.
- Explicit user matches/ignores and application user-status choices survive. Previously completed AI results/operation results remain unchanged; their attempts do not grow and no new operation-version replay is created for those emails.
- Pending work present in the baseline may legitimately progress only under the recorded execution policy; record state/effect deltas separately. Processing/updatedAt values are not immutable metadata. Any unexplained change blocks acceptance.
- For a control email that completes domain processing, compare action/event identities after processing settles. Work accepted by the first run may legitimately finish during the repeat; correlate its job/operation before judging deltas. Each email-derived effect remains unique. If live AI is intentionally deferred/held, record that state and verify existing completed-work invariants rather than demanding a new live AI result.

Preserve rows when a message is removed from Gmail or ages out of the 90-day query; this sprint is additive ingestion, not a mailbox deletion mirror.

## Read-only deployment readiness

Record intended/deployed backend and frontend commits, running API/worker process identities, all ten migration IDs/applied status, presence of the stabilized constraints/triggers, private backup location and restore evidence. Use the preflight SELECTs in `backend/STABILIZATION.md` to identify legacy duplicate/cross-owner blocks without fixing data here. Schema files and `/health` do not establish deployment readiness.

Record non-secret environment decisions: authentication/session route, actual proxy/cookie/HTTPS behavior, same-mailbox grant, retained encryption key, provider configurations, and small AI daily call allowance including pending backlog. `AI_DAILY_CALL_LIMIT=0` blocks new paid calls but does not pause workers or stop Gmail reads; it can leave retryable/failed jobs. Do not use it as a substitute for backlog planning. Completed AI work must remain zero-new-call regardless of budget.

Live Gmail proof is mandatory; successful live Gemini extraction and live Discord delivery are supplementary. Record the chosen live AI policy before running, including consequences for existing pending work. Do not replace the real Gmail adapter or modify production worker startup to manufacture a cheaper test. COM-40 separately proves the full AI/domain/browser path with fixture adapters. Observed processing failures must remain visible and explained; this distinction does not waive an actual application regression.

## API contract reference for the verification path

| Endpoint | Request / response and errors |
| --- | --- |
| `POST /api/gmail/sync` | No body or user ID. 202 `{ accepted: true }`; 401 unauthenticated, 400 `GMAIL_NOT_CONNECTED`, 409 `SYNC_IN_PROGRESS`; queue failures currently generic 500. Acceptance is not completion. |
| `GET /api/gmail/status` | 200 `{ connected, gmailEmail, status, syncStatus, lastSyncedAt, syncError? }`. No tokens/history/page cursor/counts. Expired SYNCING lease is projected as FAILED; read does not renew or repair it. |
| `GET /api/gmail/messages` | Owner-scoped messages with ID/provider ID/metadata/relevance/match and optional processingState; paginated envelope. 401 session, 400 malformed numeric parameters. |
| `GET /api/applications` and `GET /api/applications/:id` | Paginated owner list; independent detail object, 404 missing/unowned, 400 malformed UUID. Existing POST creates an application with 201 and validated company/job fields. |
| `GET /api/applications/:id/events` and `/actions` | Paginated sublists; unavailable/unowned application returns 403; malformed ID/pagination 400. |
| `GET /api/emails/unmatched`, `/ambiguous`, `GET /api/actions` | Other three paginated lists; action status filter validated. No new filter/search semantics. |
| `PATCH /api/actions/:id` | Existing status mutation, e.g. `{ status: "COMPLETED" }`; verify owned 200 result and persistence. |

All seven list surfaces use `{ items, metadata: { limit, offset, nextOffset } }`, default and maximum 20. Fetch `nextOffset` until **null**, not until a guessed total. Stable tie ordering does not prevent offsets shifting under new inserts. Use consistent DB snapshots for preservation proof. API routes always derive ownership from authenticated context.

## Commands verified from repository scripts/configuration

These are commands to execute later under the stated prerequisites, not results from this planning pass. Repository aliases are defined in [architecture review](architecture-review.md).

| Working directory | Existing command | Meaning / precondition |
| --- | --- | --- |
| Each repo | `git status --short --branch`; `git rev-parse HEAD` | Read-only checkout evidence. |
| backend | `npm run typecheck` | `tsc --noEmit`. |
| backend | `npm run lint` | `eslint src/`; record warnings separately. |
| backend | `npm test` | `vitest run`; **mutates fixtures**, guarded isolated DB only. |
| backend | `npm test -- src/tests/gmail-ingestion.test.ts src/tests/gmailSync.test.ts src/tests/gmail.test.ts src/tests/queue.test.ts` | Focused current Vitest files, same DB guard. |
| backend | `npm test -- src/tests/ai-idempotency.test.ts src/tests/stabilization-safety.test.ts src/tests/pagination.test.ts` | Paid-effect, ownership and pagination regressions. |
| backend | `npm run build` | `tsc`, produces `dist` needed by browser harness. |
| backend | `npx prisma validate`; `npx prisma migrate status` | Existing release-guide commands; schema/status inspection, not deployment. Select the intended environment explicitly. |
| frontend | `npm run typecheck`; `npm run lint`; `npm test`; `npm run build` | `tsc -b`, Oxlint, Vitest and Vite/TypeScript build; typecheck/build write local build artifacts. |
| frontend | `npm test -- src/tests/gmail.test.tsx src/tests/stabilization-ui.test.tsx src/tests/pagination-ux.test.tsx` | Focused component/query behavior. |
| frontend | `node scripts/smoke-stabilization.mjs` | Existing Puppeteer harness; both repos built, sibling layout, Chrome/`CHROME_BIN`, isolated backend `.env.test`. At inspected HEAD it fakes sync completion; only the COM-40 version proves real local worker behavior. |
| frontend | `npm run sync-contracts` | Existing copy script from sibling backend; **writes source contracts**. Use only during later implementation if a contract changes, then review diff. It is not a read-only verification command. |

No existing `npm run e2e`, `test:integration`, `sync:verify` or Playwright command was found. New tests can be run through the existing Vitest discovery without inventing scripts. No migrations are required by the default plan. Existing `db:reset`, `db:seed`, `db:migrate` and `format` scripts are not verification shortcuts and must not run on the preserved baseline/in this planning task. If test schema initialization is needed later, use the documented guarded isolated-database setup with explicit target review; it is never a requirement to reset live data.

## Live evidence template

Create a **new sanitized report during execution**, keeping protected raw artifacts outside the repo. Do not fill unknown values with illustrative successes.

| Field | Required evidence / initial state |
| --- | --- |
| Date/operator/environment | Pending; timezone and protected environment identifier |
| Runtime commits | Pending; actual backend/frontend, worker versions and log locations |
| Original baseline | Pending; measured N0, manifest identity/metadata digest, private file reference, backup/restore proof |
| Backlog/AI/notification policy | Pending; state distributions, original completed records, operation attempts, global budget, disabled Discord or separate authorization |
| Pre-change anchor | Pending; private H0 and lastSyncedAt, scope, Bpre; disclose any preparatory fallback |
| Control change | Pending; provider change time, private Gmail message ID, synthetic marker, INBOX/age verification |
| Request/attempt evidence | Pending; authenticated POST status/time, request/job/attempt IDs, worker registration |
| Ingestion result | Pending; history mode, all pages, start/end/duration, dispositions, queue offers, retry/failure/anomaly record |
| Checkpoint result | Pending; protected H1, affected-row success and checkpointAdvanced; IDs compared as arbitrary-precision decimal strings where needed |
| Control persistence | Pending; exact one-row query, matching metadata and stable local ID |
| Processing/browser | Pending; actual visible processing state, completed/deferred/held/failed explanation, optional live AI result and timeline/action if applicable, COM-40 local result reference |
| Repeat sync | Pending; correlated second request, same email ID, no new completed-operation AI calls or duplicated domain effects |
| Original preservation | Pending; zero missing/changed original identities/immutable fields, user choices and completed AI invariant comparisons |
| Unrelated changes | Pending; list/manifest of new real arrivals and permitted backlog transitions |
| Live token refresh | Pending; observed normal expiry/refresh evidence or explicitly unverified, separate from fixture refresh tests |
| Verdict | Pending; criterion-by-criterion pass/fail/blocked with artifact references, never “passed” solely from test count |

## Failure and stop conditions

Stop live acceptance on an unknown database identity, absent backup/manifest, incompatible schema, mixed pre-stabilization workers, unexplained baseline loss, identity switch, uncontrolled AI cost, or missing worker/provider evidence. Keep rows/checkpoints/claims intact and diagnose the first boundary. A missing credential or failed live case is not resolved by resetting the DB, manually completing a job, or running only mocks. Conditional infrastructure/rollout work must be handled explicitly before resuming this runbook.
