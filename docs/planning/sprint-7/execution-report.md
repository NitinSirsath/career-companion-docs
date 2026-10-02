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
