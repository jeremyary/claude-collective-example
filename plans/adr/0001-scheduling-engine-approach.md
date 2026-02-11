# ADR-0001: Scheduling Engine Approach

## Status
Proposed

## Context
The system needs to evaluate "which schedules are due now?" reliably and execute them. This is the critical-path component: missed or duplicate executions are the highest-impact failure mode for user trust (see Architect review, observation 2). The scheduling engine must handle timezone-aware cadences (daily, weekly, monthly) and eventually custom expressions (Phase 2).

Two primary approaches exist within our Celery/Redis stack.

## Options Considered

### Option 1: Celery Beat with Dynamic Schedules
Use celery-beat (or django-celery-beat / redbeat) to manage per-schedule entries. Each schedule maps to a Celery Beat entry that fires at the configured cadence.

- **Pros:** Built-in cron-like scheduling. No custom evaluation logic. Well-documented Celery pattern.
- **Cons:** Dynamic schedule management with Celery Beat is fragile -- adding/removing/modifying Beat entries at runtime requires either database-backed Beat (django-celery-beat, which pulls in Django) or RedBeat (Redis-backed). Both add operational complexity. Beat is a single process -- if it crashes, no schedules fire until it restarts. Syncing hundreds of Beat entries with the schedules table creates a dual-source-of-truth problem. Timezone handling in Beat is inconsistent across backends.

### Option 2: Custom Tick Task (SELECT FOR UPDATE)
A single Celery periodic task ("tick") runs every minute. It queries the database for schedules whose `next_run_at` is at or before the current time, claims them with `SELECT ... FOR UPDATE SKIP LOCKED`, and dispatches individual execution tasks. After dispatching, it computes and writes the next `next_run_at` for each schedule.

- **Pros:** Database is the single source of truth. No synchronization between Beat entries and the schedules table. `SELECT FOR UPDATE SKIP LOCKED` enables multiple tick workers for availability without duplicate execution. Timezone resolution happens once (when computing `next_run_at`) using the schedule's IANA timezone. Adding/modifying/deleting schedules is just a database write -- no Beat reload needed. Straightforward to test: mock the clock, run the tick, assert dispatched tasks.
- **Cons:** Requires implementing cadence evaluation logic (compute next occurrence from a cron-like rule). One-minute polling introduces up to 60 seconds of latency between scheduled time and execution start. Must handle clock skew and long-running ticks carefully.

## Decision
**Option 2: Custom Tick Task.** The database is already the authoritative store for schedule configuration. Making it also the authoritative store for "what is due" eliminates a synchronization layer that adds complexity without adding value. The one-minute polling latency is acceptable -- the product NFR says "users should not notice a delay," and a sub-60-second offset on a daily/weekly/monthly cadence is imperceptible.

Celery Beat is still used, but only for one thing: triggering the tick task every 60 seconds. This is a static, single-entry Beat configuration -- trivial to operate.

## Consequences

### Positive
- Single source of truth for schedule state (the database)
- No dual-bookkeeping between Beat entries and schedule rows
- Horizontal scalability via `SKIP LOCKED` -- can run multiple tick workers
- Timezone logic is centralized in the `next_run_at` computation
- Easy to test: the tick task is a pure function of "current time" and "database state"

### Negative
- Must implement cadence-to-next-occurrence computation (daily, weekly, monthly, and eventually custom expressions)
- Up to 60 seconds of scheduling latency (acceptable for the cadences in scope)
- Must monitor tick task health -- if the tick stops running, all scheduling stops

### Neutral
- Celery Beat is still a dependency, but only for the single static tick entry
- The `next_run_at` column doubles as an index for the tick query and a display value for the UI ("next delivery time")
