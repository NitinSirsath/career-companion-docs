# S11-04 — Deliver and verify personal follow-through controls

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — owner acceptance deferred |
| Date | 2026-10-03 |
| ID | S11-04 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | S11-01, S11-02, S11-03 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Make follow-ups, snooze and archive usable and verify their combined behavior through real local APIs.

## 2. Why

Personal control only helps if provenance, due dates, stale saves and restoration are clear to the user.

## 3. Current behavior

Existing dialogs/buttons/status editor and uncertain-save patterns can be reused. No follow-up, snooze or archive controls exist.

## 4. Scope

Add personal follow-up creation/editing, snooze/unsnooze, archived list and archive/restore controls. Label user-created work separately from email-derived work. Reuse existing styling/focus patterns; extend local smoke and record implementation and trial evidence.

## 5. Out of scope

No generated outreach, provider calls, reminder delivery, full notes editor, project/task taxonomy or unrelated UI rewrite.

## 6. Likely files and components

Frontend application detail/list and workspace routes; focused follow-up/snooze/archive dialogs, api client/contracts/query keys. Existing smoke-stabilization.mjs and relevant backend/frontend regression suites.

## 7. Implementation notes

Create a UUID per deliberate creation draft; keep it across reconciliation and never auto-resubmit after timeout. Capture row revision on open; 409 refreshes without rebasing the draft silently. After uncertain create, use an owned receipt lookup GET /api/actions/by-request/:clientRequestId defined in S11-01, before deciding to submit again. Zero results do not prove the earlier request failed while still in flight. Invalidate workspace, application, agenda and archive views after settled writes. Show original deadline on snoozed cards, archived read-only context with restore, and no promise of an external reminder.

## 8. Dependencies

All backend commands, including S11-01's owned receipt lookup, are implemented and local contracts synchronized before this UI ticket starts. These endpoints are proposed future scope and do not exist on the completed Sprint 8 baseline.

## 9. Security and privacy

Escaped user text, no body/key logging, keyboard-only dialog interactions and safe source links. Fixture-only screenshots in persisted reports.

## 10. Acceptance criteria

- [x] User can create/edit/complete a personal follow-up without a Gmail/AI/send request.
- [x] Lost creation response is reconciled by receipt with no duplicate action or misleading success.
- [x] Snoozed work shows the original due date, supports unsnooze and returns after expiry.
- [x] Archive/restore explains visibility effects and preserves owned history, status and receipt identity.
- [x] Conflict/retired/archived errors preserve draft intent and require deliberate retry.
- [x] Keyboard and 390px journeys have visible controls and restored focus.
- [x] Integrated smoke covers follow-up→snooze→archive→restore plus email correction, with zero fixture residue.
- [x] Local checks, migration lanes and separate user-trial status are recorded.

## 11. Testing

Component tests for two tabs, duplicate click, unknown response and incompatible stale state. Real API/PostgreSQL/browser fixture smoke with outbound blocked. Full backend/frontend checks, local contract comparison and all applicable preservation lanes. Record live task observations if available, otherwise pending.

## 12. Documentation updates

Update follow-through user flow, origin/coverage copy and sprint execution report with actual results and limitations; keep real-world acceptance register truthful.

## 13. Definition of done

- [x] Integrated functionality and accessibility verified.
- [x] Fixture and live evidence clearly separated.
- [x] Application code change scope matches selected tickets; no remote workflow or notification expansion.

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
