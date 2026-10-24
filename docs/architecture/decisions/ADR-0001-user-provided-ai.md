# ADR-0001 — Career Companion uses user-provided AI from a curated set of providers

| Field | Value |
| --- | --- |
| Status | **Accepted, 2026-10-02. Implemented locally 2026-10-02, not released.** Reviewed under [Project Constitution — Change Management](../../../PROJECT_CONSTITUTION.md). All providers are `hidden` until certified ([provider evaluation](../../ai/provider-evaluation.md)). See the [BYO AI execution report](../../planning/byo-ai/execution-report.md). Amended by the owner: 2026-10-17 (OpenRouter, decision 3), 2026-10-22 (several connected providers, decision 5) and 2026-10-24 (the free AI, decision 1). |
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

   **Owner decision, 2026-10-24: the free AI.** A user may choose "Career Companion free AI" instead of a key of their own. It is the one Career Companion key: the owner's OpenRouter key, set only on the server (`FREE_AI_OPENROUTER_KEY`), never stored per user, logged or sent to the browser. The rules:
   - **Opt-in.** The user connects it with one button and a confirm dialog that says what is sent and through whose account. Nothing is sent before they agree. It is never switched on by itself; a new user is taken to the AI providers page after connecting Gmail, where their own key is shown as recommended.
   - **Fixed model.** GPT-6 Luna for both steps, on OpenRouter's servers that keep nothing. The user cannot change the model.
   - **Budget.** About $10 a month, enforced by the key's own credit limit on OpenRouter.
   - **Shared.** It is one provider, `career-companion`, in the "several connected, one active" model of decision 5. Because every user on it shares one key, one pause per model is shared by all of them: a busy or rate-limited answer pauses it for everyone and ends by itself. A used-up credit, a refused key or a missing model pauses it for an hour, tells the owner once, and is never shown as the user's problem. This is the one place where users affect each other, and only users who chose the free AI.
   - **Its own limit.** About 200 mails a day per user (250 calls; decision 7). Users on their own key keep theirs.
   - **Off switch.** `FREE_AI_ENABLED=false` stops it for everyone; its users' mails wait, and they can connect their own key.
   - **No automatic fallback** (decision 11 stands): when a user's own key fails, the app offers "Use Career Companion free AI instead", and the user chooses.

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

   **Owner decision, 2026-10-17:** OpenRouter is a fourth provider with its own small adapter (protocol `openrouter`). It is the one exception to "per protocol, not per vendor", for three reasons:
   - Its model lists are public, so a model lookup proves nothing about a key. The key check reads the key's own record, then OpenRouter's list of servers that keep nothing.
   - Its error codes mean other things than OpenAI's, so it has its own error table.
   - Every request must ask for private servers: `provider: { zdr: true, require_parameters: true }`.

   The rule for a usable answer is shared with the OpenAI adapter in one place. OpenRouter may switch to another server of the same model, never to another model (decision 11). Its models change only through the catalog plus an evaluation run, and it stays `hidden` until certified.

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

   **Owner decision, 2026-10-22:** a user may connect several providers, with at most one key per provider. Exactly one is active and does all AI work: sorting, reading, retries, waiting mails and the sample test. The others are saved with their keys and are called only for a check, a key change or a model change that the user starts. The user switches by hand ("Make active"); a switch affects future work only, and mails waiting for AI go to the new active provider. Decision 11 stands: nothing switches by itself when the active provider fails. Consent is given per provider when it is connected. Career Companion's safety limit and its counts stay per user, for all providers together. Removing the active provider asks which connected provider reads new mail next, or none (new mail waits).

   How it is stored: the active setup stays in `ai_configurations` (one row per user, so access resolution, pauses and the revision guard are unchanged). Each other connected provider has one row in `ai_saved_providers` (user, provider). Making a provider active swaps the two setups in one transaction, increments the revision, and ends the old setup's model pauses.

6. **Credentials are bearer secrets of the user's billing account.**
   - Encrypted at rest with a key dedicated to AI credentials.
   - Write-only through the API and never shown again.
   - Decrypted only to make a provider call or verification.
   - Never placed in job payloads, logs, errors, API responses or browser storage.

7. **Career Companion applies a per-user AI safety limit.**
   - It is an application safety control: it bounds how many AI calls Career Companion makes for one user per day, so a defect or loop cannot run unbounded. On the free AI (decision 1, 2026-10-24) it is smaller, `AI_FREE_DAILY_CALL_LIMIT` (default 250 calls, about 200 mails), and it also protects Career Companion's budget.
   - It does not represent, mirror or predict the provider's quota, billing or rate limits. Those stay with the provider and reach Career Companion only as provider refusals (decision 9).
   - The existing global daily ceiling and global provider cooldown become per user, so one user never affects another.
   - Career Companion's own call and token counts are records of what Career Companion sent, not provider billing data.

