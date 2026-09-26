# COM-41 — Correlate Gmail sync outcomes, counts, retries, and worker health

Status: proposed local ticket; no instrumentation implemented. Primary repository: backend. Suggested priority: High. Estimate: 5 points.

## 1. Title

Correlate Gmail sync outcomes, counts, retries, and worker health using existing structured logs.

## 2. Goal

Let an operator explain when a sync ran, what it discovered/persisted/skipped, whether history advanced, why it failed or retried, and whether downstream processing is stalled. Supply the minimum credible evidence for COM-38 without a new observability platform or job-history product.

## 3. Context

The backend already emits JSON logs and stores connection/job/AI state. Existing sync completion logs include userId, two counters and duration; failed worker logs include jobId and an error class. AI and email-worker events exist. The missing piece is accurate, correlated lifecycle evidence, including partial failure and provider/queue distinctions. COM-39 establishes the outcome/attempt boundary this ticket observes.

## 4. Current implementation

`backend/src/services/gmailSync.ts:84–85,221–230` defines current counters/log. `src/jobs/gmailSyncJob.ts:62` logs failure without counts/duration/user/request. `src/services/gmailClient.ts` sanitizes most errors to `Error` plus status. `src/jobs/emailProcessingJob.ts` logs job start/completion/failure; `src/services/ai/operations.ts` logs started/completed/reused/blocked/deferred operations; `src/services/ai/gemini/GeminiProvider.ts` logs token usage without email/operation correlation. `src/services/queue.ts` and `src/index.ts` log startup errors. `GET /health` is only liveness. pg-boss 12.31.0 provides retry/state metadata through its installed types; the current worker does not request it.

## 5. Problem / gap

No complete sync start/finish pair, mode, checkpoint outcome or retry identity exists. Existing “skipped” counts only already-persisted rows; 404/age/label exclusions are silent. “Ingested” increments after enqueue, hiding a committed row if send fails. Upsert return alone cannot prove this attempt inserted a row in a race. Failed runs lose all partial counts. Generic errors obscure auth vs quota/transport/queue issues. A healthy HTTP endpoint does not establish worker registration, and queue-completed can mean a terminal error was acknowledged.

## 6. Proposed implementation

Use a small local logging helper or explicit structured event builder in the affected modules; no dependency/platform is required.

1. Correlate a logical sync request and every attempt using COM-39's request/attempt identity, queue job ID and retry count. Emit queued/started/finished events with UTC timestamp, owner ID, mode (`incremental`, `initial`, `history_expired_reconciliation`), attempt, and safe outcome. Include queue wait and execution duration separately. Add a safe provider stage/status/reason on failures; do not serialize arbitrary errors.
2. Instrument counts at the actual boundaries below. Keep a per-attempt map keyed by Gmail ID with one disposition per unique candidate across pages and any fallback transition. Every repeated provider reference increments occurrences/duplicateReferences but must not count the same message again as both persistedNew and existing. Preserve persistedNew if a later read finds that row. If a previously excluded candidate later becomes eligible and is inserted, replace its prior exclusion disposition with persistedNew; do not let counting suppress valid ingestion. Count repeated queue offers separately. Count persisted rows at confirmed insert, independent from job send. Reuse COM-39's fenced persistence boundary and the smallest conflict-safe insert/result approach; do not infer “created” merely because a prior read saw no row. On failure, distinguish candidates never processed from candidates whose database commit outcome is unknown.
3. Emit `checkpointCommitted` only when COM-39 finalization affects one row. Record `checkpointAdvanced` as boolean, not “+1”; keep raw history values in protected verification evidence, not general logs. A no-change completed run can commit without advancing. Superseded/disconnected attempts get a non-success terminal event.
4. Expose queue offer outcomes to logging: accepted with job ID, suppressed by singleton, or failed. A suppressed offer is not another persisted email, successful AI completion, or necessarily proof of future worker execution. Distinguish newly discovered-email offers from pending-recovery offers, including the current 100-row batch limit/backlog age.
5. Use pg-boss worker metadata for retry count/limit/attempt timing. Provide a read-only diagnostic procedure over current queue state plus Gmail lease/state and AI operations to identify queued-without-worker, retry waiting, exhausted/failed, stale active/expired lease, and held/unknown AI work. A process kill cannot emit a terminal application log: explain detection by missing finish plus expired job/lease state rather than fabricating an end event.
6. Reuse existing AI logs and durable claims. Correlate usage with email/operation/version where practical at the existing call boundary; retain explicit reused/deferred/held outcomes. Show how operators distinguish an ingestion success followed by AI failure and why a terminal AI hold can coexist with a queue-completed job. Do not log extraction content or introduce a generic tracing framework.
7. Add minimal worker registration/stop/failure records for Gmail/email workers and document current `/health` semantics. Supply reproducible log filters/read-only queue queries in backend `STABILIZATION.md`; no dashboard or new public endpoint.

