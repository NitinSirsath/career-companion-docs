# Sprint 6 Linear migration package

Status: ready for local review/copy; no Linear issues have been created. S6-01 through S6-05 are temporary local IDs. The proposal must be reconciled after Sprint 5 completion and approved before implementation.

## Copy procedure

Each linked ticket is the complete ready-to-copy description, with all 21 required sections. Copy the entire body, including its metadata and open acceptance checkboxes, rather than replacing it with the short summary below. Section 15 is the acceptance-criteria block; sections 4–13 contain technical context, files, data/API/UI/AI/job/security boundaries; section 17 contains dependencies; section 20 lists documentation updates. This keeps all review fixes in the engineering handoff.

Attach the shared [overview](README.md), [assessment](architecture-review.md), [verification runbook](verification-runbook.md) and [review resolutions](planning-review.md) as repository-relative links or uploaded documents accessible to the team. When publishing, use verified repository URLs at the final planning revision; do not copy machine-local paths into external tickets. The source prefixes are defined in the assessment.

Validate the actual Linear team/project, available labels, priority scale and estimation scale before creating issues. Preserve the provisional values unless the team re-estimates. Reconcile Sprint 5's supplied COM mappings before creating dependencies: existing source already mentions an unrelated historical COM-37, and this pack does not claim its local mapping is a verified live issue ID. After actual creation, replace local S6 dependency references with returned issue IDs/links. Do not invent them in advance.

## S6-01

| Field | Ready-to-copy value |
| --- | --- |
| Title | Unify effective application status across API and UI |
| Description | Entire [S6-01 ticket](S6-01.md) |
| Acceptance criteria | S6-01 section 15, all checkboxes open |
| Technical context | One backend-derived user-first rule; required response fields and runtime client validation; 64-case test matrix |
| Dependencies | Approved final Sprint 5 entry gate; blocks S6-02/03/04 |
| Suggested labels | sprint-6, backend, frontend, correctness |
| Priority / estimate | High / 3 provisional points |
| Repository | career-companion-backend; career-companion-frontend |
| Relevant files | backend/src/services/application.ts; backend/src/contracts/application.ts; backend/src/routes/application.ts; frontend/src/api/client.ts; frontend/src/routes/applications.tsx; frontend/src/routes/applications.$id.tsx; application/stabilization tests listed in the ticket |
| Relevant docs | docs/domain/domain-model.md; docs/architecture/mvp-architecture.md; backend/README.md; frontend/README.md; shared planning runbook |

## S6-02

| Field | Ready-to-copy value |
| --- | --- |
| Title | Add concurrency-safe manual application status correction |
| Description | Entire [S6-02 ticket](S6-02.md) |
| Acceptance criteria | S6-02 section 15, all checkboxes open |
| Technical context | Owned row lock, manual revision compare before no-op, strict PATCH contract, atomic set/change/clear, guarded additive migration |
| Dependencies | Sprint 5 entry gate; S6-01; coordinate shared mapper/contracts with S6-03; blocks S6-04/05 |
| Suggested labels | sprint-6, backend, api, correctness |
| Priority / estimate | High / 5 provisional points |
| Repository | career-companion-backend; coordinated career-companion-frontend contract fixtures |
| Relevant files | backend/prisma/schema.prisma; backend/prisma/migrations/ (new additive migration proposed); backend/src/services/application.ts; backend/src/routes/application.ts; backend/src/contracts/application.ts; backend/src/services/matcher.ts; backend/src/utils/testDatabase.ts; backend tests and frontend contract fixtures listed in the ticket |
| Relevant docs | docs/domain/domain-model.md; docs/product/user-flows.md UF-09; backend/README.md; backend/STABILIZATION.md; shared planning runbook |

## S6-03

