# COM-38 — Prove live Gmail incremental sync and repeat-sync idempotency

Status: proposed local ticket; live verification not performed. Primary repository: backend; frontend/browser evidence and docs required. Suggested priority: High. Estimate: 5 points.

## 1. Title

Prove live Gmail incremental sync and repeat-sync idempotency on the preserved dataset.

## 2. Goal

Demonstrate that a controlled real Gmail change is discovered via history, persisted exactly once, and visible in Career Companion while all original approximately 1,620 email identities survive. Repeating sync must not duplicate the email or repeat completed paid AI operations/domain effects.

## 3. Context

Stabilization implemented `users.history.list`, conditional checkpoints, queue handoff, uniqueness, leases and AI claims. Its eight ingestion regressions use mocked Google/queue responses; its browser smoke manually marks sync complete. Neither is live-provider evidence. This ticket preserves the original Sprint 5 requirement and cannot close with mocks, seeded emails, a direct service call, or a fresh database import.

## 4. Current implementation

`backend/src/routes/gmail.ts` queues via `src/jobs/gmailSyncJob.ts`; `src/services/gmailSync.ts` reads history, ingests and finalizes; `src/services/gmailClient.ts` refreshes encrypted credentials; `src/jobs/emailProcessingJob.ts` calls `src/services/ai/pipeline.ts`. `src/services/matcher.ts` creates domain effects on existing owned applications. `frontend/src/routes/gmail.tsx` polls connection/email state; `src/routes/applications.$id.tsx` displays detail/timeline/actions. `backend/prisma/schema.prisma` defines email identity and durable operations. Current source is anchored in [architecture review](architecture-review.md).

## 5. Problem / gap

The actual mailbox, continuation state, runtime commits, existing-data preservation and repeat-sync behavior remain unverified. A final row count alone cannot establish that the correct provider message was discovered or that old records survived. Current errors/counts are also insufficient for a complete proof until COM-41.

## 6. Proposed implementation

Execute this supervised procedure and capture a sanitized evidence report; do not add a new ingestion path:

1. Pass COM-37 readiness and COM-40's final local worker/browser smoke. Use the same final COM-39/41 backend and matching frontend builds. Retain protected baseline manifest `B0`, measured count `N0`, history anchor `H0`, successful-sync time, processing/AI state and budgets.
2. Confirm a recent valid history checkpoint using the normal path. If the existing checkpoint is missing/expired, permit the existing nondestructive scoped reconciliation only after readiness review. Account for its additions separately, record that it was fallback, and establish a new pre-change anchor/manifest. Keep the original `B0`; fallback does not count as the required incremental proof.
3. Introduce one uniquely marked synthetic job-related email into the same real mailbox, in INBOX and within 90 days. Have the mailbox owner send/arrange this controlled message; do not automate outbound mail without separate sending authorization. Use a harmless synthetic company/role/body and a private test marker. If verifying automatic matching, create/select one explicitly designated owned synthetic application before capturing the final pre-change baseline; do not alter a real application's user decisions.
4. Record the actual Gmail API message ID, receipt/internal date and INBOX state privately. The Gmail API message ID is not the RFC `Message-ID` header or a subject match. The record must prove the change occurred after `H0` was established.
5. From a signed-in browser call **the normal** `POST /api/gmail/sync`; capture 202 and request/job correlation. Observe actual pg-boss worker execution and Gmail history mode. Do not invoke `syncUser` directly, manually write a checkpoint, replace Google with a fixture, or set syncStatus in SQL.
6. Wait for bounded ingestion completion, then query the owner-scoped database/API for that exact Gmail ID. Compare subject/sender/receivedAt/thread identity privately with provider metadata. Verify one persisted row, a committed terminal checkpoint `H1`, and counts that reconcile all discovered candidates. Record acceptance, start/end, duration and anomalies.
7. Separately record downstream processing state. Successful live Gemini extraction/matching is optional supplementary evidence, not a ninth Gmail acceptance requirement; COM-40 proves the complete worker/domain/browser path with deterministic provider adapters. If a live AI pass is included and one designated unique application was prepared, verify its supported timeline/action/state without rerunning AI to force a preferred answer. Record budget deferral, unknown outcome or provider failure explicitly; do not label those as completed processing. Diagnose any unexplained regression before sprint closure.
8. Once ingestion completes, run sync again via the same endpoint with no additional intentional mailbox change. Verify the same local email ID and one owner/Gmail row. Compare downstream effects after jobs settle or reach a documented deferred/held state: a job already accepted by the first run may finish during the second, so attribute its effects by email/operation/job rather than treating every count change as a duplicate. Completed-email AI attempts must not increase and email-derived event/action identities must remain unique. Ordinary unrelated arrivals may add rows/history; enumerate them. An unchanged history ID on a quiet repeat is valid.
9. Compare every original identity and immutable field in `B0` with the final dataset, plus explicit user-decision/AI invariants. Explain all additions and allowed pending-processing transitions. Retain the synthetic message/row as evidence; no deletion/reset cleanup on the preserved dataset.

