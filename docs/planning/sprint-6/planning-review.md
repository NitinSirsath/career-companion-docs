# Sprint 6 reconciliation and planning review

Reconciled 2026-09-27. The [overview](README.md) and linked five ticket bodies are the authoritative local execution contract, subject to explicit decisions/entry gates. This record explains requirement provenance, corrected assumptions and preserved history; it is not a second plan or implementation report.

## Allocation reconciliation

`58c426c^` contains the complete earlier 21-section tickets. `58c426c` shortened/reassigned them without updating the overview, review or runbook. Requirements omitted from the shortened bodies were not documented as rejected. The current contract therefore carries them into the newer allocation rather than choosing one version wholesale.

| Material | Before 58c426c | 58c426c allocation | Surviving owner |
| --- | --- | --- | --- |
| Canonical reads | S6-01 | Combined S6-01 | S6-01 |
| Manual correction API/revision | S6-02 | Combined S6-01 | S6-01 |
| Evidence/recording semantics | S6-03 | S6-02, with “AI actions” added to title | S6-02; D1 holds undefined action-response scope |
| Worker delivery/matching concurrency | Existing protections in baseline; no dedicated ticket | New S6-03 | S6-03, with evidence-qualified diagnosis |
| Editor/evidence workflow | Detailed S6-04 | Shortened editor S6-04 | S6-04 retains the complete editor and evidence workflow |
| Integrated acceptance/docs | Detailed S6-05 | Architecture/data-preservation S6-05 | S6-05 combines both requirements |

