# Sprint 9 API and display contract

Implemented locally 2026-10-03. Shared runtime schemas are authored in backend `src/contracts/workspace.ts` and `application.ts`, then synchronized into the sibling frontend. No migration or new background worker.

## GET /api/workspace/actions

Session authentication is required. Query parameters:

| Parameter | Contract |
| --- | --- |
| bucket | all (default), overdue, today, later, undated |
| timeZone | IANA zone, default Asia/Kolkata; explicit invalid values return 400 |
| limit | Positive integer, clamped to 20, default 20 |
| offset | Nonnegative integer up to 2147483627, default 0 |

Example request: `/api/workspace/actions?bucket=today&timeZone=Asia%2FKolkata&limit=20&offset=0`.

An empty response is:

```json
{
  "items": [],
  "metadata": { "limit": 20, "offset": 0, "nextOffset": null },
  "generatedAt": "2026-10-03T06:30:00.000Z",
  "timeZone": "Asia/Kolkata",
  "nextTransitionAt": null,
  "counts": { "overdue": 0, "today": 0, "later": 0, "undated": 0, "totalPending": 0 }
}
```

Nonempty items use existing ActionWithContext fields, including application context and bounded source-email metadata. Eligibility is owned application + PENDING action + retiredAt null; CLOSED/REJECTED status does not cancel work. Counts cover all eligible actions before bucket selection and pagination. Counts, ordered IDs, row context and nextTransitionAt use one repeatable-read snapshot.

DATE uses the stored UTC calendar components compared with the selected local date. DATETIME and legacy null precision use instants; overdue is strictly before generatedAt. No deadline is undated. Sorting is overdue/today/later/undated, then deadline (nulls last), createdAt and id ascending. Filtering to a bucket preserves its ordering.

nextTransitionAt considers the entire eligible dataset: future timed/legacy deadlines plus 1 ms and the next local midnight if dated work exists. It is nullable when no time transition exists. The frontend schedules relative to generatedAt with elapsed-response age and a minimum 60-second delay, so a local clock offset does not determine server bucket membership. Counts may lag a transition by that minimum refresh interval. Hidden/unmounted views clear timers; restored visibility/focus refreshes and resets paging.

Offset pages across different requests are not a frozen snapshot. Mutations/time transitions reset the active page; empty later pages return to page one. No cursor stability guarantee.

## GET /api/workspace/review-summary

Returns `generatedAt`, `unmatched`, `ambiguous`, `pendingSubmissions`. All counts are owner-scoped in one repeatable-read snapshot:

- unmatched: RELEVANT emails with UNMATCHED matchState.
- ambiguous: AMBIGUOUS emails, matching the existing queue predicate.
- pendingSubmissions: external submissions with NEEDS_REVIEW matchState.

These categories are not a unique-application total. Read errors remain errors; no zero fallback.

## GET /api/applications extensions

Subsequent pre-migration work adds optional sorting, automation-source discovery and an UNKNOWN query sentinel, with list context retained through detail visits. See the [current application discovery contract](../application-discovery/api-contracts.md). The original S9 behavior and verification below are retained as historical scope.

Optional `q`: trim, maximum 100 characters; empty means no filter. Case-insensitive literal companyName/jobTitle substring search (%, _ and backslash are escaped for PostgreSQL LIKE). Optional `effectiveStatus`: existing status enum. Predicate is userStatus first, otherwise aiStatus; null status does not match a selected enum. Both apply before pagination and owner isolation remains mandatory. Existing createdAt-desc/id-desc ordering and bounded evidence/count enrichment are retained. Repeated/non-string filters and invalid values return 400.

List filters participate in query keys and superseded reads use AbortSignal. They are local view state, not saved/URL searches. Correction pickers keep unfiltered defaults. Acknowledged status updates remove nonmatching cached rows and invalidate pages for server refill. Revision merging that changes filtered membership triggers one bounded reread; persistent disagreement is a recoverable read error. No optimistic insertion into another page.

## Coverage and mutation limits

Existing Gmail and AI settings supply observations, not a processing-complete aggregate. Show disconnected/never synced/in-progress/failed sync, last successful sync, known gap, AI access and PENDING waiting count. Explicitly state that processing completeness is unknown: PROCESSING and FAILED are absent from waitingEmails. A malformed/failed response is unknown. All-empty action results say only “No stored pending actions.” No stale-time threshold, provider promotion or scheduler change is inferred.

Complete/Dismiss reuse PATCH /api/actions/:id. Response parsing and matching action identity are checked; uncertain outcomes refetch related views and display uncertainty without automatic replay. Opening/refreshing these views performs reads only.
