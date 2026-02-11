# Work Breakdown: Report Scheduling Phase 1

**Date:** 2026-02-10
**Author:** Project Manager
**Input:** Technical Design Phase 1 (validated), Requirements (validated), Architecture (validated)

---

## Summary

| Metric | Value |
|--------|-------|
| Work Units | 5 |
| Tasks | 11 |
| Estimated Duration | 3–4 weeks (with parallel execution) |
| Critical Path | WU-1 → WU-2 → WU-3 → WU-4 (backend sequence) |
| Parallelization | WU-5 (frontend) can run in parallel with WU-3, WU-4 |

---

## Task Breakout Rationale

The TD defined 7 implementation tasks. During work breakdown, these were reorganized into 11 tasks to comply with task sizing constraints from `agent-workflow.md` (3-5 files per task, single concern, machine-verifiable exit conditions).

| TD Task | WB Task(s) | Change | Rationale |
|---------|-----------|--------|-----------|
| TD Task 1: Database Schema and Migrations | T-1.1 | No change (1:1 mapping) | Single task, 4 files, meets sizing constraints |
| TD Task 2: API Core Infrastructure | T-2.1 | No change (1:1 mapping) | Single task, 5 files, meets sizing constraints |
| TD Task 3: Schedule CRUD Endpoints + Pydantic Schemas | T-3.1, T-3.2, T-3.3, T-3.4 | Split into 4 tasks | TD Task 3 touched 5+ files and covered multiple concerns: (1) schemas, (2) state machine + next_run computation, (3) CRUD endpoints, (4) pause/resume endpoints. This violated the single-concern rule. Split to isolate: schemas (T-3.1), services (T-3.2), CRUD (T-3.3), pause/resume (T-3.4). Grouped into WU-3 to share context. |
| TD Task 4: Delivery History Endpoint + Report Types Endpoint | T-4.1, T-4.2 | Split into 2 tasks | TD Task 4 covered two independent concerns: (1) run history endpoint with pagination/filtering, (2) report types endpoint with hardcoded list. No shared context. Split into separate tasks. |
| TD Task 5: Celery Tasks + Service Protocols | T-4.3 | Moved to WU-4 | No split needed (5-6 files acceptable for complex task). Moved from TD Task 5 position to T-4.3 for logical grouping with other backend endpoints in WU-4. |
| TD Task 6: Frontend Schemas, Services, and Hooks | T-5.1 | No change (1:1 mapping) | Single task, 4-5 files, meets sizing constraints |
| TD Task 7: Frontend Pages and Components | T-5.2 | No change (1:1 mapping) | Single task, 5 primary files + atoms/molecules, meets sizing constraints |

**Summary:** 7 TD tasks → 11 WB tasks. Primary change: TD Task 3 split into 4 tasks (T-3.1 through T-3.4) to enforce single-concern and shared-context patterns. TD Task 4 split into 2 independent tasks (T-4.1, T-4.2).

---

## Downstream Verification

**Status:** No TD inconsistencies found during breakdown.

During work breakdown, the Technical Design Document was reviewed for:
- Interface contract completeness (all API shapes, Pydantic schemas, database models specified)
- Exit condition verifiability (all exit conditions use machine-verifiable commands)
- File structure clarity (all file paths map to actual or planned codebase structure)
- Dependency consistency (no circular dependencies, all inputs defined)

All TD sections were internally consistent and complete. No gaps, contradictions, or underspecified contracts were discovered that would require upstream revision.

The task splitting described above (TD Task 3 → T-3.1/T-3.2/T-3.3/T-3.4, TD Task 4 → T-4.1/T-4.2) was driven by task sizing constraints, not by TD defects. The TD correctly specified all necessary contracts; the work breakdown applied finer granularity to respect autonomous execution limits.

---

## Work Unit Map

```
WU-1 (Database)
  └─> T-1.1: Database schema, models, session factories
           |
           v
      WU-2 (API Core)
        └─> T-2.1: FastAPI app, config, auth, errors, rate limiting
                 |
                 v
            WU-3 (Schedule API)
              ├─> T-3.1: Pydantic schemas
              ├─> T-3.2: State machine + next_run service
              ├─> T-3.3: Schedule CRUD endpoints
              └─> T-3.4: Pause/resume endpoints
                       |
                       +──────────┬──────────+
                       |                     |
                       v                     v
                  WU-4 (Additional APIs)  WU-5 (Frontend) [parallel]
                    ├─> T-4.1: Run history endpoint
                    ├─> T-4.2: Report types endpoint
                    └─> T-4.3: Celery tasks + protocols
                                            |
                                            v
                                       ├─> T-5.1: Schemas, services, hooks
                                       └─> T-5.2: Pages and components
```

---

## Work Unit 1: Database Layer

**Agent:** @database-engineer
**Tasks:** T-1.1
**Estimated Duration:** 2–3 days

### Shared Context

**Binding Decisions:**
- 4 models: `Schedule`, `ScheduleRecipient`, `ScheduleRun`, `RunRecipient`
- 4 enums: `ScheduleStatus`, `CadenceType`, `RunStatus`, `RecipientDeliveryStatus`
- `RunRecipient.recipient_email` is a snapshot, not a FK to `schedule_recipients`
- `schedule_hour` and `schedule_minute` are integers (0-23, 0-59), not a `Time` column
- Soft delete for schedules (`status=deleted`) and recipients (`is_active=false`)

**Indexes:**
- `ix_schedules_tick(status, next_run_at)` — for tick task query
- `ix_schedules_owner(owner_id, status)` — for user schedule list
- `ix_recipients_schedule_active(schedule_id, is_active)` — for active recipient lookup
- `ix_runs_schedule_created(schedule_id, created_at)` — for run history queries

**Package Structure:**
```
packages/db/
├── src/
│   ├── db/
│   │   ├── enums.py
│   │   ├── models.py
│   │   └── database.py
│   └── __init__.py
├── alembic/
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
└── alembic.ini
```

---

### Task T-1.1: Database Schema, Models, and Migration

**Agent:** @database-engineer
**Size:** L
**Dependencies:** None
**Blocks:** T-2.1, T-3.1

#### Agent Prompt

**1. Read these files:**
- Project conventions: `/home/jary/redhat/git/claude-collective-example/.claude/rules/database-development.md` — SQLAlchemy patterns, migration workflow
- Architecture: `/home/jary/redhat/git/claude-collective-example/plans/architecture.md`, Section 2 "Component Boundaries > packages/db" — public exports
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 2 "Data Model" — complete model definitions

**2. Create these files:**

Create the following file structure in `packages/db/`:

```
packages/db/src/db/enums.py
packages/db/src/db/models.py
packages/db/src/db/database.py
packages/db/src/__init__.py
packages/db/alembic.ini
packages/db/alembic/env.py
packages/db/alembic/script.py.mako
```

**3. Implementation steps:**

**Step 1: Create enums (enums.py)**

Add the 4 enum classes exactly as specified in TD section 2.1:
- `ScheduleStatus` with values: ACTIVE, PAUSED, DELETED
- `CadenceType` with values: DAILY, WEEKLY, MONTHLY
- `RunStatus` with values: PENDING, GENERATING, GENERATED, DELIVERING, DELIVERED, PARTIALLY_DELIVERED, DELIVERY_FAILED, GENERATION_FAILED
- `RecipientDeliveryStatus` with values: PENDING, SENT, FAILED

All enums must inherit from `(str, enum.Enum)`.

**Step 2: Create database session factories (database.py)**

Implement:
- `Base` declarative base
- `engine` created from `DATABASE_URL` environment variable
- `get_session()` async context manager returning `AsyncSession`
- `get_sync_session()` sync context manager returning `Session` (for Celery workers)

Use the pattern from `database-development.md`:
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import Session, sessionmaker, declarative_base
```

**Step 3: Create models (models.py)**

Implement the 4 models exactly as specified in TD sections 2.2–2.5:

**Schedule model:**
- Fields: `id`, `owner_id`, `report_type_id`, `cadence_type`, `cadence_day`, `schedule_hour`, `schedule_minute`, `timezone`, `status`, `next_run_at`, `created_at`, `updated_at`
- Indexes: `ix_schedules_tick(status, next_run_at)`, `ix_schedules_owner(owner_id, status)`
- Relationships: `recipients` (1:N to ScheduleRecipient), `runs` (1:N to ScheduleRun)
- Server defaults: `status=active`, `created_at=now()`, `updated_at=now()` with `onupdate`

**ScheduleRecipient model:**
- Fields: `id`, `schedule_id`, `email`, `is_active`, `created_at`
- Index: `ix_recipients_schedule_active(schedule_id, is_active)`
- Foreign key: `schedule_id` → `schedules.id` with `ondelete=CASCADE`
- Relationship: `schedule` (N:1 to Schedule)

**ScheduleRun model:**
- Fields: `id`, `schedule_id`, `status`, `started_at`, `completed_at`, `error_message`, `created_at`
- Index: `ix_runs_schedule_created(schedule_id, created_at)`
- Foreign key: `schedule_id` → `schedules.id` with `ondelete=CASCADE`
- Relationships: `schedule` (N:1 to Schedule), `run_recipients` (1:N to RunRecipient)
- Server defaults: `status=pending`, `created_at=now()`

**RunRecipient model:**
- Fields: `id`, `run_id`, `recipient_email`, `status`, `error_message`, `delivered_at`
- Index: `ix_run_recipients_run(run_id)`
- Foreign key: `run_id` → `schedule_runs.id` with `ondelete=CASCADE`
- Relationship: `run` (N:1 to ScheduleRun)
- Server default: `status=pending`
- IMPORTANT: `recipient_email` is NOT a foreign key — it's a snapshot

**Step 4: Create public exports (__init__.py)**

Export as specified in TD section 2.6:
```python
from .db.database import Base, engine, get_session, get_sync_session
from .db.enums import CadenceType, RecipientDeliveryStatus, RunStatus, ScheduleStatus
from .db.models import RunRecipient, Schedule, ScheduleRecipient, ScheduleRun

