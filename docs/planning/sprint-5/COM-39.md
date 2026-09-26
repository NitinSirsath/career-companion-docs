# COM-39 — Close Gmail crash-retry and completion-fencing gaps

Status: proposed local ticket; source findings require executable regressions before implementation. Primary repository: backend. Suggested priority: High. Estimate: 8 points.

## 1. Title

Close Gmail crash-retry and completion-fencing gaps while preserving existing idempotency.

## 2. Goal

Make interrupted and retried Gmail syncs recover safely, prevent a superseded worker from reporting success, and bound provider/queue work. Prove repeated ingestion preserves rows and completed AI/domain effects without replacing the current history/lease/claim architecture.

## 3. Context

Stabilization already added history paging, 404 reconciliation, unique email persistence, pending recovery, leases, async requests, durable AI claims and domain ownership constraints. This ticket extends those mechanisms only where source evidence identifies gaps. Queue retries are at-least-once execution; email uniqueness and durable effect claims remain the correctness boundary.

## 4. Current implementation

- `backend/src/jobs/gmailSyncJob.ts:11–70`: queued claim, `{ userId, claim }`, retry/expiry configuration and worker error handling.
- `backend/src/services/gmailSync.ts:57–95,203–244`: random active claim, heartbeat, final checkpoint and catch restoration.
- `backend/src/services/gmailClient.ts`, `src/services/gmailFetcher.ts`: credential refresh/sanitization and 15s Gmail request options.
- `backend/src/jobs/emailProcessingJob.ts`, `src/services/queue.ts`: singleton offers and actual worker initialization.
- `backend/src/services/ai/{pipeline,operations}.ts`, `src/services/matcher.ts`: completed-work reuse, bounded paid attempts, uncertain outcome holds and transactionally unique domain effects.
- `backend/src/tests/gmail-ingestion.test.ts`, `gmail.test.ts`, `gmailSync.test.ts`, `queue.test.ts`, `ai-idempotency.test.ts`, `stabilization-safety.test.ts`: existing coverage to retain/extend.

Installed versions inspected: pg-boss 12.31.0, google-auth-library 11.0.2, gaxios 7.3.1. Resolve exact retry/signal APIs from these installed types/source; do not assume current job expiry terminates arbitrary application promises.

## 5. Problem / gap

G1–G3 in [architecture review](architecture-review.md) are concrete source findings. A hard crash leaves a random active claim that the original queued payload cannot reclaim; its retry can be acknowledged as `SyncInProgressError`. The final checkpoint update ignores affected-row count and can log success after supersession. Pending recovery runs outside heartbeat/deadline checks. Gmail's per-request timeout does not explicitly bound library retries or the separate OAuth refresh request; workers ignore the pg-boss abort signal. Large repeated prefixes and >100 pending records are measured-capacity questions, not automatic justification for a new cursor system.

## 6. Proposed implementation

1. **Reproduce first.** Add tests for crash after active-claim acquisition, disconnect/reconnect immediately before finalization, and a stalled refresh/request. Record the first failed invariant before modifying the service.
2. **Preserve logical identity and fence attempts.** Keep queued payload `{ userId, claim: Q }`. Encode active claim as a parseable value containing Q plus a fresh attempt nonce, in the existing string `syncClaim`; do not add a table. Acquire using an atomic compare-and-set of the observed claim/status/lease. Initial Q can become an active attempt; the same Q can reclaim only an expired active attempt; a newer Q makes the old job obsolete. Read the history anchor under that acquisition boundary, rather than continuing with a pre-acquisition connection snapshot. Every heartbeat and final update matches the exact active attempt and an unexpired lease using database time; an expired attempt must re-acquire, not revive itself by heartbeat. Direct-service test calls retain an explicit local claim path.
3. **Align retry and lease timing.** Use pg-boss metadata and `job.signal` to pass attempt start/expiry and cancellation into the service. Cap renewed leases at that attempt's active-job expiry so a crashed attempt cannot extend beyond when its job becomes retryable. A retry of the same still-live request is retryable/deferred, distinct from an obsolete request; it must not be reported as successful ingestion. Prove the configured retry allowance survives the expiry schedule. Keep three retries after initial execution; do not solve timing with unlimited retries.
4. **Fence persistence and completion.** A successful heartbeat followed by an independent email upsert leaves a check/write race. After Google returns, use a short Prisma transaction to lock the owned `gmail_connections` row, verify CONNECTED + exact attempt + unexpired lease with database time, and persist/reuse that email before releasing the lock. Disconnect/replacement uses the same connection row, so either the email commit precedes the replacement or the stale attempt is rejected; no transaction spans Google work. Keep queue sends outside this transaction and check cancellation/claim before offering each item, including pending recovery. A queue offer already in flight can still settle after disconnect; its committed email remains valid and normal job/AI idempotency applies. Do not promise cancellation of previously accepted downstream work or add an outbox. Advance checkpoint/time and release the claim in one conditional update with the same fence; require exactly one affected row. Zero rows means expired/disconnected/superseded, no successful return/log, and no overwrite of the replacement connection. Keep old history on every incomplete ingestion path. Already committed prefix rows remain available for replay.
5. **Bound external work deliberately.** Retain a 240s cooperative scan budget within the 300s job expiry, including pending recovery. Configure explicit bounded Gmail and OAuth-refresh transport behavior. Preferred baseline: one transport attempt per Google request, at most the library's one reactive auth-refresh/replay cycle, each network attempt capped at 15s and all attempts constrained by remaining job budget/cancellation. Let pg-boss own delayed transient retries. Verify refresh transport defaults/options in installed source. Bound DB lock/statement/transaction waits for this path and treat unresolved queue-send outcomes as uncertain rather than unsent. A timeout wrapper alone must not allow a stale transaction to persist email/checkpoint changes. Record already-accepted queue work separately; do not infer that aborting the caller cancelled it.
6. **Keep errors useful but safe.** Preserve allowlisted status/reason/stage categories through `withGmail` instead of raw provider errors. Distinguish invalid authorization, rate limit, domain policy/permission, unavailable provider, network timeout, queue failure, deadline and supersession. Preserve conditional token writes and quota-not-revoked behavior. A credential-persistence failure must not incorrectly mark success or erase the original diagnostic category.
7. **Verify existing mechanisms, avoid rebuilding.** Exercise multi-page history (including cross-page duplicate references, INBOX additions, exclusions and message 404), initial anchor race and expiration fallback; insert-before-enqueue recovery; real queue retry; concurrent owners; same-mailbox reconnect; >100 pending recovery and partial prefix replay. Record job-send suppression separately from insertion. Continue to hold FAILED/PROCESSING/UNKNOWN AI work for reconciliation; no blanket “recover all” reset.
8. **Capacity stopping rule.** Use at least 1,620 synthetic stored emails and controlled provider delay/history pages in the isolated lane. Demonstrate bounded replay reaches the tail over retries/manual resumes. If the same prefix consumes the entire budget indefinitely, document the reproducer and propose only the minimum persisted continuation fields/atomic page-boundary semantics; review invalid-token recovery, ownership, migration and preserved-data upgrade before implementing that conditional change. No cursor migration is pre-authorized by this default plan.

