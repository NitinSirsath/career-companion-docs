# MCP feature — execution report

Date: 2026-10-02. Implemented in the local repositories, starting from the verified baseline `_baselines/mcp-feature-2026-10-02-after-byo-ai/` ([baseline.md](baseline.md)). No git, no new baseline. Synthetic data only; no live applications, Gmail or AI provider.

## Status

| Ticket | Status |
| --- | --- |
| MCP-00 spike | Done earlier (2026-10-02) |
| MCP-01 data model and migration | **Done** |
| MCP-02 integration tokens | **Done** |
| MCP-03 intake service | **Done** |
| MCP-04 MCP endpoint adapter | **Done** |
| MCP-05 review API | **Done** |
| MCP-06 contracts and timeline | **Done** |
| MCP-07 token page and review panel | **Done** |
| MCP-08 automation repo instructions | **Staged, not applied**: the repository is not on this laptop. Ready-to-apply text: [MCP-08-automation-changes.md](MCP-08-automation-changes.md) |
| MCP-09 part A (automated, this laptop) | **Done**: smoke passes twice |
| MCP-09 part B (Antigravity) | **Owner's step** on their own machine (below) |

## What was built

**Backend** (`career-companion-backend-main`)
- Migration `20261002150000_automation_submissions` (additive): `integration_tokens`, `external_submissions`, enums `ExternalSubmissionSource`, `ExternalSubmissionMatchState`, `SubmissionResolvedBy`, nullable unique `application_events.externalSubmissionId`; CHECKs (token hash format, scope, name length, expiry after creation, ref format, confirmation length, `resolvedBy`/`resolvedAt` set exactly when settled); triggers `submission_ownership`, `event_submission_ownership`, `integration_token_ownership`. Lane script `scripts/verify-mcp-migration.cjs`.
- `services/integrationTokens.ts` and `/api/integration-tokens` (create, list, revoke).
- `services/externalSubmission.ts`: strict schema, URL canonicalization, truncation, company/title keys and the ADR §7 matrix, one transaction per call under `pg_advisory_xact_lock(0x4d435001, hashtext(userId))`, daily cap, create/link with the `AUTOMATION_SUBMITTED` event; user review with the same lock.
- `src/mcp/` and `POST /mcp`, mounted before every global middleware; `/api/submissions/pending` and `/:id/resolve`.
- `submittedVia` on every application response and `sourceSubmission` on every event and `recentEvent`.
- Config: `MCP_ALLOWED_HOSTS`, `MCP_ALLOWED_ORIGINS`, `MCP_DAILY_SUBMISSION_LIMIT` in `.env.example` and `validateProductionConfig`.
- Dependencies pinned: `@modelcontextprotocol/server` 2.2.0, `@modelcontextprotocol/node` 2.1.0; dev `@modelcontextprotocol/client` 2.2.0. Not `@modelcontextprotocol/express`.

**Frontend** (`career-companion-frontend-main`)
- `/automation` page (token create, show once, copy, list, expiry warning, revoke) and nav links in `Sidebar` and `MobileNav`.
- Dashboard panel "Automation submissions to review" (link, create, ignore).
- "Applied · via automation" badge and "Submitted via automation" timeline and list entries.
- API client methods, synced contracts, Vite `/mcp` dev proxy.
- Smoke harness MCP scenario plus fixture `scripts/fixtures/mcp-daily-applications.{md,json}`.

**Diagnostic:** `scripts/mcp-client-check.cjs` runs the official SDK client against a running server (part B, step 7). Checked against a temporary local server: lists the tool on 2026-07-28, sends and replays the fixture, and reports a bad token as 401.

**Docs:** domain model §4.8, MVP architecture §10, user flows UF-11 to UF-13, product vision non-goal clarification, ADR-0002 status, backend README and `STABILIZATION.md`, frontend README, this report and the staged MCP-08 changes.

**BYO AI preserved:** no BYO AI source file changed. One BYO AI test fixture (`ai-credentials.test.ts`) gained `MCP_ALLOWED_HOSTS`, because production now requires it. The Gmail matcher, `deriveStatus`, AI processing and the email worker are unchanged.

## Implementation decisions

1. **SDK core, not the Express add-on.** The verified `AuthInfo` is passed to `toNodeHandler` on a small request view (method, URL, headers without `authorization`/`cookie`, `auth`), so the app-wide `req.auth` session type is never touched and no type is re-declared.
2. **Validation inside the tool handler.** The SDK's own validation error carries no code, so the SDK gets a pass-through schema and the handler validates with the strict zod schema. Every failure is `isError` with `{ code: "invalid_input", fields, retry }`; field names only, never values.
3. **Portable advertised schema.** The tool advertises a plain JSON Schema (single `type` per property, no null unions, no look-ahead patterns, no `$schema`), because some hosts pass tool schemas to model APIs that accept only a subset of JSON Schema. It is not client-specific; a test keeps it in step with the validated contract.
4. **Optional fields may be `null`.** Validation treats `null` like an omitted optional field, so a client that sends `null` is not refused. Empty optional strings are stored as null.
5. **Logging.** One `mcp_request` line per request, written when the response finishes, so 401/403/405/413 and validation failures are logged too. A rejected `Host`/`Origin` hostname is logged (never echoed) so the allowlist can be configured from it: this is how part B records what `Origin` Antigravity sends.
6. **Owner checks only on change.** Ownership triggers check a link when it is set or changed. Otherwise the SetNull foreign keys fired during application, token or user deletion could make deletion fail. Covered by lane and vitest tests.
7. **Extra trigger:** a token's owner is immutable too, otherwise the submission→token owner check could be bypassed by moving the token.
8. **Automatic `LINKED` re-checks the application under a row lock.** If it disappeared between the read and the lock, the submission goes to review.
9. **Token creation takes a per-user advisory lock** (separate namespace) so concurrent requests cannot exceed 5 active tokens.
10. **The plaintext token is not carried past verification:** `AuthInfo.token` holds the token ID.
11. **Review panel uses an explicit "Link" button** after choosing an application (the email panels submit on change), so keyboard users can move through the list without submitting.