__all__ = [
    "Base", "engine", "get_session", "get_sync_session",
    "CadenceType", "RecipientDeliveryStatus", "RunStatus", "ScheduleStatus",
    "Schedule", "ScheduleRecipient", "ScheduleRun", "RunRecipient",
]
```

**Step 5: Set up Alembic**

Initialize Alembic configuration:
```bash
cd /home/jary/redhat/git/claude-collective-example/packages/db
uv run alembic init alembic
```

Configure `alembic.ini`:
- Set `sqlalchemy.url` to read from environment variable
- Set `script_location = alembic`

Configure `alembic/env.py`:
- Import `Base` from `db`
- Set `target_metadata = Base.metadata`
- Configure async migrations if needed

**Step 6: Generate and review migration**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/db
uv run alembic revision --autogenerate -m "initial schema"
```

Review the generated migration in `alembic/versions/`. Verify:
- All 4 tables are created
- All 5 indexes are created
- All foreign keys have `ondelete=CASCADE`
- Enum columns use `String` type (not native enum)
- `server_default` values are present

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/db && \
  uv run alembic upgrade head && \
  uv run python -c "from db import Schedule, ScheduleRun, ScheduleRecipient, RunRecipient, get_session, get_sync_session; print('OK')"
```

Expected output: `OK`

#### Constraints

- Use SQLAlchemy 2.0 `Mapped[]` type hints, not old-style `Column()`
- All enums must use `native_enum=False` (store as strings, not Postgres ENUMs)
- All datetime columns must use `DateTime(timezone=True)`
- Foreign keys must specify `ondelete=CASCADE`
- No business logic in models — pure data definitions only

#### Test Expectations

Unit tests for this task are deferred to T-3.3 (schedule CRUD tests will exercise model CRUD). The verification command confirms models are importable and the migration runs successfully.

---

## Work Unit 2: API Core Infrastructure

**Agent:** @backend-developer
**Tasks:** T-2.1
**Estimated Duration:** 2–3 days

### Shared Context

**Binding Decisions:**
- FastAPI async app with lifespan context manager for startup/shutdown
- Auth: JWT-based with `get_current_user` dependency returning `(user_id: int, is_admin: bool)`
- Owner-scoped queries: pre-filter by `owner_id`, return 404 on not-found-or-not-owned
- Admin bypass: `is_admin=True` skips owner filter
- Rate limiting: 10 requests/minute/user on create/update endpoints
- RFC 7807 error responses for all 4xx/5xx errors

**Error Response Shape (RFC 7807):**
```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Schedule with ID '123' was not found.",
  "instance": "/v1/schedules/123"
}
```

**Package Structure:**
```
packages/api/
├── src/
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   ├── auth.py
│   │   ├── rate_limit.py
│   │   └── errors.py
│   ├── routes/
│   ├── schemas/
│   ├── services/
│   └── tasks/
└── tests/
```

---

### Task T-2.1: FastAPI App, Config, Auth, Errors, Rate Limiting

**Agent:** @backend-developer
**Size:** M
**Dependencies:** T-1.1 (database must exist)
**Blocks:** T-3.1, T-4.1

#### Agent Prompt

**1. Read these files:**
- Database models: `/home/jary/redhat/git/claude-collective-example/packages/db/src/__init__.py` — verify imports work
- Project conventions: `/home/jary/redhat/git/claude-collective-example/.claude/rules/api-development.md` — FastAPI patterns
- Error handling rules: `/home/jary/redhat/git/claude-collective-example/.claude/rules/error-handling.md` — RFC 7807 format
- Security rules: `/home/jary/redhat/git/claude-collective-example/.claude/rules/security.md` — auth requirements
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 7 "Cross-Cutting Concerns" — auth patterns, rate limiting

**2. Create these files:**

```
packages/api/src/main.py
packages/api/src/core/config.py
packages/api/src/core/auth.py
packages/api/src/core/rate_limit.py
packages/api/src/core/errors.py
packages/api/tests/test_health.py
```

**3. Implementation steps:**

**Step 1: Create configuration (core/config.py)**

Implement Pydantic Settings:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    debug: bool = False
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str  # used for token verification
    allowed_hosts: list[str] = ["*"]

    model_config = {"env_file": ".env"}

def get_settings() -> Settings:
    return Settings()
```

**Step 2: Create auth dependencies (core/auth.py)**

Implement:
- `get_current_user()` dependency that:
  - Reads `Authorization: Bearer <token>` header
  - Verifies JWT signature using `settings.jwt_secret`
  - Returns tuple `(user_id: int, is_admin: bool)`
  - Raises 401 if token is missing/invalid

- `owner_or_admin_query()` helper:
  - Takes `owner_id: int`, `is_admin: bool`
  - Returns SQLAlchemy filter: `None` if admin, `Schedule.owner_id == owner_id` otherwise

**Step 3: Create rate limiting (core/rate_limit.py)**

Implement Redis-backed rate limiter:
- `RateLimiter` class using Redis
- `rate_limit(max_requests: int, window_seconds: int)` dependency
- Default: 10 requests per 60 seconds per user
- Returns 429 with RFC 7807 error when exceeded

**Step 4: Create error handlers (core/errors.py)**

Implement:
- `ProblemDetail` Pydantic model for RFC 7807 responses
- Exception handlers for:
  - `HTTPException` → RFC 7807
  - `ValidationError` (Pydantic) → RFC 7807 with 422 status
  - Generic `Exception` → RFC 7807 with 500 status (log full stack, return sanitized message)

**Step 5: Create FastAPI app (main.py)**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from db import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: verify DB connection
    async with engine.begin() as conn:
        pass
    yield
    # Shutdown: close DB connections
    await engine.dispose()

app = FastAPI(
    title="Report Scheduling API",
    version="1.0.0",
    lifespan=lifespan,
)

# Register exception handlers from core.errors
# Register health endpoints

@app.get("/healthz")
def health():
    return {"status": "ok"}

@app.get("/readyz")
async def ready():
    # Check DB and Redis connectivity
    return {"status": "ready"}
```

**Step 6: Write health endpoint tests (tests/test_health.py)**

Write tests verifying:
- `GET /healthz` returns 200 with `{"status": "ok"}`
- `GET /readyz` returns 200 with `{"status": "ready"}` when DB and Redis are available
- Unauthenticated requests to protected endpoints return 401 with RFC 7807 error shape
- RFC 7807 error response includes `type`, `title`, `status`, `detail` fields

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_health.py -v
```

Expected: All tests pass.

#### Constraints

- Use async FastAPI patterns (async def for all route handlers)
- JWT verification must validate signature and expiration
- Health endpoints (`/healthz`, `/readyz`) must NOT require authentication
- All error responses must conform to RFC 7807 (see `error-handling.md`)
- Rate limiting must use Redis (not in-memory) for multi-worker deployments

#### Test Expectations

Integration tests in `test_health.py` cover:
- Health checks return expected status codes and JSON shapes
- Auth middleware returns 401 for missing/invalid tokens
- Error handlers produce RFC 7807 responses

---

## Work Unit 3: Schedule API Endpoints

**Agent:** @backend-developer
**Tasks:** T-3.1, T-3.2, T-3.3, T-3.4
**Estimated Duration:** 5–6 days

### Shared Context

**Files shared across all tasks in WU-3:**
- Database models: `packages/db/src/db/models.py`, `packages/db/src/db/enums.py`
- API core: `packages/api/src/core/auth.py`, `packages/api/src/core/errors.py`, `packages/api/src/core/rate_limit.py`
- Requirements: `/home/jary/redhat/git/claude-collective-example/plans/requirements.md`, Section 2 "User Stories US-001 through US-007" — acceptance criteria

**Binding Contracts:**
- API request/response shapes: TD Section 3 "API Contracts"
- Pydantic schemas: TD Section 4 "Pydantic Schemas"
- Owner-scoped query pattern: pre-filter by `owner_id`, return 404 on not-found
- State transition validation: return 409 on invalid transitions
- Recipient management: replacement semantics on PATCH
- Pause/resume: dedicated POST endpoints, not PATCH with status field

**State Machine Rules (ScheduleStatus transitions):**
```
ACTIVE   → PAUSED    (pause)
ACTIVE   → DELETED   (delete)
PAUSED   → ACTIVE    (resume)
PAUSED   → DELETED   (delete)
DELETED  → (no transitions — terminal state)
```

