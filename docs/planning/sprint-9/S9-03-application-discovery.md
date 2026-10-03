# S9-03 — Find applications by name and effective status

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED LOCALLY — real-use acceptance pending |
| Date | 2026-10-03 |
| ID | S9-03 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | OD-16; implementation authorized 2026-10-03 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Let the owner find an application by company/job title and filter by the status they actually see.

## 2. Why

Pagination-only browsing becomes cumbersome as the application list grows.

## 3. Current behavior

GET /api/applications only reads limit/offset. ApplicationService orders by createdAt/id. deriveStatus and applicationCache already preserve manual overrides and revision ordering.

## 4. Scope

Add optional q and effectiveStatus filters to the existing route, database query and frontend list. Search literal case-insensitive company/title substrings, limit q to 100 trimmed characters, preserve stable ordering and all existing derived response fields. Reset pagination when filters change.

## 5. Out of scope

No fuzzy matching, matcher normalization, new indexes without query evidence, archived state, analytics, bulk editing, saved searches or third-party search service.

## 6. Likely files and components

Backend: routes/application.ts, services/application.ts, contracts/application.ts if query schema is shared. Frontend: routes/applications.tsx, api/client.ts, lib/applicationCache.ts, relevant tests.

## 7. Implementation notes

Filter before limit/offset. Effective status is userStatus ?? aiStatus; reproduce the canonical expression in a parameterized database predicate, with contract tests covering drift. Escape LIKE wildcards for literal search. Scope first to the authenticated owner. Follow architecture §2 for filter-aware acknowledged cache updates, one bounded reread on membership-changing late results, failed-refetch handling and empty-page recovery. Every filter participates in query keys and the revision-guarded fetch path. Keep unfiltered defaults compatible for correction pickers; do not silently apply the list's search state to another consumer. Abort or ignore superseded reads and retain the user's current filter on errors.

## 8. Dependencies

Can be implemented independently of S9-01 after authorization; integrated with S9-02 under S9-04. No dependency on the agenda or provider evaluation.

## 9. Security and privacy

No email-body search and no raw search query logging. Another user's company names must never influence results/counts. Bound all inputs and SQL parameters.

## 10. Acceptance criteria

- [x] A matching application beyond the original first 20 records appears on the first filtered page.
- [x] Company/title search is case insensitive and treats wildcard characters as literal text.
- [x] Manual REJECTED over AI INTERVIEW matches REJECTED and not INTERVIEW; clearing the override updates the filter result.
- [x] Empty q preserves existing list behavior; unknown status/overlong q are rejected.
- [x] Filter changes reset offset and cannot reuse stale results from an earlier filter.
- [x] Acknowledged status updates immediately remove nonmatching cached rows even when refetch fails; late reads cannot reintroduce them, and empty later pages recover.
- [x] Existing correction picker, manual creation and detail navigation remain usable.
- [x] Foreign applications and retired-effect counts remain excluded under existing rules.

## 11. Testing

Backend tests with >20 applications and contradictory manual/AI statuses. Frontend tests for rapid query changes, two tabs with status update, empty/error and pagination. Smoke search a known fixture company, open its detail, correct status and verify filtered membership. Verify existing recent-event bounds remain intact.

## 12. Documentation updates

Document filter semantics and URL/query behavior; add user-flow examples using company/title and effective status.

## 13. Definition of done

- [x] Acceptance evidence exists; local API/UI checks pass.
- [x] Existing application/status/correction suites pass.
- [x] No matching, ingestion, status precedence or user data was rewritten.

Evidence: [execution report](execution-report.md). Checked items denote local engineering evidence or explicitly recorded pending real-use acceptance; no live-user observation is claimed.
