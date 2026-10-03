# S10-02 — Persist agenda suggestions with user control and retained history

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — live qualification/acceptance deferred |
| Date | 2026-10-03 |
| ID | S10-02 (local only) |
| Relative size | L; no delivery-date commitment |
| Depends on | S10-01; OD-18 and ADR-0005 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Create an owned, correctable agenda model that preserves original suggestions and user decisions.

## 2. Why

Temporal data must survive refresh and corrections without becoming duplicated or moving stale effects into the active agenda.

## 3. Current behavior

ApplicationEvent records history, Action records obligations, and S8 retires email-derived effects. There is no scheduled-event entity.

## 4. Scope

Add AgendaItem fields/constraints from architecture §3; idempotent projection from persisted candidates; owner-scoped paginated GET /api/agenda and revision-checked PATCH /api/agenda/:id for confirm/edit/cancel/complete. Extend match correction with agenda retirement/reactivation and explicit no-provider behavior.

## 5. Out of scope

No external calendars, automatic cross-email reschedule, generic tasks, old-mail backfill, modification of manual status, or changes to ADR-0003 unlink policy.

## 6. Likely files and components

Backend: prisma schema/additive migration, proposed services/agenda.ts, routes/agenda.ts, contracts/agenda.ts and matcher.ts; ownership constraints consistent with existing migrations; index.ts and contracts/index.ts.

## 7. Implementation notes

All reads filter owner before pagination, max 20. Validate display timezone and view=upcoming/past/review/history. Upcoming/past exclude retired and TENTATIVE rows; use inclusive from/exclusive to calendar-date ranges <=90 days, default next/previous 30 local dates, with DATETIME classified relative to server now and DATE relative to the local date. Upcoming shows CONFIRMED items; past shows CONFIRMED/COMPLETED items. Exact instants sort chronologically within date; DATE entries use a labelled time-unspecified section, stable by ID. Review includes all non-retired TENTATIVE candidates, including unresolved timing, ordered createdAt-desc/id-desc; reject date-range parameters for this view so uncertain dates cannot disappear. History includes all lifecycle states and retired rows, ordered by recording time; its bounded date filter applies to createdAt and is labelled as such. Keep original candidate and separate user fields. Reject confirming unresolved timed facts, foreign ID=404, stale revision=409. Recheck expected revision before no-op. Use user→email→sorted applications→agenda-row lock order; manual writes recheck match/retirement. Existing target user decisions win on reactivation; new target carries source decisions with provenance. Different-email candidates remain separate.

## 8. Dependencies

S10-01 persisted contract. Integrates completed S8 correction as a new effect type, not a correction-policy rewrite. S10-03 consumes API.

## 9. Security and privacy

Derive user identity from auth/worker ownership. Paired retirement constraints and cross-owner FK/trigger protection. Bounded original quote visible only to owner; omit email bodies and opaque provider errors.

## 10. Acceptance criteria

- [x] Repeated projection creates one item per application/email/candidate key.
- [x] User edits never overwrite the original suggestion; stale/uncertain writes cannot silently win.
- [x] A move/unlink retires source agenda effects and leaves userStatus/revision untouched.
- [x] Restore/reactivation preserves prior target confirmations/cancellations and avoids duplicate items.
- [x] Manual edit racing correction serializes safely and cannot mutate a retired source item.
- [x] Different-email reschedule/cancellation remains review work until user action.
- [x] Range/status pagination excludes foreign/retired items and preserves date precision.
- [x] No projection or correction calls Gmail, AI, enqueue or Discord.

## 11. Testing

Real PostgreSQL ownership and observed lock-interleaving tests, duplicate replay and opposite moves. Revision conflicts and lost-response reconciliation. Fresh/upgrade migration preservation with existing actions/events/MCP and v2 results. Retired and unresolved agenda pagination beyond 20 rows.

## 12. Documentation updates

Write API examples, lifecycle/retirement tables, migration and rollback notes. Document that no agenda backfill or provider replay occurs.

## 13. Definition of done

- [x] Data/API and concurrency acceptance verified.
- [x] Migration lanes and existing correction/status/MCP tests pass.
- [x] Additive rollback preserves rows and v3 reconciliation capability.

Execution evidence and remaining live gates: [execution report](execution-report.md).

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