### Counter contract

| Proposed field | Exact meaning |
| --- | --- |
| `candidateOccurrences` | Every provider candidate reference selected from typed messageAdded / INBOX labelAdded or list results, before deduplication. Not all mailbox history records. |
| `discoveredUnique` | Distinct candidate Gmail IDs encountered across this attempt's fetched pages, before scope/existence decisions. |
| `duplicateReferences` | `candidateOccurrences - discoveredUnique`; repeated provider references, not duplicate DB rows. |
| `persistedNew` | Rows actually inserted by this attempt; increment even if their subsequent queue offer fails. |
| `existing` | Unique candidates already stored, including conflict-lost inserts; no claim that their AI processing completed. |
| `skippedNotInbox`, `skippedTooOld`, `skippedMissing` | Mutually exclusive exclusions for previously absent candidates; 404 on message get maps to missing, not expired history. Apply a documented precedence: missing → not-INBOX → too-old. |
| `unprocessed` | Discovered unique candidates not yet persisted/reused/excluded due to failure/interruption; includes a failed current candidate only when no uncertain DB commit is outstanding. |
| `persistenceUnknown` | Unique candidates whose insert/commit was attempted but its outcome cannot be established; neither confirmed new nor merely unprocessed. Reconcile before asserting a complete persisted count. |
| `enqueueAccepted`, `enqueueSuppressed`, `enqueueFailed` | Email-job offer outcomes counted separately from message dispositions; split pending-recovery offers with a source field. |
| `pagesFetched`, `providerRequests`, `providerRetries` | Pages and actual instrumented transport work, separate from pg-boss retries. If transport-level counting is unavailable, name the field logical requests and state the limitation rather than inventing counts. |

For every observed terminal attempt: `candidateOccurrences = discoveredUnique + duplicateReferences`, and `discoveredUnique = persistedNew + existing + skippedNotInbox + skippedTooOld + skippedMissing + unprocessed + persistenceUnknown`. On complete ingestion both `unprocessed = 0` and `persistenceUnknown = 0`. A candidate moves between dispositions; it is never in two categories simultaneously. Example: ID A occurs on two pages and is inserted once: occurrences=2, discoveredUnique=1, duplicateReferences=1, persistedNew=1, existing=0. Pending-recovery rows absent from the provider candidate set affect enqueue/backlog counters only. In a killed process the final totals may be unavailable; queue/lease evidence must mark that explicitly. Unknown commit/attribution remains `persistenceUnknown` unless evidence resolves it; finding a row later alone does not prove which attempt inserted it.

## 7. Architecture impact

Instrumentation spans request/job/service/Google/AI boundaries already present. COM-39 owns lifecycle correctness; this ticket observes it and makes counting truthful. No OpenTelemetry deployment, external collector, metrics store, distributed trace system or public job-management surface.

## 8. Data impact

No schema/index/migration. Use existing `gmail_connections`, `emails.processingState`, `ai_operations`, `ai_call_budgets`, queue metadata and relevant result/domain rows. Do not add a SyncRun table solely to store logs. Run/attempt identifiers live in logs and existing claim data. Evidence retention uses the existing protected log/artifact location with a documented duration appropriate for sprint verification; it does not establish a new email retention policy.

## 9. API impact

No public response change required. Preserve POST 202, status shape and 20-item pagination. Continue safe `syncError` values; raw provider reason/body stays out of responses. Progress is observational counts from processed work, not a percent with an unknown denominator. `/health` remains liveness and is documented as such. If future UI counts are desired, that is separate scope rather than silently expanding this ticket.

## 10. Background-job impact

Request metadata from existing Gmail/email jobs without changing correctness or increasing retries. Correlate initial execution and up to three configured queue retries; record retry exhaustion with DB/queue state even when a process did not log it. Do not label a configured dead-letter queue as existing: none is set. Preserve payload compatibility, singleton semantics and COM-39 fencing. No notification replay or additional notification delivery is needed.

## 11. Security/privacy considerations

