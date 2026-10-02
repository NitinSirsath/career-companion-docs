# MCP-08 — Staged changes for `job-application-automation`

> **Before applying (2026-10-02):**
> - The staged README sentence below says only six fields are sent. The tool contract sends twelve (also `sourceRecordRef`, `submittedAt`, `portalJobId`, `destinationHost`, `discoverySource`, `workMode`). Correct it to match ADR-0002 decision 4. No answers, resume or credentials are sent; that part is accurate.
> - Add one line: the feature needs a Career Companion instance the user runs locally.
> - Check the real daily-file heading format (are seconds present?) and how the daily file records platform versus discovery source, before applying.
> - Run the walkthroughs and MCP-09 part B steps 3–5 on a **disposable** database and test user, and do not overwrite a real `applied/<date>/applications.md` file. Applications created by automation cannot be deleted.
> - Scheduled in the [Sprint 6 closeout](../sprint-6/closeout/README.md), order 6 of 8.

## Closeout ticket: MCP-08 and MCP-09 part B steps 3–5 (2026-10-02)

Closeout order 6 of 8. Run by the owner. Size S. Local ID only, not in Linear.

### Objective

Apply the staged changes below to a local copy of `job-application-automation`, prove its three walkthroughs, then finish the Antigravity walkthrough (MCP-09 part B steps 3–5) through a real daily file. This gives MCP acceptance its real-client evidence and gives MCP-10 the part B results.

### Dependencies

