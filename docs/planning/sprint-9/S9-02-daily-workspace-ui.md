# S9-02 — Show daily work with honest coverage

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED LOCALLY — real-use acceptance pending |
| Date | 2026-10-03 |
| ID | S9-02 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | S9-01; OD-17 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Make the dashboard answer what needs attention now using complete counts and existing action/review controls.

## 2. Why

Work, matching review and input coverage currently require separate interpretation; no actions on a page is not proof that all mail was processed.

## 3. Current behavior

index.tsx has Action Center, unmatched/ambiguous queues and pending submissions. AIAccessNotice and ProcessingRefreshObserver already supply access and refresh behavior.

## 4. Scope

Add overdue/today/later/undated selection and counts to the existing dashboard. Render one bounded selected page, explicit timezone and existing Complete/Dismiss/Gmail/application links. Add review-summary shortcuts to the existing sections. Compose current Gmail/AI state into coverage text without changing their backend semantics.

## 5. Out of scope

No new notification, new matching or retry flow, application-detail action evidence expansion (Sprint 6 D1), calendar, automatic sync, new AI prompt or site-wide redesign.

## 6. Likely files and components

Frontend: routes/index.tsx; a focused workspace component if it simplifies the existing file; api/client.ts; contracts mirror; lib/processingRefresh.ts; components/ProcessingRefreshObserver.tsx; existing ui/button, pagination, GmailLink and AIAccessNotice.

## 7. Implementation notes

Query keys include bucket, zone and pagination. Preserve independent loading/error states. On an uncertain Complete/Dismiss result refetch affected queries and state that the update could not be confirmed; no automatic replay. Existing correction/status invalidations include workspace data. Reset pagination on bucket/zone change or mutation; recover an empty later page. Recompute at the server-provided dataset-wide nextTransitionAt and on focus, with a minimum one-minute timer delay and cleanup. Do not use permanent processing polling. Use the partial-coverage rules in architecture §2, including READY + zero waiting + PROCESSING/FAILED cases, malformed responses and never-synced state. 'No stored pending actions' stays separate from disconnected Gmail, AI waiting, stale/gap-limited sync or failed coverage reads.

## 8. Dependencies

S9-01 API/contracts. Existing S8 recovery and correction controls remain authoritative; no dependency on unfinished Gemini certification for fixture development.

## 9. Security and privacy

Render source text as escaped text; retain safe Gmail links. No keys/provider internals in coverage copy. Keyboard access and focus restoration for existing controls remain functional.

## 10. Acceptance criteria

- [x] Counts still describe all work when the selected page has fewer than 20 rows or no rows.
- [x] Changing bucket/zone resets pagination and never displays another filter's cached rows as current.
- [x] A coverage read failure is visible and does not become an all-caught-up message.
- [x] Disconnected/waiting/gap-limited states direct users to existing Gmail/AI controls.
- [x] Completion/dismissal and email correction refresh counts and application views without duplicate writes.
- [x] Off-page deadlines update counts while Undated is selected; zero waiting never claims processing is complete.
- [x] DATE grouping matches backend data; timezone is visible and boundary refresh works after sleep/focus.
- [x] All actions/filters are keyboard accessible at desktop and 390px width.
- [x] Opening or refreshing the workspace makes no provider or sync mutation request.

## 11. Testing

Component tests for multi-page summaries, failed/partial reads, filter changes with out-of-order responses, lost mutation response, date boundary and timer cleanup. Extend browser smoke through real API with synthetic providers for selecting today, completing work and seeing updated counts; record keyboard interactions, not programmatic DOM clicks.

## 12. Documentation updates

Update dashboard user flow and screenshots after implementation. Explain coverage wording and minimum refresh behavior; retain current styling primitives.

## 13. Definition of done

- [x] Acceptance scenarios and accessibility interactions are recorded.
- [x] Frontend local checks and relevant action/correction/status tests pass.
- [x] Integrated fixture evidence is handed to S9-04; no live claim from component tests.

Evidence: [execution report](execution-report.md). Checked items denote local engineering evidence or explicitly recorded pending real-use acceptance; no live-user observation is claimed.
