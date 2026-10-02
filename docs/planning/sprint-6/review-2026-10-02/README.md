# Sprint 6 independent review — 2026-10-02

> **Status update (2026-10-02):** R01–R06 are fixed (R06 as BYO AI AI-00). R08 item 3 (retry wording) was fixed by BYO AI (AI-12). R07 and the rest of R08 are scheduled in the [Sprint 6 closeout](../closeout/README.md). Older lines below that say "R06–R08 remain open" predate this note.

Read-only review of the current working tree (backend, frontend, docs) against the Sprint 5 and Sprint 6 contracts. No application code was changed and Git was not used. Two independent reviewers (backend; frontend/harness/docs) worked from source; findings were re-checked by direct inspection. One finding was confirmed by logging Prisma's SQL inside a rolled-back transaction on the guarded test database.

**Recommendation (original): ready after specific fixes.** **Update 2026-10-02:** R01–R05 are fixed and verified (backend 236/236, frontend 90/90, typecheck/lint 0 errors/builds, migration lanes, real-worker smoke and all teardown modes); see each ticket's Resolution section. Sprint 6 is technically ready for engineering acceptance pending the open live Gmail/original-data gate; it is not fully accepted. R06–R08 remain open. No critical or high-severity defect was found. Fix R01–R05 before accepting Sprint 6 engineering. R06–R08 can follow. Even after the fixes, acceptance is engineering-only: the Sprint 6 entry gate (live Gmail and original-data evidence) remains open by owner decision.

## A. Critical issues
None found.

## B. High-risk issues
None found.

## C. Medium and low issues

| ID | Severity | Summary | Ticket |
| --- | --- | --- | --- |
| M1 | Medium | `recentEvent` loads every event (plus source email) for the page's applications and trims to one in memory. Violates S6-02 "at most one recent event per application" | [S6-R01](S6-R01-bound-recent-event-selection.md) |
| M2 | Medium | Detail stale-read guard snapshots the cache **before** the fetch, so it cannot protect an acknowledged correction. Cancellation currently hides this | [S6-R02](S6-R02-fix-stale-read-guard.md) |
| M3 | Medium | Harness HTTP drain counts socket close, not handler completion. Closing the browser can make a live handler look drained | [S6-R03](S6-R03-harden-smoke-teardown.md) |
| M4 | Medium | Runbook §8 says the smoke covered "all scenarios in the §6 table". Several rows are covered only by unit/backend tests or not at all | [S6-R04](S6-R04-correct-verification-claims.md) |
| M5 | Medium | No test proves `retry: false` on the status PATCH or create POST. TanStack v5 mutations default to no retry, so the tests would pass without it | [S6-R05](S6-R05-strengthen-sprint6-tests.md) |
| L1 | Low | Automatic matching can re-mark an already auto-matched email AMBIGUOUS or re-link it, leaving effects on two applications (mostly pre-existing) | [S6-R07](S6-R07-guard-automatic-rematch.md) |
| L2 | Low | Rethrown worker errors are stored unsanitized in `pgboss.job.output` | [S6-R06](S6-R06-worker-error-hygiene.md) |
| L3 | Low | "Outcome unknown" provider errors are stored as `processingRetryable: true` and briefly shown as "Retrying" | [S6-R06](S6-R06-worker-error-hygiene.md) |
| L4 | Low | Weak tests: no-side-effect test doesn't assert notification or Gemini mocks; no foreign-`recentEvent` test; no action-section failure test; the "unknown blocks Save" assertion can't isolate its cause | [S6-R05](S6-R05-strengthen-sprint6-tests.md) |
| L5 | Low | Dashboard uses the same `['applications', …]` cache key but skips the revision guard. Mutation cache writes use `result.id`, not the captured ID | [S6-R02](S6-R02-fix-stale-read-guard.md) |
| L6 | Low | Quarantine path restores `fetch` and releases the lane lock while a hung handler is alive. No failed-run or worker-drain-failure case. Browser non-local requests (fonts) are aborted but not explicitly allowlisted. Residue counts are not recorded by the harness | [S6-R03](S6-R03-harden-smoke-teardown.md) |
| L7 | Low | "Status saved." live region is inserted with content (may not be announced). Focus stays on a disabled Save after 409, review or unknown | [S6-R08](S6-R08-ux-contract-polish.md) |
| L8 | Low | Malformed date strings pass the contract (`z.union([z.date(), z.string()])`) and then crash rendering in `format(new Date(x))` | [S6-R08](S6-R08-ux-contract-polish.md) |
| L9 | Low | Gmail retry shows "Retry could not be confirmed" even for definitive 409 rejections. `sync-contracts` never removes deleted contract files | [S6-R08](S6-R08-ux-contract-polish.md) |

## D. Confirmed correct areas

- **S6-01:**
  - `PATCH /api/applications/:id/status` validates the path UUID, then the strict body (`userStatus` key required, null allowed, revision a non-negative integer).
  - The owned row lock uses `id` and `userId`. Revision is compared before no-op. A same-value request writes nothing. A change or clear increments the revision once. Clear also nulls the timestamp.
  - The response comes from the same transaction. 409 `STATUS_CONFLICT`. Missing and foreign IDs return the same 404 body.
  - Only this path writes manual fields. The matcher writes `aiStatus` only.
- **Locking:** the only `FOR UPDATE` sites are matcher (email → application) and PATCH (application only). No inversion.
- **Migration:** additive `NOT NULL DEFAULT 0`. The guard rejects unsafe targets before the Prisma CLI runs. The upgrade lane compares all seven tables plus trigger and unique-index names.
- **S6-02:** parent ownership kept (existing 403). A foreign source nulls both `sourceEmail` and `emailId`, and the log holds IDs only. Ordered by `createdAt`, `id`. ISO timestamps. No action evidence, per D1.
- **S6-03:**
  - `batchSize: 1` is explicit. A thrown handler fails every delivered job ID in pg-boss 12.31.0.
  - The final-attempt check matches pg-boss retry counting. Completed emails are never downgraded.
  - The `USER_CONFIRMED` fresh-state check fixes the reproduced race. Link-vs-ignore is safe.