## Validation performed

| Check | Result |
| --- | --- |
| Entry gate: baseline archives and manifests | 6/6 checksums OK; 0 changed and 0 new files before MCP-01 |
| Backend typecheck, build, `prisma validate`, `migrate status` | Pass; schema and migrations match (`migrate diff --exit-code` 0) |
| Backend lint | 0 errors; 11 warnings, all present at baseline |
| Backend tests | **621 passed** (baseline 467; 154 new tests in 7 new files) |
| MCP migration lanes (`verify-mcp-migration.cjs`) | Pass: fresh lane 15 trigger/constraint/deletion checks; upgrade lane 7 tables unchanged (users 3, applications 60, emails 600, AI results 600, events 401, actions 100, AI configurations 1), existing triggers and unique indexes unchanged, new column null on existing rows |
| Sprint 6 and BYO AI lanes, with the MCP migration present | Both pass |
| Frontend contract sync, typecheck, build | Pass; sync reports 0 files changed after the last backend contract change |
| Frontend lint | 0 errors; 23 warnings (baseline 21: the two new ones are the same fast-refresh rule every route file triggers) |
| Frontend tests | **154 passed** (baseline 118; 3 new files) |
| Smoke (`smoke-stabilization.mjs`, real API, PostgreSQL, pg-boss, Gmail and email workers; lane `career_companion_mcp_smoke_test`, left empty and migrated) | **PASS twice** after the last code change, residue 0 (`users`, `jobs`, `budgets`, `mcp`); no fixture answer or token text in the logs |

The smoke's MCP scenario: token created on the Automation page by keyboard and shown once (not kept in browser storage, URL or caches); official SDK client sends the fixture's three applied entries (created, linked, needs review), then replays them under the 2026-07-28 protocol (all `already_recorded`, same IDs); an extra answers field is refused as `invalid_input`; list and timeline show the automation entry distinctly; the review panel works at 390 px and resolves by keyboard; a fixture Gmail email then matches the created application through the real workers (with the harness's fixture AI configuration) and the AI status takes over the badge; revoking the token on the page makes the client fail with 401.

Not run: the optional Claude Code headless check (the official SDK client covers both protocol eras, and Claude Code was verified against SDK v2 in MCP-00).

## Remaining external verification (owner)

**MCP-08 — automation repository**
1. Provide a local copy of `job-application-automation`.
2. Apply [MCP-08-automation-changes.md](MCP-08-automation-changes.md), checking file names and sections against the repository.
3. Run its three walkthrough scenarios (tool absent, sync failure, replay).

**MCP-09 part B — Antigravity, on your own machine**
1. Run Career Companion locally (PostgreSQL, backend `npm run dev` on port 3000, frontend `npm run dev` on 5173). Sign in, set up AI if you want the Gmail step, and open **Automation**.
2. Create a token. Put the server in `~/.gemini/config/mcp_config.json` with `"serverUrl": "http://localhost:3000/mcp"` (the backend directly; `http://localhost:5173/mcp` also works through the Vite proxy) and the `Authorization: Bearer <token>` header. Run `chmod 600` on that file. Allow `mcp(career-companion/record_application_submission)`.
3. In the automation repository (with MCP-08 applied), set `Career Companion sync: yes` and copy `career-companion-frontend-main/scripts/fixtures/mcp-daily-applications.md` to `applied/2026-10-01/applications.md`. Create the two seed applications listed in `mcp-daily-applications.json` (`Northwind Traders` / `Data Engineer`, `Contoso` / `UX Designer`) on the Applications page.
4. Ask the agent: "sync Career Companion for 2026-10-01". Then ask it again.
5. Expect the same results as part A: Fabrikam created, Northwind Traders linked, Contoso waiting for review on the dashboard, **Skipped Labs never sent**, no duplicates after the second run, and no answers anywhere in Career Companion.
6. Record:
   - whether Antigravity lists and calls the tool, and accepts an `http://localhost` `serverUrl`;
   - what `Origin` it sends: if calls get 403, the backend log line `mcp_request` with `"outcome":"origin_not_allowed"` shows `rejectedOriginHost`; add that hostname to `MCP_ALLOWED_ORIGINS` and retry (configuration only, no code).
7. If Antigravity fails, run the official SDK client against the same server to tell server faults from client faults, from the backend folder:
   `CC_MCP_TOKEN=<token> node scripts/mcp-client-check.cjs http://localhost:3000/mcp` (lists the tool), and add `--send-fixture ../career-companion-frontend-main/scripts/fixtures/mcp-daily-applications.json` to send the fixture's applied entries. The token is read from the environment and never printed.

## Genuine blockers

None for the Career Companion side. MCP-08 and MCP-09 part B wait only on the repository copy and the owner's machine, as planned in section C.