**Next Run Computation:**
- Daily: tomorrow at `schedule_hour:schedule_minute` in `timezone`
- Weekly: next occurrence of `cadence_day` at `schedule_hour:schedule_minute`
- Monthly: next occurrence of `cadence_day` (clamped to last day of month if day > month length)
- DST transitions: use `zoneinfo` to handle spring-forward/fall-back correctly

---

### Task T-3.1: Pydantic Request/Response Schemas

**Agent:** @backend-developer
**Size:** M
**Dependencies:** T-2.1
**Blocks:** T-3.3

#### Agent Prompt

**1. Read these files:**
- WU-3 shared context (above)
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 4 "Pydantic Schemas" — complete schema definitions
- Code style: `/home/jary/redhat/git/claude-collective-example/.claude/rules/code-style.md` — Python conventions
- Security rules: `/home/jary/redhat/git/claude-collective-example/.claude/rules/security.md` — CRLF injection prevention

**2. Create these files:**

```
packages/api/src/schemas/schedules.py
packages/api/src/schemas/responses.py
```

**3. Implementation steps:**

**Step 1: Create request schemas (schemas/schedules.py)**

Implement exactly as specified in TD section 4.1:

**RecipientCreate:**
- Fields: `email` (EmailStr)
- Validator: reject CRLF characters in email (Security Engineer WARNING-1)

**ScheduleCreate:**
- Fields: `report_type_id`, `cadence_type`, `cadence_day`, `schedule_time`, `timezone`, `recipients`
- Validators:
  - `validate_cadence_day`: ensure `cadence_day` matches `cadence_type` (None for daily, 0-6 for weekly, 1-31 for monthly)
  - `validate_timezone`: ensure IANA timezone is valid (use `zoneinfo.available_timezones()`)
  - `deduplicate_recipients`: normalize by lowercased email and remove duplicates
- Use `Field(alias="...")` for camelCase JSON (e.g., `report_type_id` ↔ `reportTypeId`)

**ScheduleUpdate:**
- All fields optional
- Same validators as ScheduleCreate
- IMPORTANT: route handlers must check `update_data.model_fields_set` to distinguish "field not provided" from "field set to null" (see TD section 4.1 docstring)

**Step 2: Create response schemas (schemas/responses.py)**

Implement exactly as specified in TD section 4.2:

**RecipientResponse:**
- Fields: `id`, `email`, `is_active` (alias: `isActive`)
- `model_config = ConfigDict(from_attributes=True, populate_by_name=True)`

**RunSummaryResponse:**
- Fields: `id`, `status`, `started_at`, `completed_at`, `error_message`, `created_at`
- All timestamp fields use camelCase aliases

**ScheduleResponse:**
- Fields: `id`, `report_type_id`, `cadence_type`, `cadence_day`, `schedule_time`, `timezone`, `status`, `next_run_at`, `recipients`, `recent_runs`, `created_at`, `updated_at`
- `recipients` is `list[RecipientResponse]`
- `recent_runs` is `list[RunSummaryResponse]` with `default_factory=list`

**ScheduleListItem:**
- Same as ScheduleResponse but without `recipients` and `recent_runs`
- Includes `last_run_status` and `last_run_at` (computed from most recent run)

**StatusUpdateResponse:**
- Fields: `id`, `status`, `next_run_at`

**RunRecipientResponse:**
- Fields: `recipient_email`, `status`, `error_message`, `delivered_at`

**RunDetailResponse:**
- Fields: `id`, `status`, `started_at`, `completed_at`, `error_message`, `recipients`, `created_at`
- `recipients` is `list[RunRecipientResponse]`

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run python -c "from schemas.schedules import ScheduleCreate, ScheduleUpdate, RecipientCreate; from schemas.responses import ScheduleResponse, ScheduleListItem; print('OK')"
```

Expected output: `OK` (schemas are importable)

#### Constraints

- Use Pydantic v2 (`BaseModel`, `ConfigDict`)
- All timestamp fields in responses use camelCase aliases
- Validate IANA timezones using `zoneinfo.available_timezones()`
- Reject CRLF characters in email addresses (Security Engineer WARNING-1)
- `ScheduleUpdate` must support PATCH partial-update semantics (check `model_fields_set`)

#### Test Expectations

Schema validation tests are covered by T-3.3 endpoint tests (POST/PATCH requests exercise schema validation).

---

### Task T-3.2: State Machine and Next Run Services

**Agent:** @backend-developer
**Size:** S
**Dependencies:** T-2.1, T-3.1
**Blocks:** T-3.3, T-3.4

#### Agent Prompt

**1. Read these files:**
- WU-3 shared context (above)
- Database enums: `/home/jary/redhat/git/claude-collective-example/packages/db/src/db/enums.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 2.8 "State Machine", Section 5.5 "Next Run Computation"
- Requirements: `/home/jary/redhat/git/claude-collective-example/plans/requirements.md`, US-002 (cadence), US-006 (pause/resume), US-009 (timezone/DST)

**2. Create these files:**

```
packages/api/src/services/state_machine.py
packages/api/src/services/next_run.py
packages/api/tests/test_state_machine.py
packages/api/tests/test_next_run.py
```

**3. Implementation steps:**

**Step 1: Implement state machine (services/state_machine.py)**

```python
from db import ScheduleStatus

def can_transition(from_status: ScheduleStatus, to_status: ScheduleStatus) -> bool:
    """Check if a state transition is valid."""
    valid_transitions = {
        ScheduleStatus.ACTIVE: {ScheduleStatus.PAUSED, ScheduleStatus.DELETED},
        ScheduleStatus.PAUSED: {ScheduleStatus.ACTIVE, ScheduleStatus.DELETED},
        ScheduleStatus.DELETED: set(),  # terminal state
    }
    return to_status in valid_transitions.get(from_status, set())

def validate_transition(from_status: ScheduleStatus, to_status: ScheduleStatus) -> None:
    """Raise 409 error if transition is invalid."""
    if not can_transition(from_status, to_status):
        raise HTTPException(
            status_code=409,
            detail=f"Cannot transition from {from_status.value} to {to_status.value}."
        )
```

**Step 2: Implement next_run computation (services/next_run.py)**

```python
from datetime import datetime
from zoneinfo import ZoneInfo
from db import CadenceType, Schedule

def compute_next_run(schedule: Schedule, base_time: datetime | None = None) -> datetime:
    """Compute the next scheduled run time in UTC.

    Args:
        schedule: Schedule model with cadence configuration
        base_time: Reference time (defaults to now in schedule's timezone)

    Returns:
        Next run datetime in UTC
    """
    # Implementation must handle:
    # - Daily: next occurrence at schedule_hour:schedule_minute
    # - Weekly: next occurrence of cadence_day (0=Mon..6=Sun)
    # - Monthly: next occurrence of cadence_day (1-31), clamped to last day of month
    # - DST transitions: use zoneinfo to handle correctly
    # - Base time: if None, use datetime.now(ZoneInfo(schedule.timezone))

    # Return UTC datetime (localized.astimezone(ZoneInfo("UTC")))
```

**Step 3: Write state machine tests (tests/test_state_machine.py)**

Test all valid and invalid transitions:
- ACTIVE → PAUSED (valid)
- ACTIVE → DELETED (valid)
- PAUSED → ACTIVE (valid)
- PAUSED → DELETED (valid)
- DELETED → ACTIVE (invalid, raises 409)
- DELETED → PAUSED (invalid, raises 409)

**Step 4: Write next_run tests (tests/test_next_run.py)**

Test cases from TD section 5.5:
- Daily: tomorrow at configured time
- Weekly: next occurrence of day-of-week
- Monthly: next occurrence of day-of-month
- Monthly day clamping: day 31 in February → Feb 28/29
- DST spring-forward: 2:30 AM → 3:30 AM (skip nonexistent hour)
- DST fall-back: 1:30 AM ambiguous → use first occurrence

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_state_machine.py tests/test_next_run.py -v
```

Expected: All tests pass.

#### Constraints

- Use `zoneinfo` (Python 3.9+) for timezone handling, not `pytz`
- Next run computation must return UTC datetime
- Monthly cadence must clamp day to last day of month if day > month length
- DST transitions must use `fold` parameter to disambiguate fall-back times

#### Test Expectations

Unit tests for both services covering all edge cases specified in the TD.

---

### Task T-3.3: Schedule CRUD Endpoints

**Agent:** @backend-developer
**Size:** L (5 files)
**Dependencies:** T-3.1, T-3.2
**Blocks:** T-4.1

#### Agent Prompt

**1. Read these files:**
- WU-3 shared context (above)
- State machine service: `/home/jary/redhat/git/claude-collective-example/packages/api/src/services/state_machine.py`
- Next run service: `/home/jary/redhat/git/claude-collective-example/packages/api/src/services/next_run.py`
- Schemas: `/home/jary/redhat/git/claude-collective-example/packages/api/src/schemas/schedules.py`, `/home/jary/redhat/git/claude-collective-example/packages/api/src/schemas/responses.py`
- Auth: `/home/jary/redhat/git/claude-collective-example/packages/api/src/core/auth.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 3 "API Contracts" (endpoints 3.1–3.5)

**2. Create these files:**

```
packages/api/src/routes/schedules.py
packages/api/tests/test_schedules.py
```

**3. Implementation steps:**

**Step 1: Create router (routes/schedules.py)**

