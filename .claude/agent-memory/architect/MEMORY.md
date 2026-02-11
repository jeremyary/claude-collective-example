# Architect Memory

## Project: Report Scheduling Platform

- **Maturity:** MVP
- **Stack:** Python 3.12+/FastAPI, React 19/TypeScript, PostgreSQL 16, Celery/Redis
- **Product plan reviewed:** 2026-02-10, verdict APPROVE WITH SUGGESTIONS
- **Architecture designed:** 2026-02-10, 4 ADRs created

### Key Architecture Decisions (ADRs)

1. **ADR-0001: Custom tick task** over Celery Beat dynamic entries. DB is single source of truth. SELECT FOR UPDATE SKIP LOCKED for concurrency safety. 60s polling.
2. **ADR-0002: Protocol interface for report generation.** Python Protocol class. Stub for MVP. Swappable at startup via config.
3. **ADR-0003: Protocol interface for email delivery.** SMTP default, Mailpit for local dev. Same pattern as report generation.
4. **ADR-0004: Timezone as local time + IANA identifier.** next_run_at stored as UTC, recomputed after each execution using zoneinfo stdlib. DST-correct.

### Resolved Open Questions

- **Orphan schedules:** Keep running when owner deactivated. No cascading FK. Admin dashboard (Phase 2) is the escape hatch.
- **Recipient limits:** 50 per schedule, 25 active schedules per user. Enforced at API layer, configurable.
- **Unsubscribe:** Owner-mediated for MVP (email footer with contact info). Tokenized self-service deferred to Phase 2.
- **Recipient storage:** Email strings in a `recipients` table, not user references. Supports internal and external.
- **State machines:** Separate for Schedule (active/paused/deleted) and ScheduleRun (pending through terminal states).
- **Generation/delivery separation:** Confirmed. Separate Celery tasks with independent status tracking.

### Key Patterns

- **Protocol-based integration boundaries:** Used for both report generation and email delivery. Consistent pattern across external integrations.
- **Three-task Celery chain:** tick -> execute -> deliver. Each independently retryable and observable.
- **Dual session factories:** async (FastAPI) and sync (Celery workers) from packages/db. Same DB, separate pools.

### Stakeholder Preferences Observed

- Accepted all reviewer suggestions as "downstream guidance" rather than requiring product plan changes -- indicates preference for forward momentum over perfect plans.
- Comfortable with borderline items being resolved by downstream agents with escalation path.
