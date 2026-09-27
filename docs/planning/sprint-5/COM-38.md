# COM-38: Prove live Gmail incremental sync and repeat-sync idempotency

## Objective
Introduce one controlled real mailbox change, and drive an authenticated sync (202 -> worker -> Gmail history -> persist -> UI) to prove exactly one row is discovered, checkpoint progresses, and repeats are idempotent.

## Problem / Context
We must prove that the sync architecture works safely for new changes without breaking the established baseline, repeating work on already ingested emails, or causing data loss due to expired history IDs.

## Scope
- Validating the history-path discovery for a single new email.
- Ensuring the new email is persisted without duplicate AI calls or domain copies.
- Verifying the checkpoint progresses safely.

## Requirements
- Use the real mailbox, not a fixture.
- Must not perform bulk state reset or manually write checkpoints.

## Acceptance Criteria
- [ ] Final build already passes COM-40.
- [ ] Controlled real post-anchor change introduced and detected.
- [ ] Actual history worker path executed correctly.
- [ ] Exactly one correct row/metadata added.
- [ ] Correct committed checkpoint.
- [ ] Repeat sync on same row results in no extra completed AI calls/domain copies.
- [ ] Original dataset and user decisions remain intact.
- [ ] Correlated counts/timings recorded.
- [ ] Actual browser processing state observed.
- [ ] Live Gemini success explicitly supplementary (Gmail proof is mandatory).

## Dependencies
- Blocked by COM-37, COM-39, COM-41, and COM-40's local smoke on the final build.
- Real same-mailbox access and protected original dataset required.

## Relevant Architectural Constraints
- Do not reset the DB or fake provider responses.
- The proof must be run against the coordinated final build that has passed COM-40.

## Definition of Done
- A real in-scope Gmail change is proven through history-path discovery.
- Exactly one correct row is added, repeat-sync idempotency is shown, and successful checkpoint progression is recorded.
- No unexplained loss or duplicate effects occur.

## Important Implementation Note
Ensure the `lastHistoryId` anchor is fresh when running the live proof. If significant time passes, the history ID may expire on the provider side, triggering a bounded reconciliation rather than the intended pure incremental history sync.
