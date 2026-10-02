# S6-C02 — Docs match the code; record Sprint 6, BYO AI and MCP acceptance

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 6 closeout — order 8 of 8 (last) |
| Repository | career-companion-docs (main); career-companion-backend (`README.md`, `STABILIZATION.md`, `.env.example`, `src/eval/ai/README.md`, one script comment); career-companion-frontend (`README.md`) |
| Size / priority | L (about 70 doc edits across three repos, plus records) / honest docs and acceptance records before Sprint 7 |
| Depends on | Phase 0 gate (MV-16), incl. MV-03 commit hashes and MV-14; closeout orders 1–7; OD-01 and OD-02 recorded |
| Blocks | Closeout exit criteria, so Sprint 7 start |
| Source | [State audit 2026-10-02](../../state-audit-2026-10-02.md) DOCS-01..09, DOCS-11..19, DOCS-21, DOCS-23, DOCS-24, AIF-06..11, AIF-17, MCPF-07, MCPF-08, MCPF-13, MCPF-15, MCPB-02, MCPB-10, PLAT-04, PLAT-05, PLAT-06, PLAT-16, TEST-13, S56-08, S56-19..22, AIB-01, AIB-02, AIB-09, AIB-12; [roadmap §9](../../README.md#9-superseded-or-obsolete-plan-items); [closeout §6](README.md#6-exit-criteria) |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Every doc the audit found wrong says what the code does, through dated banners and one-line corrections. Then Sprint 6, BYO AI and MCP acceptance is recorded from new-PC evidence: a box is ticked only where its evidence is linked, and owner decisions OD-01 and OD-02 are written down where readers look for them.

## 2. Why it exists

- Many doc statements contradict the code (the rows in §4): a hosted Gemini key, a global AI budget, a 90-day sync scope, Antigravity "verified", CI that does not exist, Sprint 5 work called implemented.
- One runbook step breaks production start-up: `STABILIZATION.md:9` says to set `AI_DAILY_CALL_LIMIT=0`, which `src/utils/config.ts:33-34` refuses.
- Bookkeeping disagrees: all 18 Sprint 6 DoD boxes are unticked; BYO AI marks AI-00 "Done" with unmet criteria and AI-17 "Docs done"; MCP boxes rest on old-laptop runs only.
- It is last because the records cite the results of closeout orders 1–7.

## 3. Current behavior

Facts, checked against the code on 2026-10-02 (backend paths unless marked):

- **Gmail lookback:** `routes/gmail.ts:338` accepts only 1, 7, 14 or 30; default 1 (`prisma/schema.prisma:148`, `services/gmailSync.ts:84`); the scan is `newer_than:${syncLookbackDays}d` (`gmailSync.ts:194`). No scheduler (`gmailSync.ts:28`); the only caller of `requestGmailSync` is `POST /api/gmail/sync` (`routes/gmail.ts:397`). Only two sync events exist: `gmail_sync_completed` (`gmailSync.ts:255`) and `gmail_sync_failed` (`jobs/gmailSyncJob.ts:64`).
- **BYO AI:** production refuses `GEMINI_*` and `AI_DAILY_CALL_LIMIT` and needs `AI_CREDENTIAL_ENCRYPTION_KEY` (`validateProductionConfig`, `config.ts:24-34`). Per-user limit default 500 (`DEFAULT_USER_DAILY_CALL_LIMIT`, `services/ai/usage.ts:14`), reserved inside the claim (`reserveUserCall`, `services/ai/operations.ts:107`). A limit, pause or cooldown returns the email to `PENDING` with no retry used (`AIAccessError` branch, `jobs/emailProcessingJob.ts:114-141`). All three providers are `status: 'hidden'` (`contracts/aiCatalog.ts:72,127,182`) and `isOffered` admits them only outside production (`services/ai/access.ts:64-65`), so a production build offers no provider (`offeredProviders`, `services/ai/settings.ts:83`). The key is never returned or masked (`contracts/ai.ts:4-5`). Credentials use format `v1` and one key, no rotation (`services/ai/credentials.ts:13,24`).
- **`ai_call_budgets`** (`AICallBudget`, `schema.prisma:351-357`; stale comment at :350) has no runtime use. Besides its migration, only `sideEffectCounts` in `src/tests/application-status.test.ts:45` (S6-C01 removes it) and `scripts/verify-ai-migration.cjs:91,101` touch it.
- **MCP:** `/mcp` is mounted at `index.ts:37`. Intake creates an application with no review step: `decideMatch` (`services/externalSubmission.ts:175`) returns `CREATED` when no application has the same company (:180), and `recordSubmission` (:253) creates it (:273). Only `NEEDS_REVIEW` can be resolved (`resolveSubmission`, :327). `routes/application.ts` has no DELETE route. The Gmail matcher runs once per email (`finish`, `services/ai/pipeline.ts:118`) and drops a candidate whose normalized title differs (`matchEmailToApplication`, `services/matcher.ts:88-90`; `normalize` :119 only lowercases and strips non-alphanumerics).
- **Sprint 5 tooling is absent:** backend `scripts/` holds five files; no `baseline-snapshot*.mjs`, no `test-gmail-crash.cjs`; frontend `package.json` has no `test:smoke`. Unmatched-email code is labeled COM-36 (frontend `src/api/client.ts:261`, `src/tests/unmatched-emails.test.tsx:59`); no S5-01 or COM-37 label exists in source.
- **Environment:** no CI config and no `.git` in any of the three folders. The frontend reads no env (no `import.meta.env` in `src`; `.env.example` is 0 bytes) and Vite proxies `/api` and `/mcp` to 127.0.0.1:3000 (`vite.config.ts`). `index.ts:45` uses `??`, so an empty `OAUTH_STATE_COOKIE_SECRET` stays empty; once Google credentials are set, the signed-cookie writes (`routes/auth.ts:65`, `routes/gmail.ts:140`) throw and return 500. `.env.example:11` builds `DATABASE_URL` from `${POSTGRES_*}`; plain dotenv does not expand it. The app works only because a Prisma client generated while `.env` existed expands it first (a hidden dependency, PLAT-04).
- **Harness and client:** the smoke residue also counts `mcp` (frontend `scripts/smoke-stabilization.mjs:307-310`, checked at `:1033`); the client parses AI settings, token and submission responses too (frontend `src/api/client.ts:295-388`, `getAISettings` to `resolveSubmission`; `safeParse` at `:151`).

Already done during planning (verify, do not redo): the roadmap; the docs `README.md` index (MCP line fixed: MCPF-07 is closed); dated banners on `sprint-6/README.md`, `sprint-6/review-2026-10-02/README.md`, S6-R07, S6-R08, `sprint-5/README.md`, `sprint-5/follow-up-gmail-reliability.md` (:3), `byo-ai/issues.md`, `mcp-feature/README.md` and `MCP-08-automation-changes.md` (all present on 2026-10-02). Not present yet: the correction in `sprint-6/execution-report.md` (roadmap §9 assigns it to this ticket; row in §4.3).

Evidence for this ticket goes in the closeout execution report, `docs/planning/sprint-6/closeout/execution-report.md` (created by the first closeout ticket that needs it).

Risk, not a defect: line numbers below will drift as orders 1–7 edit the same files. Match on the quoted text, not the number.

## 4. Scope

Rules for every edit:

1. Re-read the target first. If an earlier closeout ticket already fixed it, skip it and note "already correct" in the closeout execution report.
2. Historical documents (dated plans, reports, reviews): add a dated blockquote or a line "Correction (YYYY-MM-DD): …" next to the wrong text. Never delete or rewrite it. Current-intent docs (architecture, runbooks, READMEs, `.env.example`, `ai/*.md`): replace the wrong sentence in place. Plans with open tickets (`byo-ai/`, `mcp-feature/`): fix paths, status lines, outcome lines and risk rows in place; keep their decision history.
3. Each correction cites the code (path:line plus symbol) or the ticket that owns the gap. YYYY-MM-DD is the day of the edit.

### 4.1 Docs repo — architecture, product, ai docs

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `docs/architecture/mvp-architecture.md` | 15 (§2) | "scanning the existing 90-day INBOX scope" | "the configured lookback (1, 7, 14 or 30 days; default 1)" | DOCS-02, PLAT-16 |
| same | end of §2 | (missing) | "Known gaps: crash fencing and false success (G1, G2) and sync telemetry (G4) → S5-FU-01 (Sprint 7); bounded Google requests (G3) → S5-FU-02 (Sprint 8)." | DOCS-02 |
| same | 58 (§8) | "uncertain provider work is held for operator reconciliation" | "is held; the user may approve one more attempt (§3). Engineering failures and legacy partial results stay with the operator (retry route comment, `routes/email.ts:142-146`)." | AIF-11 |
| same | 75 (§10) | "Antigravity compatibility is verified by the owner (MCP-09 part B)." | From the part B record: "Verified with Antigravity on <date> (<link>)" or "Not yet verified with Antigravity: <blocker>." | DOCS-18, MCPF-08 |
| same | 3 | link to GitHub `blob/main/STABILIZATION.md` | After the MV-03 push, open it. If `main` lacks the Sprint 6, BYO AI and MCP sections, link the pushed branch or add "may lag until merged". | DOCS-23 |
| `docs/architecture/high-level-architecture.md` | 8 (banner) | banner covers only jobs, retry, `occurredAt`, status | Extend it: §3.6, §6, §7, §9.7 are not current (hosted Gemini :105, models in env :115, :335, SDK retries :362, `career-companion/production/gemini` secret :562, "API does not call Gemini" :73, :478, "except /auth/*" :344). Now: user keys (ADR-0001); models in `src/contracts/aiCatalog.ts`; production refuses `GEMINI_*`/`AI_DAILY_CALL_LIMIT` and needs `AI_CREDENTIAL_ENCRYPTION_KEY`; the API verifies keys (`verifyModels` calls, `services/ai/settings.ts:182,262`); SDK retries off (`providers/gemini.ts:99`, `openai.ts:31`, `anthropic.ts:43`); auth uses `google-auth-library`, not Passport (:303); `/mcp` uses Bearer tokens and needs edge routing (§9.2 :411, :537); AI job failures per email-ai-pipeline, not :686-687; contracts are v2 (:818); no CI (:710) until S7-01. Link mvp-architecture §3, §10. | DOCS-04, AIF-09, MCPF-15, DOCS-24 |
| same | 3, 791-798 | header "Version 0.5"; table ends at 0.4 | Add a row for this banner. Note the 0.5 row was never written (likely the 2026-10-02 COM-26 appendix at :808; unverified). | DOCS-04 |
| `docs/architecture/email-ai-pipeline.md` | 56 | "before calling Gemini"; held claims "for operator reconciliation" | "before calling the user's provider"; held work follows "User-approved retry" (:76); only engineering failures and legacy partial results need the operator | DOCS-05, AIF-11 |
| same | 57 | "A global daily call budget and provider cooldown …" | "The per-user safety limit and per-user cooldown are enforced in the claim transaction (`reserveUserCall`, `services/ai/usage.ts:45-64`). `AI_USER_DAILY_CALL_LIMIT=0` pauses everyone." | DOCS-05, AIF-11 |
| same | 58 | retry "refuses held claims with 409 `AI_OPERATION_REQUIRES_REVIEW`" | approvable holds → 409 `AI_RETRY_NEEDS_APPROVAL` until approved; others → `AI_OPERATION_REQUIRES_REVIEW`; also `AI_ACCESS_UNAVAILABLE`, `AI_OPERATION_CHANGED` (`POST /:id/retry`, `routes/email.ts:148`, codes at :194-257) | AIF-11 |
| `docs/architecture/ai-capability-architecture.md` | 8 | "After Sprint 6 is committed and ADR-0001 is accepted" | "Implemented locally 2026-10-02 (BYO execution report); commits: <MV-03 hashes>" | DOCS-12, AIF-08 |
| same | 125; 164 (§3.5) | "Canonical home will be user flows after Sprint 6"; onboarding gets "Choose your AI provider" | "Canonical home: UF-10." Append to :164: "Not built. No onboarding exists for any feature (UF-02 is future work); entry is the AI Provider page and the AI notice on the dashboard and Gmail page." | DOCS-12, AIF-08 |
| same | 173-181 (§4.1) | "Today": `GeminiProvider.getInstance`, GLOBAL budget | Retitle "4.1 Before BYO AI (historical)". | DOCS-12 |
| same | 286 | "CI runs adapter tests against recorded responses only" | "The backend suite (`npm test`) runs adapter tests with mocked SDKs." | DOCS-24, TEST-13 |
| same | 328; 493 | "a fixed mask"; "read (masked, with usage)" | "never returned, masked or hinted (`contracts/ai.ts:4-5`)"; "read (no key material; with usage)" | AIF-08 |
| same | 450 | "on every manual or scheduled sync" | "on every manual sync (no scheduler until S7-05)" | DOCS-12, S56-20 |
| same | 479 (§11); 580-582 (§14) | "These are conceptual"; "Implementation work (later) … not a plan of record" | §11: "Built: `schema.prisma` `AIConfiguration`, `AIUsageDay`, `AIOperation`; `src/routes/ai.ts`." §14: "Done 2026-10-02 (AI-00..AI-14, AI-16). Open: AI-15 (Gemini first), AI-17, AI-18." | DOCS-12, AIF-08 |
| same | 653-654 (O7, O8) | O7 "Required before an implementation branch"; O8 "Confirm" | O7: repos and hashes from MV-03. O8: the owner's answer recorded in MV-02 item 4 (expected "none; nothing is deployed", unverified until recorded). | DOCS-09, AIF-17 |
| `docs/architecture/decisions/ADR-0001-user-provided-ai.md` | 5 | status has no commits | Add the BYO AI commit hashes from MV-03 (AI-17 criterion). | DOCS-17, AIF-17 |
| `docs/architecture/decisions/ADR-0002-automation-submissions-via-mcp.md` | 5 | "Antigravity is the target client and is verified in MCP-09" | Same outcome wording as mvp-architecture §10. | MCPF-08, DOCS-18 |
| same | 200 | "a prompt-injected agent can at worst create reviewable records" | "… can at worst create applications and link submissions; created and linked results cannot be undone in v1 (no delete or archive; OD-10). Revoke the token or set `MCP_DAILY_SUBMISSION_LIMIT=0` to stop it." | MCPB-02 |
| `docs/domain/domain-model.md` | 103-128 (§4.4) | fields `userId`, `aiProvider`, `status`, `classification`, `summary`, `errorMessage`, `processedAt`; :123 "A retry resets the current result to PENDING"; :128 versions deferred | Add "Implemented boundary (2026-10-02)" like §4.6 (:210) with the real fields (`AIProcessingResult`, `schema.prisma:190-234`: `emailId` unique, `provider`, `model`, `contractVersion`, `processingStatus`, `relevanceDecision`, `category`, flattened extraction fields, `errorCategory`, `errorDetails`). :123: retry never resets results or claims. :128: versions exist (`contractVersion`; `AIOperation` unique `(emailId, operation, version)`). | DOCS-13 |
| same | 383, 386, 407; 276-293 (§5); 371-372 (§9) | "Unique (userId, emailId)"; index "(userId, status)"; version tracking deferred; tree lacks AI tables; `summary`/`extractedData` | `emailId @unique`; `@@index([processingStatus])`; drop version tracking from §11; add `AIConfiguration`, `AIUsageDay`, `AIOperation` to §5; use real field names in §9 | DOCS-13 |
| `docs/product/product-vision.md` | 75, 77, 149, 217 | "(Proposed, 2026-10-02)", "Pending review in ADR-0001", "(proposed …)" | "Accepted 2026-10-02; implemented locally; providers hidden until certified (ADR-0001:5)". Add a versioning row. | DOCS-07 |
| same | 88 | "one onboarding step and one settings area" | "an AI Provider page next to Gmail Sync, with a notice on the dashboard and Gmail page. AI in first-run onboarding (UF-02) is future work; no onboarding exists yet." | DOCS-07 |
| `docs/product/user-flows.md` | UF-02 (65-90); :49; :104 | first-run path; "starts the initial synchronization" | "Implemented behavior (YYYY-MM-DD)" note: no first-run route (frontend `src/routes/__root.tsx:13-20`, `beforeLoad` only sends to `/login` or `/`); the Gmail callback sets `syncStatus: 'IDLE'` and starts no sync (`routes/gmail.ts:233-258`); no AI step. | DOCS-14 |
| same | UF-04 (:137, :154); UF-05 :173; UF-06 :208 | "detects new or changed Gmail messages"; "may contribute to a new application candidate"; "surfaces upcoming interviews" | Notes: sync starts only from Sync Now (twice-daily is S7-05); unmatched mail waits in Unmatched Emails, Gmail never creates applications; no interview or assessment view exists. | DOCS-14 |
| same | 450, 467, 487; 5, 500-506 | Discord not in MVP; versioning names only UF-11..13 | Correction: Discord exists for the owner only (`DISCORD_USER_ID`, `jobs/notificationJob.ts:52`). Add a versioning row: Sprint 6 notes, UF-10, these notes. | DOCS-14, AIF-08 |
| `MVP-READINESS-REPORT.md` | 3 (banner) | Sprint 6 only; "live Gemini/Discord" | Add: "BYO AI implemented locally; every provider is hidden, so a production build has no AI until a provider is certified (AI-15, Gemini first). MCP implemented locally; MCP-08 and part B: <outcome>." Say "live AI provider" instead of "live Gemini". | DOCS-06, AIB-01 |
| `docs/ai/provider-evaluation.md` | 27 | "(enforced by a CI test)" | "(enforced by `src/eval/ai/eval.test.ts`, run by `npm test`)" | DOCS-24, TEST-13 |
| `docs/engineering/stabilization-audit.md` | 62 | GitHub `blob/main` link | Same check as mvp-architecture :3. Dated audit: add a note only. | DOCS-23 |
| `README.md` (docs index) | 3-15 | (fixed in planning) | Verify. Remaining DOCS-01 item: one line linking `docs/product/product-vision.md`, `docs/product/user-flows.md`, `docs/domain/domain-model.md`, `docs/architecture/mvp-architecture.md` and `docs/DESIGN_SYSTEM.md`. | DOCS-01, MCPF-07 |
| `ai/antigravity.md` | 23-24 | "Supabase MCP" section | Delete it; no Supabase exists anywhere (HLA v0.4, :798, removed it). | DOCS-19 |
| `ai/antigravity.md`, `gemini.md`, `github.md`, `linear.md`, `documentation.md` | top | read as defined | Add "Skeleton — not filled (YYYY-MM-DD)" plus one line each: Antigravity is now the MCP target client (ADR-0002), not the implementer; "Gemini" also names a product provider (ADR-0001); git rules live in `engineering-workflow.md` §3–§8; Linear per OD-02; conventions and the checkbox rule live in roadmap §10 (from `ai/`, link `../docs/planning/README.md#10-planning-rules-from-now-on`). | DOCS-19, DOCS-21 |

### 4.2 Docs repo — Sprint 5 planning (`docs/planning/sprint-5/`)

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `S5-01.md` | 3, 13, 18 | "Status: implemented"; snapshot tool "added" | Banner: snapshot/compare tool and crash fixtures are absent; S5-FU-01 FU-5, only if OD-13 finds the dataset. | DOCS-03, S56-08 |
| `S5-02.md` | 3 | same | Banner: live Gmail/original-data proof never ran; moved by OD-01 to the pre-release gate. | DOCS-03 |
| `S5-03.md` | 3 | same | Banner: safe manual retry re-implemented in Sprint 6 (`routes/email.ts` retry route); fencing and crash regression → S5-FU-01 (Sprint 7); bounded Google requests → S5-FU-02 (Sprint 8). | DOCS-03, S56-08 |
| `S5-04.md`; `S5-05.md` | 3 | same | S5-04: re-implemented in Sprint 6 (sprint-6 execution report §1). S5-05: email-worker outcomes re-implemented (`emailProcessingJob.ts`); sync events beyond the two that exist → minimal events in S5-FU-01; full counters not now (roadmap §8). | DOCS-03, S56-20 |
| `planning-review.md`, `linear-conversion.md`, `architecture-review.md`, `verification-runbook.md` | 3 | "Main now has … 12 existing migrations, and implemented reliability/worker-browser changes" | Under each: "Correction: those changes were an uncommitted tree, never on main, and are absent from the current code. The code has 16 migrations; the newest is AI-19's `<timestamp>_ai_access_paused`. Unmatched-email code is labeled COM-36." (covers `planning-review.md:29`, `architecture-review.md:58` G8) | DOCS-03, DOCS-16 |
| `operations.md` | top; 7-15, 44, 48, 65, 69, 71, 73 | events, counters, `ai_call_budgets` "global daily counters", "15-second bound and no SDK retries", three scripts, `npm run test:smoke` | Banner "Target contract, not current code": of the events named here, only `gmail_sync_completed` (`gmailSync.ts:255`), `gmail_sync_failed` (`gmailSyncJob.ts:64`), `job_started`/`job_completed`/`job_failed` and `worker_registered` (`emailProcessingJob.ts:93,105,162,205`) exist; minimal sync events → S5-FU-01 §7; counters → not now (roadmap §8); `test-gmail-crash.cjs` → S5-FU-01 FU-4.1; `baseline-snapshot*` → FU-5, deferred per OD-13; there is no `test:smoke` script, run `node scripts/smoke-stabilization.mjs`; Gmail data calls pass a 15 s timeout but SDK retries stay on and OAuth refresh has no timeout (S5-FU-02); AI limits are per user (`ai_usage_days`), not "global daily counters" (:48); `ai_call_budgets` (:44) is unused. | DOCS-02, S56-08, S56-20, AIB-09, AIF-11 |
| `execution-report.md` | 7 | "historical unmatched-email code calls itself COM-37" | Correction: the code says COM-36; no S5-01/COM-37 label, so no collision. COM-37..41 stays unconfirmed. | DOCS-16, S56-22 |
| `linear-conversion.md` | 9, 13 | `client.ts:120`/`test:57` say "S5-01"; "paste … sections 1–19" | Same COM-36 line; ticket files have no numbered sections; no re-estimation; back-fill to Linear only per OD-02. | DOCS-16, DOCS-03, S56-22 |

### 4.3 Docs repo — Sprint 6 planning (`docs/planning/sprint-6/`)

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `README.md` | 27 (§B); 44 (§C) | "Preserve the Gmail fencing/recovery/transport work …"; "Sprint 5 reliability … present" | One line under each: absent from the current code; S5-FU-01 / S5-FU-02. | S56-20 |
| same | 77 (§F); 155 (§K) | "re-estimate these five tickets"; Linear package | §F: not done, dropped (roadmap §8). §K: one line with the OD-02 outcome. | S56-22, DOCS-08 |
| same | 144-146 (§J) | links to `../../../../career-companion-*-main/` | After MV-03, use the pushed repos' GitHub URLs: relative links cannot leave a repo on GitHub and break without `-main`. | DOCS-23 |
| same | 7 | "Sprint 6 implementation has not begun" | Covered by the :3 and :5 banners. Verify only. | S56-20 |
| `S6-01.md`, `S6-03.md`, `S6-04.md`, `S6-05.md` | 3 (also S6-01 :10, :202; S6-05 :123) | "implementation … pending"; D2 unresolved; Sprint 5 "present" | Banner like `S6-02.md:3`: "Implemented 2026-10-02 (execution report). D1 events-only, D2 approved. Live entry gate moved per OD-01. Sprint 5 Gmail reliability absent (S5-FU-01)." `S6-02.md` already has its banner (:3 covers "blocked on D1" at :5); verify only. | DOCS-15, S56-19, S56-20 |
| `linear-conversion.md` | top (25, 27) | D1/D2 pending; Sprint 5 "implemented, uncommitted"; re-estimate | Banner: D1/D2 decided (§L); Sprint 5 work absent; re-estimation dropped; Linear per OD-02. | S56-19, S56-20, S56-22 |
| `verification-runbook.md` | top (7, 32, 148) | D1/D2 unresolved; sync-contracts "Skipping sync" | Banner: D1/D2 recorded in §8 (:181); frontend `scripts/sync-contracts.mjs:15-17` now exits 1 when the backend is missing; Sprint 5 tree absent. | S56-19 |
| `architecture-review.md` | 17 | "*-main directories are now active Git worktrees" | Correction: no git until MV-03; paths at :11-13 are historical. | DOCS-15 |
| `execution-report.md` | 99 | "R06–R08 remain open follow-ups" | Correction: R06 fixed (AI-00); R08 item 3 fixed (AI-12); R07, R08 per their closeout resolution. | DOCS-15, S56-20 |
| `planning-review.md` | 107 | link `verification-runbook.md#8-execution-record--pending` (the heading is now "8. Execution record — 2026-10-02", so the anchor is broken) and "remains pending" | Point the link at `#8-execution-record--2026-10-02` and say the record exists. Found by the planning link check. | DOCS-15 |
| `review-2026-10-02/README.md` | 86-90 (§F) | ADR-0001 "a **Proposed** decision"; "do not migrate" | One line: accepted and implemented locally 2026-10-02 (ADR-0001:5). | DOCS-15 |

### 4.4 Docs repo — BYO AI and MCP planning

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `byo-ai/README.md` | 9 | gate "Sprint 6 committed · real git repositories identified" | Correction: waived by the owner (no git); met at MV-03: <hashes>. | DOCS-09, DOCS-08 |
| same | 47; 964-966 (§15.2) | "Two product decisions remain … who signs off"; only a recommendation | Under §15.2 write "**Outcome:**" from OD-09, or "Open: OD-09 (roadmap §6)". | AIF-07 |
| same | 70, 107, 607, 654, 848, 908 | `backend/eval/ai/` | `backend/src/eval/ai/` | DOCS-17 |
| same | 655, 748; 839 | "CI runs …", "Evaluation (CI part)"; "Status field: implemented, with commits" | "the backend test suite"; "… with the commit hashes recorded at MV-03" | DOCS-24, DOCS-17 |
| `byo-ai/issues.md` | 159, 163, 164, 170, 184; 31, 179 | `eval/ai/…`; "CI tests", "in CI" | `src/eval/ai/…`; "tests in the backend suite" | DOCS-17, DOCS-24 |
| same | 831 (AI-17) | "ADR-0001 status shows implemented with commits" | "… with the commit hashes recorded at MV-03" | DOCS-17, AIF-17 |
| same | 843-860 (AI-18) | scope: "Remove the Prisma model and any remaining test references" | Add to Scope: update `scripts/verify-ai-migration.cjs` (:91 INSERT, :101 snapshot); clear `STABILIZATION.md:47` and `sprint-5/operations.md:44`. Skip what S6-C01's AI-18 note already says; S6-C01 also removes the `application-status.test.ts:45` reference. | AIB-09 |
| `byo-ai/execution-report.md` | 31-42; 47 | no deviation listed; AI-15 certifies "all three" | Add to "Changes from the plan": AI-09 cut over before Gemini was certified (README:811 rule not met); a production build has no AI provider until AI-15 passes. :47: link the widened AI-15. | AIF-06, AIB-01, AIB-02 |
| `mcp-feature/README.md` | 58 | "Antigravity … is verified in MCP-09" | Part B outcome wording. | MCPF-08, DOCS-18 |
| same | 321 (§H) | "review queue for anything uncertain, user can ignore" | Append: "CREATED and LINKED results are permanent in v1 (OD-10). Start live use with a low `MCP_DAILY_SUBMISSION_LIMIT`." | MCPB-02 |
| same | §H, after 332 | (missing) | Row: an email processed before automation creates its application stays UNMATCHED; matching runs once per email (`pipeline.ts:118`). Mitigation: unmatched-email review. Accepted limitation. | MCPB-10 |
| same | §H, after 332 | (missing) | Row: different title wording ("Data Engineer" vs "Data Engineer II") leaves the email UNMATCHED; automation always sets a title (`jobTitle` `.min(1)`, `externalSubmission.ts:50`) and `matcher.ts:88-90` drops a candidate with a different title. Mitigation: unmatched review; after one manual link, later mail in the thread attaches (thread match, `matcher.ts:43-63`). Accepted; count it in daily use first (roadmap §8). | MCPF-13 |

### 4.5 Backend repo

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `STABILIZATION.md` | 9, 11 | "starting with `AI_DAILY_CALL_LIMIT=0` … the normal default is 100 calls/day"; "small nonzero AI budget" | Step 3: "starting with `AI_USER_DAILY_CALL_LIMIT=0` (per user, default 500) …; production refuses `AI_DAILY_CALL_LIMIT` (`config.ts:33-34`)". Step 5: "a small nonzero `AI_USER_DAILY_CALL_LIMIT`". One line above step 1: "These are the 2026-09-26 stabilization steps; the BYO AI and MCP sections below add their own." | DOCS-11, AIF-10, PLAT-16 |
| same | 42-43 | deploy (step 4) comes before the certification gate (step 5) | Before step 4: "Do not deploy BYO AI to production before at least one provider is certified (AI-15, Gemini first). Until then production offers no provider and every email waits as `PENDING`." | AIB-01 |
| same | 139; 140 | "Budget exhaustion … can exhaust that queue allowance"; "existing 90-day INBOX scope" | "A spent limit, pause or cooldown is waiting, not failure: the email returns to `PENDING` and no retry is used (`emailProcessingJob.ts:114-141`); non-provider failures still use queue retries." "the configured lookback (1/7/14/30 days, default 1; `gmailSync.ts:84,194`)" | DOCS-11, AIF-10, DOCS-02 |
| same | 154; 160; 177 | "Daily budgets contain only day/count/cooldown"; rollback check reads as scripted; "residual users/jobs/budgets" | "`ai_usage_days` holds calls, tokens and verifications; the cooldown is on `ai_configurations`; `ai_call_budgets` is unused (AI-18)." Add "(manual check; no script)". "users/jobs/budgets/mcp". | DOCS-11, TEST-13, MCPF-15 |
| same | after the Configuration table (:112-128; key row :123) | (missing) | Paragraph from BYO README §8 (:698): changing the key forces every user to re-enter theirs (first use shows `KEY_UNREADABLE`); no rotation tool. If compromised: set a new key; delete all `ai_configurations` rows by hand (unreadable rows still hold ciphertext the leaked key opens, also in backups); ask users to revoke their provider keys and set up AI again. | AIB-12 |
| `README.md` | 16, 22; 87 | "Node.js and Docker"; lanes name only `verify-migration-preservation.cjs`, no build | Add "A native PostgreSQL 15 also works." `npm run build && …` and all three lanes (`verify-ai-migration.cjs`, `verify-mcp-migration.cjs`). | PLAT-16, TEST-13 |
| `.env.example` | 10-11 | `DATABASE_URL` built from `${POSTGRES_*}` | Literal URL with the same dev values, plus: "write it literally; plain dotenv does not expand `${}`". | PLAT-04 |
| same | 13-14 | `ENABLE_DEV_AUTH=true`, so a `.env` copied from it turns header login on | `ENABLE_DEV_AUTH=false`, with the comment "true only for curl/tests; anyone on the network can then act as any user email". Why: the header is accepted whenever `NODE_ENV` is not `production` (`requireAuth`, `middleware/auth.ts:50-56`), it looks the user up by email (`developmentAuthMiddleware`, :17-26), and the server listens on all interfaces (`index.ts:115`). `.env.test` and `.env.smoke.test` keep their own `true` (:5); the smoke also sets it itself (frontend `scripts/smoke-stabilization.mjs:48`). No code change. | PLAT-05, PLAT-06 |
| same | 21; 31 | "bypass OAuth by setting VITE_DEV_USER in frontend"; "Any random string" | "Browser sign-in is Google only; `ENABLE_DEV_AUTH=true` allows the `X-Development-User` header for curl and tests (`requireAuth`, `middleware/auth.ts:50-56`)." "Must be non-empty: empty is kept (`index.ts:45`), and sign-in and Gmail connect then return 500." | PLAT-04, PLAT-16 |
| `src/eval/ai/README.md` | 10 | "covers them in CI" | "covers them in the backend test suite (`npm test`)" | DOCS-24, TEST-13 |
| `scripts/verify-migration-preservation.cjs` | 7 (comment) | usage has no build step | `npm run build && …` (line 15 loads `../dist/utils/testDatabase`). Comment only. S6-C01 edits other lines of this file and leaves this comment to S6-C02 (S6-C01 §5). | PLAT-16 |

### 4.6 Frontend repo

| File | Lines | Wrong statement | Correct statement | Finding |
| --- | --- | --- | --- | --- |
| `README.md` | 19 | "Ensure `.env` is created based on `.env.example`" | "No environment variables are needed; Vite proxies `/api` and `/mcp` to 127.0.0.1:3000 (`vite.config.ts`)." | PLAT-16 |
| same | 36 | only application and history responses are parsed | add AI settings, integration-token and submission responses (`src/api/client.ts:295-388`) | MCPF-15 |

### 4.7 Decisions

1. **OD-01 record** (required before start; the owner's dated answer is in the MV-16 report). In `sprint-6/README.md` §L, after :164, write the owner's words. Recommended text: "**Decision record (owner, YYYY-MM-DD) — OD-01:** Sprint 6 is closed for engineering. The live Gmail/original-data entry gate (§B, including deployed API/worker versions) was not passed. It is moved, unchanged, to the pre-release gate 'live Gmail and original-data evidence', which runs after S5-FU-01 and S5-FU-02 and before any deployment. Its original-data part needs S5-FU-01 FU-5 and applies only if OD-13 finds the original dataset; otherwise the owner records that part as not applicable, with the date. Original-data preservation is not retired." Add the same gate as a precondition line in [roadmap §7](../../README.md#7-release-track-not-scheduled).
2. **OD-02 outcome** (required before start). Recommended: amend the constitution to v0.3, following its Change Management section (`PROJECT_CONSTITUTION.md:161-171`: what, why, boundaries, risk, verification). Exact changes:
   - Roles (:42-50): "Implementation — Gemini + Antigravity" becomes "Implementation — Claude Code, run and reviewed by the owner" (keep :46, AI output is verified). "Project Execution — Linear" becomes "local planning docs in `docs/planning` with local IDs; Linear is an optional mirror of reviewed tickets." ChatGPT and Notion lines stay unless the owner says otherwise. AI rules (:84) "When using Antigravity/Gemini" becomes "When using an AI coding agent".
   - Workflow: step 4 (:71) "one Linear issue at a time" becomes "one ticket at a time"; step 8 "Close — update Linear …" becomes "record the evidence in the sprint or feature execution report".
   - Heading "Linear Issue Quality Standard" (:109) becomes "Ticket Quality Standard"; content unchanged.
   - DoD (:141) "notes are recorded in Linear" becomes "evidence is recorded in the execution report, with links". "Focused changes are committed" stays.
   - Versioning (:175-180): add row 0.3 with date, owner and "OD-02".
   - Banners, not rewrites: `docs/engineering/engineering-workflow.md` (Linear lines :14, §5 :72-99, DoD :137-138) and `docs/ai/ai-collaboration-workflow.md` (roles §2) point to v0.3; `ai/linear.md` says Linear is optional.
   If instead the owner restores Linear: no constitution change; §K and both linear-conversion banners say closed issues will be back-filled as a separate task.
   If OD-02 keeps Linear (as mirror or tracker): reviewed tickets may be mirrored right after MV-16; they do not wait for this ticket. S6-C02 only back-fills closed Sprint 5/6 issues: a note in §K and the linear-conversion banners now; the issue creation itself is the separate task in §5.
3. **BYO AI checkboxes** (DOCS-21; no OD covers it, so the owner decides at ticket start). Recommended: do not tick the per-issue boxes in `byo-ai/issues.md`; add one line under "Status" saying the status table is the record and boxes follow roadmap §10. Exception: AI-00's three boxes (:86-88) are ticked with evidence (MV-03 hashes, where the Sprint 6 layer commit is the BYO baseline; the S6-R06 resolution; the O8 answer from MV-02 and the ordering outcome at `byo-ai/README.md:963`) or marked "waived by owner (YYYY-MM-DD)".

### 4.8 Acceptance records (after §4.1–§4.7)

**Evidence source (2026-10-02).** Closeout orders 1–7 change code that these boxes cover: AI-19 adds a migration, S6-R07 changes the matcher, S6-R08 changes contracts, S6-C01 changes tests and a lane. So MV-16 is the baseline only. Sprint 6 DoD boxes and S6-R01..S6-R05 cite one **final closeout run** on the closeout head, with counts ([§11](#final-closeout-run-2026-10-02)). A box the final run does not cover stays unticked with a reason.

- [ ] Planning-time banners listed in §3 are present; the `sprint-6/execution-report.md` correction (§4.3) is added.
- [ ] **Sprint 6 DoD** (`sprint-6/README.md` §H, :98-115). Tick a box only with a link:
  - Box 1 (entry gate): stays unticked; append "moved by owner (OD-01, YYYY-MM-DD); baseline = MV-03 hashes".
  - Boxes 2, 4, 5, 6, 10, 11, 12, 13, 14: the suites, lanes and smoke of the final closeout run (§11); MV-16 is the baseline only.
  - Boxes 3 and 9: also the S6-C01 lock-test fix (TEST-08); box 9 also the S6-R07 resolution.
  - Box 7: S6-R08 item 2 (OD-12). Box 8: this ticket's §4.3 edits. Box 18: this ticket done.
  - Box 15: smoke PASS; append "re-created in Sprint 6; crash harness is S5-FU-01". Box 16: only if an `SMOKE_INJECT=http-drain` or `worker-drain` run is recorded. Box 17: the final-run checks plus a recorded 390 px keyboard check (a manual run noted in the closeout execution report; the smoke uses mouse clicks at 390 px, S56-19). Any box without evidence stays unticked with a one-line reason.
- [ ] OD-01 recorded in §L and roadmap §7; OD-02 outcome applied (§4.7); "decided YYYY-MM-DD" appended to both rows in roadmap §6.
- [ ] **S6-R01..S6-R05:** tick each acceptance box the final closeout run covers, citing that run (MV-16 is the baseline only); leave the rest with a reason.
- [ ] **`byo-ai/issues.md` status table** (:27-47): AI-00 with repo URLs and MV-03 hashes and the O8 answer (MV-02); AI-02 and AI-04 "Done in code; live Gemini part moved to AI-15"; AI-15 links [the widened ticket](../../byo-ai/AI-15-certify-gemini-first.md) (Sprint 8; from `issues.md` the link is `AI-15-certify-gemini-first.md`); M5 (:58) says Gemini first; AI-17 "Docs corrected (S6-C02, YYYY-MM-DD); release run waits for AI-15 and the release track"; rows for AI-19 and AI-20 with their closeout result.
- [ ] **`mcp-feature/README.md` §G** (:300-314): add "Re-verified on the new PC (YYYY-MM-DD): <MV-14 link>" under the ticked boxes. Tick "Antigravity connects …" only if part B steps 1–7 are recorded; tick "opt-in …" only if MCP-08 is applied and its three walkthroughs pass. Update `mcp-feature/execution-report.md` rows for MCP-08 and part B, and the status line of `MCP-08-automation-changes.md`.
- [ ] **S5-FU-01/S5-FU-02 split:** every Sprint 5 banner above names the right ticket (fencing, crash test, minimal events → S5-FU-01, Sprint 7; Google request bounds → S5-FU-02, Sprint 8; FU-5 → OD-13; full counters → not now).
- [ ] Closeout README §6: tick the exit criteria that now have evidence.

## 5. Out of scope

- Any code behavior change. This includes the misleading start-up warning at `routes/auth.ts:27`, the stale model comment at `schema.prisma:350` (AI-18 drops the model) and the `budgets` snapshot test (S6-C01).
- Rewriting historical reports, plans or reviews; filling the `ai/*.md` skeletons.
- Creating Linear issues (OD-02 decides; a separate task).
- Text owned by other tickets: the consent text (AI-19), the MCP privacy sentence (MCP-10, MCP-08), S6-R08 copy, the S5-FU-01 rescope body, and toolchain notes such as `npm ci` (S7-01).

## 6. Likely files and components

Docs repo: `README.md`, `MVP-READINESS-REPORT.md`, `PROJECT_CONSTITUTION.md`, `ai/*.md`, `docs/architecture/{mvp-architecture,high-level-architecture,email-ai-pipeline,ai-capability-architecture}.md`, both ADRs, `docs/domain/domain-model.md`, `docs/product/{product-vision,user-flows}.md`, `docs/ai/{provider-evaluation,ai-collaboration-workflow}.md`, `docs/engineering/{engineering-workflow,stabilization-audit}.md`, and in `docs/planning`: `README.md` (§6, §7), `sprint-5/*`, `sprint-6/*`, `byo-ai/*`, `mcp-feature/*`, `sprint-6/closeout/execution-report.md`. Backend: `STABILIZATION.md`, `README.md`, `.env.example`, `src/eval/ai/README.md`, `scripts/verify-migration-preservation.cjs` (comment). Frontend: `README.md`.

## 7. Implementation notes

- **Data model, migration, API, contracts, jobs:** no impact. No file in `src/contracts` changes, so `npm run sync-contracts` must report 0 changes.
- **Order:** one focused commit per repo for corrections (§4.1–§4.6), then one docs commit for decisions and records (§4.7–§4.8), so records can cite the correction commits.
- **Wording:** plain, short sentences. Fixture or synthetic results are never called live evidence; old-laptop results count only after the new-PC re-run.
- **Failure and recovery:** if evidence for a record is missing, leave the box unticked and write why. If a decision is not recorded, stop §4.7–§4.8 and finish §4.1–§4.6.
- **Rollback:** `git revert` the commits. Nothing runtime depends on these files.

## 8. Dependencies

- Phase 0 gate passed ([MV-16](../../migration-verification/README.md#mv-16--gate-record-results-and-decisions); report at `docs/planning/migration-verification/verification-report.md`): new-PC counts, lanes and smoke (the baseline for the final closeout run, §11); MV-03 repo URLs and the 8 layer commit hashes; [MV-14](../../migration-verification/README.md#mv-14--verify-mcp-with-real-clients) result; MV-02 answers for O8 and OD-13.
- Closeout orders 1–7 done or their state recorded: [AI-19](../../byo-ai/AI-19-runtime-safety-fixes.md), [AI-20](../../byo-ai/AI-20-recovery-path-and-status-ui.md), [S6-R07](../review-2026-10-02/S6-R07-guard-automatic-rematch.md), [S6-C01](S6-C01-make-done-claims-provable.md), [S6-R08](../review-2026-10-02/S6-R08-ux-contract-polish.md), MCP-08 with MCP-09 part B, [MCP-10](../../mcp-feature/MCP-10-post-verification-corrections.md).
- OD-01 and OD-02 recorded ([roadmap §6](../../README.md#6-owner-decisions)); OD-09, OD-10, OD-12 and OD-13 outcomes used where known.

## 9. Security and privacy

- `.env.example` keeps every secret empty; the literal `DATABASE_URL` uses only the dev values already in the file and `docker-compose.yml`.
- The key-compromise paragraph contains no key values. Deleting `ai_configurations` removes users' encrypted keys, provider choices, consent records and cooldowns; it touches no other table (only `User` relates to `AIConfiguration`, `schema.prisma:375-401`).
- Records cite commit hashes and test counts only: no tokens, email content, addresses or Discord IDs.

## 10. Acceptance criteria

- [ ] Every row in §4.1–§4.6 is applied or noted "already correct" in the closeout execution report.
- [ ] The grep checks in §11 return nothing (the MCP-09 check may return a line only if it cites the part B date).
- [ ] No relative link in a touched file is broken.
- [ ] Backend and frontend diffs touch only the files in §4.5–§4.6; the script change is a comment.
- [ ] OD-01 appears in `sprint-6/README.md` §L and roadmap §7; the OD-02 outcome is applied as in §4.7.
- [ ] Each ticked box in §4.8 links its evidence; Sprint 6 DoD box 1 stays unticked with the OD-01 note.
- [ ] The AI-17 status row no longer says "Docs done"; AI-00 has hashes and the O8 answer, or a dated owner waiver.

## 11. Testing

No new tests. Run from each repo root:

```bash
# backend (no behavior change expected)
npm run db:generate && npm run typecheck && npm run lint && npm run build
node --check scripts/verify-migration-preservation.cjs
node -e "const e=require('dotenv').parse(require('fs').readFileSync('.env.example'));new URL(e.DATABASE_URL);console.log('DATABASE_URL ok')"
grep -n "90-day\|AI_DAILY_CALL_LIMIT=0\|100 calls/day" STABILIZATION.md   # expect nothing
grep -n "VITE_DEV_USER" .env.example; grep -n "in CI" src/eval/ai/README.md   # expect nothing
grep -n "^ENABLE_DEV_AUTH=false" .env.example   # expect one line

# frontend
npm run sync-contracts   # expect 0 changed
npm run typecheck && npm run lint && npm run build

# docs repo
grep -rn "90-day INBOX\|global daily call budget" docs/architecture        # expect nothing
grep -rn "backend/eval/ai" docs/planning/byo-ai; grep -rn "Supabase" ai    # expect nothing
grep -n "verified in MCP-09\|compatibility is verified" docs/architecture/mvp-architecture.md docs/architecture/decisions/ADR-0002-automation-submissions-via-mcp.md docs/planning/mcp-feature/README.md   # nothing, unless part B passed and the line cites the date
grep -n "^| AI-17" docs/planning/byo-ai/issues.md                          # no "Docs done"
node -e 'const fs=require("fs"),p=require("path");let bad=0;const w=d=>fs.readdirSync(d,{withFileTypes:true}).flatMap(e=>e.isDirectory()?(e.name[0]=="."?[]:w(p.join(d,e.name))):e.name.endsWith(".md")?[p.join(d,e.name)]:[]);for(const f of w("."))for(const m of fs.readFileSync(f,"utf8").matchAll(/\]\(([^)#\s]+)/g))if(!/^[a-z]+:/i.test(m[1])&&!fs.existsSync(p.resolve(p.dirname(f),m[1]))){bad++;console.log(f,m[1])}process.exit(bad?1:0)'
```

This ticket's own edits need no smoke run: nothing browser-visible changes. The final closeout run below includes the smoke, because orders 1–7 changed code.

### Final closeout run (2026-10-02)

Run once on the closeout head: after closeout orders 1–7 and this ticket's §4.1–§4.6 commits, before any §4.8 tick. Use the Phase 0 shell setup (`ROOT`, `PGURL`, `LOG`, `mkdb`, `rmdb`) and the [MV-06..MV-10](../../migration-verification/README.md#mv-06--static-checks-and-builds) commands unchanged. Record in the closeout execution report the head commit of each repo and every count next to its MV-16 baseline. MV-16 is the baseline only; this run is the evidence §4.8 cites.

1. Static checks and builds (MV-06): backend typecheck, lint and build; frontend contracts `diff`, `sync-contracts` (0 changed), typecheck, lint and build. Record error and warning counts.
2. Test database (MV-07 guarded migrate): 16 migrations applied; the newest is AI-19's `<timestamp>_ai_access_paused`.
3. Both full suites (MV-08): backend and frontend `npm test`. Record passed tests and files; both rise from the MV-16 baseline as orders 1–7 add tests.
4. The three migration lanes (MV-09), on new fresh/upgrade database pairs: three `*_verified` events, each exit 0, with their row counts.
5. The real-worker smoke (MV-10): re-run the smoke-lane migrate first so it has all 16 migrations. Record the PASS line and the teardown JSON (residue 0).

## 12. Documentation updates

This ticket is the documentation update. Also: record the edits, skips and evidence links in the closeout execution report; append the OD-01 and OD-02 outcomes to roadmap §6 and the gate line to roadmap §7. Roadmap §9 needs no edit; it already points here.

## 13. Definition of done

- Acceptance criteria met.
- Backend and frontend typecheck, lint and build green (in CI once S7-01 exists); `sync-contracts` reports 0 changes; the grep and link checks pass.
- Every ticked box links new-PC evidence; nothing synthetic is called live evidence.
- The final closeout run (§11) is recorded on the closeout head, with commit hashes and counts: both full suites, typecheck, lint, build, `sync-contracts`, the three migration lanes and the smoke. Sprint 6 DoD and S6-R01..S6-R05 ticks cite it; MV-16 is cited only as the baseline.
- Focused commits per repo; evidence and any "already correct" or "left unticked" notes recorded in the closeout execution report.
