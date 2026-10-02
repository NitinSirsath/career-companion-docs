# AI-20 — BYO AI: reachable recovery path and honest status UI

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 6 closeout — order 2 of 8 ([closeout plan](../sprint-6/closeout/README.md)) |
| Repository | career-companion-frontend only. No backend change is needed; if one seems needed, stop and re-scope. |
| Size / priority | L / the only recovery for held AI work must be reachable |
| Depends on | [Phase 0 gate](../migration-verification/README.md#mv-16--gate-record-results-and-decisions) passed; [AI-19](AI-19-runtime-safety-fixes.md) first (closeout order 1, same feature area; its pause-storage choice shapes item 22) |
| Blocks | [S8-02](../sprint-8/S8-02-recover-stuck-and-waiting-emails.md) (builds on the visible retry action); the BYO AI acceptance record in [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md#aif--byo-ai-frontend-and-docs) AIF-03, AIF-13, AIF-14, AIF-12, AIF-02, AIF-15, AIF-05; [ADR-0001](../../architecture/decisions/ADR-0001-user-provided-ai.md) decision 10; [issues.md](issues.md) AI-11 and AI-12 acceptance; [plan](README.md) §3.11 and §9; [AI-19](AI-19-runtime-safety-fixes.md) §5 (per-user pause copy handed to this ticket) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

A keyboard or touch user can reach "Manual Retry" and approve one more AI attempt for a held email. The AI page and the Gmail page tell the truth about AI status, waiting mail and errors. Tests prove the AI-11 and AI-12 acceptance items that are marked Done today.

## 2. Why it exists

- ADR-0001 decision 10 makes user-approved retry the **only** recovery for held AI operations. Its only entry point sits inside a hover tooltip. Keyboard users lose it on Tab, and touch users cannot open it (AIF-03, confirmed, medium).
- The owner uses BYO AI with a real key every day after Phase 0. Wrong status text ("Key verified" next to "rejected your API key"), dead ends and unexplained "PENDING" rows show up in that daily use (AIF-02, AIF-12, AIF-13, AIF-14).
- The AI page repeats the live-region and focus pattern that S6-R08 found elsewhere (AIF-15). [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) explicitly hands the AI page to this ticket.
- AI-11 and AI-12 are marked Done, but 5 of 10 access reasons, the switch flow, the INCONCLUSIVE save and re-consent have no test. Seven older route test files run without an AI settings mock (AIF-05).

## 3. Current behavior

All paths are in `career-companion-frontend-main` unless they start with `backend/`. Checked against the code on 2026-10-02.

**Retry entry point (AIF-03)**
- `src/routes/gmail.tsx:347-387` (`GmailPage`, AI Status cell): when `msg.processingErrorDetails` is set, an icon-only `TooltipTrigger` (line 349, no `aria-label`) opens a `TooltipContent` that holds the error label, details, stage, failed time and the "Manual Retry" button (lines 371-383).
- `src/components/ui/tooltip.tsx:9-33` (`TooltipContent`) renders through `BaseTooltip.Portal`, at the end of `body`. Per the verifier's reading of Base UI 1.8.0: hover-open is mouse-only, focus-open needs `:focus-visible`, and the next Tab leaves the trigger and closes the tooltip. Not checked in a real browser or device.
- `gmail.tsx:113-125` (`retryMutation`): only a 409 `AI_RETRY_NEEDS_APPROVAL` opens `RetryAnywayDialog`. No other frontend code calls `api.retryEmail`. The backend approval path is only `backend/src/routes/email.ts:148` (`router.post('/:id/retry')`).
- Tests reach the button only by firing `mouseEnter`/`focus` and clicking: `src/tests/ai-status.test.tsx:125-135` (`clickManualRetry`) and `src/tests/processing-refresh.test.tsx:82-96` ("opens a window after a manual retry whose response was lost"). Under jsdom, Base UI counts every focus as `:focus-visible` (verifier), so these tests prove nothing about keyboard reach. The smoke hovers, then clicks through `page.evaluate` (`scripts/smoke-stabilization.mjs:860-864`).

**Retry dialog (AIF-13)**
- `gmail.tsx:119` casts `err.details as AIRetryApprovalDetails`. `src/components/ai/RetryAnywayDialog.tsx:32` and `:42` call `details.operations.some/.map` with no check. `AIRetryApprovalDetailsSchema` (`src/contracts/email.ts:40-52`) is never used at runtime in either repo.
- `backend/src/routes/email.ts:217` sends `currentProvider: config?.provider ?? null`. Null is reachable: `removeSettings` (`backend/src/services/ai/settings.ts:293-297`) deletes only the configuration, so held operations stay. `providerName(null)` returns "Your AI provider" (`src/lib/aiLabels.ts:5-6`), so `RetryAnywayDialog.tsx:51` reads "on your Your AI provider account".
- The "Retry anyway" button (`RetryAnywayDialog.tsx:71-73`) is always offered. When access is not READY the backend refuses safely with 409 `AI_ACCESS_UNAVAILABLE` (`backend/src/routes/email.ts:224-236`); nothing is charged. The dialog then shows "Fix AI access on the AI provider page before retrying." (`gmail.tsx:30-31`) with no link.

**Status panel (AIF-14)**
- `src/components/ai/AIStatusPanel.tsx:78` shows "Key verified" whenever `access.verified`. The API sets `verified: !!config?.verifiedAt` (`backend/src/services/ai/settings.ts:93`, `readSettings`). No rejection path clears `verifiedAt` (check rejection at `settings.ts:281-285`). This matches the backend's documented meaning ("last definitive verification success"); the bug is the UI wording.
- `src/routes/ai.tsx:32-35` (`choose`) is the only place that clears `savedNote`. Remove lives in `AIStatusPanel.tsx:64-69` with no callback to the page, so "Connected. Waiting emails are being processed." stays above the picker after Remove.
- `aiLabels.ts:20` (`time`) formats `resumesAt` as `h:mm a`. For `SAFETY_LIMIT` the reset is the next UTC midnight (`backend/src/services/ai/access.ts:104`, `deriveAccess`), up to about 24 hours away, so it can read as a time earlier today.
- `PAUSED` copy (`aiLabels.ts:84-85`) is one generic line. Today `PAUSED` comes only from the global switch `AI_USER_DAILY_CALL_LIMIT=0` (`access.ts:92-93`, `deriveAccess`), and then `safetyLimit.callsPerDay` is 0 (`backend/src/services/ai/settings.ts:102`, `readSettings`). AI-19 §4 item 1 plans a per-user pause after repeated provider 400s, shown with the same reason, and leaves its message to this ticket (AI-19 §5).

**Sample test and Check again (AIF-12)**
- The backend sends `AI_SAMPLE_FAILED` only as 502 with `details: { kind }` (`backend/src/services/ai/settings.ts:351`, `runSampleTest`). `kind` is `INVALID_OUTPUT`, `INVALID_REQUEST` or `OUTCOME_UNKNOWN`.
- `ApiError.outcomeUncertain` is true for any status >= 500 (`src/api/client.ts:67-70`). `sampleProblem` checks it first (`src/components/ai/SampleTestPanel.tsx:10`), so the `AI_SAMPLE_FAILED` branch (line 13) never runs. A definite unusable answer is shown as "did not finish".
- Check again's `onError` (`AIStatusPanel.tsx:59-62`) always shows "The check could not be completed." The 429 `AI_VERIFY_RATE_LIMITED` daily cap (`settings.ts:120-123`, `reserveVerification`; 20 checks a day, `MAX_DAILY_VERIFICATIONS` in `backend/src/services/ai/usage.ts:15`) is never explained. The settings response has no verification count, so nothing else on the page explains it either.

**Waiting rows after a limit clears (AIF-02)**
- The backend has no scheduler. Waiting mail is re-offered only by a sync, a save or a verified check, and `reofferPendingEmails` does nothing unless access is READY (`backend/src/services/gmailSync.ts:31-32`).
- On `/gmail`, once `aiSettings` refetches as READY: `aiWaiting` is false (`gmail.tsx:75`), the notice hides (`aiLabels.ts:32`), rows show raw "PENDING" (`gmail.tsx:345`), and the messages query polls every 2 s while any PENDING row is on the page (`gmail.tsx:81-84`). Nothing changes those rows until the user syncs.
- The gap appears only after `aiSettings` refetches (window focus, remount, navigation or the refresh window); the query has no `refetchInterval` (`gmail.tsx:71-74`). The 2 s poll runs only while the tab is visible (TanStack default). Both per the verifier.
- On `/ai`, the "… waiting for AI. Sync Gmail to continue." line (`AIStatusPanel.tsx:123-134`) has no state check. It shows in every configured state, including NEEDS_ATTENTION and LIMITED before `resumesAt`, where a sync does nothing.
- `settings.waitingEmails` counts **all** PENDING emails (`backend/src/services/ai/settings.ts:75`), including mail queued seconds ago. It cannot tell "waiting" from "being processed".
- A bounded refresh window already exists: `src/lib/processingRefresh.ts:6-7,24-27` (120 s window, 4 s refresh). It opens on sync and retry (`gmail.tsx:109,123`) and on a new `lastSyncedAt` (`src/components/ProcessingRefreshObserver.tsx:28-34`). Closing the window notifies no subscriber (listeners run only in `startProcessingRefresh` and `resetProcessingRefresh`).

**Accessibility on the AI page (AIF-15)**
- Live regions are mounted already holding text: `ai.tsx:50-54` (`savedNote`), `AIStatusPanel.tsx:85-95` (access copy) and `:137-141` (`checkNote`), `SampleTestPanel.tsx:69-70` (result).
- After a save, `onDone` switches to the status view (`ai.tsx:61-64`). The form and its focused Save button unmount; nothing moves focus. Cancel and picking a provider lose focus the same way. Focus targets that exist today: the status panel's `h3` (`AIStatusPanel.tsx:76`) and the form's `h3` (`ProviderSetupForm.tsx:125`). The picker is a `<section aria-label="Choose an AI provider">` with no heading of its own (`ProviderPicker.tsx:20`).
- `src/components/ui/dialog.tsx:5-19` passes `onOpenChange` straight to Base UI. `RetryAnywayDialog.tsx:34` and the remove dialog (`AIStatusPanel.tsx:162`) close on Escape or an outside press while a request is pending; only the buttons are disabled. On Gmail, `onCancel` also calls `approveMutation.reset()` (`gmail.tsx:284-287`), so an error from the in-flight approval is never shown. The row still refreshes through the mutation's `onSettled`.

**Test gaps (AIF-05)**
- Status copy is tested for 5 of the 10 access reasons (`AIAccessReasonSchema`, `src/contracts/ai.ts:14-25`): the table at `src/tests/ai-settings.test.tsx:194-206` plus `ai-status.test.tsx:74-94`. Untested: `KEY_REJECTED`, `PROVIDER_UNSUPPORTED`, `KEY_UNREADABLE`, `PROVIDER_UNAVAILABLE`, `PAUSED` (`aiLabels.ts:39-44, 57-68, 74-78, 84-85`).
- No test for the INCONCLUSIVE save note (`ai.tsx:18`), the switch flow (`ai.tsx:77-95`; fixtures offer only `['gemini']`, and `ai-settings.test.tsx:190-191` asserts there is no Switch button), or re-consent when `consent.current` is false (`src/components/ai/ProviderSetupForm.tsx:71`). The backend does not require re-consent for a same-provider update, so the frontend is the only guard for that case.
- Seven older test files render `/` or `/gmail`, which query `getAISettings` (`AIAccessNotice.tsx:8-11`, `gmail.tsx:71-74`), with an `api` mock that lacks it: `gmail`, `ambiguous-matches`, `action-queue`, `unmatched-emails`, `stabilization-ui`, `status-correction` and `processing-refresh` (all `src/tests/*.test.tsx`). The AI query fails quietly with a TypeError. Three of them call `vi.resetAllMocks()` in `beforeEach` (`processing-refresh.test.tsx:35`, `stabilization-ui.test.tsx:18`, `status-correction.test.tsx:54`).

**Suspected, not verified:** the exact touch and screen-reader behavior above comes from reading code. No real device, browser keyboard pass or screen reader was used. The AI-11 box "Keyboard-only and mobile layout checks pass" (issues.md, AI-11) is unchecked and has no recorded evidence.

## 4. Scope

**A. Reachable retry (AIF-03)**
1. In the Gmail table, rows that offer retry today (`processingErrorDetails` set) show the error label, a visible "Details" disclosure (native `<details>`/`<summary>` or a button with `aria-expanded`) and a visible "Manual Retry" button. No hover or tooltip is needed for any of them. Which rows offer retry does not change.
2. The retry button's accessible name starts with its visible text and names the email, for example `Manual Retry for "<subject>"`.
3. Remove the tooltip and `TooltipProvider` from `gmail.tsx`. Nothing else in that file uses them.
4. Keep `RetryAnywayDialog` and its rules: one approval per confirmation, never resent automatically, an uncertain result re-reads and never resends.

**B. Safe retry dialog (AIF-13)**
5. Parse the 409 details with `AIRetryApprovalDetailsSchema.safeParse`. **Recommendation: fail closed** (implementation choice; no OD needed). If parsing fails, do not open the dialog; show the server's message in the existing retry alert. Approving a possible extra charge needs the details the user is asked to judge. Note: that alert (`gmail.tsx:269-273`) hides every `AI_RETRY_NEEDS_APPROVAL` error today, so its condition must change.
6. When `currentProvider` is null, the dialog shows "Set up AI first". When the page's `aiSettings` is loaded and access is not READY, it shows the access title from `accessCopy`. In both cases it shows a link to `/ai` and no "Retry anyway". If `aiSettings` has not loaded and the provider is set, keep today's behavior; the backend refuses safely.

**C. Honest status panel (AIF-14)**
7. Show "Key verified" only when `access.state === 'READY'` and `verified` is true. When `verified` is true but the state is not READY, show "Key saved" (not "not yet confirmed"). When `verified` is false, keep today's text. The last-checked time stays as today. Do not change the backend meaning of `verifiedAt`.
8. Clear the saved note after a successful Remove, and whenever settings come back with `configured: false`.
9. `resumesAt` text includes the day when it is not today in the user's time zone (use `MMM d, h:mm a`, as the panel already does for "last checked", `AIStatusPanel.tsx:79`). Applies to all reasons that use `time()`.

**D. Specific errors (AIF-12)**
10. `sampleProblem` checks `err.code === 'AI_SAMPLE_FAILED'` before `outcomeUncertain`. `details.kind` `INVALID_OUTPUT` or `INVALID_REQUEST` → "The provider did not return a usable result for the sample email." `OUTCOME_UNKNOWN`, a missing or unknown kind, and any other 5xx → the existing "did not finish" text.
11. Check again: a definitive HTTP error with a code (for example 429 `AI_VERIFY_RATE_LIMITED`) shows the server message, which `ApiError` documents as safe to display (`client.ts:51-54`). An uncertain error keeps "The check could not be completed." Both still refresh `aiSettings`.

**E. Honest waiting rows and bounded polling (AIF-02)**
12. PENDING rows poll at 2 s only while the processing-refresh window is open, and still only when access is not known to be waiting (today's `!aiWaiting`). Outside the window, PENDING rows do not poll. The PROCESSING polls stay as they are (S8-02 owns stuck PROCESSING).
13. When access is READY and the window is closed, PENDING rows read "Waiting — sync to continue" (final wording is the implementer's, plain and short). Inside the window, keep today's `PENDING` label; that mail is likely being processed. Not READY keeps "Waiting for AI". Do not base the label on `waitingEmails`.
14. On `/ai`, show the "Sync Gmail to continue" link only when access is READY. In other states show only the waiting count; the access copy explains the fix.

**F. Accessibility on the AI page and both dialogs (AIF-15)**
15. Keep one persistent, initially empty live region for each message (`savedNote`, `checkNote`, the panel's access copy, the sample-test outcome) and change its text. For the sample test, announce one short line; keep the result table outside the live region.
16. Move focus after view changes (targets get `tabIndex={-1}` where needed): after a save, to the status panel heading; after picking a provider or "Change models or key", to the form heading; after Cancel, back to the control that opened the form, or to the status panel heading if that control is gone (the switch picker unmounts); after Remove, to the provider picker section.
17. While a request is in flight, `RetryAnywayDialog` and the remove dialog ignore close requests (Escape, outside press). Do it in the controlled `onOpenChange`; no change to `ui/dialog.tsx` is required.

**G. Prove AI-11 and AI-12 (AIF-05)**
18. One table test over all 10 access reasons: the 9 configured reasons on `/ai` (badge, title, detail, fix link when present), `NOT_SET_UP` on the notice. Include both `PAUSED` variants if item 22 lands.
19. Tests for the INCONCLUSIVE save note, the switch flow with two offered providers (status stays during the switch; a 422 keeps the old status; consent and key required), and re-consent when `consent.current` is false.
20. A shared default: add `makeAISettings()` (READY) to `src/tests/fixtures.ts`. The seven older files add `getAISettings` to their `api` mock and resolve it in `beforeEach`, not in the mock factory, because three of them call `vi.resetAllMocks()`.
21. Record a manual keyboard-only and narrow-screen pass of `/ai` and the Gmail retry flow on the new PC, so S6-C02 can decide the AI-11 box with evidence.

**H. Per-user pause copy (AI-19 §5 handoff)**
22. Only if AI-19 shipped its per-user pause as access reason `PAUSED` (its recommended storage): `accessCopy` tells the two pauses apart by `settings.safetyLimit.callsPerDay`. 0 is the global pause; keep today's text. Above 0 is the per-user pause: say the provider refused repeated requests, and that saving the settings again ("Change models or key", for example with another model) resumes processing (AI-19 §4 item 1.5). No contract change. If AI-19 chose another storage, record what was done instead in the execution report.

**Decision on the backend status code.** The audit asked whether `AI_SAMPLE_FAILED` should become a 4xx. Not in this ticket: `OUTCOME_UNKNOWN` really is uncertain, and item 10 fixes the UI without a contract change. No owner decision (OD-nn) is needed for this ticket. Item 22 follows the owner's AI-19 pause-storage choice; it adds no decision of its own.

## 5. Out of scope

- The zero-provider empty state on `/ai` (AIF-01) — [release track](../README.md#7-release-track-not-scheduled), per OD-08.
- Recovery for stuck PROCESSING rows and their polling (S56-14) — [S8-02](../sprint-8/S8-02-recover-stuck-and-waiting-emails.md).
- Any backend change: retry semantics, `verifiedAt`, error codes or statuses, or a "last verified" date in the contract.
- Any provider or catalog change (OD-09; AI-15). The two-provider switch test uses fixture data only.
- Save/check client deadlines and the consent text — AI-19.
- Rendering `fix.to` as a new button on `/ai`. AIF-14 part 2 was overstated: the panel already has "Change models or key" and "Switch provider" (`AIStatusPanel.tsx:147-154`).
- `Dialog.Description` on the two dialogs (a nice-to-have, not a WCAG failure, per the verifier).
- The same live-region pattern in `AIAccessNotice` (`AIAccessNotice.tsx:15`, dashboard and Gmail page). AIF-15 does not cite it and no ticket owns it; note it in the execution report. `StatusEditor` is S6-R08's.
- The `App.test.tsx` placeholder (unrelated to AI-11/AI-12; noted in S7-04 §5).
- Wording for scheduled sync. S7-05 will make waiting mail resume without a click and should revisit the "sync to continue" copy then.

## 6. Likely files and components

- `src/routes/gmail.tsx` — row action, details disclosure, safeParse, READY check, bounded PENDING poll, label, blocked dismissal.
- `src/components/ai/RetryAnywayDialog.tsx` — not-ready branch, `/ai` link, pending guard.
- `src/components/ai/AIStatusPanel.tsx` — "Key verified" rule, Check again errors, READY-only sync hint, live regions, remove callback and pending guard, heading focus target.
- `src/components/ai/SampleTestPanel.tsx` — error order, live region.
- `src/routes/ai.tsx` — saved note lifecycle, live region, focus moves.
- `src/lib/aiLabels.ts` — `time()` with a day; `PAUSED` copy (item 22).
- `src/components/ui/tooltip.tsx` — unchanged. `gmail.tsx` is its only importer today, so it becomes unused; leave it in place.
- Tests: `src/tests/ai-status.test.tsx`, `ai-settings.test.tsx`, `processing-refresh.test.tsx`, `fixtures.ts`, and the seven files in item 20.
- Smoke: `scripts/smoke-stabilization.mjs` (scenario "BYO AI: an uncertain outcome is held…", lines 849-874).
- Docs: frontend `README.md` ("AI provider (ADR-0001)" section).

## 7. Implementation notes

- **Data model and migrations:** none.
- **API and shared contracts:** none. Use `AIRetryApprovalDetailsSchema` from `src/contracts/email.ts` as it is. Do not edit `src/contracts/*` in the frontend; they are copied from the backend by `scripts/sync-contracts.mjs`. For `details.kind` (item 10) use a small local zod check; no shared schema exists for it. `npm run sync-contracts` must report 0 changed.
- **Background jobs:** none. Polling only gets narrower.
- **Window close:** closing the refresh window notifies no one. To switch the row label at the end of the window, read `getProcessingRefreshUntil()` through `useSyncExternalStore(subscribeProcessingRefresh, …)` and set a timeout to re-render at `refreshUntil`. Do not rely on a refetch; identical data does not re-render.
- **Label trade-off:** a large backlog can still be processing after the 120 s window. Those rows then read "sync to continue". That is safe: a re-sync re-offers PENDING mail, and enqueueing uses a per-email singleton key (`backend/src/jobs/emailProcessingJob.ts:19-20`), so no duplicate job starts within 300 s.
- **Time zones in tests:** `time()` formats in local time. Pin "now" with `vi.setSystemTime` and build the expected string with the same `date-fns` format, instead of hard-coding a zone.
- **Design rules:** use the `Button` primitive, semantic tokens, visible `focus-visible` rings and `rounded-none` ([AI_UI_RULES.md](../../AI_UI_RULES.md) positive rules 2, 3, 5 and 6; negative rule 2). Today's icon-only trigger (`gmail.tsx:349`) has no `aria-label` (positive rule 6); the new controls have visible text. The table already scrolls sideways on narrow screens (`gmail.tsx:304`); the action must stay reachable there.
- **Polling detail:** `refetchInterval` (`gmail.tsx:81-88`) can read `getProcessingRefreshUntil() > Date.now()` directly. It is re-evaluated after each fetch, so the PENDING poll stops at most one interval after the window closes.
- **Failure and recovery:** a malformed 409 fails closed with a message; the row stays retryable. A dialog cannot be dismissed mid-request, so the user always sees the outcome or the "could not be confirmed" text.
- **Compatibility and rollback:** frontend only, no stored state. Roll back by reverting the commit.

## 8. Dependencies

- Phase 0 gate (MV-16) passed, with the frontend suite green on the new PC.
- AI-19 lands first. It may change the save/check deadlines and the consent text that these tests render. Rebase on it. Item 22 depends on how AI-19 stored its per-user pause; the rest has no code dependency.
- S6-R08 (order 5) changes `StatusEditor` and contract dates. Its missing `RETRY_RECENTLY_QUEUED` wording test should use the visible retry button from this ticket.
- S6-C01 (order 4) runs after this ticket. It changes backend tests only, so it shares no file with this ticket (S6-C01 §8).
- S8-02 depends on this ticket. S6-C02 records BYO AI acceptance using this ticket's evidence.

## 9. Security and privacy

- Approval stays explicit: `acceptPossibleDuplicateCharge: true` is sent only from the confirm button, at most once per confirmation. Nothing resends after an uncertain result.
- The visible details show the same stored text the tooltip shows today: `processingErrorDetails`, at most 300 characters (`backend/src/jobs/emailProcessingJob.ts:41-57`, `describeFailure`). No new data is shown.
- Parse failures never log or render raw `details`.
- No key handling changes. The key tests in `ai-settings.test.tsx` must still pass unchanged.

## 10. Acceptance criteria

- [ ] With no hover, focus or mouse events, a FAILED row with error details shows "Manual Retry" as a `button` and its details through a disclosure (component test).
- [ ] In the smoke, "Manual Retry" is reached with the Tab key and pressed with Enter, and "Retry anyway" is approved by keyboard. On a 390×844 touch viewport it is opened with a real tap. No `page.evaluate` click is used for this path.
- [ ] Malformed 409 details do not crash the page and do not open the dialog; the user sees a message.
- [ ] With `currentProvider: null`, or access not READY, the dialog shows no "Retry anyway", shows the matching text and links to `/ai`. It never says "your Your AI provider".
- [ ] "Key verified" never appears unless the state is READY.
- [ ] If item 22 lands: a per-user `PAUSED` (`callsPerDay > 0`) shows its own cause and resume step; a global `PAUSED` (`callsPerDay` 0) keeps today's text.
- [ ] After Remove, no "Connected…" note is shown.
- [ ] A `SAFETY_LIMIT` reset on another day shows the day.
- [ ] A 502 `AI_SAMPLE_FAILED` with `INVALID_OUTPUT` shows the "usable result" text; with `OUTCOME_UNKNOWN` it shows "did not finish".
- [ ] A 429 on Check again shows the daily-cap message.
- [ ] With access READY and a PENDING row: polling runs while the window is open and stops after it closes; the row then reads the "sync to continue" label. Not READY still shows "Waiting for AI" with no polling.
- [ ] `/ai` shows "Sync Gmail to continue" only when READY.
- [ ] The saved-note, check-note, access-copy and sample-test live regions exist empty before their message and are the same element after it.
- [ ] After a save, focus is on the status heading; after picking a provider, on the form heading; after Cancel, on the opening control (or the status heading if it is gone); after Remove, on the picker section.
- [ ] While an approval or a remove is pending, Escape does not close its dialog.
- [ ] All 10 access reasons, the INCONCLUSIVE note, the two-provider switch and re-consent are covered by tests that fail if the copy or flow breaks.
- [ ] The seven older route test files mock `getAISettings` and no longer hit the TypeError.
- [ ] A manual keyboard-only and narrow-screen pass of `/ai` and the Gmail retry flow is recorded with date and browser.

## 11. Testing

**Add or change**
- `src/tests/ai-status.test.tsx`: replace `clickManualRetry` with a no-hover path; add malformed details, null provider, not READY, Escape while pending, READY + PENDING label and bounded poll (fake timers with `shouldAdvanceTime`, as in `processing-refresh.test.tsx` "expires instead of polling forever").
- `src/tests/ai-settings.test.tsx`: grow the table at lines 194-206 to every reason; add "Key verified" only when READY, INCONCLUSIVE note, switch, re-consent, note cleared after Remove, Check again 429, sample-test `INVALID_OUTPUT` vs `OUTCOME_UNKNOWN`, persistent live regions, focus moves, remove dialog Escape while pending, day in `SAFETY_LIMIT`.
- `src/tests/processing-refresh.test.tsx:82-96`: click the visible button instead of the tooltip.
- `src/tests/fixtures.ts` and the seven older files (item 20).
- `scripts/smoke-stabilization.mjs:849-874`: keyboard and tap steps. The smoke has no touch emulation today (it only sets a 390×844 viewport, lines 957 and 996); use Puppeteer's `hasTouch`/`isMobile` viewport and a real tap. Do the tap path first and cancel the dialog, so the existing assertion of one approved retry (`approvedRetries: 1`, lines 871-874) still holds. If Base UI's Escape cannot be driven in jsdom, cover Escape here and say so in the report.
- No `@testing-library/user-event` is installed. The real Tab-key path is proven in the smoke, not in jsdom; do not add a dependency for it.

**Commands (frontend, from `career-companion-frontend-main`)**
```
npm run sync-contracts        # expect 0 changed
npm run typecheck
npm run lint                  # 0 errors
npx vitest run src/tests/ai-status.test.tsx src/tests/ai-settings.test.tsx src/tests/processing-refresh.test.tsx
npm test                      # full suite; record the new count (baseline 154 on the old laptop)
npm run build
```

**Smoke (browser-visible change).** Backend code is unchanged, but the smoke needs a built backend. From `career-companion-backend-main`: `npm run db:generate && npm run build`, then `TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<empty smoke DB URL> node scripts/guarded-migrate.cjs`. From the frontend: `SMOKE_DATABASE_URL=<same URL> node scripts/smoke-stabilization.mjs`. Expect PASS and residue 0. `SMOKE_SCREENSHOTS=<dir>` is optional. Fixture AI results are not live evidence.

## 12. Documentation updates

- Frontend `README.md`, "AI provider (ADR-0001)": pending rows are polled only inside the refresh window and then read "sync to continue"; retry is a visible row action.
- [user-flows.md](../../product/user-flows.md) UF-10 "Retry anyway" line: only if the button label changes.
- Leave the AI-11/AI-12 boxes in [issues.md](issues.md) to S6-C02; it ticks them only with the evidence this ticket records (AI-11's "Keyboard-only and mobile layout checks pass" box is unchecked today).

## 13. Definition of done

- Every acceptance criterion above is met, with evidence linked.
- Frontend sync-contracts (0 changed), typecheck, lint (0 errors), full tests and build are green on the new PC (in CI once S7-01 exists). The smoke passes with residue 0.
- The keyboard and touch retry path, and the `/ai` keyboard and narrow-screen pass, are verified by hand and recorded.
- Docs in §12 are updated.
- One focused commit in the frontend repo.
- Evidence (commands, test counts, smoke line, manual pass) is recorded in the closeout execution report, `docs/planning/sprint-6/closeout/execution-report.md` (created when the closeout runs).
