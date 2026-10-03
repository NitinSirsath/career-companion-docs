# ADR-0005 — A job-search agenda with explicit temporal uncertainty

| Field | Value |
| --- | --- |
| Status | ACCEPTED for local implementation — 2026-10-03 |
| Date | 2026-10-03 |
| Decision owner | Product owner, before Sprint 10 implementation |
| Scope | Interview and assessment timing; extraction/v3 and AgendaItem |
| Dependencies | OD-18; [architecture](../../planning/future-sprints/architecture.md); S10-01 |
| Reserved elsewhere | ADR-0004 remains reserved for hosting |

## Context

Sprint 7 preserves action deadline precision. Sprint 8 preserves correction intent and retired effects. Neither introduced an event-time model. Existing extraction/v2 stores interview date/time and assessment deadline strings; the timeline is ordered by recording time. Building a calendar directly from those values would assert facts the source may not support.

## Decision

1. An Action is work to do. An AgendaItem represents a proposed interview time or assessment due date. Application status and timeline recording time are separate facts.
2. Add bounded structured candidates in extraction/v3. Backend normalization preserves DATE, DATETIME and UNRESOLVED. Exact instants require explicit unambiguous temporal evidence; no timezone inference from browser, host or Gmail schedule.
3. AI output is tentative. Preserve its bounded source reference and original suggestion separately from user confirmations/corrections. A confirmed date-only item stays date-only.
4. Separate emails remain separate candidates. Reschedule/cancellation language prompts review; it does not silently replace another email's event. The user cancels the old item and confirms the new one.
5. Create agenda effects idempotently under the existing matching ownership/lock boundary. Extend ADR-0003 retirement to agenda effects. Never reprocess email or call a provider during link correction.
6. Completed extraction/v2 stays completed. Existing operations keep their selected version and paid-call uncertainty. New extraction is enabled only after v3 qualification; no bulk reprocessing.
7. No external calendar writes, joining meetings, outbound reminders or automatic lifecycle changes in Sprint 10.

## Alternatives considered

- Use timeline timestamps: these are recording times, not event times.
- Parse every existing string during UI rendering: different clients could infer different instants and silently change historical output.
- Reuse Action for every event: mixes “attend at” with “respond by” and risks duplicate task counts.
- Infer rescheduling from thread/company similarity: insufficient event identity and unsafe in multi-round interviews.
- Reprocess the mailbox: violates completed-operation and provider-cost boundaries.

## Consequences and approval

Requires an additive candidate envelope and AgendaItem schema, owned revisioned writes, new extraction evaluations, and correction/migration tests. Historical v2 mail may lack agenda coverage. Provider/model qualification for v2 does not certify v3.

Approval must record OD-18 and this ADR's status in a later implementation decision. This planning request does not approve the proposal. The full endpoint/model/rollback design is in the linked architecture.

Implementation authorization: the owner requested Sprints 10 and 11 in the verified scratchpad on 2026-10-03. OD-18 is accepted for engineering; live extraction/v3 activation remains disabled pending qualification.