Use the [verification runbook](verification-runbook.md) for the manifest and evidence format. A second small live run after natural access-token expiry can establish real refresh behavior; record it separately from mocked refresh tests. Do not deliberately revoke the primary grant or edit token ciphertext to manufacture failure.

## 7. Architecture impact

No new production architecture. Mandatory live path: browser/session → Express → pg-boss → Gmail → Prisma/normal email-job handoff → Gmail UI. Observe the separate email-worker/AI/domain stage without making successful paid-provider execution a condition of Gmail proof. The test's control message may originate outside the application because its Gmail grant is readonly.

## 8. Data impact

Expected normal changes: one new `emails` row per actual eligible new message, Gmail `lastHistoryId/lastSyncedAt/syncStatus` transitions, normal queue rows and optional new AI/result/domain rows for the control email. Existing uniqueness constraints remain. No schema/index/migration. `B0 ⊆ Bfinal` by stable identity with unchanged baseline immutable metadata; original completed work and user overrides are preserved. The expected final count is `N0 + explained additions`, not blindly 1,621 when unrelated mail or preparatory reconciliation exists.

## 9. API impact

No API changes. `POST /api/gmail/sync` has no caller-supplied owner/body and returns 202 `{ accepted: true }`; 401 is session failure, 400 `GMAIL_NOT_CONNECTED` needs connection repair, 409 `SYNC_IN_PROGRESS` means follow current status. Queue failure currently reaches generic 500. Async Gmail auth failure is observed through status/REVOKED, not the already-returned POST response. `GET /api/gmail/status` and `/messages?limit=20&offset=...` are 200 reads. Follow `nextOffset`; do not infer total count or processing completion from a page or `lastSyncedAt`.

## 10. Background-job impact

Use actual `gmail-sync-job { userId, claim }`, normal email jobs `{ userId, emailId }`, and their existing bounded retry policies. Record attempt count and final business outcome, not just pg-boss “completed”: terminal auth/obsolete work may be acknowledged without a successful sync. Preserve AI operation claims across repeated jobs. Discord is disabled for this verification unless separately authorized for the intended owner; it is not required to send a notification to prove incremental Gmail sync.

## 11. Security/privacy considerations

Use real authenticated sessions with development impersonation disabled for production-like proof. The owner introduces only synthetic content and controls Gmail access. Keep tokens, cookies, raw mail, mailbox addresses and identity manifests private. Sanitize screenshots/logs before attaching to local docs/Linear later. Never use another user's mailbox or grant broader Gmail scopes for the test.

## 12. Testing strategy

Unit, DB, integration and mocked-provider suites are prerequisites/regression support from COM-39/40, not the acceptance evidence for this ticket. Mandatory live lane proves the eight original requirements: change, discovery, persistence, uniqueness, repeated sync, checkpoint, dataset preservation and observability. Browser observation uses real API/worker/provider. Real OAuth refresh evidence is distinct from the mocked credential-update test; if natural expiry is not observed, mark that evidence pending rather than claiming it passed.

## 13. Acceptance criteria

