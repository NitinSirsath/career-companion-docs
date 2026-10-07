# Career Companion Documentation

Central documentation repository for Career Companion including product planning, architecture, engineering decisions, and technical documentation.

See the [API Contracts](./docs/architecture/api-contracts.md) document for cross-cutting API requirements covering pagination, authentication, ownership, validation, errors, retries, and privacy.

**Start here: [planning index and roadmap](docs/planning/README.md)** (updated 2026-10-02). It gives the current state, the order of work (migration verification → Sprint 6 closeout → Sprint 7 → Sprint 8), every open ticket, owner decisions and what is out of scope. Its evidence base is the [state audit of 2026-10-02](docs/planning/state-audit-2026-10-02.md). All planning IDs are local; no Linear state is assumed.

Product decision locked 2026-10-01: scheduled Gmail sync runs twice daily (12:00 AM and 6:00 PM); manual sync remains available, and no heavier scheduled-processing scope is added. Not yet implemented: planned as [S7-05](docs/planning/sprint-7/S7-05-twice-daily-scheduled-sync.md).

Recent work, all implemented locally on 2026-10-02 without git, with evidence measured on the old laptop only (re-run in the [migration verification gate](docs/planning/migration-verification/README.md)):

- [Sprint 6 — User-controlled application status and explainable history](docs/planning/sprint-6/README.md): implemented ([execution report](docs/planning/sprint-6/execution-report.md)); D1 events-only, D2 approved. Not yet accepted: the live Gmail/original-data entry gate remains open, and the remaining work is in the [Sprint 6 closeout](docs/planning/sprint-6/closeout/README.md).
- [ADR-0001 — user-provided AI](docs/architecture/decisions/ADR-0001-user-provided-ai.md), with the [AI capability architecture](docs/architecture/ai-capability-architecture.md), [plan](docs/planning/byo-ai/README.md), [issues](docs/planning/byo-ai/issues.md) and [execution report](docs/planning/byo-ai/execution-report.md). Not released: all providers stay hidden until certified ([provider evaluation](docs/ai/provider-evaluation.md)) and their data-use text is approved.
- [ADR-0002 — automation submissions via MCP](docs/architecture/decisions/ADR-0002-automation-submissions-via-mcp.md), [plan](docs/planning/mcp-feature/README.md) and [execution report](docs/planning/mcp-feature/execution-report.md): MCP-00..MCP-07 and MCP-09 part A implemented; MCP-08 staged; Antigravity (MCP-09 part B) not yet verified.

Earlier records: [Sprint 5 — Gmail Incremental Sync & Reliability](docs/planning/sprint-5/README.md). Its Gmail reliability work is absent from the current code and is replanned as [S5-FU-01](docs/planning/sprint-5/follow-up-gmail-reliability.md) (Sprint 7) and [S5-FU-02](docs/planning/sprint-5/follow-up-google-request-bounds.md) (Sprint 8). The [stabilization audit of 2026-09-26](docs/engineering/stabilization-audit.md) is a dated historical assessment.
