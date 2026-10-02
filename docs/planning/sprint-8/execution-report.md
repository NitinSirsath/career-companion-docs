# Sprint 8 office implementation — 2026-10-03

Sprint 7 foundations passed local checks (see its execution report). GitHub write access, remote CI and real-day scheduled-use evidence remain pending. The owner authorized implementation independently of personal-Mac verification.

S8-01: ADR-0003 is proposed and committed before code. The owner explicitly approved keeping later thread mail unmatched after unlink on 2026-10-03; ADR-0003 is accepted before correction code. The current matcher lacks the documented S6-R07 rematch guard; this is a concrete correction prerequisite, not grounds to reopen unrelated Sprint 6 work. S8-02 was completed independently while that decision was pending.

## S8-02 — stuck-processing recovery

Uses a server-computed optional `processingStuck` flag after 20 minutes, with no timestamp exposure or row reset. The threshold clears queue expiry/detection/backoff and the 15-minute held-AI approval boundary. The actual start update moves updatedAt, verified against PostgreSQL. Manual Retry is visible for stopped rows even without error details and reuses the existing retry/charge-approval path. This also supplies the small missing AI-20 visible-retry prerequisite. Polling is 2 seconds only in the existing bounded refresh window, then 15 seconds only for server-confirmed live PROCESSING rows; stopped or legacy responses end polling. No automatic recovery or new queue was added.

Frontend tests cover no-details recovery, live-to-stopped polling and missing-field compatibility. Existing retry uncertainty/approval tests remain green. No migration.

S8-02 final local checks: backend 52 files / 718 tests; frontend 16 files / 175 tests; both typechecks/builds/lint passed. Browser smoke passed with 14 fixture AI calls, outbound blocked and zero users/jobs/budgets/MCP residue. Existing fixture charge-approval and lost-response flows passed with the visible retry button. Real killed-email-provider behavior remains outside this ticket's fixture acceptance.

## AI-15 — evaluation runner implemented; live certification blocked

The runner separates provider refusals from quality, waits at most three times for rate limits, stops on persistent/access refusal, emits PASS/FAIL/INCONCLUSIVE and rejects non-passing baselines. No repeat of unknown/invalid outcomes. Reports cannot overwrite prior evidence. 26 focused evaluation tests pass. Real key unavailable; AI-19 SDK/disclosure prerequisites are absent, so no live call, provider promotion, disclosure version change or mapping claim was made. The provider-evaluation document records the current official retirement/access and terms check. Other providers remain hidden.

Verification correction: the scheduler fixture had one prefer-const lint error that earlier shell command chains did not propagate. It is fixed in a dedicated test commit and lint was rerun with failure propagation. Earlier milestone lint statements should be read with this correction; test/build counts were unaffected.

AI-15 runner milestone: full backend 52 files / 724 tests, typecheck/build and corrected lint pass (7 existing warnings, no errors). Evaluation remains uncertified without live provider evidence. No migration or catalog change.

## S8-01 — correction and retained history

Implemented the accepted ADR with one owner-scoped transaction and lock order (user, email, sorted applications). Move/unlink retires old email effects; moving back reactivates the existing target rows without reopening handled actions. Source AI status is recomputed from remaining matched evidence; manual status fields and automation submissions stay intact. Correction calls no Gmail, AI or enqueue function. Retired actions are excluded from all active lists/counts and notification processing and reject updates with 409. Automatic replay preserves a completed match, and later-thread decisions are rechecked under the user lock; an accepted unlink prevents company fallback.

Migration `20261003010000_retire_match_effects` adds paired nullable retirement fields with consistency constraints and no backfill. Guarded fixture migration and all three fresh/upgrade preservation lanes passed. Backend full suite: 53 files / 734 tests before the two additional lock-interleaving cases; those focused cases are recorded below. Frontend: 17 files / 182 tests. Typechecks, builds and lint pass (backend 7 existing warnings); synchronized contracts match. Both specified mutations were observed failing and restored: removing the thread recheck and removing the active-only pending-action count.

The shared dialog freezes the expected match on open, uses a paged owned-application picker, blocks duplicate saves, and refetches after conflicts or uncertain responses. Gmail exposes current application/confirmation and Change link; detail history labels retired effects and offers Wrong application on active evidence. The browser smoke passed move → unlink → restore through the real API/queue/database, zero extra AI calls, preserved status revision and automation event, outbound blocked and zero residue. Correction entry was exercised by keyboard; native select/button interactions were automated, so this does not claim a complete manual keyboard-only session. The first smoke failed because the picker test clicked Next before loading completed; the harness now waits for the actual list response. No product workaround was required. Live daily-use acceptance and remote CI remain pending.

Additional correction lock tests: 11 focused cases passed, including a real correction holding application locks while status and MCP intake writes wait; `pg_blocking_pids` proves each wait before release. Both writes finish, the AI status is recomputed, the manual revision advances correctly and the automation event remains active. Typecheck/lint passed after these additions.
