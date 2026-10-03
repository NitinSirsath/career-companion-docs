# Final Sprint 10/11 engineering review — 2026-10-03

## Outcome and scope

Eight existing tickets were reviewed and used; no duplicates or external tickets were created. S10-01..03 and S11-01..04 are locally implemented and verified. S10-04's local dataset/scorer/rollout safeguards are complete; real-provider qualification and activation remain pending. Sprint 10 verification preceded Sprint 11 implementation. This is an engineering closeout, not product-wide or live acceptance.

The authoritative checkout remains `/private/tmp/claude-501/-Users-spurge-rental-Downloads-docs/8472a241-e50b-41f8-9bcf-4a75ed0fd02a/scratchpad`. Migration reconciliation was completed first; the polling fix and 11 original shared contracts were already present. All 13 current shared contracts match. No new checkout was used.

## Review findings resolved

- Existing held/pending extraction versions must win over a changed runtime flag; select once under the email lock and adopt completed v2/v3 results without fetching mail or calling providers.
- Correction must preserve intent after unlink, including when moving to a previously unseen target. Carry the most recent retired decision only for new target rows; existing target decisions win. Revisions increase on retirement/reactivation.
- Archive must serialize with MCP linking without combining two advisory lock namespaces. Recheck the shared application row lock, route archived-only/stale candidates to review, and preserve existing receipt identity.
- Suppression must survive restore/expiry and delayed queue delivery. Persist terminal suppression and recheck claimed work before send; acknowledge the already-in-flight external-send limitation.
- Archive-aware list keys initially broke uncertain application creation recovery. Reset filters/page and fetch the exact visible active-list key. A regression test and real API browser recovery verify this.
- Archived status editing previously displayed a misleading data-load error. Archived detail now exposes restore and history without the status editor. The browser asserts that archive is not presented as a load failure.
- The inherited browser revocation test raced its DELETE acknowledgement; it now awaits the response. Follow-up smoke navigates the real second page instead of assuming the newest undated item is on page one.

The primary agent reviewed changes directly; no delegated review, remote CI, security certification or deployment is claimed. No unresolved local engineering defect was found by the completed checks.

## Evidence and files

[S10 report](../sprint-10/execution-report.md), [S11 report](execution-report.md), [all files changed from kickoff](change-manifest.md), [test/build/migration/browser outputs](evidence/). Backend: 58 files / 806 tests. Frontend: 19 files / 209 tests. Both builds/typechecks/lint pass with existing lint warnings. Four fresh/upgrade migration suites passed after each sprint; local browser smoke passes with isolated fixtures and zero residual users/jobs/budgets/MCP rows.

Tests added: backend agenda, temporal scoring, follow-through; frontend agenda and follow-through. Tests extended: v3 pipeline, notification suppression, workspace counts, contract fixtures, action/application UI, status/creation recovery and integrated browser scenarios. Existing full regression suites remain green.

Documentation updated: retained S10/S11 tickets and indexes, execution reports/evidence, project constitution's accepted bounded decisions, ADR-0005, ADR-0002 archive extension, architecture, acceptance register, product user flows and current planning entry points. Historical S9 execution report and migration handoff hashes are unchanged.

## Preservation and Git state

Backend HEAD `6af9cd2`, frontend `155a1aa`, docs `1e98ae4`; all remain on `feat/sprint-7-8-reliability`. Nothing staged, committed, pushed or published. All initial baseline paths remain; all existing migration file hashes are unchanged. Sprint 9's intentionally uncommitted implementation remains integrated, with scoped changes to shared workspace/contracts/cache for S10/S11. No tracked files or personal data were deleted. No personal database was migrated; tests use explicitly guarded disposable fixture lanes. No Linear action or PR was created.

## Remaining validation layers

| Layer | State |
| --- | --- |
| Automated engineering verification | Passed locally |
| Controlled browser verification | Passed; real local stack, synthetic external boundaries |
| Dummy Gmail end-to-end validation | Deferred; not performed |
| Real MCP/automation-client acceptance | Deferred; SDK fixture coverage is separate |
| Owner real-world acceptance | Deferred |
| extraction/v3 live qualification | Pending; default-off; two real configuration-specific runs and prerequisites/activation decision required |
| Personal-data preservation, real scheduled days, remote CI/deployment | Pending or out of scope, as recorded in the existing acceptance register |

Do not activate v3 or claim production validation from these fixture results. The next owner-authorized phase can use the existing real-world validation plan; it has not begun here.
