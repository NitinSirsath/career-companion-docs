# Planning index and roadmap — after Sprint 8

Completed local work: [Application discovery completion](application-discovery/README.md), local AD-01 → AD-02 → AD-03, now implemented and verified. The complete plan was created and checked before implementation. It extends existing S9/S11 discovery without duplicating tickets; no migration or remote work is authorized.

Status: **local future planning, 2026-10-03. Sprint 7–11 implementation COMPLETE locally; real-world acceptance PENDING.**

Start with [future direction and sprint sequence](future-sprints/README.md), [completed-state review](future-sprints/completed-state-review.md), [architecture proposal](future-sprints/architecture.md), and [acceptance/decisions](future-sprints/acceptance-and-decisions.md). Sprint 9 was subsequently authorized and implemented; see its [execution report](sprint-9/execution-report.md). Sprint 10 and Sprint 11 local engineering is implemented; see [S10 evidence](sprint-10/execution-report.md) and [S11 evidence](sprint-11/execution-report.md).

This office-laptop workflow uses the existing local implementation checkouts and local Markdown tickets. No GitHub, Linear or Notion action, Downloads inspection, personal-machine search, personal-database change or live provider call is part of it. New migrations were tested only on guarded disposable databases.

## 1. Where things stand

| Area | Current office state | Evidence/limit |
| --- | --- | --- |
| Sprint 7 | Implementation complete | [Execution report](sprint-7/execution-report.md); real scheduled days and live Gmail remain pending. |
| Sprint 9 | Implementation complete locally; uncommitted | [Execution report](sprint-9/execution-report.md): 777 backend / 195 frontend tests and browser smoke passed; user trial pending. |
| Sprint 8 | Implementation complete | [Execution report](sprint-8/execution-report.md); runner delivered, live Gemini certification pending. |
| Local verification | Recorded 767 backend tests, 182 frontend tests, builds, migration lanes, browser smoke and real crash recovery passed | Prior implementation evidence, not rerun by this planning pass; provider fixtures are not live acceptance. |
| Source | Backend 6af9cd2, frontend 155a1aa, docs 1e98ae4 before planning; Sprint 9 intentionally uncommitted at S10 kickoff | Read directly from the office checkout; this docs delivery stays uncommitted. |
| BYO AI | Implemented, providers hidden; existing runtime/disclosure work and certification separately tracked | AI-19/20 and AI-15 retain their IDs; do not recreate delivered runner/retry work. |
| MCP | Intake and fixture evidence implemented; real automation-client acceptance pending | MCP-08/MCP-09 part B remain distinct. |
| Remote CI/deployment | No remote or deployed acceptance claimed | Out of scope for this office planning workflow. |

Downloads artifacts are not the baseline. Do not infer missing implementation from old plans. No additional Sprint 7/8 fixes without an actual defect discovered through real-world testing.

## 2. Order of work

1. Completed Sprint 7/8 forms the baseline; keep its reports intact.
2. Gather remaining real-world acceptance separately using the [acceptance register](future-sprints/acceptance-and-decisions.md). Pending evidence is not implementation work.
3. Review this future plan and record only the decisions relevant to the next selected sprint.
4. [Sprint 9](sprint-9/README.md) is implemented locally; collect its owner-trial evidence.
5. Completed [Sprint 10](sprint-10/README.md) implementation and local verification, then [Sprint 11](sprint-11/README.md) implementation and local verification. Final engineering review precedes separately authorized real-world testing.
6. Schedule release only on an explicit deployment/wider-use decision.

No remote publishing, Phase 0 restart or further baseline reconciliation is required to create or review these local plans.

## 3. Ticket index

S9/S10/S11 local engineering is implemented. The eight existing S10/S11 tickets were retained; S10-04 live qualification remains pending. Each has scope, dependencies, contracts, acceptance criteria, verification and exclusions.

| ID | Outcome | Size |
| --- | --- | --- |
| [S9-01](sprint-9/S9-01-workspace-read-model.md) | Full action buckets/counts and review summaries | M |
| [S9-02](sprint-9/S9-02-daily-workspace-ui.md) | Daily view with honest input coverage | M |
| [S9-03](sprint-9/S9-03-application-discovery.md) | Company/title search and effective-status filters | M |
| [S9-04](sprint-9/S9-04-workspace-verification.md) | Integrated checks and separate user-trial evidence | S |
| [S10-01](sprint-10/S10-01-temporal-extraction-contract.md) | Versioned temporal candidates with no old-mail replay | L |
| [S10-02](sprint-10/S10-02-agenda-domain-and-correction.md) | Owned agenda state, user revisions and correction history | L |
| [S10-03](sprint-10/S10-03-agenda-review-ui.md) | Review and confirm upcoming interviews/assessments | M |
| [S10-04](sprint-10/S10-04-agenda-qualification.md) | New-contract qualification and activation decision | M |
| [S11-01](sprint-11/S11-01-personal-follow-ups.md) | Personal follow-ups with creation receipts/revisions | M |
| [S11-02](sprint-11/S11-02-snooze-without-changing-deadlines.md) | In-app snooze that preserves real deadlines | M |
| [S11-03](sprint-11/S11-03-reversible-application-archive.md) | Reversible archive with preserved intake/history | L |
| [S11-04](sprint-11/S11-04-follow-through-ui-and-verification.md) | Combined controls and verified behavior | M |

