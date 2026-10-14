# Linear — AI Collaboration Reference

## Purpose

This document defines how Linear is used as the source of truth for project execution within Career Companion. It sets the rules for creating and keeping issues in Linear, for the owner and for every AI agent.

**AI agents:** read this whole file before you create, edit, move or comment on any Linear issue. A ticket prompt may repeat these rules or narrow them. If a prompt asks for something this file forbids (for example deleting an issue), stop and ask the owner. Do not pick one yourself.

---

## Role in the Project

Linear is the **single source of truth for project execution**. All work must be tracked here before it is started. Nothing is built without a corresponding Linear issue.

If Linear is down, or the work must start offline, use a clearly marked temporary local ID. Before the work is done, move it onto a real Linear issue. From then on, the Linear issue is the record.

---

## Fixed Values

| What | Value |
|---|---|
| Team | Career Companion, issue key `COM` |
| Labels that exist | `Improvement`, `Bug` |
| Labels that do not exist | `Product`, `Documentation` (checked 14 Oct 2026). Use `Improvement` instead |
| Statuses | Backlog, Todo, In Progress, In Review, Done, Canceled |
| Priorities | High, Medium, Low for new issues. Do not use Urgent. Older issues under COM-158 and COM-183 carry Urgent; leave them as they are |
| Links | Sub-issue (parent), Related, Blocked by, Duplicate |
| Assignee, estimate | None. One owner does all the work |

When this table and Linear disagree, Linear wins. Tell the owner, so this file gets fixed.

---

## Structure Conventions

### Epics

An epic is a parent issue for one piece of work. Its sub-issues are the work. Each sub-issue covers one layer.

- Only the owner decides a new epic. An agent puts new work under an existing parent, or asks.
- A parent says why the work exists, what the owner decided (with the date), the hard rules every sub-issue follows, the order of work, and what is not in it.

Current parents (14 Oct 2026; check Linear for the live list):

| Parent | Area |
|---|---|
| `COM-158` | Mail-first walkthrough: Mails page, mail details, one-click applications |
| `COM-183` | Dev mode and the mail sandbox |
| `COM-198` | Gemini busy and stuck mail (COM-199 to COM-204, merged 13 Oct). New AI error work goes under COM-222 |
| `COM-222` | AI errors: retry by itself, pause per model, the exact error |

### Issues

**One issue per layer.** Backend, frontend and docs work are separate issues under the same parent. One issue never covers two repositories. One PR may cover a few issues of the same layer and parent.

**Title.** Plain words that say what and where. Sub-issues start with their layer:

| Kind | Pattern | Example |
|---|---|---|
| Parent | `<Area>: …` | `AI errors: retry by itself, pause per model, and show the exact error` |
| Backend | `Backend: …` | `Backend: logs that trace one mail's AI tries from start to end` |
| Frontend | `Frontend: …` | `Frontend: the retry dialog says what happened` |
| Docs | `Docs: …` | `Docs: record the owner's 14 Oct retry decisions` |
| Mock | `Design: mock of …` | `Design: mock of how AI errors are shown` |
| Later phase | `[Later] …` (layer optional) | `[Later] Notifications for every user` |

**Description.** The whole spec is in the issue, in plain English and short sentences. It must make sense without any chat history (see the Linear Issue Quality Standard in `PROJECT_CONSTITUTION.md`). Owner decisions carry the date and the word "owner".

**Mock.** Any visible change has a `Design: mock of …` issue, and the work issues are "blocked by" it. When the owner approves the mock, its link goes into the work issues (in the description, or in a dated comment if they already exist). Code does not start before that.

**Templates.**

A piece of work:

```markdown
## Why
What is wrong or missing today, and for whom. What it should do instead.

## What we build
- What changes: screens, API fields, rules.

## Details (write each one that applies; leave the others out)
- API or data contract: fields, types, which repo owns it.
- Likely files.
- Privacy: what user data this touches.
- When it fails: what the user sees, and whether it is safe to run twice.
- Depends on: other issues (also set them as "blocked by").

## Not in it
- What this issue does not do.

## How we check
- Tests to add or run, and what to try by hand.

## Done when
- [ ] Checkable outcomes. Each one can pass or fail.
```

A bug:

