# API Contracts

This document defines cross-cutting API contracts for Career Companion. Feature-specific behavior remains in the relevant feature specification or Linear issue.

## Pagination

Collection endpoints should use the shared pagination helpers:

- `getPaginationParams()` — parses and bounds `limit` and `offset`.
- `createPaginatedResponse()` — creates the standard paginated response envelope.
- Canonical implementation: `career-companion-backend/src/utils/pagination.ts`.
- Default and maximum `limit`: 20.
- Do not introduce cursor pagination without an explicit architecture decision.
- **The one exception:** `GET /api/mailbox/threads` (the live Gmail inbox) pages by Gmail's own token. Gmail offers no offsets, and nothing is stored that an offset could count. It takes `pageToken` and answers `{ "items": [], "nextPageToken": null }`; `nextPageToken` is null on the last page. No other endpoint may copy this shape.

Paginated responses use:

```json
{
  "items": [],
  "metadata": {
    "nextOffset": null,
    "limit": 20,
    "offset": 0
  }
}
```

## Authentication and ownership

Authenticated API routes should use `requireAuth`.

User-owned resources must be scoped from the authenticated user identity rather than from a request-supplied `userId`.

Ownership checks must happen before returning or mutating user-owned data, including related evidence such as source emails or submissions.

Canonical examples are in `career-companion-backend/src/middleware/auth.ts` and the application service.

## Sign-in and the page to return to (22 Oct 2026)

- `GET /api/auth/me` answers 401 only when nobody is signed in. The frontend treats every other outcome (no answer, a network error, a 5xx such as a 502 while the free Render server wakes up) as "not known yet" and asks again; it never sends a signed-in user to the sign-in page because the server was slow. See UF-01 in [user flows](../product/user-flows.md).
- `GET /api/auth/connect?returnTo=<path>` (Google sign-in, prod mode only) takes the app page to open after sign-in. The parameter is optional. Only a path inside the app is kept: one `/` first, then printable ASCII with no backslash, so `//host`, `/\host` and full URLs never pass. Anything else becomes `/`. The rule lives in one place, `src/utils/returnPath.ts`, shared with `GET /api/gmail/connect?returnTo=`, which narrows it to mail pages.
- The path is kept in a signed, HttpOnly, SameSite=Lax cookie that lasts 5 minutes, like the sign-in state. `GET /api/auth/callback` checks it again, clears it, and after a successful sign-in redirects to `${FRONTEND_URL}<path>`. A failed sign-in goes to `/login?error=<code>`, with `&next=<path>` added when there was a path other than `/`. Neither ever redirects outside the app.
- The sign-in pages read `next` from their address with the same rule (and never return to a sign-in page): password and test-account sign-in open it afterwards, and the Google button passes it on as `returnTo`.

## AI providers (ADR-0001, 22 Oct 2026)

Schemas: `src/contracts/ai.ts` in the backend. A user can connect several providers, one key each; one is active and does all AI work. No response ever holds a key in any form.

