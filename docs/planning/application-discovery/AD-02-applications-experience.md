# AD-02 — Finish the existing Applications experience

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED AND VERIFIED LOCALLY |
| ID | AD-02 (local only) |
| Date | 2026-10-03 |
| Size | M |
| Depends on | AD-01; existing detail, S9 cache guards, S11 controls |

## 1. Objective
Let users browse, find, inspect and continue work on any recorded application, including automation submissions.

## 2. Why
Adding another Applied tab fragments the lifecycle. The existing list needs discovery controls and recovery/context continuity.

## 3. Current behavior
Applications cards link to usable detail controls. Search/status/archive exist but raw selects are inconsistent, sort/source/unknown facets are missing, filters reset after a detail visit and read errors lack a direct retry.

## 4. Scope
Use existing route/navigation. Add NativeSelect sort/submission/status/visibility controls, clear filters, explicit refresh/retry, contextual empty states, existing Automation review link, unknown-date copy and ongoing automation provenance. Keep discovery state in the Applications parent across detail visits, reset offset on filter/sort changes. Wire all parameters into API and query keys. Preserve status/archive revision guards for UNKNOWN membership.

## 5. Out of scope
No new Applied navigation, bulk editing, analytics, saved/URL searches, unrelated layout redesign, new detail actions, frontend MCP reads or fake totals.

## 6. Likely files
frontend routes/applications.tsx, components/ApplicationStatus.tsx, api/client.ts, lib/applicationCache.ts, contracts/application.ts, tests/applications.test.tsx and status/cache tests.

## 7. Implementation notes
Use semantic tokens, square NativeSelect/Input/Button/Badge, labeled controls and responsive grids. Preserve existing “Applied · via automation” unknown-state convention; show separate source text when canonical status exists. UNKNOWN option reads “No user/AI status” to distinguish evidence. Added and Applied remain distinct dates. Parent state must survive child detail; private search text stays out of URL/storage. No automatic POST retries. Creation recovery must reset to the actual visible default list key including new filters.

## 8. Dependencies
AD-01 shared schemas; current detail/timeline and Automation review behavior reused.

## 9. Security/privacy
Text-only escaped content, no secrets/answers/receipt IDs in new UI. Never assume that automation intake confirms AI/user status. All reads use normal API and AbortSignal.

## 10. Acceptance criteria
- [x] All filters/sort are accessible, bounded, reset page and participate in request/cache keys.
- [x] Opening detail and returning preserves search/filter/sort/page.
- [x] Unknown/known statuses retain truthful automation provenance and work across later lifecycle stages.
- [x] Empty active/archive/filter states guide recovery; read failure retains controls and can retry.
- [x] Refresh discovers external intake while page is open; superseded responses do not replace current results.
- [x] Existing manual creation, uncertain creation recovery, archive/restore, status correction and pending-submission selection work.
- [x] Desktop/mobile remain readable with no horizontal overflow; keyboard labels/focus use existing primitives.

## 11. Testing
Component tests for query combinations, sort/page reset, rapid search, detail return, clear/retry/refresh, UNKNOWN revision guards, source labels and creation recovery. Browser checks under AD-03 exercise actual API and mobile layout.

## 12. Documentation
Update application user flow and clarify source versus canonical status, unknown date, filters, sorting and list-context lifetime.

## 13. Definition of done
Behavioral tests and frontend build/lint pass; local browser review passes; existing design and architecture boundaries retained.

Evidence: [local execution report](execution-report.md), [verification outputs](evidence/). Checked criteria represent local engineering and synthetic fixture evidence, not real-client, owner or provider acceptance.
