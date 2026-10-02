# S6-R04 — Correct Sprint 6 verification claims

| Field | Value |
| --- | --- |
| Status | **Fixed — verified locally 2026-10-02** (local ticket; documentation only) |
| Severity / priority | Medium / fix before Sprint 6 acceptance |
| Source | [Review 2026-10-02](README.md) M4, section E |
| Repository | career-companion-docs; backend `STABILIZATION.md` |

## Problem
- `docs/planning/sprint-6/verification-runbook.md` §8 (line ~187) says the browser smoke passed "all scenarios in the §6 table". It did not cover these rows in the browser:
  - **S6-03 matching:** covered by backend tests only.
  - **Delivery retry/terminal:** the smoke shows only completed jobs; retry and terminal outcomes are in backend tests.
  - **Delayed list GET:** only a detail GET was held.
  - **Uncertain creation:** dropped 201, failed read, same-company and first-page-miss cases are in component tests only.
  - **Invalid 2xx PATCH body:** component tests only.
  - **Escaped text and failed history/action sections:** component tests only (no action-section failure test at all).
  - **Themes, and keyboard on the narrow viewport:** not covered.
- `execution-report.md` §4 and `STABILIZATION.md` call the upgrade comparison "byte-identical". It is equal SHA-256 digests of JSON-serialized rows.
- "0 residue" came from a manual `psql` after the run, not from the harness.

## Fix
Rewrite the §8 row and execution report §4 so each §6 scenario is marked "browser smoke", "component/backend test" or "not covered". Change "byte-identical" to "identical by row-digest comparison". State that residue was checked manually (until S6-R03 records it). Keep live Gmail and original-data evidence as an open gate.

## Acceptance criteria
- [ ] Every claim in runbook §8, the execution report and STABILIZATION maps to an actual test, smoke step or manual check.
- [ ] No Sprint 6 DoD checkbox is ticked while the entry gate is open.

## Verification
Cross-check each claim against `scripts/smoke-stabilization.mjs` and the test files. Check that links still resolve.

## Resolution — 2026-10-02

Fixed (documentation only):
- **`sprint-6/execution-report.md`:** added a per-scenario coverage map (browser smoke vs component/backend tests vs not covered), corrected "byte-identical" to row-digest equality, updated teardown and test counts, and added §7 recording these fixes.
- **`verification-runbook.md` §8:** no longer claims "all scenarios"; links the coverage map; records residue as harness-measured.
- **Backend `STABILIZATION.md`:** digest wording, handler-completion drain, `SMOKE_INJECT` modes, residue and browser allowlist.
- **Frontend `README.md`:** points to the coverage map.

No DoD checkbox ticked; the live Gmail/original-data gate is still open.
