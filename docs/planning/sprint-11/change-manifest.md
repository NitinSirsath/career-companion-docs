# Sprint 10/11 change manifest

Compared by SHA-256 with the pre-edit scratchpad snapshot, not HEAD (which also includes prior Sprint 9 work). All baseline paths remain. Ignored local fixture configuration/builds/dependencies are excluded. This manifest includes itself; hashes are not embedded.

## career-companion-backend

39 files differ from kickoff.

- `.env.example` — changed from kickoff
- `prisma/migrations/20261003100000_job_search_agenda/migration.sql` — new in this implementation
- `prisma/migrations/20261003110000_personal_follow_through/migration.sql` — new in this implementation
- `prisma/schema.prisma` — changed from kickoff
- `scripts/verify-sprint-migrations.cjs` — new in this implementation
- `src/contracts/action.ts` — changed from kickoff
- `src/contracts/agenda.ts` — new in this implementation
- `src/contracts/application.ts` — changed from kickoff
- `src/contracts/index.ts` — changed from kickoff
- `src/contracts/temporal.ts` — new in this implementation
- `src/contracts/workspace.ts` — changed from kickoff
- `src/eval/ai/run.ts` — changed from kickoff
- `src/eval/ai/score.ts` — changed from kickoff
- `src/eval/ai/temporal.test.ts` — new in this implementation
- `src/eval/ai/temporal.ts` — new in this implementation
- `src/index.ts` — changed from kickoff
- `src/jobs/notificationJob.ts` — changed from kickoff
- `src/middleware/error.ts` — changed from kickoff
- `src/routes/action.ts` — changed from kickoff
- `src/routes/agenda.ts` — new in this implementation
- `src/routes/application.ts` — changed from kickoff
- `src/services/action.ts` — changed from kickoff
- `src/services/agenda.ts` — new in this implementation
- `src/services/ai/capabilities.ts` — changed from kickoff
- `src/services/ai/contracts.ts` — changed from kickoff
- `src/services/ai/pipeline.ts` — changed from kickoff
- `src/services/ai/temporal.ts` — new in this implementation
- `src/services/application.ts` — changed from kickoff
- `src/services/archive.ts` — new in this implementation
- `src/services/externalSubmission.ts` — changed from kickoff
- `src/services/matcher.ts` — changed from kickoff
- `src/services/notificationSuppression.ts` — new in this implementation
- `src/services/workspace.ts` — changed from kickoff
- `src/tests/agenda.test.ts` — new in this implementation
- `src/tests/ai-pipeline-integration.test.ts` — changed from kickoff
- `src/tests/application-status.test.ts` — changed from kickoff
- `src/tests/follow-through.test.ts` — new in this implementation
- `src/tests/notificationJob.test.ts` — changed from kickoff
- `src/tests/workspace.test.ts` — changed from kickoff

## career-companion-frontend

28 files differ from kickoff.

- `scripts/smoke-stabilization.mjs` — changed from kickoff
- `src/api/client.ts` — changed from kickoff
- `src/components/Agenda.tsx` — new in this implementation
- `src/components/DailyWorkspace.tsx` — changed from kickoff
- `src/components/FollowThrough.tsx` — new in this implementation
- `src/components/MatchCorrectionDialog.tsx` — changed from kickoff
- `src/components/automation/PendingSubmissionsSection.tsx` — changed from kickoff
- `src/components/layout/MobileNav.tsx` — changed from kickoff
- `src/components/layout/Sidebar.tsx` — changed from kickoff
- `src/contracts/action.ts` — changed from kickoff
- `src/contracts/agenda.ts` — new in this implementation
- `src/contracts/application.ts` — changed from kickoff
- `src/contracts/index.ts` — changed from kickoff
- `src/contracts/temporal.ts` — new in this implementation
- `src/contracts/workspace.ts` — changed from kickoff
- `src/lib/applicationCache.ts` — changed from kickoff
- `src/lib/processingRefresh.ts` — changed from kickoff
- `src/routeTree.gen.ts` — changed from kickoff
- `src/routes/agenda.tsx` — new in this implementation
- `src/routes/applications.$id.tsx` — changed from kickoff
- `src/routes/applications.tsx` — changed from kickoff
- `src/tests/action-queue.test.tsx` — changed from kickoff
- `src/tests/agenda.test.tsx` — new in this implementation
- `src/tests/applications.test.tsx` — changed from kickoff
- `src/tests/fixtures.ts` — changed from kickoff
- `src/tests/follow-through.test.tsx` — new in this implementation
- `src/tests/stabilization-ui.test.tsx` — changed from kickoff
- `src/tests/status-correction.test.tsx` — changed from kickoff

## career-companion-docs

78 files differ from kickoff.

