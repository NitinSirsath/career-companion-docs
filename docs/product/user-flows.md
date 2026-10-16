# User Flows — Career Companion

> **Status:** Finalized
> **Linear Issue:** COM-4 — Design User Flows
> **Last Updated:** 2026-10-15 (notes where the app differs from UF-02 to UF-05 and §5)

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

### UF-01 — Sign In

**Entry point:** User opens Career Companion while unauthenticated.

**User intent:** Securely access their Career Companion account.

**Main flow:**

1. User selects **Sign in with Google**. (A user who has set a password can instead enter their email and password; see UF-17.)
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
- Email or password is wrong. The answer is the same for a wrong email and a wrong password.
- Too many wrong tries: sign-in with that email is blocked for up to 15 minutes, and the page says how long.

Career Companion must not treat a failed or cancelled authentication as a successful sign-in.

---

### UF-02 — First-Run / Onboarding

> **Not built (2026-10-15):** there is no first-run screen. After sign-in the user lands on the Mails page (UF-14) and connects Gmail from the Gmail Sync page (UF-03). The text below is the intended flow.

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
2. Career Companion requests the minimum required Google/Gmail permissions: reading mail, and sending the replies the user writes in the app (UF-16).
3. User grants consent.
4. Career Companion securely stores the authorization required for future Gmail access.
5. Career Companion starts the initial synchronization. *(2026-10-15: connecting does not start a sync. The first sync runs when the user presses Sync Now, or at the next scheduled sync, see UF-04.)*
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

> **Built (2026-10-15):** new mail is found by a sync, not detected as it arrives. A sync runs when the user presses Sync Now and by itself twice a day, at 00:00 and 18:00 India time (backend `services/gmailSchedule.ts`); a time missed while the server was off is caught up when it starts.

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
8. The message is matched to an existing application when evidence supports that match; otherwise it may contribute to a new application candidate. *(2026-10-15: mail never creates an application by itself. An unmatched job mail waits on the Mails page, where the user creates its application in one click, UF-15.)*
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

> **Changed 2026-10-11:** the Mails page (UF-14) is now the page the user lands on and does this flow's job. The Dashboard stays in the navigation and shows "Coming soon" until analytics are built. The Agenda is in the navigation too and also shows "Coming soon" (2026-10-13); until it is built, each interview and assessment date is shown on its mail. The text below is the original flow, kept for history.

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

**Changed 2026-10-11:** the application page also lists the application's conversations, and each mail in the timeline has an "Open mail" link to the mail's details page (UF-16).

---

### UF-08 — Review and Act on Required Follow-up

**Entry point:** AI identifies that a job-related communication requires user action, or identifies a follow-up opportunity.

**User intent:** Understand what needs to be done next and avoid missing recruitment actions.

**Main flow:**

1. Career Companion creates or updates an actionable item from relevant communication.
2. The action is surfaced on the Mails page (UF-14), whether or not the mail has an application yet, and on the application.
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

