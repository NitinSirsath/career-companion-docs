# User Flows — Career Companion

> **Status:** Finalized
> **Linear Issue:** COM-4 — Design User Flows
> **Last Updated:** 2026-10-02 (UF-11 to UF-13 added for ADR-0002)

---

## 1. Purpose

This document defines the critical user and system flows for the Career Companion MVP.

The flows describe behavior and outcomes rather than UI screens or implementation details. They establish the expected product behavior before API, data-model, and UX implementation begins.

Career Companion's core job is to turn relevant Gmail recruitment communication into a reliable, actionable representation of the user's job search.

---

## 2. Flow Conventions

Each flow defines:

- **Entry point** — how the flow begins
- **User intent** — what the user is trying to accomplish
- **Main flow** — expected successful path
- **End state** — resulting product state
- **Edge/error states** — important non-happy paths

System-initiated processing may occur without direct user interaction when new or synchronized Gmail messages are available.

---

## 3. Critical MVP Flows

### UF-01 — Sign In with Google

**Entry point:** User opens Career Companion while unauthenticated.

**User intent:** Securely access their Career Companion account.

**Main flow:**

1. User selects **Sign in with Google**.
2. Career Companion redirects the user to Google authentication.
3. User authenticates with Google.
4. Google returns the authentication result.
5. Career Companion creates or retrieves the user's application account.
6. User is redirected to the authenticated application.
7. If Gmail is not connected, the product presents the first-run/onboarding path.

**End state:** User is authenticated and can continue onboarding or access the dashboard when setup is already complete.

**Edge/error states:**

- User cancels authentication.
- Google authentication fails.
- Authentication callback is invalid or expired.
- Existing account cannot be loaded.
- Session creation fails.

Career Companion must not treat a failed or cancelled authentication as a successful sign-in.

---

### UF-02 — First-Run / Onboarding

**Entry point:** Authenticated user has not completed initial Career Companion setup.

**User intent:** Understand the setup requirements and connect the Gmail account needed by the product.

**Main flow:**

1. User enters the first-run experience.
2. Career Companion explains that Gmail access is required for the MVP.
3. User chooses to connect Gmail.
4. User completes the Gmail authorization flow described in UF-03.
5. Initial synchronization begins.
6. User is shown the sync/processing state and can proceed when the product is ready.
7. User reaches the dashboard.

**End state:** User has completed the required MVP setup and can use Career Companion.

**Edge/error states:**

- User leaves onboarding before connecting Gmail.
- Gmail authorization is denied.
- Initial sync fails or is only partially completed.
- User returns later and resumes setup without creating duplicate connection state.

---

### UF-03 — Connect Gmail and Perform Initial Sync

**Entry point:** Authenticated user chooses to connect Gmail during onboarding or later.

**User intent:** Allow Career Companion to access the Gmail data required to understand their job search.

**Main flow:**

1. User chooses to connect Gmail.
2. Career Companion requests the minimum required Google/Gmail permissions.
3. User grants consent.
4. Career Companion securely stores the authorization required for future Gmail access.
5. Career Companion starts the initial synchronization.
6. Gmail messages are retrieved in manageable batches.
7. Relevant messages are identified for job-search processing.
8. Relevant messages enter the AI processing pipeline.
9. Structured job-search information is persisted.
10. User is shown sync progress/state.
11. Initial synchronization completes.
12. User can view the resulting dashboard and tracked applications.

**End state:** Gmail is connected and the available job-search data has been synchronized and processed to the extent supported by the MVP.

**Edge/error states:**

- User denies Gmail permissions.
- Gmail authorization expires or is revoked.
- Gmail API is unavailable or rate-limited.
- Initial sync partially fails.
- A message cannot be retrieved or parsed.
- Processing fails for individual messages.
- Sync is interrupted and must resume without duplicating already processed messages.

A partial sync must not be represented as a fully completed sync.

---

### UF-04 — Ongoing Email Sync

**Entry point:** Gmail is connected and new or changed Gmail messages become available after initial synchronization.

**User intent:** Keep the Career Companion job-search state current without manually re-running a full sync.

**Main flow:**

1. Career Companion detects new or changed Gmail messages.
2. New messages enter the same relevance and AI processing pipeline used for synchronized messages.
3. Relevant messages update applications, timelines, and actions when appropriate.
4. Processing results are persisted.
5. The dashboard reflects the updated job-search state.

