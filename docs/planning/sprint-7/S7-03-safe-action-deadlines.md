# S7-03 — Store AI action deadlines only when the date is clear

| Field | Value |
| --- | --- |
| Status | Implemented locally — remote/live acceptance pending; [evidence](execution-report.md) (2026-10-03) |
| Phase / sprint | Sprint 7 — order 3 of 6 |
| Repository | career-companion-backend (date parser at the matcher boundary, one additive migration, action responses, Discord message); career-companion-frontend (Action Center and application detail display, synced contract) |
| Size / priority | M (a migration, a shared-contract change, a UI change and lane re-runs) / removes false "Overdue" items from the main dashboard and from Discord |
| Depends on | OD-06; Phase 0 gate and Sprint 6 closeout (S6-R08 item 2 edits the same contract file); S7-01 helpful, not blocking |
| Blocks | — |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) S56-11 (S56-12 for context only); [roadmap](../README.md) §6 OD-06, §8 row "Interview/assessment agenda"; [Sprint 7 README](README.md) §2, §3, §7 |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

When the AI gives an action deadline, store it only if the date is clear. A deadline written without a year gets its year from the email's received date (OD-06). Anything else unclear is stored as null. A date-only deadline stays date-only: no invented time, no day shift, and it is not "Overdue" before that day has ended. This holds in the Action Center, on the application page and in the Discord message. The AI prompt and the extraction contract do not change, so no AI call is replayed.

## 2. Why it exists

