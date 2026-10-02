# Migration and verification gate (Phase 0)

> Office workflow update 2026-10-03: the owner says personal-Mac migration/reconciliation was pushed to GitHub and authorizes office development from those repositories. This historical personal-Mac checklist is not a prerequisite for office implementation. S7-02 now covers checkpoint gaps up to 30 days automatically, so an interim 30-day lookback is not required solely to avoid gaps. S7-04 replaces the startup manual-restart workaround with bounded retry and `/ready`; real mailbox/original-data verification remains separate.

Status: **planned, not started (local checklist, 2026-10-02).** Phase 0 is not a sprint. MV-01..MV-16 are local IDs; no Linear identity, state or estimate is assumed. Step 1 of the [roadmap](../README.md#2-order-of-work).

Source: [State audit 2026-10-02](../state-audit-2026-10-02.md) PLAT-01..PLAT-04, PLAT-06, PLAT-07, PLAT-12, PLAT-14, PLAT-19, PLAT-20, TEST-01, TEST-02, TEST-05, TEST-10, TEST-14, MCPB-01, MCPB-03, MCPB-04, MCPF-01, MCPF-02, MCPF-04, MCPF-06, MCPF-16, AIB-14, AIF-17, AIF-18, S56-10, S56-23, DOCS-09, DOCS-23; roadmap OD-01, OD-02, OD-03, OD-11, OD-13.

Goal: move the three repos to the owner's personal PC, restore git, rebuild the environment, re-run every existing check, then verify BYO AI and MCP by hand. Results measured on the old laptop count only after they are re-run here. File and line references were checked against the code on 2026-10-02 and may drift. Paths are in the backend unless they name a frontend file (`smoke-stabilization.mjs`, `sync-contracts.mjs`, `vite.config.ts`, `scripts/fixtures/`).

## Rules

1. **Follow the owner's sequence.**

   | Owner's step | MV steps |
   | --- | --- |
   | 1 Migrate the repositories | MV-01, MV-02 (old laptop), MV-03 |
   | 2 Install dependencies, recreate the environment | MV-04, MV-05 |
   | 3 Run migrations | MV-07 (after the MV-06 build: the guard loads `dist/utils/testDatabase`, `scripts/guarded-migrate.cjs:10`) |
   | 4 Tests, typecheck, lint, build | MV-06, MV-08, MV-09, MV-10 |
   | 5 Verify BYO AI | MV-11, MV-12, MV-13 |
   | 6 Verify MCP | MV-14 |
   | 7 Fix migration and verification issues | MV-15 |
   | 8 Only then sprint work | MV-16 gate, then the [Sprint 6 closeout](../sprint-6/closeout/README.md) |

2. **No Evidence, No Fix.** Phase 0 fixes only what the move broke (MV-15). Every other defect becomes a ticket.
3. **Record every result**, pass, fail or skipped, in the report (MV-16). Names, counts and hashes only. Never keys, tokens, passwords or email content.
4. **Never commit env files.** Backend and frontend `.gitignore` ignore `.env` and `.env.*` except `.env.example`. MV-03 checks staged names before each commit.
5. **Never point tests at a real database.** Test, lane and smoke databases are new, empty and named `career_companion_*test` on localhost (`src/utils/testDatabase.ts:1-18`, `assertTestDatabase`). Never give a database with real data such a name. Never point `.env.test` at the smoke database: the guard also accepts `*smoke_test` names, and the suite deletes rows in every table.
6. **No feature work and no application-code change**, except a logged MV-15 fix.

**Shell.** Commands are POSIX shell (bash or zsh). The new PC's OS is unknown. On Windows, use Git Bash or WSL, or translate `VAR=value cmd` to PowerShell `$env:VAR='value'; cmd; Remove-Item Env:VAR`. Set `CHROME_BIN` for the smoke (MV-10). `chmod 600` is Unix-only (MV-14). MV-03 needs `rsync`, which Git Bash does not ship; use WSL or install it. Use `sha256sum` (without `-a 256`) where `shasum` is missing. Set these once per terminal:

```bash
ROOT=/path/to/parent        # holds _baselines/ and the three *-main folders side by side
PGURL='postgresql://<user>:<password>@localhost:5432'   # local role; never a real secret
mkdb() { psql "$PGURL/postgres" -v ON_ERROR_STOP=1 -c "CREATE DATABASE $1"; }
rmdb() { psql "$PGURL/postgres" -v ON_ERROR_STOP=1 -c "DROP DATABASE IF EXISTS $1"; }
LOG="${TMPDIR:-/tmp}"       # scratch logs, outside the repos
```

Keep the `-main` folder names. The baseline manifests list paths as `career-companion-backend-main/...`, and the docs' cross-repo links use them (DOCS-23). The scripts also accept `career-companion-backend` (`scripts/sync-contracts.mjs:9-13`).

## Before leaving the old laptop

### MV-01 — Final checksummed snapshot

**What and why.** No snapshot covers the MCP work or these planning docs (PLAT-03, MCPB-03). Reuse the [baseline procedure](../mcp-feature/baseline.md#how-to-use-it): same excludes, per-file manifest, `SHA256SUMS`, `TIMESTAMP`. Take it last, after every planning doc is final. Any later edit on the laptop means taking it again.

```bash
ROOT=/Users/spurge-rental/Downloads/docs; cd "$ROOT"
SNAP="$ROOT/_baselines/pre-migration-$(date +%F)"; mkdir "$SNAP"
REPOS=(career-companion-backend-main career-companion-frontend-main career-companion-docs-main)   # array: zsh does not split "$REPOS"
for r in "${REPOS[@]}"; do
  find "$r" \( -name node_modules -o -name dist -o -name coverage -o -name .vite \) -prune -o -type f \
    ! -name .DS_Store ! -name '*.log' ! -name '*.tsbuildinfo' \( ! -name '.env*' -o -name .env.example \) \
    ! -name '*.pem' ! -name '*.key' ! -name '*.p12' ! -name 'id_rsa*' -print | LC_ALL=C sort > "$SNAP/$r.filelist"
  COPYFILE_DISABLE=1 tar --no-xattrs --no-mac-metadata -czf "$SNAP/$r.tar.gz" -T "$SNAP/$r.filelist"
  tr '\n' '\0' < "$SNAP/$r.filelist" | xargs -0 shasum -a 256 > "$SNAP/$r.files.sha256"
done
grep -v 'baseline.md' _baselines/mcp-feature-2026-10-02-after-byo-ai/EXCLUDES.txt > "$SNAP/EXCLUDES.txt"
(cd "$SNAP" && shasum -a 256 *.tar.gz *.files.sha256 > SHA256SUMS)
echo "$(date -u +%FT%TZ) | $(TZ=Asia/Kolkata date '+%F %T IST')" > "$SNAP/TIMESTAMP"
# Verify
(cd "$SNAP" && shasum -a 256 -c SHA256SUMS)
CHK=$(mktemp -d); for r in "${REPOS[@]}"; do tar -xzf "$SNAP/$r.tar.gz" -C "$CHK"; done
(cd "$CHK" && for r in "${REPOS[@]}"; do shasum -a 256 -c --quiet "$SNAP/$r.files.sha256"; done)
grep -E '/(node_modules|dist|coverage|\.vite)/|\.tsbuildinfo$|\.DS_Store$|/\.env' "$SNAP"/*.filelist | grep -v '/\.env\.example$'
cat "$SNAP"/*.filelist | tr '\n' '\0' | xargs -0 grep -lE 'sk-(ant-)?[A-Za-z0-9_-]{20,}|AIza[0-9A-Za-z_-]{30,}|ccmcp_[A-Za-z0-9_-]{43}|BEGIN [A-Z ]*PRIVATE KEY'
```

- **Expect:** 6 × OK; no output from the manifest check or the exclude grep; the secret scan lists only `src/tests/ai-credentials.test.ts`, `ai-providers-openai.test.ts` and `ai-settings.test.ts` (sentinel test values). On 2026-10-02 this selection gave the earlier backend and frontend file lists plus the known new MCP files: backend 161, frontend 93. Then copy the whole `_baselines/` folder to the new PC over a private channel (for example a USB drive).
- **Record:** snapshot folder, `TIMESTAMP`, the six `SHA256SUMS` lines (keep a copy apart from the drive), file counts.

### MV-02 — Last checks on the old laptop

1. **Sprint 5 tree (PLAT-02, OD-13).** Its last known place is `/Users/spurge-rental/Documents/docs/career-companion-{backend,frontend,docs}-main` ([sprint-6 architecture review](../sprint-6/architecture-review.md) :11-13). It now holds only `.DS_Store`. Search once: `ls -la /Users/spurge-rental/Documents/docs`, `mdfind -name career-companion | grep -v /Downloads/docs/`, `tmutil destinationinfo` (Time Machine). Also check the Trash in Finder (Terminal may be denied), iCloud Drive "Recently Deleted", and any older clone elsewhere (`git status --short --branch`, `git stash list`). If found, archive it unchanged with `shasum -a 256`, keep it apart from the snapshot, and hand it to [S5-FU-01](../sprint-5/follow-up-gmail-reliability.md). Never overlay it on these folders.
2. **Test env files.** Backend `.env.test` and `.env.smoke.test` hold fixture values only and are not in any snapshot. Recommended: recreate them in MV-05. Copying over a private channel also works.
3. **Local databases (owner's call).** Run `psql -l`. For each non-test database: `psql -d <db> -tAc 'SELECT (SELECT count(*) FROM users), (SELECT count(*) FROM gmail_connections), (SELECT count(*) FROM ai_configurations)'`. `ai_configurations` exists only after the BYO AI migration; drop that part if it errors. Do not copy test lanes. A database with real data is readable only with its original encryption keys, and this laptop has no backend `.env`. Dump it only if those keys turn up.
4. **Deployment and O8 (AIF-17).** Does any deployed environment exist? Who used the hosted Gemini key, and how will they be told? Only the owner can answer.

- **Record:** places searched and the result (OD-13), the env-file choice, database names with row counts, the deployment and O8 answers.

## On the new PC

### MV-03 — Restore git (OD-03)

**What and why.** No folder has git, and the commit each zip came from is not recorded (PLAT-01, S56-23). OD-03 must be decided first. The commit hashes close AI-00 O7 later ([S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md)).

1. If clones already exist on this PC, run `git status --short --branch` and `git stash list` in each. Save uncommitted work unchanged on its own branch (for example `wip/sprint5-uncommitted`) and back it up first (PLAT-02).
2. Copy `_baselines/` into `$ROOT` and run `for d in "$ROOT"/_baselines/*/; do (cd "$d" && shasum -a 256 -c SHA256SUMS); done`. GNU tar may warn about macOS `LIBARCHIVE.xattr` headers in the two older archives. The warnings are harmless; the checksums decide.
3. Clone into the `-main` names. The backend URL is in `package.json:22-24`. The frontend and docs URLs are recorded nowhere, only the names `career-companion-frontend` and `career-companion-docs` ([sprint-5 architecture review](../sprint-5/architecture-review.md) :9-11). Confirm them on GitHub. On Windows add `-c core.autocrlf=false` so the checksums match.
4. Find each zip's base commit, then branch and layer. Backend and frontend: Sprint 6 → after BYO AI → final. Docs: after BYO AI → final (the first baseline has no docs repo).

```bash
cd "$ROOT"
git clone https://github.com/NitinSirsath/career-companion-backend.git career-companion-backend-main
git clone <frontend URL> career-companion-frontend-main && git clone <docs URL> career-companion-docs-main
find_base() {   # $1 repo folder, $2 oldest snapshot of that repo
  repo="$ROOT/$1"; snap=$(mktemp -d); start=$(git -C "$repo" symbolic-ref --short HEAD)
  tar -xzf "$ROOT/$2/$1.tar.gz" -C "$snap"
  for c in $(git -C "$repo" rev-list --all --max-count=60); do
    git -C "$repo" checkout -q --detach "$c"
    echo "$(diff -rq -x .git "$snap/$1" "$repo" | wc -l) $(git -C "$repo" log -1 --format='%h %cs %s')"
  done | sort -n | head -5
  git -C "$repo" checkout -q "$start"; rm -rf "$snap"
}
find_base career-companion-backend-main _baselines/mcp-feature-2026-10-02
find_base career-companion-frontend-main _baselines/mcp-feature-2026-10-02
find_base career-companion-docs-main _baselines/mcp-feature-2026-10-02-after-byo-ai
SCR=$(mktemp -d)
layer() {   # $1 repo folder, $2 snapshot folder, $3 commit message
  rm -rf "$SCR/x"; mkdir -p "$SCR/x"; tar -xzf "$ROOT/$2/$1.tar.gz" -C "$SCR/x"
  rsync -a --delete --exclude='.git/' --exclude='node_modules/' --exclude='dist/' --include='.env.example' \
    --exclude='.env' --exclude='.env.*' --exclude='*.tsbuildinfo' --exclude='.DS_Store' "$SCR/x/$1/" "$ROOT/$1/"
  (cd "$ROOT" && shasum -a 256 -c --quiet "$2/$1.files.sha256") || return 1
  (cd "$ROOT/$1" && git add -A && git ls-files | sed "s#^#$1/#" | LC_ALL=C sort | diff - "$ROOT/$2/$1.filelist") || return 1
  git -C "$ROOT/$1" diff --cached --name-only | grep -E '(^|/)\.env' | grep -v '\.env\.example$' && return 1
  git -C "$ROOT/$1" commit -q -m "$3" && git -C "$ROOT/$1" rev-parse HEAD
}
git -C "$ROOT/career-companion-backend-main" switch -c <branch> <backend base>
layer career-companion-backend-main _baselines/mcp-feature-2026-10-02 "Sprint 6 (local snapshot, 2026-10-02 03:51 IST)"
layer career-companion-backend-main _baselines/mcp-feature-2026-10-02-after-byo-ai "BYO AI, ADR-0001 (local snapshot, 2026-10-02 08:00 IST)"
layer career-companion-backend-main _baselines/pre-migration-<date> "MCP, ADR-0002 and final local state (MV-01 snapshot)"
# Frontend: the same three layers from its own base. Docs: the last two layers only.
git -C "$ROOT/career-companion-backend-main" push -u origin <branch>      # owner's account; repeat per repo
```

The first `find_base` column counts differing files and folders; the lowest is the base candidate. Hashes worth checking: the Sprint 5 tree HEADs `71d234e` / `ef36eb7` / `6e19696`, the 2026-09-26 remote main `b60f456` / `f675abe` / `ea7a475` and the 2026-09-26 inspected branch HEADs `c7e33f5` / `11e468d` / `7b82b2e` (backend / frontend / docs; sprint-6 architecture review :11-13, :17). Whether any zip matches them is unverified. Widen `--max-count` if no count stands out. If `git log --oneline <base>..origin/HEAD` is not empty, `main` moved after the download: stop and decide with the owner before layering. The first layer is not a clean Sprint 6 boundary: S6-R06 was implemented under AI-00, so it lands in the BYO AI commit (PLAT-01).
- **Expect:** each `layer` call prints a commit hash and nothing else (it returns non-zero on a checksum, file-list or env-file failure); 8 commits in all; branches pushed. Merging into `main` is the owner's decision.
- **Record:** three repo URLs, base hash and diff count per repo, whether `main` moved, branch name, the 8 commit hashes, push confirmation.

### MV-04 — Toolchain and dependencies

- **Node** 24.15 or later (verified on v24.20.0), or 22.22.2 or later. Not 23 or 25. The strictest locked range is frontend `jsdom` 30.0.1: `^22.22.2 || ^24.15.0 || >=26.0.0` (PLAT-14, TEST-10). **npm** 11.
- **PostgreSQL 15**, native or Docker. With Compose v2 run `docker compose up -d` in the backend; `npm run db:up` calls the old `docker-compose` (`package.json:14`). Compose creates only `career_companion_db`. The role needs CREATE DATABASE, and CREATE SCHEMA for pg-boss's `pgboss` schema.
- **Google Chrome** for the smoke. Keep the **firewall on**: the backend listens on all interfaces (`src/index.ts:115`), and Docker publishes port 5432 (PLAT-06).

```bash
node --version; npm --version; psql --version; pg_isready -h localhost -p 5432
for r in career-companion-backend-main career-companion-frontend-main; do (cd "$ROOT/$r" && npm ci && npm ls --depth=0 >/dev/null &&
  node -e "const p=require('./package-lock.json').packages;for(const[k,v]of Object.entries(p))if(/node_modules\/(jsdom|pg-boss|vitest|puppeteer|undici|eslint|vite)$/.test(k))console.log(k.slice(13),v.version,v.engines?.node)"); done
```

- **Expect:** `npm ci` exits 0 with no `EBADENGINE` warning; `npm ls` is clean; the printed ranges admit your Node. Use `npm ci`, not `npm install` or `npm update`: `npm ci` installs exactly the lockfile and fails on a mismatch. `npm update` or a regenerated lockfile could move caret-ranged packages, such as `@google/genai` (`package.json:36` `^2.22.0`; lockfile 2.22.0, `package-lock.json:247-248`; PLAT-15, pinned in AI-19). Puppeteer downloads a Chrome at install, which needs network.
- **Record:** OS, Node, npm, PostgreSQL (native or Docker) and Chrome versions; any `EBADENGINE` line.

### MV-05 — Environment files and Google OAuth

Generate each secret or key with `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`. Do not copy `.env.example` as it is (PLAT-04).

**Backend `.env` (development), by hand:**
- `DATABASE_URL` written literally, for example `postgresql://<user>:<password>@localhost:5432/career_companion_db?schema=public`. No `${...}`.
- `SESSION_SECRET`; `OAUTH_STATE_COOKIE_SECRET`, non-empty. `FRONTEND_URL=http://localhost:5173`. `PORT` unset or 3000.
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI=http://localhost:5173/api/auth/callback`; `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REDIRECT_URI=http://localhost:5173/api/gmail/callback`.
- `GMAIL_TOKEN_ENCRYPTION_KEY` and `AI_CREDENTIAL_ENCRYPTION_KEY`: new, 64 hex each, different from each other. New keys are fine because no real database moves (MV-02). Back them up: losing them means reconnecting Gmail and re-entering the AI key.
- `RELEVANCE_CONFIDENCE_THRESHOLD=0.7`; `AI_USER_DAILY_CALL_LIMIT=500` (MV-13 lowers it for a while).
- Optional: `DISCORD_WEBHOOK_URL` with `DISCORD_USER_ID` set to the owner's Career Companion `users.id` (after the first sign-in), not a Discord ID (`src/jobs/notificationJob.ts:52`).
- `ENABLE_DEV_AUTH` off unless you need header login over curl. When on, anyone on the network can act as any known user email (PLAT-06). `.env.example` turns it on.
- Leave out: `NODE_ENV` (`test` stops the workers, `src/index.ts:103`); `TRUST_PROXY_HOPS`; `MCP_ALLOWED_HOSTS`, `MCP_ALLOWED_ORIGINS`, `MCP_DAILY_SUBMISSION_LIMIT` (backend tests also load backend `.env`, and the smoke likely does through Prisma's own loader, TEST-05; MV-14 adds and removes them). Never set `GOOGLE_GENAI_USE_VERTEXAI` or `GOOGLE_GENAI_USE_ENTERPRISE` (AIB-11, until AI-19). Docker only: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_PORT`.

**Backend `.env.test`** (fixture values only; the same 18 names as on the old laptop): `DATABASE_URL` and `TEST_DATABASE_URL`, identical, `<PGURL>/career_companion_sprint6_test?schema=public` (host localhost, 127.0.0.1 or [::1]; no other query parameter); `NODE_ENV=test`; `ENABLE_DEV_AUTH=true`; `SESSION_SECRET`; `OAUTH_STATE_COOKIE_SECRET`; `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REDIRECT_URI`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` (fakes); `GMAIL_TOKEN_ENCRYPTION_KEY` and `AI_CREDENTIAL_ENCRYPTION_KEY` (fixture 64 hex, different from each other); `AI_USER_DAILY_CALL_LIMIT=100`; `DISCORD_WEBHOOK_URL=` and `DISCORD_USER_ID=` (empty); `FRONTEND_URL=http://localhost:5173`.

**Backend `.env.smoke.test`:** the same names, both URLs pointing at `career_companion_sprint6_smoke_test`. Only the smoke-lane migrate reads it (MV-10). The smoke itself reads `.env.test` and needs its two 64-hex keys (`scripts/smoke-stabilization.mjs:41`, `:53-56`). **Frontend:** no environment variables.

**Google OAuth (owner, Google Cloud console; PLAT-19):** register both redirect URIs above; scopes `openid email profile` and `https://www.googleapis.com/auth/gmail.readonly`; add the owner as a test user; note the publishing status. In Testing status Google expires refresh tokens after 7 days (Google's policy, not tested here), so Gmail turns REVOKED and must be reconnected.
- **Check:** `git -C "$ROOT/career-companion-backend-main" status --short` lists no env file.
- **Record:** env names created (never values), whether login and Gmail share one OAuth client, the publishing status.

### MV-06 — Static checks and builds

```bash
cd "$ROOT/career-companion-backend-main" && npm run db:generate && npm run typecheck && npm run lint && npm run build
cd "$ROOT/career-companion-frontend-main" && diff -r ../career-companion-backend-main/src/contracts src/contracts
npm run sync-contracts && npm run typecheck && npm run lint && npm run build
```

- **Expect:** backend typecheck 0 errors; lint 0 errors and 11 warnings (all "Unused eslint-disable directive" in `src/tests/gmail.test.ts` and `src/tests/gmailSync.test.ts`); build writes `dist/`. Frontend: `diff` prints nothing; sync ends with `(0 file(s) changed)` (`scripts/sync-contracts.mjs:34`); lint 0 errors and 23 warnings; build writes `dist/index.html`. `sync-contracts` overwrites any file that differs, so a non-zero count is a finding (MCPF-16).
- **Record:** each result and both warning counts.

### MV-07 — Databases and guarded migration

```bash
mkdb career_companion_sprint6_test; mkdb career_companion_sprint6_smoke_test; mkdb career_companion_db   # skip the last with Docker
cd "$ROOT/career-companion-backend-main"
TEST_DATABASE_URL="$PGURL/career_companion_sprint6_test?schema=public" node scripts/guarded-migrate.cjs
npx prisma migrate deploy && npx prisma migrate status          # dev database, from .env
psql "$PGURL/career_companion_sprint6_test" -tAc 'SELECT count(*) FROM _prisma_migrations WHERE finished_at IS NOT NULL'
```

The exported `TEST_DATABASE_URL` must equal the `.env.test` value exactly (`scripts/guarded-migrate.cjs:6-16`). The guard refuses non-test names, so the dev database uses plain `migrate deploy`. Never use `npm run db:migrate` (interactive `prisma migrate dev`) or `npm run db:reset` (`package.json:16`, `:19`). Optional: `npm run db:seed` creates `dev@career-companion.local`, usable only with dev auth over curl (not run in the audit; unverified). The smoke lane is migrated in MV-10.
- **Expect:** the guarded run applies 15 migrations, the last `20261002150000_automation_submissions`; `migrate status` is up to date; `validate` passes; the count prints 15. `20260912_add_googleid_to_user_and_gmail_connection` sorts after `20260912151032_…`. Never rename it (PLAT-20).
- **Record:** database names and migration counts.

### MV-08 — Test suites

```bash
cd "$ROOT/career-companion-backend-main" && npm test      # expect 621 passed in 43 files
cd "$ROOT/career-companion-frontend-main" && npm test     # expect 154 passed in 15 files
```

- Backend files run one at a time (`vitest.config.ts:6`) and delete rows across tables. Every file needs `.env.test`, even a pure unit file (`src/tests/setup.ts:6-8`). Frontend tests are offline.
- If a file fails, re-run it alone before debugging: `npx vitest run src/tests/<name>.test.ts` (frontend `.test.tsx`). Run whole files, not `-t` filters: some tests depend on earlier ones in the same file (TEST-09).
- Timing-sensitive (TEST-14): `email-worker-reliability` (real worker, polls up to 30 s), a 500 ms window in `ai-user-limits`, daily-limit tests across 00:00 UTC (05:30 IST), a frontend 2.6 s sleep under the 5 s timeout. Do not run across 00:00 UTC.

- **Record:** counts, duration, any single-file re-run and its result.

### MV-09 — Migration lanes

Each script needs two separate, empty `career_companion_*test` databases. It writes its own temporary env file, so `.env.test` is not read. `dist/` must be fresh (MV-06).

```bash
cd "$ROOT/career-companion-backend-main"
for l in s6 ai mcp; do mkdb career_companion_${l}_fresh_test; mkdb career_companion_${l}_upgrade_test; done
lane() { FRESH_DATABASE_URL="$PGURL/career_companion_$1_fresh_test?schema=public" \
  UPGRADE_DATABASE_URL="$PGURL/career_companion_$1_upgrade_test?schema=public" node "scripts/$2"; }
lane s6 verify-migration-preservation.cjs; lane ai verify-ai-migration.cjs; lane mcp verify-mcp-migration.cjs
for l in s6 ai mcp; do rmdb career_companion_${l}_fresh_test; rmdb career_companion_${l}_upgrade_test; done
```

- **Expect:** JSON events `migration_preservation_verified`, `ai_migration_preservation_verified` and `mcp_migration_preservation_verified`, each with exit 0. A failure prints `FAIL:` and exits 1. The S6 script may hang after `FAIL:` because it never calls `process.exit` (`scripts/verify-migration-preservation.cjs:178-182`; TEST-11, fixed by [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md)): press Ctrl+C and record it. Recreate a pair before re-running its lane.
- **Record:** each result and the row counts in the JSON.

### MV-10 — Real-worker smoke

Exclusive lane: nothing else may use the smoke database, and `.env.test` must not point at it. Needs both `dist/` builds, backend `.env.test` and Chrome.

```bash
SMOKE="$PGURL/career_companion_sprint6_smoke_test?schema=public"
cd "$ROOT/career-companion-backend-main" && TEST_ENV_FILE=.env.smoke.test TEST_DATABASE_URL="$SMOKE" node scripts/guarded-migrate.cjs
cd "$ROOT/career-companion-frontend-main" && SMOKE_DATABASE_URL="$SMOKE" node scripts/smoke-stabilization.mjs 2>&1 | tee "$LOG/mv10-smoke.log"
# Not macOS: put CHROME_BIN=<path to Chrome> before node (default path: smoke-stabilization.mjs:389, :1014)
```

- **Expect:** the last line starts `PASS: real API/PostgreSQL/pg-boss`, names the Sprint 5, BYO AI, MCP and Sprint 6 scenarios, and ends with `residue 0; outbound blocked.` (`scripts/smoke-stabilization.mjs:1052`). The `smoke_teardown` JSON shows `"residual":{"users":0,"jobs":0,"budgets":0,"mcp":0}`. A non-empty lane is refused; reset it with `rmdb`, `mkdb` and the migrate line. Optional, recorded only if run: `SMOKE_INJECT=scenario-failure`. `http-drain` and `worker-drain` quarantine the lane, so recreate it afterwards.
- **Record:** the PASS line, the teardown JSON, run time.

### MV-11 — Start the dev stack

Order matters until [S7-04](../sprint-7/S7-04-worker-startup-and-readiness.md): workers register once at boot and never retry (PLAT-07).

```bash
pg_isready -h localhost -p 5432                                                      # 1. PostgreSQL first
cd "$ROOT/career-companion-backend-main" && npm run dev 2>&1 | tee "$LOG/backend.log"   # 2. terminal A
cd "$ROOT/career-companion-frontend-main" && npm run dev                               # 3. terminal B, :5173
grep -E 'worker_registered|_worker_start_failed|\[Config\]|\[Auth\]' "$LOG/backend.log"
```

- **Expect:** one `worker_registered` line for `email-processing-job` (today only the email worker logs it, `src/jobs/emailProcessingJob.ts:204-206`); `Backend server is running on port 3000`; no `gmail_worker_start_failed`, `email_worker_start_failed` or `notification_worker_start_failed`; no `[Config]` warning (`src/index.ts:16-27`) and no `[Auth]` warning about missing Google settings (`src/routes/auth.ts:23-29`). `/health` says `ok` even when workers failed (`src/index.ts:82-84`), so it proves nothing here. On any `*_worker_start_failed`: check PostgreSQL, then restart the backend.
- **Record:** the lines seen and any restart.

### MV-12 — Live Gmail sanity check (non-gating)

Checks sign-in, Gmail connect and live token refresh (PLAT-19). It is **not** S5-02 or Sprint 6 entry-gate evidence (OD-01). Real mail lands only in the local dev database. A failure needs a note in the report, not a Phase 0 fix, unless MV-15 shows the move caused it.

1. At http://localhost:5173 sign in with Google. **Gmail Sync** → **Connect Gmail**; you return to `/gmail` connected.
2. Sync Settings → **30 days**: interim until [S7-02](../sprint-7/S7-02-no-silent-mail-loss-after-gaps.md), because a gap longer than the lookback skips mail silently (S56-10). Then **Sync Now**. Emails wait for AI until MV-13; that is expected.
3. Note the stored token's fingerprint (a hash of the ciphertext, no secret): `psql "$PGURL/career_companion_db" -tAc 'SELECT md5("accessToken"), status, "lastSyncedAt" FROM gmail_connections'`.
4. More than one hour later (Google access tokens usually last about an hour; not verified here), **Sync Now** again and repeat the query.

- **Expect:** both syncs finish; status `CONNECTED`; the md5 changed, so the refreshed token was saved (`withGmail`, `src/services/gmailClient.ts:41-51`). `REVOKED` or `GMAIL_AUTH_FAILED` means the refresh failed; check the publishing status (MV-05).
- **Record:** times, results, md5 changed yes or no.

### MV-13 — Verify BYO AI with a real Gemini key

A local check, not certification. Gemini stays `hidden`; hidden providers are offered outside production (`src/services/ai/access.ts:64-65`). No catalog or disclosure change; [AI-15](../byo-ai/AI-15-certify-gemini-first.md) certifies later. Real email text goes to Google under the owner's key.

1. First read Google's current Gemini API terms (`https://ai.google.dev/gemini-api/terms`, the catalog's data-use link). The app's draft text says that on the free tier Google may use what is sent to improve its products, and people may review it (`src/contracts/aiCatalog.ts:90-91`, version `gemini-draft-2026-10`, not owner-approved, OD-09). Choose a free or billed key knowingly.
2. Bound the cost: stop the backend; set `AI_USER_DAILY_CALL_LIMIT=12` in `.env` (per user per UTC day; `src/services/ai/usage.ts:24-30`, `userDailyCallLimit`). The sample test uses 2 calls and each email up to 2. Key checks on save and **Check again** are counted apart (`MAX_DAILY_VERIFICATIONS`, `usage.ts:15`). Check that `env | grep GOOGLE_GENAI` prints nothing; start the backend.
3. **AI Provider** → Google Gemini → paste a wrong key of at least 8 printable characters (a shorter one fails the local format check, `src/services/ai/settings.ts:66`), tick the consent box, **Save and verify**. Expect a rejected-key message and nothing saved. If it is saved as not verified instead, record it: the Gemini error mapping is not yet confirmed with a real key (`src/services/ai/providers/gemini.ts:72-93`, `classifyGeminiError`).
4. Paste the real key, keep the consent box ticked, **Save and verify**. Expect the key verified and saved. Note whether the browser offers to save the key as a password; decline (AIF-18).
5. Saving re-offers waiting emails at once, up to 100 (`src/services/ai/settings.ts:235` calls `reofferPendingEmails`, `src/services/gmailSync.ts:31-41`). So the emails and the sample test share the 12 calls.
6. **Try a sample email** → a result is shown. If it is refused by the safety limit because the emails used the calls first, record that, set the limit to 14, restart and run only the sample test again. A job that hits the limit is acknowledged and its email waits (`src/jobs/emailProcessingJob.ts:114-140`). It is offered again only on save, a verified check or a sync (`src/services/ai/settings.ts:235`, `:280`; `src/services/gmailSync.ts:240`).
7. About 3–5 emails get AI results; the rest wait at the safety limit. Spot-check that the results make sense. **Check again** → the key is still accepted; the status may show the safety limit once the calls are used (`src/services/ai/access.ts:104`).
8. **Remove** → **Remove key**. Expect "not set up"; waiting emails stay pending. Set `AI_USER_DAILY_CALL_LIMIT` back to the daily-use value (default 500) and restart.

Do **not** run `npm run ai:eval` yet: it scores provider rate limits as quality failures (AIB-04, AI-15).
- **Record:** free or billed tier (never the key), each step's result and error code, emails processed, the password-manager prompt per browser, anything wrong (→ a ticket, usually AI-19, AI-20 or AI-15).

### MV-14 — Verify MCP with real clients

The first real-client run against the real `/mcp` (MCPB-01, MCPB-04, MCPF-06). MCP-09 part A (MCP tests, MCP lane, smoke) is re-run in MV-08..MV-10. This step covers MCP-09 part B steps 1–2 and 6–7 from the [execution report](../mcp-feature/execution-report.md). Steps 3–5 need [MCP-08](../mcp-feature/MCP-08-automation-changes.md) and run in the closeout (order 6). Use a **disposable database**: applications created by automation cannot be deleted in the app (MCPF-02, OD-10).

1. Stop the backend. Run `mkdb career_companion_mcp_scratch`. In `.env`, comment out the dev `DATABASE_URL`, point it at the scratch database, and add `MCP_DAILY_SUBMISSION_LIMIT=10` (MCPF-04). Then `npx prisma migrate deploy` and `npm run dev 2>&1 | tee "$LOG/mv14-backend.log"`.
2. Sign in at http://localhost:5173; the user exists only in the scratch database. **Automation** → name `mv14`, expiry **30 days**, **Create token**; copy it once.
3. SDK client control, in a second terminal from the backend folder:

   ```bash
   read -rs CC_MCP_TOKEN && export CC_MCP_TOKEN                  # paste; not echoed, not in history
   node scripts/mcp-client-check.cjs http://localhost:3000/mcp
   node scripts/mcp-client-check.cjs http://localhost:5173/mcp   # Vite proxy, never exercised before
   ```

   Expect for both: `{"connected":true,"protocolVersion":"2026-07-28","tools":["record_application_submission"]}`.
4. Antigravity: in `~/.gemini/config/mcp_config.json` put `{ "mcpServers": { "career-companion": { "serverUrl": "http://localhost:3000/mcp", "headers": { "Authorization": "Bearer <token>" } } } }`. Run `chmod 600` on it (on Windows, restrict it to your account; path and command unverified). Allow `mcp(career-companion/record_application_submission)`. Never use a workspace `.agents/mcp_config.json`.
5. Ask the agent to call `record_application_submission` with the `arguments` of the Fabrikam entry in `career-companion-frontend-main/scripts/fixtures/mcp-daily-applications.json` (synthetic). Expect `created`. Ask again: expect `already_recorded` with the same IDs.
6. Read the log: `grep '"event":"mcp_request"' "$LOG/mv14-backend.log"` (IDs and outcomes only, `src/mcp/callLog.ts:38-41`).
7. If Antigravity fails, the SDK client already separates server from client faults. To also send the fixture, create the seed applications Northwind Traders / Data Engineer and Contoso / UX Designer, then add `--send-fixture ../career-companion-frontend-main/scripts/fixtures/mcp-daily-applications.json` to the `:3000` command. If step 5 already sent Fabrikam, it now returns `already_recorded`, not the fixture's expected `created`.

**Record (MCP-10 needs these):**
- **Origin.** No `origin_not_allowed` line while `MCP_ALLOWED_ORIGINS` is empty means no Origin was sent: any present Origin is refused by default (`src/mcp/config.ts:6-7`). A hostname in `rejectedOriginHost`: add it to `MCP_ALLOWED_ORIGINS`, restart, retry, record. `"rejectedOriginHost":"unparseable"`: **stop**. No setting can fix it (`src/mcp/router.ts:52-53`, `:106-110`); record it for OD-11 and [MCP-10](../mcp-feature/MCP-10-post-verification-corrections.md) item 8.
- **Protocol.** The SDK client's `protocolVersion`. For Antigravity, the first `rpcMethod`: `initialize` means the 2025 era; no `initialize` (for example `server/discover`, which is optional in 2026-07-28) points to the 2026-07-28 era. The exact version is not logged.
- **Listen stream.** A `"rpcMethod":"subscriptions/listen"` line is written only when the response finishes (`src/mcp/router.ts:86-91`); if the client disconnects first, no line is written (MCPB-08). Quit Antigravity and look again. A long `durationMs` means a stream stayed open. No line means "not seen", not "none".
- Whether Antigravity accepted the `http://localhost` `serverUrl`, listed the tool (schema accepted) and called it; the exact text of any client error; the Antigravity version.

- **Clean up:** remove the server entry from `mcp_config.json`; `unset CC_MCP_TOKEN`; stop the backend; delete every `MCP_*` line from `.env` and restore the dev `DATABASE_URL`; `rmdb career_companion_mcp_scratch` (the token goes with it); start the backend again and repeat the MV-11 checks.

### MV-15 — Fix only what the move broke

- A failure counts as "broken by the move" only when the check passed on the old laptop (per the execution reports) and fails here because of environment, paths, versions, missing files or the git restore. Prove it: re-run that one check and name the cause.
- Prefer fixing the environment (env file, database, Node version, Chrome path, folder name) over code. If code must change: the smallest change, one focused commit per fix, the failing output in the report, then re-run the failed check and that repo's full suite.
- Everything else becomes a ticket. Add it to an existing ticket when it fits (the S6 lane hang is S6-C01; Gemini behaviour AI-19 or AI-15; client issues MCP-10). Otherwise use a new local ID from an unused range ([roadmap §10](../README.md#10-planning-rules-from-now-on)), reviewed by the owner before work starts.

### MV-16 — Gate: record results and decisions

- Copy the template below into `docs/planning/migration-verification/verification-report.md` and fill every row: pass, fail or skipped with a reason.
- Record the owner decisions with dates: OD-01 (Sprint 6 entry gate), OD-02 (system of record), OD-03 (git history shape, decided before MV-03), and the OD-13 answer from MV-02 ([roadmap §6](../README.md#6-owner-decisions)).
- The gate passes when MV-01 (checksums verified on the new PC), MV-03 (layer commits made and pushed) and MV-06..MV-14 pass, or each failure has an owner-accepted note in the report. MV-12 is non-gating. Only then does the closeout start; Linear sync, if any, comes after ticket review.
- The closeout cannot start without MV-03, even with an accepted note: every closeout ticket needs git (focused commits, [S6-C01](../sprint-6/closeout/S6-C01-make-done-claims-provable.md) restore checks) and [S7-01](../sprint-7/S7-01-ci-and-toolchain-pins.md) needs a pushed remote.
- No acceptance or DoD checkbox anywhere is ticked before the gate passes. S6-C02 ticks them later, linking the report.

```markdown
# Phase 0 verification report — YYYY-MM-DD

Machine: <OS> · Node <x> · npm <x> · PostgreSQL <x> (native | Docker) · Chrome <x> · Antigravity <x>
Repos: backend <URL> · frontend <URL> · docs <URL> · branch <name>

| Step | Check | Expected | Result | Evidence | Date |
| --- | --- | --- | --- | --- | --- |
| MV-01 | Final snapshot; checksums on the new PC | 6 × OK on both machines; no excluded path | | folder, counts | |
| MV-02 | Sprint 5 tree; env files; databases; deployment and O8 | answers recorded | | | |
| MV-03 | Base commits; 8 layer commits; push | 8 hashes; branches pushed (no note can waive it) | | | |
| MV-04 | Toolchain; `npm ci` | Node ≥24.15 or ≥22.22.2; no EBADENGINE | | | |
| MV-05 | Env files; OAuth publishing status | names only | | | |
| MV-06 | Backend generate, typecheck, lint, build | 0 errors / 11 warnings | | | |
| MV-06 | Frontend contracts, sync, typecheck, lint, build | 0 changed; 0 errors / 23 warnings | | | |
| MV-07 | Guarded migrate; dev migrate | 15 migrations | | | |
| MV-08 | Backend / frontend suites | 621 in 43 / 154 in 15 | | | |
| MV-09 | S6, BYO AI, MCP lanes | 3 × *_verified | | | |
| MV-10 | Smoke | PASS, residue 0 | | | |
| MV-11 | Dev stack | worker_registered; no start failure | | | |
| MV-12 | Live Gmail (non-gating) | syncs OK; token refreshed and saved | | | |
| MV-13 | BYO AI with Gemini | reject, save, sample, 3–5 emails, check, remove | | | |
| MV-14 | SDK client :3000 and :5173; Antigravity | Origin, protocol, listen, schema recorded | | | |

## Fixes made in Phase 0 (MV-15)
| Failing check | Move-related cause | Fix and commit | Re-run result |

## Tickets raised
| Defect | Evidence | Ticket (existing or new ID) |

## Owner decisions
| ID | Decision | Date |

## Gate
Passed on YYYY-MM-DD, or not passed: <reason>. Accepted failures: <step, note, owner, date>. MV-03 cannot be an accepted failure.
```

## What breaks quietly if missed

| If missed | What happens | Where (checked 2026-10-02) | Step |
| --- | --- | --- | --- |
| `DATABASE_URL` with `${...}` | Plain dotenv keeps the text. Today Prisma's own loader expands `.env` first, because the client is imported before `dotenv.config()`. This depends on load order and on generating the client while `.env` exists (PLAT-04). | `src/index.ts:9-11`, `:49-51`; `src/services/queue.ts:8` | MV-05 |
| Empty `OAUTH_STATE_COOKIE_SECRET` | Sign-in and Gmail connect return 500: `??` keeps the empty string, and signed cookies need a secret. | `src/index.ts:45`; signed `res.cookie` in `src/routes/auth.ts:65-71` and `src/routes/gmail.ts:140-146` | MV-05 |
| `FRONTEND_URL` unset | The Gmail callback redirects to `localhost:3000/gmail`, the backend (PLAT-12). | `src/routes/gmail.ts:58-60` (`getFrontendUrl`) | MV-05 |
| `NODE_ENV=test` in dev `.env` | Workers never start; no error. | `src/index.ts:103` | MV-05 |
| PostgreSQL started after the backend | All workers fail once and never retry; `/health` stays 200; sync shows SYNCING, then FAILED with no cause (PLAT-07, S7-04). | `src/index.ts:103-113`, `:82-84` | MV-11 |
| `PORT` not 3000 | The Vite proxy for `/api` and `/mcp` breaks. | `vite.config.ts:19-29` | MV-05 |
| Wrong or missing keys with real data | Gmail decryption throws, sync records `SYNC_FAILED`; AI shows `KEY_UNREADABLE` and emails wait. | `src/services/gmailClient.ts:26-29`; `src/services/gmailSync.ts:271`; `src/services/ai/access.ts:166-171` | MV-02, MV-05 |
| `DISCORD_USER_ID` empty or not the owner's user ID | Notifications are skipped; only `notification_skipped` is logged. | `src/jobs/notificationJob.ts:52-60` | MV-05 |
| MCP through a non-localhost name, or a client Origin | 403 `host_not_allowed` until `MCP_ALLOWED_HOSTS` is set; 403 `origin_not_allowed` until `MCP_ALLOWED_ORIGINS` is set; `null` or unparseable cannot be allowed (OD-11). | `src/mcp/router.ts:99-112`; `src/mcp/config.ts:20-26` | MV-14 |
| `MCP_*` or `TRUST_PROXY_HOPS` in backend `.env` while testing | Backend tests add `.env` names missing from `.env.test` (the smoke likely too, through Prisma's loader). A host list without `127.0.0.1` gives MCP test 403s; a low submission limit gives cap errors; an invalid `TRUST_PROXY_HOPS` fails at import (TEST-05, S6-C01). | `src/tests/setup.ts:6`; `src/index.ts:11`, `:32`, `:37`; `src/utils/config.ts:4-10` | MV-05, MV-14 |
| `.env.test` missing | Every backend test file stops at the SAFETY GUARD; the default guarded migrate stops before migrating; the smoke stops at its key check (TEST-02). | `src/tests/setup.ts:6-8`; `scripts/guarded-migrate.cjs:8-9`; `scripts/smoke-stabilization.mjs:41`, `:53-56` | MV-05 |
| Backend `dist/` not built | The guarded migrate, the lanes and the smoke cannot load the guard or the app. | `scripts/guarded-migrate.cjs:10`; `scripts/verify-*.cjs:15-16`; `smoke-stabilization.mjs:26-31` | MV-06 |
| Chrome not at the macOS path, no `CHROME_BIN` | The smoke cannot launch a browser (TEST-10). | `scripts/smoke-stabilization.mjs:389`, `:1014` | MV-10 |
| Google OAuth app in Testing status | After 7 days the refresh fails with `invalid_grant`, Gmail turns REVOKED and must be reconnected (Google's policy; not tested). | `src/services/gmailClient.ts:10-13`, `:34-38` | MV-05, MV-12 |
| 1-day lookback with a longer gap | Older inbox mail is never fetched and nothing reports it (S56-10, S7-02). | `src/services/gmailSync.ts:84-93`, `:163` | MV-12 |
| `npm install` or `npm update` instead of `npm ci` | `npm ci` installs exactly the lockfile and fails on a mismatch; `npm install` does not fail and may rewrite the lockfile. `npm update` or a regenerated lockfile could move caret-ranged packages, such as `@google/genai` (PLAT-15, pinned in AI-19). | `package.json:36`; `package-lock.json:247-248` | MV-04 |
| `.env.test` pointed at the smoke database | The guard accepts `*smoke_test` names and the suite deletes rows; exclusivity is procedural only. | `src/utils/testDatabase.ts:9`; `smoke-stabilization.mjs:208` (lock) | MV-08, MV-10 |
| `GOOGLE_GENAI_USE_VERTEXAI` or `_ENTERPRISE` set | The Gemini SDK switches to Vertex mode and every Gemini call fails; exact status unverified (AIB-11, AI-19). | `src/services/ai/providers/gemini.ts:95-100` (`createGeminiClient`) | MV-13 |
| Renaming `20260912_add_googleid…` | `_prisma_migrations` no longer matches on existing databases (PLAT-20, not verified by a run). | `prisma/migrations/` | MV-07 |