- **S6-04:**
  - The editing session (ID, revision, draft) stays frozen. Rebase happens only through "Use current version".
  - Duplicate submits are blocked. Uncertain outcomes reconcile by reading. "Save outcome unknown" blocks Save.
  - Reads are cancelled before the PATCH and before the result is applied. Lower-revision cache writes are refused. The editor is keyed per application. Sections fail independently.
  - Create recovery keeps the draft, blocks resubmission and requires a deliberate "Create anyway".
- **Contracts:** all six frontend contract files are byte-identical to the backend's. Runtime parsing rejects missing fields and accepts true nulls. `ApiError` keeps status and code and never echoes the payload.
- **Sprint 5 prerequisites:**
  - The retry route no longer deletes claims or resets decisions, writes no processing state, and checks the owner.
  - Worker outcomes and sanitized `emails` error fields are correct.
  - The 15 s deadline and abort forwarding are correct. The refresh window always expires.
  - The changes are minimal. The deferred Gmail items ([S5-FU-01](../../sprint-5/follow-up-gmail-reliability.md)) cause no Sprint 6 correctness problem.
- **Harness:**
  - DB name guard, backend guard, advisory lock and emptiness check all run before any write. Outbound `http`/`https`/`fetch` is blocked.
  - Workers stop fetching before barriers are released, and `offWork({ wait: true })` waits for the active handler. Cleanup covers jobs by owner and action ID.
- **AI behaviour:** unchanged. `operations.ts` only exports `MAX_ATTEMPTS`; pipeline, prompts and models are untouched.

## E. Scope and documentation inconsistencies

- **Scope (acceptable, recorded for transparency):**
  - Detail cards lost `rounded-xl`/`shadow-sm` (design system requires square corners).
  - The actions section now shows "No actions yet." instead of hiding.
  - The retry route adds 409 `RETRY_RECENTLY_QUEUED` (truthful outcome, Sprint 5 intent).
  - Gmail retry shows an error line.
  - The harness has a test-only probe route.
  - None adds product features or AI behaviour.
- **Documentation overstatements (R04):**
  - Runbook §8 "all scenarios in the §6 table".
  - "Byte-identical" upgrade comparison: actually equal SHA-256 digests of the JSON rows.
  - "0 residue": measured once by a manual `psql` after the run, not by the harness.
- **Sprint 6 DoD:** the "entry gate and approved baseline recorded" checkbox cannot be fully met while the owner keeps the live gate open. Reports state this, but no DoD checkbox should be ticked yet.
- **S6-02 title:** still says "AI actions". The D1 note explains it is not delivered. A title change is optional.

## F. ADR-0001 assessment

- **Origin:** created by a different, concurrently active Claude Code session in the same folder, titled "AI application architecture reconnaissance". Its transcript references writing this file.
  - The ADR was first saved around 02:34 IST and revised around 02:45 IST.
  - The same session also created `docs/architecture/ai-capability-architecture.md` (02:36) and edited `README.md` (line 9) and `docs/product/product-vision.md` (02:37).
  - This review session did not create or modify it.
- **Relationship to this work:** none. It is a **Proposed** decision (user-provided AI keys, per-user limits) that explicitly starts only after Sprint 6 and after review under the Project Constitution. It would change AI cost, credential and failure boundaries, so it falls outside Sprint 6 non-goals ("no new AI capabilities").
- **Risks:**
  - Two sessions edited the same docs folder at the same time. Both README edits survived, but this needs care during migration.
  - Docs readers could take the ADR as approved. Its status says Proposed.
- **Recommendation:** leave it untouched in this sprint. Review it on its own track. Do not migrate it as part of the Sprint 6 acceptance package.

## G. Final recommendation

**Ready after specific fixes:** R01, R02, R03, R04, R05. Then rerun backend/frontend full suites, typecheck, lint, build and the smoke. R06–R08 are follow-ups. Live Gmail and original-data evidence remain an open entry gate.

## Ticket index (local; Linear later)

| Ticket | Priority | Status |
| --- | --- | --- |
| [S6-R01 — Bound recent-event selection to one row per application](S6-R01-bound-recent-event-selection.md) | Before acceptance | **Fixed — verified 2026-10-02** |
| [S6-R02 — Fix the stale-read guard and shared list cache](S6-R02-fix-stale-read-guard.md) | Before acceptance | **Fixed — verified 2026-10-02** |
| [S6-R03 — Harden smoke teardown drain and quarantine](S6-R03-harden-smoke-teardown.md) | Before acceptance | **Fixed — verified 2026-10-02** |
| [S6-R04 — Correct Sprint 6 verification claims](S6-R04-correct-verification-claims.md) | Before acceptance | **Fixed — verified 2026-10-02** |
| [S6-R05 — Strengthen Sprint 6 tests that don't prove their claim](S6-R05-strengthen-sprint6-tests.md) | Before acceptance | **Fixed — verified 2026-10-02** |
| [S6-R06 — Worker error hygiene](S6-R06-worker-error-hygiene.md) | Follow-up | **Fixed — verified 2026-10-02** |
| [S6-R07 — Guard automatic re-matching of matched emails](S6-R07-guard-automatic-rematch.md) | Follow-up | Open |
| [S6-R08 — UX, accessibility and contract polish](S6-R08-ux-contract-polish.md) | Follow-up | Open |
