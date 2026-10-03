# S10-01 — Define bounded temporal candidates and safe extraction versioning

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — live qualification/acceptance deferred |
| Date | 2026-10-03 |
| ID | S10-01 (local only) |
| Relative size | L; no delivery-date commitment |
| Depends on | OD-18; accepted ADR-0005; future implementation authorization |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Extract explicit interview/assessment temporal candidates without inventing instants or repeating paid work.

## 2. Why

extraction/v2 stores loose interview/assessment strings. A reliable agenda needs defined date/time uncertainty and a new versioned contract.

## 3. Current behavior

AI_CONTRACT_VERSIONS uses extraction/v2. Pipeline adopts completed results; AIOperation identity includes email, operation and version. Existing body bounds and provider-neutral adapters must remain.

## 4. Scope

Retain v2 definition; introduce extraction/v3 with legacy fields plus <=5 candidates following architecture §3. Add receivedAt context, strict bounded schema, deterministic DATE/DATETIME/UNRESOLVED normalization and validated optional excerpt. Persist a nullable versioned candidate JSON envelope with explicit mapping. Add default-off v3 selection and preserve pre-existing operation version identity.

## 5. Out of scope

No new provider/model, live evaluation in this ticket, bulk backfill, old-email replay, calendar writes, new classification prompt/version or automatic reschedule matching.

## 6. Likely files and components

Backend: services/ai/contracts.ts, pipeline.ts, operations.ts and provider-neutral input interface in services/ai/index.ts as needed; prisma/schema.prisma plus additive migration; eval fixtures and tests; contracts exposed only where consumed.

## 7. Implementation notes

Select extraction version before first claim. An older v2 pending/processing/held/completed operation keeps its contract; do not escape UNKNOWN by creating v3. Read both versions on resume and retain completed-result adoption. Missing/ambiguous zone cannot become a timestamp from host/browser timezone. Backend verifies any excerpt against normalized transient input; full bodies remain transient. Keep new capability disabled until S10-04 qualification. Feature disable preserves reconciliation support for already-selected v3.

## 8. Dependencies

ADR-0005 approval and OD-18. S10-02 consumes persisted candidates. Existing AI-19/AI-15 prerequisites matter before live activation, not for fixture contract development.

## 9. Security and privacy

Keep existing 8,000-character extraction body bound and no raw-body logs. Excerpts are private source data, max 280 characters, owner-only and never logged. Treat email text as data; adversarial email cannot change the schema or trigger tools.

## 10. Acceptance criteria

- [x] No candidate exceeds field/count bounds; malformed or invented temporal values stay invalid/unresolved.
- [x] Time without a source zone, DST gap/fold and ambiguous timezone abbreviation never become a confident instant.
- [x] DATE preserves its date; timestamp requires resolvable date/time and offset/zone.
- [x] Completed v2 mail produces zero provider calls after v3 is enabled.
- [x] v2 pending/held/unknown extraction resumes or requires approval under v2, never a new version claim.
- [x] Disabling new v3 selection does not strand existing v3 claims or erase candidates.
- [x] Additive migration preserves all existing rows; no eager mailbox backfill.
- [x] Prompt/input/schema changes share one new version and all adapters consume that same contract.

## 11. Testing

Table-driven temporal cases with explicit receivedAt, ambiguous 'IST', missing year/zone, leap dates and DST. Adversarial and conflicting-source examples. Real operation-ledger tests for completed/pending/unknown v2 and v3 enable/disable across simulated restarts; fail on an unexpected provider call. Run all guarded migration lanes.

## 12. Documentation updates

Document exact v2/v3 input/output, activation setting and version-selection table; add default-off/rollback instructions and updated synthetic eval cases. No live quality claim.

## 13. Definition of done

- [x] Contract and versioning acceptance demonstrated with fixtures.
- [x] Shared types, all affected adapter tests, local checks and migration preservation pass.
- [x] v3 remains disabled pending S10-04 and an activation decision.

Execution evidence and remaining live gates: [execution report](execution-report.md).

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.
