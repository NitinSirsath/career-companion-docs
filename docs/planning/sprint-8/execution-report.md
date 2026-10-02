# Sprint 8 office implementation — 2026-10-03

Sprint 7 foundations passed local checks (see its execution report). GitHub write access, remote CI and real-day scheduled-use evidence remain pending. The owner authorized implementation independently of personal-Mac verification.

S8-01: ADR-0003 is proposed and committed before code. The owner explicitly approved keeping later thread mail unmatched after unlink on 2026-10-03; ADR-0003 is accepted before correction code. The current matcher lacks the documented S6-R07 rematch guard; this is a concrete correction prerequisite, not grounds to reopen unrelated Sprint 6 work. Independent S8-02 work proceeds while the policy is reviewed.

## S8-02 — stuck-processing recovery

Uses a server-computed optional `processingStuck` flag after 20 minutes, with no timestamp exposure or row reset. The threshold clears queue expiry/detection/backoff and the 15-minute held-AI approval boundary. The actual start update moves updatedAt, verified against PostgreSQL. Manual Retry is visible for stopped rows even without error details and reuses the existing retry/charge-approval path. This also supplies the small missing AI-20 visible-retry prerequisite. Polling is 2 seconds only in the existing bounded refresh window, then 15 seconds only for server-confirmed live PROCESSING rows; stopped or legacy responses end polling. No automatic recovery or new queue was added.

Frontend tests cover no-details recovery, live-to-stopped polling and missing-field compatibility. Existing retry uncertainty/approval tests remain green. No migration.

S8-02 final local checks: backend 52 files / 718 tests; frontend 16 files / 175 tests; both typechecks/builds/lint passed. Browser smoke passed with 14 fixture AI calls, outbound blocked and zero users/jobs/budgets/MCP residue. Existing fixture charge-approval and lost-response flows passed with the visible retry button. Real killed-email-provider behavior remains outside this ticket's fixture acceptance.

## AI-15 — evaluation runner implemented; live certification blocked

The runner separates provider refusals from quality, waits at most three times for rate limits, stops on persistent/access refusal, emits PASS/FAIL/INCONCLUSIVE and rejects non-passing baselines. No repeat of unknown/invalid outcomes. Reports cannot overwrite prior evidence. 26 focused evaluation tests pass. Real key unavailable; AI-19 SDK/disclosure prerequisites are absent, so no live call, provider promotion, disclosure version change or mapping claim was made. The provider-evaluation document records the current official retirement/access and terms check. Other providers remain hidden.

Verification correction: the scheduler fixture had one prefer-const lint error that earlier shell command chains did not propagate. It is fixed in a dedicated test commit and lint was rerun with failure propagation. Earlier milestone lint statements should be read with this correction; test/build counts were unaffected.

AI-15 runner milestone: full backend 52 files / 724 tests, typecheck/build and corrected lint pass (7 existing warnings, no errors). Evaluation remains uncertified without live provider evidence. No migration or catalog change.
