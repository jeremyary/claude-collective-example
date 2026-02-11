# Technical Design: Report Scheduling -- Phase 1

**Author:** Tech Lead
**Date:** 2026-02-10
**Input:** Product plan (validated), Architecture (validated), Requirements (validated), ADR-0001 through ADR-0004, Architecture reviews (Security Engineer, API Designer), Requirements review (Architect)
**Phase:** 1 -- Core Scheduling

---

## 1. Scope

Phase 1 delivers the 8 P0 features described in user stories US-001 through US-010:

| Feature | User Story | Summary |
|---------|-----------|---------|
| Report type selection | US-001 | Choose from available report types when creating a schedule |
| Cadence configuration | US-002 | Set daily/weekly/monthly frequency with time and day |
| Recipient management | US-003 | Add/remove email recipients (max 50 per schedule) |
| Email delivery | US-004 | Automatic report generation and email delivery via Celery pipeline |
| Schedule management | US-005 | View, edit schedules from a central list |
| Pause/resume | US-006 | Pause/resume schedules without deleting configuration |
| Delete schedules | US-007 | Soft-delete schedules with history retention |
| Delivery status visibility | US-008 | View per-run and per-recipient delivery outcomes |
| Timezone support | US-009 | Schedules run in owner's local timezone with DST handling |
| Authorization | US-010 | Owner + admin access control with query-level filtering |

**Not in scope:** Automatic retries, custom cadence expressions, in-app notifications, admin dashboard, bulk operations, schedule templates, report preview. These are Phase 2/3.

---

## 2. Data Model

### 2.1 Enums

```python
# packages/db/src/db/enums.py

import enum


class ScheduleStatus(str, enum.Enum):
    """Lifecycle states for a schedule configuration."""
    ACTIVE = "active"
    PAUSED = "paused"
    DELETED = "deleted"


class CadenceType(str, enum.Enum):
    """Supported delivery frequencies."""
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"


class RunStatus(str, enum.Enum):
    """Lifecycle states for a single schedule execution."""
    PENDING = "pending"
    GENERATING = "generating"
    GENERATED = "generated"
    DELIVERING = "delivering"
    DELIVERED = "delivered"
    PARTIALLY_DELIVERED = "partially_delivered"
    DELIVERY_FAILED = "delivery_failed"
    GENERATION_FAILED = "generation_failed"


class RecipientDeliveryStatus(str, enum.Enum):
    """Per-recipient delivery outcome."""
    PENDING = "pending"
    SENT = "sent"
    FAILED = "failed"
```

### 2.2 Schedule Model

```python
# packages/db/src/db/models.py

from datetime import datetime, time

from sqlalchemy import (
    DateTime,
    Enum,
    Index,
    Integer,
    SmallInteger,
    String,
    Time,
    func,
)
from sqlalchemy.orm import Mapped, mapped_column, relationship

from .database import Base
from .enums import CadenceType, ScheduleStatus


class Schedule(Base):
    """A recurring report delivery configuration owned by a user."""

    __tablename__ = "schedules"
    __table_args__ = (
        Index("ix_schedules_tick", "status", "next_run_at"),
        Index("ix_schedules_owner", "owner_id", "status"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    owner_id: Mapped[int] = mapped_column(Integer, nullable=False, index=True)
    report_type_id: Mapped[str] = mapped_column(String(100), nullable=False)

    cadence_type: Mapped[CadenceType] = mapped_column(
        Enum(CadenceType, native_enum=False, length=20), nullable=False
    )
    cadence_day: Mapped[int | None] = mapped_column(
        SmallInteger, nullable=True,
        comment="Day of week (0=Mon..6=Sun) for weekly; day of month (1-31) for monthly; null for daily",
    )
    schedule_hour: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, comment="Hour (0-23) in owner's local timezone"
    )
    schedule_minute: Mapped[int] = mapped_column(
        SmallInteger, nullable=False, comment="Minute (0-59) in owner's local timezone"
    )
    timezone: Mapped[str] = mapped_column(
        String(64), nullable=False, comment="IANA timezone identifier"
    )

    status: Mapped[ScheduleStatus] = mapped_column(
        Enum(ScheduleStatus, native_enum=False, length=20),
        nullable=False,
        server_default="active",
    )
    next_run_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, index=True
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )

    # Relationships
    recipients: Mapped[list["ScheduleRecipient"]] = relationship(
        back_populates="schedule", cascade="all, delete-orphan"
    )
    runs: Mapped[list["ScheduleRun"]] = relationship(
        back_populates="schedule", cascade="all, delete-orphan"
    )
```

### 2.3 ScheduleRecipient Model

```python
class ScheduleRecipient(Base):
    """An email recipient associated with a schedule."""

    __tablename__ = "schedule_recipients"
    __table_args__ = (
        Index("ix_recipients_schedule_active", "schedule_id", "is_active"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    schedule_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("schedules.id", ondelete="CASCADE"),
        nullable=False,
    )
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )

    # Relationships
    schedule: Mapped["Schedule"] = relationship(back_populates="recipients")
```

### 2.4 ScheduleRun Model

```python
class ScheduleRun(Base):
    """A record of a single schedule execution attempt."""

    __tablename__ = "schedule_runs"
    __table_args__ = (
        Index("ix_runs_schedule_created", "schedule_id", "created_at"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    schedule_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("schedules.id", ondelete="CASCADE"),
        nullable=False,
    )
    status: Mapped[RunStatus] = mapped_column(
        Enum(RunStatus, native_enum=False, length=30),
        nullable=False,
        server_default="pending",
    )
    started_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True,
        comment="Timestamp when the first processing step (report generation) began. "
        "Set once by execute_schedule when transitioning to GENERATING. "
        "Not updated by subsequent steps (deliver). Null until processing starts.",
    )
    completed_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    error_message: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )

    # Relationships
    schedule: Mapped["Schedule"] = relationship(back_populates="runs")
    run_recipients: Mapped[list["RunRecipient"]] = relationship(
        back_populates="run", cascade="all, delete-orphan"
    )
```

### 2.5 RunRecipient Model

```python
class RunRecipient(Base):
    """Per-recipient delivery outcome for a specific run.

    Stores recipient_email as a snapshot -- not a FK to schedule_recipients.
    This preserves delivery history even after recipients are soft-deleted.
    """

    __tablename__ = "run_recipients"
    __table_args__ = (
        Index("ix_run_recipients_run", "run_id"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    run_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("schedule_runs.id", ondelete="CASCADE"),
        nullable=False,
    )
    recipient_email: Mapped[str] = mapped_column(
        String(255), nullable=False,
        comment="Snapshot of email at delivery time, not a FK to recipients",
    )
    status: Mapped[RecipientDeliveryStatus] = mapped_column(
        Enum(RecipientDeliveryStatus, native_enum=False, length=20),
        nullable=False,
        server_default="pending",
    )
    error_message: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    delivered_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    # Relationships
    run: Mapped["ScheduleRun"] = relationship(back_populates="run_recipients")
```

### 2.6 Database Package Public Exports

```python
# packages/db/src/__init__.py

from .db.database import Base, engine, get_session, get_sync_session
from .db.enums import (
    CadenceType,
    RecipientDeliveryStatus,
    RunStatus,
    ScheduleStatus,
)
from .db.models import RunRecipient, Schedule, ScheduleRecipient, ScheduleRun

__all__ = [
    "Base",
    "engine",
    "get_session",
    "get_sync_session",
    "CadenceType",
    "RecipientDeliveryStatus",
    "RunStatus",
    "Schedule",
    "ScheduleRecipient",
    "ScheduleRun",
    "ScheduleStatus",
    "RunRecipient",
]
```

### 2.7 Session Factories

```python
# packages/db/src/db/database.py

import os
from collections.abc import AsyncGenerator, Generator

from sqlalchemy import create_engine
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker


class Base(DeclarativeBase):
    pass


# The DB package reads its own configuration from environment variables directly.
# It does NOT import from packages/api (src.core.config) -- that would create a
# circular dependency since the architecture requires api -> db, not db -> api.
# The API package's Settings class should set these env vars or the deployment
# environment provides them.

_database_url = os.environ.get(
    "DATABASE_URL", "postgresql+asyncpg://localhost:5432/report_scheduler"
)
_sync_database_url = os.environ.get(
    "SYNC_DATABASE_URL", "postgresql+psycopg2://localhost:5432/report_scheduler"
)

# Async engine and session (for FastAPI)
async_engine = create_async_engine(_database_url, echo=False, pool_size=10)
AsyncSessionFactory = async_sessionmaker(async_engine, expire_on_commit=False)

# Sync engine and session (for Celery workers)
sync_engine = create_engine(
    _sync_database_url, echo=False, pool_size=5
)
SyncSessionFactory = sessionmaker(sync_engine, expire_on_commit=False)


async def get_session() -> AsyncGenerator[AsyncSession, None]:
    """Async session dependency for FastAPI."""
    async with AsyncSessionFactory() as session:
        yield session


def get_sync_session() -> Generator[Session, None, None]:
    """Sync session factory for Celery workers."""
    with SyncSessionFactory() as session:
        yield session
```

### 2.8 State Machine Transition Functions

A shared utility that both the API (for Schedule status) and Celery tasks (for RunStatus) call. Invalid transitions raise an error -- they do not silently succeed (per Architect requirements review observation 1, Security Engineer WARNING-2).

```python
# packages/api/src/services/state_machine.py

from db import RunStatus, ScheduleStatus

VALID_SCHEDULE_TRANSITIONS: dict[ScheduleStatus, set[ScheduleStatus]] = {
    ScheduleStatus.ACTIVE: {ScheduleStatus.PAUSED, ScheduleStatus.DELETED},
    ScheduleStatus.PAUSED: {ScheduleStatus.ACTIVE, ScheduleStatus.DELETED},
    ScheduleStatus.DELETED: set(),  # terminal
}

VALID_RUN_TRANSITIONS: dict[RunStatus, set[RunStatus]] = {
    RunStatus.PENDING: {RunStatus.GENERATING},
    RunStatus.GENERATING: {RunStatus.GENERATED, RunStatus.GENERATION_FAILED},
    RunStatus.GENERATED: {RunStatus.DELIVERING},
    RunStatus.DELIVERING: {
        RunStatus.DELIVERED,
        RunStatus.PARTIALLY_DELIVERED,
        RunStatus.DELIVERY_FAILED,
    },
    # Terminal states
    RunStatus.DELIVERED: set(),
    RunStatus.PARTIALLY_DELIVERED: set(),
    RunStatus.DELIVERY_FAILED: set(),
    RunStatus.GENERATION_FAILED: set(),
}


class InvalidStateTransitionError(Exception):
    """Raised when an invalid state transition is attempted."""

    def __init__(self, entity: str, current: str, target: str) -> None:
        self.entity = entity
        self.current = current
        self.target = target
        super().__init__(
            f"Invalid {entity} transition: {current} -> {target}"
        )


def validate_schedule_transition(
    current: ScheduleStatus, target: ScheduleStatus
) -> None:
    """Validate and raise if the schedule state transition is invalid."""
    if target not in VALID_SCHEDULE_TRANSITIONS.get(current, set()):
        raise InvalidStateTransitionError("schedule", current.value, target.value)


def validate_run_transition(current: RunStatus, target: RunStatus) -> None:
    """Validate and raise if the run state transition is invalid."""
    if target not in VALID_RUN_TRANSITIONS.get(current, set()):
        raise InvalidStateTransitionError("run", current.value, target.value)
```

