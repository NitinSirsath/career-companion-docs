# S8-01 — Let the user correct a wrong email match (ADR-0003 first)

| Field | Value |
| --- | --- |
| Status | Implemented locally — [evidence](execution-report.md), remote/live acceptance pending (2026-10-03) |
| Phase / sprint | Sprint 8 — order 1 of 4 |
| Repository | career-companion-backend (matcher, email/action/application/Gmail routes and services, notification job, one additive migration, shared contracts); career-companion-frontend (synced contracts, API client, Gmail page, application detail); career-companion-docs (ADR-0003 first, then domain model, user flows, architecture notes) |
| Size / priority | L / data correctness: a wrong match can be fixed from the UI |
| Depends on | S6-R07 done (closeout); OD-07 accepted as ADR-0003 (step 1 of this ticket, before any code); Sprint 8 entry ([README §5](README.md#5-dependencies-and-entry-conditions)) |
| Blocks | — |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-13 (and S56-01 for the guard); [roadmap §6](../README.md#6-owner-decisions) OD-07; [Sprint 8 README](README.md) §2, §5, §7; [S6-R07](../sprint-6/review-2026-10-02/S6-R07-guard-automatic-rematch.md) "separate product decision" |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

The owner can fix a wrong email-to-application link from the UI. A matched email (auto-matched or user-matched) can be moved to another application or unlinked. An ignored email can be linked to an application. The old application's event and action are retired with a reason, never deleted. Its `aiStatus` is recomputed from the emails still linked to it. `userStatus` never changes. Later mail in the same thread follows the corrected link. A correction calls no AI provider and no Gmail API, enqueues no job and sends no Discord notification. The policy is recorded first, as ADR-0003.

## 2. Why it exists

- **A wrong link is permanent today** (S56-13, confirmed, medium). Auto-matched, user-matched and ignored emails can never be re-linked. The wrong application keeps an event that cannot be removed, a raised AI status and an action the user can only dismiss.
- **The error spreads.** The thread tier copies the link of any matched email in the same thread, so later mail follows the wrong application automatically.
- **Sprint 6 did not cover it.** Manual status correction can hide a wrong AI status but cannot move evidence. Rematching was a Sprint 6 non-goal ([sprint-6 README §E](../sprint-6/README.md#e-surviving-scope-and-contract-rules)), and `mvp-architecture.md:66` still lists "no rematching/merging" as a limit. S6-R07 says reopening a match "needs a separate product decision". OD-07 is that decision; ADR-0003 records it.
- S6-R07 must land first. It stops the worker from re-linking or un-matching an already matched email. Without it, a correction could be undone, or effects could split across two applications.

## 3. Current behavior

Line numbers were checked on 2026-10-02 against the backend and frontend folders. Re-check them before editing; S6-R07, S6-R08, S7-03 and AI-20 edit some of the same files first. Find code by symbol name.

**3.1 Facts (verified in code).** Backend paths are under `career-companion-backend-main/`, frontend paths under `career-companion-frontend-main/`.

1. **Only unresolved emails can be resolved.** `MatcherService.resolveEmailMatch` (`src/services/matcher.ts:264-323`) throws `INVALID_MATCH_STATE` unless the email is AMBIGUOUS or UNMATCHED (:279-284). The email router has only `GET /ambiguous` (`src/routes/email.ts:17`), `GET /unmatched` (:54), `POST /:id/resolve` (:91-134) and `POST /:id/retry` (:148). The frontend client calls only these (`src/api/client.ts:242-293`).
2. **Effects never move.** In `applyMatch` (`matcher.ts:126-220`), a `USER_CONFIRMED` link is legal only for an UNMATCHED or AMBIGUOUS email, or as a replay of the same link (:147-159; comment "effects never move" at :150).
3. **Ignore keeps `applicationId`.** The ignore path (`matcher.ts:291-302`) sets `IGNORED` and `USER_CONFIRMED` but does not clear `applicationId` (:293-299). For a normal ambiguous email it is already null. In the S6-R07 window (S56-01) an auto-matched email could become AMBIGUOUS with `applicationId` kept and its effects left on that application; ignoring it leaves them there.
4. **Lock order and effects.** `applyMatch` runs one transaction (:132-212): email row `FOR UPDATE` (:134); `AI_AUTO` gives way to `USER_CONFIRMED` (:137-141); owned application `FOR UPDATE` (:142); email update (:160-167); raise-only `aiStatus` (:168-175); one `EMAIL_PROCESSED` event per application and email (:176-192); at most one action per application and email (:193-211); a notification job after commit (:213-219). Events and actions are unique on `(applicationId, emailId, type)` (`prisma/schema.prisma:254`, :277; `prisma/migrations/20260926110000_domain_integrity/migration.sql:4-5`).
5. **`aiStatus` is a maximum.** `canTransition` accepts any state from null and otherwise only a state of equal or higher order (`matcher.ts:338-355`, order at :344-352). So stored `aiStatus` equals the highest `inferState` result (:325-336) ever applied to the application by any email linked to it, or null. An event's `newState` is the raised value at that time, not the email's own inferred state (:185-186), so events cannot be used to recompute it. Only `applyMatch` writes `aiStatus` (:172-175); MCP never does (`src/services/externalSubmission.ts:8`), nor does the status PATCH (`src/services/application.ts:227-232`).
6. **Thread tier.** `matchEmailToApplication` (`matcher.ts:16-117`) first looks for the newest MATCHED email in the same thread, from any source, ordered by `receivedAt` then `id` descending (:42-64, order at :52). This read happens before any lock. It links with `AI_AUTO`. A `USER_CONFIRMED` email returns early (:28) or replays its own link (:30-40). The worker calls this from `src/services/ai/pipeline.ts:116-118`.
7. **Actions.** `Action.status` is a plain string, default `PENDING` (`schema.prisma:267`). `PATCH /api/actions/:id` (`src/routes/action.ts:33-54`) accepts PENDING, COMPLETED or DISMISSED (`src/contracts/action.ts:21-23`). `updateActionStatus` (`src/services/action.ts:68-143`) reads, then updates without a lock (:74-96). Lists: `getUserActions` (`action.ts:11-42`), `getApplicationActions` (`application.ts:349-396`), and `pendingActionCount` from `enrichment` (`application.ts:43-51`). The notification job skips non-PENDING actions (`src/jobs/notificationJob.ts:62`).
8. **Events.** The timeline (`getApplicationEvents`, `application.ts:271-342`) returns every event in recording order. `recentEvent` is the newest event per application via `LATERAL ... LIMIT 1` (`loadRecentEvents`, :83-125, SQL :96-104). `AUTOMATION_SUBMITTED` events have a non-null `externalSubmissionId` (`schema.prisma:246`) and null `emailId` and states; MCP creates them (`externalSubmission.ts:214-215`).
9. **Ownership triggers.** `email_ownership` (`20260926110000_domain_integrity/migration.sql:16-26`); action and event email-ownership triggers on INSERT or UPDATE (:28-37); `event_submission_ownership` (`20261002150000_automation_submissions/migration.sql:128-139`).
10. **Sprint 6 write pattern.** `PATCH /api/applications/:id/status`: strict body (`src/contracts/application.ts:113-116`), owned row lock, revision compared before no-op detection, 409 `STATUS_CONFLICT` (`application.ts:233-264`; `src/routes/application.ts:106-128`). A `ZodError` becomes 400 `VALIDATION_ERROR` (`src/middleware/error.ts:19-22`). MCP intake serializes per user with `lockUser` (`src/utils/advisoryLock.ts:8-22`).
11. **Gmail page.** `GET /api/gmail/messages` (`src/routes/gmail.ts:421-454`, select :431-447; `src/contracts/gmail.ts:50-67`) returns no `applicationId`, link source or application name. The table headers are Subject, Sender, State (shows relevance), AI Status (shows processing state) and Received (frontend `src/routes/gmail.tsx:306-312`). The messages response is not runtime-parsed (`client.ts:429-437`); events are (`client.ts:222-232`).
12. **Detail page.** `TimelineEventItem` (`src/routes/applications.$id.tsx:72-118`) shows source email and AI interpretation, with no correction control. The dashboard pickers for ambiguous and unmatched emails (`src/routes/index.tsx:135`, :213) list the dashboard's current page of applications; the backend caps a page at 20 (`src/utils/pagination.ts:4`, :17).

**3.2 Suspected risks (not demonstrated).**

- A restored database may hold split effects from the S6-R07 window (fact 3): one email with effects on two applications. The count is unknown. §4 item 5 cleans them when that email is corrected.
- Thread race: the worker reads the thread link before any lock (fact 6). If a correction commits between that read and `applyMatch`, the new email lands on the old application. This comes from reading the code; no test shows it today.
- A legacy application whose `aiStatus` does not equal the maximum of its linked emails' inferred states (for example, written by older code) will change when recomputed. Unverified whether any exist.

## 4. Scope

**4.0 Decisions (OD-07).** The owner accepts ADR-0003 before any code. Recommendation (roadmap OD-07): allow move and unlink for auto-matched, user-matched and ignored emails; retire, never delete, the old application's event and actions with a reason; recompute `aiStatus`; never touch `userStatus`. Sub-choices under OD-07, each with the recommended answer this ticket assumes:

- (a) **Unlink result:** `IGNORED` + `USER_CONFIRMED`, `applicationId` null. The worker never re-matches it (fact 6, :28), and it means the same as today's ignore. (Alternative: back to UNMATCHED review.)
- (b) **Ignored emails** can be linked (a move from no application). Unlink is not offered: they have no link. A legacy ignored email that still holds an `applicationId` (fact 3) is cleaned only by a move to another application.
- (c) **Action status carries over.** If the retired action was COMPLETED or DISMISSED, the new action on the target keeps that status. Otherwise PENDING. With legacy split effects (several retired actions), COMPLETED wins over DISMISSED. A move must not bring back a task the user already handled.
- (d) **Thread mail after an unlink** is not linked by the thread or company tier; it stays UNMATCHED for review. Otherwise the company tier can rebuild the same wrong match for every new thread email. An unlinked email looks the same as one ignored through today's resolve flow (a), so this rule also applies to new thread mail after a normal ignore. ADR-0003 must say so.
- (e) **No Discord notification** for effects a correction creates. The user is in the UI making the change.
- (f) **Conflict check by expected state**, not a new revision column (§7).

**4.1 ADR-0003 (first deliverable).** Write `docs/architecture/decisions/ADR-0003-correcting-email-matches.md` in the ADR format (header table Status, Date, Decides, Detail; Context; Decision, numbered; Not part of this decision; Consequences; Alternatives considered; Future evolution). Status "Proposed" until the owner accepts it, then "Accepted by the owner on <date>". It must record:

1. Which states can be corrected and how (§4.0 a, b). UNMATCHED and AMBIGUOUS keep the existing resolve flow.
2. The resulting email state: move = MATCHED + `USER_CONFIRMED` on the target; unlink per (a). AI never changes a corrected email again.
3. Retirement: every non-retired event and action of this email on any application other than the target gets `retiredAt` and a reason (`EMAIL_MOVED` or `EMAIL_UNLINKED`). Rows stay. Retired rows leave action lists, counts, notifications, `recentEvent` and the `aiStatus` input, but stay on the timeline, marked. `AUTOMATION_SUBMITTED` events are never retired.
4. Target effects follow the normal link rules (one event, at most one action, S7-03's deadline rule). A retired row for the same application, email and type is reactivated, not duplicated. Action status per (c).
5. `aiStatus`: each source application is recomputed from the emails still MATCHED to it with the matcher's own `inferState`/`canTransition`; null when none remain; it may go down. The target uses the normal raise-only rule. `userStatus`, `userStatusSetAt` and `userStatusRevision` are never written.
6. Thread rule: a user-confirmed decision in the thread wins over automatic links; (d) after an unlink. The thread decision is re-made under lock.
7. Concurrency: expected-state compare with 409 on stale; lock order user match lock → email → applications by ascending id.
8. Side effects: none beyond the database writes above. No provider or Gmail call, no job, no notification, no AI-ledger change, no correction event or audit log (latest-only provenance, sprint-6 D2).
9. Not part: §5 of this ticket.

**4.2 Backend: endpoint and contracts.**

1. **Route.** `PATCH /api/emails/:id/match` in `src/routes/email.ts` (behind `router.use(requireAuth)` at :11). Body `CorrectEmailMatchRequestSchema` in `src/contracts/email.ts`: `z.strictObject({ applicationId: z.uuid().nullable(), expectedMatchState: z.enum(['MATCHED', 'IGNORED']), expectedApplicationId: z.uuid().nullable() })`, refined so `applicationId !== expectedApplicationId`, and `applicationId: null` (unlink) needs `expectedMatchState: 'MATCHED'`.
2. **Response 200** `CorrectEmailMatchResponseSchema`: `{ email: { id, matchState, matchConfirmedBy, applicationId }, affectedApplicationIds: string[] }` (the target and every source). Errors: 400 `VALIDATION_ERROR`; 401; 404 `NOT_FOUND` (email missing or foreign, identical); 404 `APPLICATION_NOT_FOUND` (target missing or foreign, identical); 409 `MATCH_CONFLICT` (current `matchState` or `applicationId` differs from the expected values; checked right after the owned email is locked and before any other check, so a replay of a committed move is 409, as in Sprint 6); 409 `MATCH_NOT_CORRECTABLE` (a move of an email with no AI result); sanitized 500.
3. **Messages list.** `GET /api/gmail/messages` also selects `applicationId`, `matchConfirmedBy` and `application: { id, companyName, jobTitle }`. `EmailMessageSchema` gains them as `.nullable().optional()`, like the other added fields there.
4. **Events contract.** `ApplicationEventResponseSchema` (`contracts/application.ts:122-138`) gains required `retiredAt: IsoDateTimeSchema.nullable()` and `retiredReason: z.enum(['EMAIL_MOVED', 'EMAIL_UNLINKED']).nullable()`. Required, because events are runtime-parsed and missing fields are contract errors (sprint-6 §E rule 7). `RecentEventSchema` and the action schemas do not change.

**4.3 Backend: the correction transaction.** `MatcherService.correctEmailMatch(userId, emailId, request)` in `matcher.ts`, one `prisma.$transaction`:

5. Take a new per-user lock `lockUser(tx, LOCK_NAMESPACE.emailMatches, userId)` (new namespace value in `advisoryLock.ts`). Lock the email `WHERE id AND "userId" FOR UPDATE` (none → `NOT_FOUND`). Compare `matchState` and `applicationId` with the expected values (`MATCH_CONFLICT`). Find the **sources**: the email's current `applicationId`, plus every application with a non-retired event or action for this email, minus the target. Lock target and sources in one `SELECT id FROM applications WHERE id = ANY(...) AND "userId" = ... ORDER BY id FOR UPDATE`; a missing target is `APPLICATION_NOT_FOUND`.
6. **Retire** on the sources: events with this `emailId`, `retiredAt IS NULL` and `externalSubmissionId IS NULL`; actions with this `emailId` and `retiredAt IS NULL`. Set `retiredAt = now()` and the reason. Keep each retired action's `status`.
7. **Write the email:** move → target, MATCHED, `USER_CONFIRMED`; unlink → per (a).
8. **Target effects (move only):** extract `applyMatch`'s effect code (`matcher.ts:168-211`, as changed by S7-03) into one helper used by both paths. When a row exists for (target, email, type): reactivate it if retired (clear both columns; keep its stored status), else keep it. When none exists, create it as today; the action's status follows (c). The correction never calls `enqueueNotificationJob`.
9. **Recompute** each source's `aiStatus` with a new exported pure helper `aiStatusFromEvidence(results)` that folds `inferState`/`canTransition` from null over the `AIProcessingResult` of every email now MATCHED to it. Write only when the value changes. Never write `userStatus*`.
10. **Log** one line after commit: `{ event: 'email_match_corrected', emailId, kind: 'MOVE' | 'UNLINK', fromApplicationIds, toApplicationId, retiredEvents, retiredActions }`. No subject, sender or company.

**4.4 Backend: read paths, worker and notifications.**

11. `loadRecentEvents` adds `AND "retiredAt" IS NULL` inside the LATERAL subquery. `getApplicationEvents` returns retired events with the two new fields.
12. `getUserActions`, `getApplicationActions` and the `pendingActionCount` filter in `enrichment` add `retiredAt: null`.
13. `updateActionStatus` writes with `updateMany({ where: { id, retiredAt: null } })`. An owned retired action returns 409 `ACTION_RETIRED` and is not changed.
14. The notification job also returns early when `action.retiredAt` is set (next to `notificationJob.ts:62`).
15. **Thread tier.** Move the thread lookup into a helper `threadDecision(db, email)` returning LINK (application), STOP or NONE. Order: the newest (same `receivedAt`, `id` order) `USER_CONFIRMED` email in the thread that is MATCHED or IGNORED decides (MATCHED → LINK; IGNORED → STOP, per d); otherwise the newest MATCHED email of any source (today's rule) → LINK; otherwise NONE. STOP leaves the email UNMATCHED without trying the company tier.
16. **Re-check under lock.** When the thread tier chose the application, `applyMatch` takes the same per-user lock first, then the email lock, then calls `threadDecision(tx, email)` again. If the answer changed, it links to the new application, or returns without linking (STOP), or tells the caller to continue with the company tier (NONE). The pre-lock read is only a hint. A stale STOP is not re-checked: the email stays UNMATCHED and waits in the unmatched review, which is safe. Company-tier and resolve paths keep their current locks.

**4.5 Frontend.**

17. Run `npm run sync-contracts`. Add `api.correctEmailMatch(emailId, body)`, parsed with `CorrectEmailMatchResponseSchema`. On success invalidate `['gmailMessages']`, `['applications']`, `['actions']`, and `['application', id]`, `['application-events', id]`, `['application-actions', id]` for each affected ID.
18. **Gmail page.** Add an "Application" column: the linked application name (link to `/applications/$id`) with "auto" or "you"; "Ignored"; "Needs review" (AMBIGUOUS); "Not linked" (relevant UNMATCHED); "—" otherwise. MATCHED and IGNORED rows get a visible "Change link" button, never inside a tooltip (AI-20, AIF-03). It opens a dialog (`src/components/ui/dialog.tsx`) with an owned-application select paged 20 at a time, the current application left out, plus "Not linked to any application" for MATCHED rows. Suggested text: "The email's timeline entry and action move to the application you choose. The old ones stay on the old timeline, marked as moved. The AI status of both applications is worked out again from their emails. Your own status choices do not change. No notification is sent."
19. **Application detail.** Each non-retired event with a `sourceEmail` gets a "Wrong application?" button that opens the same dialog with expected `MATCHED` and this application. A retired event is shown muted with "Moved to another application" or "Unlinked from this application" and its `retiredAt` date, and "No longer counts toward this application's AI status" instead of the state change.
20. **Errors**, following UF-09: no automatic resend and duplicate clicks blocked. 409 `MATCH_CONFLICT`: refetch, then show "This email's link changed. Here is where it is now." 404: refetch and say it is no longer available. Timeout, lost response, 5xx or malformed success: refetch and ask the user to check the result. 409 `ACTION_RETIRED` on an action button: refetch the lists. Today both action mutations invalidate only on success (`updateMutation` in `ActionQueueSection`, `src/routes/index.tsx:34-44`, and in `ActionItem`, `src/routes/applications.$id.tsx:126-135`), so they need an error path.

## 5. Out of scope

- Automatic re-matching of any kind, and any change to the company/role tier or its normalization ([Sprint 8 README §4](README.md#4-non-goals)).
- Bulk correction, including "move the whole thread". Each email is corrected on its own.
- Application merge, archive or delete (OD-10), and creating an application from the dialog (create it first, then move).
- A full correction audit log or correction events (latest-only provenance stays).
- Unlinking or re-resolving automation submissions (ADR-0002 "Future evolution").
- Undo, restoring retired rows by hand, or changing which events the timeline shows beyond the retired marker.
- Re-running AI, re-fetching mail or re-deriving action text from a newer AI result.

## 6. Likely files and components

Docs: `docs/architecture/decisions/ADR-0003-correcting-email-matches.md` (new, first).

Backend: `src/services/matcher.ts` (`correctEmailMatch`, effect helper, `aiStatusFromEvidence`, `threadDecision`, `applyMatch` re-check); `src/utils/advisoryLock.ts`; `src/routes/email.ts`; `src/contracts/email.ts`, `application.ts`, `gmail.ts`; `src/services/application.ts`; `src/services/action.ts`, `src/routes/action.ts`; `src/routes/gmail.ts`; `src/jobs/notificationJob.ts`; `prisma/schema.prisma`; `prisma/migrations/<timestamp>_retire_match_effects/migration.sql` (new); tests in §11.

Frontend: `src/contracts/*` (synced only); `src/api/client.ts`; `src/routes/gmail.tsx`; `src/routes/applications.$id.tsx`; `src/routes/index.tsx` (action queue error path only); new `src/components/MatchCorrectionDialog.tsx`; `src/tests/fixtures.ts`; tests in §11.

## 7. Implementation notes

- **Migration.** One additive migration, timestamp after `20261002150000` and after any Sprint 7 migration. New enum `RetiredReason ('EMAIL_MOVED', 'EMAIL_UNLINKED')`. Nullable `"retiredAt" TIMESTAMP(3)` and `"retiredReason" "RetiredReason"` on `application_events` and `actions`, each table with `CHECK (("retiredAt" IS NULL) = ("retiredReason" IS NULL))`. No backfill, no index (the `applicationId` indexes serve one user). Unique keys stay, which is why §4 item 8 reactivates instead of inserting.
- **Lane hazard (TEST-03).** Each migration lane builds its legacy schema from every migration except its own (`upgradeLane` in `scripts/verify-mcp-migration.cjs:169-176`; same pattern in `verify-ai-migration.cjs:114-119` and `verify-migration-preservation.cjs:126-131`), so this migration runs *before* each lane's own migration. It must not reference `externalSubmissionId`, `userStatusRevision` or the BYO AI tables. That is why the `AUTOMATION_SUBMITTED` guard is in the service and its tests, not in a CHECK. The MCP and Sprint 6 lanes snapshot `SELECT * FROM actions` (`verify-mcp-migration.cjs:157`; `verify-migration-preservation.cjs:109-110`, events too). The new null columns exist before and after, so they should still pass. Unverified until the lanes run.
- **Triggers.** Retiring is an UPDATE, so the event and action ownership triggers (fact 9) fire. They pass for consistent rows; the tests run through them.
- **Why expected state, not a revision.** The link is fully described by `matchState` and `applicationId`, which the UI already shows. A compare under the email lock gives the Sprint 6 guarantee (stale → 409, checked first) without a new column that every matcher write would have to bump. An A→B→A change by another tab is not detected; the user's request still means what they saw.
- **Lock order** after this ticket: per-user match lock → email rows → application rows by ascending ID. Status PATCH locks one application; MCP takes its own namespace then one application; company-tier and resolve paths take email → application. No path waits in the reverse order. Two opposite moves (e1 A→B, e2 B→A) are already serialized by the per-user lock; the ascending ID order is a second guard.
- **Background jobs.** No new job or queue. The correction enqueues nothing. A later worker replay of a corrected email takes the `USER_CONFIRMED` path (:30-40) and follows existing replay rules, including its existing notification enqueue for the target's action; that behavior is unchanged.
- **API and contract.** Backend first, then the frontend sync commit (S7-01's drift check is red between them). An old frontend ignores the new fields. A new frontend on an old backend fails the events parse by design (rule 7). If S6-R08 already changed `contracts/application.ts`, build on its form.
- **Failure and recovery.** One transaction: a failure leaves nothing half-moved. A lost response is handled by re-reading (§4 item 20); a resend gets 409. A retired row is never deleted, so any correction can be reversed by another correction.
- **Rollback.** Revert the code; the columns stay, nullable and harmless. Old code ignores `retiredAt`, so retired events and actions show again on their old applications while the email stays on its new one. Recomputed `aiStatus` values stay. Record any rollback in the execution report.

## 8. Dependencies

- OD-07 and §4.0 (a)–(f) accepted in ADR-0003 (§4.1). No code before that.
- S6-R07 done ([closeout](../sprint-6/closeout/README.md), order 3). This ticket relies on its guard and keeps its tests green.
- Rebase on S7-03 ([deadline rule](../sprint-7/S7-03-safe-action-deadlines.md), same effect code), [AI-20](../byo-ai/AI-20-recovery-path-and-status-ui.md) (same Gmail table) and [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) (same contract file). Reuse the lock-wait test pattern from [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md).
- CI green once [S7-01](../sprint-7/S7-01-ci-and-toolchain-pins.md) exists.

## 9. Security and privacy

- Owner-only: `requireAuth`; email and target locked by ID and `userId`; missing and foreign give the same 404. The database ownership triggers (fact 9) back this up.
- Strict body; unknown keys are rejected. IDs are UUID-validated.
- No email body, Gmail call or AI call. The only data read is stored metadata and the stored AI result.
- New response fields are owner-scoped metadata: an owned application's ID, company and title; retirement time and reason.
- The log line holds IDs, counts and a kind only.
- Cookie-session writes keep today's posture; the Origin check on cookie writes is release-track item 2, not this ticket.

## 10. Acceptance criteria

- [ ] ADR-0003 exists with every §4.1 item, is accepted by the owner with a date, and is committed before any S8-01 code.
- [ ] Moving an auto-matched email from A to B: the email is MATCHED, `USER_CONFIRMED`, on B. A's event and action still exist with `retiredAt` and `EMAIL_MOVED`. B has exactly one non-retired event and at most one non-retired action for the email. A's `AUTOMATION_SUBMITTED` event is unchanged.
- [ ] The same holds for a user-matched email, and for an ignored email linked to B (including a legacy ignored email whose effects sit on A).
- [ ] Unlinking gives `IGNORED`, `USER_CONFIRMED`, null application; A's effects are retired with `EMAIL_UNLINKED`.
- [ ] A has emails X (inferred INTERVIEW) and Y (RECRUITER_CONTACT). Moving X leaves A at RECRUITER_CONTACT; moving Y too leaves A at null. B rises by the normal rule. `userStatus`, `userStatusSetAt` and `userStatusRevision` are unchanged on both.
- [ ] A retired action is absent from `GET /api/actions`, `GET /api/applications/:id/actions` and `pendingActionCount`. `PATCH /api/actions/:id` returns 409 `ACTION_RETIRED` and changes nothing. The notification job skips it.
- [ ] The timeline still returns a retired event with both fields; `recentEvent` skips it.
- [ ] Moving an email whose action on A is DISMISSED creates B's action as DISMISSED. Moving it back from B to A reactivates A's rows with their stored status (no new rows, no unique violation) and retires B's.
- [ ] A stale expected state gives 409 `MATCH_CONFLICT` and writes nothing. Missing or foreign email gives 404 `NOT_FOUND`; missing or foreign target gives 404 `APPLICATION_NOT_FOUND`; an unknown key or a target equal to the expected application gives 400.
- [ ] During a correction: no provider call, no `GmailFetcherService.fetchMessageBody` call, no `enqueueNotificationJob` or `enqueueEmailProcessingJob` call; `ai_operations` and `ai_usage_days` rows unchanged.
- [ ] Thread follow-on: after e1 moves from A to B, a new e2 in that thread is matched to B. If the worker read the thread before the move committed (barrier), e2 still lands on B. Removing the re-check makes that test fail (§11.3).
- [ ] A user-confirmed link in a thread wins over a newer automatic link. After an unlink, or a normal ignore through resolve, new thread mail stays UNMATCHED and appears in `GET /api/emails/unmatched`.
- [ ] With S6-R07 in place, a worker replay of a corrected email changes no link, adds no effect and reactivates nothing.
- [ ] Concurrency: two opposite moves both succeed with no deadlock; two identical moves give one 200 and one 409; a correction and a status PATCH on A both apply (the wait is proven, not slept); a correction and an MCP link to A both apply and the `AUTOMATION_SUBMITTED` event is untouched.
- [ ] Backend and frontend `src/contracts` are identical after `sync-contracts`.
- [ ] The Gmail page shows the Application column and a keyboard-reachable "Change link"; the dialog moves and unlinks; 409 and uncertain outcomes refetch and never resend. The detail page offers "Wrong application?" and shows retired events muted with label and date.
- [ ] The migration applies through `guarded-migrate.cjs`; the three migration lanes pass.
- [ ] Full backend and frontend suites, typecheck, lint (0 errors) and build pass. The smoke passes.

## 11. Testing

**11.1 Backend tests.**
- New `src/tests/match-correction.test.ts` (DB): route cases from §10 (move, unlink, ignored, legacy split, reactivation, carry-over, aiStatus, all errors, no side effects with `vi.mock` of both enqueue functions and a spy on the fetcher), plus `aiStatusFromEvidence` cases (none → null, order-independent maximum, REJECTED above OFFER).
- `src/tests/matching-concurrency.test.ts`: add the thread, S6-R07 and concurrency cases from §10, with the existing `barrier()` pattern (:14-20) and the S6-C01 `pg_stat_activity` lock-wait proof. Drive `MatcherService.matchEmailToApplication`, the worker's entry point. For the thread race, pause after the pre-lock thread `prisma.email.findFirst`.
- `src/tests/matcher.test.ts`: `threadDecision` order and STOP.
- `src/tests/action.test.ts`, `src/tests/application-evidence.test.ts`, `src/tests/notificationJob.test.ts`: retired rows in lists, counts, PATCH, `recentEvent`, timeline fields and the job.
- `src/tests/gmail.test.ts`: in the strict-keys test (`'returns messages matching the strict contract shape'`, :582-626, inside `describe('GET /api/gmail/messages (COM-20)')` at :571), add `application`, `applicationId` and `matchConfirmedBy` to the sorted key list (:604-620). Replace `expect(keys).not.toContain('applicationId')` (:624) with a check that all three are null for that unmatched email. Keep the `createdAt`, `updatedAt` and `userId` checks. S8-02 later adds `processingStuck` to this list. New cases: messages include the link fields; another user's application is never returned.

**11.2 Frontend tests.** New `src/tests/match-correction.test.tsx` (dialog, move, unlink, 409, uncertain outcome, no resend, keyboard focus); update `src/tests/gmail.test.tsx` (column, button), `src/tests/applications.test.tsx` (retired event rendering, "Wrong application?", 409 `ACTION_RETIRED` refetch), `src/tests/action-queue.test.tsx` (409 `ACTION_RETIRED` refetch) and `makeEvent` in `src/tests/fixtures.ts:37` (`retiredAt: null`, `retiredReason: null`).

**11.3 Mutation checks** (temporary, never committed): remove the re-check in `applyMatch` → the thread-race test fails; remove `retiredAt: null` from the `pendingActionCount` filter → the count test fails. Restore and confirm `git diff --exit-code`.

**11.4 Commands.**

```sh
# backend repo root
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/match-correction.test.ts src/tests/matching-concurrency.test.ts src/tests/matcher.test.ts src/tests/action.test.ts src/tests/application-evidence.test.ts src/tests/notificationJob.test.ts src/tests/gmail.test.ts
npm test
# each lane needs its own two fresh, empty career_companion_*test databases; recreate them between scripts
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=... UPGRADE_DATABASE_URL=... node scripts/verify-mcp-migration.cjs

# frontend repo root
npm run sync-contracts && npm run typecheck && npm run lint && npm test && npm run build
npx vitest run src/tests/match-correction.test.tsx src/tests/gmail.test.tsx src/tests/applications.test.tsx src/tests/action-queue.test.tsx

# smoke (empty migrated smoke DB; both repos built)
TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs   # backend
SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs                              # frontend
```

The smoke is a regression check; it has no correction step. Browser behavior is checked by hand on the dev database (§13). All results are fixture results, not live evidence.

## 12. Documentation updates

- `docs/architecture/decisions/ADR-0003-correcting-email-matches.md` (first) and a line for it in the docs `README.md` next to ADR-0002 (:13).
- [Domain model](../../domain/domain-model.md): §4.5 (a correction can lower `aiStatus`; `userStatus` untouched), §4.6 implemented history boundary (retired events), §4.7 action rules (retired actions are hidden), §6 (user correction of MATCHED and IGNORED emails).
- [User flows](../../product/user-flows.md): new UF-14 "Correct a wrong email match"; a note in §4.5 action lifecycle.
- [mvp-architecture.md](../../architecture/mvp-architecture.md):66: drop "no rematching" from the limits and describe the correction boundary. `MVP-READINESS-REPORT.md`: a dated note that wrong-match correction is implemented locally.
- Backend `README.md` API table: the new route, `ACTION_RETIRED` and the new message fields.
- `docs/planning/sprint-8/execution-report.md` (create if missing): ADR acceptance date, test counts, lane outputs, mutation results, manual check.

## 13. Definition of done

- [ ] ADR-0003 accepted and committed first; every §10 criterion met with evidence linked.
- [ ] Backend and frontend tests, typecheck, lint (0 errors) and build green on the new PC, and in CI once S7-01 exists. Lanes and smoke pass; both §11.3 mutations seen failing and restored.
- [ ] Behavior checked by hand on the dev database: move, unlink and move back from the Gmail page and the detail page, keyboard only. Live evidence is recorded only from real daily use; fixture results are not called live.
- [ ] §12 docs updated. Focused commits in this order: ADR-0003 (docs), backend, frontend sync and UI, then the §12 doc updates (docs).
- [ ] Evidence recorded in the Sprint 8 execution report. Any defect outside this scope is recorded there and raised with the owner, not fixed here.

## Implementation record — 2026-10-03

Implemented locally under accepted ADR-0003. See [execution evidence](execution-report.md#s8-01--correction-and-retained-history) for tests, mutation checks, preservation lanes, browser behavior and remaining remote/live acceptance.