- `PROJECT_CONSTITUTION.md` — changed from kickoff
- `README.md` — changed from kickoff
- `docs/architecture/decisions/ADR-0002-automation-submissions-via-mcp.md` — changed from kickoff
- `docs/architecture/decisions/ADR-0005-job-search-agenda.md` — changed from kickoff
- `docs/planning/README.md` — changed from kickoff
- `docs/planning/future-sprints/README.md` — changed from kickoff
- `docs/planning/future-sprints/acceptance-and-decisions.md` — changed from kickoff
- `docs/planning/future-sprints/architecture.md` — changed from kickoff
- `docs/planning/sprint-10/README.md` — changed from kickoff
- `docs/planning/sprint-10/S10-01-temporal-extraction-contract.md` — changed from kickoff
- `docs/planning/sprint-10/S10-02-agenda-domain-and-correction.md` — changed from kickoff
- `docs/planning/sprint-10/S10-03-agenda-review-ui.md` — changed from kickoff
- `docs/planning/sprint-10/S10-04-agenda-qualification.md` — changed from kickoff
- `docs/planning/sprint-10/evidence/ai-choose-provider.png` — new in this implementation
- `docs/planning/sprint-10/evidence/ai-setup-form.png` — new in this implementation
- `docs/planning/sprint-10/evidence/ai-status-ready.png` — new in this implementation
- `docs/planning/sprint-10/evidence/automation-timeline.png` — new in this implementation
- `docs/planning/sprint-10/evidence/automation-token-created.png` — new in this implementation
- `docs/planning/sprint-10/evidence/backend-build.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/backend-lint.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/backend-tests.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/corrected-email-timeline.png` — new in this implementation
- `docs/planning/sprint-10/evidence/dashboard-ai-not-set-up.png` — new in this implementation
- `docs/planning/sprint-10/evidence/dashboard-submission-review-mobile.png` — new in this implementation
- `docs/planning/sprint-10/evidence/final-focused.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/frontend-build.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/frontend-lint.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/frontend-tests.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/gmail-waiting-for-ai.png` — new in this implementation
- `docs/planning/sprint-10/evidence/migration-agenda.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/migration-ai.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/migration-mcp.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/migration-status.txt` — new in this implementation
- `docs/planning/sprint-10/evidence/retry-anyway-dialog.png` — new in this implementation
- `docs/planning/sprint-10/evidence/s10-agenda-desktop.png` — new in this implementation
- `docs/planning/sprint-10/evidence/s10-agenda-mobile.png` — new in this implementation
- `docs/planning/sprint-10/evidence/s9-workspace-desktop.png` — new in this implementation
- `docs/planning/sprint-10/evidence/s9-workspace-mobile.png` — new in this implementation
- `docs/planning/sprint-10/evidence/smoke.txt` — new in this implementation
- `docs/planning/sprint-10/execution-report.md` — new in this implementation
- `docs/planning/sprint-11/README.md` — changed from kickoff
- `docs/planning/sprint-11/S11-01-personal-follow-ups.md` — changed from kickoff
- `docs/planning/sprint-11/S11-02-snooze-without-changing-deadlines.md` — changed from kickoff
- `docs/planning/sprint-11/S11-03-reversible-application-archive.md` — changed from kickoff
- `docs/planning/sprint-11/S11-04-follow-through-ui-and-verification.md` — changed from kickoff
- `docs/planning/sprint-11/change-manifest.md` — new in this implementation
- `docs/planning/sprint-11/engineering-review.md` — new in this implementation
- `docs/planning/sprint-11/evidence/ai-choose-provider.png` — new in this implementation
- `docs/planning/sprint-11/evidence/ai-setup-form.png` — new in this implementation
- `docs/planning/sprint-11/evidence/ai-status-ready.png` — new in this implementation
- `docs/planning/sprint-11/evidence/automation-timeline.png` — new in this implementation
- `docs/planning/sprint-11/evidence/automation-token-created.png` — new in this implementation
- `docs/planning/sprint-11/evidence/backend-build.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/backend-lint.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/backend-tests.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/corrected-email-timeline.png` — new in this implementation
- `docs/planning/sprint-11/evidence/dashboard-ai-not-set-up.png` — new in this implementation
- `docs/planning/sprint-11/evidence/dashboard-submission-review-mobile.png` — new in this implementation
- `docs/planning/sprint-11/evidence/frontend-build.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/frontend-lint.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/frontend-tests.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/gmail-waiting-for-ai.png` — new in this implementation
- `docs/planning/sprint-11/evidence/migration-ai.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/migration-follow.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/migration-mcp.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/migration-status.txt` — new in this implementation
- `docs/planning/sprint-11/evidence/retry-anyway-dialog.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s10-agenda-desktop.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s10-agenda-mobile.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s11-archived-mobile.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s11-snoozed-desktop.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s11-workspace-failure.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s9-workspace-desktop.png` — new in this implementation
- `docs/planning/sprint-11/evidence/s9-workspace-mobile.png` — new in this implementation
- `docs/planning/sprint-11/evidence/smoke.txt` — new in this implementation
- `docs/planning/sprint-11/execution-report.md` — new in this implementation
- `docs/product/user-flows.md` — changed from kickoff

- `docs/planning/sprint-11/evidence/final-preservation-check.txt` — new in this implementation