8. **Missing or unusable AI access is a waiting condition, not a processing failure.** Emails that need AI stay pending. The reason lives once, on the user's configuration. Processing resumes after the user fixes access or the limit clears.

9. **Provider credential and refusal errors do not consume the normal email retry attempts.** A refusal (rejected key, account or permission problem, provider limit, busy provider) is a rejection made before any work is done, so it is not charged. It pauses or cools down that user's AI, and the email stays pending. Calls that returned unusable output keep the current conservative handling. Calls with no answer in time follow decision 10.

   **Owner decision, 2026-10-14:** a short server error counts as a busy provider: the email waits and is tried again, and no attempt is used. Busy means Google 503, 500 and 502; OpenAI 500, 502, 503 and 504; Anthropic 500, 503, 504 and "overloaded" (529); and 409 from OpenAI or Anthropic. OpenRouter 500 and 502, 503 with `provider_overloaded`, and a 200 answer whose error is `provider_overloaded` or `provider_unavailable` (not yet seen live, 2026-10-23). Google's own 504 is not busy: it is "no answer in time" (decision 10).

10. **User-approved retry is the recovery mechanism for held operations.** The user who pays may explicitly approve one more attempt, accepting a possible duplicate charge.

    **Owner decision, 2026-10-14:** for per-email reads, a call with no answer in time (our own timer, Google's 504 "deadline expired", or a connection that dropped after sending) uses 1 of 3 attempts, and Career Companion tries again by itself. After the 3rd attempt the operation is held and waits for the user's OK. This replaces the earlier rule that an uncertain outcome is always held at once. Batch triage keeps its own rule: a batch whose outcome is uncertain is held and never replayed automatically.

11. **There is no automatic cross-provider fallback.** Sending a user's email content to a second provider requires that user's explicit choice.

12. **The existing operation ledger, processing result (with provider and model provenance) and structured logs remain the execution record.** They hold metadata and validated output only, never prompts, raw responses or email content.

## Accepted V1 scope

- **Providers:** Gemini, OpenAI and Claude. Each becomes officially supported only after passing Career Companion's synthetic AI evaluation. **Owner decision, 2026-10-17:** OpenRouter joins on the same terms, with two small models, and stays `hidden` until certified.
- **Setup:** one active AI provider setup per user. **Owner decision, 2026-10-22:** other providers can be connected too, one key each, and wait until the user makes one active (decision 5). Credentials are encrypted and write-only.
- **Catalog:** providers and models are limited to Career Companion's supported catalog. The recommended model is used by default; Advanced selection is limited to tested supported models.
- **Safety limit:** a per-user Career Companion AI safety limit, which is an application control, not provider quota or billing.
- **Unavailable AI:** emails that cannot be processed because AI is unavailable remain `PENDING` and resume through the existing processing flow.
- **Retries:** provider credential and refusal errors do not consume the normal email retry attempts. User-approved retry is the recovery mechanism for operations with uncertain outcomes.
- **No automatic provider fallback.**
- **Free AI (owner decision, 2026-10-24):** Career Companion's own OpenRouter key with GPT-6 Luna, opt-in, free for every user, with a shared pause and its own daily limit (decision 1).

## Not part of this decision

- An enterprise provider marketplace.
- A tenant or RBAC system.
- A provider or model catalog in the database.
- Per-feature or per-capability routing.
- A usage reservation system or token budgets.
- A separate AI or Integrations application.
- User-entered endpoints, base URLs, custom headers or free-form model names.
- Backup providers that take over by themselves. (Several saved configurations are part of decision 5 since the owner decision of 2026-10-22; switching stays the user's choice.)
- A separate execution-log store.
- Hosted AI as the default. (The opt-in free AI is part of decision 1 since the owner decision of 2026-10-24.)

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

**Owner decision, 2026-10-17:** the aggregator is now offered as one curated entry with fixed models (OpenRouter, decision 3). It is also the one per-vendor adapter, because its key check, errors and request rules differ from the protocol it speaks.

## Future evolution

These are not designed now. The decision keeps each one additive.

- **More providers on existing protocols:** catalog entry + evaluation + data-use disclosure. (Owner decision, 2026-10-17: a provider whose key check, errors or request rules differ gets a small adapter of its own, as OpenRouter did.)
- **Backup provider or several saved keys:** more configurations per user plus an explicit user choice. Access resolution is the single place this would change. **Done, owner decision 2026-10-22** (decision 5): several connected providers, one active, switched by hand. Access resolution did not change: it still reads the one active setup.
- **Career Companion-provided allowance:** a second source in access resolution, plus its own limit scope. States already speak of "AI access" rather than "key present". **Done, owner decision 2026-10-24** (decision 1): the free AI. Access resolution reads Career Companion's key for the provider `career-companion`; its limit is `AI_FREE_DAILY_CALL_LIMIT`, and its pause is shared by its users.
- **Explicit reprocessing with a new provider:** a user-initiated replay, built on the user-approved retry rule.
