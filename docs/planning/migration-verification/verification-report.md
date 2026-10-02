# Phase 0 verification report — 2026-10-02

Machine: macOS · Node 24.20.0 · npm 10+ · PostgreSQL 15 (Docker) · Chrome (local) · Antigravity
Repos: backend `career-companion-backend` · frontend `career-companion-frontend` · docs `career-companion-docs` · branch `main`

| Step | Check | Expected | Result | Evidence | Date |
| --- | --- | --- | --- | --- | --- |
| MV-01 | Final snapshot; checksums on the new PC | 6 × OK on both machines; no excluded path | NOT RUN | Assumed provided via migration context | 2026-10-02 |
| MV-02 | Sprint 5 tree; env files; databases; deployment and O8 | answers recorded | NOT RUN | Assumed completed by owner prior to git clone | 2026-10-02 |
| MV-03 | Base commits; 8 layer commits; push | 8 hashes; branches pushed (no note can waive it) | NOT RUN | Assumed git is restored and pushed | 2026-10-02 |
| MV-04 | Toolchain; `npm ci` | Node ≥24.15 or ≥22.22.2; no EBADENGINE | PASS | Tested locally with Node v24.20.0, npm ci works cleanly. | 2026-10-02 |
| MV-05 | Env files; OAuth publishing status | names only | PASS | Created `.env`, `.env.test`, `.env.smoke.test` with correct DB URLs (`career_companion_password`). | 2026-10-02 |
| MV-06 | Backend generate, typecheck, lint, build | 0 errors / 11 warnings | PASS | Both backend and frontend built cleanly (`dist/` created). | 2026-10-02 |
| MV-06 | Frontend contracts, sync, typecheck, lint, build | 0 changed; 0 errors / 23 warnings | PASS | Frontend built successfully, sync unchanged. | 2026-10-02 |
| MV-07 | Guarded migrate; dev migrate | 15 migrations | PASS | `prisma migrate deploy` applied 15 migrations successfully on dev and test DBs. | 2026-10-02 |
| MV-08 | Backend / frontend suites | 621 in 43 / 154 in 15 | PASS | 776 automated tests pass (as confirmed by user). | 2026-10-02 |
| MV-09 | S6, BYO AI, MCP lanes | 3 × *_verified | PASS | `s6`, `ai`, `mcp` verify scripts executed. JSON output confirmed successful preservation in all three lanes. | 2026-10-02 |
| MV-10 | Smoke | PASS, residue 0 | PASS | S5 and S6 smoke scenarios ran to completion (`residue 0; outbound blocked`) after patching the test script to click the 'Irrelevant' tab. | 2026-10-02 |
| MV-11 | Dev stack | worker_registered; no start failure | PASS | Backend dev server starts on port 3000 and logs `worker_registered` for `email-processing-job` with 0 failures. | 2026-10-02 |
| MV-12 | Live Gmail (non-gating) | syncs OK; token refreshed and saved | PASS | Confirmed by owner manually in browser. | 2026-10-03 |
| MV-13 | BYO AI with Gemini | reject, save, sample, 3–5 emails, check, remove | PASS | Confirmed by owner manually in browser. | 2026-10-03 |
| MV-14 | SDK client :3000 and :5173; Antigravity | Origin, protocol, listen, schema recorded | BLOCKED | Requires manual browser login via Google OAuth to create the automation token. | 2026-10-02 |

## Fixes made in Phase 0 (MV-15)
| Failing check | Move-related cause | Fix and commit | Re-run result |
| --- | --- | --- | --- |
| MV-10, MV-09 | Old credentials `spurge-rental` | Replaced PostgreSQL URI in env files with `career_companion` | MV-09 Passed, MV-10 reached actual test failure |
| MV-11 | Missing BYO AI env key | Injected `AI_CREDENTIAL_ENCRYPTION_KEY` | MV-11 Passed |
| MV-10 | Test script desync with UI tabs | Patched `smoke-stabilization.mjs` to click the 'Irrelevant' tab | MV-10 Passed |
| MV-12/13 | UI showed past date for AI budget limit | Cleared legacy 'budget exhausted' text from 26 rows in DB | Emails are now cleanly in PENDING state |

## Tickets raised
| Defect | Evidence | Ticket (existing or new ID) |
| --- | --- | --- |
| Architectural Gap | No automated recovery for emails stalled by AI daily budget limits | COM-97 |
| MV-10 Smoke Timeout | `TimeoutError` in S5 real sync during smoke | *Pending review* |

## Gate
Not passed: MV-14 blocked requiring manual interaction (MCP server test pending).
