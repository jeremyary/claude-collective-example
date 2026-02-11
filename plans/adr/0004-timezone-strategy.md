# ADR-0004: Timezone Strategy

## Status
Proposed

## Context
Timezone support is a P0 feature. Schedules run according to the owner's configured timezone. The Architect review (observation 3) identified timezone handling as a cross-cutting concern affecting: schedule creation (UI), schedule evaluation (tick task), next-run computation, and delivery history display. Inconsistent handling causes reports to arrive at unexpected times -- a medium-impact risk identified in the product plan.

## Options Considered

### Option 1: Store Cadence in UTC, Convert at Display
Store the schedule's target time as UTC in the database. Convert to the owner's timezone only for display in the UI. The tick task compares against UTC.

- **Pros:** Simple storage. All comparisons are in UTC.
- **Cons:** Does not handle DST transitions correctly. If a user schedules "9:00 AM US/Eastern," that is UTC-5 in winter and UTC-4 in summer. Storing a fixed UTC time means the report arrives at 8 AM or 10 AM local time after a DST change. Users will perceive this as a bug.

### Option 2: Store Cadence in Local Time + IANA Timezone
Store the schedule's target time in the owner's local time (e.g., "09:00") alongside the IANA timezone identifier (e.g., "America/New_York"). When computing `next_run_at`, resolve the local time to UTC using the timezone rules for that specific date. Store `next_run_at` as a UTC timestamp for the tick task to query.

- **Pros:** Correctly handles DST transitions -- "9:00 AM Eastern" is always 9:00 AM local time. `next_run_at` is recomputed after each execution using the correct UTC offset for the next occurrence date. The tick task only sees UTC timestamps -- simple comparison. UI displays the stored local time directly.
- **Cons:** `next_run_at` must be recomputed after each run (not just incremented by a fixed interval). Requires the `zoneinfo` module (stdlib in Python 3.9+) or `pytz` for timezone resolution. Edge cases: ambiguous times during fall-back (2:00 AM happens twice), nonexistent times during spring-forward (2:30 AM is skipped).

### Option 3: Store Everything as Cron + Timezone, Evaluate with croniter
Store the cadence as a cron expression string plus an IANA timezone. Use a library like `croniter` to compute the next occurrence in the target timezone, then convert to UTC.

- **Pros:** Expressive -- supports Phase 2 custom cadences naturally. Well-tested libraries exist.
- **Cons:** Cron expressions are not user-friendly for Phase 1's simple cadences (daily/weekly/monthly). Adds a library dependency. Overkill for MVP, though forward-compatible.

## Decision
**Option 2: Store Cadence in Local Time + IANA Timezone** for Phase 1. The schedule stores:
- `schedule_time`: time-of-day in the owner's local timezone (e.g., "09:00")
- `timezone`: IANA timezone identifier (e.g., "America/New_York")
- `cadence_type`: enum (daily, weekly, monthly)
- `cadence_day`: day-of-week (for weekly) or day-of-month (for monthly), nullable
- `next_run_at`: UTC timestamp, recomputed after each execution

Resolution rules for edge cases:
- **Ambiguous times (fall-back):** Use the first occurrence (the earlier UTC time). A report arriving one hour "early" in wall-clock terms is less confusing than arriving one hour "late."
- **Nonexistent times (spring-forward):** Advance to the next valid minute. If 2:30 AM does not exist, deliver at 3:00 AM.
- **Monthly on the 31st:** Deliver on the last day of the month if the month has fewer than 31 days. (Resolves product plan open question 6.)

Use Python's `zoneinfo` module (stdlib) for timezone resolution. No external timezone library needed.

For Phase 2 custom cadences, the cadence fields can be replaced or augmented with a cron expression while keeping the same `next_run_at` computation pattern. This is forward-compatible but does not require implementing cron parsing now.

### Data Flow

```
UI: user picks "09:00" + "America/New_York" + "weekly, Monday"
     |
     v
API: stores schedule_time="09:00", timezone="America/New_York",
     cadence_type="weekly", cadence_day=1 (Monday)
     computes next_run_at = next Monday 09:00 Eastern -> UTC
     |
     v
DB:  next_run_at = 2026-02-16T14:00:00Z  (Mon 9AM ET = 2PM UTC in Feb)
     |
     v
Tick: SELECT WHERE next_run_at <= now() -- all UTC, no timezone logic
     |
     v
After execution: recompute next_run_at for the FOLLOWING Monday
     using zoneinfo to resolve 09:00 America/New_York on that date
```

## Consequences

### Positive
- Users always see their local time -- "9:00 AM" means 9:00 AM regardless of DST
- Tick task operates purely in UTC -- simple, fast, no timezone logic in the hot path
- Edge cases have deterministic, documented resolution rules
- No external timezone library -- `zoneinfo` is stdlib

### Negative
- `next_run_at` must be recomputed (not incremented) after each execution
- IANA timezone database updates (rare) may shift `next_run_at` for future executions -- acceptable
- Ambiguous/nonexistent time handling adds test cases

### Neutral
- The UI must present an IANA timezone picker (or a curated subset) -- standard React component
- The API validates timezone values against `zoneinfo.available_timezones()`
