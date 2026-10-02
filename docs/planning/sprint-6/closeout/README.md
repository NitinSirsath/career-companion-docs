# Sprint 6 closeout — finish and stabilize Sprint 6, BYO AI and MCP

Status: **planned, not started (local plan, 2026-10-02).** Starts only after the [migration verification gate](../../migration-verification/README.md) passes. Local IDs; no Linear identity, state or estimate is assumed. Part of the [roadmap](../../README.md).

| Field | Value |
| --- | --- |
| Name | Sprint 6 closeout |
| Kind | Stabilization pass. Fixes, tests and records only. No new features. |
| Honest size | About 2.5 to 3 weeks for one developer. If it must be shorter, defer the MCP-10 focus and feedback polish and the AI-20 waiting-row label (AIF-02) to S8-02; keep everything else. |
| Repositories | backend, frontend, docs; `job-application-automation` for MCP-08 (owner) |

## 1. Goal

Sprint 6, BYO AI and MCP can honestly be called **done**: the defects found by the 2026-10-02 audit that matter for daily use are fixed, every Done claim is backed by a test that can fail, the docs match the code, and acceptance is recorded from new-PC evidence.

## 2. Why this exists

Sprint 6, BYO AI and MCP were all implemented on 2026-10-02 without git, and the audit found the code sound. But:

- every result so far was measured on the old laptop only;
- the owner will use BYO AI with a real key every day after Phase 0, and a few runtime and UI defects show up only then (a systematic provider 400 fails every email; the only recovery path for held AI work is hidden in a hover tooltip);
- two Sprint 6 review tickets are still open (S6-R07, S6-R08);
- some "verified" claims rest on tests that cannot fail (for example the Sprint 6 lock test never sends its request while the lock is held);
- MCP still needs the automation repo change (MCP-08) and the rest of the Antigravity walkthrough;
- many docs still describe old behavior (hosted Gemini key, "MCP not implemented", a 90-day sync scope, CI that does not exist).

None of this is new product scope. It is the work needed before calling the features finished.

## 3. Scope

Included: the eight items below. Each ticket owns its own focused tests.

Not included (explicit non-goals):

- No new features, endpoints, tables or providers.
- No Gmail sync reliability work (S5-FU-01 is Sprint 7) and no scheduler.
- No AI provider certification or catalog status change (AI-15 is Sprint 8).
- No deployment or production hardening (release track).
- No application archive/delete, no rematching, no agenda.
- No rewrite of historical documents: dated banners and one-line corrections only.
- No re-estimation of finished Sprint 5/6 tickets.

## 4. Dependencies

- The [Phase 0 gate](../../migration-verification/README.md#mv-16--gate-record-results-and-decisions) has passed: git restored, every existing check re-run and recorded on the new PC.
- OD-14 is decided before AI-19 starts (it also shapes AI-20 item 22). OD-09 limits AI-19 to a wording change: no disclosure version bump.
- Owner decisions OD-01 and OD-02 are recorded before S6-C02 starts ([roadmap §6](../../README.md#6-owner-decisions)). OD-12 before S6-R08. OD-11 only if MV-14 shows `Origin: null`.
- MCP-08 needs a local copy of `job-application-automation`.

## 5. Ordered tickets

| Order | ID | Title | Size | Why in this order |
| --- | --- | --- | --- | --- |
| 1 | [AI-19](../../byo-ai/AI-19-runtime-safety-fixes.md) | BYO AI runtime safety fixes for daily use | M | Highest risk once a real key is used daily. |
| 2 | [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md) | BYO AI: reachable recovery path and honest status UI | L | The only recovery for held AI work must be reachable. |
| 3 | [S6-R07](../review-2026-10-02/S6-R07-guard-automatic-rematch.md) | Guard automatic re-matching of already matched emails (existing) | S | Matching correctness; prerequisite for S8-01. |
| 4 | [S6-C01](S6-C01-make-done-claims-provable.md) | Make the Done claims provable: tests and lanes | M | Turns "verified" claims into tests that can fail. |
| 5 | [S6-R08](../review-2026-10-02/S6-R08-ux-contract-polish.md) | UX, accessibility and contract polish (existing, amended) | S | Two Sprint 6 DoD items depend on it. |
| 6 | MCP-08 + MCP-09 part B steps 3–5 | Apply the staged automation-repo changes, then finish the Antigravity walkthrough (existing, amended; owner) | S | Can run whenever the automation repo is available. See the [banner in MCP-08-automation-changes.md](../../mcp-feature/MCP-08-automation-changes.md). |
| 7 | [MCP-10](../../mcp-feature/MCP-10-post-verification-corrections.md) | MCP corrections after real-client verification | M | Uses what MV-14 and part B found. |
| 8 | [S6-C02](S6-C02-docs-truth-and-acceptance-records.md) | Docs match the code; record Sprint 6, BYO AI and MCP acceptance | L | Last, because the records cite the results of 1–7. |

## 6. Exit criteria

- [ ] Every ticket above meets its acceptance criteria, with evidence linked.
- [ ] On the new PC, one final run on the closeout head (owned by S6-C02): backend and frontend full suites, typecheck, lint (0 errors), build, contract sync (0 changed), the three migration lanes and the real-worker smoke pass. Counts are recorded.
- [ ] MCP-08 is applied and its three walkthroughs pass; MCP-09 part B steps 1–7 are recorded, or a blocker is recorded with an owner decision.
- [ ] Sprint 6 DoD boxes are ticked only where evidence exists; the entry gate is recorded per OD-01.
- [ ] The BYO AI status table and the MCP plan status reflect reality (AI-15 widened, AI-17 not "Docs done", MCP-09 part B state).
- [ ] The work is committed in git with focused commits.

## 7. Risks

| Risk | Mitigation |
| --- | --- |
| Closeout grows into a feature sprint | Every item must trace to an audit finding in the [state audit](../../state-audit-2026-10-02.md). New ideas go to the roadmap "Not now" table. |
| A fix changes AI or matching behavior beyond its ticket | Each ticket lists its non-goals; matcher policy and AI prompts stay unchanged. |
| Docs pass turns into rewrites | Use dated banners and one-line corrections; historical text stays. |
| MCP-08 waits on the automation repo | It does not block the other items; S6-C02 records its state either way. |
