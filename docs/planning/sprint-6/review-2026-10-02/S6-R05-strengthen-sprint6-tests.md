# S6-R05 — Strengthen Sprint 6 tests that don't prove their claim

| Field | Value |
| --- | --- |
| Status | **Fixed — verified locally 2026-10-02** (local ticket; tests only) |
| Severity / priority | Medium (+ Low items) / fix before Sprint 6 acceptance |
| Source | [Review 2026-10-02](README.md) M5, L4; S6-01 §14, S6-04 §14 |
| Repository | career-companion-frontend, career-companion-backend |

## Problem
- **`retry: false` unproven.** TanStack Query v5 mutations default to no retry, and the test clients only configure `queries.retry`. The "never retries" assertions in `status-correction.test.tsx` would still pass if `retry: false` were removed from `StatusEditor.tsx:83` or `applications.tsx:113`. S6-04 §14 asks to verify `retry=false` directly.
- **"Unknown blocks Save" can't isolate its cause.** In `status-correction.test.tsx` (~242-245), the failed reconciliation also puts the detail query in error, so Save is disabled by `canEdit` alone. Removing `'unknown'` from `blocked` would not fail the test.
- **No action-section failure test.** S6-04 requires actions to fail locally too; only history is tested.
- **Backend no-side-effect test is incomplete.** `application-status.test.ts` ("makes no provider call, job, …") never asserts the mocked `enqueueNotificationJob`, Gemini or `enqueueEmailProcessingJob` weren't called. Its job count is trivially 0 if the `pgboss` schema doesn't exist.
- **Foreign source untested for `recentEvent`.** The mapper's foreign-source path is tested for events only.
- **Misleading test name.** `email-worker-reliability.test.ts` "completes and leaves no error fields" never checks the error fields.

## Fix
- Build test QueryClients with `mutations: { retry: 3, retryDelay: 0 }`, assert exactly one PATCH/POST after an uncertain error, and make the "unknown" assertion independent of query error state.
- Add an actions-failure isolation test.
- Assert the backend mocks were not called and ensure the queue schema exists before counting.
- Add a foreign-source `recentEvent` test (mapper-level mock).
- Seed error fields and assert they are cleared on completion.

## Acceptance criteria
- [ ] Each listed test fails when the behaviour it names is removed. Check this by temporarily reverting the guarded line locally, without committing.
- [ ] Full suites pass.

## Verification
Backend and frontend `npx vitest run`, `typecheck`, `lint`.

## Resolution — 2026-10-02

Fixed (tests only):
- **Frontend `status-correction.test.tsx`:**
  - QueryClients default to `mutations: { retry: 3, retryDelay: 0 }`, so "called once" proves `retry: false` for PATCH and create.
  - New test: the unknown state keeps Save blocked even after a later read succeeds.
  - New test: actions-section failure is isolated and retryable.
- **Backend `application-status.test.ts`:** the side-effect test now runs with the queue schema present and asserts no `getQueue`, `enqueueNotificationJob`, Gmail fetcher or Gemini call.
- **Backend `application-evidence.test.ts`:** foreign `recentEvent` source test.
- **Backend `email-worker-reliability.test.ts`:** the completion test uses the real pipeline with a completed AI result and asserts every error field is cleared, with no Gemini call.

Verification (each guarded line temporarily removed, test confirmed failing, file restored byte-for-byte):
- `retry: false` in the editor → 9 failures; in create → 2.
- `'unknown'` in `blocked` → 1.
- Queue call added to the correction → 1.
- Error-field clearing removed → 1.

Full suites: backend 236/236, frontend 90/90.
