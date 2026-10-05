# Migration Pending Changes

This file tracks changes made on the personal PC that need to be migrated or reconciled with the office laptop (where Sprint 7 and 8 are being developed). This will help the AI identify and re-apply these changes smoothly to avoid or resolve merge conflicts.

## Date: 2026-10-03 (Local time)
**Change:** Stop infinite frontend polling on the AI Settings page when there are no waiting emails.
**Commit:** `fix(frontend): Do not start bounded refresh window if there are no waiting emails`

**Files Modified:**
- `career-companion-frontend/src/components/ai/ProviderSetupForm.tsx`
- `career-companion-frontend/src/components/ai/AIStatusPanel.tsx`
- `career-companion-frontend/src/routes/ai.tsx`
- `career-companion-frontend/src/tests/ai-settings.test.tsx`

**Logic Changed:**
1. **`ProviderSetupForm.tsx`**: Updated `onDone` callback signature to accept `waitingEmails`. Changed `startProcessingRefresh()` to only trigger if `saved.waitingEmails > 0`.
2. **`AIStatusPanel.tsx`**: Updated the "Check again" mutation success handler to only call `startProcessingRefresh()` if `next.waitingEmails > 0`.
3. **`ai.tsx`**: Updated the UI state to conditionally display `"Connected. Waiting emails are being processed."` ONLY when `waitingEmails > 0`, otherwise it displays `"Connected."`.
4. **`ai-settings.test.tsx`**: Injected `waitingEmails: 1` into the mock save response to satisfy the smoke test expectation.

---
*Note for future AI: Read this file during migration and ensure these logical fixes are preserved if the corresponding files were modified in Sprint 7/8.*

## Office reconciliation — 2026-10-03

Preserved through a normal Git merge of frontend `origin/main` at `fcfb5f7` into `feat/sprint-7-8-reliability` (merge `155a1aa`). The underlying AI-settings polling/copy change `a51a0d4` does not conflict with Sprint 8 Gmail stopped-row polling or correction. Docs ledger `047eb62` was merged in `3244090`. No files were copied from Downloads and no commit was rewritten. Final merged-branch verification is recorded in `docs/planning/sprint-8/execution-report.md`.

## Date: 2026-10-03 (Local time)
**Change:** Record Phase 0 verification outcomes (MV-12, MV-13) and clear legacy DB errors for budget exhaustion.
**Commit:** `docs: record MV-12/13 DB fix and COM-97 ticket` and `docs: record MV-12 and MV-13 as PASS (verified by owner)`

**Files Modified:**
- `career-companion-docs/docs/planning/migration-verification/verification-report.md`

**Logic Changed:**
1. **Verification Report**: Marked MV-12 and MV-13 as `PASS` (Confirmed manually in browser) and updated Gate status.
2. **Database State**: Manually cleared legacy `"AI daily budget exhausted..."` text from the `processingErrorDetails` column for 26 emails in the local PostgreSQL DB, resetting them to `PENDING` state.
3. **Tickets**: Created Linear ticket **COM-97** to track the architectural gap for automated recovery of emails stalled by AI daily budget limits.

---

## Date: 2026-10-03 (Local time)
**Change:** Simplify Sprint 5 and 6 planning documents into proposed local tickets and clean up AI tool instructions.
**Commit:** `docs: simplify Sprint 5 and 6 planning docs to proposed tickets`

**Files Modified:**
- `ai/github.md`
- `ai/linear.md`
- `docs/planning/sprint-5/S5-01.md` through `S5-04.md`
- `docs/planning/sprint-5/S5-05.md` (deleted)
- `docs/planning/sprint-6/S6-01.md` through `S6-05.md`

**Logic Changed:**
1. **Planning Docs**: Replaced the extensive historical S5 and S6 specification documents with simplified "proposed local ticket" formats containing Goal, Problem/Gap, Proposed Implementation, and Acceptance Criteria. S5-05 was removed entirely.
2. **AI Tool Docs**: Minor updates to `ai/github.md` and `ai/linear.md`.

---

## Date: 2026-10-03 (Local time)
**Change:** Patch smoke test to correctly navigate UI tabs and improve timeout diagnostics.
**Commit:** `test: patch smoke test to click Irrelevant tab and capture screenshot on timeout` (in `career-companion-frontend`)

**Files Modified:**
- `career-companion-frontend/scripts/smoke-stabilization.mjs`
- `career-companion-frontend/.gitignore`

**Logic Changed:**
1. **smoke-stabilization.mjs**: Added a `.catch()` block to `hasText` to automatically capture `smoke-timeout.png` and dump page text to the console upon a `TimeoutError`. Added clicks on the `Irrelevant` tab before searching for 'Audit email 0' because the UI defaults to the Job Related tab where the test emails don't appear.
2. **.gitignore**: Ignored `smoke-timeout.png`.

---

