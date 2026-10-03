# AD-03 — Prove intake-to-Applications flow and preserve baseline

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED AND VERIFIED LOCALLY |
| ID | AD-03 (local only) |
| Date | 2026-10-03 |
| Size | M |
| Depends on | AD-01, AD-02; existing guarded fixture lanes |

## 1. Objective
Demonstrate that MCP-created/linked applications naturally appear in normal authenticated discovery and usable detail.

## 2. Why
Separate domain or UI mocks cannot alone prove the MCP → PostgreSQL → session API → frontend boundary.

## 3. Current behavior
SDK intake, domain/evidence and full local browser smoke already exist. Extend them for the newly identified discovery gaps; preserve historical Sprint reports.

## 4. Scope
Add real SDK-to-normal-API regression and extend existing integrated browser smoke for filter/sort/refresh/detail-return on automation fixtures. Verify replay, ownership, pending resolution, archive/restore, canonical status and evidence. Record test/build/lint/contract/preservation results and screenshots; close local tickets with evidence.

## 5. Out of scope
No migration commands, personal-data changes, real provider calls, automation-client acceptance, deployment, checkout, commit, push, PR or Linear issues. Do not duplicate MCP-08/09 or Sprint 10 qualification.

## 6. Likely files
Backend mcp-endpoint.test.ts; frontend scripts/smoke-stabilization.mjs and component tests; this planning pack and product/API entry points.

## 7. Implementation notes
Use only existing already-migrated test/smoke databases with assertTestDatabase/exclusive-lane safeguards. Stop if required schema is absent rather than migrate it. Smoke uses real API/DB with synthetic external adapters, outbound blocked and drained teardown. Report any warning or evidence boundary accurately. Compare baseline file hashes; all migrations/schema/MCP runtime files must be unchanged.

## 8. Dependencies
AD-01/02 implemented in order and focused checks passed before this final stage.

## 9. Security/privacy
Only synthetic identities/screenshots. Test two owners, no bearer access to normal API, no source secrets in responses, no reads added to tools/list. No credentials in evidence logs.

## 10. Acceptance criteria
- [x] SDK tool records a submission, then normal owned list/search/sort/detail/history retrieve its single application.
- [x] Replay creates no duplicates; linked submission preserves existing status/date; review-only records wait for resolution.
- [x] Foreign sessions cannot discover/read another user's application; archived applications require explicit visibility and retain receipt/history on restore.
- [x] Browser finds real fixture records through controls, opens detail and returns with context, refreshes new intake and displays truthful source/date/status.
- [x] Full backend/frontend suites, builds/typechecks, lint and contract sync pass, or exact blockers are reported without claiming completion.
- [x] Existing source paths, schema/migration hashes and unrelated work remain intact; no staged/committed/remote changes.
- [x] Desktop/mobile screenshots reviewed; smoke teardown reports zero fixture residue.
- [x] Local evidence is explicitly separate from deferred real-world acceptance.

## 11. Testing
Run focused tests per ticket, then full suites once final changes are ready. Repeat only when failures/new edits justify it. Use the existing Sprint 11 smoke lane; do not rerun migration suites.

## 12. Documentation
Execution report, incremental changed-file manifest, API behavior and product flow; add pack to planning index. Mark each ticket implemented locally only after its criteria pass.

## 13. Definition of done
Reviewable local changes and evidence meet these criteria. Deferred live-client/provider/owner/migration checks remain explicitly pending.


Evidence: [local execution report](execution-report.md), [verification outputs](evidence/). Checked criteria represent local engineering and synthetic fixture evidence, not real-client, owner or provider acceptance.
