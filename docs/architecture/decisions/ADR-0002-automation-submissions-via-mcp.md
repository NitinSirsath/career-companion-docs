# ADR-0002 — Career Companion receives automation-submitted applications through its own MCP server

| Field | Value |
| --- | --- |
| Status | **Accepted** by the owner on 2026-10-02 after explicit review ([Project Constitution — Change Management](../../../PROJECT_CONSTITUTION.md)). **SDK line decided by MCP-00 (2026-10-02): v2**: `@modelcontextprotocol/server` 2.2.0 and `/node` 2.1.0. The spike also used `/express` 2.0.1, but Career Companion does not: it re-types `req.auth` for the whole app (plan review 2026-10-02, MCP-04). Claude Code is verified on 2026-07-28. Antigravity is the target client and is verified in MCP-09. Codex is unsupported in the owner's environment ([spike report](../../planning/mcp-feature/MCP-00-spike-report.md)). **Implemented locally 2026-10-02** (MCP-01 to MCP-07 and MCP-09 part A; [execution report](../../planning/mcp-feature/execution-report.md)). MCP-08 is staged for the automation repository, and MCP-09 part B (Antigravity) is run by the owner. |
| Date | 2026-10-02 |
| Decides | How applications submitted by `job-application-automation` reach Career Companion: where the MCP server lives, which data crosses the boundary, how it is authenticated, how it is matched to existing applications, and which system owns what |
| Plan | [MCP Feature Planning — Automation submissions](../../planning/mcp-feature/README.md) |

## Context

`job-application-automation` is a separate project that applies to jobs for the user. It contains **no executable code**. It is a set of Markdown instructions that an AI agent (Claude Code, Codex, Antigravity or similar) follows on the user's laptop, using a browser tool.

Each outcome is written by the agent to one file per local date, `applied/YYYY-MM-DD/applications.md`. Each job gets one entry headed `### HH:MM:SS — Company — Job Title`, with these fields:

- platform and actual application destination
- company, job title, location, work mode
- job URL and portal job ID
- discovery source
- resume filename
- **submitted answers**: salary, sponsorship, notice period and other screening answers
- status: `applied`, `skipped` or `needs_user`
- the confirmation shown by the site

The same folder also holds the user's profile and plaintext company-portal passwords (`personal_data/credentials.md`, `tracking/created_accounts.csv`). The automation has no application IDs. It avoids applying twice using `date + platform + company + portal + job_id`, falling back to job title and URL.

Career Companion today:

- **Is intended to run in the cloud** (AWS ECS behind an ALB) and is multi-user. Users sign in with Google, and the server keeps a session cookie. There is no machine-to-machine authentication.
- **The Gmail pipeline never creates applications.** Users create them by hand, and emails attach to them by:
  - Gmail thread, or
  - an **exact** normalized company name, plus job title when both sides have one.
- **`Application` has no URL, platform, source or external ID**, and no business uniqueness rule (deliberately).
- **Timeline events assume an email.** Their unique index `(applicationId, emailId, type)` does not deduplicate events without an email, because Postgres treats NULLs as distinct.
- **`Email` already models the needed pattern**: a source record with `matchState` and `matchConfirmedBy` that is linked to an `Application`, automatically or by the user.

The original idea was an MCP server in front of the automation, with Career Companion as the MCP client pulling data. That would require:

- the cloud backend calling into a laptop, through a public tunnel to a machine that holds passwords and personal data;
- the laptop to be awake;
- a parser for AI-written Markdown;
- MCP to be used as a backend-to-backend sync protocol, which it is not designed for.

MCP is designed for **AI applications calling tools on external systems**. The automation is already run by an AI application that is an MCP client.

The current MCP specification is 2026-07-28: stateless, with Streamable HTTP and stdio transports, and OAuth 2.1 recommended but optional for HTTP. The official TypeScript SDK v2 (`@modelcontextprotocol/server`, `@modelcontextprotocol/express`) uses zod v4 on Node 20+, which matches Career Companion's Express 5 and zod v4 stack.

