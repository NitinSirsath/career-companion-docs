# Sprint 6 planning review and resolutions

Reviewed local proposal, 2026-09-26. The user asked for the findings to be fixed in ticket documents. These resolutions are now incorporated into the full ticket bodies and shared runbook. They do not mean Sprint 6 code, migrations or tests have been implemented.

## Findings resolved in the ticket text

| ID / priority | Finding | Applied correction | Documents / required regression evidence |
| --- | --- | --- | --- |
| R1 / P1 | The original proposal did not account for the newly available Sprint 5 draft and risked preserving the old worker-disabled smoke assumptions | Added the complete unexecuted Sprint 5 entry gate, its corrected dependency sequence and ID-reconciliation boundary. Require extending its final real-worker harness with encrypted fixture credentials, history anchor, nonzero isolated AI budget, traffic blocking and safe teardown. Zero-call assertions are scoped to manual operations/completed replay, not the entire fixture run. | [Overview](README.md), [assessment](architecture-review.md), [S6-05 sections 6/11/12/14/15](S6-05.md), [runbook steps 6/7](verification-runbook.md) |
| R2 / P1 | Reading the latest query revision at Save would silently rebase an open editor and bypass the intended 409 conflict | Freeze application ID, draft and base revision when editing starts. Refetch may update visible server state, never the draft token. Conflict requires explicit review and deliberate rebase; no automatic resend. | [S6-02 sections 6/10/14/15](S6-02.md), [S6-04 sections 6/14/15/18](S6-04.md), [S6-05](S6-05.md). Test another device saving plus background refetch before the stale editor first submits. |
| R3 / P1 | Raw migration CLI could mutate a DB before the test helper ran; .env.test overrides and missing Prisma generation made the command order unsafe/incomplete | Specify an independent expected disposable target, load overriding .env.test, validate equality and the existing safety helper before spawning Prisma with the same environment. Require provenance/exclusivity and generation before compilation. | [S6-02 sections 8/14/19](S6-02.md), [S6-05 sections 8/13/14/19](S6-05.md), executable procedure in [runbook steps 1–4](verification-runbook.md). Guard rejection must prevent CLI invocation. |
| R4 / P2 | Delayed reads, automatic mutation retries and lost responses could misrepresent persisted state or overwrite a newer correction | Propagate AbortSignal, cancel relevant reads before mutation/result application, keep target IDs captured, protect newer cached revisions and refetch. Disable automatic mutation retry. Reconcile timeout/network/5xx/invalid-success outcomes by GET; failed reconciliation preserves draft with Save outcome unknown and disables blind save. | [S6-04 sections 6/14/15/18](S6-04.md), [S6-02 recovery](S6-02.md), [S6-05](S6-05.md), [runbook scenario table](verification-runbook.md) |
| R5 / P2 | TypeScript return types do not validate JSON; missing required fields could be displayed as legitimate unknown state | Runtime-parse affected application create/list/detail/mutation and history responses using synchronized Zod schemas. Required keys have no masking defaults. Invalid contracts yield recoverable errors and block editing; true null remains valid. Invalid successful writes enter uncertain-outcome recovery. | [S6-01 sections 6/9/14/15](S6-01.md), [S6-03 sections 6/9/14/15](S6-03.md), [S6-04 sections 6/9/14/15](S6-04.md), [runbook](verification-runbook.md) |

## Cross-ticket consistency decisions

- All five tickets depend on Sprint 5 closure and final-source reconciliation. The proposal does not claim Sprint 5 is complete or that an authoritative historical Sprint 6 existed.
- All application response paths share the same mapper/required contract. S6-02/03 coordinate frontend fixtures as their contracts grow; S6-04 does not mask a mixed deployment with defaults.
- A revision protects manual decisions only. Background AI changes neither advance it nor justify discarding normal refresh; AI is reconciled through fresh reads.
- No-op detection follows the revision check. Clearing uses stored AI state, clears the current manual timestamp, and does not create a recruitment event, resolve actions, notify or rerun AI.
- History remains recording-ordered. Email date and AI interpretation do not become verified real-world event time or source quotations.
- Synthetic migration checks and Sprint 5's real mailbox preservation are separate. Real-worker fixtures require a nonzero budget; manual corrections and completed AI replay add no provider calls in their isolated intervals.
- Rollback preserves additive data and canonical S6-01 reads; it does not restore AI-first display or delete user corrections.

## Validation boundary

Planning validation checks the complete 21-section ticket structure, local links, documented source paths, command/script names, code-fence syntax and consistency of these corrections. The final save also checks that existing Sprint 5 documents are unchanged and that backend/frontend code remains untouched.

Documentation checks completed on 2026-09-26:

- All 10 Sprint 6 Markdown files have balanced code fences and no trailing whitespace.
- All five ticket files contain sections 1–21 in order and open acceptance checkboxes (105 sections total).
- Local Markdown targets and heading anchors resolve; explicitly qualified existing backend/frontend source paths were checked.
- Documented npm scripts and the installed Prisma CLI entry resolve. The runbook's Node procedure passes syntax checking only; it was not executed.
- Existing Sprint 5 files match their pre-edit SHA-256 hashes. The original root README content is preserved, with a Sprint 6 link appended.
- Backend and frontend remain clean at the inspected commit IDs; tracked documentation passes `git diff --check`.

The [execution record](verification-runbook.md#8-execution-record--pending) remains pending. No migration, test suite, worker, provider call, commit, PR or Linear operation is implied by a resolved planning finding. Implementation acceptance checkboxes remain open intentionally.