---

## 3. API Contracts

All endpoints are prefixed with `/v1`. JSON field names use camelCase. All timestamps are ISO 8601 with UTC timezone suffix (`Z`). Error responses use RFC 7807 format. Authentication is required on all endpoints (Bearer token in `Authorization` header).

### 3.1 POST /v1/schedules -- Create Schedule

Creates a new schedule with inline recipients. The authenticated user becomes the owner.

**Rate limited:** 10 requests/minute/user (applies to creates and updates per requirements section 4).

**Request:**

```json
{
  "reportTypeId": "string (required, max 100 chars)",
  "cadenceType": "daily | weekly | monthly (required)",
  "cadenceDay": "integer | null (required for weekly: 0=Mon..6=Sun; required for monthly: 1-31; null for daily)",
  "scheduleTime": "string (required, format HH:MM, 24-hour)",
  "timezone": "string (required, valid IANA timezone identifier)",
  "recipients": [
    "string (email, RFC 5322 format)"
  ]
}
```

**Validation rules:**
- `reportTypeId`: Must exist in the report types registry. Max 100 characters.
- `cadenceDay`: Required and validated by cadence type. Weekly: 0-6. Monthly: 1-31. Daily: must be null.
- `scheduleTime`: Format `HH:MM` where HH is 00-23 and MM is 00-59.
- `timezone`: Must be in `zoneinfo.available_timezones()`.
- `recipients`: 1-50 entries. Each must be valid RFC 5322 email. Duplicates are deduplicated. No CRLF characters permitted (email header injection prevention per Security Engineer WARNING-1).
- Active schedule count for user must be below 25.

**Response (201 Created):**

Headers: `Location: /v1/schedules/{id}`

```json
{
  "data": {
    "id": 123,
    "reportTypeId": "sales-summary",
    "cadenceType": "weekly",
    "cadenceDay": 0,
    "scheduleTime": "09:00",
    "timezone": "America/New_York",
    "status": "active",
    "nextRunAt": "2026-02-16T14:00:00Z",
    "recipients": [
      {
        "id": 1,
        "email": "alice@example.com",
        "isActive": true
      },
      {
        "id": 2,
        "email": "bob@example.com",
        "isActive": true
      }
    ],
    "createdAt": "2026-02-10T18:30:00Z",
    "updatedAt": "2026-02-10T18:30:00Z"
  }
}
```

**Error responses:**

```json
// 422 -- Validation error
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Invalid timezone. Expected a valid IANA timezone identifier.",
  "instance": "/v1/schedules",
  "errors": [
    {
      "field": "timezone",
      "message": "Invalid timezone. Expected a valid IANA timezone identifier."
    }
  ]
}

// 422 -- Limit exceeded
{
  "type": "https://api.example.com/errors/limit-exceeded",
  "title": "Limit Exceeded",
  "status": 422,
  "detail": "Maximum 25 active schedules per user. Pause or delete a schedule to create a new one.",
  "instance": "/v1/schedules"
}

// 429 -- Rate limit
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Too many requests. Try again in 42 seconds.",
  "instance": "/v1/schedules"
}
// Headers: X-RateLimit-Limit: 10, X-RateLimit-Remaining: 0, X-RateLimit-Reset: 1707589842
```

### 3.2 GET /v1/schedules -- List User's Schedules

