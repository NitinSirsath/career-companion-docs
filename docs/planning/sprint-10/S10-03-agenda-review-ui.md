# S10-03 — Review and confirm an interview and assessment agenda

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — live qualification/acceptance deferred |
| Date | 2026-10-03 |
| ID | S10-03 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | S10-02 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Give the owner a trustworthy upcoming agenda and a clear path to resolve uncertain timing.

## 2. Why

Loose extraction text or status badges cannot tell the user when an event actually happens.

## 3. Current behavior

There is no agenda route; application timeline explicitly distinguishes recording time from email/submission time.

## 4. Scope

Add a simple list agenda with upcoming, needs-review, past and history views using S10-02. Show interview/assessment type, application, source reference, original suggestion, precision and timezone. Add revision-frozen confirm/edit/cancel/complete controls. Link from daily workspace and relevant application detail.

## 5. Out of scope

No calendar grid library, external calendar export/sync, automatic merging, auto-notification, meeting-link generation, or alteration of existing Action completion meaning.

## 6. Likely files and components

Frontend: proposed routes/agenda.tsx, api/client.ts, contracts mirror, focused agenda components using existing dialog/button/pagination; routes/index.tsx, application detail and authenticated navigation; query invalidation lists.

## 7. Implementation notes

Keep tentative items out of confirmed upcoming. Label date-only as 'Time not specified'; unresolved timing as needing confirmation. A later reschedule/cancel candidate shows original email evidence and lets the user deliberately cancel old and confirm new through separate saves. Each dialog captures id/revision; conflicts and uncertain saves refetch without replay. Retired items remain in history with reason. Old mail without v3 coverage shows honest empty-state text.

## 8. Dependencies

API and lifecycle from S10-02. Preserve Sprint 9 action view and Sprint 6 history distinction; this is agenda evidence, not a blanket reopening of action-detail source scope.

## 9. Security and privacy

Escape all excerpts and labels. Safe owner-only source links; no remote image/HTML email rendering. Keyboard focus restoration and labelled timezone input required.

## 10. Acceptance criteria

- [x] Confirmed upcoming, tentative review and retired history are visibly distinct.
- [x] DATE never shows an invented time; unresolved timezone never silently uses the browser zone.
- [x] Original AI suggestion remains inspectable after user correction.
- [x] Two-tab stale save shows conflict; lost response triggers reconciliation without duplicate writes.
- [x] Source move/unlink refreshes agenda and application views coherently.
- [x] Reschedule/cancellation review does not claim the old event changed until that explicit mutation succeeds.
- [x] Missing legacy coverage and disabled v3 are explained without implying all historical interviews were parsed.
- [x] Full flow works by keyboard and at 390px width.

## 11. Testing

Component tests for precision, revisions, partial reschedule actions and read failure. Browser smoke with real API/fixture candidates: tentative → confirm → edit → move → unlink → restore; preserve manual status and assert no new provider calls. Test timezone conversion against backend cases.

## 12. Documentation updates

Update agenda user flow and source/uncertainty copy; record accessible screenshots only after implementation.

## 13. Definition of done

- [x] UI acceptance and ownership-safe responses verified.
- [x] Frontend checks and existing action/history/correction flows remain green.
- [x] No live scheduling or provider certification claim from UI fixtures.

Execution evidence and remaining live gates: [execution report](execution-report.md).

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
