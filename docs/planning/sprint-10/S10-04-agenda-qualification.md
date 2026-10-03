# S10-04 — Qualify temporal extraction and record activation evidence

| Field | Value |
| --- | --- |
| Status | LOCAL ENGINEERING COMPLETE — live qualification/acceptance deferred |
| Date | 2026-10-03 |
| ID | S10-04 (local only) |
| Relative size | M; no delivery-date commitment |
| Depends on | S10-01..03; AI-19 and AI-15 prerequisites for live runs; owner key/consent |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Determine whether the new extraction contract is trustworthy enough to activate and record an honest release decision.

## 2. Why

Passing extraction/v2 evaluation does not certify new temporal fields or reschedule semantics.

## 3. Current behavior

The existing runner already separates PASS/FAIL/INCONCLUSIVE and bounded refusals. Live Gemini certification is pending; do not rebuild that runner.

## 4. Scope

Extend existing evaluation data/scoring for temporal candidates, including explicit-time fidelity, correct unresolved handling, quote bounds and legacy-field regression. Run local fixtures and integrated smoke. With separately authorized live key/use, qualify every offered v3 model configuration and record the activation decision.

## 5. Out of scope

No automatic provider promotion, new provider, relaxed quality floors, retry of unknown paid outcomes, bulk mailbox evaluation or fabricated live data.

## 6. Likely files and components

Backend src/eval/ai dataset/score/tests and reports when executed; frontend existing smoke; sprint-10 execution-report.md and ADR/activation record.

## 7. Implementation notes

Before live runs freeze a reviewed dataset and thresholds: zero unsupported confident timestamps or silent cross-email cancellations on critical cases; all existing schema/critical quality floors still apply. Missing/refused calls produce INCONCLUSIVE, not a pass. Each offered configuration needs two complete runs of the temporal and existing regression cases. State call budget before authorized runs; do not lower thresholds after observing failures. If prerequisites/evidence are absent, retain default-off and record local implementation separately from qualification.

## 8. Dependencies

Reuse AI-15 certification ownership and AI-19 runtime/disclosure requirements. S10-01..03 complete locally. Live activation requires its own explicit decision; planning and fixture tests do not grant it.

## 9. Security and privacy

Use synthetic or owner-approved redacted cases; never submit mailbox contents incidentally. Keys stay in approved local environment, not docs/logs. Store bounded outcome metadata without sensitive source content.

## 10. Acceptance criteria

- [x] Dataset includes date-only, missing zone, DST, ambiguous abbreviation, contradiction, reschedule, cancellation and prompt-injection cases.
- [x] Scorer detects invented instants and unsafe automatic event identity assumptions.
- [x] Existing extraction quality, version-adoption and correction invariants remain verified.
- [x] Refusal/partial runs remain INCONCLUSIVE with no overwritten evidence.
- [x] Migration and real API smoke evidence is recorded separately from live AI evidence.
- [x] Two-run qualification per offered configuration is linked, or missing evidence is explicitly pending.
- [x] Activation is recorded only with prerequisites/approval; otherwise the feature remains default-off.

## 11. Testing

Scorer tests with known intentionally bad model outputs, regression datasets and existing refusal accounting. Full local suites/builds and migration lanes after implementation. Live calls only under the existing explicit-key/evaluation workflow; no routine repeated runs without cause.

## 12. Documentation updates

Record SDK/model/contract/dataset versions, complete/refused call counts, thresholds, reports, activation flag and rollback/reconciliation procedure. Update acceptance register without marking unrelated live Gmail gates passed.

## 13. Definition of done

- [x] Local implementation and live qualification have separate truthful statuses.
- [x] No unqualified feature/provider is promoted.
- [x] All actual evidence is linked and repeatable.

Execution evidence and remaining live gates: [execution report](execution-report.md).

## Execution evidence — 2026-10-03

Checkboxes record local engineering verification, not live qualification or owner acceptance. See the [execution report](execution-report.md) for delivered behavior, tests and limitations. Historical Current behavior describes the pre-implementation baseline.

- [ ] Two complete real model-specific extraction/v3 qualification runs, prerequisites/disclosure and explicit activation decision. Deferred; the default-off flag remains in place.