- The matcher turns free text into a date with `new Date(text)`. In Node, "by Nov 20" becomes 20 Nov **2001**. The action then shows under "Overdue" with a made-up time, and Discord sends the same wrong date (S56-11, confirmed).
- A plain ISO date ("2026-11-20") is stored as UTC midnight. West of UTC it shows as the previous day and turns Overdue before the day has ended (S56-11 verifier, point 3).
- The roadmap keeps the agenda and a temporal extraction contract out of scope until after Sprint 8. "S7-03 removes the visible wrong dates first" ([roadmap §8](../README.md#8-not-now)).

## 3. Current behavior

Facts (checked in the code on 2026-10-02; backend = `career-companion-backend-main`, frontend = `career-companion-frontend-main`):

- **The only writer.** backend `src/services/matcher.ts:126-220` (`MatcherService.applyMatch`) creates the action at :193-211. It takes `actionDeadline` when `actionRequired` is true, else `followUpDate` (:196-198). It runs `new Date(deadlineText)` (:199) and stores any result that is not NaN (:208). No other source file writes `Action.deadline`. Every match path and the user's own resolution go through `applyMatch` (:33, :56, :98, :317).
- **No rewrite on reprocessing.** If an action already exists for the application and email, `applyMatch` returns it unchanged (:194-195).
- **The received date is at hand.** `applyMatch` loads the full email row inside its transaction (:135). `Email.receivedAt` is `DateTime?` (`prisma/schema.prisma:93`). Sync sets it from Gmail `internalDate` (`src/services/gmailSync.ts:162`, :173), or else from the `Date` header (`extractHeaders`, :67-73). It can be null.
- **Storage.** `Action.deadline` is `DateTime?` (`schema.prisma:266`); the column is `TIMESTAMP(3)` (`prisma/migrations/20260912151032_add_application_event_and_action/migration.sql:29`). No constraint or trigger checks it, and nothing records whether it is a date or a date-time. The raw AI text stays in `ai_processing_results` (`actionDeadline` :216, `followUpDate` :218, both `String?`).
- **AI input.** `JobExtractionSchema` date fields are free text with no format (`src/services/ai/contracts.ts:57` `actionDeadline`, :62 `followUpDate`). The `EXTRACTION_CONTRACT` instructions (:148-154, inside :144-157) have no date rule; the version is `extraction/v2` (:7). The model gets only the email body (`src/services/ai/pipeline.ts:94-101`), so it cannot know the year.
- **Parse results today** (Node v24.20.0, re-checked 2026-10-02): "Nov 10", "20 Nov" and "by Nov 20" all give the year 2001. "10/11/2026" gives 11 Oct (US month-first order). "Nov 20, 2026" gives midnight in the server's time zone. "2026-11-20" gives UTC midnight. "Friday" gives NaN, so null (safe).
- **API.** `src/services/action.ts:20-25` sorts by `deadline` ascending; it sends ISO strings at :50 (`getUserActions`) and :127 (`updateActionStatus`). `src/services/application.ts:349-396` (`getApplicationActions`) returns the `Date` at :392 (sent as ISO JSON). Contract: `src/contracts/application.ts:148-157` (`ApplicationActionResponseSchema`, `deadline` at :154 is `z.union([z.date(), z.string()]).nullable()`); `src/contracts/action.ts:4-17` (`ActionWithContextResponseSchema`) extends it. The frontend does not run-time check action responses: frontend `src/api/client.ts` passes no schema in `getApplicationActions` (:235-240), `getActions` (:391-400) or `updateAction` (:402-407).
- **Dashboard.** frontend `src/routes/index.tsx:53-55` (`ActionQueueSection`) splits pending actions with `isPast(new Date(deadline))`. :61 shows the Overdue badge, :67 shows `MMM d, yyyy h:mm a`, :101-120 render the Overdue, Upcoming and Pending groups.
- **Application page.** frontend `src/routes/applications.$id.tsx:122` (`ActionItem`) shows `Due MMM d, yyyy` at :151-155 in the browser's zone. It hides the time but has the same day shift.
- **Discord.** backend `src/jobs/notificationJob.ts:98-105` puts `deadline.toISOString()` in the payload (:104). `src/services/notifications/DiscordProvider.ts:30-36` shows it with `toLocaleString()` in the server's zone and locale (:33).
- **Tests.** No backend test covers deadline parsing. frontend `src/tests/action-queue.test.tsx:79-112` checks the grouping with full timestamps only.
- **Not touched here.** The eval scorer accepts year-less answers (`src/eval/ai/score.ts:70-88`, `sameDate`). The AI settings sample test returns the raw `actionDeadline` text (`src/services/ai/settings.ts:384`).

Suspected risks (not demonstrated):

- How often the model returns year-less text is unknown. No live provider call has ever been made, and every eval case with `actionDeadline` puts the year in the email body.
- The model may turn "Nov 20" into an ISO date with a guessed year (for example "2024-11-20"). Not observed; plausible.
- The model may write a date-only deadline as a UTC-midnight date-time ("2026-11-20T00:00:00Z"). The parser treats that as `DATETIME` (§4 item 1b). With option (a) it keeps today's display and day shift. Not observed; change the rule only with evidence from daily use.
- How many wrong rows exist is unknown. The original dataset may not exist (OD-13).

## 4. Scope

1. **Pure parser** `parseActionDeadline(text, receivedAt)` in a new backend file `src/utils/actionDeadline.ts`. No database, no clock, no server time zone: use only `Date.UTC` and `getUTC*`, and never pass non-ISO text to `new Date()` or `Date.parse`. It returns `{ deadline, precision }` with precision `DATE` or `DATETIME`, or null with a fixed reason code. Trim the text, collapse spaces, ignore case. Accepted forms:
   - a. ISO date `YYYY-MM-DD` → `DATE`.
   - b. ISO date-time with `Z` or a `±HH:MM` offset → `DATETIME` at that instant.
   - c. ISO date-time without an offset → `DATE` of its date part. The date is clear; the time's zone is not, so no time is kept.
   - d. English month-name dates: an optional leading "by", an optional weekday ("Friday," or "Fri"), then "Nov 20" or "20 November", an optional ordinal ("20th"), an optional dot after the month ("Nov."), and an optional four-digit year ("Nov 20, 2026"). With a year → `DATE`. Without a year → the year is inferred (item 2).
   - e. **Everything else → null.** This includes "Friday", "by Friday", "next Monday", "tomorrow", "end of week", "ASAP"; every numeric form other than ISO ("10/11", "10/11/2026", "25/11/2026", "11.10.2026"); a month-name date with a time ("Nov 20 at 5 PM"); a month or year alone; two-digit years; non-English month names; empty text.
   - Every result must be a real calendar date (no 30 Feb, no month 13). A `DATE` is stored as UTC midnight of that calendar date.
2. **Year inference (OD-06).** The anchor is the UTC calendar date of `receivedAt` minus one day. The one-day allowance covers a sender whose local date is a day behind UTC. It is how this ticket reads OD-06's "on or after the received date" without a per-user time zone; confirm it at ticket review. Use the anchor's year; if that month-day falls before the anchor, use the next year. If the result is not a real date (29 Feb in a common year) → null. If `receivedAt` is null → null.
3. **Unclear means null (OD-06).** An explicit-year result (forms a–d) whose calendar date (UTC date for form b) is before the anchor → null. A deadline that had already passed when the email arrived is not clear. This is the cheap guard against implausible dates the S56-12 verifier suggested. It is skipped when `receivedAt` is null. If a weekday is given and does not match the resulting date → null.
4. **Matcher.** Replace `matcher.ts:199` and :208 with one call: `parseActionDeadline(deadlineText, email.receivedAt)`. The action is still created when the result is null; it then lands in "Pending" as today. When the text is not empty but the result is null, log one JSON line `{ event: 'action_deadline_unclear', emailId, reason }`. Never log the text.
5. **Precision storage — decision (under OD-06 "keep date-only deadlines date-only"). Recommendation: (a).**
   - (a) **Additive column.** New Prisma enum `DeadlinePrecision { DATE DATETIME }` and nullable `Action.deadlinePrecision`. New rows get it from the parser. Existing rows stay null, meaning "legacy, unknown", and keep today's display. Cost: one guarded additive migration, one contract field, the frontend sync, three mappers.
   - (b) **UTC-midnight convention.** No schema change. A deadline at exactly 00:00:00.000 UTC means date-only, and every reader shows its UTC calendar date with no time. Weaknesses: a real time at 00:00 UTC (05:30 in India) shows as a date; the rule lives only in code and docs, and every reader must repeat it; existing rows that sit at UTC midnight change their display silently.
   - Why (a): it is explicit and cannot be misread, legacy rows stay distinguishable, and the change is small. The later temporal contract (S56-12) must decide precision anyway, and a stored field is a clean start for it. If the owner wants no migration in Sprint 7, use (b) and record that choice in the execution report and in the domain model.
   - With (b): skip item 6 and the migration. Items 7 and 8 test "deadline is exactly 00:00:00.000 UTC" in place of `deadlinePrecision === 'DATE'`, and the Discord payload gets no new field. Drop the precision-field tests and criteria in §10–§11.
6. **API and shared contract** (with option a). Add `deadlinePrecision: z.enum(['DATE', 'DATETIME']).nullable()` to `ApplicationActionResponseSchema` (`contracts/application.ts:148-157`); `ActionWithContextResponseSchema` inherits it. Send it from `action.ts:44-65`, `action.ts:121-142` and `application.ts:374-395` (add it to the `select` too). Frontend: `npm run sync-contracts`.
7. **Frontend display.** New helper `src/lib/deadline.ts`, used by `index.tsx` and `applications.$id.tsx`. It takes a `now` argument so tests can fix the time.
   - `DATE`: show the calendar date from the UTC parts as `MMM d, yyyy`, with no time, in both places. It is Overdue only when that date is before the viewer's local today, so an action due today stays in Upcoming until local midnight.
   - `DATETIME`, or null/missing precision with a deadline: exactly today's behavior (dashboard `MMM d, yyyy h:mm a`, detail `MMM d, yyyy`, Overdue by `isPast`).
8. **Discord.** Add `deadlinePrecision` to `NotificationPayload` (`NotificationProvider.ts:1-8`) and fill it at `notificationJob.ts:104`. In `DiscordProvider.ts:30-36`, a `DATE` deadline is shown with `toLocaleDateString('en-US', { timeZone: 'UTC', year: 'numeric', month: 'short', day: 'numeric' })` (for example "Nov 20, 2026"). `DATETIME` and null stay on `toLocaleString()`.
9. **Existing rows: no automatic rewrite or backfill.** Leave them and document it (§12). Reason: a rewrite changes stored data for little gain, the original-data preservation rule stays, and the owner can dismiss a wrong action in the Action Center. Reprocessing will not fix them either (§3, matcher :194-195). Optional owner check, read-only, on the owner's own database: `SELECT count(*) FROM actions WHERE status = 'PENDING' AND deadline < '2020-01-01';`. Record the count in the execution report. It finds only the year-2001 kind; a shifted day or a swapped day and month cannot be found this way.

## 5. Out of scope

- The agenda, interview and assessment dates (`interviewDate`, `interviewTime`, `assessmentDeadline`), rescheduled-invite rules (S56-12; [roadmap §8](../README.md#8-not-now); [Sprint 8 README](../sprint-8/README.md) non-goals).
- Any prompt, schema description or contract version change (no `extraction/v3`), and any AI replay or re-extraction.
- Per-user time zones, and the time zone used for `DATETIME` text in Discord.
- Rewriting existing actions; a new action-editing UI; showing the raw AI text in the Action Center.
- Eval scorer changes (`score.ts:70-88`) and the sample test display.
- More date forms (numeric, times in text, non-English, "before …"). Widen only with evidence from daily use.
- An upper bound on far-future dates. Tightening other contract date fields (S6-R08 item 2, OD-12).

## 6. Likely files and components

- backend: `src/utils/actionDeadline.ts` (new), `src/services/matcher.ts`, `prisma/schema.prisma`, `prisma/migrations/<timestamp after the newest migration at implementation time>_action_deadline_precision/migration.sql` (new; AI-19 adds one in the closeout), `src/contracts/application.ts`, `src/services/action.ts`, `src/services/application.ts`, `src/jobs/notificationJob.ts`, `src/services/notifications/NotificationProvider.ts`, `src/services/notifications/DiscordProvider.ts`; tests in §11.
- frontend: `src/contracts/application.ts` (synced), `src/lib/deadline.ts` (new), `src/routes/index.tsx`, `src/routes/applications.$id.tsx`; tests in §11.
- docs: §12.

## 7. Implementation notes

- **Migration (option a).** Two statements: `CREATE TYPE "DeadlinePrecision" AS ENUM ('DATE', 'DATETIME');` and `ALTER TABLE "actions" ADD COLUMN "deadlinePrecision" "DeadlinePrecision";`. No backfill, no default, no constraint. The folder timestamp must sort after the newest migration at implementation time (AI-19 adds one in the closeout, `<timestamp>_ai_access_paused`, after `20261002150000_automation_submissions`; S7-02 adds one, and S5-FU-01 may). Re-check it at merge time. Write the SQL by hand. Apply it to test databases only through `scripts/guarded-migrate.cjs`, and to the dev database with `npx prisma migrate deploy` (the guard refuses non-test databases). Never run `db:migrate` (interactive `prisma migrate dev`) or `db:reset` against a database with real data.
- **Migration lanes.** No new lane script: the change is one nullable column with no backfill. Re-run the three existing lanes. Each builds its "legacy" schema from every migration except its own (for example `scripts/verify-mcp-migration.cjs:18`, :174-175), so the new migration is already in the legacy set and lands there out of order (TEST-03). After the closeout the legacy sets also hold AI-19's `ai_access_paused` migration. It alters `AIAccessIssue`, which the AI lane's own migration (`20261002120000_user_provided_ai`) creates, so the AI lane needs AI-19's lane fix first; build on that version. The lanes should still pass: the Sprint 6 and MCP lanes insert actions with explicit columns (`verify-migration-preservation.cjs:98`, `verify-mcp-migration.cjs:142`), and their `SELECT * FROM actions` snapshots (:110, :157) include the new column both before and after. Unverified until they run.
- **API and contracts.** Additive field only. Merge the backend first, then the frontend sync commit (S7-01 `contract-drift` is red in between). An old frontend ignores the new field. The frontend helper treats a missing field as null, so a new frontend on an old backend shows today's behavior. If S6-R08 has already changed `contracts/application.ts`, build on its form.
- **Background jobs.** No new job or queue. The matcher runs inside the email worker as today. The notification job only passes one more field. The helper never throws, so it cannot fail an email job.
- **Failure and recovery.** An unclear deadline gives an action with no date (in "Pending") and one log line with a reason code. The raw text stays in `ai_processing_results`, so a later agenda can parse it again.
- **Sorting.** `DATE` values sit at UTC midnight, so the existing `deadline asc` order still works across both precisions.
- **Coordination.** S6-R07 (closeout, lands first) also edits `matcher.ts`: the ambiguous `updateMany` (:106-115) and the `AI_AUTO` guard in `applyMatch` (:137-141). These edits do not overlap with :196-208, but line numbers will shift; find the code by symbol.
- **Rollback.** Revert the parser, display and Discord commits. Keep the migration and the column: never delete an applied migration folder. If the column must go, add a new migration that drops it and the type.

## 8. Dependencies

- OD-06 recorded ([roadmap §6](../README.md#6-owner-decisions)); this ticket follows its recommendation. The one-day allowance in §4 item 2 and the option (a)/(b) choice in §4 item 5 are confirmed at ticket review.
- Phase 0 gate passed and Sprint 6 closeout done ([Sprint 7 README](README.md) §5). [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) item 2 lands first in the same contract file.
- [S7-01](S7-01-ci-and-toolchain-pins.md) CI is helpful, not blocking. The ticket can run in parallel with S7-02 and S7-04.

## 9. Security and privacy

- No new data leaves the machine. No prompt change, so the AI provider receives exactly what it receives today. Discord gets the same fields; only the date's formatting changes.
- The log line carries the email ID and a fixed reason code, never the deadline text or other email content.
- Ownership does not change: actions are still read through `application.userId` (`action.ts:11-17` in `getUserActions`, :74-81 in `updateActionStatus`, `application.ts:355-362` in `getApplicationActions`).
- The migration is additive and leaves every existing row unchanged.

## 10. Acceptance criteria

- [ ] `parseActionDeadline` gives the expected result for every case in the §11 table, with the same results under `TZ=America/Los_Angeles` and `TZ=Asia/Kolkata`.
- [ ] Year-less text such as "by Nov 20" is never stored as a date more than one day before the email's received date (UTC); "by Friday", "10/11" and other unclear text is stored as null.
- [ ] `matcher.ts` no longer calls `new Date()` on AI text; a matcher test shows an email received 2026-11-12 with "Nov 20" gives an action with deadline `2026-11-20T00:00:00.000Z` and precision `DATE`.
- [ ] An unclear deadline still creates the action with a null deadline and logs `action_deadline_unclear` without the text.
- [ ] (Option a) The migration applies through `guarded-migrate.cjs`, existing actions keep their values with a null precision, and the three migration lanes pass.
- [ ] (Option a) `GET /api/actions`, `PATCH /api/actions/:id` and `GET /api/applications/:id/actions` return `deadlinePrecision`; backend and frontend contracts are identical after `npm run sync-contracts`.
- [ ] Dashboard and application page show a `DATE` deadline with no time and the right day under `TZ=America/Los_Angeles`; a `DATE` deadline due today is in Upcoming, not Overdue.
- [ ] `DATETIME` and legacy (null precision) deadlines show exactly as before.
- [ ] The Discord message shows a `DATE` deadline as "Nov 20, 2026" with no time, under any server time zone.
- [ ] The docs in §12 state the deadline rule and the legacy-row limit.

## 11. Testing

Backend tests to add or change:

- `src/tests/action-deadline.test.ts` (new; no database queries, but `.env.test` is still needed). `it.each` table with `receivedAt = 2026-11-12T10:00:00Z` unless noted:
  - "2026-11-20" → `DATE` 2026-11-20; "2026-02-30" and "2026-13-01" → null.
  - "2026-11-20T17:00:00Z" → `DATETIME` same instant; "2026-11-20T17:00:00+05:30" → `DATETIME` 2026-11-20T11:30:00Z; "2026-11-20T17:00:00" → `DATE` 2026-11-20.
  - "Nov 20", "November 20", "20 November", "20th Nov", "Nov. 20th", "by Nov 20", "Friday, November 20" → `DATE` 2026-11-20; "Thursday, November 20" → null.
  - "Nov 20, 2026" → `DATE` 2026-11-20; "Nov 20, 2024" and "2001-11-10" → null (before the received date); "Nov 20, 2024" with `receivedAt` null → `DATE` 2024-11-20 (guard skipped, item 3).
  - Year rules: "Nov 10" → 2027-11-10; "Nov 11" → 2026-11-11 (one-day allowance); "Jan 5" with received 2026-12-20 → 2027-01-05; "Feb 29" with received 2026-03-01 → null; "Feb 29" with received 2027-12-01 → 2028-02-29; "Nov 20" with `receivedAt` null → null.
  - Null: "by Friday", "Friday", "next Monday", "tomorrow", "ASAP", "10/11", "10/11/2026", "25/11/2026", "11.10.2026", "November", "2026", "Nov 20 at 5 PM", "Nov 32", "20 de noviembre", "", "   ", null.
- `src/tests/matcher.test.ts`: action deadline and precision for a year-less `actionDeadline`; null for "by Friday"; the `followUpDate` path; a second match of the same email does not change the stored deadline.
- `src/tests/action.test.ts` and `src/tests/application.test.ts`: responses include `deadlinePrecision`.
- `src/tests/notificationJob.test.ts`: the payload passed to the mocked `send` carries `deadlinePrecision`.
- `src/tests/discordProvider.test.ts`: `DATE` shows "Nov 20, 2026" and no time; `DATETIME` keeps today's output.

Frontend tests to add or change:

- `src/tests/deadline.test.ts` (new): `DATE` label and overdue rule with a fixed `now` (due yesterday, today, tomorrow); `DATETIME` and null unchanged; a missing field treated as null.
- `src/tests/action-queue.test.tsx`: add `deadlinePrecision` to `makeAction` (:38-53); a `DATE` deadline due today shows in Upcoming with no time; a `DATE` deadline due yesterday shows as Overdue.
- `src/tests/applications.test.tsx`: add the field to `makeAction` (:34-46); a `DATE` deadline shows the right day. Fix any other fixture that typecheck flags (for example `stabilization-ui.test.tsx:41`, :63).

Commands, backend:

```bash
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npx vitest run src/tests/action-deadline.test.ts
TZ=America/Los_Angeles npx vitest run src/tests/action-deadline.test.ts src/tests/discordProvider.test.ts
TZ=Asia/Kolkata npx vitest run src/tests/action-deadline.test.ts src/tests/discordProvider.test.ts
npx vitest run src/tests/matcher.test.ts src/tests/action.test.ts src/tests/application.test.ts src/tests/notificationJob.test.ts
npm test
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-mcp-migration.cjs
```

Each lane needs two new, empty `career_companion_*test` databases. Frontend:

```bash
npm run sync-contracts        # expect only contracts/application.ts to change
npm run typecheck && npm run lint && npm test && npm run build
TZ=America/Los_Angeles npx vitest run src/tests/deadline.test.ts src/tests/action-queue.test.tsx src/tests/applications.test.tsx
```

Smoke: run the existing smoke once as a regression check. Migrate an empty smoke database from the backend (`TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs`), then run `SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs` from the frontend. Its actions have no deadline (frontend `scripts/smoke-stabilization.mjs:378`, and the extraction at :518-521), so it does not prove the new display; the component tests do. If a real action with a date appears in daily use, note in the execution report how it looked; otherwise write "not observed with real mail".

## 12. Documentation updates

- [domain-model.md](../../domain/domain-model.md) §4.7 Action (:216): a short "Implemented deadline rule (S7-03)" note: the code field is `deadline` (the table there calls it `dueAt`) plus `deadlinePrecision`; the accepted forms; year inference from `receivedAt`; null when unclear; a date-only deadline is Overdue only after its local day ends; legacy rows have null precision.
- [mvp-architecture.md](../../architecture/mvp-architecture.md) "Remaining limits" (:66): action deadlines are now stored only when clear; event and interview dates are still free text; actions created before S7-03 keep their stored deadline, and a wrong one is dismissed by hand.
- [email-ai-pipeline.md](../../architecture/email-ai-pipeline.md) "Next Steps" fields (:32): `actionDeadline` and `followUpDate` stay free text in the AI result; the matcher stores a date only by the S7-03 rule.
- [Sprint 7 README](README.md) §7: tick the S7-03 exit criterion with evidence links.
- Sprint 7 `execution-report.md` (new, dated): the option (a)/(b) choice, test counts, lane results, the optional legacy-row count, and whether a real deadline was observed.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend typecheck, lint, tests and build are green (in CI once S7-01 exists); the guarded migration and the three lanes pass.
- [ ] Behavior verified: the parser table under two server time zones, and the display tests under a zone west of UTC.
- [ ] Docs updated per §12.
- [ ] Focused commits: backend parser and matcher; migration and contract; Discord; frontend sync and display; docs.
- [ ] Evidence recorded in the Sprint 7 `execution-report.md`. Fixture results are not called live evidence.
