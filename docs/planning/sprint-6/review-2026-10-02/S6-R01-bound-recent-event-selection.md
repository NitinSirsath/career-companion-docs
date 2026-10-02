# S6-R01 — Bound recent-event selection to one row per application

| Field | Value |
| --- | --- |
| Status | **Fixed — verified locally 2026-10-02** (local ticket; Linear later) |
| Severity / priority | Medium / fix before Sprint 6 acceptance |
| Source | [Review 2026-10-02](README.md) M1; S6-02 §6, §14 |
| Repository | career-companion-backend |

## Problem
`src/services/application.ts` (the `enrichment` include, lines ~36-43) loads `recentEvent` with `events: { orderBy: [...], take: 1, select: { ..., email: sourceEmailSelect } }`. Prisma 6.19.3 uses the default relation strategy (no `relationJoins`). Logged SQL inside a rolled-back transaction shows no per-application LIMIT:

```
SELECT "id","type","createdAt","emailId","applicationId" FROM "application_events"
WHERE "applicationId" IN ($1,$2) ORDER BY "createdAt" DESC, "id" DESC OFFSET $3
```

Every event of every application on the list page is loaded, and email metadata is selected for them, before one per application is kept in memory. Detail, create and the status PATCH use the same include. For PATCH this happens inside the transaction holding the application row lock.

## Why it matters
S6-02 requires "at most one recent event per application; avoid a per-event unbounded query pattern" and asks to assert bounded selection. The execution report marks this boundary "Pass", which is wrong. The cost grows with history size, and the query reads more owned email metadata than it needs (none of it is returned).

## Fix
Fetch recent events for the page's application IDs with one bounded query. Either:
- a raw `SELECT DISTINCT ON ("applicationId") ... ORDER BY "applicationId", "createdAt" DESC, id DESC` with an email lookup for at most one event per application, or
- enable `relationJoins` and use `relationLoadStrategy: 'join'`, after confirming it emits a LIMIT 1 lateral subquery.

Keep the single shared mapper and the owner check on the source email. For PATCH, keep the response in the same transaction snapshot.

## Acceptance criteria
- [ ] List, detail, create and PATCH read at most one event (and at most one email) per application.
- [ ] Response shape and ordering are unchanged (latest by `createdAt` desc, `id` desc). Owned-source and foreign-source behaviour is unchanged.
- [ ] A test asserts bounded selection, for example via the query log or a count of rows read, with applications that have many events.

## Verification
Backend `typecheck`, `lint`, `build`; focused `application-evidence.test.ts`, `application-status.test.ts`; full `npx vitest run`; frontend smoke unchanged and passing.

## Resolution — 2026-10-02

Fixed in `src/services/application.ts`. `loadRecentEvents` runs one `unnest(ids) CROSS JOIN LATERAL (... ORDER BY "createdAt" DESC, id DESC LIMIT 1)` query for the page (also used by detail and PATCH, inside the same transaction), then one `email.findMany` for at most one source email per application. The shared mapper and owner check are unchanged; create returns `recentEvent: null` without querying.

Verification:
- New test "reads at most one recent event and one source email per application": 30 + 5 events → 2 rows, 2 email IDs, no `applicationEvent.findMany`, SQL contains `LATERAL … LIMIT 1`, correct latest types.
- Removing `LIMIT 1` makes it (and the latest-event test) fail; restored and checked byte-for-byte.
- `application-evidence.test.ts` 10/10, `application-status.test.ts` 27/27, backend full suite 236/236, smoke pass.
