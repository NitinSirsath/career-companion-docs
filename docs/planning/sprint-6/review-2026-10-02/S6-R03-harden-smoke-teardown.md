# S6-R03 — Harden smoke teardown drain and quarantine

| Field | Value |
| --- | --- |
| Status | **Fixed — verified locally 2026-10-02** (local ticket) |
| Severity / priority | Medium (+ Low items) / fix before Sprint 6 acceptance |
| Source | [Review 2026-10-02](README.md) M3, L6; S6-05 §6, §14, §18; runbook §7 |
| Repository | career-companion-frontend (`scripts/smoke-stabilization.mjs`) |

## Problem
1. **Drain counts sockets, not handlers.** Lines ~224-227: `inFlight.add(res); res.on('close', () => inFlight.delete(res))`, and teardown closes the browser first. `res` emits `close` when the client socket drops, even if the Express handler is still running. A browser-originated PATCH or POST still executing can therefore count as drained, and fixture deletion can race it. The passing probe uses Node `fetch`, whose socket stays open, so it does not exercise this case.
2. **Quarantine releases too early.** On a failed drain, teardown restores `globalThis.fetch` and releases the advisory lock while the hung handler is still alive. Runbook §7 steps 2 and 4 require keeping isolation until no future writes are possible.
3. **Untested teardown cases.** Only a hung HTTP handler is injected. There is no worker-fails-to-drain mode and no teardown-after-scenario-failure run. S6-05 §14 asks for successful and failed runs.
4. **Unrecorded checks.** Browser requests to non-local origins (Google Fonts) are aborted and logged but not explicitly allowlisted, and other hosts don't fail the run (S6-05 §18). Residual users, jobs and budgets after cleanup are not counted by the harness; the "0 residue" result came from a manual `psql` check.

## Fix
- Track handler completion in a harness middleware registered before the routes. Increment on entry; decrement in `finally` or on response `finish`, independent of the socket.
- In quarantine, keep outbound blocking and the lock until the process exits.
- Add `SMOKE_INJECT_WORKER_DRAIN_FAILURE` (never-released gate) and a forced scenario-failure mode, and verify ordering and quarantine for both.
- Allowlist expected browser asset hosts explicitly and fail on anything else. Record residual counts in the teardown report.

## Acceptance criteria
- [ ] With a browser-originated write held in its handler and the browser closed, cleanup waits for (or quarantines on) that handler.
- [ ] Worker-drain failure and scenario failure both quarantine or clean correctly, with recorded diagnostics.
- [ ] Teardown output includes residual users, jobs and budgets. Unexpected browser hosts fail the run.

## Verification
Run the smoke normally, then each injected-failure mode on a fresh exclusive `career_companion_*smoke_test` lane, resetting the lane after each quarantine.

## Resolution — 2026-10-02

Fixed in `scripts/smoke-stabilization.mjs`:
- The server wraps `res.end` per request, so a request stays in flight until its handler responds, regardless of socket close.
- The drain probe adds a browser-originated slow write (it finishes last) to the Node-originated write and the blocked worker.
- Quarantine returns early, keeping outbound blocking and the lane lock until the process exits.
- `SMOKE_INJECT=scenario-failure|http-drain|worker-drain` replaces the old single mode.
- Non-local browser hosts other than `fonts.googleapis.com` and `fonts.gstatic.com` fail the run.
- The teardown report records residual users, jobs and budgets.

Verification:
- **Normal run:** pass. 2 writes plus the active worker finished before cleanup; residual 0/0/0.
- **scenario-failure:** pass. Same ordering, 0 residue.
- **http-drain:** quarantined (`httpDrained:false`), 2 fixture users retained.
- **worker-drain:** quarantined (`workersDrained:false`), 2 users retained.
- **Lane reset:** the lane was reset and migrated through the guard after each quarantine.
- **Mutation check:** restoring the old socket-close tracking makes `scenario-failure` fail with `httpWritesFinishedBeforeCleanup:false`; file restored byte-for-byte.
