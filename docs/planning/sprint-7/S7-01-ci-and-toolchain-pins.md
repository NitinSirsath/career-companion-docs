# S7-01 — Minimal CI for backend and frontend, with toolchain pins

| Field | Value |
| --- | --- |
| Status | Planned — not started (local ticket, 2026-10-02) |
| Phase / sprint | Sprint 7 — order 1 of 6 |
| Repository | career-companion-backend, career-companion-frontend (docs repo: doc updates only; a docs link check is optional and not part of this ticket) |
| Size / priority | M / first in Sprint 7; guards every later change |
| Depends on | Phase 0 gate passed with git restored and pushed (MV-03, OD-03); Sprint 6 closeout, especially S6-C02 (CI wording); OD-02; OD-08 (no deploy) |
| Blocks | S5-FU-01 (CI lands before the large fencing change); S7-05 and S5-FU-02 (both list S7-01 CI as a dependency) |
| Source | [State audit 2026-10-02](../state-audit-2026-10-02.md) PLAT-08, TEST-04, TEST-18, PLAT-14, TEST-10, TEST-02, PLAT-17; [Sprint 7 README](README.md) §3, §6, §7; [roadmap](../README.md) §9 row "'CI' enforces checks" |
| Linear | Not created. Local ID only; no identity, priority or estimate is assumed. |

## 1. Objective

Every push and pull request in the backend and frontend repos runs the checks the owner runs by hand today: install from the lockfile, generate, typecheck, lint, build, guarded migrate and the full test suites (the backend against a throwaway PostgreSQL 15), plus the three migration lanes and a contract drift check. Both repos declare the Node version they need. There is no deploy step.

## 2. Why it exists

- 775 tests (backend 621 in 43 files, frontend 154 in 15 files; static count 2026-10-02) and the test-database guard run only when someone runs them by hand on one machine (PLAT-08, TEST-04).
- S5-FU-01 is the only L ticket in Sprint 7 and changes Gmail sync fencing. It should land on a CI-guarded baseline ([Sprint 7 README](README.md) §6).
- Several docs say "CI" where only a manual `npm test` exists. S6-C02 corrects the wording first; this ticket makes a true "CI runs it" statement possible (TEST-04).
- Backend/frontend contract drift is caught only if someone remembers to run `sync-contracts` (TEST-18).
- Node is not pinned, and a locked dev dependency needs Node `^22.22.2 || ^24.15.0 || >=26` (PLAT-14, TEST-10).
- The backend `.env.test` has no template, and a move through git drops it (TEST-02).
- A few stray items add noise once checks run on every push (PLAT-17).

## 3. Current behavior

Facts (checked in the code on 2026-10-02; backend = `career-companion-backend-main`, frontend = `career-companion-frontend-main`):

