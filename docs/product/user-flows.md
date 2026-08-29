# User Flows — Career Companion

> **Status:** Draft for COM-4 review
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
7. If Gmail is not connected, the product presents the Gmail connection path.

**End state:** User is authenticated and can continue to Gmail connection or the dashboard.

**Edge/error states:**

- User cancels authentication.
- Google authentication fails.
- Authentication callback is invalid or expired.
- Existing account cannot be loaded.
- Session creation fails.

Career Companion must not treat a failed or cancelled authentication as a successful sign-in.

---

### UF-02 — Connect Gmail and Perform Initial Sync

**Entry point:** Authenticated user has not connected Gmail, or chooses to connect/reconnect Gmail.

**User intent:** Allow Career Companion to access the Gmail data required to understand their job search.

**Main flow:**

1. User chooses to connect Gmail.
2. Career Companion requests the required Google/Gmail permissions.
3. User grants consent.
4. Career Companion securely stores the authorization required for future Gmail access.
5. Career Companion starts the initial synchronization.
6. Gmail messages are retrieved in manageable batches.
7. Relevant messages are identified for job-search processing.
8. Relevant messages enter the AI processing pipeline.
9. Structured job-search information is persisted.
10. User is shown sync progress/state.
11. Initial sync completes.
12. User can view the resulting dashboard and tracked applications.

**End state:** Gmail is connected and the user's available job-search data has been synchronized and processed to the extent supported by the MVP.

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

### UF-03 — Process a Job-Related Email with AI

**Entry point:** A new or synchronized Gmail message is available for processing.

**User intent:** No direct user intent is required; the system is converting unstructured recruitment communication into structured job-search information.

**Main flow:**

1. Career Companion receives or discovers a Gmail message.
2. The message is checked for whether it is relevant to the job search.
3. Relevant messages are classified into an MVP-supported category such as Recruiter, Interview, Assessment, Offer, Rejection, or Follow-up.
4. AI extracts useful structured information from the message.
5. AI generates a concise summary when appropriate.
6. AI determines whether the message requires user action.
7. If action is required, the required action and relevant deadline/date are extracted when available.
8. The message is associated with an existing application when a reliable match exists, or contributes to a new application when appropriate.
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

---

### UF-04 — View Job Search Dashboard

**Entry point:** Authenticated user with available job-search data opens the dashboard.

**User intent:** Quickly understand the current state of their job search without manually reviewing Gmail.

**Main flow:**

1. User opens the dashboard.
2. Career Companion loads tracked applications and recent relevant activity.
3. Dashboard presents the current application states.
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

### UF-05 — View Application and Timeline

**Entry point:** User selects an application from the dashboard or search/filter results.

**User intent:** Understand the complete known history and current state of a specific application.

**Main flow:**

1. User opens an application.
2. Career Companion displays available structured job details.
3. Current application status is shown.
4. Relevant recruitment events are displayed chronologically.
5. Associated emails/events are represented in the timeline.
6. Pending actions, interviews, assessments, offers, or rejection information are surfaced when applicable.
7. User can navigate back to the dashboard or another application.

**End state:** User understands the current application state and its known history.

**Edge/error states:**

- Application contains incomplete extracted information.
- Timeline contains events with uncertain dates.
- An email cannot be loaded or is no longer available from Gmail.
- Conflicting evidence suggests different application states.
- Application data cannot be loaded.

The system should preserve uncertainty rather than invent missing application details.

---

### UF-06 — Review and Act on Required Follow-up

**Entry point:** AI identifies that a job-related communication requires user action, or identifies a follow-up opportunity.

**User intent:** Understand what needs to be done next and avoid missing recruitment actions.

**Main flow:**

1. Career Companion creates or updates an actionable item from relevant communication.
2. The action is surfaced on the dashboard/application context.
3. User opens the action.
4. User sees the reason for the action and available deadline/date information.
5. User reviews the associated application/email context.
6. User marks the action as completed, dismissed, or otherwise handled according to the MVP interaction model.
7. Career Companion updates the action state.

**End state:** The user has either completed or intentionally dismissed the surfaced action, and the product reflects that state.

**Edge/error states:**

- Action has no reliable deadline.
- AI incorrectly identifies an action.
- Action becomes obsolete because a later email changes the application state.
- Duplicate actions are generated from multiple related emails.
- Action cannot be updated.

Actions should be traceable to the underlying job-search evidence that caused them.

---

### UF-07 — Send Discord Notification for Important Job Activity

**Entry point:** A qualifying job-related event is successfully processed and meets the MVP notification criteria.

**User intent:** Receive timely awareness of important job-search activity without continuously checking Gmail or Career Companion.

**Main flow:**

1. A job-related email is processed.
2. Career Companion determines that the event qualifies for Discord notification.
3. Career Companion builds a concise notification from structured information.
4. Career Companion sends the notification through the configured Discord webhook.
5. Delivery succeeds.
6. The event remains available in Career Companion for later review.

**End state:** The user receives a Discord notification for the qualifying event.

**Edge/error states:**

- Discord is not configured.
- Webhook is invalid or revoked.
- Discord is temporarily unavailable.
- Notification delivery fails after the job-search event was successfully persisted.
- Duplicate notification is attempted for the same event.

Notification failure must not cause the underlying job-search event or application update to be lost.

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

### 4.4 Application state

Application status is derived from available job-search evidence and should not be changed solely because an individual email was processed without sufficient confidence.

### 4.5 External service failure isolation

Failure of Gmail, the AI provider, or Discord should not unnecessarily destroy already persisted Career Companion data.

---

## 5. Important Product Boundaries

These flows intentionally do **not** include:

- Applying to jobs automatically
- Job discovery/job-board aggregation
- Resume generation or optimization
- Interview coaching
- Outlook/LinkedIn/Slack/WhatsApp/Telegram integrations
- Advanced analytics
- Full email-client functionality
- Enterprise/team workflows

These are outside the MVP boundary established by Product Vision and MVP Scope.

---

## 6. End-to-End MVP Journey

The critical flows connect into one primary journey:

**Sign in → Connect Gmail → Initial sync → AI processing → Application state/timeline updates → Dashboard → Review application/actions → Discord notification for qualifying events**

The key product outcome is not successful email synchronization by itself. The successful outcome is that the user can understand their job-search state and know what requires attention without manually organizing Gmail.

---

## 7. Review Notes

This document deliberately focuses on behavioral flows rather than implementation architecture or UI design. API contracts, database entities, AI schemas, and screen-level UX should be derived from these flows in subsequent tasks.

### Review status

- [ ] Product flow review completed
- [ ] Edge/error states reviewed
- [ ] Approved for implementation planning

## Document Versioning

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2026-08-30 | Initial COM-4 user-flow definition. | 
