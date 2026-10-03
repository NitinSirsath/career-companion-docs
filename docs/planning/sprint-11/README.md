# Sprint 11 — Personal follow-through

Status: **LOCAL ENGINEERING VERIFIED**, 2026-10-03. See the [execution report](execution-report.md). Owner acceptance remains deferred; extraction/v3 stays default-off pending live qualification.

## Goal

Let the owner record follow-up work, defer it deliberately, and remove old applications from the daily working view without losing evidence.

## Tickets and order

| Order | Ticket | Size | Dependencies |
| --- | --- | --- | --- |
| 1 | [S11-01](S11-01-personal-follow-ups.md) — personal follow-up creation and revisioned intent | M | OD-19; Sprint 9 workspace |
| 2 | [S11-02](S11-02-snooze-without-changing-deadlines.md) — in-app snooze | M | S11-01, OD-19 |
| 3 | [S11-03](S11-03-reversible-application-archive.md) — archive and restore | L | OD-20; coordinate S10 agenda boundary |
| 4 | [S11-04](S11-04-follow-through-ui-and-verification.md) — integrated controls and evidence | M | S11-01..03 |

S11-03 spans matching, intake, reads and notification suppression; treat it as the largest item. If capacity is insufficient, deliver manual follow-ups/snooze first and schedule archive separately without weakening preservation.

## Entry and exit

Entry: explicit implementation authorization, OD-19/20 recorded, observed need reviewed. The default sequence follows Sprint 10 so archive has one agenda integration target. If reordered, amend scope explicitly.

Exit:

- [x] User actions are distinct from AI/email-derived evidence and creation cannot duplicate on response loss.
- [x] Snooze preserves the actual deadline and resurfaces in-app without a job or send.
- [x] Archive/restore preserves status, history, linked email, MCP receipt identity and user decisions.
- [x] No hidden archived-target duplication or replay of notifications occurs.
- [x] Ownership, stale writes, match correction and migration preservation are verified.
- [x] Integrated local evidence and user-trial status are recorded separately.

## Boundaries

Use [architecture §4](../future-sprints/architecture.md#4-sprint-11-user-controlled-follow-through). No email sending, new Discord schedule, automatic follow-up messages, AI-written outreach, generalized task projects, retention/deletion, application merge or deployment.

Archive is an implemented user intent capability, not a cleanup of existing Sprint 7/8 code. The accepted unlink/later-thread policy remains unchanged.
