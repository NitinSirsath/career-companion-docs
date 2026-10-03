# S11-03 — Archive and restore applications without losing evidence

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — owner acceptance deferred |
| Date | 2026-10-03 |
| ID | S11-03 (local only) |
| Relative size | L; no delivery-date commitment |
| Depends on | OD-20; S10 agenda boundary; S11-01/02 integration |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Let the owner remove applications from the daily working view and restore them without deleting or corrupting their history.

## 2. Why

CLOSED is an application status override. It is not an archive choice and does not remove clutter consistently.

## 3. Current behavior

Application has no archive fields; matcher candidates include all owned applications, and MCP receipts retain application identity. S8 effects already use retirement for email-link correction.

## 4. Scope

Add archivedAt/archiveRevision and owned expected-revision archive/restore command. Add active/archived/all list filters and suppression across workspace actions, agenda and notifications. Keep archived details/evidence accessible. Extend matcher/MCP behavior exactly as architecture §4 specifies.

## 5. Out of scope

No deletion, merge, retention cleanup, automatic archival, bulk operation, status override, new MCP tool or auto-unarchive.

## 6. Likely files and components

Backend schema/migration; application/action/workspace/agenda read services; matcher.ts; externalSubmission.ts; notificationJob.ts; contracts and routes. Frontend consumes fields/controls in S11-04.

## 7. Implementation notes

Archive does not retire effects; visibility is a separate predicate. Archive takes the emailMatches user advisory lock then application row lock. MCP retains its distinct externalSubmissions namespace and rechecks archive after the shared application row lock; do not take both advisory namespaces. A stale candidate must become NEEDS_REVIEW rather than falling through to duplicate creation. Recheck archive under lock for manual writes. Existing user/thread email links continue ingesting while remaining archived; new company/role candidates exclude archived apps. A new MCP submission matching only an archived app becomes reviewable rather than a duplicate application; replayed receipts keep original identity. Explicit new links and personal/agenda edits require restore. Restore resurfaces pending work without replaying notifications. Suppress eligible sends before claim and recheck before send, but do not promise to cancel already in-flight deliveries.

## 8. Dependencies

OD-20 must cover linked-mail behavior and the proposed ADR-0002 intake extension. Existing manual-status revision remains independent. If archive is moved earlier than S10, explicitly amend its agenda dependency.

## 9. Security and privacy

Ownership on every direct detail/archive request; missing/foreign=404 on new commands. Additive columns with no deletion/backfill. Safe structured logs only.

## 10. Acceptance criteria

- [x] Archive removes only that application from default lists/workspace/agenda and preserves explicit archived detail access.
- [x] Restore preserves status/revision, actions, confirmations, events, submissions and email links.
- [x] New linked thread mail remains attached while archived; unlink policy still wins over automatic fallback.
- [x] Company/role candidate selection never silently reactivates an archived application.
- [x] Existing MCP receipt replay returns its original result; new archived-only match is reviewable without duplicate app creation.
- [x] Explicit archived-target writes return restore-required conflict; stale archiveRevision returns 409.
- [x] New eligible notifications are suppressed and restore does not resend old work.
- [x] Migration preserves all existing application/receipt/effect data.

## 11. Testing

Observed lock interleavings for archive vs correction, status and MCP. Fixture mail into archived thread; explicit unlink followed by later mail; archived-only MCP candidate and replay. Real DB filtered pagination/ownership, notification fake with pre-send check and documented in-flight boundary. All guarded migration lanes.

## 12. Documentation updates

Record archive/matching/intake policy approval, response codes, in-flight limits, restoration steps and rollback restrictions; update ADR-0002 only when the extension is approved.

## 13. Definition of done

- [x] All ownership/intake/preservation scenarios pass.
- [x] No deletion or reinterpretation of user status occurred.
- [x] Local evidence records actual notification limits and remaining live acceptance.

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