| Endpoint | What it does |
|---|---|
| `GET /api/ai/settings` | The active setup in the top-level fields (provider, models, access, consent, use today, safety limit), and every connected provider in `connections`: the active one first, then the others in the order connected. Each has `provider`, `active`, `state`, `reason`, `modelId`, `failure`, `models`, `verified`, `lastCheckedAt` and `consentCurrent`. |
| `PUT /api/ai/providers/:provider` | Connects a provider, replaces its key or changes its models: `{ apiKey?, models?: { fast?, detailed? }, consentDisclosure?, makeActive? }`. A provider not connected yet needs the key and the current consent. The key and models are verified first; 422 `AI_ACCESS_REJECTED` saves nothing. The first provider, or one connected while nothing is active, becomes active; any other stays inactive unless `makeActive` is true. A model `null` follows the recommended one. Answers the settings with `verification` (`VERIFIED` or `INCONCLUSIVE`) and `verificationFailure`. |
| `POST /api/ai/providers/:provider/activate` | Makes a connected provider the active one; the one it replaces stays connected. Doing it twice changes nothing. Errors: 404 `AI_PROVIDER_NOT_CONNECTED`, 409 `AI_PROVIDER_NOT_READY` (its last check found a key, billing or model problem; `details.reason`), 409 `AI_PROVIDER_UNSUPPORTED`, 409 `AI_SETTINGS_CHANGED` (the setup changed meanwhile; try again). |
| `POST /api/ai/providers/:provider/check` | Checks a connected provider's key and models again. For an inactive provider only its own state changes. Answers with `verification` (`VERIFIED`, `REJECTED` or `INCONCLUSIVE`). 404 `AI_PROVIDER_NOT_CONNECTED`. |
| `DELETE /api/ai/providers/:provider` | Removes the key. For the active provider, `?then=<provider>` makes that connected provider active, and `then=none` (the default) leaves none active, so new mail waits. Removing a provider that is not connected changes nothing. |
| `POST /api/ai/settings/sample-test` | Runs both AI steps on a built-in sample mail with the active provider. 404 `AI_NOT_CONFIGURED` when none is active. |
| `POST /api/ai/waiting/read` | "Read waiting mails now", on the active provider. |

The old `PUT /api/ai/settings`, `DELETE /api/ai/settings` and `POST /api/ai/settings/check` are gone.