```markdown
**Where:** page, mode (dev or prod), which mail or account.
**Steps:**
1. …
**Expected:** …
**Actual:** …
**Evidence:** screenshot, log line, or the request and response.
```

A later-phase issue starts with:

```markdown
> **LATER.** Decided by the owner on <date>: <why later>. Do not start before the owner says so.
```

### Milestones / Cycles

Not used now (owner, 14 Oct 2026). Sprints 5 to 8 were planned as sprints; see `docs/planning/`. Current work is ordered with "blocked by" links. An issue that only waits for an earlier issue keeps its status and gets that link.

---

## Issue Lifecycle

- **Backlog or Todo**: not started yet. `[Later]` issues stay in Backlog.
- **In Progress**: work has started. This includes an open draft PR.
- **In Review**: the code is done and waits only for the owner's own checks, for example a run with real Gmail or a real AI key.
- **Done**: the PR is merged (Linear does this by itself) and every "Done when" item passes. If a check is still open after the merge, move the issue back to In Review and say which check.
- **Canceled**: not needed any more, or replaced by another issue (see Editing).

When a PR opens, its comment in Linear says what was checked and on which commit. Name any check that could not run as "not checked". Never call it passed.

Keep the status true at all times. Do not leave an issue in Backlog or Todo once work has started.

---

## Workflow Rules

**Before you create.**

1. **Search first.** Look for an open or recently done issue for the same work. If one exists, comment on it or update it instead of creating another.
2. **Pick the parent** from the table above. If none fits, ask the owner.
3. **Plan the mock.** A visible change needs a `Design: mock of …` issue (see Mock above).
4. **Justify each new control or feature** against the real case. Anything speculative is left out or marked as a question for the owner.

**When you create.**

- Use only labels from the Fixed Values table. If a label is missing, use `Improvement` and tell the owner.
- Priority: **High** for work on the critical path, including a docs or mock issue that other issues wait for. **Medium** for follow-ups and docs that block nothing. **Low** for every `[Later]` issue.
- Set "blocked by" links for the order of work.

**After you create.**

- **Read it back:** parent, labels, priority, status and links are what you meant.
- **Report** a list: each issue ID, title and URL; each link; each comment added or skipped; any label you had to change.
- **Never blindly retry a create or a comment.** A slow call may still have worked. Look in Linear first, or you make duplicates.

**Editing.**

- Comments are short and dated. Start with "owner:" when the comment records an owner decision.
- Do not rewrite the title or description of an existing issue unless the owner asks. Put new facts in a comment.
- Status changes follow Issue Lifecycle.
- **An issue replaced before it was built:** link it to the new issue as Related or Duplicate, comment why, and move it to Canceled.
- **An issue that was built and is later changed by newer work:** keep it Done, and add a comment that names the newer issue.
- Never delete an issue.

---

## Integration with GitHub

- Every branch and PR belongs to one Linear issue, or to a few issues of the same layer and parent.
- **The PR title or description names its own child issue IDs only, never a parent's ID.** The branch name follows the same rule. Linear moves every issue named in a PR to Done when the PR merges; naming a parent closed COM-176 too early.
- When a PR opens: link it to its issues, move them to In Progress, and add a short comment. Never move an issue to Done by hand; the merge does it. Check that the parent stays In Progress.
- Branch names use the prefixes in `ai/github.md` (`feat/`, `fix/`, `refactor/`, `docs/`, `test/`, `chore/`), and may carry a child issue ID, for example `fix/COM-199-gemini-models-and-busy`.
- Stacked branches follow the parent's order of work.
- Pull requests start as drafts. Who commits and who merges is set in `DEVELOPMENT-MODES.md`.

---

## Integration with AI Agents

AI agents must reference the real Linear issue for issue-driven work and keep its status true.

If you used a temporary local ID:
- note the local ID on the real Linear issue;
- do not mark the issue Done for work that is only on your machine;
- put what was built and how it was checked in Linear before Done;
- say clearly what is only local and what is on GitHub.

An agent that creates issues from a prompt (for example ChatGPT in mode 3, "Prompt-Only / Remote Repository Mode" in `DEVELOPMENT-MODES.md`) follows this file. If a step fails, it stops and says which step failed and why. It does not work around it.

Gemini/Antigravity remains the primary implementation environment under the Project Constitution, while other approved agents may assist within the development-mode boundaries.
