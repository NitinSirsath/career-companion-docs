# S9-01 — Read complete daily action and review summaries

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED LOCALLY — real-use acceptance pending |
| Date | 2026-10-03 |
| ID | S9-01 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | OD-16, OD-17; implementation authorized 2026-10-03 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Provide bounded owner-scoped action pages and complete counts for overdue, today, later and undated work.

## 2. Why

The current dashboard groups the current page; a user cannot infer total workload from it.

## 3. Current behavior

ActionService.getUserActions filters status and retirement, orders by deadline and returns at most one page. Matcher and submission services already define review eligibility. No workspace endpoint exists.

## 4. Scope

Add GET /api/workspace/actions and GET /api/workspace/review-summary with the exact contracts and bucket table in architecture §2. Parse parameters, derive server time once, apply owner/bucket filters before pagination, aggregate in PostgreSQL and return a consistent per-request snapshot. Preserve contextual action/source fields.

## 5. Out of scope

No stored preference, new migration, AI call, sync, job, new mutation route, task lifecycle change or new review-resolution policy.

## 6. Likely files and components

Backend: src/contracts/workspace.ts (new), contracts/index.ts, routes/workspace.ts (new), services/workspace.ts (new), index.ts; reuse services/action.ts and utils/pagination.ts. Frontend contract mirror later via local sync.

## 7. Implementation notes

Use session user identity only. Page maximum remains 20. Counts, page and dataset-wide nextTransitionAt share repeatable-read snapshot and generatedAt. Follow architecture §2 for the next deadline/local-midnight boundary regardless of the selected bucket/page. Each row belongs to exactly one bucket; no deadline is undated, not overdue. Match legacy timestamp semantics explicitly. Review counts reuse existing queue predicates. Parameter errors are 400. Transient failure returns an error, never a zero-filled response. Register static routes before dynamic IDs where applicable. No global status rollup based on only the current page.

## 8. Dependencies

Record OD-17's timezone choice before code. S7-03 precision and S8 retirement are completed prerequisites, not work to redo. S9-02 consumes this contract.

## 9. Security and privacy

Only owner data, bounded source metadata already exposed by existing action contracts, and aggregate counts. Parameterized queries; no action text, subjects or email content in logs. Cross-owner fixtures must not affect counts.

## 10. Acceptance criteria

- [x] A fixture with 47 pending actions spanning all four buckets reports full counts while returning at most 20 rows.
- [x] Retired/completed/dismissed actions and another user's data affect neither items nor counts.
- [x] DATE today remains today until the user's next local date; an earlier DATETIME today is overdue.
- [x] Legacy null precision retains timestamp treatment; null deadlines are undated.
- [x] Filtering occurs before pagination, with deterministic ties and accurate nextOffset.
- [x] All four counts sum to totalPending; review categories follow their existing predicates without pretending to be unique applications.
- [x] Invalid zones/filters/offsets return 400; injected query strings cannot change ownership.
- [x] A timed action outside the selected bucket/page drives nextTransitionAt; local-midnight/DST boundaries are server-derived.
- [x] Reads cause zero Gmail, AI, enqueue or notification calls.

## 11. Testing

Use real PostgreSQL fixtures for date predicates/counts and two users; include Asia/Kolkata, America/Los_Angeles, local midnight and DST transitions with an injected clock. Include 25 overdue records plus later records so a client-page implementation fails. Test concurrent status change against one read snapshot. Provider/queue fakes throw if called. Run contracts and action regression suites.

## 12. Documentation updates

Document request/response examples, time-bucket table, precision fallback and offset-pagination limitations. Update local API docs and record actual implementation evidence when it exists.

## 13. Definition of done

- [x] Every acceptance item has linked evidence.
- [x] Contract parsing, local typecheck/lint/tests/build pass for changed boundaries.
- [x] No migration/provider/scheduler behavior changed.
- [x] Local execution report distinguishes fixture results and pending live acceptance.

Evidence: [execution report](execution-report.md). Checked items denote local engineering evidence or explicitly recorded pending real-use acceptance; no live-user observation is claimed.