Implement APIRouter with prefix `/v1/schedules` and tag `["schedules"]`.

**Step 2: Implement POST /v1/schedules (create)**

- Request: `ScheduleCreate`
- Response: `ScheduleResponse` (201 status)
- Auth: require `get_current_user`, extract `owner_id`
- Rate limit: 10 requests/minute/user
- Validation:
  - Validate `report_type_id` exists (hardcoded list for Phase 1)
  - Deduplicate recipients (already done in schema validator)
- Create:
  - Parse `schedule_time` ("HH:MM") into `schedule_hour` and `schedule_minute`
  - Create `Schedule` with `owner_id`, `status=ACTIVE`
  - Compute `next_run_at` using `compute_next_run()`
  - Create `ScheduleRecipient` records for all recipients
  - Commit transaction
- Return: `ScheduleResponse` with recipients populated

**Step 3: Implement GET /v1/schedules (list)**

- Response: paginated list of `ScheduleListItem`
- Auth: require `get_current_user`
- Query params: `cursor` (opaque), `limit` (default 20, max 100)
- Owner scoping: filter by `owner_id` (admin bypass)
- Cursor pagination: use `id` as cursor key
- Include: `last_run_status` and `last_run_at` (computed from most recent run via subquery or join)

**Step 4: Implement GET /v1/schedules/{id} (get detail)**

- Response: `ScheduleResponse`
- Auth: require `get_current_user`
- Owner scoping: filter by `owner_id`, return 404 if not found or not owned
- Include:
  - `recipients` (only where `is_active=true`)
  - `recent_runs` (last 10 runs ordered by `created_at DESC`)

**Step 5: Implement PATCH /v1/schedules/{id} (update)**

- Request: `ScheduleUpdate`
- Response: `ScheduleResponse` (200 status)
- Auth: require `get_current_user`
- Rate limit: 10 requests/minute/user
- Owner scoping: filter by `owner_id`, return 404 if not found
- PATCH semantics:
  - Check `update_data.model_fields_set` to identify provided fields
  - Update only provided fields
  - If `recipients` is provided: soft-delete all existing (`is_active=false`), create new ones
  - If `cadence_type`, `cadence_day`, or `schedule_time` changed: recompute `next_run_at`
- Return: updated `ScheduleResponse`

**Step 6: Implement DELETE /v1/schedules/{id} (soft delete)**

- Response: 204 No Content
- Auth: require `get_current_user`
- Owner scoping: filter by `owner_id`, return 404 if not found
- Soft delete: set `status=DELETED`, validate transition using `validate_transition()`

**Step 7: Write tests (tests/test_schedules.py)**

Tests must verify (from TD section 9, Task 3 exit condition):
- POST creates schedule, returns 201 with correct shape
- GET list returns paginated schedules scoped to owner
- GET detail returns schedule with recipients
- PATCH updates fields and recomputes `next_run_at`
- DELETE soft-deletes
- Authorization: 404 when accessing another user's schedule
- Validation: invalid timezone, cadence_day, recipients (min 1, max 50)
- Rate limiting: 429 after 10 requests/minute

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_schedules.py -v
```

Expected: All tests pass.

#### Constraints

- Owner-scoped queries must pre-filter by `owner_id` and return 404 on not-found
- Recipients are managed inline (no separate recipient endpoints)
- PATCH recipients: replacement semantics (soft-delete all, create new)
- Rate limit only on POST and PATCH (not on GET or DELETE)
- Soft delete: set `status=DELETED`, do not remove from database

#### Test Expectations

Integration tests covering all endpoints, authorization, validation, pagination, and rate limiting.

---

### Task T-3.4: Pause/Resume Endpoints

**Agent:** @backend-developer
**Size:** S
**Dependencies:** T-3.2, T-3.3
**Blocks:** None

#### Agent Prompt

**1. Read these files:**
- WU-3 shared context (above)
- State machine service: `/home/jary/redhat/git/claude-collective-example/packages/api/src/services/state_machine.py`
- Schedules router: `/home/jary/redhat/git/claude-collective-example/packages/api/src/routes/schedules.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 3.6–3.7 "Pause/Resume Endpoints"

**2. Modify these files:**

```
packages/api/src/routes/schedules.py (add 2 endpoints)
packages/api/tests/test_schedules.py (add pause/resume tests)
```

**3. Implementation steps:**

**Step 1: Add POST /v1/schedules/{id}/pause**

- Response: `StatusUpdateResponse` (200 status)
- Auth: require `get_current_user`
- Owner scoping: filter by `owner_id`, return 404 if not found
- Validate: current status is ACTIVE (use `validate_transition()`, return 409 if invalid)
- Update: set `status=PAUSED`, set `next_run_at=None` (no scheduled runs while paused)
- Return: `StatusUpdateResponse` with new status

**Step 2: Add POST /v1/schedules/{id}/resume**

- Response: `StatusUpdateResponse` (200 status)
- Auth: require `get_current_user`
- Owner scoping: filter by `owner_id`, return 404 if not found
- Validate: current status is PAUSED (use `validate_transition()`, return 409 if invalid)
- Update: set `status=ACTIVE`, recompute `next_run_at` using `compute_next_run()`
- Return: `StatusUpdateResponse` with new status

**Step 3: Add tests (tests/test_schedules.py)**

Tests must verify:
- Pause: ACTIVE → PAUSED (valid, `next_run_at` becomes null)
- Pause: PAUSED → PAUSED (409 conflict)
- Pause: DELETED → PAUSED (409 conflict)
- Resume: PAUSED → ACTIVE (valid, `next_run_at` recomputed)
- Resume: ACTIVE → ACTIVE (409 conflict)
- Resume: DELETED → ACTIVE (409 conflict)

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_schedules.py::test_pause_schedule tests/test_schedules.py::test_resume_schedule -v
```

Expected: All tests pass.

#### Constraints

- Pause: set `next_run_at=None` (no scheduled runs while paused)
- Resume: recompute `next_run_at` using `compute_next_run()`
- State validation: use `validate_transition()` to ensure valid transitions
- Return 409 (not 400) for invalid state transitions

#### Test Expectations

Unit tests covering all valid and invalid state transitions for pause and resume.

---

## Work Unit 4: Additional API Endpoints

**Agent:** @backend-developer
**Tasks:** T-4.1, T-4.2, T-4.3
**Estimated Duration:** 5–6 days

### Shared Context

**Files shared across all tasks in WU-4:**
- Database models: `packages/db/src/db/models.py`, `packages/db/src/db/enums.py`
- Schemas: `packages/api/src/schemas/responses.py`
- Auth: `packages/api/src/core/auth.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`

**Binding Contracts:**
- Run history endpoint: Section 3.8 "GET /v1/schedules/{id}/runs"
- Report types endpoint: Section 3.9 "GET /v1/report-types"
- Celery tasks: Section 5 "Celery Tasks"

---

### Task T-4.1: Delivery History Endpoint

**Agent:** @backend-developer
**Size:** S
**Dependencies:** T-3.3 (schedule routes must exist to create test data)
**Blocks:** None

#### Agent Prompt

**1. Read these files:**
- WU-4 shared context (above)
- Response schemas: `/home/jary/redhat/git/claude-collective-example/packages/api/src/schemas/responses.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 3.8 "Delivery History Endpoint"
- Requirements: `/home/jary/redhat/git/claude-collective-example/plans/requirements.md`, US-008 (delivery status visibility)

**2. Create these files:**

```
packages/api/src/routes/schedule_runs.py
packages/api/tests/test_schedule_runs.py
```

**3. Implementation steps:**

**Step 1: Create router (routes/schedule_runs.py)**

Implement APIRouter with prefix `/v1/schedules/{schedule_id}/runs` and tag `["runs"]`.

**Step 2: Implement GET /v1/schedules/{schedule_id}/runs**

- Response: paginated list of `RunDetailResponse`
- Auth: require `get_current_user`
- Owner scoping: verify schedule belongs to current user (query schedule with owner filter, return 404 if not found)
- Query params: `cursor` (opaque), `limit` (default 20, max 100), `status` (optional filter)
- Cursor pagination: use `id` as cursor key
- Eager load: include `run_recipients` for each run
- Order: `created_at DESC` (most recent first)

**Step 3: Register router**

Add to `packages/api/src/main.py`:
```python
from routes.schedule_runs import router as schedule_runs_router
app.include_router(schedule_runs_router)
```

**Step 4: Write tests (tests/test_schedule_runs.py)**

