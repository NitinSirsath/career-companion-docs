# Sprint 10 — Interview and assessment agenda

Status: **LOCAL ENGINEERING VERIFIED**, 2026-10-03. See the [execution report](execution-report.md). Owner acceptance remains deferred; extraction/v3 stays default-off pending live qualification.

## Goal

Show upcoming interviews and assessment due dates with explicit uncertainty, evidence and user confirmation. Keep actual event time separate from status, recording time and action deadlines.

## Tickets and order

| Order | Ticket | Size | Dependencies |
| --- | --- | --- | --- |
| 1 | [S10-01](S10-01-temporal-extraction-contract.md) — temporal contract and safe version selection | L | OD-18, proposed ADR-0005 accepted before implementation |
| 2 | [S10-02](S10-02-agenda-domain-and-correction.md) — agenda persistence and correction | L | S10-01 |
| 3 | [S10-03](S10-03-agenda-review-ui.md) — agenda and user confirmation | M | S10-02 |
| 4 | [S10-04](S10-04-agenda-qualification.md) — contract qualification and integrated evidence | M | S10-01..03; existing AI-19/AI-15 prerequisites for live activation |

Two L tickets make this larger than Sprint 9. At kickoff, if measured capacity cannot fit the bounded scope, split foundation and UI/activation into sequential delivery slices; do not weaken temporal or versioning acceptance to fit a date. IDs retain their scope.

## Entry and exit

Entry: explicit implementation authorization, OD-18 and ADR-0005 accepted, Sprint 9 local evidence reviewed; owner feedback deferred. This does not reauthorize provider use, bulk reprocessing or a provider catalog promotion.

Exit:

- [x] Bounded extraction/v3 candidates preserve unknown date/time/zone.
- [x] Existing completed and held v2 operations cannot cause new paid calls on deployment.
- [x] Agenda suggestions, user confirmations and retirement survive replay and correction.
- [x] Reschedule/cancellation candidates require deliberate user review.
- [x] Migration preservation and cross-version regression evidence is recorded.
- [x] New contract qualification and live activation status are explicit; default-off remains if evidence is incomplete.

## Scope boundaries

[ADR-0005](../../architecture/decisions/ADR-0005-job-search-agenda.md) and [architecture §3](../future-sprints/architecture.md#3-sprint-10-trustworthy-temporal-meaning) own the accepted policy. No external calendar sync, reminders, meeting automation, free-form AI assistant, whole-mailbox reprocessing or task duplication. Historical v2 mail may have no agenda coverage; communicate that limitation.

AI-15 owns provider qualification, and AI-19 owns its existing runtime/disclosure prerequisites. S10-04 qualifies the changed extraction contract; it does not recreate their completed runner or bypass their remaining requirements.
