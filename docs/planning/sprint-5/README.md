# Sprint 5 — Gmail Incremental Sync & Reliability

> **Roadmap update (2026-10-02):** the Sprint 5 Gmail reliability work is absent from the current code. [S5-FU-01](follow-up-gmail-reliability.md) is rescoped and scheduled in [Sprint 7](../sprint-7/README.md); its transport part moved to [S5-FU-02](follow-up-google-request-bounds.md) (Sprint 8). Ticket status lines below that say "implemented" describe the lost working tree, not the current code. See the [roadmap](../README.md).

Engineering implementation is present as uncommitted backend/frontend file changes, per the user's instruction. See the [execution report](execution-report.md) for current results, evidence, limitations and review findings. Historical planning documents remain useful background; their fixed 90-day scope, old migration count and ticket filenames are superseded here.

The supplied COM-37–COM-41 list and migrated descriptions disagree. The following sequential mapping preserves every indexed workstream and is **inferred pending Linear confirmation**. No remote ticket was overwritten or transitioned.

| Ticket | Scope | Evidence |
| --- | --- | --- |
| [COM-37 / S5-01](S5-01.md) | Baseline/readiness and preservation tooling | Private snapshot/compare utility; separate guarded databases; source/backup record |
| [COM-38 / S5-02](S5-02.md) | Incremental sync and repeat idempotency | History, multi-page, reconciliation, repeat/preservation tests; real local worker/browser path |
| [COM-39 / S5-03](S5-03.md) | Crash recovery, fencing and bounded execution | Process-kill/redelivery, transaction/ownership, transport and AI retry safeguards |
| [COM-40 / S5-04](S5-04.md) | Browser flow and asynchronous cache refresh | Delayed matching across navigation, bounded polling, failure recovery |
| [COM-41 / S5-05](S5-05.md) | Observability and operations | Correlated events, reconciled dispositions, retry/health runbook |

Implementation order: readiness → recovery → observability → final browser regression → incremental evidence consolidation. No new feature/provider/scheduler or schema rewrite is introduced. Gmail sync stays manual and additive, with main's configured **1/7/14/30 days, default 1 day**. Stored rows and user choices survive narrower lookback and provider reconciliation.

## Completion boundary

The current user instruction prioritizes implementation and automated code-quality validation. Missing manual Gmail/AI capability does not stop sprint engineering. Live Gmail discovery, live OAuth refresh and original historical dataset preservation are still **unverified**, and must not be inferred from fixture tests. Local database inspection found zero emails and no Gmail connection; the historical ~1,620-row dataset was not migrated here.

**Follow-up (2026-10-02):** the Gmail sync fencing, bounded transport, telemetry, crash-test and snapshot-tooling work is absent from the current downloaded baseline and is recorded as proposed, not-started follow-up [S5-FU-01](follow-up-gmail-reliability.md).

[Execution report](execution-report.md) · [Operational runbook](operations.md) · [Preserved-data/live verification procedure](verification-runbook.md) · [Historical architecture review](architecture-review.md) · [Historical planning review](planning-review.md)
