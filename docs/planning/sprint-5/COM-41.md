# COM-41: Correlate Gmail sync outcomes, counts, retries, and worker health

## Objective
Provide accurate correlated lifecycle evidence of Gmail sync outcomes, failure outcomes, counts, retries, and worker health without introducing a new observability platform.

## Problem / Context
Current logs lack complete sync start/finish correlation and hide partial insertion outcomes. Counts often omit filtered/404 messages, and generic error names mean operators cannot explain loss or distinguish failures clearly.

## Scope
- Introduce a structured event builder in `gmailSync.ts` and `emailProcessingJob.ts`.
- Reconcile partial and full counters.
- Categorize failures safely.

## Requirements
- Retain current infrastructure (no new telemetry platform).
- Do not log credentials or email contents.

## Acceptance Criteria
- [ ] Correlated lifecycle or detectable crash tracked.
- [ ] Mode/timing/retry/checkpoint fields populated.
- [ ] Reconciled partial/full counters for distinct duplicate/reference/suppression semantics.
- [ ] Safe failure categories defined and used.
- [ ] Worker/queue/AI diagnosis metrics available.
- [ ] Independent ingestion/AI milestones logged.
- [ ] Redaction and useful live evidence gathered without logging sensitive data.

## Dependencies
- Blocked by COM-37 and finalized COM-39 lifecycle; design may start earlier.

## Relevant Architectural Constraints
- Privacy must be maintained; no email body or credentials in logs.

## Definition of Done
- Sync outcomes and attempts are correlated with unique IDs.
- Operators can see retry metadata, queue wait times, and explicit counters for duplicates, references, and suppressed messages.
- Success/failure/retry/expired-worker states are explained accurately in the logs.
