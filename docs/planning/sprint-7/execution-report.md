# Sprint 7/8 office implementation — 2026-10-02

## Baseline and authority

The owner authorized Sprint 7 then Sprint 8 implementation from GitHub, independently of personal-Mac live acceptance. Downloads artifacts are excluded. Existing clean shallow Git clones were discovered in the office laptop's `/private/tmp/claude-501/` scratch workspace, unshallowed and fast-forwarded from the verified public `NitinSirsath/career-companion-{backend,frontend,docs}` remotes. Initial heads: backend `78ac1ee`, frontend `64f14a4`, docs `3a1c8c5`. Work uses `feat/sprint-7-8-reliability` in each existing checkout. No archive files were copied and no history was reconstructed.

GitHub reports `push: false` for the current account. Local work can proceed; publishing and green-on-main evidence are blocked pending repository write access. No live acceptance is inferred from local tests.

## Assessment and execution decisions

- Architecture: Express/session ownership boundary, Prisma/PostgreSQL domain state, pg-boss workers in the API process, Gmail history with a bounded fallback scan, durable per-operation AI claims, per-user BYO credentials/limits, MCP intake, synchronized Zod contracts, and a Carbon-inspired React UI.
- Sprint 6/BYO/MCP implementation is present in GitHub. Their migration/live acceptance remains separate. Existing gaps are checked against source, not assumed fixed by old reports. Closeout work will be touched only when a concrete prerequisite blocks this scope.
- Reuse the queue, sync entrypoint, operation ledger, matcher transactions, notification delivery claim, ownership constraints and existing UI primitives. No new providers, queue system or architecture replacement.
- S7-01 first: Node 24, CI with disposable PostgreSQL 15, all three migration lanes, and a frontend contract comparison against public backend main. Local baseline checks establish actual results before completion claims.
- S7-02: fixed per-attempt window covering the last checkpoint plus overlap, at most 30 days, with a persistent capped-gap notice. Then S7-03: explicit deadline parsing anchored to receivedAt; preserve date-only precision.
- S7-04: bounded worker registration retries and a separate in-memory readiness endpoint. Then S5-FU-01: attempt fencing and real crash/redelivery tests. S7-05 only after those boundaries: 00:00/18:00 Asia/Kolkata, durable catch-up and existing per-user enqueue/claim handling.
- Sprint 8 follows the implemented Sprint 7 foundations: ADR-0003 and audited correction with retired effects, stuck-state recovery and bounded polling, Gemini evaluation/error accounting without uncertified catalog promotion, and bounded Google transports. The owner's request authorizes implementation now; two-day scheduled use, longer daily-use feedback, live Gemini certification and disclosure approval remain acceptance evidence, not fabricated results.
- Risks to test: stale leases/disconnect during writes; checkpoint loss; ownership and replay; received-date year boundaries; date-only display; partial worker startup; schedule overlap; correction vs later thread mail; held paid-call uncertainty; migration preservation.

## Evidence

Baseline passed: backend 43 files / 621 tests; frontend 15 files / 155 tests; both typechecks and builds; lint 0 errors (11 backend warnings, existing frontend warnings). All three fresh/upgrade lanes passed on newly created databases, including the synthetic 1,620-email preservation lane. This is fixture evidence only. Node 24.20.0 and PostgreSQL 15.19 are available. Both lockfile installs passed. Test databases will be newly created dedicated local fixture databases; no existing user database is reset or reused.

### S7-01 — implemented locally; remote acceptance blocked

Added backend test and migration-lanes jobs, frontend check and contract-drift jobs, Node 24 pins and engines, fixture env template, expanded script lint, and frontend package/tab naming. Current public remotes and default branches were verified through GitHub API. Official Actions releases are checkout v7.0.1 and setup-node v7.0.0; workflows use v7. The lockfiles changed only root metadata. Fixed script-local crypto name collisions found by expanded lint and removed raw transport error text from the MCP diagnostic's connect failure. Retained the unrelated tracked placeholder script instead of deleting it.

CI is configured but has not run on GitHub; no green-on-main claim, branch protection change or live-provider test. README rollout claims remain unchanged pending publication. Browser smoke remains manual and requires Chrome, both builds and a dedicated smoke database. The backend build still includes tests and the guard's dist/utils/testDatabase module.

### S7-02 — implemented and locally verified; live evidence pending

Added migration `20261002160000_gmail_unscanned_gap` (two nullable columns, no backfill). Backend 44 files / 633 tests; frontend 15 files / 158 tests; both typechecks/builds and lint (0 errors) passed. Focused Gmail tests: 53 backend, 11 UI. All three fresh/upgrade migration lanes passed after the migration. A five-day gap regression failed before the fix (four-day-old mail was missing); restoring the old age filter also failed it. Both were restored and the passing implementation retained.

The real API/PostgreSQL/pg-boss/browser smoke passed with fixture Gmail/AI, 14 AI calls, outbound blocked and residue 0. Its first run exposed an outdated pagination step: fixtures are IRRELEVANT but the newer UI defaults to Job Related. The harness now selects Irrelevant before asserting its unchanged 20/6 pagination counts; no product behavior or assertion was weakened.

