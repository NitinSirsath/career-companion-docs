# MCP feature — engineering baseline (before MCP-01)

The repositories are not git repositories, and none were created. This baseline is the review and rollback point for the MCP feature, recorded with a checksum manifest and a tar snapshot (owner decision, 2026-10-02). The repositories were not modified by this step.

**Current baseline: taken after BYO AI landed.** The first snapshot (`_baselines/mcp-feature-2026-10-02/`, taken 2026-10-02 03:51 IST) predates the BYO AI implementation. Do not use it to list or roll back MCP changes: it would report BYO AI work as MCP changes, and a rollback to it would delete BYO AI. It is kept only as history.

## Snapshot

| Field | Value |
| --- | --- |
| Taken | 2026-10-02T02:30:42Z (2026-10-02 08:00:42 IST) |
| Location | `/Users/spurge-rental/Downloads/docs/_baselines/mcp-feature-2026-10-02-after-byo-ai/`. This is outside all three repositories. |
| Backend source | `career-companion-backend-main`: 140 files |
| Frontend source | `career-companion-frontend-main`: 80 files |
| Docs source | `career-companion-docs-main`: 63 files (this file excluded, because it is written after the snapshot) |
| Code state | Sprint 6 plus BYO AI (ADR-0001). Just before the snapshot: backend typecheck and 467 tests pass; frontend typecheck and 118 tests pass |
| Verified | 2026-10-02, all checks below passed |

## Files and SHA-256 checksums

| File | SHA-256 |
| --- | --- |
| `career-companion-backend-main.tar.gz` | `62635224f3ee34256db49988cf815811376e4ed6b6fd844aea7ac6297fa2814b` |
| `career-companion-frontend-main.tar.gz` | `969d629e1a1ba00ca1d164fea20d4465b0af23d3d6e4ce7dee621e22ca619ee3` |
| `career-companion-docs-main.tar.gz` | `77601f1f8fc72ebac0f15370ff04d3b83372c887239dc329dfe40cbcfb91cb97` |
| `career-companion-backend-main.files.sha256` (per-file manifest) | `7bbf3841277472b45062fb79cdb6c6d7d0177e8758b924a06a0c1bc225ed4dcd` |
| `career-companion-frontend-main.files.sha256` (per-file manifest) | `6502891288441b069114dd163f0121faa6148b02d1270e26c8631bef6bde678b` |
| `career-companion-docs-main.files.sha256` (per-file manifest) | `daf2f86d6cb4bdf9743634a6b5dc214a241a9892bc18e5047fc5e55f19a52158` |

The same folder also holds:
- `SHA256SUMS`: the six lines above, in `shasum -c` format;
- `*.filelist`: the sorted list of included paths;
- `EXCLUDES.txt`: the excluded paths;
- `TIMESTAMP`.

## Excluded

| Path or pattern | Why | Present at snapshot time |
| --- | --- | --- |
| `node_modules/` | Dependencies, reinstallable from `package-lock.json` | Yes (backend, frontend) |
| `dist/` | Build output | Yes (backend, frontend) |
| `*.tsbuildinfo` | TypeScript build cache | Yes (frontend `tsconfig.app.tsbuildinfo`) |
| `.env`, `.env.*` | Local environment files | Yes: backend `.env.test`, `.env.smoke.test` |
| `.DS_Store`, `*.log`, `coverage/`, `.vite/` | Metadata, logs, caches | Guard only |
| `*.pem`, `*.key`, `*.p12`, `id_rsa*` | Key material | Not present (guard only) |
| `docs/planning/mcp-feature/baseline.md` | This record, written after the snapshot | Yes (docs) |

**Included on purpose:** `.env.example` in the backend and frontend. It is a committed template; every secret key in it is empty.

A secret-pattern scan of the included files found hits in four BYO AI test files (`src/tests/ai-credentials`, `ai-providers-anthropic`, `ai-providers-openai`, `ai-settings`). All are obvious test sentinels (`sk-test-SENTINEL…`, `sk-ant-test-SENTINEL…`). No real credentials or tokens were found.

## Verification performed

| Check | Result |
| --- | --- |
| Archive and manifest checksums (`shasum -a 256 -c SHA256SUMS`) | 6/6 OK |
| Extracted archives vs their per-file manifests | 0 failures; 140/140, 80/80, 63/63 files |
| Excluded paths found inside any archive | 0 |
| Live repositories vs manifests after the snapshot | 0 failures (all three) |

## How to use it

From `/Users/spurge-rental/Downloads/docs`:

- **Verify the snapshot is intact:**
  `(cd _baselines/mcp-feature-2026-10-02-after-byo-ai && shasum -a 256 -c SHA256SUMS)`
- **List files changed since the baseline:**
  `shasum -a 256 -c _baselines/mcp-feature-2026-10-02-after-byo-ai/career-companion-backend-main.files.sha256 | grep -v ': OK$'` (the same command works for the frontend and docs manifests). New files are those missing from the matching `.filelist`.
- **Before MCP-01:** both commands must show nothing for the backend and frontend. For the docs, only edits made after this snapshot may show.
- **Review a change:** extract the archive to a scratch folder and `diff -ru` against the live folder. Exclude `node_modules` and `dist`.
- **Roll back:** extract the archive over a copy of the folder. Excluded files (`node_modules`, `dist`, `.env.test`, `.env.smoke.test`) are not in the archive and are left as they are.