**End state:** New relevant Gmail activity is incorporated into Career Companion without requiring a full manual resynchronization.

**Edge/error states:**

- Gmail authorization has been revoked or expired.
- Gmail service is unavailable or rate-limited.
- New-message detection is delayed.
- A message is discovered more than once.
- Individual messages fail processing while other messages continue.
- Sync/retry processing is interrupted.

The flow defines the required product behavior; the underlying mechanism for detecting new Gmail messages is an architecture decision for a later phase.

---

### UF-05 — Process a Job-Related Email with AI

**Entry point:** A new or synchronized Gmail message is available for processing.

**User intent:** No direct user intent is required; the system converts unstructured recruitment communication into structured job-search information.

**Main flow:**

1. Career Companion receives or discovers a Gmail message.
2. The message is checked for job-search relevance.
3. Relevant messages are classified into an MVP-supported category such as Recruiter, Interview, Assessment, Offer, Rejection, or Follow-up.
4. AI extracts useful structured information from the message.
5. AI generates a concise summary when appropriate.
6. AI determines whether the message requires user action.
7. If action is required, the required action and relevant deadline/date are extracted when available.
8. The message is matched to an existing application when evidence supports that match; otherwise it may contribute to a new application candidate.
9. Application state and timeline information are updated when the message represents a meaningful recruitment event.
10. Processing result is persisted.

**End state:** The email has either been safely ignored as non-relevant or converted into structured job-search information.

**Edge/error states:**

- AI classification fails.
- AI returns invalid or incomplete structured output.
- Email content is malformed or unusable.
- The message is ambiguous between categories.
- No reliable application match exists.
- Multiple applications appear to match.
- The same message is processed more than once.
- AI processing is temporarily unavailable.

Invalid AI output must not silently overwrite reliable existing application information.

**Application matching principle:** Matching should use stable evidence available from the message and existing application records, such as normalized company, role, sender/domain, relevant identifiers, and temporal/contextual signals. When evidence is insufficient or conflicting, the system should preserve the ambiguity rather than silently attaching the message to the wrong application. Detailed matching logic belongs to the data/AI design phase.

---

### UF-06 — View Job Search Dashboard

**Entry point:** Authenticated user with available job-search data opens the dashboard.

**User intent:** Quickly understand the current state of their job search without manually reviewing Gmail.

**Main flow:**

1. User opens the dashboard.
2. Career Companion loads tracked applications and recent relevant activity.
3. Dashboard presents current application states.
4. Dashboard surfaces items requiring action.
5. Dashboard surfaces upcoming interviews and pending assessments when available.
6. Dashboard surfaces recent important recruiter/application communication.
7. User can select an application or actionable item for more detail.

**End state:** User has a consolidated view of what has happened, what is active, and what requires attention.

**Edge/error states:**

- No Gmail connection exists.
- Sync is still in progress.
- No job-related emails have been found.
- Some application data is incomplete.
- Dashboard data cannot be loaded.
- Individual records fail to load while the rest of the dashboard remains available.

The dashboard should distinguish **no data**, **data still processing**, and **data failed to load** rather than presenting them as the same state.

---

### UF-07 — View Application and Timeline

**Entry point:** User selects an application from the dashboard.

**User intent:** Understand the current state and known history of a specific application.

**MVP boundary:** Application detail and timeline are included because understanding application state and recruitment history is part of the product's core value. Advanced editing, analytics, and complex search/filter experiences are outside this flow.

**Main flow:**

1. User opens an application.
2. Career Companion displays available structured job details.
3. Current application status is shown.
4. Relevant recruitment events are displayed chronologically.
5. Associated email/event evidence is represented in the timeline when available.
6. Pending actions, interviews, assessments, offers, or rejection information are surfaced when applicable.
7. User can navigate back to the dashboard.

**End state:** User understands the current application state and its known history.

**Edge/error states:**

- Application contains incomplete extracted information.
- Timeline contains events with uncertain dates.
- An email cannot be loaded or is no longer available from Gmail.
- Conflicting evidence suggests different application states.
- Application data cannot be loaded.

The system should preserve uncertainty rather than invent missing application details.