- [ ] A dated provider-side control change after a captured valid `H0` is documented with exact message identity.
- [ ] Authenticated POST returns 202, and a correlated actual worker consumes Gmail **history**, not only a list/fallback scan.
- [ ] Exactly one row exists for `(owner, control Gmail ID)`; immutable fields match the controlled provider message.
- [ ] The terminal checkpoint is committed only after ingestion/handoff; for the new history event `H1` advances in numeric-string order, without assuming contiguous IDs or converting to JS Number.
- [ ] A second normal sync leaves the same row ID and no additional copy, event/action or completed-operation paid attempt. All unrelated changes are explained.
- [ ] Every original `B0` email identity and immutable metadata value remains; no reset/delete/re-import has occurred, and user decisions/completed AI work are preserved.
- [ ] Counts, execution/queue timing, retries, failures, fallback mode and anomalies are captured and reconciled; `lastSyncedAt` is not treated as AI completion.
- [ ] The control email is visible in the browser with its actual processing state. Optional live AI/matching success is separately labeled; deferred/held/failed processing is explained, never reported as complete. COM-40's successful deterministic worker/domain/browser evidence is linked.
- [ ] The report separates preparatory reconciliation, live history proof, live refresh observation, mocked regressions and browser fixtures.

## 14. Dependencies

COM-37 baseline/readiness; COM-39 recovery/bounds; COM-41 diagnostics; COM-40 local worker/browser acceptance on the build used here. Requires retained dataset/backup, valid same-mailbox readonly grant, authenticated production-like environment, actual registered workers, access to protected logs/read-only DB evidence, and a recorded AI budget/backlog policy. A nonzero paid-call allowance is needed only for an optional live AI pass; do not bypass existing workers/claims to make that pass succeed. No Linear access is required to execute the evidence procedure.

## 15. Non-goals

No clean install, bulk reimport, mailbox-switch migration, old-mail expansion, Gmail deletions/label mirroring, push notifications, scheduler, AI accuracy benchmark, new dashboard, forced AI replay or notification-channel expansion.

## 16. Failure/recovery scenarios

History 404: retain original data, complete nondestructive reconciliation if safe, then restart the live proof with a new change after a valid checkpoint. Transient Google/queue failure: keep persisted prefix and old checkpoint; observe bounded retries. Lost session: sign in and inspect status before repeating POST. 409: wait for the existing request. Revoked grant: reconnect the same mailbox; do not disconnect/reset as routine preparation. Timeout/worker crash: use COM-39 recovery and retained checkpoint. Ambiguous AI effect: reconcile the operation; never reset it. Any unexplained missing baseline identity stops acceptance immediately, retaining all evidence.

## 17. Verification commands

Backend prerequisite checks, **only on the guarded test database**: `npm test -- src/tests/gmail-ingestion.test.ts src/tests/gmail.test.ts src/tests/ai-idempotency.test.ts src/tests/stabilization-safety.test.ts`; `npm run typecheck`; `npm run lint`; `npm run build`. Frontend: `npm test -- src/tests/gmail.test.tsx src/tests/stabilization-ui.test.tsx`; `npm run typecheck`; `npm run lint`; `npm run build`. These commands exist in package scripts and use Vitest file filters. Live proof uses the UI/endpoints above and read-only manifest queries; no live-verification npm script currently exists. Do not run the fixture smoke script against the preserved DB.

## 18. Documentation updates

Add the completed sanitized live report beside this sprint pack, linked from `docs/docs/architecture/mvp-architecture.md` and `docs/docs/engineering/stabilization-audit.md` as new evidence, retaining historical limits. Update backend `STABILIZATION.md` only with verified provider/recovery facts. Record private artifact locations and digest results without committing raw mailbox exports.

## 19. Engineering notes

A row appearing after a seeded DB insert proves nothing about Gmail history. The strongest proof correlates provider ID, request/attempt, history mode, final DB identity and browser result. On initial expired history, record both the original preservation baseline and the later incremental anchor; never discard the original manifest. Avoid performance claims from one control email: timing is operational evidence, while capacity regressions belong to COM-39.
