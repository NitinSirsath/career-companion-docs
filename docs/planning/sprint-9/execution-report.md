# Sprint 9 execution report — 2026-10-03

Status: **local engineering implementation complete; owner real-use acceptance pending.** S9-01..S9-04 implemented in the existing scratchpad siblings. OD-16/17 accepted for this sprint after the owner authorized resolving the review findings and starting Sprint 9. Sprint 10/11 remain proposed.

## Scope and source state

Backend baseline: `6af9cd2d994e4d3d1944a5d0029b6d64f0085519`. Frontend baseline: `155a1aacd9d7f2e6df56657a128b1cf511b616d6`. Both were clean at kickoff. Existing uncommitted documentation planning was preserved and extended. All Sprint 9 changes remain **uncommitted**; no branch switch, commit, push, PR or external tracker operation.

Workspaces remain under `/private/tmp/claude-501/-Users-spurge-rental-Downloads-docs/8472a241-e50b-41f8-9bcf-4a75ed0fd02a/scratchpad/`. No new migration, provider contract, scheduler or background job was added. Existing migrations were applied only to newly created guarded fixture databases for testing.

## Delivered behavior

- **S9-01:** authenticated action read model with complete disjoint bucket counts, bounded pages, stable ordering and one repeatable-read snapshot. Owner isolation, retirement and PENDING eligibility apply before pagination. Separate review counts preserve existing queue predicates.
- **S9-02:** dashboard buckets, full counts, bounded pages, existing Complete/Dismiss and context links, review shortcuts and explicit partial coverage. Request/response validation catches malformed coverage/actions. Uncertain writes refresh without replay. Existing processing, correction, review and status invalidations reach the workspace.
- **S9-03:** literal case-insensitive company/title search and canonical effective-status filters in PostgreSQL before pagination. Filters enter query keys; stale requests cannot repaint another filter. Existing picker defaults and bounded recent evidence remain intact. Empty results distinguish no matching applications from no applications.
- **S9-04:** database/component regressions and extended real-API browser smoke, with keyboard interactions, narrow viewport and preserved correction flows. A user trial is still pending.

## Three review findings resolved

1. `nextTransitionAt` comes from **all eligible actions**, regardless of selected bucket/page. It considers timed/legacy deadlines and the next local midnight, with DST-aware timezone conversion. The UI uses server-relative delay, at least 60 seconds, and refreshes on restored focus/visibility. A test fires the unseen deadline timer while Undated is selected and checks cleanup.
2. Acknowledged cache updates remove nonmatching rows without inventing destination-page membership. Newer manual revisions win even over out-of-order acknowledgements. If revision merging changes a late GET's membership, one bounded reread is allowed; failure/inconsistency stays a read error. Tests cover clear, delayed data and failed rereads.
3. Coverage never infers processing completion from READY AI, zero waiting mail or recent Gmail sync. It explicitly states that PROCESSING/FAILED mail is outside the waiting count. Failed/malformed reads are unknown. Empty action data says only “No stored pending actions.”

Detailed request/response and display rules: [API contract](api-contracts.md).

## Verification measured here

| Check | Result | Evidence |
| --- | --- | --- |
| Backend full suite | **55 files, 777 tests passed** | [Output](evidence/backend-tests.txt) |
| New backend workspace/discovery cases | **10 passed**, real PostgreSQL | `src/tests/workspace.test.ts`: 47-row counts, >20 paging, owner/retired/handled exclusion, DATE/DATETIME/legacy semantics, both LA DST boundaries, unseen next transition, real concurrent retirement, queue predicates, validation and literal search/status precedence |
| Frontend final full suite | **17 files, 195 tests passed** | [Output](evidence/frontend-tests.txt) |
| Typechecks and builds | Backend and frontend passed | `npm run typecheck`; `npm run build` in each sibling ([backend build](evidence/backend-build.txt), [frontend build](evidence/frontend-build.txt)) |
| Lint | Both passed; baseline warnings retained | [Backend](evidence/backend-lint.txt), [frontend](evidence/frontend-lint.txt). Backend 7 existing warnings; frontend existing Fast Refresh/unused catch/Gmail effect warnings. New page-recovery effect warnings were removed. |
| Shared contracts | Sibling contract files identical | Local `npm run sync-contracts` and byte comparison |
| Browser smoke | **Passed**, real API/PostgreSQL/pg-boss, external Gmail/AI replaced by fixtures | [Output](evidence/browser-smoke.txt) |
| Formatting / scoped diff | `git diff --check` passes in all three checkouts | Local verification; no staging |

Backend tests used new `career_companion_s9_20261003_test`. Browser smoke used separate new `career_companion_s9_20261003_smoke_test`. Test setup now accepts `TEST_ENV_FILE` while retaining the original equality/local-host/database-name safety guard; existing default `.env.test` is unchanged. Ignored fixture environment files contain only the inherited fixture configuration and new test targets.

Repeat the backend suite with `TEST_ENV_FILE=.env.sprint9.test npm test` from backend. Frontend uses `npm test`. Smoke uses the existing `scripts/smoke-stabilization.mjs`, an explicit `BACKEND_DIR`, and `SMOKE_DATABASE_URL` from the guarded smoke fixture configuration. Never substitute a real user database. No new migration means no additional preservation lane was required by S9-04.

Browser assertions include multi-page counts matching active owner data, keyboard bucket selection and completion, decremented totals, literal application search, manual override filtering/clear, failed coverage reads, and 390px horizontal-overflow checks. The S9 interval produced **zero additional AI or Gmail sync calls**. The inherited worker scenarios made 14 synthetic AI calls; no real provider calls occurred. Existing move/unlink/restore, unknown-write and ownership scenarios still passed. This is automated fixture evidence, not a manual accessibility certification or a live Gmail/provider result.

The initial smoke reached S9 then failed because the harness focused the search input before React mounted it. Adding a selector wait fixed the harness; the complete rerun passed. On success both HTTP and worker work drained before cleanup. Residual **users/jobs/budgets/MCP = 0/0/0/0**; unexpected outbound traffic was blocked. Existing Vite CJS configuration/bundle-size and jsdom scrollTo warnings remain baseline notices.

Documentation verification: 121 relative link targets checked across 16 current index, architecture, user-flow and Sprint 9 documents; no missing targets. All 11 backend/frontend contract files match byte-for-byte. After the final uncertainty-copy clarification and formatting, frontend build/lint and all 11 workspace UI tests passed again ([output](evidence/workspace-ui-final.txt)).

## Visual evidence

Synthetic-data screenshots: [desktop workspace](evidence/s9-workspace-desktop.png), [390px workspace](evidence/s9-workspace-mobile.png). The same run captured inherited AI/automation/correction screens in the evidence folder. No real recruiter content or provider credential is used.

## Pending acceptance

- **S9 user trial:** pending. Ask the owner to find due work, complete/dismiss an action, search for an application, change/clear a status and interpret coverage during daily use. Record dated observations separately.
- Existing Sprint 7/8 live scheduled days/Gmail behavior, original personal-data preservation, Gemini certification and real automation-client acceptance remain pending under their existing IDs.
- No remote CI, deployment or provider promotion is claimed. Sprint 10/11 implementation has not begun.
