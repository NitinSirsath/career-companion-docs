# How we build Career Companion

Read this before writing or reviewing code. Each code repo's `AGENTS.md` has the short version. The PR that fixes a "Today" note removes it.

## 1. Backend

1. **Routes** (`src/routes/`): HTTP only. Read the session user, validate input with the contract schema, call one service function, and send the result. No database queries, no business rules, no error JSON. Copy `routes/ai.ts`.
2. **Services** (`src/services/`): all business rules, as plain exported functions. No classes with static methods. Group by area (`services/ai/`).
3. **Errors**: for expected failures (not found, conflict, not allowed, invalid), services throw `AppError(status, code, message, details?)` from `src/errors.ts`. The central handler (`middleware/error.ts`) sends `{ error: { code, message, details? } }`. Routes don't catch errors to build responses. Internal errors used for control flow (Gmail sync, AI providers) stay as they are, and MCP keeps its own mapping because it is a different protocol.
4. **Background work**: files in `src/jobs/` only register pg-boss workers and call services. Services queue work with the `enqueue…` functions in `services/enqueue.ts`, never by importing from `jobs/`. Copy `jobs/notificationJob.ts`.
5. **One direction**: Gmail sync → queue → AI processing → matching → actions and agenda (the golden path). A step never calls back up the chain. Steps shared by both sides, such as re-offering waiting emails, live in `services/ai/offer.ts`.
6. **Contracts** (`src/contracts/`): Zod schemas and types for every request and response. Change them only in the backend; the frontend copies them with `npm run sync-contracts`. Cross-cutting API rules live in the [API contracts doc](../architecture/api-contracts.md).
7. **Config**: only `src/utils/config.ts` reads `process.env`. It parses and validates each variable once and exports small getters (read at call time, so tests can still change env).
8. **Logs**: `logEvent`, `logWarn`, `logError` and `logDebug` from `src/utils/log.ts`. On a failure, pass the error as the third argument of `logError`. Log events, not data: IDs, counts and outcomes. Never log keys, tokens, request or response bodies, or email content. Raw `console.*` fails lint.
9. **Integrations**: follow the AI provider shape (`services/ai/providers/`): one adapter per outside service behind one function, and the adapter never sees users or the database.
10. **Tests**: `src/tests/<area>.test.ts` in kebab-case. API tests go through HTTP (supertest), with shared setup in `src/tests/helpers/`.

## 2. Frontend

1. **Routes** (`src/routes/`): TanStack Router file routes. A route file holds the route definition, search params and a page that composes components. Logic goes into hooks and `src/lib/`.
2. **Data** (`src/api/`): one file per area (`gmail.ts`, `applications.ts`, `ai.ts`, `actions.ts`, `agenda.ts`, `workspace.ts`, `automation.ts`), sharing one `request` helper and `ApiError`. Each area file holds its API calls, its cache keys (one keys object, such as `gmailKeys`) and its hooks (`useGmailMessages`, `useTriggerSync`). Pages and components use the hooks. They never call `useQuery`/`useMutation` with their own keys. *Today: one 511-line `ApiClient` class and 36 inline calls; until COM-156, use or add a key helper like `applicationKey()` in `lib/applicationCache.ts`.*
3. **Decisions** (`src/lib/`): plain, unit-tested functions for labels, statuses, dates and formatting. No React. No decisions inside JSX. Copy `lib/statusLabels.ts`. *Today: `GmailPage` decides labels inside JSX, and `src/utils/gmail.ts` is a second helper folder (COM-156).*
4. **UI parts** (`src/components/ui/`): use `Button`, `Badge`, `NativeSelect`, `Dialog` and `Input`. No hand-styled raw elements. *Today: 7 raw `<select>` elements, 5 raw `<button>` elements and a hand-made badge (COM-156).*
5. **Feature components** (`src/components/`): one main component per file. Small private parts may stay with it. When an area has several components, group them in a folder (`components/ai/`). *Today: `FollowThrough.tsx` holds three stateful components in 520 lines (COM-156).*
6. **Forms**: react-hook-form with `zodResolver` and the contract schema. Copy the create-application form in `routes/applications.tsx`. *Today: 5 forms keep each field in `useState` (COM-156).*
7. **Contracts** (`src/contracts/`): never edit by hand. Run `npm run sync-contracts`. CI's contract-drift check fails otherwise.
8. **Tests**: `src/tests/<area>.test.tsx`, with shared data in `src/tests/fixtures.ts`.

## 3. Both repos

1. Build only what the ticket needs now. No parameters, options or layers "for later". Don't add a parameter just so tests can pass a value. Tests set the time with `vi.useFakeTimers({ toFake: ['Date'] })` and `vi.setSystemTime()`.
2. One home per rule, list or constant. Search before writing a new helper.
3. Fix first: if the code you must change breaks these standards, bring that part in line in a separate refactor commit (no behaviour change), then make your change.
4. Comments explain why, in plain words. No ticket IDs or plan sections in code.
5. No nested ternaries. No `any`. Avoid `!`: handle the null case.
6. File names follow their folder: backend `camelCase.ts`, React components `PascalCase.tsx`, `components/ui` `kebab-case.tsx`, tests `kebab-case.test.ts(x)`.
7. **Size is guidance, not a rule.** A function over about 80 lines, a file over about 500 lines, or deep nesting is a sign to split by responsibility. Lint shows a warning. If you keep it, say why in the PR.
8. One PR does one thing. Refactor PRs change no behaviour and no test expectations. Commit messages follow `type(scope): what changed`. No scratch files: use a git-ignored `scratch/` folder.
9. Agents: first read `AGENTS.md` and every file you will change. Follow the pattern and copy the reference file named here. Before saying "done", run typecheck, lint, format check and tests. In the PR, list how you verified the change and any exception.
