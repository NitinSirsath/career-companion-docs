# Acceptance register and proposed decisions

Date: 2026-10-03. Local S9/S10/S11 engineering authorized; live acceptance deferred. The owner's instruction completes Sprint 7/8 implementation and leaves real-world acceptance pending. Nothing here reopens their implementation scope.

## Remaining acceptance

| Evidence | Current status | Existing owner/location | What closes it |
| --- | --- | --- | --- |
| Sprint 10/11 owner trial and extraction/v3 qualification | Deferred; local engineering evidence in the execution reports | [S10 report](../sprint-10/execution-report.md), [S11 report](../sprint-11/execution-report.md) | Two complete real model-specific v3 evaluations with prerequisites/disclosure and owner activation decision; dated user workflow observations. |
| Sprint 9 owner task trial | Pending; local engineering checks passed | [Sprint 9 report](../sprint-9/execution-report.md) | Dated owner observations of daily counts, search, status correction and coverage interpretation. |
| Two actual scheduled days, manual overlap and resume after sleep | Pending | S7-05 / Sprint 7 execution report | Dated observations against real Gmail; distinguish on-time runs from startup catch-up. |
| Live Gmail gap/repeat behavior | Pending | S7-02, S5-FU-01/02 | Observed message/checkpoint behavior and repeat without duplication; record redacted evidence. |
| Original personal dataset preservation | Pending, availability unverified here | Existing migration ledger, OD-13, S5-FU-01 FU-5 | Owner-run evidence from the actual dataset; synthetic 1,620-row lanes do not substitute. No search of another machine in this task. |
| Gemini certification | Pending | [AI-15](../byo-ai/AI-15-certify-gemini-first.md) | Existing AI-19 prerequisites, real-key model runs, error rows, approved disclosure. Delivered runner is retained. |
| Antigravity automation walkthrough | Pending | MCP-08, MCP-09 part B | Real-client evidence with the owner's tool; SDK smoke is separate. |
| Remote publication/CI | Pending and out of scope here | S7-01, execution reports | Only in a separately authorized remote workflow. Not a local planning gate. |
| Deployment readiness | Unverified, no deployment scheduled | Release track / reserved ADR-0004, AI-17 | Hosting decision plus operational/security and supervised-release evidence. |

On a real-world failure: capture the first failing boundary, expected/actual behavior, minimal redacted reproduction and affected commit. Create a focused local defect ticket only then. A pending observation is not a defect or permission for speculative changes.

Live keys, sending notifications, personal database restoration and external service runs are not performed in this implementation. Historical provider/release obligations retain their IDs and remain separately scoped.

## Proposed choices for future work

OD-16 and OD-17 are **ACCEPTED for Sprint 9 on 2026-10-03**, following the owner’s authorization to resolve the review and implement Sprint 9. OD-18..20 are **ACCEPTED for local Sprint 10/11 engineering on 2026-10-03**. OD-21 remains PROPOSED; no release decision is implied.

| ID | Proposed choice | Reason | Affected work |
| --- | --- | --- | --- |
| OD-16 | Sprint 9 first: daily action workspace and application discovery. | Uses trusted existing data with a small boundary. | S9 kickoff |
| OD-17 | User's browser IANA timezone for workspace display; explicit fallback Asia/Kolkata with visible label. The Gmail schedule stays Asia/Kolkata. | “Today” should match the visible local date; no scheduler or stored-preference expansion. | S9-01/02 |
| OD-18 | Agenda supports interviews and assessment due dates only; AI supplies tentative candidates, and the user confirms unclear date/time/zone and cross-email changes. | Avoid invented instants and speculative reschedule matching. | S10; accepted ADR-0005 |
| OD-19 | Future follow-ups and snooze are in-app features; no outbound reminder schedule or automatic recruiter contact. | Useful personal control without another delivery system. | S11-01/02 |
| OD-20 | Archive is reversible visibility/suppression, separate from CLOSED and deletion; linked mail remains tracked. | Preserve evidence and avoid corrupting status or receipt identity. | S11-03 |
| OD-21 | Continue local owner-only workflow; consider release only on an explicit deployment/wider-use decision. | Scope matches current instruction. | Release scheduling |

OD-04/05/06/07/15 already have implementation decisions in Sprint 7/8 reports and ADR-0003; do not ask to decide them again. ADR-0004 stays reserved for hosting. ADR-0005 is accepted for local temporal semantics; live v3 activation is separately gated.

## Entry checks for later implementation

- Sprint 9, 10 and 11 implementation authorized 2026-10-03; live acceptance is explicitly deferred and does not block development.
- Record decisions for the selected sprint only; no blanket approval of later sprints.
- Confirm local change status and retain unrelated work. No GitHub dependency for local engineering evidence.
- Rerun checks appropriate to actual code changes when implementing, using the existing guarded fixture databases. Never use a real user database as a test target.
- Sprint 9 needs no provider certification to build or validate read-only views with fixtures. Real-product usefulness still depends on live input coverage, which the UI must disclose.
- Sprint 10's extraction/v3 cannot be activated for live use on the strength of extraction/v2 certification; qualify the new contract explicitly.
- Historical AI-19/20 changes, if selected later, remain separate work under those IDs and are not hidden in future feature tickets.

## Sprint 10/11 implementation authorization — 2026-10-03

The owner authorized implementation in the existing verified scratchpad, preserving uncommitted Sprint 9 work. OD-18, OD-19 and OD-20 are accepted for their documented bounded engineering scope, including archived-only MCP intake review. No external tracker, push, PR, branch move or live Gmail/MCP acceptance is authorized. Sprint 9 owner trial and all provider/live acceptance gates remain pending and do not block engineering. Extraction/v3 remains default-off.