Gap notices persist across successful uncapped scans and are replaced by newer capped scans. Reconnect uses the old checkpoint. No historical user records were rewritten. Live Gmail gap evidence and remote CI remain pending.

### S7-03 — implemented and locally verified

Chose the additive `DeadlinePrecision` enum/nullable action field rather than a midnight convention. Migration `20261002170000_action_deadline_precision` does not backfill legacy rows. Parsing accepts the documented ISO and English month-name forms, infers missing years from receivedAt with one day of tolerance, and rejects ambiguous/impossible/past-at-receipt text. No AI prompt or extraction contract version changed. All three action API mappings, both UI displays, overdue grouping and Discord preserve precision.

45 parser cases passed under both America/Los_Angeles and Asia/Kolkata. Focused backend: 99 tests, including matcher replay, follow-up, unclear logging and all three action routes. Frontend focused: 37 tests under Los Angeles, plus the application detail date-only assertion. Full backend: 46 files / 682 tests; frontend: 16 files / 166 tests; typechecks, builds and lint passed (existing warnings only). All three fresh/upgrade lanes passed with the new migration. Existing user actions were not read or rewritten.

### S7-04 — implemented and locally verified

Bounded startup retries only failed registrations (2, 4, 8, 16, 30, 30, 30 seconds), then exits 1. Health and readiness precede sessions/CORS. Readiness checks in-memory pg-boss worker state and never starts a queue or queries PostgreSQL. Driver diagnostics omit messages/stacks. Shutdown suppresses subsequent retries and give-up exit.

Focused worker/readiness compatibility tests passed; final backend suite 47 files / 689 tests, typecheck/build/lint passed. A real backend child process behind an initially closed local TCP proxy returned health 200/readiness 503, then registered workers and returned readiness 200 when the proxy opened to the fixture PostgreSQL database. A separate exhausted-startup child exited 1 (waits injected to zero; backoff duration separately unit-tested). The existing PostgreSQL service was never stopped. This is local fixture startup evidence, not provider/live acceptance. Registration readiness does not detect a later database outage.

### S5-FU-01 — crash recovery and ownership fencing

Each delivery keeps a stable request ID and acquires a distinct attempt claim. Short PostgreSQL row-lock transactions use database time, a 2-second lock timeout, 5-second statement timeout and 10-second transaction timeout. Email inserts and checkpoint finalization recheck the connected owner and live lease in the same transaction; external requests and queue sends stay outside. Pending recovery checks ownership before each send. Busy attempts retry; obsolete/disconnected attempts terminate without a completion event. Delivery events contain allowlisted identifiers/counters and explicit checkpoint certainty. Queue enqueue failure now has a typed 503 response.

The real crash harness uses an independently confirmed, empty local `*_crash_test` database and advisory lock. It SIGKILLs a child after acquisition, ages only the fixture job and lease, runs pg-boss supervision, observes the same job at retryCount 1, and verifies two starts/one completion, one checkpoint and no duplicate mail on repeat sync. Two runs passed with zero fixture residue; CI includes the same two runs. This is real process/queue/database evidence with synthetic Gmail, not live Gmail evidence. A blocked row lock was observed through `pg_blocking_pids` and failed within its bound. Tests also cover concurrent disconnect/claim replacement/expiry during Google I/O, the four-minute budget, retry identity, bounded pending recovery, and production auth classification.

Final milestone checks: backend 49 files / 707 tests, typecheck/build/lint passed (7 existing warnings, no errors), diff whitespace clean. No new migration. Remote CI remains pending repository write access.

### S7-05 — implementation decision

OD-05 is resolved by the owner's explicit request: 00:00 and 18:00 Asia/Kolkata, one startup catch-up and durable missed-slot recovery. Reuse the existing claim path and pg-boss scheduler, with cron-parser pinned to its installed version. Manual "Sync to continue" copy remains accurate; the Gmail page adds the next automatic time. The backend must remain running for on-time execution; stopped/asleep periods are caught up when it resumes. Two real-day observation remains an acceptance gate.

S7-05 implemented locally. Backend 52 files / 716 tests, frontend 16 files / 170 tests; both typechecks/builds/lint passed (existing warnings only). Real pg-boss registration tests verify one schedule, one startup job and one worker, inclusive latest-slot edges, disabled removal and readiness. Fan-out tests cover busy/recent/revoked/disconnected users, duplicate runs and continuing after a failed user. Frontend fake-timer evidence verifies a status refresh about one minute after the slot with no hot loop. Contract sync changed one file, then zero. `npm ls cron-parser` resolves one deduplicated 5.10.0. Browser smoke passed: real API/PostgreSQL/queue, fixture Gmail/AI, 14 synthetic AI calls, outbound blocked, all residue counts zero. No Prisma migration for scheduling. Two real days, catch-up while asleep, live Gmail and green remote CI remain unverified.
