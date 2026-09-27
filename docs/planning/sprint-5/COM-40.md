# COM-40: Complete the critical Gmail-to-application browser smoke and fix async invalidation

## Objective
Extend installed Puppeteer harness to run actual local workers with deterministic provider fixtures; test email->application timeline/action, repeat sync, and one failure recovery. Fix frontend presentation defects where application-detail invalidation occurs prematurely.

## Problem / Context
The frontend `gmail.tsx` synchronization/refetch behavior relies on `lastSyncedAt`, which updates when ingestion completes, missing the backend-delayed application-detail invalidations that happen asynchronously during AI processing. Current browser tests also fake completion instead of driving real workers.

## Scope
- Delay query invalidations or implement a bounded refresh tied to terminal processing.
- Provide a browser smoke test using real local workers and encrypted fixture tokens.

## Requirements
- Run real local API/queue/worker/domain behaviors.
- Do not mutate DB sync state to fake completion.

## Acceptance Criteria
- [ ] Real local API/queue/worker smoke running without fake completion writes.
- [ ] Encrypted fixture token/history anchor/nonzero AI budget used in an exclusive smoke DB.
- [ ] Separate ingestion/processing UI functions correctly.
- [ ] Delayed cached-detail refresh works as intended.
- [ ] Repeat idempotency confirmed in the UI.
- [ ] Persisted action/pagination/ownership correctly handled.
- [ ] Provider/API timeout recovery tested.
- [ ] No outbound fixture leakage.
- [ ] Worker-first owner/action/budget cleanup implemented.
- [ ] Tested-build handoff to COM-38 ready.

## Dependencies
- Development after COM-37.
- Final local acceptance blocked by COM-39 and COM-41. (Does not depend on COM-38).

## Relevant Architectural Constraints
- Isolated backend `.env.test` and exclusive smoke DB required.
- No live secrets or ledger bypass in the fixture tests.

## Definition of Done
- Frontend correctly invalidates cache only when AI processing is terminal, or uses a bounded refresh.
- Browser E2E smoke test runs and passes with real local workers against fixture data.
- COM-40 closes on local worker/browser evidence without depending on live proof.
