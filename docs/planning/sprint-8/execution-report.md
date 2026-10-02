# Sprint 8 office implementation — 2026-10-03

Sprint 7 foundations passed local checks (see its execution report). GitHub write access, remote CI and real-day scheduled-use evidence remain pending. The owner authorized implementation independently of personal-Mac verification.

S8-01: ADR-0003 was proposed and committed before code. The owner explicitly approved keeping later thread mail unmatched after unlink on 2026-10-03; ADR-0003 is accepted before correction code. At kickoff the matcher lacked the documented S6-R07 rematch guard; it was implemented as the concrete correction prerequisite, without reopening unrelated Sprint 6 work. S8-02 was completed independently while that decision was pending.

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

## S5-FU-02 — bounded Google calls and cancellation

Recorded OD-15 before coding under the owner's explicit engineering delegation. Added nullable access-token expiry with no backfill/index; all refresh token/expiry saves retain compare-and-swap protection. Known-expiry quota 403 does not refresh/revoke; known-expiry 401 revokes. Legacy null-expiry rows keep bounded reactive refresh until expiry is saved. One Google factory configures 15 s data / 10 s OAuth / 5 s revoke bounds and disables SDK retries. Original-signal interceptors also prevent SDK-internal work after cancellation; pre-call checks alone cannot cover that installed gaxios edge case. No Google URL override exists in production code.

Sync uses one absolute 240 s signal combined with the job's cancellation signal, rechecks after obtaining owned database locks, preserves checkpoints on failure and classifies the first signal reason. Its budget is tested to be at least 30 s below queue expiry. Email workers pass cancellation only to Gmail fetches. Raw provider errors are reduced to status and one allowlisted reason; fixed terminal categories cover cancellation, deadline, request timeout, network and quota vs forbidden. Google error documentation was checked for the three accepted 403 quota reasons (source linked in the ticket).

28 real installed-SDK loopback tests passed. The seven focused compatibility suites passed 133 tests before the final pipeline-signal test. Final full backend is 54 files / 767 tests, frontend 17 files / 182 tests. Typecheck/build/lint pass (7 existing backend warnings and existing frontend warnings; no errors). Contract synchronization changed zero files. Guarded migration and all three fresh/upgrade lanes passed with `20261003020000_gmail_access_token_expiry`. Browser smoke passed after transport changes with real API/PostgreSQL/pg-boss, fixture Gmail/AI, 14 AI calls, outbound blocked and zero users/jobs/budgets/MCP residue. No live provider result is inferred.

## Remaining acceptance and handoff

- Publishing: current GitHub account `nitin-stockypro` has `push: false` on all three NitinSirsath repositories, rechecked after implementation. No push/PR/remote CI run is claimed. Authenticate an account with repository write access, publish the local branches, land backend before the frontend contract-drift check against backend main, then verify CI on main.
- Gemini: evaluation runner is delivered, but both two-run model pairs, supervised error rows, corrected AI-19 SDK/disclosure prerequisites and owner-approved disclosure remain pending. No real evaluation key is available. All providers remain hidden; no new model or fabricated certification report.
- Live acceptance: two real scheduled days, live Gmail gap/repeat behavior, original personal-Mac dataset preservation and deployed readiness are unverified. Fixture databases were newly created and guarded; original/user data and Downloads artifacts were untouched.
- Local source: the existing Git checkouts under `/private/tmp/claude-501/-Users-spurge-rental-Downloads-docs/8472a241-e50b-41f8-9bcf-4a75ed0fd02a/scratchpad/`, each on `feat/sprint-7-8-reliability`. They are full-history clones synchronized from the public GitHub remotes; these exact paths are temporary storage and should be retained until publishing succeeds.

No implementation scope was added for OpenAI/Anthropic, deployment, retention, push Gmail, manual migration repair, expanded telemetry or unrelated Sprint 6 closeout. Concrete prerequisites delivered were the automatic-rematch guard and the visible existing retry entry point. Remaining warnings are documented baseline lint/Vite configuration/bundle warnings, not test failures.

Final regression confirmation: after the last cancellation/checkpoint review, the full 767-test backend suite passed again. The final browser smoke passed with 14 fixture AI calls and all residue counts zero. The real SIGKILL/same-job-redelivery harness also passed again with zero residue against its own guarded database after the new migrations.

## Concurrent GitHub updates reconciled

At the final remote check, frontend main had advanced from `64f14a4` to `fcfb5f7` and docs main from `3a1c8c5` to `047eb62`; backend main stayed `78ac1ee`. Inspected all incoming changes, then merged them normally into the implementation branches without conflicts. Frontend merge `155a1aa` preserves `a51a0d4` (AI-settings refresh/copy only when waiting emails exist); docs merge `3244090` preserves the personal-PC migration ledger. No history rewrite, cherry-pick recreation, archive copy or personal-Mac access was used. Implementation heads before these merges: backend `6af9cd2`, frontend `5c93300`, docs `ae990de`.

Merged-branch verification: frontend contracts unchanged, typecheck/lint/build pass, all 182 tests pass. The browser smoke passed again after the GitHub merge, including S8 move/unlink/restore and the preserved AI-settings behavior: 14 fixture AI calls, outbound blocked and every residue count zero. Backend remains at the fully checked 767-test commit `6af9cd2`. All implementation changes are committed locally; publication is blocked only by repository write access.
