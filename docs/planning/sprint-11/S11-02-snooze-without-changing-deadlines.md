# S11-02 — Defer work in-app while keeping the real deadline

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — owner acceptance deferred |
| Date | 2026-10-03 |
| ID | S11-02 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | S11-01; OD-19 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Let the user temporarily hide pending work without changing when it is actually due.

## 2. Why

Completing/dismissing an action to reduce clutter incorrectly records the user's intent when they merely want to revisit it.

## 3. Current behavior

Action has a deadline and status but no snooze state. Workspace bucket semantics were introduced in Sprint 9.

## 4. Scope

Add nullable snoozedUntil and revision-checked snooze/unsnooze operations for active PENDING actions. Add a separate snoozed page/count and derive wake eligibility at read time. Display original due date throughout.

## 5. Out of scope

No background wake job, outbound reminder schedule, deadline edit, recurrence, automatic completion or notification resend.

## 6. Likely files and components

Backend schema/additive migration; action service/routes/contracts; workspace query predicates/counts; matcher carry-forward; existing notification eligibility. Frontend implementation in S11-04.

## 7. Implementation notes

Accept explicit future instant up to 365 days from server time, or null to unsnooze; UI displays chosen timezone and resolves DST explicitly. Snooze preserves status and deadline. At expiry, return to its original overdue/today/later/undated bucket; never rewrite the deadline. Complete/dismiss clears snooze. Total pending partitions into visible buckets plus snoozed. Retired actions reject changes. Corrections preserve target decisions or carry source snooze to a new target as defined in architecture. Suppress existing pending notification eligibility while snoozed, with no automatic send on wake.

## 8. Dependencies

S11-01 actionRevision and OD-19. Workspace refresh includes next visible snooze expiry under existing bounded timer rules.

## 9. Security and privacy

No text or identifiers from another user; auth owner check and expected revision on every new route. Store instant, not a host-dependent local string.

## 10. Acceptance criteria

- [x] Snooze preserves actual deadline and PENDING status while moving the row out of normal buckets.
- [x] At expiry, read-time behavior restores the correct bucket without any job or notification.
- [x] Visible bucket counts plus snoozed equals active pending total, across more than one page.
- [x] Invalid/past/out-of-bound snooze times are rejected; DST ambiguity is not guessed.
- [x] Completion clears snooze; stale/retired/foreign mutation is rejected.
- [x] Move/unlink/restore retains decisions without reopening handled work.
- [x] In-flight notification limits are documented; wake never replays a delivered or suppressed notification.

## 11. Testing

Injected clock tests around expiry/midnight, real DB pagination/counts and correction interleavings. Two-tab revision conflict. Fake queue/provider must observe zero wake/send. Extend existing notification eligibility tests without inventing exactly-once guarantees.

## 12. Documentation updates

Document snooze versus deadline, in-app-only behavior, notification limits and count equations; update API examples.

## 13. Definition of done

- [x] Acceptance and preservation tests pass with actual evidence.
- [x] No worker/scheduler/external reminder added.
- [x] Original due dates remain unchanged in DB and UI.

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
