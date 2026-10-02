# S6-R02 — Fix the stale-read guard and shared list cache

| Field | Value |
| --- | --- |
| Status | **Fixed — verified locally 2026-10-02** (local ticket) |
| Severity / priority | Medium (+ two Low items) / fix before Sprint 6 acceptance |
| Source | [Review 2026-10-02](README.md) M2, L5; S6-04 §6 |
| Repository | career-companion-frontend |

## Problem
1. `src/lib/applicationCache.ts:33-37`:
   ```ts
   queryFn: async ({ signal }) => preferNewerManualState(
     queryClient.getQueryData(applicationKey(id)),   // evaluated first, before the fetch
     await api.getApplication(id, { signal }))
   ```
   JavaScript evaluates arguments left to right, so the cached value is taken **before** the request. A correction acknowledged while the GET is in flight is never seen, and the guard only covers reads that start after the acknowledgement. `fetchApplicationsPage` does this correctly. Today the StatusEditor's cancellation before the PATCH and before applying the result hides the bug. The test "ignores a read restarted during the mutation" passes because of the cancellation, not the guard.
2. `src/routes/index.tsx:283-285` shares the cache key `['applications', { offset, limit: 20 }]` with the list page but calls `api.listApplications` directly, without `preferNewerManualState`.
3. `applyAcknowledgedApplication` writes caches under `result.id`. `StatusEditor` never checks that it equals the captured `vars.applicationId`.

## Why it matters
S6-04 says stale reads must not overwrite an acknowledged correction and cache writes must stay scoped to the captured resource. The second line of defence is ineffective today. Any read that escapes cancellation (another component, a future change) could repaint older manual state.

## Fix
- Read the cache after the await: `const fetched = await api.getApplication(id, { signal }); return preferNewerManualState(queryClient.getQueryData(applicationKey(id)), fetched);`
- Use `fetchApplicationsPage` in `index.tsx`, or give the dashboard its own key.
- In the mutation, treat `result.id !== vars.applicationId` as a contract error (uncertain outcome → reconcile) and write caches only under `vars.applicationId`.

## Acceptance criteria
- [ ] A test with a detail GET that is **not** cancelled, started before the acknowledgement and resolving after it with a lower revision, keeps the acknowledged manual state.
- [ ] The same protection holds for a dashboard-initiated list fetch.
- [ ] A mismatched response ID never writes another application's cache.

## Verification
Frontend `typecheck`, `lint`, `npx vitest run`, `build`; smoke.

## Resolution — 2026-10-02

Fixed:
- `src/lib/applicationCache.ts`: the cache is read after the response arrives.
- `src/routes/index.tsx`: the dashboard uses `fetchApplicationsPage`.
- `src/components/StatusEditor.tsx`: a PATCH response whose `id` differs from the captured application is an uncertain contract error, so nothing is cached and the editor reconciles.

Verification:
- New tests:
  - an uncancelled detail read resolving after an acknowledgement keeps the newer manual state;
  - dashboard list fetch keeps a cached higher revision;
  - a mismatched response ID enters review and caches nothing.
- Reverting each fix temporarily makes exactly its test fail; files restored byte-for-byte.
- Frontend full suite 90/90, smoke pass.
