# MCP-10 — MCP corrections after real-client verification

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 6 closeout — order 7 of 8 (after MV-14 and MCP-08 + MCP-09 part B) |
| Repository | backend (`src/mcp`, small), frontend (Automation page and dashboard panel); docs for records |
| Size / priority | M (8 items across backend logging, protocol tests, frontend focus and a manual screen-reader pass) / low risk; honest wording, accurate logs and keyboard use before MCP acceptance |
| Depends on | Phase 0 gate including [MV-14](../migration-verification/README.md#mv-14--verify-mcp-with-real-clients); MCP-08 + MCP-09 part B steps 3–5 (closeout order 6); OD-11 (item 8 only, and only if needed) |
| Blocks | [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) (records MCP acceptance) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) MCPF-09, MCPB-07, MCPB-08, MCPB-11 (verifier-failure test only), MCPB-05, MCPF-10 (focus and button names), MCPF-11, MCPF-12 (revoke-uncertain test only), MCPF-01 / MCPB-04 (conditional, OD-11); [roadmap §4](../README.md#4-mcp-where-verification-and-productionization-go) and [OD-11](../README.md#6-owner-decisions); [closeout plan](../sprint-6/closeout/README.md) order 7; [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) banner (Automation page focus belongs here) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Fix the small MCP gaps found by the 2026-10-02 audit and by the real-client run, so that MCP acceptance rests on an honest privacy statement, a server that offers only what it does, logs that tell an outage from an auth failure, a tested 2025-era path, and an Automation page and review panel that keep keyboard and screen-reader users oriented. Decide the `Origin: null` case (OD-11) only if the real-client run shows it.

## 2. Why it exists

- The audit found the MCP code sound and matching [ADR-0002](../../architecture/decisions/ADR-0002-automation-submissions-via-mcp.md). It also found a few low-severity gaps that matter once the owner uses MCP daily. None is a security hole; several affect honesty or diagnosis.
- The Automation page tells users that "only" six fields are sent. Twelve are sent. A privacy sentence that starts with "only" must be complete (MCPF-09).
- The server claims `tools.listChanged: true` for a tool list that never changes. That lets a modern client hold an idle, authenticated stream that survives token revocation (MCPB-07).
- A database outage during token checks is logged as `unauthorized`, and a request the client abandons is not logged at all. ADR-0002 decision 13 promises one line per call (MCPB-08).
- Antigravity's protocol version is unknown, and nothing tests 2025-06-18 or 2025-03-26 (MCPB-05).
- [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) hands the Automation page focus problems to this ticket (MCPF-10). The review panel hides when an unrelated query fails (MCPF-11).
- `Origin: null` cannot be fixed by configuration. The roadmap ([OD-11](../README.md#6-owner-decisions)) decides it only if [MV-14](../migration-verification/README.md#mv-14--verify-mcp-with-real-clients) or part B shows it (MCPF-01, MCPB-04).

## 3. Current behavior

Facts, checked against the code on 2026-10-02. Backend paths are under `career-companion-backend-main/`, frontend paths under `career-companion-frontend-main/`. The SDK is `@modelcontextprotocol/server` 2.2.0, `/node` 2.1.0, dev `/client` 2.2.0; the backend is CommonJS, so the `.cjs` builds are the ones that run.

1. **Privacy sentence (MCPF-09).** `src/routes/automation.tsx:47-51` (`AutomationPage`) says "Only the company, job title, platform, job link, location and the confirmation text are sent." The advertised tool schema has 12 properties (`src/mcp/server.ts:35-65`, `ADVERTISED_INPUT_SCHEMA`): also `sourceRecordRef` and `submittedAt` (required) and `portalJobId`, `destinationHost`, `discoverySource`, `workMode`. The backend stores them and the pending-review contract returns them (`src/contracts/submission.ts:13-31`, `PendingSubmissionSchema`). The "answers, resume and passwords stay on your computer" part is accurate. No test pins the sentence. The same sentence in the staged MCP-08 README text is handled by the [MCP-08 banner](MCP-08-automation-changes.md), not here.
2. **List-changed capability (MCPB-07).** `createMcpServer` builds `new McpServer({ name: 'career-companion', version: '1.0.0' })` with no options (`src/mcp/server.ts:102-103`). `registerTool` then sets `tools.listChanged` to `true` because of `?? true` (SDK `dist/mcp-TfXCpFsH.cjs:1724`, `setToolRequestHandlers`). `createMcpHandler` answers a 2026-07-28 `subscriptions/listen` request with an SSE stream (SDK `dist/index.cjs:1370-1374`), with a 15 s keepalive (`mcp-TfXCpFsH.cjs:64`) and a cap of 1024 streams per process (`:187`), not per user. The token is checked once per request (`src/mcp/router.ts:114-127`), so revoking it does not close an open stream. The app never emits list changes. No test covers `subscriptions/listen`.
3. **Logging (MCPB-08).** The log line is written only on `res.on('finish')` (`src/mcp/router.ts:86-91`). When the client disconnects first, `finish` does not fire, so no line is written. The bearer `catch` sets `outcome = 'unauthorized'` for every error (`router.ts:121-126`), but `verifyIntegrationToken` lets database errors through (`src/services/integrationTokens.ts:127-136`), and the SDK turns any non-`OAuthError` into HTTP 500 `server_error` (`mcp-TfXCpFsH.cjs:543-547`, `bearerAuthChallengeResponse`). The line still carries status 500 and `errorCategory` (`router.ts:124`). The runbook list in `STABILIZATION.md:85` omits `not_found` (`router.ts:149`), `bad_request` (`:162`) and `error` (`:164`), and does not say that SDK-level rejections (406, 415, unsupported protocol version) are logged with a status but no outcome. Tests cover only the 401 `unauthorized` path (`src/tests/mcp-endpoint.test.ts:313-333`).
4. **2025-era versions (MCPB-05).** The raw `initialize` in tests uses only `2025-11-25` (`mcp-endpoint.test.ts:32-37`); the SDK client runs in `legacy` (2025-11-25) and `auto` (2026-07-28) modes (`:40-47`, `:122-139`). The SDK supports `2025-06-18` and `2025-03-26` (`node_modules/@modelcontextprotocol/core/dist/auth-D1CeV_Jr.cjs:34-40`, `SUPPORTED_PROTOCOL_VERSIONS`). A missing `MCP-Protocol-Version` header defaults to `2025-03-26` (SDK `dist/index.cjs:771`); only an unlisted header is refused (`:895-901`). The legacy fallback answers over SSE (`dist/index.cjs:1060-1064` passes no `enableJsonResponse`; default `false` at `:384`).
5. **Focus (MCPF-10).** `CreateTokenForm` replaces the whole form, including the focused "Create token" button, with the "Token created" section (`src/components/automation/CreateTokenForm.tsx:69-104`); nothing moves focus there. "Done, I saved it" (`:95`) unmounts it with no focus return. In `TokenRow` (`src/components/automation/TokenList.tsx:35-91`) the Revoke button swaps to "Revoke now"/"Cancel" (`:72-86`) with no focus handling, and those two buttons have no row context (only `:83` has `Revoke ${token.name}`). In `SubmissionCard` (`src/components/automation/PendingSubmissionsSection.tsx:79-93`), "Link", "Create application" and "Ignore" repeat on every card. `StatusEditor` (`src/components/StatusEditor.tsx:55-70`) is the app's existing focus pattern. Its live region (`:161`) is mounted with the text already in it; S6-R08 item 1 fixes that, so copy its focus handling, not today's live region.
6. **Dashboard gating (MCPF-11).** `DashboardPage` renders `PendingSubmissionsSection` only inside `{applications && (…)}` (`src/routes/index.tsx:318-324`). If the applications fetch fails, the user sees only "Could not load applications" (`:309`); pending submissions and their own load error are hidden, although Create and Ignore do not need the list. A successful resolve only resets the offset and invalidates caches (`PendingSubmissionsSection.tsx:117-121`, `onSuccess`); there is no notice, although the response carries `applicationId` (`src/contracts/submission.ts:47-51`). When the last card is resolved, the section returns `null` (`:133`).
7. **Revoke-uncertain test (MCPF-12).** `TokenRow` shows "We could not confirm the revoke…" for an uncertain outcome (`TokenList.tsx:62-68`). `src/tests/automation-page.test.tsx:189-208` only makes revoke succeed or cancels it.
8. **`Origin: null` (MCPF-01, MCPB-04).** `router.ts:106-110` rejects through the SDK's `validateOriginHeader`, which fails on `new URL('null')`, on `file://` origins (empty hostname) and before any allowlist check. `loggableHostname` (`router.ts:52-53`) logs all of these, and also hostnames with characters such as `_`, as `unparseable`. No `MCP_ALLOWED_ORIGINS` value can admit them (`src/mcp/config.ts:6-7`, `:24`; `_` also fails the `HOSTNAME` check at `:11`). Tests pin this (`mcp-endpoint.test.ts:273-280`, `:335-347`).

Suspected risks, not demonstrated:
- Whether Claude Code or Antigravity open listen streams is unverified. Today's log may not show it: an open stream that the client drops fires `close` without `finish`, so no line is written (inferred, not run).
- An open listen stream would likely hold `server.close` during shutdown (`src/index.ts:119-128`, `shutdown`). Inferred, not run.
- That an abandoned request leaves no log line was shown by the verifier in an in-memory Node 24.20 check, not against this app.
- Whether Antigravity sends any Origin is unknown until MV-14.

## 4. Scope

1. **Privacy sentence.** Rewrite the sentence at `automation.tsx:47-51` so it lists everything sent. Keep "Your answers, resume and passwords stay on your computer" and the sentence about the dashboard. Recommended text: "Only these details of each confirmed application are sent: company, job title, platform, job link, job ID on the portal, the application site's host name, where the job was found, location, work mode, the confirmation text, the time it was submitted and a reference to its entry in your daily file." Keep it consistent with the corrected MCP-08 README sentence if MCP-08 is already applied.
2. **No list-changed claim.** Construct the server with `new McpServer({ name: 'career-companion', version: '1.0.0' }, { capabilities: { tools: { listChanged: false } } })`. The SDK constructor then sets up the tool handlers with `listChanged: false` (`mcp-TfXCpFsH.cjs:1657-1661`); `?? true` keeps `false`; `honoredSubset` (`:162-170`) returns an empty set, so a listen request is acknowledged and closed at once (`:289-292`). The SDK client also skips its automatic listen when the server does not advertise it (`client/dist/index.cjs:3412-3419`). Tool listing and calls must not change.
3. **Log accuracy.**
   1. In the bearer `catch` (`router.ts:121-126`), keep `outcome = 'unauthorized'` for an `OAuthError`. For any other error set a distinct outcome. Recommendation: `auth_unavailable`, so it is not confused with the tool-level `unavailable`. Keep `errorCategory` (error class name only) and the SDK's 500 response.
   2. Log a request the client abandons: exactly one line, with `aborted: true`. Guard with a flag so `finish` and `close` never both write. The outcome is whatever was known when the client left.
   3. Complete the outcome list in `STABILIZATION.md:85`: add `not_found`, `bad_request`, `error`, the new verifier outcome, the `aborted` flag, and one sentence that SDK-level rejections (406, 415, unsupported protocol version) carry a status and possibly `errorCategory` but no outcome. Update the matching comments at `router.ts:9` and `src/mcp/callLog.ts:3-7` and `:14` (`McpCallRecord.outcome`).
   4. Add the verifier-failure test (MCPB-11; the other MCPB-11 cases are dropped).
4. **2025-era test.** Add one supertest case, run for two versions: `initialize` then `tools/call` with protocol `2025-06-18` (with the `MCP-Protocol-Version` header on the call) and `2025-03-26` (no header). Both must record the submission. Do not test 2024 versions: those clients normally use the old HTTP+SSE transport (GET), which this POST-only endpoint refuses by design.
5. **Focus and names on the Automation page and review panel.**
   1. After a token is created, move focus to the created-token section's first line ("Copy your new token … now", a `<p>` at `CreateTokenForm.tsx:72`; make it focusable with `tabIndex={-1}`). Do not focus the token input: its `onFocus` selects the secret and a screen reader would read it aloud.
   2. After "Done, I saved it", move focus to the "Create a token" heading (`:108`, `tabIndex={-1}`).
   3. Revoke: entering the confirm step moves focus to "Cancel" (the safe choice); the warning (`TokenList.tsx:74`) is linked to both buttons with `aria-describedby`. Cancel returns focus to the "Revoke <name>" button. After a revoke settles (success, failure or uncertain), focus moves to the row's token name (`:51`, `tabIndex={-1}`), because the buttons may disappear once the list refreshes.
   4. After a successful resolve on the dashboard, move focus to the success note (item 6).
   5. Give repeated buttons distinct accessible names that start with their visible text (WCAG 2.5.3), for example "Revoke now: Laptop", "Cancel: Laptop", "Link: Acme Inc. — Backend Engineer", "Create application: …", "Ignore: …". Visible text does not change. Do not move focus on first page load.
6. **Review panel independence and success note.**
   1. Render `PendingSubmissionsSection` outside the `{applications && …}` gate (`index.tsx:318-324`). Pass `applications` as possibly `undefined`. While it is missing (loading or failed), disable the select and Link, with a short visible reason; Create and Ignore stay enabled. The email panels stay as they are.
   2. On a successful resolve, show a short note in a persistent `role="status"` region (mounted before the text changes, as S6-R08 requires): for `LINKED` and `CREATED`, a router `Link` to `/applications/$id` using the returned `applicationId`; for `IGNORED`, no link. The note is plain text and does not echo submission fields. The region stays after the last card is resolved; the heading and list still disappear as today.
7. **Revoke-uncertain test.** Add the test for `TokenList.tsx:62-68`: `revokeIntegrationToken` rejects with an uncertain `ApiError` (for example `new ApiError('The request timed out.', 'timeout')`); the uncertain message shows, the list is refetched, and revoke is called exactly once (never resent).
8. **Conditional: `Origin: null` and other part B fixes.**
   1. If MV-14 and part B show no `origin_not_allowed` with `rejectedOriginHost: "unparseable"`, record "OD-11 not needed" with a link to that evidence, and skip 8.2.
   2. If they do, stop and get the owner's OD-11 decision first. Recommendation (OD-11): an opt-in setting, off by default (suggested name `MCP_ACCEPT_NULL_ORIGIN=true|false`; the final name goes in the amendment), that accepts only the literal `Origin: null`, only after the Host check passes and only with a valid Bearer token. `file://` origins, empty hostnames and other unparseable values stay refused. Record it as an ADR-0002 amendment to decision 11 before the code. If `MCP_ACCEPT_NULL_ORIGIN` (or its final name) is added, add it to [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md)'s pinned test-env list (§4 item 3) with its default, `false`. If the evidence cannot tell `null` from the other unparseable cases, first add a bounded `rejectedOriginKind` (`null`, `empty_host`, `invalid_host`) to the log line, never the raw value, and re-run the client. If the case is not the literal `null`, OD-11 does not cover it: record it and ask the owner; do not implement.
   3. Fold in any other server-side defect found in part B, limited to fixes in `src/mcp` or the token and intake services, each with its evidence and a test. Examples, hypothetical until seen: a client that rejects an advertised schema keyword (`format`, `pattern`, `additionalProperties`) — drop it from `ADVERTISED_INPUT_SCHEMA` only, keep the strict handler validation, and update the schema tests that pin that keyword (`mcp-endpoint.test.ts:105-109`, `:199-203`); a version-specific failure — log the validated `MCP-Protocol-Version` value. A fix may not change the accepted fields, matching rules, auth model or add client-specific code. Anything larger goes to the roadmap "Not now" table or a new ticket (next free ID MCP-11) after owner review.

## 5. Out of scope

- Per-token request limits (MCPB-06) and deployed `/mcp` edge routing (MCPF-05): [release track](../README.md#7-release-track-not-scheduled).
- Company-name normalization in matching (MCPF-13) and showing the job link after resolve (MCPF-14): [Not now](../README.md#8-not-now).
- OAuth 2.1, new tools, resources, read tools; application archive or undo (OD-10).
- The other MCPB-11 tests (415, Set-Cookie, legacy batch, abort-then-replay, submittedAt-only replay, endpoint-level `linked`/`needs_review`) and the other MCPF-12 tests (list load error, clipboard failure, generic resolve branch, panel load error).
- The token-name error's `aria-describedby` link (already announced by `role="alert"`), and a refresh of the applications cache after an uncertain resolve (MCPF-11 part 3).
- The same gating and naming in the email review panels (`index.tsx:135-285`) and the AI page (AI-20).
- The staged MCP-08 README sentence (MCP-08 banner) and the MCP worst-case wording (S6-C02).

## 6. Likely files and components

- Backend: `src/mcp/server.ts` (`createMcpServer`), `src/mcp/router.ts` (log middleware `:86-91`, Host/Origin `:99-112`, bearer `:114-127`), `src/mcp/callLog.ts` (`McpCallRecord`, `writeCallLog`), `src/tests/mcp-endpoint.test.ts`, `STABILIZATION.md`. Item 8 only: `src/mcp/config.ts`, `src/tests/mcp-config.test.ts`, `.env.example:78`, `README.md:191`, `STABILIZATION.md:127`.
- Frontend: `src/routes/automation.tsx`, `src/components/automation/CreateTokenForm.tsx`, `src/components/automation/TokenList.tsx`, `src/components/automation/PendingSubmissionsSection.tsx`, `src/routes/index.tsx`, `src/tests/automation-page.test.tsx`, `src/tests/pending-submissions.test.tsx`, possibly `scripts/smoke-stabilization.mjs` (`:956-973`).
- Docs: `docs/product/user-flows.md` (UF-13); item 8 only: ADR-0002 decision 11 and [plan MCP-04](README.md#mcp-04--mcp-endpoint-adapter).

## 7. Implementation notes

- **Data model and migration:** none. No Prisma change, no migration; the guarded migrate is only needed to prepare the test database.
- **Shared contracts:** none. `npm run sync-contracts` must report 0 changed. The only wire change is the capability bit: `initialize` (2025 era) and `server/discover` (2026-07-28) now report `tools.listChanged: false`.
- **Logging sketch (item 3):** in the first router middleware, keep one `logged` flag; `res.on('finish')` writes the line; `res.on('close')` writes it only if `!res.writableFinished`, adding `aborted: true`. Log `status` only if headers were sent, else `null` (`writeCallLog` then takes `number | null`). A committed intake whose client left may show no outcome on its line; the `external_submissions` row stays the source of truth, and a replay returns `already_recorded`. Making the line wait for the tool result is out of scope.
- **Focus sketch (item 5):** refs plus an effect keyed on the state change (`created` set or cleared, `confirming` toggled, mutation settled), as in `StatusEditor`. `Button` already forwards `ref` (`src/components/ui/button.tsx:9`, used at `StatusEditor.tsx:155`). `TokenRow` is keyed by token ID (`TokenList.tsx:113`), so refs survive the refetch.
- **Panel sketch (item 6):** the component always renders a wrapper holding the status region; the heading, cards and pagination render only when there are items, as today. The existing error `notice` (`role="alert"`, `:144-148`) is unchanged.
- **Background jobs:** none. Workers, matcher and `deriveStatus` are untouched.
- **Failure and recovery:** a verifier outage still returns 500 with no detail; the client retries later; the line now says `auth_unavailable`. A listen request now gets an ack and an immediate close; a client that wants list changes simply gets none, which is the truth.
- **Compatibility:** tool name, schema and results are unchanged. The smoke clicks buttons by visible text (`smoke-stabilization.mjs:463-470`), so new accessible names do not break it. The smoke step at `:971` waits for the heading to disappear after the last resolve; the success note must not contain "Automation submissions to review". The same scenario focuses the select as soon as the heading shows (`:958`, `:964-967`). After item 6 the panel can show before the application list loads, while the select is still disabled, so the smoke likely needs to wait for the select to be enabled. Existing tests that query `Revoke now`, `Cancel`, `Link`, `Create application` and `Ignore` by exact name must be updated.
- **Rollback:** each item is a separate commit and can be reverted alone. No data is written differently. For item 8, setting the flag off restores today's behavior.

## 8. Dependencies

- The Phase 0 gate has passed, and MV-14's real-client results are recorded (Origin, and protocol version if known).
- MCP-08 is applied and MCP-09 part B steps 3–5 are recorded (closeout order 6), or a blocker is recorded with an owner decision.
- OD-11 is decided by the owner only if item 8.2 applies.
- Items 1–7 need no real-client result. Keep the roadmap order unless a part B failure needs item 2 or 4 earlier as a diagnostic.

## 9. Security and privacy

- Item 1 makes a user-facing privacy statement complete. No data flow changes.
- Item 2 removes an idle, authenticated stream that outlives revocation and a per-process resource that any token holder could hold open.
- Log lines keep carrying no payload values, no token material and no error messages: only outcome names, the `aborted` flag and error class names.
- Focus never lands on the token input; the plaintext stays only in component state. The existing "SENTINEL" cache and storage checks must still pass.
- The success note links by the parsed `applicationId` from our own API and renders no untrusted submission text.
- Item 8 weakens one check, so it is opt-in, off by default, logged, limited to the literal `null`, and still requires an allowed Host (DNS-rebinding guard) and a valid Bearer token. Bearer-only auth means a web page cannot ride a session (ADR-0002 decision 11). It needs the owner's decision and an ADR amendment first.
- MCP manual checks run on a throwaway database and user: applications created by automation cannot be deleted (MCPF-02, OD-10).

## 10. Acceptance criteria

- [ ] The Automation page sentence names every field the tool sends and still says answers, resume and passwords stay on the computer; a component test checks it.
- [ ] `initialize` and `server/discover` advertise `tools.listChanged: false`; tool listing and calls in both eras are unchanged.
- [ ] A `subscriptions/listen` request from an authenticated 2026-07-28 client gets an empty honored filter and the stream closes well before the 15 s keepalive; this test fails on today's code.
- [ ] A token-verifier database error returns 500 and logs outcome `auth_unavailable` (or the chosen name) with `errorCategory` and no `userId`; real auth failures still log `unauthorized` with 401.
- [ ] Every `/mcp` request writes exactly one log line; a request the client abandons writes one line with `aborted: true`.
- [ ] `STABILIZATION.md` lists every outcome the router sets, the new outcome and flag, and the no-outcome SDK rejections.
- [ ] `initialize` plus `tools/call` succeed at `2025-06-18` (with header) and `2025-03-26` (no header) and record the submission.
- [ ] Focus moves as in item 5 after create, Done, entering revoke confirm, Cancel, revoke settled and resolve; component tests assert it with `toHaveFocus`.
- [ ] Repeated buttons have distinct accessible names that start with their visible text; visible text is unchanged.
- [ ] With the applications query failing, the dashboard still shows pending submissions; Link is disabled with a reason; Create and Ignore work.
- [ ] A successful resolve shows a success note in a persistent status region, with a link to `/applications/<applicationId>` for Link and Create.
- [ ] The revoke-uncertain test exists and proves the revoke is not resent.
- [ ] Item 8 outcome is recorded: "not needed" with evidence, or OD-11 decision + ADR-0002 amendment + flag with tests (off → 403; on + `null` + valid token + allowed Host → 200; on + bad token → 401; on + bad Host → 403; on + `file://` → 403; invalid value fails production startup). Any other part B fix is listed with its evidence and test, or "none".
- [ ] No contract or migration change: `sync-contracts` reports 0 changed.

## 11. Testing

Tests to add or change:
- `src/tests/mcp-endpoint.test.ts` (backend): capability and listen test with the SDK client in `auto` mode (`client.getServerCapabilities()`, `client.listen({ toolsListChanged: true })` returns `{ honoredFilter, close, closed }` per `client/dist/index.cjs:3964-4073`; race `closed` against a 2 s timer); verifier failure via `vi.spyOn(prisma.integrationToken, 'findUnique').mockRejectedValueOnce(new Error('database unavailable'))` (the pattern at `integration-tokens.test.ts:213` and `ai-idempotency.test.ts:77`); one-line-per-request and aborted-request logging (suggested: a raw `http.request` to `baseUrl` with a valid token and a `Content-Length` larger than the body, then destroy the socket; mechanics unverified); `it.each` over `2025-06-18` and `2025-03-26` with `.buffer(true)` and parsing of the SSE `data:` line.
- `src/tests/mcp-config.test.ts`: only if item 8 adds the flag.
- `src/tests/automation-page.test.tsx` (frontend): privacy sentence; focus after create and Done; revoke focus and names; revoke-uncertain; update the `Revoke now`/`Cancel` queries (`:196`, `:205`).
- `src/tests/pending-submissions.test.tsx`: applications fetch rejected (`listApplications`) → panel shown, Link disabled, Create works; success note and link; focus after resolve; update the `Link`/`Create application`/`Ignore` queries (`:110-149`).

Commands (backend, from `career-companion-backend-main`):
```bash
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs   # only to prepare the test DB; no new migration
npx vitest run src/tests/mcp-endpoint.test.ts
npx vitest run src/tests/mcp-config.test.ts                                     # item 8 only
npm test
```
Commands (frontend, from `career-companion-frontend-main`):
```bash
npm run sync-contracts            # expect 0 changed
npm run typecheck && npm run lint
npx vitest run src/tests/automation-page.test.tsx src/tests/pending-submissions.test.tsx
npm test && npm run build
```
Smoke (browser-visible behavior changes): create an empty `career_companion_<x>_smoke_test`, migrate it with `TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL=<smoke URL> node scripts/guarded-migrate.cjs` (backend), then from the frontend `SMOKE_DATABASE_URL=<smoke URL> node scripts/smoke-stabilization.mjs`. Expect PASS and residue 0.

Manual, on a throwaway database and user: keyboard-only and one screen reader (VoiceOver or NVDA, depending on the personal PC) through create, Done, revoke and resolve; then `CC_MCP_TOKEN=<token> node scripts/mcp-client-check.cjs http://localhost:3000/mcp` still lists the tool. If item 8 is implemented, repeat the MV-14 Antigravity connect, list and call with the flag on. Manual results are local evidence, not automated proof.

## 12. Documentation updates

- Backend `STABILIZATION.md:85` outcome list (item 3); comments in `router.ts` and `callLog.ts`.
- `docs/product/user-flows.md` UF-13: one line each for the success note and for the panel showing when the application list fails (Link then unavailable).
- Item 8 only: dated amendment under ADR-0002 decision 11; one line in [plan MCP-04](README.md#mcp-04--mcp-endpoint-adapter) next to "any fix is configuration"; backend `README.md:191`, `STABILIZATION.md:127` and `.env.example`.
- Results, test counts and the item 8 outcome go into the closeout execution report (`sprint-6/closeout/execution-report.md`, created if absent). S6-C02 updates the MCP plan status from it.

## 13. Definition of done

- Every acceptance criterion above is met, with evidence linked.
- Backend and frontend tests, typecheck, lint (0 errors) and build are green on the new PC, and the smoke passes. There is no CI yet; once [S7-01](../sprint-7/S7-01-ci-and-toolchain-pins.md) exists, the same checks run there.
- Behavior verified by hand on a throwaway database and user, as in section 11.
- Docs in section 12 updated; ADR-0002 amended only if OD-11 was decided.
- Focused commits (one per item, or per repo), in git.
- Evidence recorded in the closeout execution report. Fixture and synthetic results are not called live evidence.