| Field | Ready-to-copy value |
| --- | --- |
| Title | Expose owned source evidence and recording semantics in application history |
| Description | Entire [S6-03 ticket](S6-03.md) |
| Acceptance criteria | S6-03 section 15, all checkboxes open |
| Technical context | Bounded owned metadata in events/recentEvent, explicit recordedAt, nullable source, unchanged recording-order pagination, runtime response schema |
| Dependencies | Sprint 5 entry gate; S6-01; coordinate shared mapper/contracts with S6-02; blocks S6-04/05 |
| Suggested labels | sprint-6, backend, api, privacy |
| Priority / estimate | Normal / 3 provisional points |
| Repository | career-companion-backend; coordinated career-companion-frontend contract fixtures |
| Relevant files | backend/src/services/application.ts; backend/src/contracts/application.ts; backend/src/routes/application.ts; backend/prisma/schema.prisma (read only); backend/src/services/matcher.ts; frontend/src/api/client.ts; application/matcher/stabilization tests listed in the ticket |
| Relevant docs | docs/domain/domain-model.md; docs/architecture/mvp-architecture.md; docs/product/user-flows.md UF-07; backend/README.md; shared planning runbook |

## S6-04

| Field | Ready-to-copy value |
| --- | --- |
| Title | Deliver the application status correction and evidence-review workflow |
| Description | Entire [S6-04 ticket](S6-04.md) |
| Acceptance criteria | S6-04 section 15, all checkboxes open |
| Technical context | Frozen editor revision/target, explicit conflict review, cancelled stale reads, structured errors/runtime parsing, uncertain-save reconciliation, independent section states |
| Dependencies | Sprint 5 entry gate; S6-01/02/03; blocks S6-05 |
| Suggested labels | sprint-6, frontend, ux, correctness |
| Priority / estimate | High / 5 provisional points |
| Repository | career-companion-frontend |
| Relevant files | frontend/src/routes/applications.$id.tsx; frontend/src/routes/applications.tsx; frontend/src/api/client.ts; frontend/src/contracts/application.ts; frontend/src/components/ui/; frontend/src/tests/applications.test.tsx; frontend/src/tests/stabilization-ui.test.tsx |
| Relevant docs | docs/product/user-flows.md UF-07/UF-09; docs/DESIGN_SYSTEM.md if a primitive is added; frontend/README.md; shared planning runbook |

## S6-05

| Field | Ready-to-copy value |
| --- | --- |
| Title | Verify the complete status workflow and publish its operating contract |
| Description | Entire [S6-05 ticket](S6-05.md) |
| Acceptance criteria | S6-05 section 15, all checkboxes open |
| Technical context | Extend completed Sprint 5 real-worker harness; exclusive fixture lanes; guard before CLI; migration preservation; isolated manual/replay side-effect counts; stop workers before safe cleanup |
| Dependencies | Final Sprint 5 evidence/harness; S6-01/02/03/04; blocks Sprint 6 completion |
| Suggested labels | sprint-6, qa, integration, documentation |
| Priority / estimate | High / 3 provisional points |
| Repository | career-companion-backend; career-companion-frontend; career-companion-docs |
| Relevant files | frontend/scripts/smoke-stabilization.mjs; backend/src/tests/setup.ts; backend/src/utils/testDatabase.ts; backend/src/tests/application.test.ts; backend/src/tests/matcher.test.ts; backend/src/tests/ai-idempotency.test.ts; backend/src/tests/stabilization-safety.test.ts; frontend/src/tests/applications.test.tsx; frontend/src/tests/stabilization-ui.test.tsx; backend/prisma/schema.prisma and migrations |
| Relevant docs | All exact update targets in overview section J and S6-05 section 20; Sprint 5 runbook/review; Sprint 6 shared runbook and execution record |

## Completion bookkeeping

Total proposed estimate is 19 points. This is neither verified capacity nor a calendar promise. Retain open checkboxes until executable evidence exists. Link final commits and evidence during implementation, and distinguish document review from feature completion. This local package does not authorize external issue creation or messaging.