**The free AI (owner decision, 24 Oct 2026; ADR-0001 decision 1).** The catalog provider `career-companion` runs on Career Companion's own OpenRouter key, with GPT-6 Luna for both steps:
- It is in `offeredProviders` only while the server has its key and it is switched on.
- `PUT /api/ai/providers/career-companion` takes only `{ consentDisclosure, makeActive? }`. A key or a model choice is refused with 400 `VALIDATION_ERROR`, and nothing is checked with OpenRouter. It answers `verification: VERIFIED`.
- `POST /api/ai/providers/career-companion/check` answers 409 `AI_PROVIDER_HAS_NO_KEY`. Activate and remove work as for any provider.
- New access reasons (in `access.reason`, `connections[].reason`, `access.pauses[].reason` and the mail `aiStatus.reason`): `FREE_AI_BUSY` (OpenRouter busy or rate limited), `FREE_AI_BUDGET` (Career Companion's credit is used up), `FREE_AI_PROBLEM` (its key was refused or the model is gone) and `FREE_AI_OFF` (switched off). All are state `LIMITED`: they end by themselves, and the user never has to fix them. The first three are one pause shared by every user on the free AI, with no test mail; "Read waiting mails now" does not test it.
- `safetyLimit.callsPerDay` is the free AI's own limit (`AI_FREE_DAILY_CALL_LIMIT`, default 250) while it is active.

## Validation

Validate API input at the boundary with Zod.

Reusable request and response schemas belong in the backend repository's `src/contracts/` directory.

Route handlers should not duplicate validation rules that can be represented by a shared contract.

## Error contract

API errors should use the standard envelope:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {}
  }
}
```

`details` is optional.

Stable error codes should be used where clients may need to distinguish error categories. Common validation and unexpected errors should continue to flow through the central error handler.

## Idempotency and retries

Writes that can be retried and could create duplicate side effects need an explicit idempotency, uniqueness, concurrency, or equivalent domain-level strategy.

Do not add a generic idempotency layer to every endpoint without a demonstrated requirement.

## Privacy

Only persist or expose data required for the product behavior.

Gmail and email-related APIs should avoid exposing raw message bodies, credentials, or unrelated email data when structured job-search information is sufficient.

Email text is returned by exactly one endpoint, `GET /api/mailbox/threads/:threadId`, and only to the signed-in owner of the mailbox. It is fetched from Gmail for that request and never stored, cached or logged. Every `/api/mailbox` response carries `Cache-Control: no-store`. Attachments are always sent as a file download, with the file name taken from Gmail and never from the request.

Ownership checks must happen before disclosing user-owned email-derived information.

## Conversations and mailbox (mail-first walkthrough)

Schemas: `src/contracts/conversation.ts` and `src/contracts/mailbox.ts` in the backend.

| Endpoint | What it does |
|---|---|
| `GET /api/conversations` | One page of job-mail conversations (offset envelope). Filters: `q`, `request` (`open`, `replied`, `done`, `none`), `category`, `application` (`any`, `none` or an application ID), `unseen=true`, `sort` (`attention` by default, or `newest`). No email text. |
| `GET /api/conversations/:key` | One conversation with what the AI read from each of its emails. `:key` is the Gmail thread ID, or the ID of any email of the conversation. |
| `POST /api/conversations/:key/seen` | Ends the New mark. Safe to repeat. |
| `PATCH /api/conversations/:key/request` | The user's answer to the newest request: `{ emailId, state }` with `DONE` or `OPEN`. `emailId` names the request the user saw; a newer request answers 409 `REQUEST_CHANGED`. |
| `POST /api/conversations/:key/application` | Gives the conversation its application: `{}` (from what the AI read), `{ companyName?, jobTitle? }`, or `{ applicationId }`. Errors: 400 `COMPANY_REQUIRED`, 409 `APPLICATION_CHOICE_REQUIRED` (with `details.candidates`), 409 `CONVERSATION_LINKED`, 409 `CONVERSATION_NOT_LINKABLE`, 409 `APPLICATION_ARCHIVED`. Pressing twice never makes two applications. |
| `GET /api/mailbox/threads` | One page of the live Gmail inbox (token paging, see Pagination). Marks the threads that are job-mail conversations. |
| `GET /api/mailbox/threads/:threadId` | One thread read live, with each message's text, labels and attachment names. The user's own messages are included; drafts are not. Also says who a reply from the app would reach: `replyTo` (the newest mail from someone else, its Reply-To when it has one; null when nobody else wrote) and `replyCc` (the other people on that mail, without the user). |
| `POST /api/mailbox/threads/:threadId/replies` | Sends the user's plain-text reply into the thread: `{ replyId, text, toAll? }`. `replyId` is a UUID made by the page for this one reply; the same ID never sends twice. `toAll: true` also sends to `replyCc`, in Cc. The request never names an address. Errors: 403 `GMAIL_SEND_NOT_ALLOWED`, 409 `MAIL_REPLY_NO_RECIPIENT`, 502 `MAIL_REPLY_UNCONFIRMED` (the mail may have gone out; nothing is sent again by the app). |
| `GET /api/mailbox/messages/:messageId/attachments/:partId` | One attached file, as a download. |

Mailbox errors: 409 `GMAIL_NOT_CONNECTED`, 404 `NOT_FOUND`, 502 `GMAIL_UNAVAILABLE`, 413 `ATTACHMENT_TOO_LARGE`.

**Fake job offers (owner decision, 23 Oct 2026):** a conversation and each mail's AI read carry `fakeOfferWarning: string | null`, the AI's short reason when a mail offers a job and asks for money or bank, card or ID details. When it is set, the conversation's `request.text` is Career Companion's own safe step, "Don't pay. Check this job on the company's own careers site.", with no due date. The sandbox mail result carries the same field. See [Email AI pipeline](email-ai-pipeline.md), "Job offers that ask for money".

## Verification

Cross-cutting contracts should be verified with focused automated tests where practical, especially:

- ownership isolation;
- standard error responses;
- pagination bounds and response envelopes;
- boundary validation for shared request contracts.

Verification should be independent of chat history so an AI coding agent can discover and validate the contract from the repository documentation.

## Deferred on purpose

- **Rate limiting:** apply domain-specific controls where needed; universal rate limiting is not currently required.
- **Caching:** current API behavior does not require a server-side caching layer.
- **Universal transaction wrapper:** transaction boundaries belong to domain operations rather than every request.
- **Cursor pagination:** offset pagination is sufficient for the current product; cursors require an explicit design decision.
- **Static every-route-has-auth check:** authentication boundaries should be explicit and reviewed without adding a new governance or static-analysis framework.
