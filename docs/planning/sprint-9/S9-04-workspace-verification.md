# S9-04 — Verify the daily workspace and record a user trial

| Field | Value |
| --- | --- |
| Status | IMPLEMENTED LOCALLY — real-use acceptance pending |
| Date | 2026-10-03 |
| ID | S9-04 (local only) |
| Relative size | S; no delivery-date commitment |
| Depends on | S9-01, S9-02, S9-03 |
| Architecture | [Future architecture](../future-sprints/architecture.md) |
| Baseline | [Completed-state review](../future-sprints/completed-state-review.md) |

## 1. Objective

Prove the integrated daily workflow and separate local engineering completion from real-use acceptance.

## 2. Why

A working component or a green count of tests alone cannot prove that all work is visible and understandable.

## 3. Current behavior

Sprint 7/8 recorded extensive local fixtures and explicitly pending live acceptance. The existing smoke uses real API/PostgreSQL/pg-boss with outbound blocked.

## 4. Scope

Extend the existing smoke with multi-page action/application data, coherent summary, manual override, match move/unlink and incomplete coverage cases. Run relevant/full suites and local builds after implementation, compare sibling contracts, and record a short owner task trial when available.

## 5. Out of scope

No repeat of Sprint 7/8 implementation, live provider certification, new benchmark infrastructure, remote CI, deployment, database restore or invented daily-use evidence.

## 6. Likely files and components

Frontend scripts/smoke-stabilization.mjs and relevant test fixtures; backend existing action/application/match tests; new sprint-9 execution-report.md after execution. Existing guarded database helpers are reused.

## 7. Implementation notes

Use disposable guarded databases only. Preserve zero-residue teardown and outbound blocking. Fixture mutations should include retiring a pending source action while a workspace read is in flight, then observing corrected totals after refresh. Record baseline warnings separately. If a live observation is unavailable, keep its checkbox pending and publish the local result accurately.

## 8. Dependencies

All three feature tickets locally implemented. Owner presence is needed only for the real-use task observation, not fixture verification. No GitHub access.

## 9. Security and privacy

Do not place real subjects, recruiter emails, tokens or provider keys in screenshots/reports. Use synthetic data for stored artifacts.

## 10. Acceptance criteria

- [x] Multi-page daily counts equal owned active pending work after completion and correction.
- [x] Search/status behavior is correct across pages and manual overrides.
- [x] Disconnected, AI-waiting and failed-read states do not claim complete coverage.
- [x] Keyboard and 390px viewport journeys retain visible controls and focus.
- [x] Fixture smoke reports zero outbound calls beyond allowed loopback fixtures and zero residual users/jobs/budgets/MCP fixtures.
- [x] Full backend/frontend checks and local contract comparison are recorded with actual counts and warnings.
- [x] A dated real-use task observation is recorded, or explicitly remains pending without holding back an honest local implementation report.

## 11. Testing

Run the existing relevant tests first, then full suites/typecheck/lint/build and smoke for the integrated change. Reuse existing local runners and guards. No migration lane is needed if no migration was introduced; if scope changes, document why and run all applicable lanes. Do not run provider evaluations for a read-only workspace.

## 12. Documentation updates

Execution report states commits/worktree status, commands/results, fixture versus live evidence, screenshot paths, unresolved decisions and actual defects if found. Update sprint exit checkboxes only with linked evidence.

## 13. Definition of done

- [x] Local engineering result is reproducible and accurately described.
- [x] Real-world acceptance status is explicit.
- [x] No unrelated regression fix or remote action was bundled into verification.

Evidence: [execution report](execution-report.md). Checked items denote local engineering evidence or explicitly recorded pending real-use acceptance; no live-user observation is claimed.
