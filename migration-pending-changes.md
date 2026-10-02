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
