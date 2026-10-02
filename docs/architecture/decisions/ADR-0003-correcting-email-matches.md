# ADR-0003 — Correcting email matches without deleting history

| Field | Value |
| --- | --- |
| Status | Accepted by the owner on 2026-10-03: correction scope authorized and unlink/future-thread review policy explicitly confirmed |
| Date | 2026-10-03 |
| Decides | Move/unlink behavior, retirement, status reconstruction and later thread matching |
| Detail | [S8-01](../../planning/sprint-8/S8-01-correct-a-wrong-email-match.md) |

## Context

An incorrect link currently leaves evidence and actions on the wrong application and can influence later thread mail. The owner authorized moves/unlinks with preserved history and backend authority. The current code lacks the S6-R07 automatic-rematch guard; implement that concrete prerequisite with correction, without reopening unrelated Sprint 6 work.

## Decision

1. MATCHED emails may move or unlink; IGNORED emails may link to an existing owned application. UNMATCHED/AMBIGUOUS retain resolve. Move produces MATCHED + USER_CONFIRMED; unlink produces IGNORED + USER_CONFIRMED with no application. AI cannot change the decision.
2. Retire every active event/action belonging to this email on source applications with retirement time and EMAIL_MOVED or EMAIL_UNLINKED. Never delete history or retire AUTOMATION_SUBMITTED events. Retired effects leave action lists/counts, notification eligibility, recentEvent and AI-status evidence, but remain marked in the timeline.
3. Create target effects using the normal deadline and matching rules. Reactivate existing target rows instead of duplicating them, preserving their stored status. A new target action carries COMPLETED, otherwise DISMISSED, otherwise PENDING from old actions, so correction does not reopen handled work.
4. Recompute each source application's aiStatus from still-matched email evidence, including a possible decrease to null. The target retains the ordinary raise-only rule. Never change userStatus, userStatusSetAt or userStatusRevision.
5. A user-confirmed thread decision wins over automatic links. After unlink or ordinary ignore, later thread mail remains UNMATCHED for review, bypassing both thread and company matching. The owner explicitly confirmed this policy; it prevents the same incorrect company match from reappearing.
6. Compare the expected match state/application under lock; stale requests return 409 and are never automatically replayed. Lock order is user match lock, email, then applications in ascending ID order. Thread-derived decisions are rechecked under that lock.
7. Correction performs only the database transaction and a safe structured completion event. No AI/Gmail call, queue job, Discord send or AI-ledger mutation. Retirement is latest-only provenance, not a separate audit-history system.

## Not part of this decision

Application merging/deletion, moving whole threads, automation-submission correction, automatic rematching, AI reprocessing and a full correction audit log.

## Consequences

Useful history remains visible and completed tasks stay handled. Unlink requires explicit review of future thread mail. A move can change derived AI status while preserving the user's override. Add nullable retirement fields and reason checks without backfill; retain unique effect keys. A code rollback would expose retired effects again, so it requires a reviewed rollback plan.

## Alternatives considered

Returning unlink to UNMATCHED would let automatic matching restore the mistake. Deleting effects loses history. Copying effects without retirement leaves contradictory status and duplicate tasks. A new revision column is unnecessary for the documented expected-state comparison.

## Future evolution

Bulk thread moves, full audit history and application consolidation require separate product decisions. Verification covers ownership, stale requests, opposite moves, thread/correction races, retired reads/notifications, all migration lanes and the browser flow.