**Implemented behavior (Sprint 6, 2026-10-02):** list and detail show one effective status with its source ("Set by you", "Inferred by AI", or a neutral "Status unknown"), plus "AI suggests …" when AI disagrees. The timeline is listed in the order the app recorded events. Each entry shows "Recorded" time; when an owned source email exists it shows its subject, sender and a separate "Email date" (or "Email date unknown"), otherwise "Source email unavailable". AI state changes are labelled "AI status" (no arrow when unchanged) and description/provenance are labelled "AI interpretation (not verified source text)". All metadata renders as text. Application, history and actions load, fail and retry independently; a history or action failure never blocks status editing. Event occurrence dates, agenda and Gmail deep links from history are not provided.

---

### UF-08 — Review and Act on Required Follow-up

**Entry point:** AI identifies that a job-related communication requires user action, or identifies a follow-up opportunity.

**User intent:** Understand what needs to be done next and avoid missing recruitment actions.

**Main flow:**

1. Career Companion creates or updates an actionable item from relevant communication.
2. The action is surfaced on the dashboard or application context.
3. User opens the action.
4. User sees the reason for the action and available deadline/date information.
5. User reviews the associated application/email context.
6. User marks the action as completed or dismissed.
7. Career Companion updates the action state.

**End state:** The user has either completed or intentionally dismissed the surfaced action, and the product reflects that state.

**Edge/error states:**

- Action has no reliable deadline.
- AI incorrectly identifies an action.
- Action becomes obsolete because a later email changes the application state.
- Duplicate actions are generated from multiple related emails.
- Action cannot be updated.

Actions should remain traceable to the underlying job-search evidence that caused them.

---

### UF-09 — Manually Correct Application Status

**Entry point:** User determines that the current application status is incorrect or no longer reflects reality.

**User intent:** Correct the application state while preserving the distinction between automated inference and user-confirmed information.

**Main flow:**

1. User opens the relevant application.
2. User reviews the current status and available evidence.
3. User chooses to correct the status.
4. User selects the intended valid MVP status.
5. Career Companion records the user-confirmed status and the fact that it was manually changed.
6. The application reflects the corrected status.
7. Existing email/timeline evidence remains available rather than being deleted or rewritten.

**End state:** The application reflects the user's explicit correction while retaining historical evidence and provenance.

**Edge/error states:**

- User attempts to select an invalid/unsupported status.
- Status update fails.
- Later AI processing produces conflicting evidence.
- Multiple users/devices attempt updates concurrently.

User-confirmed status must not be silently overwritten by subsequent AI processing. Detailed status precedence and transition rules belong to the data/domain design phase.

**Implemented behavior (Sprint 6, 2026-10-02):** "Change status" on the application detail opens an inline editor offering the seven statuses and "Use AI status (clear my status)". The editor explains that clearing uses the latest stored AI status without rerunning AI, and that a status change does not complete actions or send notifications. Opening the editor freezes the application, draft and base revision; background refresh may show "This status changed after you started editing" but never changes the draft or its revision.

- **Save** sends the frozen revision once; duplicate clicks are blocked and the request is never retried automatically. Success closes the editor, announces "Status saved." and returns focus.
- **409 conflict** keeps the draft and shows the current status. Only "Use current version" captures the new revision before another deliberate save.
- **Timeout, lost response, 5xx or malformed success** may have committed: the app re-reads the application and asks the user to review it. If that read fails, "Save outcome unknown" stays visible and Save stays disabled until "Retry loading status" succeeds. No path resends the PATCH with a fresh revision.
- **400** keeps the draft editable; **404** marks the application unavailable; **401** follows the normal login recovery.
- A malformed application response shows a recoverable load error and disables editing; retained earlier data is labelled as not refreshed. Navigating away during a save applies the result only to the original application.
- **Add Application:** if creation's response is lost, times out, fails with 5xx or is malformed, the draft is kept, "Creation outcome unknown" is shown, Save is blocked and the owned list is refreshed. Neither a similar entry nor a missing entry proves the outcome; the user can review applications, retry the refresh, or deliberately "Create anyway". There is no automatic resubmission or deduplication.

### UF-10 — Set Up or Switch the AI Provider (ADR-0001)

**Entry point:** "AI Provider" in the navigation, or the AI notice on the dashboard and Gmail page ("AI is not set up", a key or billing problem, or a pause).

