# MCP-00 — MCP SDK and client compatibility spike: report

Date: 2026-10-02. Throwaway code only; it was deleted after the run. No Career Companion code was changed.

## Decision

**Use SDK v2:**
- `@modelcontextprotocol/server` 2.2.0
- `@modelcontextprotocol/express` 2.0.1 (peer: Express ^4.18 || ^5)
- `@modelcontextprotocol/node` 2.1.0
- zod v4, Node ≥ 20

`createMcpHandler` serves 2026-07-28 clients and, by default, 2025-era clients through a stateless fallback. No v1 fallback is needed for the clients tested.

## What was tested

Setup:
- Server: an Express 5.2.1 app on Node 24.20.0, shaped like MCP-04:
  - a `/mcp` router mounted before the app's global CORS/JSON middleware;
  - a 405 guard, host allowlist, Origin rejection, `requireBearerAuth`, a 32 KB JSON limit;
  - `toNodeHandler(createMcpHandler(factory))`.
- Tool: one dummy `record_application_submission` with a strict zod input, an output schema and annotations.

| Check | Result |
| --- | --- |
| No token, wrong token, cookie only, token in query string | 401 JSON + `WWW-Authenticate: Bearer` ✅ |
| GET / DELETE | 405 ✅ |
| Browser `Origin` header | 403 ✅ |
| Host not in allowlist | 403 ✅ |
| 40 KB body | 413 ✅ (see finding 2) |
| Official SDK client, legacy mode | Negotiated 2025-11-25; stateless, no session ✅ |
| Official SDK client, `auto` and pinned `2026-07-28` | Negotiated 2026-07-28 ✅ |
| `structuredContent` + `outputSchema` + annotations | Delivered to client ✅ |
| Replay of the same `sourceRecordRef` | Same record ID returned (`already_recorded`) ✅ |
| Unknown input key (e.g. `salaryAnswer`) | Rejected as a tool error (`isError`) ✅ |
| Tool error with code | `isError` + code reached client and model ✅ |
| User identity from token reaches tool handler | Via `ctx.http.authInfo.extra` ✅ |
| **Claude Code 2.1.267**, headless, `--mcp-config` with `"Authorization": "Bearer ${CC_MCP_TOKEN}"` | Connected on **2026-07-28**; tool called; result `created` ✅ |
| Claude Code: tool not allowlisted | Permission denial in headless mode; the agent reports it ✅ (prompt needed interactively) |
| `claude mcp add --scope local … --header 'Authorization: Bearer ${CC_MCP_TOKEN}'` | Stored the variable reference, not the token ✅ |
| `claude mcp list` with a good, wrong or unset variable | `✔ Connected` / `✘ … HTTP 401 …` / warning "Missing environment variables: CC_MCP_TOKEN" ✅ |
| **Codex CLI** | **Unsupported for this environment** (owner, 2026-10-02): office laptop, so no extra AI clients are installed or configured |
| **Antigravity** | **Target client for implementation and testing** (owner, 2026-10-02). Verified at MCP-09, not in this spike |

## Findings that change the MCP feature tickets

1. **`requireBearerAuth` rejects tokens without an expiry** ("Token has no expiration time"). The MCP-02 verifier must return `expiresAt`. Planned tokens always expire, so this fits.
2. **A body over the limit returns Express's default HTML error page with a stack trace.** MCP-04 needs a router-level error handler that returns JSON (413/400) with no stack.
3. **Input that fails validation never reaches the tool handler,** so per-call logging inside the handler misses it. MCP-04 must log outcomes at the router or transport level (`onerror` plus response status), not only in the handler.
4. **With a bad, expired or missing token, Claude Code shows no error to the agent; the tool is simply absent.** The agent cannot tell "not configured" from "token broken", so an expired token would silently stop sync. Changes:
   - **MCP-08:** at START, if the user has set up the integration but the tool is absent, write `Career Companion sync: unavailable` on entries and tell the user once to run `claude mcp list`. Never block applying.
   - **MCP-07:** show token expiry and last use, with a warning in the last 14 days.
5. **Claude Code asks permission per call unless the tool is allowed.** This confirms the plan. MCP-08 setup must include the allow rule `mcp__career-companion__record_application_submission`.
6. **The SDK's `createMcpExpressApp` is not used,** because Career Companion already has an Express app. Its pieces are used directly on the router: `hostHeaderValidation`, `requireBearerAuth`, `toNodeHandler`. The `@modelcontextprotocol/node` package is required for `toNodeHandler`.
   - **Superseded by the plan review (2026-10-02):** `@modelcontextprotocol/express` is not used at all. It re-types `req.auth` app-wide and breaks Career Companion's typecheck, which this throwaway app could not show. MCP-04 uses the same runtime-neutral functions from `@modelcontextprotocol/server` instead.

## Client configuration (for MCP-08)

Claude Code (user's own machine, not committed):

```bash
claude mcp add --scope local --transport http career-companion https://<career-companion-host>/mcp --header 'Authorization: Bearer ${CC_MCP_TOKEN}'
```

Single quotes keep `${CC_MCP_TOKEN}` as a reference, so the token itself is never written to Claude Code's config. The token lives only in the shell environment.

Equivalent `--mcp-config` / `.mcp.json` entry:

```json
{ "mcpServers": { "career-companion": { "type": "http", "url": "https://<career-companion-host>/mcp", "headers": { "Authorization": "Bearer ${CC_MCP_TOKEN}" } } } }
```

Antigravity (from its [MCP docs](https://antigravity.google/docs/mcp); **not yet tested**). Use the global file `~/.gemini/config/mcp_config.json`, which takes the key `serverUrl` (not `url`):

```json
{ "mcpServers": { "career-companion": { "serverUrl": "https://<career-companion-host>/mcp", "headers": { "Authorization": "Bearer <token>" } } } }
```

- **Environment-variable expansion is not documented**, so the token is stored in plain text in that file.
- Keep the file user-only (`chmod 600`).
- **Never** use the workspace file `.agents/mcp_config.json` inside the automation repo, which is public. Add it to that repo's `.gitignore` as a guard.
- Antigravity runs MCP tools in "Ask" mode by default. Allow the one tool with the policy `mcp(career-companion/record_application_submission)`.

Codex: unsupported in this environment (owner decision, 2026-10-02).

## Still open

- **Antigravity compatibility is unverified.** The server serves both 2026-07-28 and 2025-era clients, and both passed here with the official SDK client and Claude Code. Public bug reports mention remote-MCP problems in Antigravity (tools discovered but not invocable; stalls with custom remote servers). Verify in MCP-09 before relying on it.
- The SDK v2 line is about two months old. Pin exact versions in MCP-04.
- **The server stays one standards-compliant endpoint.** There is no per-client server or client-specific code. Client differences are handled only in setup docs.