## 7. Architecture impact

Touches Gmail request → job → service claim lifecycle and shared Gmail transport; regression-test GmailFetcher because it also uses `withGmail`. pg-boss, Prisma and Google clients remain. AI/domain/notification contracts remain safety dependencies, not redesign targets. No transaction stays open across a provider request.

## 8. Data impact

Default: no schema migration. Reuse `gmail_connections.syncClaim/syncLeaseUntil/syncStatus/syncError/lastHistoryId/lastSyncedAt`. State transitions: queued Q → active Q/attempt → IDLE + committed checkpoint; caught transient failure → FAILED + retryable Q; hard crash → expired active claim → new attempt for Q; superseded Q → no write. Revocation remains a connection status, independent from sync status. Existing email/AI/event/action indexes and ownership triggers remain intact. Drain incompatible old claim-format workers during later rollout; never drop data to handle legacy in-flight claims.

## 9. API impact

Preserve `POST /api/gmail/sync` 202 `{ accepted: true }`, 409 concurrent request, 400 disconnected, 401 unauthenticated and the existing generic 500 error envelope for unexpected queue failures. Preserve status fields and optional safe `syncError`. A stale lease is currently projected as FAILED by the status read; retain a useful retry route without pretending the read has repaired DB state. Provider failure after 202 is communicated via status, not a synchronous 503 promise. No public cursor, run-history endpoint, caller-supplied owner or progress percentage is introduced.

## 10. Background-job impact

Gmail job payload retains Q. Worker requests metadata (`includeMetadata`) to identify attempt/retry count/start/expiry and uses the existing signal. Distinguish business outcomes completed/failed/superseded/auth-required from queue acknowledgements. Transient failures retain bounded backoff; auth/obsolete jobs are terminal without a false sync success. Queue unavailable before handoff leaves a recoverable FAILED connection. An accepted job with no worker is diagnosed by lease/queue state. Email singleton suppression is not proof of completed processing; recovery must remain possible after the window without duplicating paid effects. No separate dead-letter queue is added.

## 11. Security/privacy considerations

Derive owner only from auth/job data checked against owned rows. Cross-user payloads must fail before provider/AI work. Preserve conditional token update guards, owner triggers and mailbox identity lock. Redact reasons through an allowlist; never propagate Gmail error bodies, token requests, snippets, addresses or session values into logs. Do not inject faults, corrupt credentials, kill live workers or expire real cursors on the preserved database to run these tests.

## 12. Testing strategy