Existing unfinished obligations keep their current IDs: [AI-19](byo-ai/AI-19-runtime-safety-fixes.md), [AI-20](byo-ai/AI-20-recovery-path-and-status-ui.md), [AI-15](byo-ai/AI-15-certify-gemini-first.md), [MCP-08](mcp-feature/MCP-08-automation-changes.md)/MCP-09 part B, and remaining [closeout](sprint-6/closeout/README.md) items. They are not automatically scheduled into the new sprint. Historical ticket allocations remain in the [preserved roadmap](roadmap-through-sprint8-2026-10-03.md#3-ticket-index).

## 4. MCP: where verification and productionization go

Keep completed intake, idempotency, ownership and manual-review behavior. Real-client acceptance remains MCP-08/MCP-09 part B. Any discovered defect retains a focused evidence-backed ticket; do not reopen the entire feature.

S11-03 implements the accepted OD-20 archived-only intake review boundary, preserving existing receipt identity without adding a tool. Public exposure/OAuth/new tools remain release or later scope.

## 5. BYO AI: what blocks "production-ready"

All providers remain hidden. AI-15's evaluation runner is delivered; actual provider certification, AI-19 prerequisites and owner-approved disclosure remain pending. The visible Manual Retry prerequisite is delivered, while AI-20's remaining historical scope must not be marked done wholesale.

Sprint 9 reads existing results and does not need new AI capability. Sprint 10 changes extraction and therefore needs separate extraction/v3 qualification, preserving existing paid-call and completed-result rules. Neither a fixture pass nor a future sprint plan permits a provider promotion.

No model availability, pricing, terms or support status was freshly verified on the internet in this local planning pass. Recheck those facts within the authorized certification work when it is executed.

## 6. Owner decisions

[OD-16..OD-21](future-sprints/acceptance-and-decisions.md#proposed-choices-for-future-work) record sequence, display timezone, agenda semantics, in-app follow-through, archive and release timing. OD-16..20 were accepted for the authorized local sprints; OD-21 remains proposed.

Existing OD-04/05/06/07/15 have execution decisions in Sprint 7/8 reports and [ADR-0003](../architecture/decisions/ADR-0003-correcting-email-matches.md); do not decide them again. Other old decisions remain documented in the [historical table](roadmap-through-sprint8-2026-10-03.md#6-owner-decisions), with their current acceptance relevance in the new register. No personal-data availability or live acceptance is assumed.

[ADR-0005 — agenda semantics](../architecture/decisions/ADR-0005-job-search-agenda.md) is accepted for local implementation. ADR-0004 stays reserved for hosting.

## 7. Release track (not scheduled)

Release is an independent future choice; it cannot replace the already completed Sprint 8. Keep the existing outline until deployment is selected:

1. ADR-0004: hosting/runtime topology, same-origin/TLS, OAuth setup, backup/restore and rollback; MCP exposure decision.
2. Production auth/session/request configuration and limits for the chosen exposure.
3. Bounded shutdown and safe recovery of in-flight jobs/provider uncertainty.
4. Existing AI-17 supervised release, with actual provider certification and honest no-provider UI.
5. Existing AI-18 only after one stable release, never as incidental destructive cleanup.

Before additional users, explicitly scope onboarding, consent enforcement, retention/account deletion and automation-created application recovery. No host/provider purchase or external deployment is selected in this plan.

## 8. Not now

- Automatic applications, job discovery, resumes, interview coaching or generalized AI chat.
- New AI providers, Outlook/LinkedIn ingestion, Gmail push, mobile app or queue replacement.
- Calendar integrations, sending follow-up messages or a new notification schedule/outbox.
- Application merge, data deletion/retention implementation or destructive cleanup.
- Bulk historical AI reprocessing, wider telemetry or analytics before useful daily-workflow evidence.
- Previously deferred action-detail source evidence (Sprint 6 D1) and submitted job-link display remain separate decisions; agenda evidence is scoped to the new agenda.
- GitHub publishing, remote CI and deployment in this office-laptop planning task.

## 9. Superseded or obsolete plan items

The [roadmap preserved through Sprint 8](roadmap-through-sprint8-2026-10-03.md) retains old decisions and history. Its old sequence “migration → closeout → Sprint 7 → Sprint 8” is not the next-work instruction. Existing sprint tickets keep their historical why/current-behavior sections; their execution reports and this index determine completed implementation status.

The latest source review is [completed-state-review.md](future-sprints/completed-state-review.md). The [2026-10-02 state audit](state-audit-2026-10-02.md) is historical and must not reopen delivered capabilities.

## 10. Planning rules (from now on)

- Local Markdown and local IDs for this workflow. No COM numbers or external issue creation.
- Preserve finished Sprint 7/8 and existing unfinished ticket IDs. New allocations in this pack: S9-01..04, S10-01..04, S11-01..04, OD-16..21 and ADR-0005.
- New tickets use the header plus 13 sections: Objective, Why, Current behavior, Scope, Out of scope, Likely files, Implementation notes, Dependencies, Security/privacy, Acceptance, Testing, Documentation, Definition of done.
- Proposed decisions remain proposed until explicitly accepted. Sprint 9–11 local code is authorized and complete. Live provider calls, real-world Gmail/MCP acceptance and deployment remain outside this execution scope.
- Checkboxes require linked evidence. Implementation complete, local fixtures, real-world acceptance and remote/deployed evidence are separate states.
- For actual later implementation, preserve unrelated work, use local contract synchronization and existing guarded test databases, and record actual results. No Git action is implied merely by planning.