Tests must verify (from TD section 9, Task 4 exit condition):
- GET returns paginated runs with per-recipient detail
- Cursor pagination works (next page returns correct items)
- Status filter works (only returns runs with matching status)
- Authorization: 404 for runs of non-owned schedule

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_schedule_runs.py -v
```

Expected: All tests pass.

#### Constraints

- Owner scoping: verify schedule ownership before returning runs
- Eager load `run_recipients` to avoid N+1 queries
- Cursor pagination using `id` field (not offset-based)
- Status filter: optional, accepts `RunStatus` enum values

#### Test Expectations

Integration tests covering pagination, filtering, and authorization.

---

### Task T-4.2: Report Types Endpoint

**Agent:** @backend-developer
**Size:** XS
**Dependencies:** T-2.1
**Blocks:** None

#### Agent Prompt

**1. Read these files:**
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 3.9 "Report Types Endpoint"

**2. Create these files:**

```
packages/api/src/routes/report_types.py
packages/api/tests/test_report_types.py
```

**3. Implementation steps:**

**Step 1: Create router (routes/report_types.py)**

Implement APIRouter with prefix `/v1/report-types` and tag `["report-types"]`.

**Step 2: Implement GET /v1/report-types**

- Response: `{"reportTypes": [{"id": "...", "name": "...", "description": "..."}]}`
- Auth: require `get_current_user` (no owner scoping — all users see the same list)
- Hardcoded list for Phase 1:
  ```python
  REPORT_TYPES = [
      {"id": "sales-summary", "name": "Sales Summary", "description": "Weekly sales performance"},
      {"id": "inventory-status", "name": "Inventory Status", "description": "Current stock levels"},
  ]
  ```

**Step 3: Register router**

Add to `packages/api/src/main.py`:
```python
from routes.report_types import router as report_types_router
app.include_router(report_types_router)
```

**Step 4: Write tests (tests/test_report_types.py)**

Tests must verify:
- GET returns hardcoded list
- Response includes `id`, `name`, `description` for each type

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_report_types.py -v
```

Expected: All tests pass.

#### Constraints

- Phase 1: hardcoded list (no database table)
- Must require authentication (all users see the same list, but unauthenticated requests get 401)

#### Test Expectations

Simple integration test verifying the endpoint returns the expected static list.

---

### Task T-4.3: Celery Tasks and Service Protocols

**Agent:** @backend-developer
**Size:** L (5 files, plus protocol definitions)
**Dependencies:** T-3.2 (state machine, next_run)
**Blocks:** None

#### Agent Prompt

**1. Read these files:**
- WU-4 shared context (above)
- State machine: `/home/jary/redhat/git/claude-collective-example/packages/api/src/services/state_machine.py`
- Next run: `/home/jary/redhat/git/claude-collective-example/packages/api/src/services/next_run.py`
- Database models: `/home/jary/redhat/git/claude-collective-example/packages/db/src/db/models.py`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 5 "Celery Tasks", Section 6 "Service Protocols"
- Requirements: `/home/jary/redhat/git/claude-collective-example/plans/requirements.md`, US-004 (email delivery pipeline)

**2. Create these files:**

```
packages/api/src/tasks/__init__.py
packages/api/src/tasks/tick.py
packages/api/src/tasks/execute.py
packages/api/src/tasks/deliver.py
packages/api/src/services/report_generator.py
packages/api/src/services/email_sender.py
packages/api/tests/test_tasks.py
```

**3. Implementation steps:**

**Step 1: Initialize Celery app (tasks/__init__.py)**

```python
from celery import Celery

celery_app = Celery(
    "report_scheduler",
    broker=os.getenv("CELERY_BROKER_URL", "redis://localhost:6379/0"),
    backend=os.getenv("CELERY_RESULT_BACKEND", "redis://localhost:6379/0"),
)

celery_app.conf.update(
    task_acks_late=True,
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
)
```

**Step 2: Define service protocols**

**ReportGenerator protocol (services/report_generator.py):**
```python
from typing import Protocol

class ReportOutput:
    content: bytes
    content_type: str

class ReportGenerator(Protocol):
    def generate(self, report_type_id: str, schedule_id: int) -> ReportOutput:
        """Generate a report. Raises ReportGenerationError on failure."""
        ...

# Stub implementation for Phase 1
class StubReportGenerator:
    def generate(self, report_type_id: str, schedule_id: int) -> ReportOutput:
        return ReportOutput(
            content=b"Stub report content",
            content_type="text/plain"
        )
```

**EmailSender protocol (services/email_sender.py):**
```python
from typing import Protocol

class EmailSender(Protocol):
    def send(
        self,
        to_email: str,
        subject: str,
        html_body: str,
        attachment_content: bytes,
        attachment_content_type: str,
        attachment_filename: str,
    ) -> None:
        """Send an email. Raises EmailDeliveryError on failure."""
        ...

# SMTP implementation for Phase 1
class SMTPEmailSender:
    def __init__(self, smtp_host: str, smtp_port: int):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port

    def send(self, ...):
        # Implement using smtplib
        # IMPORTANT: validate to_email to reject CRLF characters (Security Engineer WARNING-1)
        ...
```

**Step 3: Implement tick task (tasks/tick.py)**

```python
from datetime import datetime, timezone
from db import Schedule, ScheduleRun, ScheduleStatus, RunStatus, get_sync_session
from services.next_run import compute_next_run
from tasks import celery_app
from tasks.execute import execute_schedule

@celery_app.task
def tick_task():
    """Find schedules due for execution and dispatch execute tasks.

    Runs every 60 seconds via Celery Beat.
    """
    with get_sync_session() as session:
        now = datetime.now(timezone.utc)

        # Query schedules where status=ACTIVE and next_run_at <= now
        due_schedules = session.query(Schedule).filter(
            Schedule.status == ScheduleStatus.ACTIVE,
            Schedule.next_run_at <= now,
        ).all()

        for schedule in due_schedules:
            # Create run record
            run = ScheduleRun(
                schedule_id=schedule.id,
                status=RunStatus.PENDING,
            )
            session.add(run)

            # Compute next_run_at
            schedule.next_run_at = compute_next_run(schedule)

            session.commit()

            # Dispatch execute task
            execute_schedule.delay(run.id)
```

**Step 4: Implement execute task (tasks/execute.py)**

```python
from db import ScheduleRun, RunStatus, get_sync_session
from services.report_generator import StubReportGenerator
from tasks import celery_app
from tasks.deliver import deliver_report

@celery_app.task(max_retries=0)
def execute_schedule(run_id: int):
    """Generate the report for a schedule run.

    State transitions:
    - PENDING → GENERATING → GENERATED (success, dispatch deliver)
    - PENDING → GENERATING → GENERATION_FAILED (failure, terminal)
    """
    # Input validation (Security Engineer WARNING-2)
    if not isinstance(run_id, int) or run_id <= 0:
        raise ValueError("Invalid run_id")

    with get_sync_session() as session:
        run = session.get(ScheduleRun, run_id)
        if not run:
            raise ValueError(f"Run {run_id} not found")

        if run.status != RunStatus.PENDING:
            raise ValueError(f"Run {run_id} is not in PENDING state")

        # Transition to GENERATING
        run.status = RunStatus.GENERATING
        run.started_at = datetime.now(timezone.utc)
        session.commit()

        try:
            # Generate report
            generator = StubReportGenerator()
            report = generator.generate(
                run.schedule.report_type_id,
                run.schedule_id,
            )

            # Transition to GENERATED
            run.status = RunStatus.GENERATED
            session.commit()

            # Dispatch deliver task
            deliver_report.delay(
                run_id=run.id,
                report_content=base64.b64encode(report.content).decode(),
                content_type=report.content_type,
            )
        except Exception as e:
            # Transition to GENERATION_FAILED
            run.status = RunStatus.GENERATION_FAILED
            run.completed_at = datetime.now(timezone.utc)
            run.error_message = str(e)[:1000]
            session.commit()
```

**Step 5: Implement deliver task (tasks/deliver.py)**

```python
from db import ScheduleRun, RunRecipient, RunStatus, RecipientDeliveryStatus, get_sync_session
from services.email_sender import SMTPEmailSender
from tasks import celery_app

@celery_app.task(max_retries=0)
def deliver_report(run_id: int, report_content: str, content_type: str):
    """Deliver report to all active recipients.

    State transitions:
    - GENERATED → DELIVERING → DELIVERED (all success)
    - GENERATED → DELIVERING → PARTIALLY_DELIVERED (some failures)
    - GENERATED → DELIVERING → DELIVERY_FAILED (all failures)
    """
    # Input validation
    if not isinstance(run_id, int) or run_id <= 0:
        raise ValueError("Invalid run_id")

    report_bytes = base64.b64decode(report_content)

    with get_sync_session() as session:
        run = session.get(ScheduleRun, run_id)
        if not run:
            raise ValueError(f"Run {run_id} not found")

        if run.status != RunStatus.GENERATED:
            raise ValueError(f"Run {run_id} is not in GENERATED state")

        # Transition to DELIVERING
        run.status = RunStatus.DELIVERING
        session.commit()

        # Get active recipients
        recipients = [r for r in run.schedule.recipients if r.is_active]

        # Create RunRecipient records
        for recipient in recipients:
            run_recipient = RunRecipient(
                run_id=run.id,
                recipient_email=recipient.email,
                status=RecipientDeliveryStatus.PENDING,
            )
            session.add(run_recipient)
        session.commit()

        # Send emails
        sender = SMTPEmailSender(
            smtp_host=os.getenv("SMTP_HOST", "localhost"),
            smtp_port=int(os.getenv("SMTP_PORT", "1025")),
        )

        success_count = 0
        for run_recipient in run.run_recipients:
            try:
                sender.send(
                    to_email=run_recipient.recipient_email,
                    subject=f"Scheduled Report: {run.schedule.report_type_id}",
                    html_body="<p>Your scheduled report is attached.</p>",
                    attachment_content=report_bytes,
                    attachment_content_type=content_type,
                    attachment_filename="report.txt",
                )
                run_recipient.status = RecipientDeliveryStatus.SENT
                run_recipient.delivered_at = datetime.now(timezone.utc)
                success_count += 1
            except Exception as e:
                run_recipient.status = RecipientDeliveryStatus.FAILED
                run_recipient.error_message = str(e)[:1000]

        # Update run status
        total = len(run.run_recipients)
        if success_count == total:
            run.status = RunStatus.DELIVERED
        elif success_count > 0:
            run.status = RunStatus.PARTIALLY_DELIVERED
        else:
            run.status = RunStatus.DELIVERY_FAILED

        run.completed_at = datetime.now(timezone.utc)
        session.commit()
```

