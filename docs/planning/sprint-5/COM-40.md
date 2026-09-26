# COM-40 — Complete the critical Gmail-to-application browser smoke

Status: proposed local ticket; no new E2E run performed. Primary repository: frontend, with backend fixture/worker support. Suggested priority: High. Estimate: 5 points.

## 1. Title

Complete the critical Gmail-to-application browser smoke with actual local workers.

## 2. Goal

Build the smallest repeatable browser check that drives Gmail sync acceptance through actual local queue/worker/persistence/matching and verifies the user's email, application timeline and one action. Cover one failure/recovery path and async UI timing; keep real Gmail proof separately mandatory in COM-38.

## 3. Context

Stabilization shipped an existing Puppeteer harness and UI tests for pagination and async sync. The harness currently disables workers, queues a Gmail job and directly sets `syncStatus=IDLE/lastSyncedAt` in Prisma. It validates polling presentation, not the critical processing path. The brief allows Playwright/browser verification; Puppeteer is already installed, so a second browser framework is unnecessary.

## 4. Current implementation

- `frontend/scripts/smoke-stabilization.mjs`: built UI + real Express/Prisma/pg-boss, fixture users, blocked external browser traffic, cleanup and Chrome configuration.
- `frontend/package.json`: `puppeteer` dependency and build/test/typecheck/lint scripts; no Playwright dependency or E2E script.
- `frontend/src/routes/gmail.tsx:37–76`: two-second status/current-page processing polling; domain invalidation on lastSyncedAt.
- `frontend/src/api/client.ts`: credentialed fetch, 401 redirect and safe API error message; no explicit fetch deadline.
- `frontend/src/routes/applications.tsx`, `src/routes/applications.$id.tsx`, `src/routes/index.tsx`: application creation/detail, timeline/actions, unmatched/ambiguous matching and paginated selectors.
- `backend/src/index.ts`, `src/jobs/{gmailSyncJob,emailProcessingJob}.ts`, `src/services/{gmailSync,matcher}.ts`: runtime path to execute. Backend test DB guard already exists.

## 5. Problem / gap

The current script can pass with a broken Gmail or email worker. It sets `AI_DAILY_CALL_LIMIT=0`, stores a non-encrypted placeholder token, has no history anchor, and deletes queue rows before stopping workers; those choices are unsuitable once real workers run. Notification jobs carry `actionId`, so its current `data->>'userId'` cleanup misses jobs created by real matching. Ingestion finishes before AI matching; invalidation at ingestion completion can fetch stale domain state, and application detail has no polling. A stalled API fetch has no app-defined deadline. Tests must reproduce these user-visible gaps before adding targeted behavior. OAuth/provider/UI mocks are not equivalent to full live verification.

## 6. Proposed implementation

Extend `scripts/smoke-stabilization.mjs` rather than adding a competing harness/framework:

1. Keep the explicit local test DB guard before all setup or cleanup. Run exclusively on a dedicated smoke database containing only this harness's fixtures; do not share it with concurrent Vitest runs or arbitrary pre-existing queued jobs. Ordinary workers consume a queue across users, not just the fixture user. Use unique synthetic owners/mailboxes and fixture-only secrets; no real Gmail/Gemini/Discord credentials. Set environment before importing backend. Starting selected Gmail/email workers explicitly in this test is allowed even though `NODE_ENV=test` disables automatic startup. Before starting them, encrypt a fake token with the real `encryptToken` helper and fixture key, set a valid fixture `lastHistoryId`, and make the provider fixture assert `history.list` is used. Replace the zero AI ceiling with a small fixture-only allowance sufficient for one classification and one extraction (plus only explicitly tested retries). Ensure the global UTC-day budget has no leftover cooldown/exhaustion in this exclusively owned test DB, and clear only that harness-owned budget state after workers stop. Do not bypass `runOperation` to escape its budget check.
2. Install deterministic test-only provider substitutes at Google/Gemini adapter boundaries in the harness process (or a narrow test-only helper if needed). Use real API routes, queue dispatch, workers, DB, AI operation ledger and matcher. The fixture Gmail history contains one new eligible message; the fixture Gemini response maps it to exactly one existing synthetic application with one requested action. Do not mock API responses for the positive end-to-end path, directly call `syncUser`, or manually write completion/result/domain rows after triggering sync.
3. Keep notification sending disabled and block unexpected outbound provider traffic from the backend fixture process as well as the browser, while allowing only the verified local PostgreSQL connection and harness HTTP origin. Stubbed provider boundaries must fail closed if an unexpected request escapes. Do not introduce a production `MOCK_GMAIL` flag or test-control endpoint.
4. Positive smoke: sign in with the existing local fixture boundary, visit a known application to populate cache, visit Gmail, click Sync Now, capture 202 and disabled Syncing state, observe actual worker-driven IDLE and the exact new email, then navigate to the application while AI processing is deliberately delayed. Wait for the resulting timeline/action/state in the mounted page, complete one action and verify persistence. Repeat sync and assert no duplicate row/effect or extra completed-operation fixture call.
5. Preserve compact pagination checks: 26 fixture emails/applications, first page capped at 20, second page reachable, direct older application detail, one timeline next-page assertion. This proves the new email is not inferred solely from first-page count. No need to duplicate every unit pagination case in the browser.
6. Recovery smoke: inject one controlled transient Gmail failure through the fixture adapter, observe failure/retry status, recover via actual job retry or normal Sync Now after the claim is eligible, and show completion. Separately exercise a status/POST network timeout and revoked-status UI with targeted UI tests or a small browser subcase; these are simulated failures, clearly labeled.
7. Fix only reproduced async presentation defects. When observed email processing transitions to a terminal state, invalidate related application/timeline/action queries. For an already-mounted detail entered while the controlled job is still processing, use bounded refresh tied to a recent sync/known pending work; stop on the observed terminal condition, navigation/unmount or a documented 60s observation cap. If still pending at the cap, show an incomplete/retry state rather than claiming matching finished. No global permanent polling or progress percentage.
8. Add explicit bounded cancellation to Gmail sync/status requests in `src/api/client.ts` (proposed client deadline: 15s), with a recoverable error. A POST timeout is an unknown acceptance result: refetch status before allowing a new request, because the server may already have queued it. A status polling timeout stops the uncontrolled loop and permits a deliberate retry. Keep 401 session handling and 409 state refresh.

