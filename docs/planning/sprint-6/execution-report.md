# Sprint 6 execution report — 2026-10-02

Local engineering execution of the [Sprint 6 contract](README.md) on the downloaded `career-companion-{backend,frontend,docs}-main` folders. No Git, Linear, deployment, live Gmail, live Gemini or Discord action was performed. All evidence below is synthetic/local.

## 1. Baseline and Sprint 5 reconciliation (Phase 1)

The downloaded folders are the older GitHub main (backend baseline suite: 177 tests, matching the pre-Sprint-5 audit count; frontend 52). The uncommitted Sprint 5 working tree described in the [Sprint 5 execution report](../sprint-5/execution-report.md) is **not** present. Per instruction it was not searched for or rebuilt wholesale. The Sprint 5 docs contain no separate Definition of Done section, so only items that Sprint 6 depends on were implemented:

| Missing Sprint 5 item | Why Sprint 6 needs it | Minimal implementation |
| --- | --- | --- |
| Safe manual email retry (S5-03 decision 5, follow-up P2 #1). Route deleted all AI operation claims and reset relevance/match state, then wrote PENDING after enqueue | S6 DoD forbids replay/reset; S6-03 forbids paid-claim resets | No claim deletion or state reset; held claims → 409 `AI_OPERATION_REQUIRES_REVIEW`; suppressed enqueue → 409 `RETRY_RECENTLY_QUEUED`; acknowledgment writes no processing state |
| Email worker retry metadata, final-attempt FAILED, safe allowlisted error text (S5-05). Exhausted retries left emails PROCESSING forever; raw error text was stored | S6-03 requires an attributable outcome per delivered job, preserving Sprint 5 retry semantics | `includeMetadata`, outcome logging (`completed`/`retry_scheduled`/`failed_terminal`/`failed_exhausted`), final failure → FAILED, sanitized details. Also cleared the 2 pre-existing backend lint errors in that file |
| 15-second client deadline with AbortSignal forwarding (S5-04) | S6-04 reuses it for the PATCH and cancellation | Bounded `ApiClient.request` |
| Authenticated-shell bounded refresh across navigation; retry refresh on settle (S5-04, follow-up P2 #2) | S6-04/05 "background refresh must not rebase the editor" and delayed-processing refresh | `ProcessingRefreshObserver` + `lib/processingRefresh.ts` (2-minute window, 4-second interval, always expires) |
| Real-worker Puppeteer harness (S5-04). Existing smoke disabled workers, faked sync completion, used a plaintext token, zero AI budget and the wrong sibling path | S6-05 must extend it | Rewritten harness (see §4) |

Not implemented (Sprint 5 scope not required by Sprint 6; still missing in this baseline): Gmail request/attempt fencing and false-success fixes (G1/G2), bounded Google/OAuth transport (G3), correlated sync telemetry/counters (G4), process-kill crash harness, baseline snapshot tooling. Recorded as proposed follow-up [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md); outside Sprint 6 scope. Live Gmail/original-data evidence remains unverified.

Phase 1 verification: backend 189 tests, frontend 58 tests, typecheck/lint (0 errors)/build pass; real-worker smoke passed with zero residue.

## 2. Decisions (owner responses, 2026-10-02)

- **Entry gate:** proceed with Sprint 6; the live Gmail incremental/repeat-sync and original ~1,620-email preservation gate stays **open**, not passed. Deployed API/worker versions are also unverified.
- **D1:** events/recentEvent evidence only. Action responses keep their existing `emailId`; action source evidence is deferred. The S6-02 title's "AI actions" promise is not delivered.
- **D2:** approved as written (clear = null status and timestamp, latest-only provenance, dedicated revision with 409, recording-order history).
- **Scheduled Gmail sync (README 2026-10-01 note):** left out of Sprint 6; no ticket owns it and S6-03 excludes new schedulers. Separate follow-up.

## 3. Changes by ticket

**S6-01** — Additive migration `20261002090000_add_user_status_revision`. Shared contract adds required `userStatusRevision`, `effectiveStatus`, `statusSource`, `hasStatusConflict` with a precedence refinement, plus strict `UpdateApplicationStatusRequestSchema`. One mapper for create/list/detail/PATCH. `PATCH /api/applications/:id/status` locks the owned row, compares revision before no-op, returns the canonical response from the same transaction. Frontend: `sync-contracts` resolves `BACKEND_DIR`/`-main` sibling and fails instead of skipping; runtime parsing with `ApiError` (status, code, kind, `outcomeUncertain`); list/detail canonical display with neutral unknown; Add Application creation-outcome recovery (draft kept, Save blocked, owned list reconciled, read retry, deliberate "Create anyway", no automatic replay or deduplication).

**S6-02** — `recordedAt` and bounded owned `sourceEmail` on events and recentEvent (list, detail, PATCH). Foreign sources (legacy-only) are nulled with their `emailId` and logged as `evidence_ownership_mismatch` without content. Exported `ListApplicationEventsResponseSchema`. No migration, backfill, provider call or new endpoint.

**S6-03** — Facts: installed pg-boss 12.31.0, `batchSize` default 1, `localConcurrency` 1. `jobs[0]` was not a batch-loss defect. Invariant made explicit: registration `{ includeMetadata: true, batchSize: 1 }`; a multi-job delivery throws so pg-boss retries every delivered job. **Confirmed and fixed defect:** a user link whose pre-lock read was overtaken by a committed automatic match re-linked the email and left event/action effects on two applications. `applyMatch` now re-checks match state under the email lock (failing regression first, then fix). Verified safe without code change: distinct emails on one application (all effects kept, monotonic AI state, manual fields untouched), concurrent duplicate processing, stale automatic link/ambiguous selection versus later user link/ignore, link-vs-link and link-vs-ignore (one accepted decision), deliberate ambiguity, manual PATCH versus an AI write holding the application lock.

**S6-04** — Inline status editor (frozen session, explicit "Use current version" rebase, 409 review, uncertain-save reconciliation, "Save outcome unknown" with Save disabled, no automatic PATCH retry, duplicate-click guard, cancellation before PATCH and before applying results, revision-ordered cache writes, navigation-safe cache scoping, Escape/focus return, live announcements). Evidence timeline (Recorded vs Email date, source unavailable, AI-only state changes without equal-state arrows, AI interpretation label, text-only rendering). Independent application/history/action loading, error and retry; stale cached data labelled and not editable. New `NativeSelect` primitive.

**S6-05** — Extended real-worker smoke (§4), guarded migration runner, fresh/upgrade preservation lanes, documentation updates and the boundary review below.

## 4. Verification results (final run, updated after review fixes R01–R05)

| Check | Result |
| --- | --- |
| Backend `db:generate`, typecheck, build | Pass |
| Backend lint | 0 errors, 11 pre-existing warnings (baseline had 2 errors) |
| Guarded migrate/status/validate (test lane) | 13 migrations, up to date, schema valid |
| Guard rejection ordering | Absent expected target, `.env.test` mismatch, unsafe DB name, unequal URLs: all refused before the Prisma CLI |
| Backend full Vitest | **236 passed, 23 files** (baseline 177; 234 before the review fixes) |
| Frontend `sync-contracts` | Synced from backend; final run 0 files changed (identical) |
| Frontend typecheck, build | Pass (existing >500 KB bundle warning) |
| Frontend lint | 0 errors, 20 warnings (same as baseline) |
| Frontend full Vitest | **90 passed, 10 files** (baseline 52; 85 before the review fixes) |
| Fresh lane | 13 migrations; new application revision 0; 4 ownership triggers present |
| Upgrade lane (synthetic) | 3 owners, 72 applications (24 legacy user statuses with unknown time), 1,620 emails, 162 events/actions/results/operations: all 7 tables identical after upgrade (equal SHA-256 digests of the JSON-serialized rows), all revisions 0, triggers/unique indexes unchanged, cross-owner/duplicate/owner-change still rejected |
| Real-worker browser smoke | **Pass** for the scenarios listed here; the per-scenario coverage map below shows what was verified in the browser versus only in component/backend tests. Sprint 5: pagination, ownership 404, action persistence, real sync 202 → history.list → workers → delayed detail refresh across SPA navigation, repeat-sync idempotency, provider 503 failure → recovery. Sprint 6: canonical list/detail agreement, keyboard set with focus return, zero side effects for manual set and clear (jobs, AI ledger/budget, events, actions, provider calls), AI-after-correction keeps manual fields/revision/timestamp, competing editors (background refetch → 409 → explicit rebase), clear, delayed pre-save read, lost response after commit, failed reconciliation, invalid contract, navigation during pending save, malformed 201 after commit (one POST), delivery evidence (pg-boss 12.31.0, batchSize 1, all fixture email jobs completed), 390px viewport with editor |
| Smoke teardown | Drain tracks handler completion, not socket close. A browser-originated and a Node-originated in-flight write and an active worker all finished before cleanup; residual users/jobs/budgets recorded by the harness as 0/0/0. `SMOKE_INJECT=scenario-failure`: same ordering and 0 residue after a failed run. `http-drain` and `worker-drain`: lane quarantined with fixtures retained, outbound blocking and lane lock held until exit (lane reset afterwards). Re-adding the old socket-close tracking makes the failed-run mode fail |
| Smoke logs | No subjects, bodies, fixture tokens or addresses; 6 fixture AI calls (3 emails × 2), no live provider traffic; browser font requests aborted |

### Coverage map for the runbook §6 scenarios

| Scenario | Real-worker browser smoke | Component / backend tests only | Not covered |
| --- | --- | --- | --- |
| Inherited Sprint 5 path | Sync 202, real workers, delayed mounted refresh, repeat sync with no duplicates or new AI calls | — | Live Gmail/Gemini |
| Canonical reads | User-over-AI and unknown on list and detail; app older than page one | All 64 AI/user combinations (backend) | — |
| Manual set/change | Keyboard set; DB revision and timestamp checked | Revision/no-op/change rules (backend) | — |
| S6-03 delivery | pg-boss version, batchSize 1, every fixture email job completed | Retry and terminal/exhausted outcomes through installed pg-boss (backend) | Retry/terminal in the browser run |
| S6-03 matching | — | Distinct-email, duplicate, stale automatic selection, competing resolutions (backend) | Browser-level matching races |
| AI after correction | Yes | Yes (backend) | — |
| Competing editors | Two pages, background refetch, 409, explicit rebase | Yes (component) | — |
| Clear | Yes, with zero side effects | Yes | — |
| Delayed reads | Detail GET held across the save (the page cancels it) | Uncancelled detail read after acknowledgement; restarted read; dashboard list guard (component) | Delayed **list** GET in the browser |
| Uncertain save | Response dropped after commit; failed reconciliation keeps draft, Save disabled; one PATCH | Timeout, 5xx, invalid 2xx, mismatched ID (component) | — |
| Uncertain creation | Malformed 201 after commit, successful refresh, one POST | Timeout, failed read, same company/role, empty first page (component) | Dropped 201 and failed read in the browser |
| Invalid contract | Detail missing a required field: load error, editing disabled, retry | Invalid PATCH/event bodies; valid null (component) | Invalid PATCH body in the browser |
| Evidence/sections | Recorded / Email date labels; source unavailable | Escaped text; failed history and action sections recover locally (component) | Those failure/escaping cases in the browser |
| Navigation/pagination | Pending save across navigation; 20-item paging; >20 history | Yes (component) | — |
| Accessibility/layout | Keyboard open/select/save/focus return on desktop; 390 px with editor, no overflow | Escape/focus, labels (component) | Keyboard on narrow viewport; non-default themes |

## 5. Architecture boundary review (S6-05 §7)

| Boundary | Result | Evidence |
| --- | --- | --- |
| React/Query → API/services → Prisma/PostgreSQL + pg-boss; no new provider, queue, workflow or infrastructure | Pass | Diff adds one route, one column, client/UI code and test tooling only |
| S6-01 canonical mapper and owner-scoped revision writes; AI writes only AI fields; manual writes create no events/actions/jobs/paid work | Pass | `application-status.test.ts` (64 combinations, revision/no-op/clear, rollback, side-effect counts, AI/manual interleavings); smoke manual interval |
| S6-02 bounded, owner-checked, recording-ordered evidence; no backfill or raw content; D1 reflected | Pass | `application-evidence.test.ts`; events-only per D1 |
| S6-03 delivery matches installed configuration; locks and fresh checks preserve decisions | Pass | `email-worker-reliability.test.ts` (real pg-boss lane), `matching-concurrency.test.ts`; one fixed race |
| S6-04 runtime contracts, frozen drafts, cancellation, uncertain-save recovery, Sprint 5 bounded refresh kept | Pass | `status-correction.test.ts`, `processing-refresh.test.ts`, smoke |
| Additive migration/rollback preserve data, constraints and choices; no business uniqueness | Pass (synthetic) | Upgrade lane; ambiguity test creates/merges nothing |

## 6. Remaining gaps and limits

- **Open gates:** live Gmail incremental/repeat sync, original ~1,620-email preservation, deployed API/worker versions, live Gemini/Discord. Fixture success is not live evidence.
- Sprint 5 Gmail-side reliability work (G1–G4, crash harness, snapshot tool) is absent from this baseline (§1); tracked as proposed, not-started follow-up [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md).
- Accepted limits retained: final-attempt hard kill can leave an email PROCESSING; offset pagination can shift; notification enqueue window; monotonic AI inference; unrestricted date strings; no agenda, rematching, retention policy, action evidence (D1) or scheduled sync.
- Re-estimation of the five tickets and Linear conversion were not performed.

## 7. Review fixes R01–R05 — 2026-10-02

Fixes for the [independent review](review-2026-10-02/README.md); R06–R08 remain open follow-ups.

- **R01:** recent event is read with one `LATERAL … LIMIT 1` query per page, then at most one source email per application. No nested `events: { take: 1 }` load.
- **R02:** the detail guard reads the cache after the response arrives. The dashboard uses the same guarded list fetch. A PATCH response for a different ID is treated as an uncertain outcome and is not cached.
- **R03:** harness drain counts handler completion. A browser-originated write is part of the drain probe. Quarantine keeps outbound blocking and the lane lock. Failed-run and worker-drain modes were added. Unexpected browser hosts fail the run. Residue is recorded.
- **R04:** claims in this report, the runbook and STABILIZATION corrected (coverage map above).
- **R05:** tests now fail if their guarded behaviour is removed. This was checked by temporarily reverting each guarded line, then restoring and comparing files byte-for-byte:
  - frontend: both `retry: false`, the unknown-state block, guard ordering, dashboard guard, ID check;
  - backend: `LIMIT 1`, a queue call during correction, clearing of error fields.

Sprint 6 is technically ready for engineering acceptance pending the open live Gmail and original-data gate. It is not fully accepted, and no DoD checkbox is ticked.