Returns schedules owned by the authenticated user. Excludes soft-deleted schedules. Uses cursor-based pagination (per API conventions for dynamic datasets; the user's schedule list may change between pages due to background status updates, and cursor-based handles this correctly).

**Query parameters:**
- `cursor` (string, optional): Pagination cursor (opaque, base64-encoded `id` value).
- `limit` (integer, optional): Items per page. Default 20, max 100.
- `status` (string, optional): Filter by status (`active`, `paused`). Omit to return all non-deleted.

**Response (200):**

```json
{
  "data": [
    {
      "id": 123,
      "reportTypeId": "sales-summary",
      "cadenceType": "weekly",
      "cadenceDay": 0,
      "scheduleTime": "09:00",
      "timezone": "America/New_York",
      "status": "active",
      "nextRunAt": "2026-02-16T14:00:00Z",
      "lastRunStatus": "delivered",
      "lastRunAt": "2026-02-09T14:00:00Z",
      "createdAt": "2026-02-01T10:00:00Z",
      "updatedAt": "2026-02-09T14:01:00Z"
    }
  ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIzfQ==",
    "hasMore": true
  }
}
```

Note: `lastRunStatus` and `lastRunAt` are computed from the most recent `ScheduleRun` row. If no runs exist, both are `null`.

### 3.3 GET /v1/schedules/{id} -- Get Schedule Detail

Returns a single schedule with its recipients and the 5 most recent runs. The query is scoped to `owner_id = current_user.id` (returns 404 if not found or not owned, per Security Engineer SUGGESTION-1). Admin users bypass the owner filter.

**Response (200):**

```json
{
  "data": {
    "id": 123,
    "reportTypeId": "sales-summary",
    "cadenceType": "weekly",
    "cadenceDay": 0,
    "scheduleTime": "09:00",
    "timezone": "America/New_York",
    "status": "active",
    "nextRunAt": "2026-02-16T14:00:00Z",
    "recipients": [
      {
        "id": 1,
        "email": "alice@example.com",
        "isActive": true
      }
    ],
    "recentRuns": [
      {
        "id": 456,
        "status": "delivered",
        "startedAt": "2026-02-09T14:00:05Z",
        "completedAt": "2026-02-09T14:00:32Z",
        "errorMessage": null,
        "createdAt": "2026-02-09T14:00:00Z"
      }
    ],
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-02-09T14:01:00Z"
  }
}
```

**Error responses:**

```json
// 404 -- Schedule not found or not owned by current user
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Schedule not found.",
  "instance": "/v1/schedules/999"
}
```

### 3.4 PATCH /v1/schedules/{id} -- Update Schedule

Partial update. Only provided fields are changed. Recipients are managed by replacement: if `recipients` is present in the body, the entire list is replaced (active recipients are soft-deleted and new ones created). State transitions (pause/resume/delete) use dedicated endpoints, not PATCH.

**Implementation note (PATCH semantics):** The route handler must use `ScheduleUpdate.model_fields_set` to determine which fields the client explicitly included in the request. This is required to distinguish "field omitted" (keep current value) from "field explicitly set to null" (e.g., `cadenceDay: null` when switching to daily cadence). Only fields in `model_fields_set` are applied to the database model.

**Rate limited:** 10 requests/minute/user.

**Request (all fields optional):**

```json
{
  "reportTypeId": "string",
  "cadenceType": "daily | weekly | monthly",
  "cadenceDay": "integer | null",
  "scheduleTime": "string (HH:MM)",
  "timezone": "string (IANA)",
  "recipients": ["string (email)"]
}
```

**Validation:** Same rules as creation. If cadence or timezone changes, `next_run_at` is recomputed. If the schedule is paused, changes are saved but the schedule remains paused (editing does not implicitly resume). If `recipients` is provided, at least 1 is required.

**Response (200):** Same shape as the creation response (full schedule with recipients).

**Error responses:** Same as creation (422, 429), plus 404 for not found/not owned.

### 3.5 DELETE /v1/schedules/{id} -- Soft Delete Schedule

Sets `status = deleted`. The schedule is excluded from all list queries and the tick task. Delivery history is retained.

**Response (204 No Content)**

**Error responses:**

```json
// 409 -- Invalid state transition (already deleted)
{
  "type": "https://api.example.com/errors/conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "Cannot modify a deleted schedule.",
  "instance": "/v1/schedules/123"
}
```

### 3.6 POST /v1/schedules/{id}/pause -- Pause Schedule

Transitions schedule from `active` to `paused`.

**Request:** Empty body.

**Response (200):**

```json
{
  "data": {
    "id": 123,
    "status": "paused",
    "nextRunAt": null
  }
}
```

When paused, `nextRunAt` is set to `null` in the response (the stored value is retained in the database for informational purposes, but the schedule is excluded from the tick query by status filter). On resume, `next_run_at` is recomputed.

**Error responses:**

```json
// 409 -- Not in a state that can be paused
{
  "type": "https://api.example.com/errors/conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "Schedule is not active. Current status: paused.",
  "instance": "/v1/schedules/123/pause"
}
```

### 3.7 POST /v1/schedules/{id}/resume -- Resume Schedule

Transitions schedule from `paused` to `active`. Recomputes `next_run_at` from current time.

**Request:** Empty body.

**Response (200):**

```json
{
  "data": {
    "id": 123,
    "status": "active",
    "nextRunAt": "2026-02-17T14:00:00Z"
  }
}
```

**Error responses:** 409 if not paused. Same pattern as pause.

### 3.8 GET /v1/schedules/{id}/runs -- Delivery History

Cursor-based pagination. Ordered by `created_at DESC`. Filterable by status.

**Query parameters:**
- `cursor` (string, optional): Pagination cursor.
- `limit` (integer, optional): Default 20, max 100.
- `status` (string, optional): Filter by run status (e.g., `delivered`, `generation_failed`).

**Response (200):**

```json
{
  "data": [
    {
      "id": 456,
      "status": "partially_delivered",
      "startedAt": "2026-02-09T14:00:05Z",
      "completedAt": "2026-02-09T14:00:32Z",
      "errorMessage": null,
      "createdAt": "2026-02-09T14:00:00Z",
      "recipients": [
        {
          "email": "alice@example.com",
          "status": "sent",
          "deliveredAt": "2026-02-09T14:00:20Z",
          "errorMessage": null
        },
        {
          "email": "bob@example.com",
          "status": "failed",
          "deliveredAt": null,
          "errorMessage": "Mailbox not found"
        }
      ]
    }
  ],
  "pagination": {
    "nextCursor": "eyJpZCI6NDU2fQ==",
    "hasMore": false
  }
}
```

### 3.9 GET /v1/report-types -- List Available Report Types

Returns a simple unpaginated list. Report types are few (assumed < 20 per architecture). If the list grows beyond 20, pagination and search can be added in a future phase.

Phase 1 returns a hardcoded list from configuration (since the report generation system is a stub). The list structure is stable -- when the real system is integrated, it provides the data through the same shape.

**Response (200):**

```json
{
  "data": [
    {
      "id": "sales-summary",
      "name": "Sales Summary",
      "description": "Weekly summary of sales performance by region."
    },
    {
      "id": "ops-dashboard",
      "name": "Operations Dashboard",
      "description": "Daily operational metrics and alerts."
    }
  ]
}
```

---

## 4. Pydantic Schemas

All schemas use Pydantic v2 with `alias` for camelCase JSON serialization. The `model_config` includes `populate_by_name=True` so that both snake_case and camelCase work for deserialization.

### 4.1 Request Schemas

```python
# packages/api/src/schemas/schedules.py

import re
from datetime import datetime

from pydantic import BaseModel, ConfigDict, EmailStr, Field, field_validator

from db import CadenceType


# Shared constant for CRLF prevention (Security Engineer WARNING-1)
_CRLF_PATTERN = re.compile(r"[\r\n]")


class RecipientCreate(BaseModel):
    """A single email recipient."""
    email: EmailStr

    @field_validator("email")
    @classmethod
    def reject_crlf_in_email(cls, v: str) -> str:
        """Prevent email header injection via CRLF in email addresses."""
        if _CRLF_PATTERN.search(v):
            msg = "Email address must not contain control characters."
            raise ValueError(msg)
        return v.lower().strip()


class ScheduleCreate(BaseModel):
    """Request schema for POST /v1/schedules."""

    model_config = ConfigDict(populate_by_name=True)

    report_type_id: str = Field(
        ..., alias="reportTypeId", min_length=1, max_length=100
    )
    cadence_type: CadenceType = Field(..., alias="cadenceType")
    cadence_day: int | None = Field(None, alias="cadenceDay")
    schedule_time: str = Field(
        ..., alias="scheduleTime", pattern=r"^([01]\d|2[0-3]):[0-5]\d$"
    )
    timezone: str = Field(..., min_length=1, max_length=64)
    recipients: list[RecipientCreate] = Field(..., min_length=1, max_length=50)

    @field_validator("cadence_day")
    @classmethod
    def validate_cadence_day(cls, v: int | None, info) -> int | None:
        """Validate cadence_day against cadence_type."""
        cadence_type = info.data.get("cadence_type")
        if cadence_type == CadenceType.DAILY:
            if v is not None:
                msg = "cadenceDay must be null for daily cadence."
                raise ValueError(msg)
        elif cadence_type == CadenceType.WEEKLY:
            if v is None or not (0 <= v <= 6):
                msg = "cadenceDay must be 0-6 (Mon-Sun) for weekly cadence."
                raise ValueError(msg)
        elif cadence_type == CadenceType.MONTHLY:
            if v is None or not (1 <= v <= 31):
                msg = "cadenceDay must be 1-31 for monthly cadence."
                raise ValueError(msg)
        return v

    @field_validator("timezone")
    @classmethod
    def validate_timezone(cls, v: str) -> str:
        """Validate against IANA timezone database."""
        from zoneinfo import available_timezones
        if v not in available_timezones():
            msg = "Invalid timezone. Expected a valid IANA timezone identifier."
            raise ValueError(msg)
        return v

    @field_validator("recipients")
    @classmethod
    def deduplicate_recipients(
        cls, v: list[RecipientCreate],
    ) -> list[RecipientCreate]:
        """Deduplicate recipients by normalized email."""
        seen: set[str] = set()
        unique: list[RecipientCreate] = []
        for r in v:
            normalized = r.email.lower().strip()
            if normalized not in seen:
                seen.add(normalized)
                unique.append(r)
        return unique


class ScheduleUpdate(BaseModel):
    """Request schema for PATCH /v1/schedules/{id}. All fields optional.

    PATCH partial-update semantics: To distinguish "field not provided" from
    "field explicitly set to null", the route handler MUST check
    ``update_data.model_fields_set`` after parsing. Only fields present in
    ``model_fields_set`` should be applied to the database model. Fields not
    in ``model_fields_set`` are absent from the request and must be left
    unchanged.

    This is critical for ``cadence_day``: when switching from weekly to daily,
    the client sends ``{"cadenceType": "daily", "cadenceDay": null}``. Without
    checking ``model_fields_set``, both "cadenceDay omitted" and "cadenceDay
    set to null" would appear identical (both are ``None``).

    Route handler pattern::

        update_data = ScheduleUpdate(**body)
        for field_name in update_data.model_fields_set:
            value = getattr(update_data, field_name)
            setattr(schedule, field_name, value)
    """

    model_config = ConfigDict(populate_by_name=True)

    report_type_id: str | None = Field(
        None, alias="reportTypeId", min_length=1, max_length=100
    )
    cadence_type: CadenceType | None = Field(None, alias="cadenceType")
    cadence_day: int | None = Field(None, alias="cadenceDay")
    schedule_time: str | None = Field(
        None, alias="scheduleTime", pattern=r"^([01]\d|2[0-3]):[0-5]\d$"
    )
    timezone: str | None = Field(None, min_length=1, max_length=64)
    recipients: list[RecipientCreate] | None = Field(None, min_length=1, max_length=50)

    @field_validator("timezone")
    @classmethod
    def validate_timezone(cls, v: str | None) -> str | None:
        if v is None:
            return v
        from zoneinfo import available_timezones
        if v not in available_timezones():
            msg = "Invalid timezone. Expected a valid IANA timezone identifier."
            raise ValueError(msg)
        return v
```

### 4.2 Response Schemas

```python
# packages/api/src/schemas/responses.py

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field

from db import CadenceType, RecipientDeliveryStatus, RunStatus, ScheduleStatus


class RecipientResponse(BaseModel):
    """Recipient in API responses."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    email: str
    is_active: bool = Field(alias="isActive")


class RunSummaryResponse(BaseModel):
    """Abbreviated run for schedule detail view."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    status: RunStatus
    started_at: datetime | None = Field(alias="startedAt")
    completed_at: datetime | None = Field(alias="completedAt")
    error_message: str | None = Field(alias="errorMessage")
    created_at: datetime = Field(alias="createdAt")


class ScheduleResponse(BaseModel):
    """Full schedule in creation and detail responses."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    report_type_id: str = Field(alias="reportTypeId")
    cadence_type: CadenceType = Field(alias="cadenceType")
    cadence_day: int | None = Field(alias="cadenceDay")
    schedule_time: str = Field(alias="scheduleTime")
    timezone: str
    status: ScheduleStatus
    next_run_at: datetime | None = Field(alias="nextRunAt")
    recipients: list[RecipientResponse]
    recent_runs: list[RunSummaryResponse] = Field(
        default_factory=list, alias="recentRuns"
    )
    created_at: datetime = Field(alias="createdAt")
    updated_at: datetime = Field(alias="updatedAt")


class ScheduleListItem(BaseModel):
    """Schedule in list responses (no recipients or runs)."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    report_type_id: str = Field(alias="reportTypeId")
    cadence_type: CadenceType = Field(alias="cadenceType")
    cadence_day: int | None = Field(alias="cadenceDay")
    schedule_time: str = Field(alias="scheduleTime")
    timezone: str
    status: ScheduleStatus
    next_run_at: datetime | None = Field(alias="nextRunAt")
    last_run_status: RunStatus | None = Field(None, alias="lastRunStatus")
    last_run_at: datetime | None = Field(None, alias="lastRunAt")
    created_at: datetime = Field(alias="createdAt")
    updated_at: datetime = Field(alias="updatedAt")


class StatusUpdateResponse(BaseModel):
    """Response for pause/resume operations."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    status: ScheduleStatus
    next_run_at: datetime | None = Field(alias="nextRunAt")


class RunRecipientResponse(BaseModel):
    """Per-recipient delivery result in run detail."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    email: str
    status: RecipientDeliveryStatus
    delivered_at: datetime | None = Field(alias="deliveredAt")
    error_message: str | None = Field(alias="errorMessage")


class RunDetailResponse(BaseModel):
    """Full run in delivery history responses."""
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)

    id: int
    status: RunStatus
    started_at: datetime | None = Field(alias="startedAt")
    completed_at: datetime | None = Field(alias="completedAt")
    error_message: str | None = Field(alias="errorMessage")
    created_at: datetime = Field(alias="createdAt")
    recipients: list[RunRecipientResponse]


class ReportTypeResponse(BaseModel):
    """A single report type."""
    id: str
    name: str
    description: str


# Pagination wrapper

class PaginationMeta(BaseModel):
    """Cursor-based pagination metadata."""
    model_config = ConfigDict(populate_by_name=True)

    next_cursor: str | None = Field(None, alias="nextCursor")
    has_more: bool = Field(alias="hasMore")


class PaginatedScheduleList(BaseModel):
    """Paginated list of schedules."""
    data: list[ScheduleListItem]
    pagination: PaginationMeta


class PaginatedRunList(BaseModel):
    """Paginated list of runs."""
    data: list[RunDetailResponse]
    pagination: PaginationMeta


class DataEnvelope[T](BaseModel):
    """Single resource response envelope."""
    data: T


class ListEnvelope[T](BaseModel):
    """List response envelope (non-paginated)."""
    data: list[T]
```

---

## 5. Task Queue Architecture

### 5.1 Celery Application Setup

```python
# packages/api/src/tasks/__init__.py

from celery import Celery
from celery.schedules import crontab

from src.core.config import get_settings

settings = get_settings()

celery_app = Celery(
    "report_scheduler",
    broker=settings.redis_url,
    backend=settings.redis_url,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_acks_late=True,
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
    beat_schedule={
        "tick-every-60s": {
            "task": "src.tasks.tick.tick_task",
            "schedule": 60.0,
        },
    },
)
```

### 5.2 Tick Task

```python
# packages/api/src/tasks/tick.py

import logging
from datetime import datetime, timedelta, timezone as dt_timezone

from sqlalchemy import select

from db import RunStatus, Schedule, ScheduleRun, ScheduleStatus, get_sync_session
from src.services.next_run import compute_next_run
from src.tasks import celery_app

logger = logging.getLogger(__name__)


@celery_app.task(name="src.tasks.tick.tick_task")
def tick_task() -> dict:
    """Periodic tick: find due schedules and dispatch execution tasks.

    Uses SELECT FOR UPDATE SKIP LOCKED to allow concurrent ticks safely.
    Each due schedule gets:
    1. A new ScheduleRun (status=pending)
    2. Updated next_run_at for the following occurrence
    3. An execute_schedule task dispatched

    Orphan recovery: Also re-dispatches runs stuck in PENDING state for more
    than 5 minutes. This handles the case where the worker crashes after
    committing the ScheduleRun + advancing next_run_at but before the
    execute_schedule.delay() call succeeds. The execute_schedule task is
    idempotent on entry (checks state is PENDING before proceeding), so
    re-dispatching a run that already has an in-flight task is safe -- the
    duplicate will see the run in GENERATING state and exit.
    """
    session_gen = get_sync_session()
    session = next(session_gen)
    dispatched = 0
    recovered = 0

    try:
        now = datetime.now(dt_timezone.utc)

        # --- Phase 1: Orphan recovery ---
        # Find runs stuck in PENDING for more than 5 minutes.
        # These are likely orphans from a prior tick that crashed between
        # commit and dispatch.
        orphan_threshold = now - timedelta(minutes=5)
        orphan_stmt = (
            select(ScheduleRun)
            .where(
                ScheduleRun.status == RunStatus.PENDING,
                ScheduleRun.created_at <= orphan_threshold,
            )
        )
        orphan_runs = session.execute(orphan_stmt).scalars().all()

        for orphan_run in orphan_runs:
            from src.tasks.execute import execute_schedule
            execute_schedule.delay(orphan_run.id)
            recovered += 1

        if recovered > 0:
            logger.warning(
                "Recovered orphaned pending runs",
                extra={
                    "recovered_count": recovered,
                    "service": "worker",
                },
            )

        # --- Phase 2: Normal tick processing ---
        # Find due schedules with row-level locking
        stmt = (
            select(Schedule)
            .where(
                Schedule.status == ScheduleStatus.ACTIVE,
                Schedule.next_run_at <= now,
            )
            .with_for_update(skip_locked=True)
        )
        due_schedules = session.execute(stmt).scalars().all()

        logger.info(
            "Tick fired",
            extra={
                "schedules_found": len(due_schedules),
                "service": "worker",
            },
        )

        for schedule in due_schedules:
            # Create a run record
            run = ScheduleRun(
                schedule_id=schedule.id,
                status=RunStatus.PENDING,
            )
            session.add(run)
            session.flush()  # get run.id

            # Compute next occurrence
            next_run = compute_next_run(
                cadence_type=schedule.cadence_type,
                cadence_day=schedule.cadence_day,
                hour=schedule.schedule_hour,
                minute=schedule.schedule_minute,
                timezone_str=schedule.timezone,
                after=now,
            )
            schedule.next_run_at = next_run

            session.commit()

            # Dispatch execution task. If this fails (e.g., worker crash),
            # the orphan recovery phase in the next tick will re-dispatch.
            from src.tasks.execute import execute_schedule
            execute_schedule.delay(run.id)
            dispatched += 1

        return {"dispatched": dispatched, "recovered": recovered}

    except Exception:
        session.rollback()
        logger.exception(
            "Tick task failed",
            extra={"service": "worker"},
        )
        raise
    finally:
        try:
            next(session_gen)
        except StopIteration:
            pass
```

### 5.3 Execute Task

```python
# packages/api/src/tasks/execute.py

import logging
from datetime import datetime, timezone as dt_timezone

from sqlalchemy import select

from db import RunStatus, ScheduleRun, get_sync_session
from src.services.report_generator import get_report_generator
from src.services.state_machine import validate_run_transition
from src.tasks import celery_app

logger = logging.getLogger(__name__)


@celery_app.task(
    name="src.tasks.execute.execute_schedule",
    max_retries=0,
)
def execute_schedule(run_id: int) -> dict:
    """Generate a report for a schedule run.

    Input validation at task entry (Security Engineer WARNING-2):
    - Validates run_id exists
    - Validates run is in PENDING state
    """
    # Input validation at task entry
    if not isinstance(run_id, int) or run_id <= 0:
        logger.error(
            "Invalid run_id received by execute_schedule",
            extra={"run_id": run_id, "service": "worker"},
        )
        return {"error": "invalid_run_id"}

    session_gen = get_sync_session()
    session = next(session_gen)

    try:
        run = session.execute(
            select(ScheduleRun).where(ScheduleRun.id == run_id)
        ).scalar_one_or_none()

        if run is None:
            logger.error(
                "ScheduleRun not found",
                extra={"run_id": run_id, "service": "worker"},
            )
            return {"error": "run_not_found"}

        if run.status != RunStatus.PENDING:
            logger.warning(
                "ScheduleRun not in PENDING state, skipping",
                extra={
                    "run_id": run_id,
                    "current_status": run.status.value,
                    "service": "worker",
                },
            )
            return {"error": "invalid_state", "current": run.status.value}

        # Transition to GENERATING
        validate_run_transition(run.status, RunStatus.GENERATING)
        run.status = RunStatus.GENERATING
        run.started_at = datetime.now(dt_timezone.utc)
        session.commit()

        # Call report generator protocol
        generator = get_report_generator()
        schedule = run.schedule

        try:
            report_output = generator.generate(
                report_type_id=schedule.report_type_id,
                schedule_id=schedule.id,
            )
        except Exception as exc:
            # Generation failed -- terminal state
            validate_run_transition(run.status, RunStatus.GENERATION_FAILED)
            run.status = RunStatus.GENERATION_FAILED
            run.error_message = str(exc)[:1000]
            run.completed_at = datetime.now(dt_timezone.utc)
            session.commit()

            logger.error(
                "Report generation failed",
                extra={
                    "run_id": run_id,
                    "schedule_id": schedule.id,
                    "error": str(exc),
                    "service": "worker",
                },
            )
            return {"status": "generation_failed", "error": str(exc)}

        # Generation succeeded
        validate_run_transition(run.status, RunStatus.GENERATED)
        run.status = RunStatus.GENERATED
        session.commit()

        # Dispatch delivery task
        from src.tasks.deliver import deliver_report
        deliver_report.delay(run_id, report_output.content, report_output.content_type)

        logger.info(
            "Report generated, dispatching delivery",
            extra={
                "run_id": run_id,
                "schedule_id": schedule.id,
                "service": "worker",
            },
        )
        return {"status": "generated", "run_id": run_id}

    except Exception:
        session.rollback()
        logger.exception(
            "Execute task failed unexpectedly",
            extra={"run_id": run_id, "service": "worker"},
        )
        raise
    finally:
        try:
            next(session_gen)
        except StopIteration:
            pass
```

### 5.4 Deliver Task

```python
# packages/api/src/tasks/deliver.py

import logging
import re
from datetime import datetime, timezone as dt_timezone

from sqlalchemy import select

from db import (
    RecipientDeliveryStatus,
    RunRecipient,
    RunStatus,
    ScheduleRecipient,
    ScheduleRun,
    get_sync_session,
)
from src.services.email_sender import get_email_sender
from src.services.state_machine import validate_run_transition
from src.tasks import celery_app

logger = logging.getLogger(__name__)

# CRLF pattern for email header injection prevention
_CRLF_PATTERN = re.compile(r"[\r\n]")

# Monitoring threshold per Security Engineer SUGGESTION-2
_EMAIL_VOLUME_WARNING_THRESHOLD = 500


@celery_app.task(
    name="src.tasks.deliver.deliver_report",
    max_retries=0,
)
def deliver_report(
    run_id: int, report_content: str, content_type: str
) -> dict:
    """Deliver a generated report to all active recipients.

    Input validation at task entry (Security Engineer WARNING-2):
    - Validates run_id exists and is in GENERATED state
    - Sanitizes all email addresses against CRLF injection
    """
    # Input validation
    if not isinstance(run_id, int) or run_id <= 0:
        logger.error(
            "Invalid run_id in deliver_report",
            extra={"run_id": run_id, "service": "worker"},
        )
        return {"error": "invalid_run_id"}

    session_gen = get_sync_session()
    session = next(session_gen)

    try:
        run = session.execute(
            select(ScheduleRun).where(ScheduleRun.id == run_id)
        ).scalar_one_or_none()

        if run is None:
            logger.error(
                "ScheduleRun not found for delivery",
                extra={"run_id": run_id, "service": "worker"},
            )
            return {"error": "run_not_found"}

        if run.status != RunStatus.GENERATED:
            logger.warning(
                "ScheduleRun not in GENERATED state for delivery",
                extra={
                    "run_id": run_id,
                    "current_status": run.status.value,
                    "service": "worker",
                },
            )
            return {"error": "invalid_state", "current": run.status.value}

        # Transition to DELIVERING
        # Note: started_at is NOT set here. It was already set by the execute
        # task when transitioning to GENERATING (the first processing step).
        # started_at represents "when processing began", not "when delivery began".
        validate_run_transition(run.status, RunStatus.DELIVERING)
        run.status = RunStatus.DELIVERING
        session.commit()

        # Get active recipients
        schedule = run.schedule
        active_recipients = session.execute(
            select(ScheduleRecipient).where(
                ScheduleRecipient.schedule_id == schedule.id,
                ScheduleRecipient.is_active.is_(True),
            )
        ).scalars().all()

        sender = get_email_sender()

        # Build subject line -- sanitize against CRLF injection
        subject = f"{schedule.report_type_id} - {schedule.cadence_type.value}"
        if _CRLF_PATTERN.search(subject):
            subject = _CRLF_PATTERN.sub("", subject)

        sent_count = 0
        failed_count = 0

        for recipient in active_recipients:
            # CRLF check on recipient email (defense in depth)
            email = recipient.email
            if _CRLF_PATTERN.search(email):
                logger.warning(
                    "Rejected recipient with CRLF in email",
                    extra={
                        "run_id": run_id,
                        "email_hash": hash(email),
                        "service": "worker",
                    },
                )
                run_recipient = RunRecipient(
                    run_id=run.id,
                    recipient_email=email[:255],
                    status=RecipientDeliveryStatus.FAILED,
                    error_message="Invalid email: contains control characters",
                )
                session.add(run_recipient)
                failed_count += 1
                continue

            try:
                sender.send(
                    to_email=email,
                    subject=subject,
                    html_body=f"<p>Your scheduled report is attached.</p>",
                    attachment_content=report_content,
                    attachment_content_type=content_type,
                    attachment_filename=f"report.{_extension_for(content_type)}",
                )
                run_recipient = RunRecipient(
                    run_id=run.id,
                    recipient_email=email,
                    status=RecipientDeliveryStatus.SENT,
                    delivered_at=datetime.now(dt_timezone.utc),
                )
                session.add(run_recipient)
                sent_count += 1

            except Exception as exc:
                run_recipient = RunRecipient(
                    run_id=run.id,
                    recipient_email=email,
                    status=RecipientDeliveryStatus.FAILED,
                    error_message=str(exc)[:1000],
                )
                session.add(run_recipient)
                failed_count += 1

                logger.warning(
                    "Email delivery failed for recipient",
                    extra={
                        "run_id": run_id,
                        "error": str(exc),
                        "service": "worker",
                    },
                )

        # Determine final run status
        if failed_count == 0:
            final_status = RunStatus.DELIVERED
        elif sent_count == 0:
            final_status = RunStatus.DELIVERY_FAILED
        else:
            final_status = RunStatus.PARTIALLY_DELIVERED

        validate_run_transition(run.status, final_status)
        run.status = final_status
        run.completed_at = datetime.now(dt_timezone.utc)
        session.commit()

        logger.info(
            "Delivery completed",
            extra={
                "run_id": run_id,
                "schedule_id": schedule.id,
                "final_status": final_status.value,
                "sent": sent_count,
                "failed": failed_count,
                "service": "worker",
            },
        )

        return {
            "status": final_status.value,
            "sent": sent_count,
            "failed": failed_count,
        }

    except Exception:
        session.rollback()
        logger.exception(
            "Deliver task failed unexpectedly",
            extra={"run_id": run_id, "service": "worker"},
        )
        raise
    finally:
        try:
            next(session_gen)
        except StopIteration:
            pass


def _extension_for(content_type: str) -> str:
    """Map content type to file extension."""
    mapping = {
        "application/pdf": "pdf",
        "text/html": "html",
        "text/csv": "csv",
    }
    return mapping.get(content_type, "bin")
```

### 5.5 Service Protocols

```python
# packages/api/src/services/report_generator.py

from dataclasses import dataclass
from typing import Protocol


@dataclass
class ReportOutput:
    """Output of a report generation operation."""
    content: str  # Base64-encoded bytes for JSON serialization via Celery
    content_type: str  # MIME type (e.g., "application/pdf")


class ReportGenerator(Protocol):
    """Protocol for report generation. MVP uses a stub."""

    def generate(self, report_type_id: str, schedule_id: int) -> ReportOutput:
        """Generate a report.

        Args:
            report_type_id: The type of report to generate.
            schedule_id: The schedule that triggered this generation.

        Returns:
            ReportOutput with content bytes and MIME type.

        Raises:
            Exception: If generation fails. The error message should be
                human-readable (it is stored in ScheduleRun.error_message).
        """
        ...


class StubReportGenerator:
    """Stub implementation for MVP. Returns a placeholder report."""

    def generate(self, report_type_id: str, schedule_id: int) -> ReportOutput:
        import base64
        placeholder = f"<html><body><h1>Report: {report_type_id}</h1><p>This is a stub report for schedule {schedule_id}.</p></body></html>"
        return ReportOutput(
            content=base64.b64encode(placeholder.encode()).decode(),
            content_type="text/html",
        )


def get_report_generator() -> ReportGenerator:
    """Factory: returns the configured generator implementation."""
    from src.core.config import get_settings
    settings = get_settings()
    if settings.report_generator == "stub":
        return StubReportGenerator()
    msg = f"Unknown report generator: {settings.report_generator}"
    raise ValueError(msg)
```

```python
# packages/api/src/services/email_sender.py

from typing import Protocol


class EmailSender(Protocol):
    """Protocol for email delivery. MVP uses SMTP via smtplib."""

    def send(
        self,
        to_email: str,
        subject: str,
        html_body: str,
        attachment_content: str | None = None,
        attachment_content_type: str | None = None,
        attachment_filename: str | None = None,
    ) -> None:
        """Send a single email.

        The protocol handles one recipient at a time. The caller fans out
        per-recipient so that one failure does not block others.

        Input contract (Security Engineer WARNING-1):
        - to_email MUST NOT contain CRLF characters
        - subject MUST NOT contain CRLF characters
        - The implementation MUST reject inputs with control characters

        Args:
            to_email: RFC 5322 validated email address.
            subject: Email subject line.
            html_body: HTML email body.
            attachment_content: Base64-encoded attachment bytes (optional).
            attachment_content_type: MIME type of attachment (optional).
            attachment_filename: Filename for attachment (optional).

        Raises:
            ValueError: If to_email or subject contains CRLF characters.
            Exception: If delivery fails (message stored in RunRecipient.error_message).
        """
        ...


class SmtpEmailSender:
    """SMTP implementation for MVP. Sends via configured SMTP server."""

    def __init__(self, host: str, port: int, username: str | None, password: str | None) -> None:
        self.host = host
        self.port = port
        self.username = username
        self.password = password

    def send(
        self,
        to_email: str,
        subject: str,
        html_body: str,
        attachment_content: str | None = None,
        attachment_content_type: str | None = None,
        attachment_filename: str | None = None,
    ) -> None:
        import base64
        import re
        import smtplib
        from email.mime.application import MIMEApplication
        from email.mime.multipart import MIMEMultipart
        from email.mime.text import MIMEText

        # CRLF injection prevention (Security Engineer WARNING-1)
        crlf = re.compile(r"[\r\n]")
        if crlf.search(to_email):
            msg = f"Rejected email address with control characters"
            raise ValueError(msg)
        if crlf.search(subject):
            msg = f"Rejected subject with control characters"
            raise ValueError(msg)

        message = MIMEMultipart()
        message["To"] = to_email
        message["Subject"] = subject
        message["From"] = "reports@example.com"  # Configured via settings

        # HTML body with unsubscribe footer
        footer = (
            "<hr><p style='font-size: 12px; color: #666;'>"
            "You are receiving this report because the schedule owner added you "
            "to a scheduled delivery. To stop receiving this report, contact the "
            "schedule owner or your account administrator.</p>"
        )
        message.attach(MIMEText(html_body + footer, "html"))

        # Attachment
        if attachment_content and attachment_filename:
            decoded = base64.b64decode(attachment_content)
            attachment = MIMEApplication(decoded, Name=attachment_filename)
            attachment["Content-Disposition"] = f'attachment; filename="{attachment_filename}"'
            message.attach(attachment)

        with smtplib.SMTP(self.host, self.port, timeout=30) as server:
            if self.username and self.password:
                server.starttls()
                server.login(self.username, self.password)
            server.sendmail("reports@example.com", [to_email], message.as_string())


def get_email_sender() -> EmailSender:
    """Factory: returns the configured sender implementation."""
    from src.core.config import get_settings
    settings = get_settings()
    if settings.email_sender == "smtp":
        return SmtpEmailSender(
            host=settings.smtp_host,
            port=settings.smtp_port,
            username=settings.smtp_username,
            password=settings.smtp_password,
        )
    msg = f"Unknown email sender: {settings.email_sender}"
    raise ValueError(msg)
```

### 5.6 Next Run Computation

A pure function with no database access (per Architect requirements review observation 2). Handles DST transitions per ADR-0004.

```python
# packages/api/src/services/next_run.py

import calendar
from datetime import datetime, timedelta, timezone as dt_timezone
from zoneinfo import ZoneInfo

from db import CadenceType


def compute_next_run(
    cadence_type: CadenceType,
    cadence_day: int | None,
    hour: int,
    minute: int,
    timezone_str: str,
    after: datetime,
) -> datetime:
    """Compute the next occurrence of a schedule after the given time.

    Returns a UTC datetime for storage in next_run_at.

    DST rules per ADR-0004:
    - Ambiguous times (fall-back): uses first occurrence (fold=0)
    - Nonexistent times (spring-forward): advances to next valid minute (3:00 AM)
    - Monthly day clamping: day 31 in February -> last day of month

    Args:
        cadence_type: daily, weekly, or monthly.
        cadence_day: Day of week (0=Mon) for weekly, day of month (1-31) for monthly.
        hour: Hour (0-23) in local timezone.
        minute: Minute (0-59) in local timezone.
        timezone_str: IANA timezone identifier.
        after: UTC datetime. The next run must be strictly after this time.

    Returns:
        UTC datetime of the next occurrence.
    """
    tz = ZoneInfo(timezone_str)

    # Convert 'after' to local time in the schedule's timezone
    local_after = after.astimezone(tz)

    if cadence_type == CadenceType.DAILY:
        candidate = local_after.replace(
            hour=hour, minute=minute, second=0, microsecond=0
        )
        if candidate <= local_after:
            candidate += timedelta(days=1)

    elif cadence_type == CadenceType.WEEKLY:
        assert cadence_day is not None
        # cadence_day: 0=Monday..6=Sunday
        days_ahead = cadence_day - local_after.weekday()
        if days_ahead < 0:
            days_ahead += 7
        candidate = local_after.replace(
            hour=hour, minute=minute, second=0, microsecond=0
        ) + timedelta(days=days_ahead)
        if candidate <= local_after:
            candidate += timedelta(weeks=1)

    elif cadence_type == CadenceType.MONTHLY:
        assert cadence_day is not None
        # Start with the current month
        year, month = local_after.year, local_after.month
        clamped_day = min(cadence_day, calendar.monthrange(year, month)[1])
        candidate = local_after.replace(
            day=clamped_day, hour=hour, minute=minute, second=0, microsecond=0
        )
        if candidate <= local_after:
            # Move to next month
            if month == 12:
                year += 1
                month = 1
            else:
                month += 1
            clamped_day = min(cadence_day, calendar.monthrange(year, month)[1])
            candidate = candidate.replace(year=year, month=month, day=clamped_day)
    else:
        msg = f"Unsupported cadence type: {cadence_type}"
        raise ValueError(msg)

    # Re-localize to handle DST: construct a naive datetime and localize it
    naive_candidate = candidate.replace(tzinfo=None)
    localized = naive_candidate.replace(tzinfo=tz)

    # Handle nonexistent times (spring forward): if the time was folded,
    # advance to the next valid time
    utc_candidate = localized.astimezone(dt_timezone.utc)

    # Round-trip check: if converting back changes the time, it was nonexistent
    round_trip = utc_candidate.astimezone(tz)
    if round_trip.hour != hour or round_trip.minute != minute:
        # Time was nonexistent (spring forward). Use the post-transition time.
        # The round_trip value IS the correct post-transition time.
        utc_candidate = round_trip.replace(
            minute=0, second=0, microsecond=0
        ).astimezone(dt_timezone.utc)

    return utc_candidate
```

### 5.7 Email Volume Monitoring

Per Security Engineer SUGGESTION-2, a monitoring threshold of 500 emails/day/user is tracked via structured logging. Phase 1 uses log-based monitoring (not a hard block). The deliver task logs the cumulative send count, and ops configures alerts on the structured log output.

The deliver task log entry (in section 5.4) includes `sent` and `schedule_id` fields. An ops team can aggregate `sent` counts per `owner_id` per day from structured logs to detect threshold breaches.

---

## 6. Frontend Interface Contracts

### 6.1 TypeScript Types

```typescript
// packages/ui/src/schemas/schedule.ts

import { z } from "zod";

// -- Enums --

export const scheduleStatusSchema = z.enum(["active", "paused", "deleted"]);
export type ScheduleStatus = z.infer<typeof scheduleStatusSchema>;

export const cadenceTypeSchema = z.enum(["daily", "weekly", "monthly"]);
export type CadenceType = z.infer<typeof cadenceTypeSchema>;

export const runStatusSchema = z.enum([
    "pending",
    "generating",
    "generated",
    "delivering",
    "delivered",
    "partially_delivered",
    "delivery_failed",
    "generation_failed",
]);
export type RunStatus = z.infer<typeof runStatusSchema>;

export const recipientDeliveryStatusSchema = z.enum(["pending", "sent", "failed"]);
export type RecipientDeliveryStatus = z.infer<typeof recipientDeliveryStatusSchema>;

// -- Pagination --

export const paginationSchema = z.object({
    nextCursor: z.string().nullable(),
    hasMore: z.boolean(),
});

// -- Recipient --

export const recipientSchema = z.object({
    id: z.number(),
    email: z.string().email(),
    isActive: z.boolean(),
});
export type Recipient = z.infer<typeof recipientSchema>;

// -- Run Summary (in schedule detail) --

export const runSummarySchema = z.object({
    id: z.number(),
    status: runStatusSchema,
    startedAt: z.string().datetime().nullable(),
    completedAt: z.string().datetime().nullable(),
    errorMessage: z.string().nullable(),
    createdAt: z.string().datetime(),
});
export type RunSummary = z.infer<typeof runSummarySchema>;

// -- Schedule (full) --

export const scheduleSchema = z.object({
    id: z.number(),
    reportTypeId: z.string(),
    cadenceType: cadenceTypeSchema,
    cadenceDay: z.number().nullable(),
    scheduleTime: z.string(),
    timezone: z.string(),
    status: scheduleStatusSchema,
    nextRunAt: z.string().datetime().nullable(),
    recipients: z.array(recipientSchema),
    recentRuns: z.array(runSummarySchema).default([]),
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
});
export type Schedule = z.infer<typeof scheduleSchema>;

// -- Schedule List Item --

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
export type ScheduleListItem = z.infer<typeof scheduleListItemSchema>;

// -- Paginated Schedule List --

export const paginatedScheduleListSchema = z.object({
    data: z.array(scheduleListItemSchema),
    pagination: paginationSchema,
});

// -- Run Detail --

export const runRecipientSchema = z.object({
    email: z.string(),
    status: recipientDeliveryStatusSchema,
    deliveredAt: z.string().datetime().nullable(),
    errorMessage: z.string().nullable(),
});
export type RunRecipientDetail = z.infer<typeof runRecipientSchema>;

export const runDetailSchema = z.object({
    id: z.number(),
    status: runStatusSchema,
    startedAt: z.string().datetime().nullable(),
    completedAt: z.string().datetime().nullable(),
    errorMessage: z.string().nullable(),
    createdAt: z.string().datetime(),
    recipients: z.array(runRecipientSchema),
});
export type RunDetail = z.infer<typeof runDetailSchema>;

export const paginatedRunListSchema = z.object({
    data: z.array(runDetailSchema),
    pagination: paginationSchema,
});

// -- Report Type --

export const reportTypeSchema = z.object({
    id: z.string(),
    name: z.string(),
    description: z.string(),
});
export type ReportType = z.infer<typeof reportTypeSchema>;

// -- Status Update Response --

export const statusUpdateSchema = z.object({
    id: z.number(),
    status: scheduleStatusSchema,
    nextRunAt: z.string().datetime().nullable(),
});

// -- Create/Update Request Types (not Zod -- used for form data) --

export interface ScheduleCreateRequest {
    reportTypeId: string;
    cadenceType: CadenceType;
    cadenceDay: number | null;
    scheduleTime: string;
    timezone: string;
    recipients: string[]; // email strings
}

export interface ScheduleUpdateRequest {
    reportTypeId?: string;
    cadenceType?: CadenceType;
    cadenceDay?: number | null;
    scheduleTime?: string;
    timezone?: string;
    recipients?: string[];
}
```

### 6.2 TanStack Query Hooks

```typescript
// packages/ui/src/hooks/schedules.ts

import {
    useMutation,
    useQuery,
    useQueryClient,
    type UseQueryOptions,
} from "@tanstack/react-query";

import {
    createSchedule,
    deleteSchedule,
    fetchSchedule,
    fetchScheduleRuns,
    fetchSchedules,
    pauseSchedule,
    resumeSchedule,
    updateSchedule,
} from "@/services/schedules";
import type {
    Schedule,
    ScheduleCreateRequest,
    ScheduleUpdateRequest,
} from "@/schemas/schedule";

// Query key factory
export const scheduleKeys = {
    all: ["schedules"] as const,
    lists: () => [...scheduleKeys.all, "list"] as const,
    list: (params: { cursor?: string; status?: string }) =>
        [...scheduleKeys.lists(), params] as const,
    details: () => [...scheduleKeys.all, "detail"] as const,
    detail: (id: number) => [...scheduleKeys.details(), id] as const,
    runs: (id: number) => [...scheduleKeys.detail(id), "runs"] as const,
    runList: (id: number, params: { cursor?: string; status?: string }) =>
        [...scheduleKeys.runs(id), params] as const,
};

export function useSchedules(params: { cursor?: string; status?: string } = {}) {
    return useQuery({
        queryKey: scheduleKeys.list(params),
        queryFn: () => fetchSchedules(params),
    });
}

export function useSchedule(id: number) {
    return useQuery({
        queryKey: scheduleKeys.detail(id),
        queryFn: () => fetchSchedule(id),
    });
}

export function useScheduleRuns(
    id: number,
    params: { cursor?: string; status?: string } = {},
) {
    return useQuery({
        queryKey: scheduleKeys.runList(id, params),
        queryFn: () => fetchScheduleRuns(id, params),
    });
}

export function useCreateSchedule() {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: (data: ScheduleCreateRequest) => createSchedule(data),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: scheduleKeys.lists() });
        },
    });
}

export function useUpdateSchedule(id: number) {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: (data: ScheduleUpdateRequest) => updateSchedule(id, data),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: scheduleKeys.detail(id) });
            queryClient.invalidateQueries({ queryKey: scheduleKeys.lists() });
        },
    });
}

export function usePauseSchedule(id: number) {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: () => pauseSchedule(id),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: scheduleKeys.detail(id) });
            queryClient.invalidateQueries({ queryKey: scheduleKeys.lists() });
        },
    });
}

export function useResumeSchedule(id: number) {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: () => resumeSchedule(id),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: scheduleKeys.detail(id) });
            queryClient.invalidateQueries({ queryKey: scheduleKeys.lists() });
        },
    });
}

export function useDeleteSchedule() {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: (id: number) => deleteSchedule(id),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: scheduleKeys.lists() });
        },
    });
}
```

```typescript
// packages/ui/src/hooks/report-types.ts

import { useQuery } from "@tanstack/react-query";
import { fetchReportTypes } from "@/services/report-types";

export const reportTypeKeys = {
    all: ["report-types"] as const,
};

export function useReportTypes() {
    return useQuery({
        queryKey: reportTypeKeys.all,
        queryFn: fetchReportTypes,
        staleTime: 5 * 60 * 1000, // Report types change rarely
    });
}
```

### 6.3 Key Component Breakdown

| Component | Type | Responsibility |
|-----------|------|---------------|
| `ScheduleListPage` | Route/page | `routes/schedules/index.tsx`. Fetches paginated list, renders table with status indicators. |
| `ScheduleDetailPage` | Route/page | `routes/schedules/$id.tsx`. Fetches schedule with recipients and recent runs. |
| `ScheduleCreatePage` | Route/page | `routes/schedules/create.tsx`. Multi-step form: report type, cadence, recipients, confirm. |
| `ScheduleEditPage` | Route/page | `routes/schedules/$id/edit.tsx`. Pre-populated form for editing. |
| `ScheduleForm` | Organism | Reusable form for create and edit. Handles cadence configuration, timezone picker, recipient input. |
| `CadenceSelector` | Molecule | Frequency dropdown + conditional day/time inputs. |
| `TimezoneSelector` | Molecule | IANA timezone picker with search. |
| `RecipientInput` | Molecule | Email input with add/remove, validation, deduplication. |
| `DeliveryHistory` | Organism | Paginated run list with status badges. |
| `RunDetail` | Organism | Per-recipient delivery results for a single run. |
| `StatusBadge` | Atom | Color-coded badge for schedule/run status. |
| `ConfirmDialog` | Atom | Reusable confirmation modal for delete/pause. |

---

## 7. Authorization

### 7.1 Current User Dependency

The authentication system is assumed to exist. The scheduling feature integrates with it via a FastAPI dependency that extracts the current user from the Bearer token.

```python
# packages/api/src/core/auth.py

from dataclasses import dataclass

from fastapi import Depends, HTTPException, Request


@dataclass
class CurrentUser:
    """Authenticated user extracted from the request."""
    id: int
    is_admin: bool


async def get_current_user(request: Request) -> CurrentUser:
    """FastAPI dependency: extract and validate the current user.

    This is a placeholder that must be connected to the real auth system.
    For development/testing, it reads from a header or returns a default user.
    """
    # TODO: Replace with real auth integration
    # For development, accept X-User-Id and X-User-Admin headers
    user_id = request.headers.get("X-User-Id")
    if not user_id:
        raise HTTPException(status_code=401, detail="Authentication required.")
    try:
        return CurrentUser(
            id=int(user_id),
            is_admin=request.headers.get("X-User-Admin", "false").lower() == "true",
        )
    except ValueError:
        raise HTTPException(status_code=401, detail="Invalid user identity.")
```

### 7.2 Owner-Scoped Query Pattern

Per Security Engineer SUGGESTION-1, all queries are pre-filtered by `owner_id`. The pattern returns 404 (not 403) when a schedule is not found or not owned, preventing information disclosure.

```python
# packages/api/src/core/auth.py (continued)

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from db import Schedule, ScheduleStatus


async def get_owned_schedule(
    schedule_id: int,
    user: CurrentUser,
    session: AsyncSession,
) -> Schedule:
    """Fetch a schedule owned by the current user or accessible by admin.

    Returns 404 if not found OR not owned (no information disclosure).
    Excludes soft-deleted schedules.
    """
    stmt = select(Schedule).where(
        Schedule.id == schedule_id,
        Schedule.status != ScheduleStatus.DELETED,
    )
    if not user.is_admin:
        stmt = stmt.where(Schedule.owner_id == user.id)

    result = await session.execute(stmt)
    schedule = result.scalar_one_or_none()

    if schedule is None:
        raise HTTPException(status_code=404, detail="Schedule not found.")

    return schedule
```

### 7.3 Rate Limiting

Rate limiting uses a Redis-backed sliding window counter. It applies to both create and update endpoints (per requirements section 4, acknowledged as broader than architecture section 6.6 by Architect review W2).

```python
# packages/api/src/core/rate_limit.py

import os
import time

from fastapi import HTTPException, Request
from redis import ConnectionPool, Redis


RATE_LIMIT = 10  # requests per window
RATE_WINDOW = 60  # seconds

# Module-level connection pool and client singleton. This avoids creating a
# new TCP connection per rate-limited request, which would exhaust file
# descriptors under load. The pool manages connection lifecycle and reuse.
_redis_pool: ConnectionPool | None = None


def _get_redis_client() -> Redis:
    """Return a Redis client backed by a shared connection pool.

    The pool is lazily initialized on first call and reused for all
    subsequent calls. The pool size (max_connections=20) is appropriate
    for the API process; adjust if running behind multiple workers.
    """
    global _redis_pool
    if _redis_pool is None:
        redis_url = os.environ.get("REDIS_URL", "redis://localhost:6379/0")
        _redis_pool = ConnectionPool.from_url(redis_url, max_connections=20)
    return Redis(connection_pool=_redis_pool)


async def check_rate_limit(user_id: int) -> dict[str, str]:
    """Check rate limit for schedule mutations. Returns rate limit headers.

    Raises 429 if limit exceeded.
    """
    redis_client = _get_redis_client()
    key = f"rate_limit:schedule:{user_id}"
    now = time.time()
    window_start = now - RATE_WINDOW

    pipe = redis_client.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, RATE_WINDOW)
    results = pipe.execute()

    current_count = results[2]
    remaining = max(0, RATE_LIMIT - current_count)

    headers = {
        "X-RateLimit-Limit": str(RATE_LIMIT),
        "X-RateLimit-Remaining": str(remaining),
        "X-RateLimit-Reset": str(int(now + RATE_WINDOW)),
    }

    if current_count > RATE_LIMIT:
        raise HTTPException(
            status_code=429,
            detail=f"Too many requests. Try again in {RATE_WINDOW} seconds.",
            headers=headers,
        )

    return headers
```

---

## 8. File Structure

### 8.1 packages/db

```
packages/db/
  src/
    __init__.py                    # Public exports (models, enums, session factories)
    db/
      __init__.py
      database.py                  # Base, engine, get_session, get_sync_session
      enums.py                     # ScheduleStatus, CadenceType, RunStatus, RecipientDeliveryStatus
      models.py                    # Schedule, ScheduleRecipient, ScheduleRun, RunRecipient
  alembic/
    env.py                         # Alembic environment config
    script.py.mako                 # Migration template
    versions/
      001_initial_schema.py        # Initial migration for all 4 tables
  alembic.ini
  pyproject.toml
```

### 8.2 packages/api

```
packages/api/
  src/
    __init__.py
    main.py                        # FastAPI app factory, router registration, error handlers
    core/
      __init__.py
      config.py                    # Settings (Pydantic BaseSettings)
      auth.py                      # get_current_user, get_owned_schedule, CurrentUser
      rate_limit.py                # Redis-backed sliding window rate limiter
      errors.py                    # RFC 7807 error response helpers, exception handlers
    routes/
      __init__.py
      schedules.py                 # POST/GET/PATCH/DELETE /v1/schedules, pause, resume
      schedule_runs.py             # GET /v1/schedules/{id}/runs
      report_types.py              # GET /v1/report-types
    schemas/
      __init__.py
      schedules.py                 # ScheduleCreate, ScheduleUpdate, RecipientCreate
      responses.py                 # ScheduleResponse, RunDetailResponse, PaginationMeta, etc.
    services/
      __init__.py
      state_machine.py             # validate_schedule_transition, validate_run_transition
      next_run.py                  # compute_next_run (pure function)
      report_generator.py          # ReportGenerator protocol + StubReportGenerator
      email_sender.py              # EmailSender protocol + SmtpEmailSender
    tasks/
      __init__.py                  # Celery app setup, beat schedule
      tick.py                      # tick_task
      execute.py                   # execute_schedule
      deliver.py                   # deliver_report
  tests/
    conftest.py                    # Shared fixtures (test client, test DB, mock services)
    test_schedules.py              # Schedule CRUD endpoint tests
    test_schedule_runs.py          # Run history endpoint tests
    test_report_types.py           # Report types endpoint tests
    test_state_machine.py          # State transition validation tests
    test_next_run.py               # Next run computation tests (pure function)
    test_tasks.py                  # Celery task tests (tick, execute, deliver)
  pyproject.toml
```

### 8.3 packages/ui

```
packages/ui/
  src/
    schemas/
      schedule.ts                  # Zod schemas + TypeScript types
    services/
      schedules.ts                 # API client functions (fetchSchedules, createSchedule, etc.)
      report-types.ts              # fetchReportTypes
    hooks/
      schedules.ts                 # TanStack Query hooks for schedules
      report-types.ts              # useReportTypes hook
    routes/
      __root.tsx                   # Root layout
      schedules/
        index.tsx                  # Schedule list page
        create.tsx                 # Schedule creation page
        $id.tsx                    # Schedule detail page
        $id/
          edit.tsx                 # Schedule edit page
    components/
      atoms/
        status-badge.tsx           # Color-coded status badge
        confirm-dialog.tsx         # Reusable confirmation modal
      molecules/
        cadence-selector.tsx       # Frequency + day + time inputs
        timezone-selector.tsx      # IANA timezone picker with search
        recipient-input.tsx        # Email input with add/remove/validate
      organisms/
        schedule-form.tsx          # Reusable create/edit form
        delivery-history.tsx       # Paginated run list
        run-detail.tsx             # Per-recipient delivery results
  vitest.config.ts
  package.json
```

---

## 9. Implementation Tasks

Tasks are ordered by dependencies. Each touches 3-5 files and has a machine-verifiable exit condition.

### Task 1: Database Schema and Migrations

**Scope:** Create SQLAlchemy models, enums, session factories, and the initial Alembic migration.

**Files (4):**
- `packages/db/src/db/enums.py` (new)
- `packages/db/src/db/models.py` (new)
- `packages/db/src/db/database.py` (new)
- `packages/db/src/__init__.py` (new)

Plus the Alembic setup (`alembic.ini`, `alembic/env.py`, `alembic/versions/001_initial_schema.py`).

**Exit condition:**
```bash
cd packages/db && uv run alembic upgrade head && uv run python -c "from db import Schedule, ScheduleRun, ScheduleRecipient, RunRecipient, get_session, get_sync_session; print('OK')"
```

**Dependencies:** None. This is the foundation.

---

### Task 2: API Core Infrastructure

**Scope:** FastAPI app factory, configuration, auth dependencies, RFC 7807 error handlers, rate limiting.

**Files (5):**
- `packages/api/src/main.py` (new)
- `packages/api/src/core/config.py` (new)
- `packages/api/src/core/auth.py` (new)
- `packages/api/src/core/rate_limit.py` (new)
- `packages/api/src/core/errors.py` (new)

**Exit condition:**
```bash
cd packages/api && uv run pytest tests/test_health.py -v
```
Where `test_health.py` tests:
- `GET /healthz` returns 200
- `GET /readyz` returns 200 (with DB and Redis available)
- Unauthenticated requests return 401
- RFC 7807 error shape is correct

**Dependencies:** Task 1 (database models must exist for readyz check).

---

### Task 3: Schedule CRUD Endpoints + Pydantic Schemas

**Scope:** All schedule endpoints (create, list, get detail, update, delete, pause, resume) with Pydantic schemas, state machine validation, and next_run computation.

**Files (5):**
- `packages/api/src/schemas/schedules.py` (new)
- `packages/api/src/schemas/responses.py` (new)
- `packages/api/src/routes/schedules.py` (new)
- `packages/api/src/services/state_machine.py` (new)
- `packages/api/src/services/next_run.py` (new)

**Exit condition:**
```bash
cd packages/api && uv run pytest tests/test_schedules.py tests/test_state_machine.py tests/test_next_run.py -v
```
Tests verify:
- POST creates a schedule and returns 201 with correct shape
- GET list returns paginated schedules scoped to owner
- GET detail returns schedule with recipients
- PATCH updates fields and recomputes next_run_at
- DELETE soft-deletes
- Pause/resume transition validation (409 on invalid)
- Authorization: 404 when accessing another user's schedule
- Validation: invalid timezone, cadence_day, recipients
- Rate limiting: 429 after 10 requests/minute
- State machine: all valid and invalid transitions
- next_run: daily, weekly, monthly, DST spring-forward, DST fall-back, monthly day clamping

**Dependencies:** Task 1, Task 2.

---

### Task 4: Delivery History Endpoint + Report Types Endpoint

**Scope:** Run history endpoint with cursor pagination and status filtering. Report types endpoint with hardcoded list.

**Files (3):**
- `packages/api/src/routes/schedule_runs.py` (new)
- `packages/api/src/routes/report_types.py` (new)
- `packages/api/tests/test_schedule_runs.py` (new -- test file)

**Exit condition:**
```bash
cd packages/api && uv run pytest tests/test_schedule_runs.py tests/test_report_types.py -v
```
Tests verify:
- GET /v1/schedules/{id}/runs returns paginated runs with per-recipient detail
- Cursor pagination works (next page returns correct items)
- Status filter works
- GET /v1/report-types returns hardcoded list
- Authorization: 404 for runs of non-owned schedule

**Dependencies:** Task 1, Task 2, Task 3 (schedule routes must exist to create test data).

---

### Task 5: Celery Tasks (Tick, Execute, Deliver) + Service Protocols

**Scope:** All three Celery tasks plus the ReportGenerator and EmailSender protocols and stub implementations.

**Files (5):**
- `packages/api/src/tasks/__init__.py` (new -- Celery app)
- `packages/api/src/tasks/tick.py` (new)
- `packages/api/src/tasks/execute.py` (new)
- `packages/api/src/tasks/deliver.py` (new)
- `packages/api/src/services/report_generator.py` (new)

Plus `packages/api/src/services/email_sender.py` (new -- 6th file, but the protocol definition is a contract dependency for the task code; acceptable given that protocols are small interface files).

**Exit condition:**
```bash
cd packages/api && uv run pytest tests/test_tasks.py -v
```
Tests verify:
- Tick task finds due schedules and creates runs
- Tick task skips paused/deleted schedules
- Tick task computes next_run_at correctly
- Execute task transitions run through generating -> generated
- Execute task handles generation failure -> generation_failed (terminal)
- Deliver task sends to all recipients and records results
- Deliver task handles partial failure -> partially_delivered
- Deliver task handles total failure -> delivery_failed
- Input validation at task entry (invalid run_id, wrong state)
- CRLF rejection in email addresses

**Dependencies:** Task 1, Task 3 (state machine and next_run services).

---

### Task 6: Frontend Schemas, Services, and Hooks

**Scope:** Zod schemas, API client services, and TanStack Query hooks.

**Files (4):**
- `packages/ui/src/schemas/schedule.ts` (new)
- `packages/ui/src/services/schedules.ts` (new)
- `packages/ui/src/services/report-types.ts` (new)
- `packages/ui/src/hooks/schedules.ts` (new)

Plus `packages/ui/src/hooks/report-types.ts` (new -- small file).

**Exit condition:**
```bash
cd packages/ui && pnpm vitest run --reporter=verbose src/schemas/ src/hooks/
```
Tests verify:
- Zod schemas parse valid API responses correctly
- Zod schemas reject invalid shapes
- Hook query keys match expected structure

**Dependencies:** None (frontend schemas are defined from the API contract in this document, not from running backend code). Can run in parallel with Tasks 3-5.

---

### Task 7: Frontend Pages and Components

**Scope:** All route pages and UI components for schedule management.

**Files (5 primary + atom/molecule files):**
- `packages/ui/src/routes/schedules/index.tsx` (new -- list page)
- `packages/ui/src/routes/schedules/create.tsx` (new -- create page)
- `packages/ui/src/routes/schedules/$id.tsx` (new -- detail page)
- `packages/ui/src/components/organisms/schedule-form.tsx` (new)
- `packages/ui/src/components/organisms/delivery-history.tsx` (new)

Supporting atom/molecule components (`status-badge.tsx`, `cadence-selector.tsx`, `timezone-selector.tsx`, `recipient-input.tsx`, `confirm-dialog.tsx`, `run-detail.tsx`) are small and created alongside their parent organisms.

**Exit condition:**
```bash
cd packages/ui && pnpm vitest run --reporter=verbose src/routes/ src/components/ && pnpm tsc --noEmit
```
Tests verify:
- Schedule list renders loading, empty, and populated states
- Schedule form validates required fields
- Cadence selector shows/hides day input based on frequency
- Recipient input validates email format
- Delete confirmation dialog works
- TypeScript compiles without errors

**Dependencies:** Task 6 (hooks and schemas must exist).

---

## 10. Cross-Task Dependencies

| Produces | Consumed By | Contract |
|----------|-------------|----------|
| Task 1 (DB models + enums) | Tasks 2, 3, 4, 5 | SQLAlchemy models, enums, session factories |
| Task 2 (API core) | Tasks 3, 4 | FastAPI app, auth dependencies, error handlers |
| Task 3 (Schedule CRUD + state machine + next_run) | Tasks 4, 5 | State machine functions, next_run function, schedule routes |
| Task 5 (Celery tasks + protocols) | Task 4 (test data) | Run records created by tasks for history endpoint tests |
| Task 6 (Frontend schemas + hooks) | Task 7 | Zod types, TanStack Query hooks |
| This document (API contracts) | Task 6 | JSON response shapes define Zod schemas |

Parallelizable: Tasks 6-7 (frontend) can proceed in parallel with Tasks 3-5 (backend) since they share contracts defined in this document, not runtime dependencies.

```
Task 1 (DB) --> Task 2 (API Core) --> Task 3 (Schedule CRUD)
                                          |
                                          +--> Task 4 (Runs + Report Types)
                                          |
                                          +--> Task 5 (Celery Tasks)

Task 6 (FE Schemas/Hooks) --> Task 7 (FE Pages)
```

---

## 11. Context Package

### Work Area: Database Layer

**Files to read:**
- `packages/db/src/db/enums.py` -- all enum definitions
- `packages/db/src/db/models.py` -- all model definitions
- `packages/db/src/db/database.py` -- session factories

**Binding contracts:**
- 4 models: Schedule, ScheduleRecipient, ScheduleRun, RunRecipient (section 2)
- 4 enums: ScheduleStatus, CadenceType, RunStatus, RecipientDeliveryStatus (section 2.1)
- Indexes: `ix_schedules_tick(status, next_run_at)`, `ix_schedules_owner(owner_id, status)`, `ix_recipients_schedule_active(schedule_id, is_active)`, `ix_runs_schedule_created(schedule_id, created_at)`

**Key decisions:**
- RunRecipient.recipient_email is a snapshot, not a FK to schedule_recipients
- schedule_hour and schedule_minute are integers (not a Time column) for simpler computation
- Soft delete for schedules (status=deleted) and recipients (is_active=false)

**Scope:** Models, enums, session management only. No business logic.

---

### Work Area: API Endpoints

**Files to read:**
- `packages/api/src/schemas/schedules.py` -- request schemas
- `packages/api/src/schemas/responses.py` -- response schemas
- `packages/api/src/core/auth.py` -- auth dependencies and owner-scoped query pattern
- `packages/api/src/core/errors.py` -- RFC 7807 error helpers
- `packages/api/src/services/state_machine.py` -- transition validation

**Binding contracts:**
- API request/response shapes (section 3)
- Pydantic schemas (section 4)
- Owner-scoped query pattern: pre-filter by owner_id, return 404 on not-found-or-not-owned (section 7.2)
- State transition validation: 409 on invalid transitions (section 2.8)
- Rate limit: 10 requests/minute/user on create and update (section 7.3)

**Key decisions:**
- Recipients managed inline via schedule endpoints (replacement semantics on PATCH)
- Pause/resume use dedicated action endpoints, not PATCH with status field
- camelCase JSON via Pydantic alias, snake_case internal
- Cursor-based pagination for both schedule list and run list
- Admin users bypass owner filter; no other admin-specific endpoints in Phase 1

**Scope:** REST endpoints, schemas, auth, errors. No Celery tasks.

---

### Work Area: Celery Tasks

**Files to read:**
- `packages/api/src/tasks/tick.py` -- tick task
- `packages/api/src/tasks/execute.py` -- execute task
- `packages/api/src/tasks/deliver.py` -- deliver task
- `packages/api/src/services/report_generator.py` -- ReportGenerator protocol
- `packages/api/src/services/email_sender.py` -- EmailSender protocol

**Binding contracts:**
- Task signatures: `tick_task()`, `execute_schedule(run_id: int)`, `deliver_report(run_id: int, report_content: str, content_type: str)`
- ReportGenerator protocol: `generate(report_type_id: str, schedule_id: int) -> ReportOutput`
- EmailSender protocol: `send(to_email, subject, html_body, attachment_content, attachment_content_type, attachment_filename) -> None`
- Input validation at task entry for all tasks (Security Engineer WARNING-2)
- CRLF rejection in email addresses and subjects (Security Engineer WARNING-1)
- Run status state machine transitions (section 2.8)

**Key decisions:**
- Workers use synchronous DB sessions (`get_sync_session()`), not async
- Report content is base64-encoded for JSON serialization via Celery
- No automatic retries in Phase 1 (max_retries=0)
- `task_acks_late=True` and `task_reject_on_worker_lost=True` for crash safety
- Email volume monitoring via structured logging (500/day/user threshold)

**Scope:** Background task execution only. No HTTP handling.

---

### Work Area: Frontend

**Files to read:**
- `packages/ui/src/schemas/schedule.ts` -- Zod schemas and TypeScript types
- `packages/ui/src/hooks/schedules.ts` -- TanStack Query hooks
- `packages/ui/src/services/schedules.ts` -- API client functions

**Binding contracts:**
- Zod schemas match API response shapes exactly (section 6.1)
- TanStack Query key factory: `scheduleKeys.all`, `.lists()`, `.list(params)`, `.detail(id)`, `.runs(id)`, `.runList(id, params)` (section 6.2)
- Component -> Hook -> Service -> API data flow pattern

**Key decisions:**
- All timestamps displayed in user's browser local timezone (converted from UTC)
- Schedule form is a reusable organism for both create and edit
- Recipient input handles client-side deduplication and email format validation
- Timezone selector uses a curated list with search (not the full IANA database)

**Scope:** UI rendering, state management, API integration. No backend logic.

---

## 12. Risks and Open Questions

| Risk | Mitigation |
|------|------------|
| Auth system does not exist yet | `get_current_user` is a stub dependency. Phase 1 can develop and test with header-based auth. Real auth integration is a prerequisite for production but not for development. |
| Report type validation (free-form string vs. validated list) | Phase 1 validates `report_type_id` against a hardcoded list in config. When the real report system is integrated, the list becomes dynamic. The validation location (config-based) is a single point of change. |
| Celery worker crash during run execution | `task_acks_late=True` ensures unfinished tasks are re-queued. The run record exists (created by tick) so the status shows the in-progress state. Runs stuck in `pending` (orphaned by a tick crash between commit and dispatch) are automatically recovered by the next tick invocation, which re-dispatches any `pending` runs older than 5 minutes. Runs stuck in `generating` or `delivering` (worker crash mid-task) have no automatic recovery in Phase 1; Phase 2 adds timeout-based cleanup for these states. Ops should monitor for runs stuck in non-terminal states for more than 10 minutes. |
| SMTP connection failure blocks all deliveries | The EmailSender has a 30-second timeout per connection. If SMTP is down, all deliveries fail with error messages. The schedule continues on the next cycle. No automatic retry in Phase 1. |
| Monthly day clamping may surprise users | Day 31 in February delivers on Feb 28/29. This matches the product plan resolution (open question 6) and is documented in the UI, but users may not notice. Phase 2 adds a schedule validation warning. |

| Open Question | Impact | Recommendation |
|---------------|--------|----------------|
| How many report types exist? | Affects whether `/v1/report-types` needs pagination. | Phase 1 assumes < 20. If more, add pagination in a follow-up. |
| Does the auth system provide an `is_admin` flag? | Affects the `CurrentUser` dataclass and admin bypass. | Assume yes. If not, the admin check can be deferred to Phase 2 admin dashboard. |
| SMTP credentials in production | Security Engineer INFO-2 flagged this. | Use environment variables for MVP. Document the requirement for Kubernetes Secrets in production. The DevOps Engineer handles this during deployment setup. |

---

## 13. Checklist

- [x] All cross-task interface contracts defined with concrete types (no TBDs)
- [x] Data flow covers happy path and primary error paths (section 5)
- [x] Error handling strategy is feature-specific: RFC 7807, state machine 409s, validation 422s, rate limit 429s
- [x] File/module structure maps to existing project layout (section 8)
- [x] Every implementation task has a machine-verifiable exit condition (section 9)
- [x] Key technical decisions documented with rationale
- [x] Cross-task dependency map complete (section 10)
- [x] Context Package defined for each work area (section 11)
- [x] No TBDs in binding contracts -- open questions flagged separately (section 12)
- [x] Email header injection prevention addressed (Security Engineer WARNING-1)
- [x] Celery task input validation at entry (Security Engineer WARNING-2)
- [x] Query-level authorization with pre-filtering (Security Engineer SUGGESTION-1)
- [x] Email volume monitoring threshold documented (Security Engineer SUGGESTION-2)
- [x] Schedule creation response includes inline recipients (API Designer SUGGESTION-1)
- [x] Cursor-based pagination for schedules and runs (API Designer SUGGESTION-2, SUGGESTION-3)
- [x] Run history endpoint with status filtering (API Designer SUGGESTION-3)
- [x] Report types endpoint defined (API Designer SUGGESTION-4)
- [x] State transition validation at API boundary (API Designer SUGGESTION-5)
- [x] Rate limit applies to both creates and updates (Architect review W2)
- [x] Concurrent modification: last-write-wins (Architect review S2)
- [x] State machine enforcement via shared utility function (Architect review observation 1)
- [x] DB package does not depend on API package config (Code Reviewer W1)
- [x] Orphan recovery for pending runs in tick task (Code Reviewer W2)
- [x] PATCH semantics use model_fields_set for null disambiguation (Code Reviewer W3)
- [x] started_at field semantics clarified (Code Reviewer W4)
- [x] Redis connection pool singleton for rate limiting (Code Reviewer W5)

---

## 14. Revision Log

### Revision 1 -- 2026-02-10 (Code Reviewer feedback)

Addresses all 5 warnings from the Code Reviewer's Technical Design review.

**W1: Circular dependency in database.py (section 2.7)**

The reviewer identified that `packages/db/src/db/database.py` imported `from src.core.config import get_settings`, which belongs to `packages/api`. This violates the architecture boundary: `packages/api` imports from `packages/db`, not the reverse. If the DB package depends on the API package, the two packages become circularly coupled and cannot be installed independently.

*Fix:* Replaced the `get_settings()` import with direct `os.environ.get()` calls for `DATABASE_URL` and `SYNC_DATABASE_URL`. The DB package now reads its own connection configuration from environment variables, with sensible development defaults. The API package's `Settings` class (which may set these env vars) remains in `packages/api` where it belongs. The DB package has zero imports from `packages/api`.

**W2: Tick task commit-then-dispatch gap (section 5.2)**

The reviewer identified a crash window: if the tick worker crashes after committing `ScheduleRun(status=pending)` + advancing `next_run_at` but before `execute_schedule.delay()` succeeds, the run is orphaned in `pending` state forever. No mechanism existed to detect or recover these orphans.

*Fix:* Added an orphan recovery phase to the tick task. Before processing new due schedules, the tick now queries for `ScheduleRun` records stuck in `pending` state for more than 5 minutes and re-dispatches `execute_schedule.delay()` for each. This is safe because `execute_schedule` is idempotent on entry -- it checks the run is in `PENDING` state before proceeding, so a duplicate dispatch (where the original task is still in-flight or already completed) exits harmlessly. Updated the Risks table (section 12) to reflect that `pending`-state orphans are now automatically recovered.

**W3: PATCH partial update ambiguity (section 4.1, section 3.4)**

The reviewer identified the classic PATCH ambiguity: with all fields typed as `X | None` and defaulting to `None`, Pydantic cannot distinguish "field not sent in the request" from "field explicitly set to null". This matters for `cadence_day`, where switching from weekly to daily requires explicitly setting `cadenceDay: null`.

*Fix:* Added documentation to the `ScheduleUpdate` docstring specifying that the route handler must use `model_fields_set` to determine which fields the client explicitly provided. Only fields in `model_fields_set` are applied to the database model. Added a concrete code pattern showing the correct iteration. Also added an implementation note to the PATCH endpoint description in section 3.4 explaining this requirement.

**W4: started_at field ambiguity (section 2.4, section 5.4)**

The reviewer identified that `started_at` was serving double duty: set by `execute_schedule` (section 5.3) when generation starts, then conditionally re-assigned in `deliver_report` (section 5.4) with `run.started_at = run.started_at or datetime.now(...)`. This obscured whether `started_at` means "generation started" or "delivery started" and the conditional assignment added confusion about which task owns the field.

*Fix:* Clarified the semantics: `started_at` means "when the first processing step (report generation) began". Added a column comment to the `ScheduleRun` model documenting this. Removed the conditional `started_at` assignment from the deliver task entirely -- it was always a no-op in the happy path (execute_schedule already set it), and the defensive fallback masked the real question of what the field means. If started_at is somehow null when deliver runs, that is a bug in execute_schedule and should be visible, not silently patched.

**W5: Redis client per request (section 7.3)**

The reviewer identified that `check_rate_limit` called `Redis.from_url()` on every invocation, creating a new TCP connection per rate-limited request. Under load, this would exhaust file descriptors and cause latency spikes from repeated TCP handshakes.

*Fix:* Replaced the per-call `Redis.from_url()` with a module-level `ConnectionPool` singleton, lazily initialized on first use. The `_get_redis_client()` helper returns a `Redis` instance backed by the shared pool (max 20 connections). The pool manages connection lifecycle and reuse. Also removed the dependency on `src.core.config.get_settings()` in favor of reading `REDIS_URL` directly from the environment, consistent with the pattern established in W1 for the DB package (though `rate_limit.py` lives in `packages/api`, so the import was not a boundary violation -- the change is for consistency and to avoid eager settings evaluation at import time).
