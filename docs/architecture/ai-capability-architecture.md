# AI Capability Architecture — User-Provided AI

| Field | Value |
| --- | --- |
| Status | **Accepted design (2026-10-02); implemented locally 2026-10-02, not released.** Decision record: [ADR-0001](decisions/ADR-0001-user-provided-ai.md). The implementation follows this design; the decisions this document left open (§15) are recorded in the [BYO AI plan §14](../planning/byo-ai/README.md#14-engineering-decisions-and-complexity-review) and the [execution report](../planning/byo-ai/execution-report.md). No provider is certified yet ([provider evaluation](../ai/provider-evaluation.md)). |
| Date | 2026-10-02 |
| Baseline | Backend AI layer after Sprint 6: `backend/src/services/ai/*`, `backend/src/jobs/emailProcessingJob.ts`, `backend/src/services/gmailSync.ts` |
| Implementation | After Sprint 6 is committed and ADR-0001 is accepted |

This document explains how Career Companion uses AI once each user brings their own AI account from one of several providers. It aims for a real choice of provider with a controlled scope. That means:
- a small curated set of providers
- three protocol adapters
- one configuration per user
- reuse of what already works: capability interfaces, operation ledger, bounded retries, Gmail sync recovery, sanitized errors

---

## 1. Product requirements

### 1.1 Why provider choice matters

When the user brings the AI account, the account they already have decides whether setup takes two minutes or becomes a chore. A single-vendor key form tells most users to open a new account first. It also ties Career Companion's AI quality, price, limits and data terms to one company.

But provider choice is only valuable if results stay trustworthy. Career Companion's AI output updates application state and creates actions. A provider that extracts poorly damages the product. So the choice is **curated**: every offered model is checked against Career Companion's own contracts before users can pick it.

Four product facts follow from user-provided AI:

1. **The user pays.** Career Companion must not waste calls, and must bound its own processing per user with a safety limit. That limit is an application control, not a model of the provider's quota or billing.
2. **The user fixes AI problems.** A rejected key, missing billing, an exhausted quota or a retired model needs a clear cause and a provider-specific fix.
3. **The user's content goes to the chosen provider.** Data-use and data-residency terms differ by provider and must be shown before the user chooses.
4. **Users are isolated.** One user's limits or errors never pause another user.

### 1.2 Who the users are

Experienced professionals in an active job search. They typically fall into three groups:

- **No AI developer account.** They need the lowest-friction path: a provider with a free tier and a short guide.
- **Already pays for an AI API** (OpenAI or Anthropic). They want to reuse it and choose a model.
- **Has a consumer AI subscription** (ChatGPT, Claude or Gemini apps). They need to learn that a subscription is not an API key, and where to get one.

### 1.3 Requirements

| ID | Requirement |
| --- | --- |
| R1 | A user can choose a provider from a curated list, follow provider-specific guidance to get a key, and learn immediately whether it works. |
| R2 | Each provider shows its cost model (free tier or paid billing), what is sent, and its data-use and data-residency summary before the user commits. |
| R3 | Recommended models are preselected. A user may choose among that provider's verified models; free-form model names are not accepted. |
| R4 | A definitively rejected key or inaccessible model is not saved. |
| R5 | A saved key is never shown again. It can be replaced or removed. Switching provider keeps the current setup working until the new one is verified. |
| R6 | A user's AI work uses only that user's AI access. Limits, cooldowns, usage and failures are per user. |
| R7 | Career Companion applies a per-user daily AI safety limit. It is an application safety control, independent of provider quota, billing and rate limits. Its value, bounds and any user adjustment are implementation decisions. |
| R8 | Without usable AI access, sign-in, Gmail connection and sync still work. Emails needing AI wait visibly; they are not failed. |
| R9 | When access stops working, processing pauses for that user with a provider-specific cause and fix, and resumes once fixed. |
| R10 | Switching provider, model or key never re-processes completed emails, and nothing is ever sent to a second provider without the user's choice. |
| R11 | Each AI result records which provider and model produced it. |
| R12 | Business features do not depend on keys, SDKs, protocols or provider error formats. |
| R13 | Credentials and Gmail-derived content stay within §8. |

---

## 2. Provider strategy

### 2.1 Selection criteria

A provider earns a place when it meets all four of these:

1. **Demand:** users plausibly already have an account.
2. **Reliable structured output:** it can be held to Career Companion's schemas.
3. **Coverage by an existing protocol adapter:** or it justifies a new one.
4. **Data terms Career Companion can explain honestly.**

The cost model (a free tier vs paid billing only) shapes the guidance shown to users, not eligibility.

### 2.2 Protocols, not vendors

| Protocol adapter | Covers | Structured output approach |
| --- | --- | --- |
| **Gemini** | Google Gemini API | Native response schema |
| **OpenAI-compatible** | OpenAI; later Kimi (Moonshot), DeepSeek, Mistral and similar, each a fixed catalog entry with its own base URL | JSON-schema response format where supported, otherwise JSON mode; always validated |
| **Anthropic** | Claude | Native structured output where available, otherwise a forced tool call with the schema |

Every adapter's output is validated against the same Zod schema, exactly as today. A provider with weaker native guarantees shows up as more `invalid_output` failures. The evaluation gate (§5.4) keeps such providers out until they are good enough.

### 2.3 Scope by release

| Provider | Release | Reason |
| --- | --- | --- |
| Gemini | **V1** | Existing integration; free tier gives the lowest-friction start |
| OpenAI | **V1** | Most widely held developer account; strong structured output; establishes the OpenAI-compatible adapter |
| Claude (Anthropic) | **V1** | Widely held; strong extraction; the third protocol completes the core set |
| Kimi (Moonshot), DeepSeek, Mistral | **V1.x**, one at a time | Same OpenAI-compatible adapter. Each needs evaluation and a clear data-use and residency statement before it is offered. Residency matters because the content is Gmail-derived. |
| Aggregator (for example OpenRouter) | **Later** | One key for many models is attractive, but it adds a third party to the data path and model quality varies. If offered, only as a curated entry with fixed models. |
| Azure OpenAI, Amazon Bedrock, Vertex AI | **Later, if demanded** | Credentials are endpoints, deployments or cloud IAM rather than one key. That is an enterprise shape. |
| User-entered endpoints, local or self-hosted models | **Not planned** | A hosted server cannot reach a user's local machine. Calling user-chosen URLs from the server is an SSRF and data-exfiltration risk. Quality is unverifiable. |

**Release gate:** V1 ships with every provider that has passed evaluation. Gemini is the minimum. A provider that is not ready is not shipped in a weaker form; it waits for V1.x.

**Decision (accepted 2026-10-02):** the V1 provider set is Gemini, OpenAI and Claude, under the release gate above. Kimi, DeepSeek and Mistral are V1.x candidates, each subject to evaluation and a data-use and residency decision (O6).

---

## 3. User experience

### 3.1 AI settings

AI settings sit alongside the Gmail connection: Career Companion's two accounts. The page shows one of two things.

**No configuration: provider choice**
- A card per provider, showing:
  - name
  - "free tier available" or "requires paid API billing"
  - a one-line data-use summary
  - a "recommended for" note (for example, "quickest start: free tier")
- A short note: a ChatGPT, Claude or Gemini app subscription is not an API key, with a link to where each provider issues keys.

**Configured: status card**
- provider and models in use
- access state (§3.4) and when last checked
- Career Companion's own call and token counts, labeled as such (not provider billing)
- the Career Companion safety limit in effect
- actions: Check again · Try a sample email · Change models · Switch provider · Remove

### 3.2 UF-10 — Set up or switch AI provider

*Canonical home will be [user flows](../product/user-flows.md) after Sprint 6 (§14).*

1. **Choose a provider** from the cards.
2. **Get a key: guided.** Steps for that provider, with a direct link to its key page and its billing page. If the provider needs paid billing, this is said before the key is pasted.
3. **Paste the key.** The field is a password field and is never prefilled.
4. **Models.** Recommended models are shown by role ("fast screening" and "detailed analysis"). An *Advanced* section lets the user pick from the provider's verified models only.
5. **Safety limit.** The Career Companion safety limit in effect is shown, explained as Career Companion's own safeguard, not the provider's quota.
6. **Data consent.** The user confirms they understand what is sent (bounded email metadata, and bounded body text for job-related mail) and that it goes to this provider under their own account and its terms. Consent is recorded with the provider and time.
7. **Save and verify** (§10), with no email content sent:
   - **Verified** → saved; Ready.
   - **Rejected** → not saved. The provider-specific reason and fix are shown (wrong key, billing not set up, no access to this model). The key field is cleared.
   - **Inconclusive** (provider busy, network) → saved as unverified; Ready.
8. **Optional: Try a sample email.** Career Companion runs both capabilities on a built-in synthetic recruiter email, never user data, and shows the extracted result. This proves structured output end to end for this key and model, and costs a few tokens on the user's account.
9. **Processing starts.** Emails already waiting begin processing. Step 6 and the save screen make that clear, so saving is consent.

**Switching provider** uses the same flow:
- The current configuration keeps working until the new key is verified.
- Then the old key is deleted.
- Future emails use the new provider. Completed results are not reprocessed, and each keeps its own provenance.

**Removing** deletes the configuration. Processed data stays.

### 3.3 Visible provenance

Email and application views show a small "analyzed by <provider> · <model>" label from the stored result. When results differ between providers, the user can see why.

### 3.4 AI access states

| State | Meaning | Shown | Processing |
| --- | --- | --- | --- |
| **Not set up** | No configuration | Prompt to choose a provider; count of waiting emails | Waits |
| **Ready** | Verified, or saved while verification was inconclusive | Provider, models, usage | Normal |
| **Needs attention** | Key rejected, billing or permission problem, model unavailable | Provider-specific cause and one fix (link to the provider's key or billing page, or "choose another model") | Waits until fixed |
| **Limited** | Provider refusal (rate limit, quota, overload), or Career Companion's safety limit reached | "Resumes about <time>", saying which of the two applies | Waits until it clears |

States describe **AI access**, not "a key exists", so a future Career Companion allowance fits without renaming.

### 3.5 Effect on existing flows

- **Onboarding (UF-02):** "Choose your AI provider" joins "Connect Gmail". Either order works. Users without any provider account are pointed to the free-tier option first.
- **Processing (UF-05, §4.3):**
  - "Waiting for AI" is shown from the user's access state.
  - Held emails (outcome unknown or invalid output) offer **"Retry anyway"**, which warns of a possible duplicate charge (§9.5).

---

## 4. Where this fits the current pipeline

### 4.1 Today

```
gmail-sync-job {userId} → enqueue email-processing-job {userId, emailId}
  → EmailAIPipeline.processEmail(userId, emailId)
      deterministic filter → GeminiProvider.getInstance()   ← env key, process-wide
      → runOperation(...)  claim → GLOBAL budget/cooldown → call → checkpoint
      → AIProcessingResult → matcher
```

### 4.2 After

```
email-processing-job {userId, emailId}              ← unchanged payload; never a key
  → EmailAIPipeline.processEmail(userId, emailId)
      deterministic filter (no AI)                   ← unchanged
      → resolve AI access for userId                 ← once per job
          Ready → capabilities bound to (adapter, models, key) for this user
          else  → email stays PENDING; job acknowledged; stop
      → runOperation(...)  claim → PER-USER limit/cooldown → adapter call → validate → checkpoint
                           ↳ usage counted per user (calls, tokens)
      → AIProcessingResult (provider, model provenance) → matcher   ← unchanged
```

### 4.3 Responsibilities

| Area | Owns | Does not see |
| --- | --- | --- |
| **Business features** (email processing, matching, applications, actions) | When AI is worth calling, what input to give, what results mean, email and application states, consent to process | Keys, protocols, SDKs, provider errors |
| **AI layer** (`services/ai`) | Capability contracts, catalog, access resolution, ledger, per-user limits and usage, validation, error classification, credentials, verification | Job-search meaning of results |
| **Protocol adapters** | Translating a contract into one protocol, schema translation, parameter differences, error mapping, verification calls | Users, database, email states |

The feature-facing interfaces (`RelevanceClassifier`, `EmailAnalyzer`) stay. Features never learn which adapter implements them.

---

## 5. Provider boundary

### 5.1 Provider-neutral contracts

Each capability has one Career Companion-owned definition:
- contract version
- instructions (prompt)
- output schema (Zod, the single source of truth)
- role (`fast` for screening, `detailed` for analysis)
- input limits

Prompts move out of the Gemini implementation into these definitions with **text and versions unchanged**, so no completed result is invalidated.

**Why now:**
- With three providers, prompts copied per provider would let one contract version mean different instructions.
- Today's Gemini response schema is a hand-written copy of the Zod schema; three copies would drift.

Each adapter derives its provider's schema dialect from the Zod schema. Each dialect has known limits (required fields, nullable fields, unsupported keywords). Adapter tests cover these.

### 5.2 Adapter interface

| Operation | Meaning |
| --- | --- |
| Generate structured | Run a contract's instructions and input against a model with its schema. Returns the parsed object (validated afterwards by the AI layer) and token usage. |
| Verify | Content-free check that the key authenticates and the selected models are accessible (§10) |
| Classify error | Map any provider error to Career Companion's categories (§9.3) using status and error details, without copying provider text |

Hardening lives in the adapter:
- SDK or library retries off
- per-model timeout
- output-token limit (using the right parameter name per provider)
- reasoning or thinking kept at the minimum the model allows
- low temperature where supported

These differ by provider and model, which is exactly what catalog metadata records.

**Implementation option:** a multi-provider SDK may sit **behind** this interface if it meets these criteria:
- retries can be disabled
- explicit timeouts
- access to status and error details
- token usage
- structured output with Zod v4

This choice belongs to implementation. The boundary is the same either way.

### 5.3 The catalog (in code)

One code-defined catalog lists, for each provider:
- display name
- protocol
- fixed base URL (for OpenAI-compatible entries)
- key-page and billing links
- cost model (free tier or paid)
- data-use and residency summary
- setup steps
- verified models

For each model it records:
- the roles it suits
- the recommended default per role
- capability metadata: structured-output mode, token-limit parameter, temperature support, reasoning control, timeout
- retirement date

It changes only through a release. It reaches the frontend through the existing backend→frontend contract sync, so no catalog endpoint is needed.

**Model retirement:**
- A retired model is removed from choice.
- Configurations using it switch to that provider's recommended model, and the user is told.
- The provider itself never changes without the user.

### 5.4 Evaluation gate

Every provider and model in the catalog must pass Career Companion's contract evaluation. That is a fixed set of **synthetic** recruiter and non-recruiter emails (never user data) with expected classifications and extractions, plus thresholds for:
- schema validity
- relevance accuracy
- key-field extraction accuracy

The evaluation runs manually with real keys when a catalog entry is added or changed. CI runs adapter tests against recorded responses only.

This gate is what makes provider choice safe for product quality. Its upkeep is the main ongoing cost of multi-provider support, which is why the catalog stays small.

---

## 6. AI access resolution

One function answers "can this user's AI work run now, and with what?". It returns either the capabilities bound to the user's adapter, models and key, or "unavailable" with a reason: not set up, needs attention, or limited.

- **It runs in the worker, once per job, before any ledger claim.** Unavailable access consumes no attempt and no budget.
- Classification and extraction for one email use the same resolved access, even if the user switches provider mid-job.
- **It reads by the `userId` in the job.** Keys never travel through the queue.
- Settings verification and the sample-email test use the same adapters.

This is also the only place future sources would be added (a backup provider, a Career Companion allowance).

---

## 7. Credentials and security

### 7.1 Ownership

- One active configuration per user, keyed by `userId`, cascading with the user.
- The API derives the user from the session only.
- The worker reads only the configuration of the job's `userId`.

### 7.2 Encryption

- AES-256-GCM at the application layer, the same approach as Gmail tokens.
- A dedicated encryption key for AI credentials: one more secret, but no reuse of a Gmail-named key for unrelated data, and independent rotation.

### 7.3 Where plaintext may exist

| Place | Allowed |
| --- | --- |
| Request body of save, verify or sample test | Yes; one request |
| Backend memory while verifying and encrypting | Yes; one request |
| Worker memory for one job's provider calls | Yes; one job |
| Database | No; ciphertext only |
| Queue payloads | No |
| Logs, metrics, error messages, stored error fields | No |
| API responses, including the save response | No; a fixed mask |
| Browser: stores, query cache, local/session storage, URLs | No; the password field only, cleared after submit and on unmount |

### 7.4 Network boundary

Adapters call only base URLs fixed in the catalog. No user input ever determines where the server sends a key or email content. This rules out SSRF and silent exfiltration through a user-supplied endpoint.

### 7.5 API rules

- The key is write-only.
- A blank key on update keeps the stored one.
- Reads select fields explicitly.
- Validation errors never echo the key.
- Verify and sample-test are rate-limited per user.

### 7.6 Logging and errors

Stored or logged error text is application-written, or a fixed safe message. Raw provider messages can echo content or the key, so they are never kept.

Redaction tests push a recognizable fake key through every path: save, verify, sample test, processing and failures. They assert it never appears outside the ciphertext.

**Precondition: Sprint 6 review ticket [S6-R06](../planning/sprint-6/review-2026-10-02/S6-R06-worker-error-hygiene.md).** Today the email worker rethrows the original error, and pg-boss stores it, with message, stack and properties, in `pgboss.job.output`. With several provider SDKs in the path, that would put raw provider errors in the database. S6-R06 (throw only the sanitized category) must be complete before user-provided AI ships.

---

## 8. Privacy boundaries

Career Companion keeps structured job-search information, not email content.

### 8.1 May be stored

- Email metadata already stored: IDs, sender, subject, dates.
- Validated structured AI output with provider and model provenance.
- Ledger metadata: operation, version, status, attempts, timestamps, error code.
- The configuration:
  - provider
  - encrypted key
  - model choices
  - access state and reason
  - last checked
  - consent (provider and time)
- Per-user daily counts (calls, input and output tokens) and cooldown, recorded by Career Companion; not provider billing.
- Structured logs: IDs, events, durations, token counts, error categories.

### 8.2 Never stored or logged

- the plaintext key, or any part of it
- prompts or provider request payloads
- raw provider responses (only the validated object is kept)
- raw provider error text
- email bodies, snippets or extra headers
- any key or content in queue payloads
- the sample-email test result (it is shown, then discarded)

### 8.3 Provider-side data

Content goes to the user's chosen provider under the user's own agreement. Before the user commits, the catalog's per-provider summary states:
- whether API data may be used for training or product improvement (this differs by provider and by paid or unpaid tier)
- where data is processed

Consent is recorded per provider. Switching provider asks again. Each summary is checked against the provider's current terms when the catalog entry is added or reviewed.

### 8.4 Existing exposure (unchanged)

Free-text output fields (classification `reasoning`, extraction `provenance`) can paraphrase email wording. A future contract version should give each a purpose and a length bound.

---

## 9. Execution, limits and failures

### 9.1 The execution record

The operation ledger stays as it is:
- one row per `(email, operation, version)`
- claim before call
- completed results reused
- unknown outcomes held

`AIProcessingResult` records provider and model. Logs record tokens. No new execution-log table.

### 9.2 Safety limit, counts and cooldown

- **Safety limit, per user.**
  - The existing daily call limit, keyed by user instead of global.
  - It is an application safety control: it bounds how many AI calls Career Companion makes for one user per day, so a defect or loop cannot run unbounded.
  - It does not represent, mirror or predict provider quota, billing or rate limits. Those reach Career Companion only as provider refusals (§9.3).
  - A global value of zero remains the operational kill switch.
  - The value, bounds and whether users may adjust it are implementation decisions.
- **Career Companion's own counts.** The same daily record counts the calls and input/output tokens Career Companion sent. They are diagnostics and user transparency, not billing. No currency estimates in V1.
- **Provider cooldown, per user.** Set by that user's provider refusals; grows with consecutive refusals and resets on success. The safety limit counts every call sent. Together they bound any loop.

### 9.3 Failure categories

| Category | Examples across providers | Effect | User sees |
| --- | --- | --- | --- |
| **Access: not set up** | No configuration | No claim; email pending | Choose a provider |
| **Access: key rejected** | Invalid or revoked key | Claim released, attempt not used; Needs attention; email pending | Replace key (provider key link) |
| **Access: account, billing or model** | No billing or credit, API disabled, region, model not available to this key | Same | Provider billing or permission link, or "choose another model" |
| **Access: limited** | Provider rate limit, quota exhausted or overloaded; Career Companion safety limit reached | Claim released, attempt not used; per-user cooldown, or wait for the next day for the safety limit; email pending | Resumes about <time> |
| **Outcome unknown** (existing) | Timeout, connection lost | Held | "Retry anyway" (§9.5) |
| **Invalid output** (existing) | Empty, non-JSON, schema-invalid | Failed | "Retry anyway" (§9.5) |
| **Invalid request** (existing) | Request rejected for a non-credential reason | Failed; engineering fix | Generic failure |

**Providers signal these differently**, and the details must be confirmed for each adapter with real keys. Examples:
- an invalid key as HTTP 400 vs 401
- "out of credit" and "rate limited" sharing 429 with different error codes
- a dedicated "overloaded" status

Each adapter owns a tested mapping table.

### 9.4 Why refusals no longer use attempts

The stabilization rule "at most three attempts for explicit 429/503 rejections" bounded spend on a shared key. With user keys, a refusal costs nothing, and the per-user cooldown and limit bound the loop. Under the old rule, a user who hits a provider quota would exhaust all attempts in minutes and strand emails for an operator who cannot see their account.

Unknown and invalid outcomes keep the conservative handling.

### 9.5 Retry approved by the payer

Held operations are never replayed automatically. The user may approve one more attempt after being told it may be charged twice. The approval is recorded on the operation.

### 9.6 Resuming waiting work

"Waiting for AI" is simply `PENDING`. Gmail sync already re-offers up to 100 pending emails per user on every manual or scheduled sync, so:
- a cleared cooldown resumes on the next sync (no new scheduler)
- fixing access (save, or a successful "Check again") triggers the same bounded re-offer immediately
- jobs for a user without usable access return without calling a provider and without using queue retries

---

## 10. Verification and sample test

### 10.1 Verify (on save, and "Check again")

- **Content-free:** the provider's model lookup for each selected model. Gemini, OpenAI and Anthropic all expose one. No tokens, no email data, not an AI operation, not counted in the limit.
- **Three results:**
  - verified
  - rejected (with a category and the provider-specific fix)
  - inconclusive (never overwrites a known-good state)
- **Stored** on the configuration as access state and last-checked time. Real calls also update the state: access problems set Needs attention or Limited, and a success clears it.

### 10.2 Sample test (optional, user-initiated)

- Runs both capabilities on one built-in synthetic email through the real adapter and validation.
- Shows the result, then discards it. Not written to the ledger.
- Counts toward Career Companion's counts and the safety limit, because it is a real call.
- **Why it exists:** model lookup proves access, not structured output for this key and account. The sample test is the user's proof, and a support tool.

---

## 11. Data and API implications

These are conceptual. Exact schema and routes are decided during implementation.

| Item | Change |
| --- | --- |
| AI configuration | **New**, one per user: provider, encrypted key, optional model per role, access state with reason, last checked, consent (provider, time), timestamps. A per-user safety-limit value only if implementation makes it adjustable. |
| Daily safety limit and counts | **Changed:** the existing daily mechanism keyed by user, counting calls, input tokens, output tokens and cooldown |
| `AIProcessingResult` | Unchanged; provider and model now come from the resolved adapter and model |
| `AIOperation` | Unchanged, except a marker recording the user's approval for a retry |
| Email processing states | Unchanged; waiting is `PENDING`, with the reason from the user's access state |
| Catalog | Code only; synced to the frontend with contracts |
| Hosted Gemini key (env and secret), model env variables | Removed; defaults come from the catalog |

**API:**
- **One AI settings resource for the session user:**
  - read (masked, with usage)
  - save (verifies first)
  - check again
  - sample test
  - remove
- **Email retry** gains an explicit "accept possible duplicate charge" confirmation.

---

## 12. Reference architecture: kept, simplified, rejected

Test applied to every pattern: *what problem does it solve for Career Companion?*

**Kept**

| Pattern | Problem it solves here |
| --- | --- |
| Features depend on capabilities, not providers | Several providers must not reach job-search logic |
| Provider knowledge as data: key and billing links, structured-output mode, token parameter, temperature support, timeout, retirement | Three protocols and many models differ exactly here; guidance and fixes need the links |
| Verified-model lists per provider | Users must only pick models that meet our contracts |
| Encrypted, write-only, never-echoed credentials | The key controls the user's billing account |
| Verify before saving, three results | Wrong keys or missing billing would otherwise fail every email later |
| Access state stored and updated by real calls | Pause and resume per user; show the cause |
| Classified errors with remediation links (credential, permissions, billing) | The user fixes access problems |
| Metadata-only execution records | Gmail content must not reach diagnostics |

**Simplified**

| Reference | Here |
| --- | --- |
| Database catalog with API | Code catalog, synced with contracts |
| Separate credential connection plus AI provider configuration | One configuration per user |
| Enabled-model lists, ordering and tenant default | Recommended model per role; optional pick from the verified list |
| Two tests (key, then model completion) | Content-free verify, plus an optional synthetic sample test |
| Large failure vocabulary | Existing error types plus one access-problem category with four reasons |
| Execution-log table with revisions | Existing ledger, result provenance and per-user daily counts |

**Rejected**

| Pattern | Why not |
| --- | --- |
| Separate AI and Integrations applications | One user owns both credentials and settings |
| Tenant roles and permissions | No tenants or administrators |
| Free-form models; custom base URLs and headers | Unverifiable quality; server-side requests to user-chosen URLs |
| Several provider configurations active per user | One active account meets the requirements; a backup provider is later |
| Per-feature model routing; feature and trigger switches | Two email capabilities, no administrator |
| Prompt/schema fingerprints; selection and policy revisions | Explicit contract versions already identify results |
| Usage reservations, queues, token budgets | A per-user safety limit and cooldown suffice |
| Separate asynchronous AI-run system | pg-boss and the ledger already provide it |
| Admin dashboards and log explorers | The status card's counts are enough |

**Needed here that the reference does not address:**
- consumer-subscription confusion
- per-provider data consent
- the evaluation gate for product quality
- waiting as pending with resume through sync
- the user as payer (refusal rule, user-approved retry, per-user safety limit)

---

## 13. V1 and later

| Capability | V1 | V1.x | Later | Not planned |
| --- | :-: | :-: | :-: | :-: |
| Gemini, OpenAI, Claude (each after passing evaluation) | ● | | | |
| Kimi, DeepSeek, Mistral via the OpenAI-compatible adapter | | ● | | |
| Aggregator as a curated entry | | | ● | |
| Azure / Bedrock / Vertex credentials | | | ● (if demanded) | |
| User-entered endpoints, local models, free-form model names | | | | ● |
| Provider cards with cost model, data-use summary, guided key steps | ● | | | |
| Recommended models per role; pick from verified models | ● | | | |
| Content-free verify; sample-email test | ● | | | |
| Per-provider data consent | ● | | | |
| Per-user safety limit; Career Companion call and token counts | ● | | | |
| Provenance label on results | ● | | | |
| Switch provider (verify-then-replace) | ● | | | |
| Waiting as pending; refusal rule; per-user cooldown | ● | | | |
| User-approved retry for held operations | ● | | | |
| Currency cost estimates | | | ● | |
| Several saved keys; user-chosen backup provider | | | ● | |
| Explicit reprocessing with a new provider (cost shown first) | | | ● | |
| Per-capability provider routing | | | ● | |
| Career Companion-provided allowance | | | ● | |
| Automatic cross-provider fallback | | | | ● |

---

## 14. Implementation work (later)

This is not a plan of record. It names the work so it can be scheduled after Sprint 6.

**Preconditions:**
- Sprint 6 committed.
- Sprint 6 review ticket S6-R06 (sanitized errors in queue storage) complete (§7.6).
- ADR-0001 accepted (done 2026-10-02).

1. **Behavior-preserving seam.**
   - Provider-neutral contracts (prompts and Zod moved out of Gemini, unchanged).
   - Adapter interface.
   - Gemini adapter built from a given key.
   - Access resolution still backed by the environment key.
   - Tests and smoke harness moved off `GeminiProvider.getInstance()`.
2. **Evaluation set and runner** (synthetic emails, thresholds). Baseline the current Gemini models first.
3. **Catalog** and the access-problem error category; Gemini error mapping confirmed with a real key.
4. **Data:** AI configuration; per-user safety limit, counts and cooldown.
5. **AI settings API:** encryption with a dedicated key, verify, sample test.
6. **Switch to user access:**
   - remove the environment key and model variables
   - waiting as pending
   - the refusal rule
   - re-offer on fix
7. **User-approved retry** for held operations (§9.5).
8. **Frontend:** provider cards, guided setup, consent, models, safety limit and counts, status card, provenance labels, waiting prompts, "Retry anyway".
9. **OpenAI-compatible adapter + OpenAI entry; Anthropic adapter + Claude entry.** Each is officially supported only after passing evaluation and confirming its error mapping.
10. **Operations and documentation:**
    - secrets (add the encryption key, retire the Gemini key)
    - rollout (drain old workers; existing users choose a provider)
    - backend `STABILIZATION.md`, `.env.example`
    - the Sprint 6-owned documents below
11. **Verification:**
    - isolation, redaction, safety limit and counts
    - adapter mapping tests
    - real-queue smoke with a fake adapter
    - one supervised live run per V1 provider

**Documents Sprint 6 is editing. Update them in step 10, not now:**

| Document | Change |
| --- | --- |
| [User flows](../product/user-flows.md) | Add UF-10 (§3.2); onboarding and processing changes (§3.5) |
| [Domain model](../domain/domain-model.md) | AI configuration and per-user safety limit concepts; result provenance; privacy wording from §8 |
| [MVP architecture](mvp-architecture.md) | §3 user-provided AI from curated providers and the per-user safety limit; §7 replace "Other AI Providers" with the release scope in §13 |
| [High-level architecture](high-level-architecture.md) | §3.6/§6/§7 AI access and adapters; §9.7 secrets; COM-26 section links here and corrects two stale claims (there is no provider factory; the Gemini schema is a hand-written copy, not generated from Zod) |
| [Email AI pipeline](email-ai-pipeline.md) | Per-user safety limit, counts and cooldown; waiting as pending; refusal and user-approved retry rules |

---

## 15. Accepted, and left to implementation

**Accepted V1 architecture (ADR-0001, 2026-10-02):**
- **Providers:** Gemini, OpenAI and Claude, through protocol adapters. Each is officially supported only after passing Career Companion's synthetic AI evaluation.
- **Contracts:** provider-neutral, owned by Career Companion.
- **Setup:** one active AI provider setup per user. Credentials are encrypted and write-only (§7), and the privacy lists in §8 apply.
- **Catalog:** providers and models are limited to Career Companion's code-defined catalog with fixed base URLs. The recommended model is the default; Advanced selection is limited to tested supported models.
- **Safety limit:** a per-user Career Companion AI safety limit, which is an application control, not provider quota or billing. Cooldowns are per user.
- **Unavailable AI:** emails remain `PENDING` and resume through the existing processing flow.
- **Retries:** provider credential and refusal errors do not consume the normal email retry attempts. User-approved retry is the recovery mechanism for operations with uncertain outcomes.
- **No automatic provider fallback.**
- **Excluded:** an enterprise marketplace, tenant/RBAC, a database catalog, per-feature routing, usage reservations, a separate AI/Integrations application, and new execution-log tables.

**Implementation decisions (open within the accepted architecture):**

| # | Decision | Starting recommendation |
| --- | --- | --- |
| O1 | Evaluation set size and thresholds | About 30 synthetic emails covering every category and key fields; thresholds from the current Gemini baseline |
| O2 | Provider-specific error mappings | Confirm with real keys for each adapter before the provider is officially supported |
| O3 | Data-use and residency statements per provider | Write from current terms at catalog entry; review when entries change |
| O4 | Safety-limit value and bounds, and whether users may adjust it | Set from expected daily email volume; an application safeguard, not a quota estimate |
| O5 | SDK or library behind the adapter interface | Decide in step 1 against the §5.2 criteria |
| O6 | Whether V1.x providers with non-EU/US data residency are offered at all | Product decision before each entry; disclose clearly if offered |
| O7 | Authoritative repository and Sprint 6 commit | Required before an implementation branch |
| O8 | Who relies on the hosted key in production today | Confirm; they choose a provider at rollout |
| O9 | Catalog contents, default models, final UI wording | Decide during implementation and evaluation |

---

## Document Versioning

| Version | Date | Notes |
| --- | --- | --- |
| 0.1 | 2026-10-02 | First proposal (Gemini-only, hosted-fallback assumptions removed later). |
| 0.2 | 2026-10-02 | Critical review: removed catalog, model choice, ledger additions and formal error vocabulary; waiting reuses `PENDING`. |
| 0.3 | 2026-10-02 | Provider experience re-evaluated. Added a curated multi-provider set (Gemini, OpenAI, Claude in V1; OpenAI-compatible providers such as Kimi in V1.x), protocol adapters, provider-neutral contracts, code catalog with capability metadata, evaluation gate, guided setup, per-provider consent, sample test, usage and personal limit, provenance labels, fixed base URLs. Kept the single active configuration and all v0.2 simplifications not affected by provider choice. |
| 0.4 | 2026-10-02 | V1 provider set (Gemini, OpenAI, Claude, evaluation-gated) accepted. Added S6-R06 as a precondition. |
| 0.5 | 2026-10-02 | ADR-0001 accepted in full. The per-user limit is defined as an application safety control, separate from provider quota and billing; its value and adjustability are left to implementation. User-approved retry moved into V1. §15 lists the accepted architecture and the implementation decisions. |