**Step 6: Write tests (tests/test_tasks.py)**

Tests must verify (from TD section 9, Task 5 exit condition):
- Tick task finds due schedules and creates runs
- Tick task skips paused/deleted schedules
- Tick task computes next_run_at correctly
- Execute task transitions run through generating → generated
- Execute task handles generation failure → generation_failed (terminal)
- Deliver task sends to all recipients and records results
- Deliver task handles partial failure → partially_delivered
- Deliver task handles total failure → delivery_failed
- Input validation at task entry (invalid run_id, wrong state)
- CRLF rejection in email addresses

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/api && \
  uv run pytest tests/test_tasks.py -v
```

Expected: All tests pass.

#### Constraints

- Workers use synchronous DB sessions (`get_sync_session()`), not async
- Report content is base64-encoded for JSON serialization via Celery
- No automatic retries in Phase 1 (`max_retries=0`)
- `task_acks_late=True` and `task_reject_on_worker_lost=True` for crash safety
- Email sender must validate email addresses to reject CRLF characters (Security Engineer WARNING-1)
- Tasks must validate inputs at entry (Security Engineer WARNING-2)

#### Test Expectations

Integration tests covering the full pipeline: tick → execute → deliver, including all state transitions and error paths.

---

## Work Unit 5: Frontend

**Agent:** @frontend-developer
**Tasks:** T-5.1, T-5.2
**Estimated Duration:** 5–6 days (parallel with WU-3, WU-4)

### Shared Context

**Technology Stack:**
- React 19, TypeScript
- TanStack Router (file-based routing)
- TanStack Query (server state)
- Zod (runtime validation)
- Tailwind CSS + shadcn/ui

**API Integration Pattern:**
```
Component → Hook → TanStack Query → Service → API
```

**Project Structure:**
```
packages/ui/
├── src/
│   ├── schemas/
│   ├── services/
│   ├── hooks/
│   ├── routes/
│   └── components/
```

**Binding Contracts:**
- API contracts from TD Section 3 define JSON shapes
- Zod schemas mirror Pydantic response schemas
- camelCase JSON field names (already in API responses via Pydantic aliases)

---

### Task T-5.1: Schemas, Services, and Hooks

**Agent:** @frontend-developer
**Size:** M
**Dependencies:** None (schemas defined from API contracts, not from running backend)
**Blocks:** T-5.2

#### Agent Prompt

**1. Read these files:**
- UI conventions: `/home/jary/redhat/git/claude-collective-example/.claude/rules/ui-development.md` — TanStack patterns
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 3 "API Contracts", Section 4.2 "Response Schemas"
- Architecture: `/home/jary/redhat/git/claude-collective-example/plans/architecture.md` — frontend package structure

**2. Create these files:**

```
packages/ui/src/schemas/schedule.ts
packages/ui/src/services/schedules.ts
packages/ui/src/services/report-types.ts
packages/ui/src/hooks/schedules.ts
packages/ui/src/hooks/report-types.ts
```

**3. Implementation steps:**

**Step 1: Define Zod schemas (schemas/schedule.ts)**

Create Zod schemas mirroring the Pydantic response schemas from TD section 4.2:

```typescript
import { z } from 'zod';

export const cadenceTypeSchema = z.enum(['daily', 'weekly', 'monthly']);
export const scheduleStatusSchema = z.enum(['active', 'paused', 'deleted']);
export const runStatusSchema = z.enum([
  'pending', 'generating', 'generated', 'delivering',
  'delivered', 'partially_delivered', 'delivery_failed', 'generation_failed'
]);
export const recipientDeliveryStatusSchema = z.enum(['pending', 'sent', 'failed']);

export const recipientResponseSchema = z.object({
  id: z.number(),
  email: z.string().email(),
  isActive: z.boolean(),
});

export const runSummaryResponseSchema = z.object({
  id: z.number(),
  status: runStatusSchema,
  startedAt: z.string().datetime().nullable(),
  completedAt: z.string().datetime().nullable(),
  errorMessage: z.string().nullable(),
  createdAt: z.string().datetime(),
});

