# Gmail incremental sync operations

Use the existing application logs, PostgreSQL queue/domain state and AI ledger. Ingestion completion and AI/application completion are different milestones. Select the intended environment explicitly; never run fixture commands or blanket reset/deletion/replay against a preserved dataset.

## Implemented evidence boundary — 2026-10-03

Delivered: `gmail_sync_started` and exactly one terminal completed/failed/superseded event per handled delivery, with request/attempt/job IDs, trigger, retry metadata, duration and checkpoint certainty; completions include window/gap and ingestion counts. All fixed failure categories below are implemented. `/ready` reports in-memory registration only, not database/provider health. The `gmail_sync_queued`, `gmail_sync_job_outcome`, queue-wait/mode fields, detailed counter equations and snapshot commands below remain target documentation, not shipped tooling. See [Sprint 7](../sprint-7/execution-report.md) and [Sprint 8](../sprint-8/execution-report.md) evidence.

## Target correlation and counters

`gmail_sync_queued` links user/request/job. `gmail_sync_started` adds the unique attempt, retry count/limit, queue wait and mode. The attempt ends with `gmail_sync_completed`, `gmail_sync_failed`, or `gmail_sync_superseded`; missing terminal logs plus an expired active job/lease indicate interruption. `gmail_sync_job_outcome` explains retry scheduling, exhaustion or terminal acknowledgment. An acknowledged obsolete/auth job is **not** successful ingestion. `worker_registered` proves registration in that process, not ongoing health.

- `references = uniqueCandidates + duplicateReferences`.
- `uniqueCandidates = existing + persistedNew + notFound + filteredLabel + filteredAge + invalidMetadata + unresolved + persistenceUnknown`.
- `queueOffers = queueAccepted + queueSuppressed + queueUnknown` for a settled attempt; `pendingRecoveryOffers` is a subset, not an additional total.
- `persistedNew` counts confirmed committed inserts even if enqueue later fails. Unknown commits remain `persistenceUnknown` until reconciled. All page references are registered before processing, so unvisited IDs remain unresolved after a partial failure.
- A candidate can change filtered disposition if a later page observes changed eligibility; confirmed existing/new attribution remains stable.
- Suppression means no new queue job was created in the singleton window. It does not imply AI completion. Pending recovery offers at most 100 rows per successful sync and never resets completed/failed/processing emails.
- `checkpointCommitted: true` is emitted only after confirmed final commit. `false` means no successful final write; `null` on failure means the final write may have committed but its acknowledgment is uncertain. `checkpointAdvanced: false` also occurs on legitimate no-change repeats. History IDs/cursors are never logged.
- `job_started`, `job_completed`, `job_failed` identify email processing separately, with retry/exhaustion, age, duration and allowlisted category/stage. A failure at the final permitted delivery becomes FAILED; completed email/AI results remain protected.

## Read-only diagnostics

Run against the explicitly selected database, using a read-only transaction. No secret columns or job payloads are selected. Bind `$1` to a verified owner UUID in a parameterized client for owner queries; do not paste private output into issue comments.

```sql
BEGIN READ ONLY;
SELECT name, state, count(*) AS jobs,
       min(created_on) AS oldest_created, min(started_on) AS oldest_started,
       max(retry_count) AS highest_retry
FROM pgboss.job
WHERE name IN ('gmail-sync-job', 'email-processing-job', 'discord-notification-job')
GROUP BY name, state ORDER BY name, state;

SELECT status, "syncStatus", count(*) AS connections,
       count(*) FILTER (WHERE "syncStatus" = 'SYNCING'
         AND ("syncLeaseUntil" IS NULL OR "syncLeaseUntil" <= clock_timestamp())) AS expired
FROM gmail_connections GROUP BY status, "syncStatus";

SELECT "processingState", count(*) AS emails, min("createdAt") AS oldest
FROM emails WHERE "userId" = $1::uuid GROUP BY "processingState";

SELECT operation, status, count(*) AS operations, max(attempts) AS max_attempts,
       min("retryAfter") AS earliest_retry
FROM ai_operations o JOIN emails e ON e.id = o."emailId"
WHERE e."userId" = $1::uuid GROUP BY operation, status;

SELECT day, calls, "cooldownUntil" FROM ai_call_budgets ORDER BY day DESC LIMIT 7;
COMMIT;
```