## Decision

1. **Career Companion hosts the MCP server; the automation agent is the client.**
   - The server is an adapter module (`backend/src/mcp/`) in the existing backend process. It is mounted at `POST /mcp` over Streamable HTTP and calls the same domain services as the REST routes.
   - There is no new service, no new database and no queue.
   - It is **one standards-compliant MCP server for all clients**. There are no per-client servers and no client-specific server code; client differences live only in setup documentation.

2. **Data moves one way, pushed at the moment of submission.**
   - After the agent appends an `applied` entry to the daily file, it calls one tool with values copied from that entry.
   - Career Companion never writes back to the automation and never reads the laptop.

3. **The MCP surface is one write-only, idempotent tool, `record_application_submission`** (contract below).
   - No resources and no prompts.
   - No tools that read Career Companion data.
   - The tool's result carries no data beyond the outcome and a record ID.

4. **The data crossing the boundary is minimal.**
   - **Sent:** identity and job fields only.
   - **Never sent:**
     - submitted answers;
     - resume file or contents;
     - credentials or account records;
     - `skipped` and `needs_user` entries;
     - plan, limit, cooldown, tab and session state.
   - Unknown input keys are rejected, so extra data cannot slip in by accident.

5. **The idempotency key comes from the source record, not from job fields.**
   - `sourceRecordRef` = `YYYY-MM-DD/HH:MM:SS`, taken from the daily file's entry heading. The agent applies one job at a time, so this is unique per submission. A replay copies it verbatim.
   - Uniqueness rule: `(userId, source, sourceRecordRef)`. The first write wins, and a repeat returns `already_recorded`.
   - Job-field keys were rejected because a model does not reproduce URL forms or optional IDs consistently.

6. **Career Companion persists an immutable `ExternalSubmission` record, modelled on `Email`.**
   - It holds the minimal fields, the match state (`NEEDS_REVIEW`, `LINKED`, `CREATED`, `IGNORED`), whether the match was automatic or made by the user, an optional `applicationId`, and the token that created it.
   - The automatic-vs-user marker is its own enum (`AUTOMATIC | USER`). The Gmail `AI_AUTO` value is not reused, because no AI makes this match.
   - A linked or created submission adds one `ApplicationEvent` of type `AUTOMATION_SUBMITTED`, keyed by a unique `externalSubmissionId`.
   - Ownership is enforced by database triggers, following the `domain_integrity` migration.