export const scheduleResponseSchema = z.object({
  id: z.number(),
  reportTypeId: z.string(),
  cadenceType: cadenceTypeSchema,
  cadenceDay: z.number().nullable(),
  scheduleTime: z.string().regex(/^([01]\d|2[0-3]):[0-5]\d$/),
  timezone: z.string(),
  status: scheduleStatusSchema,
  nextRunAt: z.string().datetime().nullable(),
  recipients: z.array(recipientResponseSchema),
  recentRuns: z.array(runSummaryResponseSchema),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const scheduleListItemSchema = z.object({
  id: z.number(),
  reportTypeId: z.string(),
  cadenceType: cadenceTypeSchema,
  cadenceDay: z.number().nullable(),
  scheduleTime: z.string(),
  timezone: z.string(),
  status: scheduleStatusSchema,
  nextRunAt: z.string().datetime().nullable(),
  lastRunStatus: runStatusSchema.nullable(),
  lastRunAt: z.string().datetime().nullable(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const reportTypeSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string(),
});

// TypeScript types
export type CadenceType = z.infer<typeof cadenceTypeSchema>;
export type ScheduleStatus = z.infer<typeof scheduleStatusSchema>;
export type RunStatus = z.infer<typeof runStatusSchema>;
export type RecipientResponse = z.infer<typeof recipientResponseSchema>;
export type ScheduleResponse = z.infer<typeof scheduleResponseSchema>;
export type ScheduleListItem = z.infer<typeof scheduleListItemSchema>;
export type ReportType = z.infer<typeof reportTypeSchema>;
```

**Step 2: Create API service (services/schedules.ts)**

```typescript
import { z } from 'zod';
import {
  scheduleListItemSchema,
  scheduleResponseSchema,
  type ScheduleResponse,
  type ScheduleListItem,
} from '@/schemas/schedule';

const API_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

async function fetchWithAuth(url: string, options?: RequestInit) {
  const token = localStorage.getItem('auth_token');
  return fetch(url, {
    ...options,
    headers: {
      ...options?.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
  });
}

export async function fetchSchedules(cursor?: string, limit = 20): Promise<{
  data: ScheduleListItem[];
  nextCursor: string | null;
}> {
  const params = new URLSearchParams();
  if (cursor) params.set('cursor', cursor);
  params.set('limit', String(limit));

  const response = await fetchWithAuth(`${API_URL}/v1/schedules?${params}`);
  if (!response.ok) throw new Error('Failed to fetch schedules');

  const json = await response.json();
  return {
    data: z.array(scheduleListItemSchema).parse(json.data),
    nextCursor: json.pagination?.nextCursor ?? null,
  };
}

export async function fetchSchedule(id: number): Promise<ScheduleResponse> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules/${id}`);
  if (!response.ok) throw new Error('Failed to fetch schedule');
  return scheduleResponseSchema.parse(await response.json());
}

export async function createSchedule(data: unknown): Promise<ScheduleResponse> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules`, {
    method: 'POST',
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create schedule');
  return scheduleResponseSchema.parse(await response.json());
}

export async function updateSchedule(id: number, data: unknown): Promise<ScheduleResponse> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules/${id}`, {
    method: 'PATCH',
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to update schedule');
  return scheduleResponseSchema.parse(await response.json());
}

export async function deleteSchedule(id: number): Promise<void> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules/${id}`, {
    method: 'DELETE',
  });
  if (!response.ok) throw new Error('Failed to delete schedule');
}

export async function pauseSchedule(id: number): Promise<ScheduleResponse> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules/${id}/pause`, {
    method: 'POST',
  });
  if (!response.ok) throw new Error('Failed to pause schedule');
  return scheduleResponseSchema.parse(await response.json());
}

export async function resumeSchedule(id: number): Promise<ScheduleResponse> {
  const response = await fetchWithAuth(`${API_URL}/v1/schedules/${id}/resume`, {
    method: 'POST',
  });
  if (!response.ok) throw new Error('Failed to resume schedule');
  return scheduleResponseSchema.parse(await response.json());
}
```

**Step 3: Create report types service (services/report-types.ts)**

```typescript
import { z } from 'zod';
import { reportTypeSchema, type ReportType } from '@/schemas/schedule';

const API_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

export async function fetchReportTypes(): Promise<ReportType[]> {
  const token = localStorage.getItem('auth_token');
  const response = await fetch(`${API_URL}/v1/report-types`, {
    headers: {
      'Authorization': `Bearer ${token}`,
    },
  });
  if (!response.ok) throw new Error('Failed to fetch report types');
  const json = await response.json();
  return z.array(reportTypeSchema).parse(json.reportTypes);
}
```

**Step 4: Create TanStack Query hooks (hooks/schedules.ts)**

```typescript
import {
  useQuery,
  useMutation,
  useQueryClient,
  type UseQueryResult,
  type UseMutationResult,
} from '@tanstack/react-query';
import {
  fetchSchedules,
  fetchSchedule,
  createSchedule,
  updateSchedule,
  deleteSchedule,
  pauseSchedule,
  resumeSchedule,
} from '@/services/schedules';
import type { ScheduleResponse, ScheduleListItem } from '@/schemas/schedule';

export function useSchedules(cursor?: string) {
  return useQuery({
    queryKey: ['schedules', cursor],
    queryFn: () => fetchSchedules(cursor),
  });
}

export function useSchedule(id: number) {
  return useQuery({
    queryKey: ['schedules', id],
    queryFn: () => fetchSchedule(id),
  });
}

export function useCreateSchedule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: createSchedule,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
    },
  });
}

export function useUpdateSchedule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: unknown }) => updateSchedule(id, data),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
      queryClient.invalidateQueries({ queryKey: ['schedules', variables.id] });
    },
  });
}

export function useDeleteSchedule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: deleteSchedule,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
    },
  });
}

export function usePauseSchedule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: pauseSchedule,
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
      queryClient.invalidateQueries({ queryKey: ['schedules', data.id] });
    },
  });
}

export function useResumeSchedule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: resumeSchedule,
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
      queryClient.invalidateQueries({ queryKey: ['schedules', data.id] });
    },
  });
}
```

**Step 5: Create report types hook (hooks/report-types.ts)**

```typescript
import { useQuery } from '@tanstack/react-query';
import { fetchReportTypes } from '@/services/report-types';

export function useReportTypes() {
  return useQuery({
    queryKey: ['report-types'],
    queryFn: fetchReportTypes,
  });
}
```

**Step 6: Write tests**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/ui && \
  pnpm vitest run --reporter=verbose src/schemas/ src/hooks/
```

Tests verify:
- Zod schemas parse valid API responses correctly
- Zod schemas reject invalid shapes
- Hook query keys match expected structure

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/ui && \
  pnpm vitest run --reporter=verbose src/schemas/ src/hooks/
```

Expected: All tests pass.

#### Constraints

- Zod schemas must match API response shapes exactly (camelCase field names)
- Services must use `fetchWithAuth()` helper for Authorization header
- TanStack Query hooks must invalidate cache on mutations
- Auth token stored in `localStorage` (not secure for production, but acceptable for MVP)

#### Test Expectations

Unit tests for Zod schemas (parsing valid/invalid data) and hook structure tests.

---

### Task T-5.2: Pages and Components

**Agent:** @frontend-developer
**Size:** L (5 primary files + supporting components)
**Dependencies:** T-5.1
**Blocks:** None

#### Agent Prompt

**1. Read these files:**
- UI conventions: `/home/jary/redhat/git/claude-collective-example/.claude/rules/ui-development.md` — component patterns, atomic design
- Hooks: `/home/jary/redhat/git/claude-collective-example/packages/ui/src/hooks/schedules.ts`, `/home/jary/redhat/git/claude-collective-example/packages/ui/src/hooks/report-types.ts`
- Schemas: `/home/jary/redhat/git/claude-collective-example/packages/ui/src/schemas/schedule.ts`
- Technical Design: `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`, Section 8 "Frontend Components"

**2. Create these files:**

Primary pages:
```
packages/ui/src/routes/schedules/index.tsx
packages/ui/src/routes/schedules/create.tsx
packages/ui/src/routes/schedules/$id.tsx
```

Organism components:
```
packages/ui/src/components/organisms/schedule-form.tsx
packages/ui/src/components/organisms/delivery-history.tsx
```

Supporting atom/molecule components (create as needed):
```
packages/ui/src/components/atoms/status-badge.tsx
packages/ui/src/components/molecules/cadence-selector.tsx
packages/ui/src/components/molecules/timezone-selector.tsx
packages/ui/src/components/molecules/recipient-input.tsx
packages/ui/src/components/molecules/confirm-dialog.tsx
packages/ui/src/components/molecules/run-detail.tsx
```

**3. Implementation steps:**

**Step 1: Create schedule list page (routes/schedules/index.tsx)**

```tsx
import { createFileRoute } from '@tanstack/react-router';
import { useSchedules } from '@/hooks/schedules';
import { ScheduleListItem } from '@/schemas/schedule';

export const Route = createFileRoute('/schedules/')({
  component: ScheduleList,
});

function ScheduleList() {
  const { data, isLoading, error } = useSchedules();

  if (isLoading) return <div>Loading schedules...</div>;
  if (error) return <div>Error loading schedules: {error.message}</div>;
  if (!data?.data.length) return <div>No schedules found. Create your first schedule!</div>;

  return (
    <div>
      <h1>My Schedules</h1>
      <a href="/schedules/create">Create New Schedule</a>
      <table>
        <thead>
          <tr>
            <th>Report Type</th>
            <th>Cadence</th>
            <th>Next Run</th>
            <th>Status</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {data.data.map((schedule) => (
            <tr key={schedule.id}>
              <td>{schedule.reportTypeId}</td>
              <td>{schedule.cadenceType} {schedule.cadenceDay && `(Day ${schedule.cadenceDay})`}</td>
              <td>{schedule.nextRunAt ? new Date(schedule.nextRunAt).toLocaleString() : 'N/A'}</td>
              <td><StatusBadge status={schedule.status} /></td>
              <td>
                <a href={`/schedules/${schedule.id}`}>View</a>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

**Step 2: Create schedule detail page (routes/schedules/$id.tsx)**

```tsx
import { createFileRoute } from '@tanstack/react-router';
import { useSchedule, useDeleteSchedule, usePauseSchedule, useResumeSchedule } from '@/hooks/schedules';
import { StatusBadge } from '@/components/atoms/status-badge';
import { DeliveryHistory } from '@/components/organisms/delivery-history';
import { ConfirmDialog } from '@/components/molecules/confirm-dialog';

export const Route = createFileRoute('/schedules/$id')({
  component: ScheduleDetail,
});

function ScheduleDetail() {
  const { id } = Route.useParams();
  const { data: schedule, isLoading, error } = useSchedule(Number(id));
  const deleteMutation = useDeleteSchedule();
  const pauseMutation = usePauseSchedule();
  const resumeMutation = useResumeSchedule();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!schedule) return <div>Schedule not found</div>;

  return (
    <div>
      <h1>Schedule Details</h1>
      <div>
        <p><strong>Report Type:</strong> {schedule.reportTypeId}</p>
        <p><strong>Cadence:</strong> {schedule.cadenceType}</p>
        <p><strong>Time:</strong> {schedule.scheduleTime} ({schedule.timezone})</p>
        <p><strong>Status:</strong> <StatusBadge status={schedule.status} /></p>
        <p><strong>Next Run:</strong> {schedule.nextRunAt ? new Date(schedule.nextRunAt).toLocaleString() : 'N/A'}</p>
      </div>

      <h2>Recipients</h2>
      <ul>
        {schedule.recipients.filter(r => r.isActive).map(r => (
          <li key={r.id}>{r.email}</li>
        ))}
      </ul>

      <h2>Recent Runs</h2>
      <DeliveryHistory scheduleId={schedule.id} recentRuns={schedule.recentRuns} />

      <div>
        <a href={`/schedules/${schedule.id}/edit`}>Edit</a>
        {schedule.status === 'active' && (
          <button onClick={() => pauseMutation.mutate(schedule.id)}>Pause</button>
        )}
        {schedule.status === 'paused' && (
          <button onClick={() => resumeMutation.mutate(schedule.id)}>Resume</button>
        )}
        <ConfirmDialog
          trigger={<button>Delete</button>}
          onConfirm={() => deleteMutation.mutate(schedule.id)}
          title="Delete Schedule"
          message="Are you sure you want to delete this schedule?"
        />
      </div>
    </div>
  );
}
```

**Step 3: Create schedule form page (routes/schedules/create.tsx)**

```tsx
import { createFileRoute, useNavigate } from '@tanstack/react-router';
import { ScheduleForm } from '@/components/organisms/schedule-form';
import { useCreateSchedule } from '@/hooks/schedules';

export const Route = createFileRoute('/schedules/create')({
  component: CreateSchedule,
});

function CreateSchedule() {
  const navigate = useNavigate();
  const createMutation = useCreateSchedule();

  const handleSubmit = async (data: unknown) => {
    const result = await createMutation.mutateAsync(data);
    navigate({ to: `/schedules/${result.id}` });
  };

  return (
    <div>
      <h1>Create Schedule</h1>
      <ScheduleForm onSubmit={handleSubmit} />
    </div>
  );
}
```

**Step 4: Create schedule form organism (components/organisms/schedule-form.tsx)**

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useReportTypes } from '@/hooks/report-types';
import { CadenceSelector } from '@/components/molecules/cadence-selector';
import { TimezoneSelector } from '@/components/molecules/timezone-selector';
import { RecipientInput } from '@/components/molecules/recipient-input';

const scheduleCreateSchema = z.object({
  reportTypeId: z.string().min(1),
  cadenceType: z.enum(['daily', 'weekly', 'monthly']),
  cadenceDay: z.number().nullable(),
  scheduleTime: z.string().regex(/^([01]\d|2[0-3]):[0-5]\d$/),
  timezone: z.string().min(1),
  recipients: z.array(z.object({ email: z.string().email() })).min(1).max(50),
});

type ScheduleFormData = z.infer<typeof scheduleCreateSchema>;

interface ScheduleFormProps {
  onSubmit: (data: ScheduleFormData) => void;
  initialData?: Partial<ScheduleFormData>;
}

export function ScheduleForm({ onSubmit, initialData }: ScheduleFormProps) {
  const { data: reportTypes } = useReportTypes();
  const { register, handleSubmit, watch, setValue, formState: { errors } } = useForm<ScheduleFormData>({
    resolver: zodResolver(scheduleCreateSchema),
    defaultValues: initialData,
  });

  const cadenceType = watch('cadenceType');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label>Report Type</label>
        <select {...register('reportTypeId')} required>
          <option value="">Select a report type</option>
          {reportTypes?.map(type => (
            <option key={type.id} value={type.id}>{type.name}</option>
          ))}
        </select>
        {errors.reportTypeId && <span>{errors.reportTypeId.message}</span>}
      </div>

      <CadenceSelector
        cadenceType={cadenceType}
        onCadenceChange={(type) => setValue('cadenceType', type)}
        onDayChange={(day) => setValue('cadenceDay', day)}
      />

      <div>
        <label>Time</label>
        <input type="time" {...register('scheduleTime')} required />
        {errors.scheduleTime && <span>{errors.scheduleTime.message}</span>}
      </div>

      <TimezoneSelector
        value={watch('timezone')}
        onChange={(tz) => setValue('timezone', tz)}
      />

      <RecipientInput
        recipients={watch('recipients') || []}
        onChange={(recipients) => setValue('recipients', recipients)}
      />
      {errors.recipients && <span>{errors.recipients.message}</span>}

      <button type="submit">Create Schedule</button>
    </form>
  );
}
```

**Step 5: Create delivery history organism (components/organisms/delivery-history.tsx)**

```tsx
import type { RunSummaryResponse } from '@/schemas/schedule';
import { StatusBadge } from '@/components/atoms/status-badge';
import { RunDetail } from '@/components/molecules/run-detail';

interface DeliveryHistoryProps {
  scheduleId: number;
  recentRuns: RunSummaryResponse[];
}

export function DeliveryHistory({ scheduleId, recentRuns }: DeliveryHistoryProps) {
  if (!recentRuns.length) {
    return <div>No delivery history yet.</div>;
  }

  return (
    <div>
      {recentRuns.map(run => (
        <RunDetail key={run.id} run={run} />
      ))}
    </div>
  );
}
```

**Step 6: Create supporting components**

Implement atoms/molecules as needed:
- `status-badge.tsx`: Display status with color-coded badge
- `cadence-selector.tsx`: Cadence type + day input (show/hide based on type)
- `timezone-selector.tsx`: Dropdown with IANA timezones
- `recipient-input.tsx`: Dynamic list of email inputs (add/remove)
- `confirm-dialog.tsx`: Modal dialog for delete confirmation
- `run-detail.tsx`: Expandable run detail with per-recipient status

**Step 7: Write tests**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/ui && \
  pnpm vitest run --reporter=verbose src/routes/ src/components/ && \
  pnpm tsc --noEmit
```

Tests verify (from TD section 9, Task 7 exit condition):
- Schedule list renders loading, empty, and populated states
- Schedule form validates required fields
- Cadence selector shows/hides day input based on frequency
- Recipient input validates email format
- Delete confirmation dialog works
- TypeScript compiles without errors

**4. Verify:**

```bash
cd /home/jary/redhat/git/claude-collective-example/packages/ui && \
  pnpm vitest run --reporter=verbose src/routes/ src/components/ && \
  pnpm tsc --noEmit
```

Expected: All tests pass, TypeScript compiles without errors.

#### Constraints

- Use TanStack Router file-based routing (`createFileRoute`)
- Use TanStack Query hooks for all data fetching
- Use `react-hook-form` with Zod validation for form handling
- Use Tailwind CSS for styling
- Prefer shadcn/ui components for accessibility (Button, Dialog, Select, etc.)
- All components must render loading, error, and empty states

#### Test Expectations

Component tests covering:
- Rendering of loading, error, empty, and populated states
- Form validation (required fields, email format, max recipients)
- User interactions (pause, resume, delete)
- TypeScript compilation

---

## Dependency Summary

| Task | Depends On | Blocks |
|------|-----------|--------|
| T-1.1 (Database) | None | T-2.1, T-3.1 |
| T-2.1 (API Core) | T-1.1 | T-3.1, T-4.1 |
| T-3.1 (Schemas) | T-2.1 | T-3.3 |
| T-3.2 (State Machine + Next Run) | T-2.1, T-3.1 | T-3.3, T-3.4 |
| T-3.3 (Schedule CRUD) | T-3.1, T-3.2 | T-4.1 |
| T-3.4 (Pause/Resume) | T-3.2, T-3.3 | None |
| T-4.1 (Run History) | T-3.3 | None |
| T-4.2 (Report Types) | T-2.1 | None |
| T-4.3 (Celery Tasks) | T-3.2 | None |
| T-5.1 (FE Schemas/Hooks) | None | T-5.2 |
| T-5.2 (FE Pages) | T-5.1 | None |

**Critical Path:** T-1.1 → T-2.1 → T-3.1 → T-3.2 → T-3.3 → T-4.1 (backend sequence)

**Parallelization:** T-5.1 and T-5.2 (frontend) can run in parallel with T-3.x and T-4.x (backend) since they share contracts defined in the TD, not runtime dependencies.

---

## Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Alembic migration conflicts** | High — multiple developers modifying schema | T-1.1 must complete and merge before other tasks begin |
| **CRLF injection missed** | High — security vulnerability | Code Reviewer and Security Engineer must verify email validation in T-3.1 and T-4.3 |
| **Next run computation bugs** | Medium — schedules run at wrong times | T-3.2 tests must cover all DST and monthly edge cases before T-3.3 proceeds |
| **Frontend diverges from API** | Medium — runtime errors | T-5.1 Zod schemas must be validated against running backend after T-3.3 completes |
| **Celery task crashes** | High — no retries in Phase 1 | T-4.3 tests must cover all error paths; monitoring required in production |

---

## Verification Plan

After all tasks complete, run the full integration test suite:

```bash
# Backend
cd /home/jary/redhat/git/claude-collective-example/packages/api
uv run pytest -v

# Frontend
cd /home/jary/redhat/git/claude-collective-example/packages/ui
pnpm vitest run && pnpm tsc --noEmit

# E2E (if implemented)
pnpm test:e2e
```

Manual verification:
1. Create a schedule via UI
2. Verify schedule appears in list
3. Verify next_run_at is computed correctly
4. Wait for tick task to create a run (or manually trigger)
5. Verify run transitions through states: pending → generating → generated → delivering → delivered
6. Verify email sent to recipients (check Mailpit inbox)
7. Verify delivery history shows per-recipient status
8. Test pause/resume/delete operations

---

## Export: Agent Task Plan

This work breakdown is ready for conversion to TaskCreate calls with blockedBy dependencies. The critical path is:

```
Phase 1: Database
1. [@database-engineer] T-1.1: Database schema, models, migration → blockedBy: none

Phase 2: API Core
2. [@backend-developer] T-2.1: FastAPI app, config, auth, errors → blockedBy: [1]

Phase 3: Schedule API
3. [@backend-developer] T-3.1: Pydantic schemas → blockedBy: [2]
4. [@backend-developer] T-3.2: State machine + next_run service → blockedBy: [2, 3]
5. [@backend-developer] T-3.3: Schedule CRUD endpoints → blockedBy: [3, 4]
6. [@backend-developer] T-3.4: Pause/resume endpoints → blockedBy: [4, 5]

Phase 4: Additional APIs (parallel)
7. [@backend-developer] T-4.1: Delivery history endpoint → blockedBy: [5]
8. [@backend-developer] T-4.2: Report types endpoint → blockedBy: [2]
9. [@backend-developer] T-4.3: Celery tasks + protocols → blockedBy: [4]

Phase 5: Frontend (parallel with Phase 3-4)
10. [@frontend-developer] T-5.1: Schemas, services, hooks → blockedBy: none
11. [@frontend-developer] T-5.2: Pages and components → blockedBy: [10]

Phase 6: Review
12. [@code-reviewer] Review all implementation → blockedBy: [6, 7, 8, 9, 11]
13. [@security-engineer] Security audit → blockedBy: [6, 7, 8, 9, 11]
```

---

## Summary

- **5 Work Units** covering database, API core, schedule API, additional APIs, and frontend
- **11 implementation tasks** sized S/M/L (3–5 files, machine-verifiable exit conditions)
- **Sequential backend path:** Database → API Core → Schedule API → Additional APIs
- **Parallel frontend path:** Schemas/Hooks → Pages/Components
- **Critical path duration:** ~3–4 weeks with parallel execution
- **All tasks include:** agent assignments, file lists, verification commands, inlined context from TD
- **Context propagation verified:** Each task is self-contained and executable without reading upstream documents

This work breakdown is ready for implementation. Each task prompt is written as a direct instruction set for the implementing agent.
