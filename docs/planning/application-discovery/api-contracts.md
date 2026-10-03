# Application discovery API and view contract

AD-01/02, 2026-10-03. Additive refinement of [S9 discovery](../sprint-9/api-contracts.md) and S11 archive semantics. Runtime schemas remain backend-authored and synchronized into the frontend.

## GET /api/applications

Existing normal session authentication and response shape remain. MCP bearer tokens do not authenticate this endpoint. All predicates apply to the authenticated owner before ordering and pagination.

| Query | Values/default | Meaning |
| --- | --- | --- |
| q | Trimmed string, maximum 100 characters; empty = absent | Case-insensitive literal company/title substring; %, _ and backslash escaped |
| effectiveStatus | Existing status enum, or UNKNOWN; absent = all | userStatus ?? aiStatus; UNKNOWN means both null, never a new stored status |
| submittedVia | AUTOMATION; absent = all | At least one owned LINKED/CREATED external submission; independent of effective status |
| archive | active (default), archived, all | Existing archivedAt visibility, independent of status |
| sort | added_desc (default), applied_desc, applied_asc, company_asc | Database ordering described below |
| limit / offset | Existing pagination: default/max 20, offset 0 | Existing limit+1 next-page detection; no total count |

Malformed, repeated/non-string, unsupported and overlong values return existing 400 VALIDATION_ERROR. No query values are newly logged.

Ordering:

- added_desc: createdAt DESC, id DESC (unchanged).
- applied_desc: appliedAt DESC NULLS LAST, id DESC.
- applied_asc: appliedAt ASC NULLS LAST, id DESC.
- company_asc: companyName ASC using database collation, id DESC.

Applied date is application.appliedAt. It is not event createdAt/recordedAt and is not inferred from creation time. Multiple submissions still yield one application. Linked intake fills an empty appliedAt only; a later receipt does not overwrite the application's existing date/status. Unknown applied dates sort after known dates in both applied-date orders.

Example: `/api/applications?q=engineer&submittedVia=AUTOMATION&effectiveStatus=UNKNOWN&sort=applied_desc&archive=active&limit=20&offset=0`.

Response remains `{ items: ApplicationResponse[], metadata: { limit, offset, nextOffset } }`. Current status derivation/revisions, submittedVia, bounded latest active event and non-retired pending-action count are unchanged. No sourceRecordRef, token, answers, resume or extra evidence fields are added.

NEEDS_REVIEW/IGNORED receipts are not extra application rows. Unresolved receipts stay in the existing Automation review workflow until a user creates/links an application. Archived-only matching still requires review; receipt replay retains its original identity.

## UI semantics and read lifecycle

Applications remains the sole navigation entry for the complete application lifecycle. APPLIED is a canonical status filter, not an alias for all applications or for submission-source membership. “No user/AI status” includes records whose submission is reported by automation without inventing an AI/user status. Such cards retain the existing “Applied · via automation” badge; later status takes precedence with separate “Submitted via automation” text.

Each filter and sort participates in query keys and normal API requests with AbortSignal. Filters/sort reset offset. Parent route state preserves controls/page through detail and breadcrumb/nav return; reload or leaving Applications clears it. No private search state is stored in URLs or local/session storage.

Explicit refresh rereads the visible page, including external intake; normal query focus/mount behavior also applies. No new poller, AI call, Gmail sync or MCP read. Offset pagination is not a frozen snapshot across requests. New intake can change page positions; users can choose the first page/newest-added order. Showing X–Y describes only the current returned slice, never a global total.

Acknowledged manual status/archive revisions retain their existing guards. UNKNOWN participates in membership checks. If merging newer state changes membership, the fetch rereads once; disagreement/failure remains a recoverable error. Empty later pages recover to page one without clearing filters. Read retry preserves the current controls. Successful or uncertain manual creation resets discovery to the exact default active-list key; uncertain outcomes never automatically replay POST.

## Boundaries

No schema, migration, new index, MCP tool/intake change, provider behavior, archive/retirement rewrite or new API route. Local SDK/browser fixture evidence does not replace real automation-client/owner acceptance.