Allowlist event fields and low-cardinality categories. Internal user/email/job IDs are still sensitive operational identifiers: restrict logs and redact public evidence. Exclude mailbox addresses, Gmail content/headers, raw IDs where unnecessary, tokens, OAuth codes/state, cookies, prompts, validated extraction payloads and raw exception objects. Tests include sentinel secrets in provider errors to prove redaction. Never print database connection strings in diagnostic instructions.

## 12. Testing strategy

Unit tests validate counter arithmetic, category allowlists and redaction. DB/provider-fixture integration tests capture logs for success, existing rows, cross-page repeated references after insertion, all exclusion types, DB insert followed by enqueue failure, uncertain commit, later-page failure, retry and lost finalization. Assert the per-candidate equations on each fixture, including the two-page example above. Real pg-boss tests prove metadata/correlation and distinguish business outcome from queue state. Crash tests verify the runbook detects missing terminal logs through expired jobs/leases. Existing AI tests prove held/reused/deferred events. COM-40 checks browser-to-job evidence; COM-38 provides actual Gmail counts/timings. Never use logging assertions as a substitute for row/checkpoint invariants.

## 13. Acceptance criteria

- [ ] Every accepted request can be linked to queued/start/terminal attempt events or an explicitly detectable expired/crashed attempt.
- [ ] Mode, queue wait, execution duration, retry number, job/request/attempt identity and checkpoint outcome are present and unambiguous.
- [ ] Message disposition counters reconcile on complete and partial-failure fixtures; committed insert-before-send-failure is not lost from persisted counts.
- [ ] Duplicate provider references, existing DB rows and suppressed job sends are distinct; none is mislabeled a duplicate email insertion.
- [ ] Auth, quota, policy/permission, network timeout, provider unavailability, queue failure, deadline and supersession are distinguishable without raw provider payloads.
- [ ] Operators can identify unregistered/failed workers, retry/exhaustion, stale lease and held AI operations using the documented existing state.
- [ ] Ingestion completion is clearly separate from downstream processing; completed AI reuse and uncertain outcomes remain visible.
- [ ] Secret/content sentinel tests pass and sanitized COM-38 evidence can show counts/timing/failure anomalies without mailbox content.

## 14. Dependencies

COM-37 baseline and documentation; COM-39 final claim/outcome/deadline semantics. Design the log schema early after COM-37, but complete against COM-39. Requires access to current stdout/stderr and read-only queue/domain metadata in the later verification environment. COM-38 final proof and COM-40 final diagnosis depend on these logs. No cloud account/platform installation is required.

## 15. Non-goals

No telemetry vendor, new DB run-history entity, full analytics dashboard, full body/provider-payload capture, alerting service, retention feature, business performance SLA, or correctness changes unrelated to the observed Gmail lifecycle.

## 16. Failure/recovery scenarios

Log sink unavailable: correctness and claims remain in PostgreSQL; report evidence unavailable instead of declaring success. Process death: find start without finish and inspect expired lease/job; do not expect a crash to log its own completion. Provider failure after partial insertion: record partial counts and unchanged checkpoint, then correlate the retry. AI failure after successful ingestion: identify email/operation hold and follow existing reconciliation, not a sync reset. No worker registered: use registration failure + aged queued state rather than trusting `/health`.

## 17. Verification commands

From backend on the isolated DB: `npm test -- src/tests/gmail-ingestion.test.ts src/tests/gmail.test.ts src/tests/queue.test.ts src/tests/ai-idempotency.test.ts`; `npm test`; `npm run typecheck`; `npm run lint`; `npm run build`. Add logging tests under current Vitest discovery and record exact added paths when implemented. The operational log/SQL recipes added by this ticket must be tested against actual pg-boss schema/version; no nonexistent `sync:status` or monitoring npm command is assumed. See the shared runbook for read-only diagnostics and command provenance.

## 18. Documentation updates

Update backend `STABILIZATION.md` diagnostics with event/counter definitions, retry/worker state recipes and redaction rules. Update backend `README.md` with the diagnostics link. Update `docs/docs/architecture/mvp-architecture.md` with observability boundaries and record final examples beside this sprint pack. Preserve historical audit logs/results as historical evidence.

## 19. Engineering notes

Correctness remains a database assertion; logs explain it. Counts should not sum queue offers into discovered emails. Prefer a terminal per-attempt summary plus selected boundary events over noisy per-message logs. The per-attempt ID set is bounded by current run scope; measure it in the representative fixture before adding aggregation infrastructure. Do not store raw history/page tokens in routine logs or use numeric arithmetic to manufacture the next history ID.