7. **Matching is conservative and separate from the Gmail matcher.** The Gmail matcher is unchanged.
   - Company key = the existing normalization, after removing trailing legal-suffix words: inc, incorporated, llc, llp, ltd, limited, pvt, private, corp, corporation, co, plc, gmbh.
   - Title key = the existing normalization.

   Let C = the user's applications with the same company key, E = those in C with the same title key, and N = those in C with no title.

   | Situation | Result |
   | --- | --- |
   | C is empty | **CREATED**: new Application, `appliedAt = submittedAt` |
   | E has exactly one application and N is empty (other titles at that company don't matter) | **LINKED**: `appliedAt` is set only if empty |
   | Anything else (E empty or several, or any untitled application at that company) | **NEEDS_REVIEW**: the user links, creates or ignores, as with ambiguous emails |

   - Intake runs as one transaction per call under a per-user advisory lock: check ref, check cap, insert, match, create or link with event.
   - The lock means concurrent submissions can't create duplicate applications.

8. **Status semantics are unchanged.**
   - Automation never writes `aiStatus` or `userStatus`. `deriveStatus` and its response check stay as they are.
   - The application response gains `submittedVia: 'AUTOMATION' | null`. The UI shows "Applied · via automation" only when the derived status source is `UNKNOWN`.
   - User corrections always win.

9. **The timeline stays in recording order.**
   - The `AUTOMATION_SUBMITTED` event is rendered as reported by the automation, never as AI interpretation. It shows the submission time, platform, destination host and confirmation text.
   - A backfilled submission can appear after later emails. No occurrence-time column is added.

10. **Authentication uses per-user integration tokens.**
    - **Format:** `ccmcp_` + 32 random bytes, base64url. Only a SHA-256 hash and a short display prefix are stored.
    - **Lifecycle:** shown once, single scope `submissions:write`, expiry by default 90 days (at most 365), revocable immediately, `lastUsedAt` recorded.
    - **Use:** sent as `Authorization: Bearer`. It is never in a committed file.
      - Clients that support environment-variable expansion (Claude Code) read it from the environment.
      - Antigravity, the owner's client, stores it in plain text in its user-level config, which is outside any repo and user-readable only.
      - Short expiry, write-only scope and revocation keep a leak's impact bounded.
    - **Separation from the web app:**
      - The user is taken only from the token, never from tool input.
      - Tokens are accepted only on `/mcp`, and session cookies are not accepted there.
    - The specification allows this for a single-scope, first-party client. OAuth 2.1 is future work.

11. **The endpoint is hardened at the HTTP boundary.**
    - Mounted before all global middleware (CORS, JSON body parser, cookies, session), with its own 32 KB body limit.
    - POST only; GET and DELETE return 405.
    - Host allowlist from configuration.
    - `Origin` is validated against a configured allowlist of hostnames (default empty); requests without `Origin` pass.
      - Rejecting every `Origin` could block IDE-based clients.
      - Bearer-header auth, never cookies, already removes cross-site request risk.
    - HTTPS in production through the existing load balancer.
    - Abuse bound: at most `MCP_DAILY_SUBMISSION_LIMIT` (default 500) new records per user per UTC day, counted in Postgres. No Redis.

12. **Failures never block applying.**
    - The daily file stays the local source of truth.
    - The agent records `Career Companion sync: sent | failed | unavailable` on the entry and can later replay a day; idempotency makes this safe.
    - Tool errors are returned as MCP tool errors (`isError`) with a code and a retry hint.

13. **Every call leaves an audit trail without personal data.**
    - One structured log line per call: token ID, user ID, tool, result, duration, and whether a repeat differed from the stored record. It contains no payload values.
    - The `ExternalSubmission` row and the token's `lastUsedAt` complete the trail.
    - No separate audit table.

## Tool contract (v1)

**`record_application_submission`**
- Annotations: `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: true`, `openWorldHint: false`.
- The tool description tells the agent to call it only after the site showed a confirmation and the entry was appended.

**Input** (strict object; unknown keys are rejected):

| Field | Rule |
| --- | --- |
| `sourceRecordRef` | required; `^\d{4}-\d{2}-\d{2}/\d{2}:\d{2}:\d{2}$` |
| `platform` | required; `linkedin \| indeed \| naukri \| wellfound \| instahyre \| workday \| company_direct \| discovery` (the actual destination workflow) |
| `company` | required; 1–200 characters |
| `jobTitle` | required; 1–200 characters |
| `submittedAt` | required; ISO 8601 with UTC offset; not more than 5 minutes in the future |
| `jobUrl` | optional; `http`/`https`, ≤ 2048. Stored without the fragment; the query string is reduced to the job-ID parameters `jk`, `currentJobId`, `gh_jid`, `jobId` |
| `portalJobId` | optional; ≤ 200 |
| `destinationHost` | optional; a valid hostname, ≤ 253 |
| `discoverySource` | optional; ≤ 100 |
| `location` | optional; ≤ 200 |
| `workMode` | optional; `remote \| hybrid \| onsite` |
| `confirmationText` | optional; longer text is truncated to 300 characters (not rejected); stored and shown as plain text |

**Output** (`structuredContent`, also mirrored as a text block): `{ result: "created" | "linked" | "needs_review" | "already_recorded", recordId: string }`.

**Errors** (tool errors, `isError: true`):

| Code | Retry |
| --- | --- |
| `invalid_input` | do not retry |
| `rate_limited` | retry the next UTC day |
| `unavailable` | retry later |

A missing, expired or revoked token gets HTTP 401 before any MCP processing.

## Ownership

| System | Owns |
| --- | --- |
| job-application-automation (laptop) | Whether and what was submitted; the daily file; submitted answers; resume; credentials and account records; plans, limits, cooldowns; `skipped` / `needs_user` outcomes |
| Career Companion | The Application and its lifecycle after submission; Gmail data; timeline; actions; notifications; matching decisions; user status corrections |

An `ExternalSubmission` is evidence, not a replica: it is never updated from a later call. A resolved review is final in v1.

## Not part of this decision

- A pull or sync API, list or read tools, resources, prompts, or batch tools.
- Any copy of submitted answers, resume content or account records in Career Companion.
- Changes to the Gmail matcher, `deriveStatus`, AI processing, or timeline ordering.
- OAuth 2.1 authorization, several scopes, or third-party MCP clients.
- Unlinking or re-resolving a submission after it is linked or created.
- Notifications for submissions.
- Code in `job-application-automation`. Its change is an optional instruction section and setup notes.

## Consequences

**Positive**
- Uses MCP for what it is built for: an AI host calling a tool on an external system.
- No tunnel, no laptop daemon, no Markdown parser, and no new service or infrastructure.
- Sensitive answers and credentials never leave the laptop. The write-only surface means a prompt-injected agent can at worst create reviewable records; it cannot read data.
- Applications now exist before the recruiter's email arrives, so the existing Gmail matcher has something to attach to. It attaches only when the email names the company the same way after normalization, because the Gmail matcher stays exact.
- MCP is a thin adapter over domain services, so a REST import can be added later without new logic.

**Negative**
- A model is in the write path. Field quality equals the daily file's quality. Strict validation and the review queue bound this; the server cannot verify that a confirmation was really shown.
- Conservative matching sends same-company, different-title submissions to review, which adds review work.
- Backfilled submissions appear in recording order, not real chronology.
- MCP clients ask permission per tool call (Antigravity "Ask" mode, Claude Code prompts) unless the user allows this one tool locally.
- The SDK v2 line and spec 2026-07-28 are recent. MCP-00 verified Claude Code. Antigravity, the target client, is unverified until MCP-09 and has public remote-MCP bug reports. `@modelcontextprotocol/sdk` 1.x remains the fallback.
- Until Career Companion is deployed, the integration only works with it running on the same laptop.

## Alternatives considered

| Alternative | Reason rejected |
| --- | --- |
| MCP server inside the automation; Career Companion pulls | Needs a public tunnel into a laptop holding passwords; only works while the laptop is awake; requires the first code and a Markdown parser in a no-code repo; uses MCP as a backend sync protocol |
| Local sync script posting to a REST import endpoint (no MCP) | Deterministic, but needs a parser for AI-written Markdown that drifts. It remains a viable later addition on top of the same submission service |
| Separate MCP service or repository | A second deployable and auth surface for one tool, with no benefit at this scale |
| Idempotency key derived from job fields | The model does not reproduce URL forms and optional IDs consistently, so replays would duplicate |
| Create an application whenever no exact match exists | Exact normalized matching misses "Acme Inc." vs "Acme" and title variants, so this creates duplicates |
| Write `aiStatus = APPLIED` on import | Mislabels a reported fact as AI inference and breaks the status contract |
| OAuth 2.1 from day one | Career Companion is not an authorization server today; the specification makes it optional; a scoped revocable token covers one first-party client |

## Future evolution

These are not designed now. Each is additive.

- **OAuth 2.1:** add protected-resource metadata and an authorization server; the token check is the single place that changes.
- **Local, read-only answer viewer:** a stdio MCP server inside the automation, limited to `applied/**/applications.md`, so the user's own AI host can show submitted answers without them leaving the laptop.
- **Deterministic backfill:** a REST import endpoint or script reusing the same submission service.
- **Unlink and re-resolve:** an explicit user action with event history.
