# AD-01 — Complete normal application discovery queries

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED AND VERIFIED LOCALLY |
| ID | AD-01 (local only) |
| Date | 2026-10-03 |
| Size | M |
| Depends on | Existing S9-03, S11-03; [plan decisions](README.md) |

## 1. Objective
Make every owned recorded application discoverable through the normal API with useful sorting and submission/status facets.

## 2. Why
Recorded submission evidence is independent of user/AI status. Creation order alone cannot answer when applications were submitted.

## 3. Current behavior
S9 search/status and S11 archive filters already run before pagination. List enrichment includes submittedVia and bounded latest non-retired event. Only createdAt DESC/id DESC is supported.

## 4. Scope
Extend ApplicationFiltersSchema with sort (added_desc, applied_desc, applied_asc, company_asc), submittedVia=AUTOMATION and effectiveStatus=UNKNOWN query sentinel. Filter by owned linked/created submission existence. Apply filters and deterministic ordering in PostgreSQL before limit/offset. Keep runtime contracts synchronized.

## 5. Out of scope
No schema/index/migration, count endpoint, new route, MCP change, status rewrite, new provenance payload, fuzzy/body search, provider call or saved search.

## 6. Likely files
Backend contracts/application.ts, services/application.ts, new tests/application-discovery.test.ts; frontend mirrored contract in AD-02.

## 7. Implementation notes
Defaults preserve existing consumers including correction pickers. Unknown means userStatus AND aiStatus null. APPLIED remains canonical status, not automation membership. Applied sorts put nulls last and tie by id; company sort uses stored name/database collation and id. Continue literal escaped q, owner/archived eligibility and bounded history. No raw query logging.

## 8. Dependencies
Use existing Prisma fields, relations, Zod contract and pagination. AD-02 consumes these additions; AD-03 verifies intake integration.

## 9. Security/privacy
Authenticate via existing session API; token identity never powers frontend reads. Scope submission relation to owner. Invalid/repeated sort/source/status values return 400; no raw SQL interpolation or new response fields.

## 10. Acceptance criteria
- [x] Legacy no-filter response/order and correction consumers remain compatible.
- [x] Each sort is deterministic across >20 rows, ties and null dates.
- [x] Combined q/status/source/archive predicates apply before pagination and never disclose foreign applications.
- [x] UNKNOWN includes automation-only records without inventing a status; APPLIED excludes them until canonical status is APPLIED.
- [x] Pending/ignored receipts never qualify; multiple receipts never duplicate an application.
- [x] Invalid inputs fail explicitly; retired evidence stays excluded.

## 11. Testing
PostgreSQL-backed normal API tests with two users, 25+ applications, null/tied dates, contradictory statuses, active/archived rows and linked/created/review receipts. Rerun workspace, status, evidence, archive and MCP suites.

## 12. Documentation
Record exact query/default/null/status/source contracts in this pack; link from S9 API documentation without rewriting its historical report.

## 13. Definition of done
Criteria pass, focused API checks pass, no schema/MCP/intake changes, and evidence is recorded in AD-03 closeout.

Local check: 5 backend files / 64 tests passed (new discovery, workspace, application status/evidence and follow-through suites). Full regression and shared-contract verification follow under AD-03.

Evidence: [local execution report](execution-report.md), [verification outputs](evidence/). Checked criteria represent local engineering and synthetic fixture evidence, not real-client, owner or provider acceptance.
