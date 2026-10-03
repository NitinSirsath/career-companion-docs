# S11-01 — Create personal follow-ups without duplicate work

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — owner acceptance deferred |
| Date | 2026-10-03 |
| ID | S11-01 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | OD-19; Sprint 9; future implementation authorization |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Allow a user to record job-search follow-up work that did not arrive as an extracted email action.

## 2. Why

Some useful next steps are known only to the user; current actions cannot be created or edited through a personal-work flow.

## 3. Current behavior

Action supports PENDING/COMPLETED/DISMISSED, optional emailId and deadline precision. API currently updates status only. No manual-action origin or creation receipt exists.

## 4. Scope

Add nullable action origin/creation receipt/hash and actionRevision as designed; owned POST /api/applications/:id/actions, GET /api/actions/by-request/:clientRequestId for creation reconciliation, and revisioned manual edit route. Lookup returns the current owned action or generic 404 for missing/foreign receipts; register the static path before dynamic action IDs. Support bounded description, optional typed deadline and user completion/dismissal. Preserve existing status-only compatibility for legacy email actions.

## 5. Out of scope

No automatic suggestions, AI drafting, email sends, notification enqueue, email-derived action description/deadline editing, recurring tasks or additional task model.

## 6. Likely files and components

Backend prisma schema/additive migration; contracts/action.ts and application.ts action response; routes/action.ts/application.ts; services/action.ts; matcher.ts revision changes for retire/reactivate; relevant tests. Frontend consumption follows S11-04.

## 7. Implementation notes

origin USER and emailId null distinguish personal work; existing rows remain unmodified and are interpreted conservatively. Require clientRequestId UUID and retain normalized initial creation payload hash. Same owned key/payload returns same id even after edits; different payload=409, foreign key=404. Require expected actionRevision for USER actions. All user changes and retirement/reactivation increment revision; stale compared before no-op. Existing email status-only requests can omit revision for compatibility, while new UI sends it. Manual write locks use existing user boundary then application and action.

## 8. Dependencies

OD-19 decides in-app scope. S11-02 adds snooze on this revision foundation. Sprint 10 agenda remains a distinct entity.

## 9. Security and privacy

Session-derived owner only; bounded text and no raw payload logs. No origin spoofing or foreign application IDs. Existing privacy and ownership guards apply to nullable email sources.

## 10. Acceptance criteria

- [x] Creation writes one owned USER_FOLLOW_UP with no email linkage and no provider/job/send.
- [x] Duplicate same-key creation returns the same id after a simulated lost response, including after an edit.
- [x] Changed payload with the same key conflicts; cross-owner keys/IDs leak no record.
- [x] User edit/status changes require a captured revision; stale edits cannot win or silently rebase.
- [x] Email-origin source text/deadline cannot be overwritten through manual edit.
- [x] Existing actions and creation/status behavior remain compatible after migration.
- [x] Email correction never moves or retires emailId-null personal work.

## 11. Testing

PostgreSQL concurrent-create tests and deliberate lost-response replay with same receipt; ownership and changed-payload conflicts; two-tab stale updates. Legacy action fixtures and S8 correction regression. Run all guarded fresh/upgrade lanes and local contract comparison.

## 12. Documentation updates

Document personal origin, deadline precision, receipt lifetime and revision semantics. Update future architecture with actual choices; no checkboxes ticked before evidence.

## 13. Definition of done

- [x] Creation/edit/idempotency criteria verified.
- [x] Existing action/correction tests and migration preservation pass.
- [x] No provider, notification or historical source change is introduced.

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