Use event/response/DB predicates and bounded waits, not fixed sleeps. Save safe logs/screenshots on failure; the harness must exit nonzero and clean only its fixture IDs. Teardown first stops request intake and worker fetching, drains/aborts active handlers and confirms they cannot write, then removes fixture jobs/domain rows, then closes database/server/browser resources. Preserve fixture action IDs before cascading user deletion: cancel/remove Gmail and email jobs by fixture owner and notification jobs by those action IDs. Use the installed queue API or a verified still-open DB connection after worker stop; do not restart workers to clean up. Queue initialization/maintenance must not continue recreating fixture work. The COM-38 real-mailbox run follows this local gate and remains separate evidence.

## 7. Architecture impact

Test harness now traverses real local backend boundaries instead of simulating sync completion. Runtime changes are limited to evidenced Gmail request timeouts and query refresh/invalidation. Reuse current components/query keys/design tokens; no layout redesign, E2E infrastructure service or new backend product endpoint.

## 8. Data impact

No schema, indexes or migration changes. Synthetic fixture owners, applications, emails, AI results/operations, actions/events, the fixture daily AI budget and queue rows exist only in the exclusive guarded smoke DB. Cleanup includes owner-scoped Gmail/email jobs and action-scoped notification jobs, after active handlers stop. Never point this script at the original dataset or reset shared AI budget state. Positive-path assertions verify actual unique email and domain rows rather than creating expected output rows manually.

## 9. API impact

No response-contract change. Observe `POST /api/gmail/sync` 202; status/messages 200; 400 disconnected; 401 session expired; 409 in progress; safe 500 queue failure. Verify owned application detail 200/404, events/actions 200 paginated or 403 unavailable, and `PATCH /api/actions/:id { status: "COMPLETED" }` 200. Existing email resolve endpoint can cover unmatched behavior with unit/integration tests; it need not become another broad browser flow. All list requests retain `{ items, metadata: { limit, offset, nextOffset } }`, default/max 20, no total-count assumption. Timeout is a client failure state, not a fabricated server status code.

## 10. Background-job impact

Run actual Gmail/email worker registrations in the fixture harness; use normal payloads, claims, queues and completion rules. Count fixture AI adapter calls to establish reuse. No scheduler, altered production retry limit or direct queue-row completion. Do not start a live notification worker with credentials. Stop workers and close DB/server/browser even on failed assertions. Queue completion alone is insufficient; assert domain rows and UI.

## 11. Security/privacy considerations

The local smoke may use `X-Development-User` only with non-production fixture configuration; it does not validate Google sign-in. COM-38 validates the real session/provider lane. Browser screenshots contain synthetic content only. A second synthetic user must not read the first user's email/detail or mutate its action; preserve backend ownership regressions and one browser/API denial assertion. Never expose fixture adapters or debug endpoints in production builds.

## 12. Testing strategy

