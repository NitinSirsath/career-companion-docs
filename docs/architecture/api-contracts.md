# API Contracts

This document defines cross-cutting API contracts for Career Companion. Feature-specific behavior remains in the relevant feature specification or Linear issue.

## Pagination

Collection endpoints should use the shared pagination helpers:

- `getPaginationParams()` — parses and bounds `limit` and `offset`.
- `createPaginatedResponse()` — creates the standard paginated response envelope.
- Canonical implementation: `career-companion-backend/src/utils/pagination.ts`.
- Default and maximum `limit`: 20.
- Do not introduce cursor pagination without an explicit architecture decision.

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

Ownership checks must happen before disclosing user-owned email-derived information.

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
