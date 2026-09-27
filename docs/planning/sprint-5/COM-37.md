# COM-37: Establish the preserved-dataset baseline and Sprint 5 readiness gates

## Objective
Establish a non-destructive starting record for the existing emails, and complete a manual live-verification of Gmail sync on that baseline to prove the end-to-end integration works on real data without destruction. Capture actual runtime commits, migration state, owner/mailbox, original email identity/metadata manifest, backlog/AI state and protected backup.

## Problem / Context
We need to measure the baseline count of existing emails (approx 1,620) and run a live verification. Previously, we lacked a formalized baseline and a safe live verification procedure. Running tests or live syncs directly against the database risks destroying existing dataset choices and user data.

## Scope
- Recording exactly which code/versions are running.
- Taking an immutable snapshot/backup of the current database (identities, metadata, AI choices).
- Setting up a separate local test DB for fixture tests.

## Requirements
- Measure the original N0 baseline dataset size.
- Separate fixture DB from the preserved lane.
- Capture schema and state before testing.

## Acceptance Criteria
- [ ] Recorded checkout/runtime/worker versions.
- [ ] Measured original N0; complete identity/metadata manifest and protected backup/restore evidence.
- [ ] Backlog/AI/user choices preserved.
- [ ] Separate DB lanes established.
- [ ] Read-only migration checks.
- [ ] Provider/session readiness documented.


## Dependencies
- None in Sprint 5; runtime/schema/provider/backup access are external readiness gates.

## Relevant Architectural Constraints
- Zero reset/delete/reseed operations on the baseline dataset.
- Privacy/security of the preserved data must be strictly maintained.

## Definition of Done
- Exact checkout SHAs recorded and original dataset measured.
- Protected backup reference exists.
- Separate test database available for local smoke tests.
