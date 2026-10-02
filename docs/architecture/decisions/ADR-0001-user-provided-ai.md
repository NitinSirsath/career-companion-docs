# ADR-0001 — Career Companion uses user-provided AI from a curated set of providers

| Field | Value |
| --- | --- |
| Status | **Accepted, 2026-10-02. Implemented locally 2026-10-02, not released.** Reviewed under [Project Constitution — Change Management](../../../PROJECT_CONSTITUTION.md). All three providers are `hidden` until certified ([provider evaluation](../../ai/provider-evaluation.md)). See the [BYO AI execution report](../../planning/byo-ai/execution-report.md). |
| Date | 2026-10-02 (proposed, revised after review and accepted the same day; design history in the [design document's versioning table](../ai-capability-architecture.md#document-versioning)) |
| Decides | Whose AI access Career Companion uses, which providers it supports and how, where the provider boundary sits, how credentials are protected, and how AI failures affect processing |
| Detail | [AI capability architecture](../ai-capability-architecture.md) |

## Context

Career Companion's value depends on AI. Without it, the product can sync Gmail and store metadata, but cannot tell which emails matter or what the user must do next.

Today every AI call uses one server-side Gemini key. That shared key explains three current properties:

- Career Companion pays for every user's AI work.
- A single global call ceiling and a single global cooldown apply. One user's volume or rate-limit response affects everyone.
- AI failures are operator problems.

The product direction is that users bring their own AI account. People hold accounts with different providers: Google (Gemini), OpenAI, Anthropic (Claude), Moonshot (Kimi) and others. A product that accepts only one vendor's key turns "bring your own AI" into "go and open a Gemini account". It also ties the product's AI quality, price and availability to one company.

Supporting many providers has a real cost, though:

- Providers differ in API shape, structured-output support, error formats, limits, model churn and data-use terms.
- Career Companion's AI output drives application state, so a provider that returns lower-quality extraction harms the product, not only the user's bill.

A mature internal multi-tenant AI platform was studied as a reference. Its code-defined knowledge about providers and models is sound and transfers well:
- key and billing links
- structured-output mode
- parameter differences
- default timeouts
- retirement dates

Most of its other structure answers tenant and administrator needs Career Companion does not have:
- database catalogs and their APIs
- per-tenant provider sets
- per-feature routing
- usage reservations
- separate configuration apps

## Decision

1. **AI work for a user runs only on that user's own AI access.** There is no Career Companion key, and no fallback to one. One user's access, limits and failures never affect another user.

2. **Users choose from a curated set of providers and models.**
   - The set is defined in code.
   - A provider or model enters the set only after it passes Career Companion's contract evaluation: a fixed, synthetic test set for each AI capability.
   - Users cannot enter arbitrary model names or endpoints.

   Curation is what lets Career Companion promise consistent results across providers.

3. **Providers are integrated per API protocol, not per vendor.** Three protocols cover the providers that matter:
   - Gemini
   - Anthropic Messages
   - OpenAI-compatible chat completions, which covers OpenAI and many others such as Kimi, each through a fixed catalog entry

   A new provider that speaks an existing protocol is a catalog entry plus evaluation, not new integration code.

4. **Career Companion owns its AI contracts, and they are provider-neutral.**
   - Each capability has one versioned definition: prompt, output schema and input limits.
   - Adapters only translate that definition into their protocol.
   - A contract version always means the same instructions and schema, whichever provider ran it. The ledger identity `(email, operation, version)` and the no-replay rules depend on this.
   - Business features keep depending on capabilities (relevance classification, email analysis), never on providers.

5. **Each user has one active AI configuration.**
   - provider
   - encrypted key
   - the recommended model per role by default, or an Advanced choice limited to that provider's tested supported models
   - access state

   Switching provider replaces the configuration after the new key is verified. It affects future work only.

6. **Credentials are bearer secrets of the user's billing account.**
   - Encrypted at rest with a key dedicated to AI credentials.
   - Write-only through the API and never shown again.
   - Decrypted only to make a provider call or verification.
   - Never placed in job payloads, logs, errors, API responses or browser storage.

7. **Career Companion applies a per-user AI safety limit.**
   - It is an application safety control: it bounds how many AI calls Career Companion makes for one user per day, so a defect or loop cannot run unbounded.
   - It does not represent, mirror or predict the provider's quota, billing or rate limits. Those stay with the provider and reach Career Companion only as provider refusals (decision 9).
   - The existing global daily ceiling and global provider cooldown become per user, so one user never affects another.
   - Career Companion's own call and token counts are records of what Career Companion sent, not provider billing data.

8. **Missing or unusable AI access is a waiting condition, not a processing failure.** Emails that need AI stay pending. The reason lives once, on the user's configuration. Processing resumes after the user fixes access or the limit clears.

9. **Provider credential and refusal errors do not consume the normal email retry attempts.** A refusal (rejected key, account or permission problem, provider limit) is a rejection made before any work is done, so it is not charged. It pauses or cools down that user's AI, and the email stays pending. Calls whose outcome is unknown, or that returned unusable output, keep the current conservative handling.

10. **User-approved retry is the recovery mechanism for held operations.** Operations held because their outcome is uncertain are never replayed automatically. The user who pays may explicitly approve one more attempt, accepting a possible duplicate charge.

11. **There is no automatic cross-provider fallback.** Sending a user's email content to a second provider requires that user's explicit choice.

12. **The existing operation ledger, processing result (with provider and model provenance) and structured logs remain the execution record.** They hold metadata and validated output only, never prompts, raw responses or email content.

## Accepted V1 scope

- **Providers:** Gemini, OpenAI and Claude. Each becomes officially supported only after passing Career Companion's synthetic AI evaluation.
- **Setup:** one active AI provider setup per user. Credentials are encrypted and write-only.
- **Catalog:** providers and models are limited to Career Companion's supported catalog. The recommended model is used by default; Advanced selection is limited to tested supported models.
- **Safety limit:** a per-user Career Companion AI safety limit, which is an application control, not provider quota or billing.
- **Unavailable AI:** emails that cannot be processed because AI is unavailable remain `PENDING` and resume through the existing processing flow.
- **Retries:** provider credential and refusal errors do not consume the normal email retry attempts. User-approved retry is the recovery mechanism for operations with uncertain outcomes.
- **No automatic provider fallback.**

## Not part of this decision

- An enterprise provider marketplace.
- A tenant or RBAC system.
- A provider or model catalog in the database.
- Per-feature or per-capability routing.
- A usage reservation system or token budgets.
- A separate AI or Integrations application.
- User-entered endpoints, base URLs, custom headers or free-form model names.
- Several saved configurations or backup providers.
- A separate execution-log store.
- Hosted AI.

## Left to implementation

Decided during implementation, within this decision:
- the exact safety-limit value and bounds, and whether users may adjust it
- provider-specific error mappings
- SDK or library choice behind the adapter boundary
- catalog contents and default models
- evaluation set size and thresholds
- final UI wording

## Consequences

**Positive**
- Users bring the AI account they already have. The product is not tied to one vendor's quality, price or terms.
- Three protocol adapters cover most of the market. Adding an OpenAI-compatible provider is mostly catalog and evaluation work.
- Users see and fix their own AI problems. A per-user safety limit bounds Career Companion's own processing.
- The Gmail pipeline, matching, domain promotion and application-status semantics are unchanged.

**Negative**
- Three adapters, cross-provider schema translation and per-provider error mapping are real engineering work. The first release is larger than a Gemini-only change.
- **Every supported model needs contract evaluation, and the catalog needs upkeep.** Providers retire models frequently, so supported models must be re-checked. This is the main ongoing cost, and it is why the set stays small.
- Results can differ between providers. Provenance on each result makes this visible.
- Career Companion stores a third-party secret for every user. Data-use and data-residency terms differ by provider and must be disclosed before a user chooses.
- Two stabilization rules change for user-funded calls (decisions 9 and 10).

## Alternatives considered

| Alternative | Reason rejected |
| --- | --- |
| Keep the hosted key | Product funds all users; global limits couple users |
| Gemini-only key form | Forces users onto one vendor; ties product quality and availability to it |
| Any OpenAI-compatible endpoint and model, user-entered | Unverified output quality; server-side requests to user-chosen URLs (SSRF and data-exfiltration risk); unbounded support burden |
| Aggregator-only (one key for many models via a third party) | Adds a third party to the Gmail data path; model quality varies; may be offered later as one curated entry |
| Adopt the reference platform's structure (database catalogs, per-tenant multi-provider sets, enabled-model lists, per-feature routing, revisions, reservations, separate integrations app) | Solves tenant and administrator problems Career Companion does not have |
| Automatic fallback to another provider on failure | Moves user content to another account without consent; mixes result quality silently |
| Per-vendor adapters for every provider | Most vendors share the OpenAI-compatible protocol; per-protocol adapters give the same coverage with a fraction of the code |

## Future evolution

These are not designed now. The decision keeps each one additive.

- **More providers on existing protocols:** catalog entry + evaluation + data-use disclosure.
- **Backup provider or several saved keys:** more configurations per user plus an explicit user choice. Access resolution is the single place this would change.
- **Career Companion-provided allowance:** a second source in access resolution, plus its own limit scope. States already speak of "AI access" rather than "key present".
- **Explicit reprocessing with a new provider:** a user-initiated replay, built on the user-approved retry rule.