- **No CI anywhere.** No `.github/`, hook config, `.nvmrc`, `.node-version`, `.tool-versions` or `.npmrc` exists in any of the three folders. [engineering-workflow.md](../../engineering/engineering-workflow.md):163 says CI/CD is deferred.
- **Backend scripts.** backend `package.json:6-21` (`scripts`): `test` = `vitest run`, `build` = `tsc`, `lint` = `eslint src/` (:11), `typecheck` = `tsc --noEmit` (:13), `db:generate` = `prisma generate` (:17), `db:up` = `docker-compose up -d` (:14). No `engines` field.
- **Test guard.** backend `vitest.config.ts:5-7` includes `src/**/*.test.ts`, sets `fileParallelism: false` and the setup file. `src/tests/setup.ts:6-8` loads `.env.test` with override, then calls `assertTestDatabase`. `src/utils/testDatabase.ts:1-18` (`assertTestDatabase`) requires `DATABASE_URL === TEST_DATABASE_URL`, a `postgres:` or `postgresql:` URL, a database name matching `^career_companion_[a-z0-9_]*test$`, host `localhost`, `127.0.0.1` or `[::1]`, and no query parameter except `schema=public`. So even DB-free test files need `.env.test` (or both URLs exported in the shell).
- **Guarded migrate.** backend `scripts/guarded-migrate.cjs:6-7` requires an exported `TEST_DATABASE_URL`; :8-9 loads `TEST_ENV_FILE` or `.env.test` and throws if it is missing; :10 requires `../dist/utils/testDatabase` (so `npm run build` must run first); :19 runs `migrate deploy`, `migrate status`, `validate` by default.
- **Migration lanes.** `scripts/verify-migration-preservation.cjs`, `verify-ai-migration.cjs` and `verify-mcp-migration.cjs` each need `FRESH_DATABASE_URL` and `UPGRADE_DATABASE_URL` (two separate, empty, guard-named databases), load `../dist/utils/testDatabase` (for example `verify-mcp-migration.cjs:16`) and write their own temporary env file in `guardedMigrate` (`verify-mcp-migration.cjs:30-37`). They never read `.env.test`.
- **Env files.** backend `.gitignore:4-6` ignores `.env` and `.env.*` except `!.env.example`, so a new `.env.test.example` would be ignored without its own `!` line. `.env.example` has 16 of the 18 names `.env.test` uses; it lacks `TEST_DATABASE_URL` and `NODE_ENV`. The 64-hex and "must differ" key rules are at `.env.example:42-51`.
- **The 18 `.env.test` names** (local fixture file, not in git): `DATABASE_URL`, `TEST_DATABASE_URL`, `NODE_ENV` (=test), `ENABLE_DEV_AUTH` (=true), `SESSION_SECRET`, `OAUTH_STATE_COOKIE_SECRET`, `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REDIRECT_URI`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `GMAIL_TOKEN_ENCRYPTION_KEY`, `AI_CREDENTIAL_ENCRYPTION_KEY`, `AI_USER_DAILY_CALL_LIMIT` (=100), `DISCORD_WEBHOOK_URL` (empty), `DISCORD_USER_ID` (empty), `FRONTEND_URL` (=http://localhost:5173). Many tests set their own values (for example `gmail-ingestion.test.ts:48`, `ai-user-limits.test.ts:61`, `auth.test.ts:49-50`).
- **Frontend scripts.** frontend `package.json:2` name is `temp-vite`; :6-14: `build` = `vite build && tsc -b`, `lint` = `oxlint`, `test` = `vitest run`, `typecheck` = `tsc -b`. No `engines`, no `repository` field. `temp-vite` also appears at `index.html:10` (`<title>`, visible as the browser tab name) and `package-lock.json:2` and :8.
- **Contract sync.** frontend `scripts/sync-contracts.mjs:9-18` finds the backend through `BACKEND_DIR` or a sibling `career-companion-backend-main` / `career-companion-backend` folder and exits 1 if none exists; :23-34 copies only `.ts` files and has no check mode. It never deletes stale frontend files (S6-R08 item 4). Today the 10 contract files are identical (`diff -r`).
- **Frontend tests** need no backend, database or env. `vite.config.ts` has no `test` section, so Vitest uses its default include pattern over the whole repo folder, and `oxlint` lints the current folder.
- **Puppeteer.** frontend `puppeteer` 25.11.0 downloads Chrome in its `postinstall` (`node_modules/puppeteer/package.json:42`); `PUPPETEER_SKIP_DOWNLOAD` turns that off (`node_modules/puppeteer/lib/puppeteer/getConfiguration.js:116`, `getConfiguration`). Only `scripts/smoke-stabilization.mjs` uses it. The smoke loads backend `.env.test` (:41), needs two 64-hex keys (:53-56) and defaults to the macOS Chrome path (:389, :1014).
- **Node ranges in the lockfiles** (scan of every `engines.node`, 2026-10-02): frontend's strictest is jsdom 30.0.1 `^22.22.2 || ^24.15.0 || >=26.0.0` (undici 8.10.2 `>=22.19.0` is looser). The backend alone allows `^22.13 || ^24 || >=26` (eslint 10.10.0 family `^20.19.0 || ^22.13.0 || >=24`; vitest 5.0.0 `^22.12.0 || ^24.0.0 || >=26.0.0`). Node 23 and 25 fail both. The old laptop ran v24.20.0. npm only warns (`EBADENGINE`) on a mismatch.
- **PLAT-17 items.** backend `test_gmail_api.js:1` is one `console.log` line; no code or doc in the three folders references it, apart from this plan. `eslint` over `scripts/` and `prisma/seed.ts` (read-only run, eslint 10.10.0): 30 errors in `scripts/*.cjs` (29 `@typescript-eslint/no-require-imports` across all five files, 1 `preserve-caught-error` at `scripts/mcp-client-check.cjs:24`), 0 in `prisma/seed.ts`. `seed.ts` is type-checked whenever the seed runs (`package.json:75`, `ts-node-dev` without `--transpile-only`); a standalone `tsc --noEmit` of it passed. `tsconfig.json:5,12` compiles all of `src/`, so `dist/tests` (45 files) and `dist/eval/ai` are built; `dist/` is ignored (`.gitignore:2`).
- **Docs that claim CI today** (before S6-C02): backend `src/eval/ai/README.md:10`; [provider-evaluation.md](../../ai/provider-evaluation.md):27; [byo-ai/README.md](../byo-ai/README.md):655, :748; [byo-ai/issues.md](../byo-ai/issues.md):31, :179; [ai-capability-architecture.md](../../architecture/ai-capability-architecture.md):286; [high-level-architecture.md](../../architecture/high-level-architecture.md):710, :779. [ai/github.md](../../../ai/github.md):45-47 says "To be defined".

Suspected risks (not demonstrated):

- Suite run time on GitHub runners is unknown. `email-worker-reliability.test.ts` polls a real worker for up to 30 s (150 × 200 ms loops at :320-321 and :357-358), and a few tests are timing-sensitive (TEST-14). A flake on CI is the roadmap trigger to revisit test hygiene ([roadmap §8](../README.md#8-not-now)).
- The S6 lane can hang after a failure (TEST-11). S6-C01 scope item 2 fixes it; until that lands, only the job timeout ends a hang.
- The S6 and AI lanes build their "legacy" schema out of order (TEST-03). A future migration that depends on the excluded migration could turn a lane red even though a real upgrade works.
- Backend repo visibility, the frontend remote and both default branch names are unverified. The backend remote in `package.json:24` is `github.com/NitinSirsath/career-companion-backend`.

## 4. Scope

1. **Backend workflow** `.github/workflows/ci.yml`. Triggers: `push`, `pull_request`, `workflow_dispatch`. Top-level `permissions: contents: read`. Job `test` on `ubuntu-latest`, `timeout-minutes: 20`, with a `postgres:15` service (same major as `docker-compose.yml:5`): `POSTGRES_USER: cc_ci`, `POSTGRES_PASSWORD: cc_ci_fixture_only`, `POSTGRES_DB: career_companion_ci_test`, port `5432:5432`, health check `pg_isready`. Job env: `TEST_DATABASE_URL=postgresql://cc_ci:cc_ci_fixture_only@localhost:5432/career_companion_ci_test?schema=public`. Steps, in this order:
   checkout → `actions/setup-node` with `node-version-file: .nvmrc` and `cache: npm` → `npm ci` → write `.env.test` (item 2) → `npm run db:generate` → `npm run typecheck` → `npm run lint` → `npm run build` (the guards load `dist`) → `node scripts/guarded-migrate.cjs` → `npm test`. The suite runs files serially and expects an empty, migrated database; a fresh service container gives exactly that.
2. **Fixture `.env.test` written in the job**, never from real secrets and never printed:
   ```sh
   umask 077
   GMAIL_KEY=$(openssl rand -hex 32); AI_KEY=$(openssl rand -hex 32)
   [ "$GMAIL_KEY" != "$AI_KEY" ] || exit 1
   cat > .env.test <<EOF
   DATABASE_URL="$TEST_DATABASE_URL"
   TEST_DATABASE_URL="$TEST_DATABASE_URL"
   NODE_ENV=test
   ENABLE_DEV_AUTH=true
   SESSION_SECRET=$(openssl rand -hex 32)
   OAUTH_STATE_COOKIE_SECRET=$(openssl rand -hex 32)
   GMAIL_CLIENT_ID=ci-fake-gmail-client-id
   GMAIL_CLIENT_SECRET=ci-fake-gmail-client-secret
   GMAIL_REDIRECT_URI=http://localhost:5173/api/gmail/callback
   GOOGLE_CLIENT_ID=ci-fake-google-client-id
   GOOGLE_CLIENT_SECRET=ci-fake-google-client-secret
   GOOGLE_REDIRECT_URI=http://localhost:5173/api/auth/callback
   GMAIL_TOKEN_ENCRYPTION_KEY=$GMAIL_KEY
   AI_CREDENTIAL_ENCRYPTION_KEY=$AI_KEY
   AI_USER_DAILY_CALL_LIMIT=100
   DISCORD_WEBHOOK_URL=
   DISCORD_USER_ID=
   FRONTEND_URL=http://localhost:5173
   EOF
   diff <(grep -oE '^[A-Z0-9_]+=' .env.test.example | sort) <(grep -oE '^[A-Z0-9_]+=' .env.test | sort)
   ```
   The last line fails the job if the committed example (item 7) and the CI file list different names. Do not set `MCP_ALLOWED_HOSTS`, `MCP_ALLOWED_ORIGINS`, `MCP_DAILY_SUBMISSION_LIMIT` or `TRUST_PROXY_HOPS`, and do not create a backend `.env` in CI: these leak into the tests (TEST-05; S6-C01 scope item 3 pins them in `setup.ts`).
3. **Migration lanes — decision. Recommendation: include them** as a second backend job, `migration-lanes`, in the same workflow, running in parallel with `test`. Reason: the Sprint 7 exit criteria require the lanes when a migration is added, and S5-FU-01 or S7-02 may add one. The job has its own `postgres:15` service, `timeout-minutes: 15` (this bounds the TEST-11 hang), and runs: checkout → setup-node → `npm ci` → `npm run db:generate` → `npm run build` → create six empty databases (`career_companion_{s6,ai,mcp}_{fresh,upgrade}_test`, for example with `psql` against the service's `postgres` database; `psql` on the runner image is unverified, `docker exec` on the service container is the fallback) → run `verify-migration-preservation.cjs`, `verify-ai-migration.cjs` and `verify-mcp-migration.cjs`, each with its own `FRESH_DATABASE_URL` / `UPGRADE_DATABASE_URL` pair. No `.env.test` is needed. If the owner prefers a faster pipeline, the fallback is to run the lanes by hand whenever `prisma/` changes, as today.
4. **Frontend workflow** `.github/workflows/ci.yml`. Same triggers and `permissions`. Job `check` on `ubuntu-latest`, `timeout-minutes: 15`, env `PUPPETEER_SKIP_DOWNLOAD: 'true'` (the smoke does not run in CI). Steps: checkout → setup-node (`.nvmrc`, npm cache) → `npm ci` → `npm run typecheck` → `npm run lint` → `npm test` → `npm run build`.
5. **Contract drift check — decision on how CI gets the backend contracts. Recommendation:** a separate frontend job, `contract-drift`, that checks out the frontend to `frontend/` and the backend repo at its default branch (expected `main`; unverified) to `career-companion-backend/`, side by side and outside the frontend folder, then runs `diff -r career-companion-backend/src/contracts frontend/src/contracts`. On failure, print `Frontend contracts differ from backend main. Run npm run sync-contracts and commit.` Reason: no script change, and `diff -r` catches changed, new and stale files. It needs no `npm ci`. Keeping it a separate job keeps the backend files away from Vitest and oxlint (§3) and keeps the `check` result readable. Repo visibility is unverified: if the backend repo is private, the owner creates a fine-grained token with read-only Contents on that one repo, with an expiry, stored as frontend secret `BACKEND_READ_TOKEN` and passed as `token:` to that checkout step; if it is public, omit `token:`. A `--check` flag in `sync-contracts.mjs` (TEST-18) is not needed with this design.
6. **Toolchain pins** in both repos: add `"engines": { "node": "^22.22.2 || ^24.15.0 || >=26" }` to `package.json` and a `.nvmrc` containing `24`. Re-run the lockfile engines scan at implementation time; if the locked versions changed, use the new strictest range. Use the same range in both repos, even though the backend alone would allow `^22.13 || ^24 || >=26`, so there is one rule. Update the lockfile root entry with `npm install --package-lock-only --ignore-scripts` and confirm that `git diff package-lock.json` touches only the root `name`/`engines` fields; if anything else moves, revert and edit the root entry by hand.
7. **Backend `.env.test.example`** (TEST-02): the 18 names in item 2's order, fake non-key values, and comments that state the guard rules (same URL in both variables; name `career_companion_*test`; host `localhost`, `127.0.0.1` or `[::1]`; only `?schema=public`; the suite runs unscoped deletes such as `prisma.user.deleteMany()` (`application.test.ts:16`), so use a dedicated empty database). Leave both keys empty, with the comment "64 hex, fixture only, must differ: `openssl rand -hex 32`". Add one note: for a smoke lane, copy it to `.env.smoke.test`, use a `career_companion_*smoke_test` name, and pass it to `guarded-migrate.cjs` with `TEST_ENV_FILE`. Add `!.env.test.example` to backend `.gitignore`.
8. **PLAT-17 hygiene, kept small:**
   - Delete backend `test_gmail_api.js` after a final grep shows no reference.
   - Rename the frontend package `temp-vite` → `career-companion-frontend` (`package.json:2`, lockfile root via item 6) and set `index.html:10` `<title>` to `Career Companion`.
   - Lint scope — **recommendation: lint `scripts/` and `prisma/seed.ts`; do not add type-checking for them.** Change backend `lint` to `eslint src/ scripts/ prisma/seed.ts`. Add an `eslint.config.mjs` block for `scripts/**/*.cjs` with `sourceType: 'commonjs'` that turns off `@typescript-eslint/no-require-imports`. Handle the one `preserve-caught-error` at `mcp-client-check.cjs:24` with an `eslint-disable-next-line` plus a reason, unless attaching `{ cause }` is shown not to print the Bearer token. Reason: these scripts guard the test database and now run in CI. Type-checking `.cjs` files (checkJs) costs more than it finds, and `seed.ts` is already type-checked when the seed runs.
   - Tests compiled into `dist`: note only, in the backend README. Any later build split (release track) must keep `dist/utils/testDatabase.js`, which the guards load.
9. **Smoke stays manual.** It needs Chrome, both builds, backend `.env.test` and an exclusive smoke database (backend `STABILIZATION.md:175`). Say so in both READMEs.
10. **Docs**, only after CI is green on main (§12). Coordinate with S6-C02: S6-C02 first rewrites false "CI" claims as "the backend test suite (`npm test`)" or similar; this ticket then states what CI really runs.
11. **Optional owner step (GitHub settings, not code):** branch protection on `main` that requires `test`, `migration-lanes`, `check` and `contract-drift`. Record whether it was done.

## 5. Out of scope

- Deploy or CD: Docker images, ECR/ECS, frontend upload ([high-level-architecture §9.10](../../architecture/high-level-architecture.md) stays planned; OD-08).
- Smoke in CI, live AI or Gmail calls, `npm run ai:eval`, `mcp-client-check.cjs`.
- Coverage thresholds, a Node version matrix, pre-commit hooks, `--max-warnings 0` / oxlint deny-warnings, and fixing the existing 11 backend and 23 frontend lint warnings.
- Branch protection itself (owner's settings; item 11 is optional).
- CI for the docs repo. A relative-link check there is optional; recommended only if broken links become a problem.
- The other PLAT-17 items: `src/client/api.ts` (unimported), the unused `eslint-disable` lines, a `tsconfig.build.json`, and `AICallBudget` (AI-18).
- `db:up` still calls the legacy `docker-compose` (PLAT-14); the Smoke Chrome fallback to Puppeteer's own browser (TEST-10); a cumulative upgrade lane or lane cutoffs (TEST-03); timing-test hygiene (TEST-14).

## 6. Likely files and components

- backend: `.github/workflows/ci.yml` (new), `.nvmrc` (new), `.env.test.example` (new), `.gitignore`, `package.json`, `package-lock.json` (root entry only), `eslint.config.mjs`, `scripts/mcp-client-check.cjs` (one lint line), `test_gmail_api.js` (deleted), `README.md`, `STABILIZATION.md`.
- frontend: `.github/workflows/ci.yml` (new), `.nvmrc` (new), `package.json`, `package-lock.json` (root entry only), `index.html`, `README.md`.
- docs: see §12.

## 7. Implementation notes

- **Data model / migrations:** none. CI applies all existing migrations (16 after AI-19) through `guarded-migrate.cjs` only, never a raw `prisma migrate`.
- **API and shared contracts:** none changed. The drift job only reads. `sync-contracts.mjs` is unchanged. Merge order for a contract change: backend first, then the frontend sync commit. A backend merge does not trigger the frontend workflow. Between the two, any frontend run of `contract-drift` (push, pull request or manual dispatch) is red; that red means "sync now". A frontend change that syncs before the backend change reaches main is also red, so merge the backend first.
- **Background jobs:** none changed. Tests start a real pg-boss, which creates the `pgboss` schema; the service's `POSTGRES_USER` is a superuser, so this works.
- **Action versions:** use the current major tags of `actions/checkout` and `actions/setup-node` (unverified which major is current at implementation time); pinning by commit SHA is stronger and optional. No other third-party actions.
- **Failure and recovery:** a red job blocks nothing by itself until branch protection exists (item 11). Red `contract-drift` → run `npm run sync-contracts` (after S6-R08 item 4 it also removes or reports stale files). A lane that turns red only because of the out-of-order legacy schema (TEST-03) → fix the lane script cutoff; never weaken the migration. A hang → the job timeout ends it.
- **Compatibility:** local development is unchanged. `engines` only warns under npm defaults, and this ticket does not set `engine-strict`. CI tests only the `.nvmrc` version; Node 22 and 26 are allowed by the dependencies but untested here.
- **Cost:** GitHub Actions minutes for private repos are limited on free plans (quota unverified). Two backend jobs and two frontend jobs per push are expected to fit; record real durations in the execution report.
- **Rollback:** delete the workflow files and revert the pin/rename commits. Nothing at runtime depends on them.
- **Commits:** focused commits per repo: pins, rename and hygiene; `.env.test.example`; workflow; docs.

## 8. Dependencies

- Phase 0 gate passed; repos restored as git repositories and pushed to GitHub (MV-03, OD-03, [migration verification](../migration-verification/README.md)). CI needs a remote.
- OD-02 recorded: Git/GitHub is the system of record and Done means committed with tests green ([roadmap §6](../README.md#6-owner-decisions)).
- [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) done, so §12 updates its wording instead of racing it.
- Helpful, not blocking: [S6-R08](../sprint-6/review-2026-10-02/S6-R08-ux-contract-polish.md) item 4 (sync removes stale files) and the TEST-11 lane-exit fix and TEST-05 env pinning in [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md) (scope items 2 and 3).
- Owner: confirm repo visibility and default branch names; create `BACKEND_READ_TOKEN` only if the backend repo is private.
- Can run in parallel with S7-02 ([Sprint 7 README](README.md) §6). Must finish before [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md) starts.

## 9. Security and privacy

- No real secret enters CI. Every value in the CI `.env.test` is fake or generated per run, and the backend workflow uses no repository secret. The file is created with `umask 077` and never printed (`cat`, `env` dumps or debug echo).
- The database guard is not bypassed or weakened. The CI database passes it because it is a local service with a guard-matching name.
- Least privilege: `permissions: contents: read`; `pull_request`, never `pull_request_target`; only official `actions/*` actions.
- `BACKEND_READ_TOKEN` (if needed) is a fine-grained, read-only, single-repo token with an expiry, never a classic token with `repo` scope.
- The tests mock the provider and Google SDKs (for example `vi.mock('googleapis')`; not checked file by file). CI holds no real key or token, so a missed mock fails instead of reaching a real account. `AI_EVAL_API_KEY` is never set in CI.
- `.env.test.example` holds no usable key, so it cannot be copied into a real environment by mistake.

## 10. Acceptance criteria

- [ ] Backend `ci.yml` runs on push, pull request and manual dispatch; job `test` runs the item 1 steps in that exact order and is green on main.
- [ ] Job `migration-lanes` runs all three lane scripts, each prints its `*_verified` JSON, and it is green on main (or the owner's choice to skip it is recorded in the execution report).
- [ ] The CI `.env.test` is generated in the job with two different 64-hex keys and fake values only; the name check against `.env.test.example` passes; no workflow log shows its values.
- [ ] Frontend `ci.yml` job `check` runs `npm ci` without a Chrome download, then typecheck, lint, test and build, and is green on main.
- [ ] Job `contract-drift` passes when the folders match and fails on a throwaway branch where one frontend contract file is edited (run link recorded; branch deleted, not merged).
- [ ] A deliberately failing test on a throwaway branch turns backend `test` red, and another turns frontend `check` red (links recorded; branches deleted).
- [ ] Both repos have `engines.node` = `^22.22.2 || ^24.15.0 || >=26` (or the re-verified range) in `package.json` and the lockfile root, and `.nvmrc` = `24`; `npm ci` on Node 24.15 or later prints no `EBADENGINE` warning.
- [ ] Backend `.env.test.example` is committed (the `.gitignore` exception works), lists exactly the CI names, has no key values and states the guard rules.
- [ ] `test_gmail_api.js` is gone; the frontend package and `<title>` no longer say `temp-vite`.
- [ ] Backend `npm run lint` covers `scripts/` and `prisma/seed.ts` with 0 errors (or the owner's other lint decision is recorded).
- [ ] Both READMEs say the smoke stays manual and why.
- [ ] No doc claims CI beyond what the two workflows run (§12 grep is clean).

## 11. Testing

No new unit tests: this ticket changes CI configuration, pins and docs. Proof comes from real runs: green on main, plus the red runs on throwaway branches above.

Run locally before pushing (backend):

```bash
npm ci
npm run db:generate && npm run typecheck && npm run lint && npm run build
TEST_DATABASE_URL='<same URL as .env.test>' node scripts/guarded-migrate.cjs
npm test                      # expect 621 passed in 43 files, unless closeout changed the count
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-migration-preservation.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-ai-migration.cjs
FRESH_DATABASE_URL=… UPGRADE_DATABASE_URL=… node scripts/verify-mcp-migration.cjs
```

Each lane needs two new empty `career_companion_*test` databases. Frontend:

```bash
npm ci
npm run sync-contracts        # expect 0 file(s) changed
npm run typecheck && npm run lint && npm test && npm run build   # expect 154 passed in 15 files
diff -r ../career-companion-backend/src/contracts src/contracts  # or ../career-companion-backend-main
```

Optional: lint the workflow files with `actionlint` if it is installed. No smoke run is needed: the only browser-visible change is the tab title. Check it once in `dist/index.html` or `npm run dev`.

## 12. Documentation updates

Do these only after CI is green on main. Line numbers are from 2026-10-02; S6-C02 edits several of these files first, so re-find them with `grep -rnw "CI"` across the three repos.

- Backend `README.md`: Foundation (:7) and Setup (:16-17) → Node from `.nvmrc` (`engines` range) and `npm ci`. Test-database paragraph (:74) → point to `.env.test.example`. Add a short "CI" paragraph: what each job runs, the smoke stays manual, and tests are compiled into `dist` (note only).
- Backend `STABILIZATION.md` "Verification commands" (:156): one line saying CI runs the static checks, the suite and the three lanes on every push; the smoke and live checks stay manual.
- Frontend `README.md`: Setup (:17-18) → `.nvmrc` and `npm ci`; Verification (:63-72) → the CI jobs and the drift fix (`npm run sync-contracts`).
- [engineering-workflow.md](../../engineering/engineering-workflow.md) §8 (:163): validation CI exists per repo since S7-01; the deploy steps stay planned (OD-08).
- [high-level-architecture.md](../../architecture/high-level-architecture.md) §9.10 (:628): note that only the "validate" step exists. [ai/github.md](../../../ai/github.md) CI/CD section (:45-47) → what the workflows run. S6-C02 only adds a "Skeleton — not filled" banner at the top of that file.
- The S6-C02-corrected spots (§3 list): say "CI" again only where CI really runs that test. Mocked adapter tests are not "recorded responses" (ai-capability-architecture.md:286).
- [Roadmap](../README.md) §1 row "CI and deployment" and §9 row "'CI' enforces checks"; the S7-01 exit criterion in the [Sprint 7 README](README.md) §7, with evidence links.
- Sprint 7 `execution-report.md` (new, dated): run links, test counts, job durations, the lane and branch-protection decisions.

## 13. Definition of done

- [ ] Every acceptance criterion in §10 is met, with evidence linked.
- [ ] Backend and frontend workflows are green on main, and the throwaway red runs are recorded.
- [ ] Behavior verified: a clean `npm ci` on Node 24 works in both repos; `diff -r` on the contracts is clean.
- [ ] Docs updated per §12; no stale or premature CI claim remains.
- [ ] Focused commits in each repo (OD-02).
- [ ] Evidence (run URLs, counts, durations, decisions) recorded in the Sprint 7 `execution-report.md`. Nothing synthetic is called live evidence.
