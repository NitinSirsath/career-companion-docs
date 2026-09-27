# COM-39: Close Gmail crash-retry and completion-fencing gaps

## Objective
Fix the `SyncInProgressError` false-success path where `gmailSync.ts` creates an active claim that diverges from the queued `gmailSyncJob.ts` payload, and close transport/deadline gaps.

## Problem / Context
`gmailSyncJob.ts` creates a `queuedClaim`, but `gmailSync.ts` overwrites it with a `randomUUID()`. This causes a lock/claim divergence, meaning the queued job cannot reclaim its own active claim on retry. Additionally, worker crashes or mid-page failures currently can lead to false success reporting or lost claims before final writes.

## Scope
- Ensure `gmailSync.ts` inherits and respects the `queuedClaim`.
- Implement safe atomic lock acquisitions.
- Guard against OAuth refresh or HTTP retries outlasting the job/lease.

## Requirements
- Failed finalization must not claim success.
- Ensure partial progress and checkpoint safety during mid-page failures.
- Implement robust bounds on retries and quotas.

## Acceptance Criteria
- [ ] Real crash/retry recovery functioning correctly.
- [ ] No busy/obsolete false success.
- [ ] Owner/concurrency/attempt fencing correctly bounds execution.
- [ ] Failed finalization cannot claim success.
- [ ] Partial progress/checkpoint safety guaranteed.
- [ ] Effective deadlines enforced.
- [ ] Auth/quota and retry bounds respected.
- [ ] Representative capacity evidence gathered.
- [ ] Preserved AI/domain idempotency.

## Dependencies
- Blocked by COM-37.
- Existing stabilization schema/code is a prerequisite.

## Relevant Architectural Constraints
- Atomic lock acquisitions and DB-time unexpired-lease checks are required.
- Do not introduce a new outbox or new table.

## Definition of Done
- A crashed job can successfully retry and reclaim its lock.
- `gmailSync.ts` uses the `queuedClaim` from the job payload.
- No false-success logs are emitted when supersession occurs.
- Queued/active claim mismatch and zero-row finalization are resolved.