- A local copy of `job-application-automation`, provided by the owner.
- The Phase 0 gate has passed, including [MV-14](../migration-verification/README.md#mv-14--verify-mcp-with-real-clients) (part B steps 1–2 and 6–7). If MV-14 recorded `"rejectedOriginHost":"unparseable"`, Antigravity cannot call the tool: record that as the blocker with the owner's OD-11 decision instead of steps 3–5.
- A disposable database and test user, set up as in MV-14 steps 1–2 (a new scratch database; MV-14 deletes its own at cleanup).
- Followed by [MCP-10](MCP-10-post-verification-corrections.md) (order 7), which uses these results.

### Scope

1. **Check the real repository first.**
   - The daily-file heading format: does the entry time have seconds? The tool accepts only `YYYY-MM-DD/HH:MM:SS` (backend `src/services/externalSubmission.ts:40-44`, `RecordApplicationSubmissionInputSchema.sourceRecordRef`). If there are no seconds, stop and get the owner's decision before applying; the agent must never invent them.
   - How the daily file records platform versus discovery source (field names, values). Match the section 2 table and the fixture labels to the real ones.
2. **Correct the staged README text (section 4).** Replace the "only the company, job title, platform, job link, location and the confirmation text" sentence with all 12 fields (ADR-0002 decision 4), using the same list as [MCP-10](MCP-10-post-verification-corrections.md) item 1. Keep "answers, resume and passwords never leave your computer". Add one line: the feature needs a Career Companion instance the user runs locally.
3. **Apply** sections 1–6 below in the local copy, in the repository's own style.
4. **Run the three walkthroughs** (section 6): tool absent, sync failure, replay. Use entries or a date other than the fixture's `2026-10-01`, or step 5 sees `already_recorded`.
5. **MCP-09 part B steps 3–5** ([execution report](execution-report.md#remaining-external-verification-owner)) on the disposable database: set `Career Companion sync: yes`, copy `career-companion-frontend-main/scripts/fixtures/mcp-daily-applications.md` to `applied/2026-10-01/applications.md`, create the two seed applications, then ask "sync Career Companion for 2026-10-01" twice. If that file already exists, do not overwrite it: use a second, throwaway copy of the repository.

### Acceptance criteria

- [ ] Heading format and platform/discovery-source recording are written down, and section 2 matches them (or the owner's decision is recorded).
- [ ] The applied README names all 12 fields, still says answers, resume and passwords stay on the computer, and has the local-instance line.
- [ ] File names and sections match the repository. No token and no MCP settings file is in the repository.
- [ ] With `Career Companion sync: no` (the default), a session behaves exactly as before: no tool call, no message, no sync lines.
- [ ] The three walkthroughs give the results written in section 6.
- [ ] Part B: Fabrikam created, Northwind Traders linked, Contoso in the dashboard review panel, Skipped Labs never sent. The second ask records nothing new (`already_recorded`). The three `applied` entries end with `- Career Companion sync: sent`.
- [ ] No hit for `FIXTURE-SECRET-ANSWER` in the backend log or in `pg_dump --data-only` of the scratch database.
- [ ] Or: a blocker is recorded with an owner decision.

### Security and privacy

- Disposable database and test user only. Applications created by automation cannot be deleted (MCPF-02, OD-10).
- Token: 30-day expiry (the shortest option, frontend `src/components/automation/CreateTokenForm.tsx:10`, `EXPIRY_OPTIONS`), kept only in the user-level `~/.gemini/config/mcp_config.json` with `chmod 600`, never in the repository. Revoke it on the Automation page after the run.
- A low `MCP_DAILY_SUBMISSION_LIMIT` (10, as in MV-14) in `.env` while running; remove it after.
- Fixture data only. No submitted answers, resume or credentials leave the laptop; the tool rejects unknown keys (`z.strictObject`).

### Testing and evidence

Manual only; the automation repository gets no code. For each walkthrough and part B step, record the prompt, the sync lines written to the daily file, the tool results, and the `mcp_request` log lines (`grep '"event":"mcp_request"'`; IDs and outcomes only). Also record the Antigravity version and the automation repository commit. These are local manual results, not automated proof.

### Definition of done

- Every acceptance criterion is met with evidence, or a blocker is recorded with an owner decision.
- The automation-repository changes are committed there in one focused commit.
- Clean-up done as in MV-14: token revoked, client entry removed, `unset CC_MCP_TOKEN`, `MCP_*` lines removed from `.env`, dev `DATABASE_URL` restored, scratch database dropped.
- Evidence recorded in `../sprint-6/closeout/execution-report.md` (created if absent). [S6-C02](../sprint-6/closeout/S6-C02-docs-truth-and-acceptance-records.md) §4.8 then updates the MCP plan DoD boxes, the MCP execution report rows and the status line below.

---

Status: **ready to apply, not applied (2026-10-02).** The `job-application-automation` repository is not on this laptop (plan section C), so these changes could not be made or checked against its files. Everything on the Career Companion side that MCP-08 depends on is implemented and tested (MCP-02 to MCP-04).

Before applying:
- Use a local copy of the repository you provide.
- Check each file name and section number below against the repository. They come from ADR-0002's reading of it: `AGENTS.md`, `README.md`, `personal_data/profile_template.md`, `applied/YYYY-MM-DD/applications.md`, `.gitignore`, and the walkthrough checklist in `AGENTS.md` §12.
- Keep the repository's own wording style. The text below is the content to add, not a fixed format.
- No code is added: these are instructions and setup notes only (ADR-0002, "Not part of this decision").

Done when: a user who has not set `Career Companion sync: yes` sees no difference at all.

---

## 1. `personal_data/profile_template.md` — opt-in flag

Add, near the other preferences:

```markdown
- Career Companion sync: no   <!-- yes | no. "yes" only after following the README setup. -->
```

The default is `no`. The agent uses the integration only when the user's profile says `yes`. The flag is what lets the agent tell "not set up" from "set up but broken".

## 2. `AGENTS.md` — new optional section

Add as a new section. Use it **only** when the profile says `Career Companion sync: yes`.

```markdown
## Career Companion sync (optional)

Skip this whole section unless the profile says `Career Companion sync: yes`.

**At START:** check that the MCP tool `record_application_submission` (server `career-companion`) is available.
If it is not, tell the user once: "Career Companion sync is turned on, but its tool is not available. Check the
MCP server status in your AI client (Antigravity: MCP servers panel; Claude Code: `claude mcp list`). A bad or
expired token makes the tool disappear without an error." Then mark every `applied` entry this session with
`- Career Companion sync: unavailable`. Never stop or slow down applying because of this.

**After each `applied` entry is appended** (only after the site showed a submission confirmation), call
`record_application_submission` once, with values copied from that entry:

| Tool field | Value |
| --- | --- |
| `sourceRecordRef` | `YYYY-MM-DD/HH:MM:SS`: the date folder of the daily file, `/`, and the time in the entry heading. Copy it exactly; never invent or reformat it. |
| `platform` | The application workflow actually used, one of: `linkedin`, `indeed`, `naukri`, `wellfound`, `instahyre`, `workday`, `company_direct`, `discovery`. **Workday wins** whenever a Workday form was used, wherever the job was found. Never put the source name here. |
| `company`, `jobTitle` | From the entry. |
| `submittedAt` | The confirmation time with your local UTC offset, for example `2026-10-01T09:15:00+05:30` (get the offset from `date +%z`). |
| `jobUrl`, `portalJobId`, `destinationHost`, `location`, `workMode` (`remote`, `hybrid` or `onsite`) | From the entry when known; otherwise leave the field out. |
| `discoverySource` | Where the job was found (for example `we_work_remotely`, `the_reliable_jobs`). |
| `confirmationText` | The confirmation the site showed (it may be shortened by Career Companion). |

**Never send** submitted answers, salary, notice period or any screening answer, the resume or its contents,
credentials or account records, `skipped` or `needs_user` entries, or plan, limit, cooldown, tab or session
state. The tool rejects any field it does not know.

Then add one line to the entry:

- `- Career Companion sync: sent` when the result is `created`, `linked`, `needs_review` or `already_recorded`.
- `- Career Companion sync: failed: invalid_input <field>` when the tool returns `invalid_input`. Name the fields it
  lists, include it in the end-of-session summary, and do not retry until the entry is fixed.
- `- Career Companion sync: failed: rate_limited` (retry the next day) or `- Career Companion sync: failed: unavailable`
  (retry later) for those errors, or on any other failure.

Never block, delay or retry-loop an application because of sync. The daily file stays the source of truth.

**Replay on request:** when the user says "sync Career Companion for YYYY-MM-DD", read that day's file and call the
tool for every `applied` entry whose sync line is not `sent`, then update those lines. Repeats are safe: the same
`sourceRecordRef` is recorded once and returns `already_recorded`.
```

## 3. Daily file entry format

The new line goes at the end of each `applied` entry, for example:

```markdown
### 09:15:00 — Fabrikam — Backend Engineer
- …existing fields…
- Status: applied
- Confirmation: Thank you for applying to Fabrikam.
- Career Companion sync: sent
```

`skipped` and `needs_user` entries get no sync line.

## 4. `README.md` — setup

Add a section:

```markdown
## Optional: show your applications in Career Companion

If you use Career Companion, the agent can report each confirmed application to it, so it appears in your
dashboard and later recruiter emails attach to it. Your submitted answers, resume and passwords never leave
your computer: only the company, job title, platform, job link, location and the confirmation text are sent.

1. In Career Companion, open **Automation** and create a token. Copy it; it is shown once.
2. Add the server to your AI client's **user-level** settings. Never put the token in a file inside this repository.
   - **Antigravity (recommended):** edit the global file `~/.gemini/config/mcp_config.json`:

     ```json
     { "mcpServers": { "career-companion": { "serverUrl": "<MCP server URL from the Automation page>", "headers": { "Authorization": "Bearer <token>" } } } }
     ```

     Then run `chmod 600 ~/.gemini/config/mcp_config.json`. Never use `.agents/mcp_config.json` in this repository.
     Allow the one tool so you are not asked every time: `mcp(career-companion/record_application_submission)`.
   - **Claude Code (optional):** keep the token in your shell environment as `CC_MCP_TOKEN`, then run
     `claude mcp add --scope local --transport http career-companion <MCP server URL> --header 'Authorization: Bearer ${CC_MCP_TOKEN}'`
     (single quotes keep the variable reference instead of the token) and allow the tool
     `mcp__career-companion__record_application_submission`.
3. In your profile in `personal_data/` (created from `profile_template.md`), set `Career Companion sync: yes`.
4. Tokens expire (90 days by default). When one expires, the tool disappears without an error and the agent marks
   entries `unavailable`. Create a new token and replace it in the settings above, then ask
   "sync Career Companion for YYYY-MM-DD" to send any missed days.
```

## 5. `.gitignore`

Add:

```gitignore
# MCP client settings may contain a Career Companion token; never commit them.
.agents/mcp_config.json
```

## 6. Walkthrough checklist (`AGENTS.md` §12) — three new scenarios

```markdown
- **Sync tool absent:** profile has `Career Companion sync: yes` but the tool is missing (no server, or an expired
  token). Expect: one message at START, entries marked `unavailable`, applying continues normally.
- **Sync failure:** the tool returns `invalid_input` (for example a source name sent as `platform`). Expect:
  `failed: invalid_input platform` on the entry and in the session summary, no retry loop, applying continues.
- **Replay:** "sync Career Companion for YYYY-MM-DD" twice. Expect: only entries not marked `sent` are sent the first
  time; the second time nothing new is recorded in Career Companion (`already_recorded`).
```

## 7. Remaining verification (owner)

- Apply the changes above in a local copy of the repository and check them against the real file names and sections.
- Run the three walkthrough scenarios, then MCP-09 part B (see the [execution report](execution-report.md)).
