# S6-R08 — UX, accessibility and contract polish

> **Amended and scheduled (2026-10-02):** Sprint 6 closeout, order 5 of 8 ([closeout plan](../closeout/README.md)).
> - **Item 3 (retry wording) is done** by BYO AI AI-12: `gmail.tsx` says "could not be confirmed" only for uncertain outcomes. Only a test for the `RETRY_RECENTLY_QUEUED` wording is still missing.
> - **Items 1, 2 and 4 remain open.**
> - **Item 2:** per owner decision OD-12 (recommended), validate the application response date fields in `contracts/application.ts` as ISO strings, so a bad date becomes the recoverable contract error that Sprint 6 DoD item 7 requires. The backend mapper must then send ISO strings, and the affected backend tests must be updated. If the owner prefers, the smaller fallback is to render invalid dates safely and amend DoD item 7.
> - The same live-region and focus problems on the AI and Automation pages are handled in [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md) and [MCP-10](../../mcp-feature/MCP-10-post-verification-corrections.md), not here.

| Field | Value |
| --- | --- |
| Status | Open — not started (local ticket) |
| Severity / priority | Low / follow-up |
| Source | [Review 2026-10-02](README.md) L7, L8, L9 |
| Repository | career-companion-frontend (shared date schema change needs backend contract coordination) |

## Problem
1. **Announcement and focus.** `StatusEditor.tsx`: `close()` unmounts the form and mounts a new `<p role="status">` that already holds "Status saved.". Screen readers may not announce live regions inserted with content. After 409, review or unknown, focus stays on a Save button that is now disabled.
2. **Date crash.** Contract date fields use `z.union([z.date(), z.string()])`, so a malformed date string passes runtime validation and then throws in `format(new Date(x))` during render, crashing the route instead of showing the recoverable contract error.
3. **Retry wording.** `gmail.tsx` shows "Retry could not be confirmed: …" even for definitive 409 rejections (`AI_OPERATION_REQUIRES_REVIEW`, `RETRY_RECENTLY_QUEUED`).
4. **Stale contract files.** `scripts/sync-contracts.mjs` never removes frontend contract files that no longer exist in the backend.

## Fix
- Keep one persistent live region outside the open/closed branches, and move focus to the alert or the next available action.
- Tighten wire date fields to ISO strings in the shared backend contract (coordinate the backend mapper and sync), or render invalid dates safely.
- Use "could not be confirmed" only for `outcomeUncertain` errors.
- Make `sync-contracts` delete stale `.ts` files (or report them) and print the diff.

## Acceptance criteria
- [ ] Component tests cover announcement and focus, malformed-date contract errors, and retry wording by error kind.
- [ ] Contract sync leaves the frontend directory identical to the backend directory.

## Verification
Frontend `typecheck`, `lint`, `npx vitest run`, `build`; backend contract tests if the schema changes; smoke.

## Closeout amendment (2026-10-02)

The sections above stay as written. This section adds what the closeout needs and assumes OD-12 is accepted as recommended. Line numbers were checked on 2026-10-02 against the backend and frontend folders (their `src/contracts` folders are identical today). Re-check them before editing, and find code by symbol name.

### Dependencies