Canonical titles and dependencies are maintained in the [overview table](README.md#f-ticket-list), full ticket bodies and conversion index. Old estimates described the old allocation; 19 points is historical, with re-estimation required rather than an invented new total.

## Requirement traceability and conflict resolutions

| Area | Existing source / conflict | Reconciled requirement and owner |
| --- | --- | --- |
| Effective status | Domain model; old S6-01; shortened S6-01 | S6-01: userStatus ?? aiStatus, USER/AI/UNKNOWN and conflict rule; all 64 combinations; create/list/detail/PATCH share mapper |
| Manual correction | UF-09; old S6-02; shortened S6-01 | S6-01: strict owned set/change/clear API, seven statuses/null; S6-04 editor |
| Revision/concurrency | Old S6-02/04; review R2 | S6-01 owned row lock, dedicated revision and atomic snapshot; S6-04 frozen target/base revision and explicit rebase |
| Clear/no-op | Old S6-02 sections 6/8/21; overview rules | Compare revision before no-op; current identical value preserves timestamps/revision; changed set/clear increments once; clear nulls confirmation time and reveals stored AI/unknown |
| Uncertain-save recovery | Old S6-04; R4/R5; shared runbook; follow-up R6 | No automatic PATCH replay; cancel stale reads, protect newer revisions, refetch; failed reconciliation preserves draft with Save outcome unknown and blocked save. S6-01 separately handles uncertain POST creation through owned reads and deliberate review, without automatic matching or blind repeat POST |
| AI/user ownership | Domain model; old S6-01/02/05 | AI never writes manual fields/revision; manual correction changes no AI state/ledger, events, actions, jobs or providers; completed replay uses stored results |
| Source email evidence | Old S6-03; shortened S6-02; UF-07/08 | S6-02 event/recentEvent metadata is bounded and owner-checked; null unavailable source; foreign emailId also null; no raw content. Action-response scope stops at D1, not silently dropped |
| recordedAt | Old S6-03; domain distinction between recorded and occurred | S6-02 alias of createdAt, ISO serialized, no migration/backfill or inferred occurrence date; Email date remains separate |
| Timeline/history UI | Old S6-03/04; domain manual-provenance rule | S6-04 recording order, AI-only transitions, no equal-state arrow, escaped evidence/AI interpretation; independent section errors; no correction recruitment event/full audit log |
| Worker delivery | Shortened S6-03; investigation; installed pg-boss source | S6-03 all delivered work accounted for; explicit tested single-job or complete-array invariant; no demonstrated batch loss at batchSize=1 |
| Matching concurrency | Shortened S6-03; existing row locks/uniqueness and domain rules | S6-03 deterministic distinct-email/duplicate/stale-auto/user-resolution interleavings; preserve ownership, manual choices and compatible lock order without policy redesign |
| Final architecture/preservation | Both S6-05 versions; runbook | S6-05 explicit architecture boundary checklist plus real-worker/browser integration, manual/replay accounting, fresh/upgrade preservation and accurate operating docs |
| Runtime response validation | Old S6-01/03/04; R5 | S6-01/02 schemas and affected API methods; S6-04 mutation/recovery; missing required keys are errors, true null is valid; no generic unrelated-client rewrite |
| Guarded migration | Old S6-02/05; R3; runbook | S6-01 additive revision only; independent target, overriding .env.test equality, safety helper before CLI, generation before compilation, exclusive synthetic lanes and stop-before-cleanup |
| Sprint 5 baseline | Old unexecuted-draft wording versus execution report/follow-up fixes | All tickets use implemented uncommitted migrated files and real-worker harness; live Gmail/original-data gate stays pending and distinct from engineering completion |

## Removed claims, downgraded assumptions and held decisions

- **Unsupported failure claim removed:** “jobs[0] silently drops batches” and “massive concurrency bugs” were not demonstrated. Installed pg-boss 12.31.0 uses batchSize=1 without an override here. Risk under changed configuration and all-delivered-jobs reliability remain S6-03 verification requirements. Existing matcher locks/tests also preclude calling every race a confirmed bug without a reproduction.
- **Stale baseline claims replaced:** Sprint 5 is not unimplemented; the smoke is no longer worker-disabled. Historical statements/results remain dated below and in the stabilization audit. Live evidence has not been upgraded to verified.
- **Old allocation estimates withdrawn from current use:** new worker scope and combined status scope invalidate mechanically reused ticket points and the 19-point total. Re-estimation is planning work, not scope deletion.
- **No material engineering requirement silently removed.** Manual/no-op/clear, concurrency, runtime validation, uncertain recovery, evidence, migration safety and final architecture/data preservation survive. No mandatory batching, new matching policy, schema redesign or live Gemini test for manual correction is invented.
- **D1 is genuinely unresolved:** title-only “AI actions” and UF-08 traceability do not select action DTOs/endpoints/UI. Keep the promise visible and stop on new action-response design until the owner chooses event-only scope or additional action enrichment with explicit acceptance.
- **D2 is an inherited approval gate:** clear/latest-only provenance, revision/conflict and recording-order semantics are fully specified proposals but no approval record was found. Document reconciliation is not product approval.
- **Live Gmail/original-data entry requirements are retained.** No provider access was requested to edit documentation. Missing evidence cannot be treated as fulfilled or silently demoted to post-sprint work.

## Validation for this reconciliation

Documentation verification only: check all 10 Sprint 6 files, five 21-section ticket bodies and their explicit definition of done, canonical title/dependency consistency, local links/anchors, code fences, runbook snippet syntax, and Git whitespace. Compare before/after hashes of backend/frontend and unrelated Sprint 5 documents to preserve existing work. Actual results are recorded after the checks below; no application suites, workers, migrations or providers are run for this task.

Checks completed for this reconciliation (2026-09-27):

- All 10 Sprint 6 documents have balanced fences; all five tickets contain ordered sections 1–21 and explicit definitions of done. Canonical titles match overview and conversion; dependency/ownership references were reviewed against the final allocation.
- 97 local links/anchors resolve. The runbook Node snippet and 10 shell blocks pass syntax checks only; documented npm script names exist. The migrated sync-path skip is explicitly blocked by a runbook prerequisite, not claimed fixed.
- Git whitespace checks pass. Before/after hashes show changes in exactly the 10 Sprint 6 documents, repository README and dated stabilization-audit pointer. Backend/frontend files, existing Sprint 5 documents, other repository files, Git HEADs and indexes are unchanged from task start.
- No application tests, migration, database/worker/provider operation, commit, PR or Linear action was performed. Sprint 6 execution evidence stays pending; D1/D2 are not marked approved.

## Follow-up review corrections — 2026-09-27

These are documentation corrections to the runtime-validation and safe-teardown requirements already retained above. They do not change the five-ticket allocation, D1/D2 decisions, evidence gates or application code.

| ID / priority | Review finding | Correction in the execution contract | Required implementation evidence |
| --- | --- | --- | --- |
| R6 / P2 | Validating POST-create responses can fail after the row is committed, but recovery was described only for a status PATCH with a known application ID/revision | S6-01 section 6 owns existing Add Application dialog recovery: retain draft and unknown-outcome state, disable automatic/blind repeat POST, reconcile owned paginated reads, and require deliberate review before any new creation. Same-company/role or absence from a page is not proof of request identity/outcome. S6-04 explicitly keeps this separate from status-PATCH recovery; no idempotency API or automatic deduplication is added | Commit a real fixture create, corrupt/drop its 201, and verify no blind second POST. Cover failed reads, unusable response ID, same-company/role records and inconclusive list refresh. S6-05 and runbook carry the integrated scenario |
| R7 / P2 | The contract said to preserve the exact Sprint 5 lifecycle while requiring HTTP intake to stop before cleanup; the current harness closes its HTTP server after fixture deletion | S6-05 explicitly owns extending teardown: stop new browser/API activity, HTTP intake and worker fetch; drain/abort both handler types before destructive cleanup; retain provider isolation until quiescent; clean using live DB access; close remaining resources and release lane ownership last. Failed drain skips deletion and quarantines the lane. The runbook identifies the actual baseline and required change separately | Verify successful and injected-failure teardown with an in-flight HTTP write and active worker; no fixture deletion overlaps either handler. This is future verification, not a claim that the current harness already satisfies the order |

The overview, ticket requirements/acceptance and shared runbook reflect both corrections. Documentation validation is recorded separately from pending implementation evidence; neither finding is marked runtime-fixed by editing this contract.

Follow-up documentation checks passed: all 10 documents have balanced fences, five tickets retain ordered sections 1–21, canonical titles match both indexes, and 97 local links/anchors resolve. The 10 shell blocks and one embedded Node snippet pass syntax checks only; Git whitespace checks pass. A separate before/after snapshot for this follow-up confirms changes only to the overview, S6-01, S6-04, S6-05, this review and the runbook. Backend/frontend files, Sprint 5 and other documentation, Git HEADs and indexes remain unchanged. No application tests, migrations, database/worker/provider operations, commits, PRs or Linear actions were performed.

## Historical review findings — retained and remapped to current owners

| ID / priority | Finding | Applied correction | Documents / required regression evidence |
| --- | --- | --- | --- |
| R1 / P1 | The original proposal did not account for the newly available Sprint 5 draft and risked preserving the old worker-disabled smoke assumptions | Added the complete unexecuted Sprint 5 entry gate, its corrected dependency sequence and ID-reconciliation boundary. Require extending its final real-worker harness with encrypted fixture credentials, history anchor, nonzero isolated AI budget, traffic blocking and safe teardown. Zero-call assertions are scoped to manual operations/completed replay, not the entire fixture run. | [Overview](README.md), [assessment](architecture-review.md), [S6-05 sections 6/11/12/14/15](S6-05.md), [runbook steps 6/7](verification-runbook.md) |
| R2 / P1 | Reading the latest query revision at Save would silently rebase an open editor and bypass the intended 409 conflict | Freeze application ID, draft and base revision when editing starts. Refetch may update visible server state, never the draft token. Conflict requires explicit review and deliberate rebase; no automatic resend. | [S6-01 sections 6/10/14/15](S6-01.md), [S6-04 sections 6/14/15/18](S6-04.md), [S6-05](S6-05.md). Test another device saving plus background refetch before the stale editor first submits. |
| R3 / P1 | Raw migration CLI could mutate a DB before the test helper ran; .env.test overrides and missing Prisma generation made the command order unsafe/incomplete | Specify an independent expected disposable target, load overriding .env.test, validate equality and the existing safety helper before spawning Prisma with the same environment. Require provenance/exclusivity and generation before compilation. | [S6-01 sections 8/14/19](S6-01.md), [S6-05 sections 8/13/14/19](S6-05.md), executable procedure in [runbook steps 1–4](verification-runbook.md). Guard rejection must prevent CLI invocation. |
| R4 / P2 | Delayed reads, automatic mutation retries and lost responses could misrepresent persisted state or overwrite a newer correction | Propagate AbortSignal, cancel relevant reads before mutation/result application, keep target IDs captured, protect newer cached revisions and refetch. Disable automatic mutation retry. Reconcile timeout/network/5xx/invalid-success outcomes by GET; failed reconciliation preserves draft with Save outcome unknown and disables blind save. | [S6-04 sections 6/14/15/18](S6-04.md), [S6-01 recovery](S6-01.md), [S6-05](S6-05.md), [runbook scenario table](verification-runbook.md) |
| R5 / P2 | TypeScript return types do not validate JSON; missing required fields could be displayed as legitimate unknown state | Runtime-parse affected application create/list/detail/mutation and history responses using synchronized Zod schemas. Required keys have no masking defaults. Invalid contracts yield recoverable errors and block editing; true null remains valid. Invalid successful writes enter uncertain-outcome recovery. | [S6-01 sections 6/9/14/15](S6-01.md), [S6-02 sections 6/9/14/15](S6-02.md), [S6-04 sections 6/9/14/15](S6-04.md), [runbook](verification-runbook.md) |

## Historical cross-ticket decisions — requirements retained

- All five tickets depend on Sprint 5 closure and final-source reconciliation. At that review date Sprint 5 was still a proposal; this is superseded by the current Sprint 5 execution report and the baseline section of the reconciled overview.
- All application response paths share the same mapper/required contract. S6-01/02 coordinate frontend fixtures as their contracts grow; S6-04 does not mask a mixed deployment with defaults.
- A revision protects manual decisions only. Background AI changes neither advance it nor justify discarding normal refresh; AI is reconciled through fresh reads.
- No-op detection follows the revision check. Clearing uses stored AI state, clears the current manual timestamp, and does not create a recruitment event, resolve actions, notify or rerun AI.
- History remains recording-ordered. Email date and AI interpretation do not become verified real-world event time or source quotations.
- Synthetic migration checks and Sprint 5's real mailbox preservation are separate. Real-worker fixtures require a nonzero budget; manual corrections and completed AI replay add no provider calls in their isolated intervals.
- Rollback preserves additive data and canonical S6-01 reads; it does not restore AI-first display or delete user corrections.

## Historical validation boundary — 2026-09-26 only

The following checks describe the original 2026-09-26 save, before `58c426c` shortened the ticket files. They do not describe the migrated working tree or establish current runtime acceptance. Links in the review table above now point to the surviving owners; historical ticket IDs map as shown below.

Documentation checks completed on 2026-09-26:

- All 10 Sprint 6 Markdown files have balanced code fences and no trailing whitespace.
- All five ticket files contain sections 1–21 in order and open acceptance checkboxes (105 sections total).
- Local Markdown targets and heading anchors resolve; explicitly qualified existing backend/frontend source paths were checked.
- Documented npm scripts and the installed Prisma CLI entry resolve. The runbook's Node procedure passes syntax checking only; it was not executed.
- Existing Sprint 5 files match their pre-edit SHA-256 hashes. The original root README content is preserved, with a Sprint 6 link appended.
- Backend and frontend remain clean at the inspected commit IDs; tracked documentation passes `git diff --check`.

The [execution record](verification-runbook.md#8-execution-record--pending) remains pending. No migration, test suite, worker, provider call, commit, PR or Linear operation is implied by a resolved planning finding. Implementation acceptance checkboxes remain open intentionally.