Inspect schema first on an older deployment: queue/AI/lease tables may not exist. Absence is a deployment gate, not zero backlog. AI budgets are global daily counters; owner-specific call counts come from owned operation attempts.

## Failure handling

| Category/state | Interpretation and next step |
| --- | --- |
| AUTH_REVOKED / REVOKED connection | Reconnect the same mailbox. Preserve emails and operation history. |
| RATE_LIMIT / PROVIDER_UNAVAILABLE / NETWORK_ERROR / REQUEST_TIMEOUT | Inspect bounded job retries/backoff and provider health. Do not clear AI claims to work around quotas. |
| FORBIDDEN | Inspect OAuth scope/provider policy; this is not automatically token revocation. |
| REQUEST_BUSY | Another attempt for the same request still owns a valid lease; delivery retries. |
| SUPERSEDED | A newer request or disconnect replaced ownership. Verify the new request; do not interpret as completed ingestion. |
| DEADLINE_EXCEEDED / CANCELLED | Check retained checkpoint, partial counts and next retry. Completed prefix records are reused. |
| QUEUE_UNAVAILABLE / queueUnknown | Queue acknowledgment may be uncertain. Existing identities/singletons guard replay; pending recovery can safely offer retained rows. |
| PROCESSING/UNKNOWN AI operation | Provider may already have charged/completed. Reconcile before authorizing a new call; manual retry refuses to erase this evidence. |
| FAILED email with safe resumable ledger | Resolve underlying outage/budget first, then use authenticated manual retry. Cooldown/exhausted/terminal claims remain held. |
| Queue retries exhausted / stale PROCESSING after process death | Inspect actual job and operation state. Never use a blanket SQL state reset. A stale paid-operation claim is intentionally not reclaimed automatically. |

The four-minute ingestion budget bounds new work and propagates cancellation. Gmail data calls have a 15-second bound, OAuth token/cert calls 10 seconds, revoke 5 seconds; SDK retries are disabled. Stored expiry enables proactive refresh; known-expiry 401 revokes without refresh and quota 403 does neither. Legacy null-expiry rows retain one bounded reactive 401/403 refresh/replay until an expiry is saved. Workers propagate pg-boss cancellation; the original signal is checked before SDK-internal refresh/replay. Short database cleanup transactions may finish after cancellation; report this as bounded cleanup, not an exact wall-clock 240-second process guarantee. Row locks and attempt checks prevent late publication. Queue sends accepted before cancellation can still run; durable AI/domain idempotency guards their effects.

## Verification commands

Backend: `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`, `node --test scripts/baseline-snapshot.test.mjs`. Vitest requires ignored `.env.test` matching the strict local test database guard. Snapshot commands appear in `node scripts/baseline-snapshot.mjs --help`; artifacts are private and never overwrite existing paths.

Crash proof: after build, run `node scripts/test-gmail-crash.cjs` with equal `DATABASE_URL`/`TEST_DATABASE_URL` targeting an empty exclusive `career_companion_*_crash_test` database with current migrations. It deliberately kills only its child worker and ages only its fixture job/lease. No application user or queue/budget state may pre-exist.

Frontend: `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`, `npm run test:smoke`. Follow the frontend README for exclusive smoke URL, backend path and Chrome options. The smoke uses deterministic test adapters and real workers, not live Gmail/AI.

For real mailbox proof use [verification-runbook.md](verification-runbook.md) with [execution-report.md](execution-report.md)'s corrected lookback/environment facts. Missing live access is an evidence limitation, never permission to reuse a fixture or reset historical work.
