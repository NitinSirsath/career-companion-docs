# Application discovery completion — pre-migration local plan

Status: **IMPLEMENTED AND VERIFIED LOCALLY**, 2026-10-03. This complete plan was created and validated before implementation. Owner-authorized local work in the verified scratchpad; no migration, checkout creation, commit, push, PR or external tracker.

## Inspection and scope

The current scratchpad is ahead of the personal-PC report. S9-03 already implements literal company/title search and canonical status filtering before pagination. S11-03 already implements active/archived/all visibility. Application detail has status correction, bounded evidence, actions, follow-ups and archive/restore. MCP intake already creates or links Application plus ExternalSubmission and AUTOMATION_SUBMITTED evidence transactionally; retry identity is durable.

Missing: selectable stable ordering; a way to find automation submissions across later statuses; a way to find applications without user/AI status; persistent list context during detail visits; explicit read retry/refresh and filter recovery; joined MCP-to-normal-API-to-browser discovery evidence. Existing sort is createdAt DESC/id DESC. EffectiveStatus deliberately displays “Applied · via automation” for otherwise unknown state: this is submission evidence, not APPLIED status.

## Decisions and compatibility

1. Enhance /applications and its existing desktop/mobile navigation. No separate Applied tab: applications continue through interview/offer/rejection, and “Applied” is already a domain status.
2. Keep MCP → application domain → PostgreSQL → normal session-authenticated API → frontend. MCP tool/schema/auth/intake remain unchanged; no reads added.
3. Add optional sort and submittedVia filters, and UNKNOWN as an effectiveStatus query sentinel only. UNKNOWN is not a persisted status. Preserve all existing status values and default ordering, response shapes, page size, ownership and archive defaults.
4. Sort choices: newest added (existing default), applied date newest/oldest (nulls last in both), company A–Z (database collation). Every order ends with unique id for ties. Applied date is stored application.appliedAt, never event recording time or guessed createdAt.
5. submittedVia=AUTOMATION means an owned LINKED/CREATED ExternalSubmission exists. It is independent of effective status; NEEDS_REVIEW/IGNORED receipts do not appear as new applications. Unresolved records remain in existing Automation review.
6. No schema, migration, index, backfill, AI, queue or notification work. Existing fields and relations suffice. New indexes require measured need.
7. List context stays in the mounted Applications route while opening/returning from detail. It is not persisted in URLs/storage; searches contain private company/title text. A reload or leaving Applications starts fresh.
8. Reuse IBM Carbon-inspired semantic tokens, square Input/NativeSelect/Button/Badge primitives. Preserve detail actions and evidence. Explain unknown dates; retain submission provenance when user/AI status advances.
9. Offset pagination remains live rather than snapshot-stable across requests. Filter/sort changes reset page; acknowledged revision guards and bounded reread remain. No fabricated global totals.

## Tickets and execution order

| Order | Ticket | Scope | Depends on |
| --- | --- | --- | --- |
| 1 | [AD-01](AD-01-api-discovery.md) | Additive discovery query contract, SQL behavior and API tests | Existing S9-03, S11-03 |
| 2 | [AD-02](AD-02-applications-experience.md) | Coherent existing Applications view and cache behavior | AD-01 |
| 3 | [AD-03](AD-03-flow-verification.md) | MCP/API/browser proof, regression, documentation | AD-01, AD-02 |

These tickets extend rather than duplicate S9-03, S11-03/04 and MCP-01..07. MCP-08/MCP-09 real-client acceptance and provider qualification retain their original IDs and remain deferred. No new Sprint 9–11 tickets.

## Verification plan and boundaries

Use existing guarded, already-migrated local fixture databases only. Do not run migration commands or touch personal data. API tests cover all filters before pagination, stable null/tie ordering, owner isolation, archive/retirement and default compatibility. SDK test crosses real /mcp into PostgreSQL then normal /api/applications, detail and evidence; includes replay and review resolution. Frontend covers rapid supersession, status/source semantics, pagination reset, detail return, clear/retry/refresh and revision guards. Extend the real local browser smoke with synthetic submissions; capture desktop/mobile views and check keyboard/overflow. Run full suites, build/typecheck, lint, contract sync and preservation checks.

Baseline recorded before planning: backend HEAD 6af9cd2, frontend 155a1aa, docs 1e98ae4, all on feat/sprint-7-8-reliability with pre-existing uncommitted Sprint 9–11 work. Per-file hashes saved outside the checkout at /tmp/career-applied-baseline-20261003.json. Preserve every initial path and unrelated contents. Local fixtures are not live Gmail, real automation-client, owner acceptance or migration readiness approval.


## Local closeout

All three tickets are implemented. See the [execution report](execution-report.md), [API contract](api-contracts.md), [file manifest](change-manifest.md) and [evidence](evidence/). Local checks passed; real-world acceptance and migration remain deferred.
