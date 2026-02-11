# Tech Lead Memory

## Project: Report Scheduling

### Key Decisions Made in Phase 1 TD
- Generation and delivery are separate Celery tasks chained via dispatch (not Celery chain primitive)
- Scheduling engine uses `SELECT ... FOR UPDATE SKIP LOCKED` for tick concurrency safety
- Timezone: `schedule_hour`/`schedule_minute` as integers (not Time column) in local tz, `timezone` as IANA string, `next_run_at` in UTC
- Monthly day clamping: day 31 in February -> last day of month
- Max 50 recipients per schedule, 25 active schedules per user (from architecture section 6.2)
- No automatic retries in Phase 1 (max_retries=0, deferred to P2)
- Stub implementations for both report generation and email delivery
- `ReportGenerator` protocol is the integration boundary to the existing report system
- Recipients managed inline via schedule endpoints (PATCH replaces entire list)
- Pause/resume use dedicated action endpoints (POST /pause, POST /resume), not PATCH with status
- Cursor-based pagination for both schedule list and run history
- Rate limit: 10/min/user applies to BOTH creates and updates (broader than architecture's create-only)
- Report content base64-encoded for JSON serialization through Celery
- Workers use synchronous DB sessions (get_sync_session), not async
- Last-write-wins for concurrent modifications (no optimistic locking in Phase 1)
- Email volume monitoring threshold: 500/day/user via structured logging (not hard block)

### Security Decisions Addressed
- CRLF injection prevention: regex check on email addresses and subjects at both API (Pydantic validator) and task boundary (deliver task)
- Query-level authorization: pre-filter by owner_id, return 404 (not 403) on not-found-or-not-owned
- Celery task input validation at entry: validate run_id exists and is in correct state before proceeding
- Redis treated as trusted boundary; production access must be network-isolated

### Patterns Used
- Pydantic v2 `alias` + `populate_by_name=True` for camelCase JSON <-> snake_case Python mapping
- `compute_next_run` as a pure function (no DB access) for testability
- State machine validation as shared utility (both API and Celery tasks call same function)
- Zod schemas on frontend mirror Pydantic response schemas for consistent validation
- TanStack Query key factory pattern (`scheduleKeys.all`, `.lists()`, `.detail(id)`)
- RFC 7807 error responses with optional `errors[]` array for field-level validation
- Owner-scoped query helper (`get_owned_schedule`) as reusable FastAPI dependency pattern

### Open Questions Flagged
1. Authentication mechanism (`get_current_user` dependency) -- placeholder with X-User-Id header for dev
2. Report type validation -- hardcoded list in config for Phase 1, dynamic when real system integrates
3. Auth system `is_admin` flag availability -- assumed yes, fallback to defer admin to Phase 2

### Codebase State
- Repository is greenfield -- no existing source code in packages/ yet
- Only planning artifacts exist (product-plan.md, architecture.md, requirements.md, reviews)
- Project conventions define the target structure but nothing is built yet

### Task Breakdown (7 tasks)
1. DB Schema + Migrations (foundation, no deps)
2. API Core Infrastructure (depends on 1)
3. Schedule CRUD + Schemas + State Machine + Next Run (depends on 1, 2)
4. Run History + Report Types Endpoints (depends on 1, 2, 3)
5. Celery Tasks + Protocols (depends on 1, 3)
6. Frontend Schemas + Hooks (parallelizable with 3-5)
7. Frontend Pages + Components (depends on 6)

### Reviewer Feedback Addressed
- Security Engineer: WARNING-1 (CRLF injection), WARNING-2 (task input validation), SUGGESTION-1 (query-level auth), SUGGESTION-2 (email volume monitoring)
- API Designer: SUGGESTION-1 (response shape), SUGGESTION-2 (pagination), SUGGESTION-3 (run filtering), SUGGESTION-4 (report types), SUGGESTION-5 (state transitions)
- Architect: W2 (rate limit scope), S2 (concurrent modification), observation 1 (shared state machine), observation 2 (tick atomicity), observation 3 (recipient email snapshot)
- Code Reviewer: W1 (DB->API circular dep), W2 (tick orphan recovery), W3 (PATCH null ambiguity), W4 (started_at semantics), W5 (Redis per-request)

### Lessons from Code Reviewer Feedback (Revision 1)
- DB package must never import from API package -- use env vars directly for config
- Commit-then-dispatch gaps need explicit orphan recovery (idempotent re-dispatch on next tick)
- Pydantic PATCH schemas need `model_fields_set` documentation for null disambiguation
- Fields shared across tasks (like `started_at`) need explicit ownership semantics in column comments
- Module-level connection pools for Redis (not per-call `from_url()`)
