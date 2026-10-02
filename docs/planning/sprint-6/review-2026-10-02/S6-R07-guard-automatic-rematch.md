# S6-R07 — Guard automatic re-matching of already matched emails

> **Scheduled (2026-10-02):** Sprint 6 closeout, order 3 of 8 ([closeout plan](../closeout/README.md)). Still open in code: `matcher.ts` has not changed since the review. MCP's automatic application creation makes the trigger slightly more likely. It is a prerequisite for [S8-01](../../sprint-8/S8-01-correct-a-wrong-email-match.md). The fix and acceptance criteria below are unchanged.

| Field | Value |
| --- | --- |
| Status | Open — not started (local ticket) |
| Severity / priority | Low / follow-up (mostly pre-existing behaviour) |
| Source | [Review 2026-10-02](README.md) L1; S6-03 §6 "ambiguous-branch updates", "effects never move" |
| Repository | career-companion-backend (`src/services/matcher.ts`) |

## Problem
- **Ambiguous branch:** its `updateMany` sets `matchState: AMBIGUOUS` with only a `matchConfirmedBy != USER_CONFIRMED` condition. It has no `matchState` condition.
- **`applyMatch` with `AI_AUTO`:** it refuses only `USER_CONFIRMED` emails.
- **Effect:** if matching re-runs on an email already auto-matched to application A, and the candidates have changed, then:
  - the email can become AMBIGUOUS while `applicationId = A`, with A's event and action kept; a later user link then moves it to B, so one email has effects on two applications; or
  - automatic matching can re-link the email to B.
- **How it can re-run:** duplicate processing, or a retry after matching succeeded but before COMPLETED was written.
- **Example change:** the user adds a second application for the same company, or a thread match appears.

## Fix
Add `matchState: { in: ['UNMATCHED', 'AMBIGUOUS'] }` to the ambiguous update. In `applyMatch` for `AI_AUTO`, return without changes when the email is already `MATCHED` to a different application. Matching policy is otherwise unchanged. If re-matching should ever be allowed, that needs a separate product decision.

## Acceptance criteria
- [ ] Deterministic interleaving tests: re-matching a matched email never changes its application or state and never adds effects to another application.
- [ ] Existing matcher, stabilization and matching-concurrency tests still pass.

## Verification
`matching-concurrency.test.ts`, `matcher.test.ts`, full backend suite.

## Closeout amendment (2026-10-02)

The sections above stay as written. This section adds what the closeout needs. Line numbers were checked on 2026-10-02 against the backend folder. Re-check them before editing, and find code by symbol name.

### Dependencies