**Entry point:** "AI Provider" in the navigation, or the AI notice on the Mails page and Gmail page ("AI is not set up", a key or billing problem, or a pause).

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
- **Needs attention** (key rejected, billing or permission, model unavailable, key unreadable) or **Limited** (provider rate limit or outage, Career Companion's daily safety limit, paused): emails wait as `PENDING`, shown as "Waiting for AI". They resume after the fix. After a provider pause they are read by themselves when the pause ends, and the notice offers **Read waiting mails now** for a new try at once. A busy provider is shown as busy ("Google Gemini is busy right now"), not as a limit on the user's account.
- **Retry anyway:** a mail with no answer in time is tried again by itself, up to 3 tries; after the 3rd try, or after an unusable answer, it waits for the user's OK and is not tried again by itself. A batch with no answer in time is held at once. "Manual Retry" explains where the earlier attempt went and that one more attempt may be charged again. It names the new provider if the user switched. Only "Retry anyway" sends exactly one more attempt.

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

**Entry point:** "Automation submissions to review" at the top of the Automation page (moved from the old dashboard on 2026-10-11).

**Main flow:** each card shows company, title, platform, destination, location, work mode, submission and recording times, the confirmation text and, for an `http`/`https` URL only, a link to the posting. The user links it to an existing application (choose, then "Link"), creates a new application from it, or ignores it.

**Rules:** the decision is final. Linking sets the application date only if it is empty. If two decisions race, only one applies; the other is told the submission was already resolved. If the outcome is unknown, the list is refreshed and nothing is resent automatically.

---

### UF-14 — See New Job Mail and What It Needs (Mails page)

**Entry point:** the user opens Career Companion. The Mails page is the home page.

**User intent:** see every job mail, what it asks for, and which application it belongs to, without setting anything up first.

**Main flow:**

1. The **Job mails** tab lists one row per conversation (a Gmail thread of job-related mail), with open requests first.
2. Each row shows the subject, sender and company, the type of mail (Recruiter, Interview, Assessment, Offer, Rejection, Update), and a **New** mark until the user opens it.
3. The **Action** column shows what the mail asks for, in the AI's words, with a due date when one is known. It works with or without an application. Its state is **Needs action**, **Replied** (the user sent a later message in the thread) or **Done**.
4. The user marks an action done, reopens it, or, after a reply, asks to be reminded again.
5. The **Application** column shows the application and its stage, or a **Create application** button.
6. The user can search, filter by action, type and application, show new mail only, and sort by attention or by date. The filters live in the page address, so Back returns to the same view.

**End state:** the user knows what each job mail needs and has acted on it or left it open on purpose.

**Edge/error states:**

- No job mail yet: the page says so and points to Gmail Sync.
- AI access is not ready: a notice on the page says so; mail waits and is listed once it has been read.
- A newer mail replaced the request while the user looked at it: the answer is refused and the row is refreshed.

No label on this page uses an internal pipeline word such as "unmatched" or "processed".

---

### UF-15 — Create an Application from a Mail in One Click

**Entry point:** a conversation with no application shows **Create application**, on the Job mails tab or on the mail's details page.

**Main flow:**

1. The user clicks **Create application**. There is no form: company and role come from what the AI read in the mail.
2. Career Companion creates the application and links every mail of the conversation to it. The timeline, status and actions follow as for any linked mail.
3. Other waiting conversations from the same company join the application by the normal matching rules, so the user is not asked again.

**Variations:**

- The user already has one application that fits: the conversation joins it. No second application is made.
- More than one application fits: the row offers only those applications to choose from.
- The AI could not read a company name: the row asks for one.
- Pressing twice never creates two applications.

**End state:** the conversation belongs to an application. Later mail from the same company joins it automatically.

---

### UF-16 — Open a Mail and Read the Conversation

**Entry point:** the user opens a row on the Job mails tab or on the **Gmail** tab, or follows **Open mail** from an application's timeline or its Conversations list.

**Main flow:**

1. Career Companion fetches the conversation from Gmail at that moment and shows every message in order: sender, recipients, date, Gmail labels, the text, and attachments with a download link. Under the sender, a mail names the other people it went to ("to me, Daniel Okafor · cc Lena Fischer"). The user's own replies are shown as theirs.
2. For a job mail, a panel beside the conversation shows the action with its buttons, the application or **Create application**, and what the AI read from each mail, labelled as the AI's reading.
3. For any other mail opened from the Gmail tab, only the conversation is shown, with a note that Career Companion did not analyze it and keeps nothing from it.
4. **Reply.** Under the conversation the user writes a plain-text reply and chooses **Send reply**. It goes to the person who wrote, in the same Gmail conversation, with the earlier mail quoted below. The box shows who it goes to; no address can be typed. The reply then shows in the conversation, and a job mail's action shows **Replied** at once. **Open in Gmail** and **Reply in Gmail** still lead to Gmail.
   - **Other people on the mail:** when the mail also went to other people, the reply goes to **all of them** unless the user says not. The box says "Reply to all" with the number of people, the person who wrote in **To** and everyone else in **Cc**. The user's own address is left out.
   - **Only the sender:** the link **Reply only to <sender>** sends the reply to that person alone; the box then says who will not get it, and **Reply to all** leads back. These are the only two choices: nobody can be added or removed one by one.
5. Opening a conversation ends its New mark.

**Reply edge states:**

- **Gmail was connected before replying existed:** the box shows **Allow sending**. Google asks once for the permission to send, and the user comes back to the same mail. Nothing the app holds is lost.
- **Gmail does not answer:** the box says "Could not confirm that your reply was sent" and keeps the text. Nothing is sent again by itself: the user checks the conversation, then chooses **Send again** or not.
- **Nobody else wrote in the conversation:** there is no reply box. The app never writes to an address the user did not hear from.
- **Not in this version:** new mail, attachments, adding or removing single receivers, saved drafts and formatting.

**The Gmail tab** lists the inbox live from Gmail, newest first, with Older and Newer. It has no filters, no search and no actions on the list. A mail is answered from its details page. Rows that are job mail are marked and open with the job panel.

**Privacy:** the mail text is fetched on open, shown and dropped. It is never stored or logged. A sender's HTML cannot run scripts, and pictures from the internet stay blocked until the user asks for them. The text of a reply goes to Gmail and is not stored or logged by Career Companion either.

**Edge/error states:**

- Gmail is not connected, or access ended: the page says so and links to Gmail Sync. The job panel still shows.
- The mail is no longer in Gmail, or Gmail cannot be reached: the page says so and offers to try again.
- A reply sent in Gmail shows in the conversation at once, and as **Replied** on the action after the next sync.

---

### UF-17 — Set a Password and Sign In with It

**Entry point:** A signed-in user opens the **Account** page.

**User intent:** Sign in with email and password as well as with Google.

**Main flow:**

1. The Account page shows the user's email and how they sign in: Google, and whether a password is set.
2. User enters a new password twice (at least 10 characters) and saves it.
3. From then on the login page accepts the email and that password.
4. To change the password, the user also enters the current one. After a change, the user is signed out everywhere else.

**Forgot password:** Career Companion sends no email. The user signs in with Google and sets a new password on the Account page.

**Rules:**

- A new account is created by Google sign-in only. There is no sign-up with a password.
- Career Companion never shows, logs or returns a password.

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

Since 2026-10-11 the user can read a mail's text in the app (UF-16). The text is fetched from Gmail when the mail is opened and is never stored or logged. Of the user's own sent mail, only the message ID, thread ID and sent time are kept, to show that they replied. Since 2026-10-13 the user can also reply from the app; that text is handed to Gmail and kept nowhere else.

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
- Discord notifications in the MVP *(2026-10-15: Discord notifications exist for one configured user only, `DISCORD_USER_ID`; it is the owner's own tool, not a product feature)*

These are outside the MVP boundary established by Product Vision and MVP Scope.

---

## 6. End-to-End MVP Journey

The primary MVP journey is:

**Sign in → First-run onboarding → Connect Gmail → Initial sync → Ongoing email sync → AI processing → Mails page: see what each job mail needs → Create the application in one click (or it links by itself) → Open the mail and reply in Gmail → Review application/actions → Manually correct status when necessary**

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
| 1.2 | 2026-10-11 | Mail-first walkthrough (COM-158): added UF-14 to UF-16; the Mails page replaces the dashboard as home; the automation review moved to the Automation page. |
| 1.3 | 2026-10-12 | Sign-in with email and password (COM-183): UF-01 renamed "Sign In"; added UF-17. Dev mode and its mail sandbox are a testing tool, described in engineering/dev-mode.md, not a user flow. |
| 1.4 | 2026-10-13 | UF-16: reply to a mail from the app (COM-176); a reply goes to everyone on the mail unless the user chooses the sender alone, and a mail names the other people on it. UF-03 names the send permission. The Agenda is back in the navigation as "Coming soon". |
| 1.5 | 2026-10-15 | Dated notes where the app differs: no first-run screen (UF-02); connecting does not start a sync (UF-03); sync runs on Sync Now and twice a day (UF-04); mail never creates an application by itself (UF-05); Discord for one configured user (§5). |
