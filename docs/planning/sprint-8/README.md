# Sprint 8 — Correctable and recoverable results (provisional)

Status: **provisional plan, not started (local plan, 2026-10-02).** Re-plan it after Sprint 7 exits and after about two weeks of real daily use. If the owner decides to deploy first (OD-08), the [release track](../README.md#7-release-track-not-scheduled) replaces this sprint. Local IDs; no Linear identity, state or estimate is assumed. Part of the [roadmap](../README.md).

| Field | Value |
| --- | --- |
| Sprint number | 8 |
| Sprint name | Correctable and recoverable results |
| Size | 4 tickets: 1 L, 2 M, 1 S. About 2 to 3 weeks for one developer. |
| Confidence | Medium. The facts behind each ticket are confirmed; the choice depends on OD-07, OD-08 and what daily use shows. |

## 1. Goal

When the app gets something wrong, the owner can fix it from the UI: a matched email can be moved to another application or unlinked, an ignored email can be linked, and a stuck or waiting email can be recovered without an operator. Gemini is certified on real evidence, and every Google request has a time bound.

## 2. Why this sprint exists

After Sprint 7 the input side is reliable. The remaining confirmed risks are later in the golden path:

- **A wrong match is permanent.** `resolveEmailMatch` accepts only AMBIGUOUS or UNMATCHED emails. An auto-matched, user-matched or ignored email can never be moved; its event, raised AI status and actions stay on the wrong application, and later thread mail follows the wrong link.
- **Stuck emails.** An email left PROCESSING after a hard kill on its last attempt has no recovery path in the UI, and the Gmail page keeps polling.
- **No certified provider.** No provider error mapping has ever been confirmed with a real key, and the evaluation runner scores rate limits as quality failures. The owner uses Gemini daily, so a wrong mapping means wrong cooldowns or refusals.
- **Unbounded Google calls.** The OAuth token refresh and the token routes have no time limit, and SDK retries are on.

## 3. Scope

- Wrong-match correction, after the rematch policy is recorded as ADR-0003 (S8-01).
- Recovery for stuck PROCESSING rows and bounded Gmail page polling (S8-02).
- Gemini certification, with the evaluation-runner fix as step 0 (AI-15, widened).
- Time bounds on every Google and Gmail request (S5-FU-02).

## 4. Non-goals

- No deployment or production hardening (release track).
- No OpenAI or Claude certification unless the owner asks (OD-09).
- No agenda view or temporal extraction contract.
- No retention or account deletion.
- No application archive/delete, including automation-created applications (OD-10).
- No matcher company-suffix or title normalization.
- No notification outbox, snapshot tooling or full sync counters.

## 5. Dependencies and entry conditions

- Sprint 7 exit criteria met and recorded.
- About two weeks of real daily use with the scheduled sync. If that use shows a different top pain (for example the agenda), re-plan this sprint before starting.
- OD-07 recorded as ADR-0003 before S8-01 coding starts.
- OD-09: the owner approves the corrected Gemini data-use text before AI-15 changes the catalog.
- S6-R07 done (closeout) before S8-01.
- OD-08 = no deployment in this window.

## 6. Ordered tickets

| Order | ID | Title | Size | Depends on |
| --- | --- | --- | --- | --- |
| 1 | [S8-01](S8-01-correct-a-wrong-email-match.md) | Let the user correct a wrong email match (ADR-0003 first) | L | S6-R07, OD-07 |
| 2 | [S8-02](S8-02-recover-stuck-and-waiting-emails.md) | Recover stuck and waiting emails without an operator | S | AI-20 |
| 3 | [AI-15](../byo-ai/AI-15-certify-gemini-first.md) | Certify Gemini first (existing, widened; eval-runner fix is step 0) | M | AI-19, OD-09, a real Gemini key |
| 4 | [S5-FU-02](../sprint-5/follow-up-google-request-bounds.md) | Bound every Google and Gmail request | M | S5-FU-01 |

AI-15 and S5-FU-02 can run in parallel with S8-01.

## 7. Exit criteria

- [ ] S8-01: per ADR-0003, a user can move or unlink a matched email (auto-matched or user-matched) and link an ignored one. The old application's effects are retired with a reason, `aiStatus` is recomputed, `userStatus` is untouched, there are no duplicate effects, and no provider is called. Deterministic concurrency tests pass, including later thread mail.
- [ ] S8-02: a PROCESSING row with no write for 20 minutes (`EMAIL_PROCESSING_STUCK_MS`, S8-02 §4.0) shows "Stopped" and offers Manual Retry; Gmail page polling always ends.
- [ ] AI-15: the evaluation runner reports rate limits and refusals separately from quality. Gemini has a passing report for every offered model (pair A and pair B, or the recorded alternative), 2 runs each. Each live error-mapping row is observed or recorded as "not produced", and every mapping change has a test row. Gemini has an owner-approved disclosure with a new version, and catalog status `supported`. Or Gemini stays hidden and the reason is recorded.
- [ ] S5-FU-02: every Gmail and OAuth request has an enforced bound with SDK retries off and cancellation honored; loopback transport tests pass.
- [ ] CI green on main; the smoke and migration lanes (if a migration was added) pass; a dated `execution-report.md` in this folder records counts. No fixture result is called live evidence.