**User intent:** Let Career Companion read job emails with the user's own AI account, knowing what is sent and what it costs.

**Main flow:**

1. User chooses a provider from the curated cards: name, "Free tier available" or "Requires paid API billing", a one-line data-use summary. The page notes that an app subscription is not an API key.
2. The guided steps link to the provider's key and billing pages. A paid-only provider says so before the key is pasted.
3. User pastes the key into a password field that is never prefilled.
4. Recommended models are shown per role. "Advanced" offers only tested models.
5. The page states the Career Companion safety limit (its own safeguard, not the provider's quota) and exactly what is sent. The user confirms consent for this provider.
6. **Save and verify:** a content-free check with the provider.
   - **Verified:** saved, Ready; waiting emails resume.
   - **Rejected:** nothing saved; the provider-specific cause and one fix are shown, and the key field is cleared.
   - **Inconclusive:** saved, Ready, shown as not yet confirmed.
7. Optional: "Try a sample email" runs both AI steps on a built-in example email, never the user's mail, and shows the result. It costs a few tokens and is not stored.

**End state:** one active provider setup. The status shows provider, models, access state with one fix, Career Companion's own counts and the safety limit.

**Edge/error states:**

- **Switching provider** uses the same flow with a new key and consent. The current setup keeps working until the new key is verified. Completed emails are not reprocessed and keep their own provenance.
- **Remove** deletes the setup and key. New emails wait; processed data stays.
- **Needs attention** (key rejected, billing or permission, model unavailable, key unreadable) or **Limited** (provider rate limit or outage, Career Companion's daily safety limit, paused): emails wait as `PENDING`, shown as "Waiting for AI". They resume after the fix, or on the next sync after a pause.
- **Retry anyway:** an email whose AI step had an uncertain or unusable outcome is never retried automatically. "Manual Retry" explains where the earlier attempt went and that one more attempt may be charged again. It names the new provider if the user switched. Only "Retry anyway" sends exactly one more attempt.

### UF-11 — Connect the User's Own Automation (ADR-0002)

**Entry point:** "Automation" in the navigation.

**User intent:** Let the user's own job-application automation tell Career Companion about each application it submits.

**Main flow:**

1. User names a token (for example the computer it is used on) and chooses an expiry (30, 90, 180 or 365 days; default 90).
2. Career Companion shows the token once, with a copy button and a warning that it will not be shown again. "Done, I saved it" removes it from the page.
3. The page shows the MCP server URL and the setup steps: add the server to the AI client's user-level settings with the `Authorization: Bearer` header (never in a repository file), allow the one tool, and turn on "Career Companion sync" in the automation profile.
4. The list shows each token's name, prefix, status (active, expired, revoked), last use and expiry.

**Edge/error states:**

- **At most 5 active tokens:** creating a sixth is refused with a message; revoke one first.
- **Expiring soon:** in a token's last 14 days the list warns that the automation will stop syncing without an error when it expires.
- **Revoke** asks for confirmation and takes effect at once; the row stays listed.
- **Outcome unknown** (timeout or lost response): the list is refreshed and the user is told to revoke any new token that appears, because its value cannot be shown again. Nothing is resent automatically.

### UF-12 — Automation Reports a Submission (system flow, ADR-0002)

**Trigger:** the user's automation agent appends an `applied` entry to its daily file after the site confirmed the submission, and calls `record_application_submission` once with values copied from the entry.

**Flow:**

1. Career Companion identifies the user from the token only.
2. A repeat of the same `sourceRecordRef` returns `already_recorded`; nothing changes.
3. The submission is matched conservatively: no application at that company → a new application is created (`appliedAt` = submission time); exactly one with the same title and none untitled → linked; anything else → waits for review.
4. A linked or created submission adds a "Submitted via automation" timeline entry. The application shows "Applied · via automation" until a user or AI status exists.

**Never:** submitted answers, resume, credentials, `skipped` or `needs_user` entries. Unknown fields are rejected (`invalid_input`, naming the field). A failure never blocks applying; the daily file stays the source of truth and a day can be replayed safely.

### UF-13 — Review an Uncertain Automation Submission (ADR-0002)

**Entry point:** "Automation submissions to review" on the dashboard, next to the email review panels.

**Main flow:** each card shows company, title, platform, destination, location, work mode, submission and recording times, the confirmation text and, for an `http`/`https` URL only, a link to the posting. The user links it to an existing application (choose, then "Link"), creates a new application from it, or ignores it.

**Rules:** the decision is final. Linking sets the application date only if it is empty. If two decisions race, only one applies; the other is told the submission was already resolved. If the outcome is unknown, the list is refreshed and nothing is resent automatically.

---

## 4. Cross-Flow State Rules

### 4.1 Authentication state

A user must be authenticated before accessing their Career Companion job-search data.

### 4.2 Gmail connection state

The product must distinguish at least:

- Not connected
- Connected
- Syncing
- Sync completed
- Sync failed/partially failed
- Authorization revoked/expired

### 4.3 Processing state

Email processing should be treated as asynchronous work. A message may be:

- Pending
- Processing
- Processed
- Ignored as non-relevant
- Failed

### 4.4 Application status provenance

Application status may be informed by email-derived evidence or explicit user correction. The system must preserve enough provenance to distinguish automated inference from user-confirmed state.

Detailed status transition and conflict-resolution rules are intentionally deferred to the application/domain design phase.

### 4.5 Action lifecycle

An actionable item should have an explicit lifecycle sufficient to distinguish at least:

- Open/pending
- Completed
- Dismissed/ignored
- Obsolete when later job-search events invalidate it

The exact domain model will be defined during data design.

### 4.6 Privacy and data handling

Career Companion should minimize persisted Gmail content and prefer structured job-search information and necessary metadata over permanently storing complete email bodies when possible. Temporary processing data should be distinguishable from persisted application data, and access must remain scoped to the authenticated user.

Specific retention periods and storage mechanisms will be defined during security/data design.

### 4.7 External service failure isolation

Failure of Gmail or the AI provider should not unnecessarily destroy already persisted Career Companion data. A downstream notification integration is not part of the MVP user flows.

---

## 5. Important Product Boundaries

These flows intentionally do **not** include:

- Applying to jobs automatically (Career Companion only receives submissions reported by the user's own automation, UF-12)
- Job discovery/job-board aggregation
- Resume generation or optimization
- Interview coaching
- Outlook/LinkedIn/Slack/WhatsApp/Telegram integrations
- Advanced analytics
- Complex search/filter functionality
- Full email-client functionality
- Enterprise/team workflows
- Discord notifications in the MVP

These are outside the MVP boundary established by Product Vision and MVP Scope.

---

## 6. End-to-End MVP Journey

The primary MVP journey is:

**Sign in → First-run onboarding → Connect Gmail → Initial sync → Ongoing email sync → AI processing → Application state/timeline updates → Dashboard → Review application/actions → Manually correct status when necessary**

The key product outcome is not successful email synchronization by itself. The successful outcome is that the user can understand their job-search state and know what requires attention without manually organizing Gmail.

---

## 7. Review Status

COM-4 review incorporated the following decisions:

- Discord notification flow removed from MVP.
- First-run/onboarding flow added.
- Ongoing email sync flow added.
- Manual application status correction flow added.
- Application detail/timeline retained as a core MVP behavior with a controlled boundary.
- Application matching ambiguity explicitly acknowledged and deferred for detailed domain design.
- Detailed status transition/conflict rules deferred to domain design.
- Action lifecycle clarified at behavioral level.
- Privacy/data minimization expectations made explicit.
- Search/filter references removed from the application-detail entry point.

The document is now considered sufficient as the behavioral foundation for the next design phase.

## Document Versioning

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2026-08-30 | Initial COM-4 user-flow definition. |
| 1.0 | 2026-08-30 | Incorporated review findings; finalized MVP flows and boundaries. |
| 1.1 | 2026-10-02 | Added UF-11 to UF-13: automation tokens, automation-reported submissions and their review (ADR-0002). |

## UF-14 — Correct a wrong email match

Choose Change link on a matched/ignored Gmail row, or Wrong application? on an
active email timeline event. Select an existing owned application (paged choices)
or unlink a matched email. Review the consequences and save once. A move retires
the old evidence/action, preserves handled work and user status choices, and sends
no notification. Unlink leaves later thread mail for manual review. On a conflict
or uncertain response, lists refresh and the dialog blocks another save until the
user closes it and checks the current link. Retired actions cannot be changed;
action controls refresh their lists after an error as well as after success.