- [Phase 0 gate](../../migration-verification/README.md#mv-16--gate-record-results-and-decisions) passed (MV-16): git restored (needed for the mutation checks below) and the test database migrated. After AI-19 it has 16 migrations, AI-19's `<timestamp>_ai_access_paused` the newest. This ticket adds none.
- Closeout order 3 of 8, after [AI-19](../../byo-ai/AI-19-runtime-safety-fixes.md) and [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md). Neither touches `matcher.ts` or the matcher tests. No owner decision (OD-nn) is needed.
- Blocks [S8-01](../../sprint-8/S8-01-correct-a-wrong-email-match.md). It relies on this guard, so a worker replay cannot undo a correction, and it adds its own cases to `matching-concurrency.test.ts`.
- [S6-C01](../closeout/S6-C01-make-done-claims-provable.md) (order 4) runs after it (soft order; no shared file). [S6-C02](../closeout/S6-C02-docs-truth-and-acceptance-records.md) cites this result for Sprint 6 DoD box 9.
- [S7-03](../../sprint-7/S7-03-safe-action-deadlines.md) edits `applyMatch` later (deadline parsing, :196-208). This ticket lands first; S7-03 finds the code by symbol.

### Likely files and components

- Backend `src/services/matcher.ts`:
  - `MatcherService.matchEmailToApplication`, ambiguous branch (:104-116; `prisma.email.updateMany` at :106-115).
  - `MatcherService.applyMatch`, the `AI_AUTO` guard (:137-141), just before the application lock (:142).
- Backend tests: `src/tests/matcher.test.ts` and `src/tests/matching-concurrency.test.ts`.
- Backend `STABILIZATION.md:21` (one sentence, see Documentation updates).
- Not changed: `resolveEmailMatch` (:264-323); the worker path `finish` (`src/services/ai/pipeline.ts:116-131`); contracts, Prisma schema, frontend.

### Implementation notes

- **Why it re-runs (checked).** `finish` calls the matcher (`pipeline.ts:118`) before it writes `processingState: 'COMPLETED'` (:119-130). A crash between the two, or a duplicate job, runs matching again on an email that is already matched.
- **Ambiguous update.** Add `matchState: { in: [EmailMatchState.UNMATCHED, EmailMatchState.AMBIGUOUS] }` to the `where` at :107-112, next to the existing `OR`. The update runs outside a transaction. Under PostgreSQL's default READ COMMITTED level, an `UPDATE` that waits for a row lock re-checks its `WHERE` on the newest row. So an automatic match that commits first turns this update into a no-op (count 0). No extra lock is needed.
- **`applyMatch` guard.** After the `USER_CONFIRMED` return (:137-141) and before the application lock (:142), return `null` when `source === AI_AUTO`, `email.matchState === MATCHED` and `email.applicationId !== applicationId`. The check reads the row under the email lock (:134), so it sees committed state. Returning before :142 means the other application is never locked or written. `actionId` is null, so no notification job is queued (:213-219).
- **What does not change.**
  - A replay to the same application still runs and stays idempotent: the existing event and action checks (:176-192, :194-195) and the raise-only `aiStatus` (:168-175).
  - An UNMATCHED or AMBIGUOUS email is still matched automatically when there is one candidate or a thread match.
  - `USER_CONFIRMED` paths are unchanged (:28-40, `resolveEmailMatch`).
  - Ownership checks are unchanged for every link that proceeds (:142-146). One visible difference: for an email already MATCHED to A, an `AI_AUTO` call with a foreign application ID now returns `null` instead of throwing `APPLICATION_NOT_FOUND`. Nothing is written in either case.
- **Existing split rows are not repaired.** An email that already became AMBIGUOUS with `applicationId` kept (before this fix) stays as it is. [S8-01](../../sprint-8/S8-01-correct-a-wrong-email-match.md) §4 item 5 cleans it when that email is corrected.
- **Data model, migrations, API, contracts:** none. No new log line; a skipped re-match is a normal replay result.
- **Rollback:** revert the commit. Only matcher code and tests change.

### Security and privacy

- Fail-safe direction: when matching runs again on a matched email, the existing link wins. Effects never move between applications.
- User isolation is unchanged: email lock first, then the owned-application lock and check for any link that proceeds. The database ownership triggers are unchanged.
- No new logging. No email subject, sender or body goes into a log, test name or evidence record.
- Tests use synthetic users and emails on the guarded test database only.
- Stored data from before the fix is not rewritten (original-data preservation rule).

### Testing

**Tests to add.**
- `src/tests/matcher.test.ts` (sequential):
  - An email auto-matched to A (single candidate). Then a second application with the same company is created, and matching runs again. The email stays `{ applicationId: A, matchState: MATCHED, matchConfirmedBy: AI_AUTO }`. Events and actions exist only on A.
  - An email auto-matched to A in thread T. A newer email in T is MATCHED to B. Matching runs again on the first email. It stays on A, and B gets no event or action.
- `src/tests/matching-concurrency.test.ts` (deterministic interleavings). Use `barrier()` (:14-20), `relevantEmail()` (:22-28) and `effects()` (:31-35). Move `pauseAfterCandidateRead()` (:82-92) to file scope, or add the cases inside its `describe` (:80).
  - **Ambiguous after a committed match:** two applications with the same company, email UNMATCHED. Pause the matcher after its candidate read. Restore the spy, then commit `MatcherService.applyMatch(email.id, A.id, result, 'AI_AUTO')`. Release. The email stays MATCHED to A; effects only on A.
  - **Re-link after a committed match:** one candidate B, email UNMATCHED. Pause after the candidate read. Commit an `AI_AUTO` match to A. Release. The email stays on A. B has no event or action, and its `aiStatus` is unchanged.
- Existing tests must stay green, in particular: "Idempotency: Reprocessing creates no duplicate events" (`matcher.test.ts:197`), "USER_CONFIRMED matches are preserved upon AI reprocessing" (:267), "keeps one event and action when the same email is processed concurrently" (`matching-concurrency.test.ts:68`), "preserves deliberate ambiguity …" (:183), and `src/tests/stabilization-safety.test.ts`.

**Mutation checks.** Stage this ticket's changes first (`git add`). Then make each change below, one at a time. Restore with `git checkout -- src/services/matcher.ts` and confirm `git diff --exit-code` is empty. Record the results.

| Temporary change | Tests that must fail |
| --- | --- |
| Remove the new `matchState` condition from the ambiguous `updateMany` | "Ambiguous after a committed match"; the sequential second-application case |
| Remove the new `MATCHED` guard in `applyMatch` | "Re-link after a committed match"; the sequential thread case |

**Commands.** Backend repo root, test database per the Phase 0 guide:

```sh
npm run typecheck && npm run lint && npm run build
npx vitest run src/tests/matcher.test.ts src/tests/matching-concurrency.test.ts src/tests/stabilization-safety.test.ts
npm test
```

Frontend: no code or contract change. `npm run sync-contracts` must report `0 file(s) changed`. The frontend `typecheck`, `lint`, `test` and `build` are not needed for this ticket; the [final closeout run](../closeout/S6-C02-docs-truth-and-acceptance-records.md#final-closeout-run-2026-10-02) runs them.

### Documentation updates

- Backend `STABILIZATION.md:21` (the S6-03 matching paragraph): add one sentence. "Automatic matching never re-links or re-marks an email that is already MATCHED to another application; the ambiguous update applies only to UNMATCHED or AMBIGUOUS emails (S6-R07)."
- This ticket: set Status to Done, with the commit hash and a link to the evidence. [Review README](README.md) row for S6-R07 (:106): Open → Done.
- Closeout execution report (`docs/planning/sprint-6/closeout/execution-report.md`; create it if no earlier closeout ticket has): the new test names, the mutation results and the backend suite counts.
- No change to `docs/architecture/mvp-architecture.md` :32 (ambiguity) or :66 ("no rematching/merging"). Both stay true.

### Definition of done

- [ ] Both acceptance criteria above are met, with evidence linked.
- [ ] Each mutation check failed as listed and was restored byte-for-byte.
- [ ] Backend typecheck, lint (0 errors), build, the focused tests and the full suite are green on the new PC.
- [ ] No contract, schema, migration or frontend change; `npm run sync-contracts` reports 0 changed.
- [ ] Docs updated as listed. One focused backend commit and one docs commit.
- [ ] Evidence recorded in the closeout execution report.