| Layer | Required evidence |
| --- | --- |
| Unit | Claim parsing/outcome classification, real `googleAuthFailure`, safe provider reason mapping, deadline/cancellation behavior; replace copied 401-or-403 pseudo-logic in `gmailSync.test.ts`. |
| Database integration | Atomic claim exclusion/fencing; pause after a heartbeat, replace/disconnect the connection, then resume the email write and prove rejection; expiry without heartbeat revival; zero-row finalization; unchanged checkpoint on later-page or enqueue failure; unique rows/effects, immutable ownership and preserved user decisions. |
| Real pg-boss | API handoff to worker, catch/retry, killed worker after active claim then same-job recovery, expiry/lease timing, retry exhaustion and no false-success acknowledgment. Use test process control in isolated DB only. |
| Provider-adapter fixtures | Multi-page/duplicate history, 404 history vs 404 message, out-of-scope/old mail, token refresh/persistence/replacement, 401/invalid_grant, quota/permission 403, 429/5xx and stuck transports. Stub actual HTTP/transport boundaries for refresh instead of merely changing mock credentials. |
| AI/domain regressions | Completed legacy/current results cause zero new calls; matching failure resumes without paid replay; unknown calls remain held; operation budgets/attempts and event/action uniqueness hold. |
| Capacity/browser/live | 1,620 synthetic stored rows and >100 pending in isolated DB; browser integration in COM-40; actual Gmail evidence in COM-38. Mocks never substitute for live proof. |

## 13. Acceptance criteria

- [ ] A real queued job interrupted after acquiring its active claim is retried/reclaimed after expiry under the same logical request and finishes without duplicate rows/effects.
- [ ] Same-request busy and obsolete/superseded outcomes are distinct; neither is logged as a successful completed sync.
- [ ] Two concurrent requests cannot own active attempts for one user; separate users remain isolated; stale attempts cannot advance a newer checkpoint or restore cleared credentials.
- [ ] Disconnect/reconnect at finalization causes zero-row update to produce non-success; no `lastSyncedAt` or success record is fabricated.
- [ ] Later-page, DB/queue-send and provider failures retain committed prefix rows and the previous history anchor; rerun completes without duplication.
- [ ] Effective Gmail/refresh attempt bounds and cancellation are tested; pending recovery respects the budget; no email/checkpoint transaction from an expired or replaced attempt passes the write fence. Already-accepted or uncertain queue offers are accounted for and safely deduplicated, not falsely described as cancelled.
- [ ] Auth failures are terminal/reconnectable; quota 403/429 does not revoke a valid connection; retries exhaust to visible failure without a storm.
- [ ] >100 pending and representative-size replay make measured progress or produce a documented blocking capacity finding; no unreviewed continuation migration is introduced.
- [ ] Previously completed AI work incurs zero new provider calls, and uncertain work is not reset; all relevant existing safety tests pass.

## 14. Dependencies

COM-37 and the stabilized schema/client/worker versions; an isolated local PostgreSQL test DB, fixture-only credentials, installed Google/pg-boss dependencies and process-control capability for crash tests. COM-41 consumes the final outcome/counter hooks; COM-38 depends on this ticket. No real mailbox is needed for the failure-injection suite.

## 15. Non-goals

No new queue platform, outbox, ingestion rewrite, schema-wide refactor, automatic AI replay, mailbox switching, scheduled sync, Gmail label/deletion mirroring, global retry service or unrelated notification repair. No production data purge or migrations in this planning task.

## 16. Failure/recovery scenarios

Repeated success: replay may discover zero candidates; identity and effects unchanged. Partial insertion: resume from old anchor, reoffer pending rows. Crash: expired attempt reclaimed with new fence. Lease still live: bounded defer/retry; operator can resume through normal POST after expiry if job allowance is exhausted. API send uncertainty: inspect queue/claim before retry, preserve singleton/DB protections. Invalid grant: retain rows/history, reconnect same account. Supersession: old job terminates without overwriting replacement. Unknown AI effect: hold for review. Non-progressing prefix: retain all data and block capacity acceptance until the minimal fix is approved.

## 17. Verification commands

From backend, only after the existing guard proves a disposable fixture DB: `npm test -- src/tests/gmail-ingestion.test.ts src/tests/gmailSync.test.ts src/tests/gmail.test.ts src/tests/queue.test.ts src/tests/ai-idempotency.test.ts src/tests/stabilization-safety.test.ts`; then `npm test`, `npm run typecheck`, `npm run lint`, `npm run build`. Add new regressions under the existing Vitest discovery `src/**/*.test.ts`; final commands must name any new files accurately. Schema validation/status are in the runbook. Do not run reset/seed/migration commands against the preserved dataset. No new recovery CLI is assumed to exist.

## 18. Documentation updates

Update backend `STABILIZATION.md` with logical/attempt claims, lease/expiry/deadline/retry semantics and explicit operator recovery. Update `docs/docs/architecture/mvp-architecture.md` and the relevant sections of `docs/docs/architecture/high-level-architecture.md`. Record test reproductions and resolved G1–G3 findings beside this pack. If a capacity migration becomes necessary, add its separate schema/upgrade decision before implementing it.

## 19. Engineering notes

Claim encoding avoids a new table while retaining both request identity and attempt fencing. A stable request ID alone is insufficient: a paused old attempt could otherwise write through a replacement's lease. `Promise.race` is not cancellation. pg-boss expiration does not itself prove Google/DB work stopped. Use actual clock/expiry behavior in integration tests and document backoff jitter instead of promising exact retry timestamps. Keep the safety boundary stricter than the availability preference for uncertain paid AI effects.