- [Phase 0 gate](../../migration-verification/README.md#mv-16--gate-record-results-and-decisions) passed (MV-16), including git (needed for the mutation checks). No migration here; after AI-19 there are 16, AI-19's the newest.
- [OD-12](../../README.md#6-owner-decisions) recorded before item 2 starts. If the owner picks the fallback instead, skip every backend change below. Add a safe date formatter at the four frontend rendering sites listed under item 2, and S6-C02 amends Sprint 6 DoD item 7.
- Closeout order 5 of 8, after AI-19, AI-20, S6-R07 and S6-C01.
- [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md) (order 2) first. The `RETRY_RECENTLY_QUEUED` test uses its visible "Manual Retry" row action. AI-20 also edits `ai-status.test.tsx` and adds `getAISettings` to the `status-correction.test.tsx` mock (its item 20). Build on its versions.
- [S6-C01](../closeout/S6-C01-make-done-claims-provable.md) (order 4) edits backend `application-status.test.ts` (lock test, `sideEffectCounts`). Build on its version of that file.
- Unblocks:
  - [S6-C02](../closeout/S6-C02-docs-truth-and-acceptance-records.md): Sprint 6 DoD box 7 cites item 2; box 17 needs the keyboard check below.
  - [S7-03](../../sprint-7/S7-03-safe-action-deadlines.md) rebases on this change to `contracts/application.ts` (same file; it adds `deadlinePrecision` to `ApplicationActionResponseSchema`).
  - [MCP-10](../../mcp-feature/MCP-10-post-verification-corrections.md) (order 7) copies the fixed `StatusEditor` live region and focus pattern.
  - [S7-01](../../sprint-7/S7-01-ci-and-toolchain-pins.md) `contract-drift` recovery relies on item 4 (helpful, not blocking).
- Merge order: the backend commit (contract, mapper, backend tests) first, then the frontend commit with the sync. S7-01's drift check, if it exists by then, is red in between.

### Likely files and components

Backend (`career-companion-backend`):
- `src/contracts/application.ts`: `ApplicationResponseObjectSchema` (:71-91), fields `userStatusSetAt` (:78), `appliedAt` (:83), `createdAt` (:84).
- `src/services/application.ts`: `ApplicationService.mapToResponse` (:399-437), lines :413, :416, :417.
- `src/tests/application-status.test.ts`.

Frontend (`career-companion-frontend`):
- `src/contracts/application.ts`: synced by `npm run sync-contracts`, never edited by hand.
- `src/components/StatusEditor.tsx` (item 1).
- `scripts/sync-contracts.mjs` (item 4).
- Tests: `src/tests/status-correction.test.tsx`, `src/tests/ai-status.test.tsx`, `src/tests/stabilization-ui.test.tsx` (one fixture), new `src/tests/sync-contracts.test.ts`.
- Checked, no change expected: `src/api/client.ts`, `src/components/ApplicationStatus.tsx`, `src/routes/applications.tsx`, `src/routes/applications.$id.tsx`, `src/routes/gmail.tsx`.

### Implementation notes

**Item 2: application date fields (OD-12).** Every `z.union([z.date(), z.string()])` in `src/contracts/application.ts`:

| Line | Field | Runtime-checked by the frontend | Rendered with `format(new Date(x))` | This ticket |
| --- | --- | --- | --- | --- |
| :47 | `RecentEventSchema.createdAt` | yes | no (cards show `recentEvent.recordedAt`, `applications.tsx:66`, already ISO) | unchanged |
| :78 | `userStatusSetAt` | yes | yes: `EffectiveStatus`, `ApplicationStatus.tsx:45` | ISO, nullable |
| :83 | `appliedAt` | yes | yes: `applications.tsx:54`, `applications.$id.tsx:257` | ISO, nullable |
| :84 | `createdAt` | yes | yes: `applications.tsx:56` | ISO |
| :85 | `updatedAt` | yes | no | unchanged |
| :131 | `ApplicationEventResponseSchema.createdAt` | yes | no (timeline shows `recordedAt`, `applications.$id.tsx:84`) | unchanged |
| :154 | `ApplicationActionResponseSchema.deadline` | no | yes (`index.tsx:67`, `applications.$id.tsx:153`) | left to S7-03 |
| :156 | `ApplicationActionResponseSchema.createdAt` | no | no | unchanged |

- "Runtime-checked" means parsed in `ApiClient.request` (`src/api/client.ts:151-154`): `createApplication` (:178-184), `listApplications` (:186-199), `getApplication` (:201-207), `updateApplicationStatus` (:210-219) and `getApplicationEvents` (:222-232). Action responses pass no schema (`getApplicationActions`, :235-240).
- So the three fields in OD-12 are `userStatusSetAt`, `appliedAt` and `createdAt`: the application response fields the frontend both checks and renders. A bad value in the other fields cannot crash a render today, so they stay as they are. `deadline` (:154) is left to S7-03.
- **Schema:** replace the union with `IsoDateTimeSchema` (`z.iso.datetime({ offset: true })`, :29), which `recordedAt` already uses. Add `.nullable()` for `userStatusSetAt` and `appliedAt`. It accepts `Z` and numeric offsets. It rejects other text, date-only strings and `Date` objects.
- **Mapper:** `mapToResponse` returns `Date` objects today. Emit `app.userStatusSetAt?.toISOString() ?? null` (:413), `app.appliedAt?.toISOString() ?? null` (:416) and `app.createdAt.toISOString()` (:417). `npm run typecheck` fails until this is done, because `ApplicationResponse` (:104) then types these fields as `string`.
- **Serialization (checked):** every route sends with `res.json` (`src/routes/application.ts:23`, :35, :120, :140). Express turns a `Date` into the same `toISOString()` text, so the JSON does not change. Old and new frontends keep working.
- **Valid rows always pass.** `userStatusSetAt` comes from the server clock (`application.ts:254`) and `createdAt` from the database default. `appliedAt` comes from validated input: `CreateApplicationRequestSchema` (:21) or MCP `submittedAt` (`externalSubmission.ts:51-53`). So `toISOString()` always gives a four-digit year the schema accepts.
- **Frontend (no code change for item 2).** After the sync, a malformed date in these fields becomes `ApiError` kind `contract` with a fixed message (`client.ts:153-154`). The four rendering sites (`ApplicationStatus.tsx:45`, `applications.tsx:54` and :56, `applications.$id.tsx:257`) are then never reached with bad data. The detail page shows `SectionError` with Retry (`applications.$id.tsx:237-243`, component :185-192). Editing is off because `canEdit` is false on error (:216). The list shows "Failed to load applications" (`applications.tsx:233-234`).

**Item 1: `StatusEditor` live region and focus.**
- Today `close()` (:65-70) unmounts the form. The closed branch (:152-164) then mounts a new `<p role="status">` (:161) that already holds "Status saved.". After a save outcome, Save (:243) is disabled by `blocked` (:167-170), so focus is left on a disabled control.
- **Live region:** render the status `<p>` once, after the open/closed branch, at the same position in both. For example `<>{session ? form : closed}<p className="sr-only" role="status" aria-live="polite">{announcement}</p></>`. React then keeps one DOM node that exists, empty, before any text is set. Keep `setAnnouncement('')` in `start()` (:113).
- **Focus:** give the form's alert container (:217) a ref and `tabIndex={-1}`. Focus it with `requestAnimationFrame`, as `close()` does (:69):
  - in the `save()` error handlers (:136-147): 409 conflict, 404 unavailable, a definitive error message;
  - at the end of `reconcile()` (:72-80): review or unknown. This also covers "Retry loading status" (:257-261), whose button disappears when the phase changes.
- After `rebase()` (:116-124), "Use current version" disappears. Focus the status select (`select` ref, :56).
- Never move focus when only a background read changes the alert (the stale notice, :219-221). A refetch must not pull focus away while the user is choosing.

**Item 3 leftover: `RETRY_RECENTLY_QUEUED` test only.** The code is done: `retryMessage` (`gmail.tsx:26-41`, case at :34-35); the backend sends 409 `RETRY_RECENTLY_QUEUED` (`src/routes/email.ts:278-283`). Only a test is missing (Testing below).

**Item 4: stale contract files.**
- Today `scripts/sync-contracts.mjs:23-34` copies backend `.ts` files and never looks at frontend files the backend no longer has.
- **Recommendation: delete, not only report.** The acceptance criterion asks for identical folders, and git shows the deletion for review.
- After the copy loop, list the top-level `.ts` files in the frontend folder that are not in the backend list. Delete each (`fs.unlinkSync`), print `Removed <file>`, and count it in `changed`.
- Keep the last line's form `(<n> file(s) changed)`. S7-01 and the final closeout run check for `0 file(s) changed`.
- If the backend folder has no `.ts` files, exit 1 before writing or deleting anything. A wrong `BACKEND_DIR` must never empty the frontend folder.
- The per-file lines (`Synced`, `Unchanged`, `Removed`) are the printed diff summary. For the line diff, use `git diff -- src/contracts` (frontend `README.md:32` already says to review it).

**Commit order:** backend commit (item 2 backend part). Then one frontend commit: item 4 first, then `npm run sync-contracts`, item 1 and the tests.

**Rollback:** revert the frontend commit, then the backend commit. The JSON never changed, so neither revert breaks the other repo.

### Security and privacy

- A contract error never echoes the response payload (`client.ts:152-154`). The new tests assert the fixed message only, with no date value in it.
- The live region announces fixed text only ("Status saved."), no application data.
- `sync-contracts` deletes only top-level regular `.ts` files inside the frontend `src/contracts`. It refuses an empty backend folder and only reads the backend.
- No auth, ownership or API change. The JSON is unchanged.
- Synthetic fixtures only; no real application data in tests or evidence.

### Added acceptance criteria (2026-10-02)

- [ ] Backend: `ApplicationResponseSchema` rejects a malformed string (`'x'`, `'not-a-date'`), a date-only string (`'2026-09-01'`) and a `Date` object in `userStatusSetAt`, `appliedAt` and `createdAt`. It still accepts null for the first two, and ISO strings with `Z` or an offset.
- [ ] Backend: create, list, detail and status PATCH responses parse with the tightened schema, including non-null `appliedAt` and `userStatusSetAt`. The JSON text of these fields is unchanged.
- [ ] A malformed date string in any of the three fields fails runtime validation (`ApiError` kind `contract`, fixed message). The detail page shows the recoverable "Failed to load application data" error with Retry. Editing is disabled ("Change status" is not offered, or is disabled while earlier data is shown). Nothing throws during render. A malformed list response shows "Failed to load applications".
- [ ] `deadline` (:154) and the other union fields are unchanged.
- [ ] Item 1: the status live region is the same DOM node before and after a save. After 409, review, unknown and 404, focus is on the alert, never on a disabled control or `body`. After "Use current version", focus is on the status select.
- [ ] Item 3: a test covers the `RETRY_RECENTLY_QUEUED` message, reaching retry through AI-20's visible row action.
- [ ] Item 4: a stale frontend contract file is removed and reported. An empty backend contracts folder makes the script exit 1 and change nothing.

### Testing

**Backend tests.**
- `application-status.test.ts`, "rejects responses with missing or inconsistent canonical fields" (:101-115). Its `valid` object uses `createdAt: 'x'` (:105), which fails the new schema, so the `true` check at :108 would fail. Change `createdAt` and `updatedAt` to an ISO string (for example `'2026-09-01T00:00:00.000Z'`). Add `false` cases for each of the three fields: `'x'`, `'2026-09-01'`, `new Date()`. Add `true` cases for null `appliedAt` and `userStatusSetAt` and for `'2026-09-01T10:00:00+05:30'`.
- Same file, "returns revision 0 and canonical fields from create" (:118-123): add a create with `appliedAt: '2026-09-01T10:00:00.000Z'` and assert the parsed response returns that exact string. The parse at :136 now proves `userStatusSetAt` is ISO; :140 stays.
- Must stay green, because they parse real responses: `application-status.test.ts:89` and :95, `application-submission-evidence.test.ts:47-74`, `application-evidence.test.ts:129-134`.

**Frontend tests.**
- `status-correction.test.tsx`, "runtime contract validation (real client)" (:67-99): `it.each` over the three fields with `'not-a-date'`. `getApplication` rejects with `{ kind: 'contract', message: 'The server returned an unexpected response.' }`. Add one `listApplications` case for `createdAt`.
- Same file, "section isolation and evidence" (:338): add a route test through the real parse. The file keeps the real `ApiClient` (:14-24). Point `api.getApplication` at `new ApiClient().getApplication` and use `respond(makeApplication({ appliedAt: 'not-a-date' }))`. Assert "Failed to load application data", no "Change status" button, and no uncaught error. This test fails today: the union accepts the string and `format` throws.
- Same file, "status editor" (:152):
  - Extend "sets a status, closes, announces and returns focus" (:153-166): the live region exists and is empty before opening, and the same node holds "Status saved." after.
  - Assert focus on the alert container in the 409 case (:195-215), the uncertain cases (:256-273), the unknown cases (:275-305) and a new 404 case. After "Use current version", the select has focus.
- `ai-status.test.tsx`: turn "gives definitive refusals their own message" (:173-178) into an `it.each` over `AI_OPERATION_REQUIRES_REVIEW` and `RETRY_RECENTLY_QUEUED`. Reach retry through AI-20's no-hover row action, not `clickManualRetry` (:125-135). Assert "A retry was queued recently. Give it a few minutes." and no "could not be confirmed".
- New `src/tests/sync-contracts.test.ts` (Vitest's default Node environment):
  - Copy `scripts/sync-contracts.mjs` into a temp `frontend/scripts/` folder. Create a temp backend `src/contracts/a.ts`, and frontend `src/contracts/a.ts` (different) plus `stale.ts`. Run the copy with `node` and `BACKEND_DIR`. Assert `stale.ts` is gone, `a.ts` matches, and the output has `Removed stale.ts`.
  - Second case: an empty backend contracts folder gives exit 1, and the frontend files are untouched.
  - The script finds its target from its own path (:20), so the real `src/contracts` is never touched. `tsconfig.app.json` sets no `types` list and `@types/node` is installed, so Node imports should typecheck; confirm with `npm run typecheck`.
- `stabilization-ui.test.tsx:16`: change `createdAt` and `updatedAt` from `'2026-09-01'` to full ISO strings, so fixtures match the contract. The test mocks the client, so it passes either way.

**Mutation checks.** Stage this ticket's changes first. Make each change below, one at a time. Restore with `git checkout -- <file>` and confirm `git diff --exit-code` is empty. Record the results.

| Temporary change | Test that must fail |
| --- | --- |
| Frontend `src/contracts/application.ts`: `appliedAt` back to `z.union([z.date(), z.string()]).nullable()` | the malformed-date route test and its contract case |
| Move the status `<p>` back into the closed branch of `StatusEditor` | the live-region node check |
| Remove the focus call in the 409 handler | the 409 focus check |
| Remove the delete loop in `sync-contracts.mjs` | `sync-contracts.test.ts` |

**Commands.**

```sh
# backend repo root
npm run typecheck && npm run lint && npm run build
npx vitest run src/tests/application-status.test.ts src/tests/application-evidence.test.ts src/tests/application-submission-evidence.test.ts
npm test
# frontend repo root, after the backend commit
npm run sync-contracts      # first run: application.ts synced; second run: 0 file(s) changed
diff -r <backend>/src/contracts src/contracts   # no output
npm run typecheck && npm run lint
npx vitest run src/tests/status-correction.test.tsx src/tests/ai-status.test.tsx src/tests/sync-contracts.test.ts src/tests/stabilization-ui.test.tsx
npm test
npm run build
```

**Manual and smoke.**
- Record a keyboard-only pass of the status editor at desktop width and at 390 px: save, a 409 from a second tab, Escape. With a screen reader if one is available, confirm "Status saved." is read.
- The smoke's status-editor steps (`scripts/smoke-stabilization.mjs:602-696`) check "Status saved." and focus return. They must still pass. The [final closeout run](../closeout/S6-C02-docs-truth-and-acceptance-records.md#final-closeout-run-2026-10-02) runs the smoke.

### Documentation updates

- Backend `README.md:128`: add one sentence. "`userStatusSetAt`, `appliedAt` and `createdAt` are ISO 8601 date-time strings (the first two may be null)."
- Frontend `README.md:32`: say that `sync-contracts` also removes frontend contract files the backend no longer has, and prints each file synced or removed. `README.md:36`: add that malformed application dates (`appliedAt`, `createdAt`, `userStatusSetAt`) are a contract error. S6-C02 §4.6 edits :36 later; keep both changes.
- This ticket: record the OD-12 outcome, set Status to Done with both commit hashes. [Review README](README.md) row for S6-R08 (:107): Open → Done.
- Closeout execution report: the new test names, mutation results, test counts and the manual keyboard note.
- Sprint 6 DoD box 7 is ticked by S6-C02, not here.

### Definition of done

- [ ] The OD-12 outcome is recorded in this ticket.
- [ ] The original and the added acceptance criteria are met, with evidence linked.
- [ ] Backend: typecheck, lint (0 errors), build, the focused tests and the full suite are green on the new PC.
- [ ] Frontend: `sync-contracts` (0 changed on the second run), contracts identical to the backend (`diff -r`), typecheck, lint (0 errors), full suite and build are green.
- [ ] Each mutation check failed as listed and was restored byte-for-byte.
- [ ] The manual keyboard pass is recorded.
- [ ] The backend commit is merged before the frontend commit; one docs commit.
- [ ] Evidence is recorded in the closeout execution report.
