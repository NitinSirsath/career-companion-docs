# User Flows — Career Companion

> **Status:** Finalized
> **Linear Issue:** COM-4 — Design User Flows
> **Last Updated:** 2026-08-30

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

- Applying to jobs automatically
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
