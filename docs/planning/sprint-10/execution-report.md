# Sprint 10 engineering execution — 2026-10-03

Status: local engineering implemented and verified. Live v3 qualification and owner acceptance remain deferred; extraction/v3 remains default-off.

## Reconciled baseline

Authoritative checkout: the existing scratchpad supplied by the owner. Backend `6af9cd2`, frontend `155a1aa`, docs `1e98ae4`, all on `feat/sprint-7-8-reliability`. Sprint 9 is intentionally uncommitted: backend 6 modified/4 untracked entries, frontend 21 modified/3 untracked entries; docs 5 modified/6 untracked entries. Nothing staged at kickoff. All 11 shared contracts match.

The complete migration ledger was read: polling/copy fix `a51a0d4` is already present through merge `155a1aa`; no reapplication is needed. Downloads copies lack Git metadata and are not used. No migration artifact has been proved disposable; no cleanup is performed.

Eight existing S10/S11 tickets are retained. Sprint 9 verification (777 backend/195 frontend tests and smoke) is recorded prior evidence, not a rerun here. Manual owner, dummy Gmail and MCP end-to-end acceptance are deferred. No real database is a test target.

## Decisions

Accept ADR-0005/OD-18 for local implementation. Keep extraction/v2 contract immutable, select v3 once under the email row lock, and preserve selection independently of runtime enablement. Temporal input includes receivedAt; source timezone remains explicit. Candidate projection and user edits are separate from action/status behavior. New v3 selection is default-off until live qualification.

## Delivered and verified

S10-01: retained v2 contract, explicit receivedAt v3 input, at most five validated candidates, bounded source-verified quotes, deterministic DATE/DATETIME/UNRESOLVED normalization, durable version selection under the email lock. Disabling the flag preserves selected operations and user decisions.

S10-02: additive candidate envelope/AgendaItem migration, owner guards and immutable suggestion, SQL-filtered bounded agenda pages, revisioned user timing/state, correction retirement/reactivation with target intent preserved. New targets carry prior decisions with provenance. No automatic cross-email reschedule, action creation or external work.

S10-03: list agenda and confirmation dialog, explicit timezone/date precision, separate review/history, original evidence, uncertain-save reconciliation without replay, navigation and query invalidation.

S10-04 local portion: extended the existing evaluator with `--extraction-version extraction/v3`, versioned synthetic temporal cases and zero-error temporal scoring. Missing/refused runs stay INCONCLUSIVE. No provider was called for qualification; two complete model-specific runs and activation approval remain pending.

| Verification | Result |
| --- | --- |
| Full backend suite | 57 files / 794 tests passed before two final added regressions |
| Final backend focus | 20 tests passed, including actual v3 provider-boundary fixture and observed edit/correction lock interleaving |
| Full frontend suite | 18 files / 201 tests passed |
| Builds/typechecks | Passed |
| Lint | Passed; existing warnings only (removed new Agenda export warning) |
| Fresh/additive preservation | Agenda, status, AI and MCP suites passed on eight newly created guarded fixture databases |
| Browser | Real local API/DB/queue with synthetic providers: confirmation, timed edit, move/unlink/restore, desktop and 390px; zero extra provider calls during agenda flow |
| Cleanup | Smoke drained; users/jobs/budgets/MCP residue 0/0/0/0 |

Evidence is in [evidence](evidence/). The backend target is `.env.sprint10.test`, a new disposable database; the existing `.env.test` and personal database were not changed. Schema rollback retains additive columns and v3 reconciliation; disable new selection with `AGENDA_EXTRACTION_V3_ENABLED=false`, never downgrade to a binary unaware of selected v3 operations.

## API and coverage

GET /api/agenda uses view=upcoming|past|review|history, validated timeZone, optional paired from/to calendar bounds (inclusive/exclusive, <=90 days), optional applicationId, limit<=20 and offset. Review rejects date bounds. Upcoming/past use actual instant or calendar date; history orders by recording time. PATCH /api/agenda/:id accepts expectedRevision, optional state and optional {date,time,sourceTimeZone}. Missing/foreign=404; stale/retired=409; unresolved confirmation=400. Revision is checked before no-op. History and original suggestions remain available.

Extraction/v3 inputs contain {receivedAt,body}; the body remains transient and bounded to 8000 characters. Stored candidates have an agenda/v1 envelope and stable index keys. No legacy backfill. No real provider, dummy Gmail, live MCP, owner trial, remote CI or deployment acceptance is claimed.