- Unit/UI: add delayed processing/cache invalidation, request abort, ambiguous POST timeout, revoked connection, polling failure/retry and terminal stop cases to current Gmail/stabilization UI suites.
- DB/integration: rely on COM-39 for exhaustive retry/claim/ownership faults; retain backend API tests for contract status codes and pagination.
- Provider: deterministic adapter fixtures in the local smoke; zero live-provider evidence claimed from them.
- Browser E2E: two compact flows above, built frontend plus real API/queue/workers/DB, semantic button/text predicates, no uncaught browser errors or unhandled promise failures.
- Live: hand the tested build/evidence to COM-38 for its later separate mailbox/session/UI run. COM-40 does not wait on COM-38 to close. Do not replace live proof with CI fixtures or automate Google consent through the local harness.

## 13. Acceptance criteria

- [ ] A clean isolated run drives the normal 202 endpoint through actual Gmail/email workers to one persisted email and one expected timeline/action without direct completion writes.
- [ ] The browser shows Syncing until ingestion terminates and distinguishes pending processing from finished application state.
- [ ] A previously cached, mounted application view updates after delayed processing without a full-page reload; refresh is bounded and stops as documented.
- [ ] Repeat sync leaves the same email ID and unique domain effects and makes no extra completed-operation fixture AI calls.
- [ ] One action can be completed and its status survives reload; owner isolation and one older-page/detail case pass.
- [ ] A provider-failure/retry path and request timeout/recovery are visible; ambiguous POST timeout checks status before resubmission.
- [ ] No unexpected outbound fixture-process/browser request occurs; no live credentials or baseline rows are used/deleted.
- [ ] Harness uses encrypted fake tokens, a history anchor and a nonzero fixture AI allowance through the real operation ledger; it cannot consume another run's queue or budget state.
- [ ] Harness failure produces bounded nonzero exit plus sanitized evidence and fixture-only cleanup. Workers stop before cleanup; Gmail/email and action-scoped notification jobs plus fixture budget state are removed without later handler writes.
- [ ] Local smoke evidence is explicitly marked provider-simulated and records the coordinated build handed to COM-38. Live evidence remains a later sprint gate, not a COM-40 closure dependency.

## 14. Dependencies

COM-37 establishes environment separation. COM-39's final lifecycle and COM-41 diagnostics are required for completed smoke acceptance. Harness development can proceed earlier. COM-38 depends on this local gate, not the reverse. Requires sibling repo layout, both builds, ignored backend `.env.test`, installed Chrome or `CHROME_BIN`, Node dependencies and an exclusively used dedicated smoke PostgreSQL database. No Playwright installation is needed.

## 15. Non-goals

No large E2E suite, visual regression platform, browser matrix, automatic real-account OAuth, replacement design system, global dashboard polling or new job-history UI. Do not move all integration scenarios into slow browser tests.

## 16. Failure/recovery scenarios

Browser deadline: inspect correlated job state; fail test rather than infinitely waiting. POST timeout: acceptance may have occurred; status read determines next action. Provider error: verify bounded retry/failure and recovered UI. AI pending beyond UI cap: show incomplete state, preserve work and allow explicit refresh. Fixture crash: teardown stops processes and removes only owned fixtures; never broad truncate/reset. Real live test blocked: retain local evidence and keep provider gate open.

## 17. Verification commands

From backend on the guarded test DB: `npm run build` and the COM-39 regression commands. From frontend: `npm test -- src/tests/gmail.test.tsx src/tests/stabilization-ui.test.tsx src/tests/pagination-ux.test.tsx`; `npm run typecheck`; `npm run lint`; `npm test`; `npm run build`; then the existing **`node scripts/smoke-stabilization.mjs`**. The backend must also be built first. `CHROME_BIN` overrides the script's default macOS Chrome path. There is no existing `npm run e2e` or `playwright test` command; do not claim either passed.

## 18. Documentation updates

Update frontend `README.md` with this exact smoke command, prerequisites, isolation and simulated-provider limits. Update backend `STABILIZATION.md` to replace its current worker-disabled/direct-completion smoke description only after the new harness exists. Add final smoke evidence beside this sprint pack and link COM-38 real-provider evidence without conflating them.

## 19. Engineering notes

Puppeteer satisfies the requested browser smoke and avoids duplicating tooling. A fixture AI response is appropriate to make domain behavior deterministic; keep real API/queue/worker/persistence boundaries intact. Test delayed processing intentionally because ordinary fast fixtures can hide cache bugs. Keep refresh work scoped to recent sync/observed processing and preserve 20-item pagination rather than repeatedly loading every historical email.
