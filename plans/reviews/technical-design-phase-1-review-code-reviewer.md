# Technical Design Review: Report Scheduling -- Phase 1

**Reviewer:** Code Reviewer
**Date:** 2026-02-10
**Document under review:** `plans/technical-design-phase-1.md`
**Review type:** Technical Design Review (per review-governance.md checklist)

---

## Verdict: APPROVE WITH SUGGESTIONS

The Technical Design is thorough, well-structured, and implementation-ready. Contracts are concrete with actual JSON shapes, type definitions, and file paths. Data flow covers both happy and error paths. All 7 implementation tasks have machine-verifiable exit conditions. The design faithfully traces requirements US-001 through US-010 to specific endpoints and task implementations. The issues identified below are addressable without structural changes.

---

## TD Review Checklist Results

| Check | Result | Notes |
|-------|--------|-------|
| **Contracts are concrete** | PASS | Actual JSON shapes (sections 3.1-3.9), Pydantic schemas (section 4), Zod schemas (section 6.1), SQLAlchemy models (section 2), Protocol signatures (section 5.5). No hand-waving. |
| **Data flow covers error paths** | PASS | Sections 3, 5.2-5.4 cover validation failures, state transition violations, generation failures, partial delivery, SMTP failures, database errors, and worker crashes. |
| **Exit conditions are machine-verifiable** | PASS | All 7 tasks have bash commands that return pass/fail (pytest, vitest, tsc). |
| **File structure maps to actual codebase** | PASS (with note) | The `packages/` directory does not exist yet (greenfield), so there is nothing to conflict with. The proposed structure matches the architecture rule (`architecture.md`) and the project conventions (`api-development.md`, `ui-development.md`, `database-development.md`). |
| **No TBDs in binding contracts** | PASS | Open questions are isolated in section 12 and do not affect binding contracts. The single TODO in `get_current_user` (line 2152) is intentional and explicitly documented as a development stub. |

---

## Findings

### Critical

None.

### Warning

- **TD section 2.7 / `packages/db/src/db/database.py` (line 302)** -- The `database.py` module imports `from src.core.config import get_settings`, but this file lives in `packages/db`, not `packages/api`. The `db` package should not depend on the `api` package's configuration module. This creates a circular dependency: the architecture states that `packages/api` imports from `packages/db`, not the reverse.

  **Suggestion:** The database module should accept the database URL as a parameter (or read from its own environment configuration). Either (a) define a separate `packages/db/src/db/config.py` with its own `BaseSettings` that reads `DATABASE_URL` from the environment, or (b) refactor `database.py` to accept connection strings as constructor arguments and let `packages/api` wire them at startup. Option (b) is cleaner and aligns with the architecture's boundary rules.

- **TD section 5.2 / tick task (lines 1116-1141)** -- The tick task commits inside the `for schedule in due_schedules` loop (line 1136) and then dispatches the Celery task (line 1140). If the tick task crashes between `session.commit()` and `execute_schedule.delay()`, the schedule's `next_run_at` has been advanced and a `ScheduleRun(status=pending)` exists, but no execute task was dispatched. The run will be stuck in `pending` with no mechanism to recover it in Phase 1.

  **Suggestion:** Acknowledge this as a known limitation in section 12 (Risks) with the specific failure mode described. The `ScheduleRun(status=pending)` record is visible in delivery history, so it is not silent. Phase 2's "timeout-based cleanup" (already mentioned in section 12 for worker crashes) can cover this case as well. For Phase 1, document that ops should monitor for runs stuck in `pending` state for more than 5 minutes.

- **TD section 5.4 / deliver task (line 1371)** -- `run.started_at = run.started_at or datetime.now(dt_timezone.utc)` overwrites the `started_at` timestamp only if it was null. However, `started_at` was already set in the execute task (line 1225, `execute.py`). This conditional assignment is correct but obscures the data model semantics. `started_at` represents execution start (set by execute_schedule), not delivery start. The field is serving double duty for two different concepts.

  **Suggestion:** Consider adding a `delivered_at` or `delivery_started_at` field to `ScheduleRun`, or documenting explicitly that `started_at` means "first processing step started" (not "delivery started"). This is a minor data model ambiguity that could confuse implementers. At minimum, add a code comment in the deliver task explaining why the conditional is there.

- **TD section 4.1 / `ScheduleUpdate` schema (lines 853-878)** -- The `ScheduleUpdate` schema has `cadence_day` typed as `int | None` with a default of `None`. But because all fields are optional for PATCH, there is no way to distinguish between "cadence_day was not provided" (keep existing value) and "cadence_day was explicitly set to null" (which is the correct value for a daily cadence). This is the classic PATCH partial-update ambiguity problem. If a user changes cadence from weekly to daily, they need to send `cadenceDay: null`, but Pydantic will not distinguish this from the field being absent.

  **Suggestion:** Use Pydantic's `model_fields_set` attribute in the route handler to check which fields were explicitly provided. Document this pattern in the PATCH endpoint implementation notes. Alternatively, use a sentinel value (e.g., `UNSET = object()`) for optional fields. This is a common Pydantic v2 pattern that should be called out in the TD so implementers handle it correctly.

- **TD section 7.3 / rate_limit.py (lines 2228-2229)** -- The `check_rate_limit` function creates a new `Redis.from_url()` connection on every invocation. This bypasses connection pooling and will create a new TCP connection per rate-limited request. Under load, this could exhaust file descriptors or cause latency spikes.

  **Suggestion:** Initialize the Redis client once at module level or in the FastAPI app startup, and inject it as a dependency. The Celery app already has a Redis connection via its broker -- consider sharing the connection pool or creating a shared Redis client in `config.py`.

### Suggestion

- **TD section 3.1 / POST /v1/schedules -- Request shape (line 413-416)** -- The `recipients` field in the create request takes an array of email strings in the JSON contract (line 413: `"string (email, RFC 5322 format)"`), but the Pydantic schema (line 806) defines it as `list[RecipientCreate]` where `RecipientCreate` has an `email` field. This means the actual request body requires `"recipients": [{"email": "alice@example.com"}]`, not `"recipients": ["alice@example.com"]`. The frontend `ScheduleCreateRequest` interface (line 1954) defines recipients as `string[]` (plain email strings), which matches the JSON contract but not the Pydantic schema.

  **Suggestion:** Align the three representations. Either: (a) change the Pydantic schema to accept `list[EmailStr]` directly and wrap internally, or (b) update the JSON contract and frontend interface to use the `{"email": "..."}` object form. Option (a) is simpler for the API consumer. This inconsistency, if not resolved, will cause the frontend integration to break immediately.

- **TD section 2.2 / Schedule model (line 107)** -- `owner_id` has a standalone `index=True` on line 107, but the composite index `ix_schedules_owner(owner_id, status)` already covers queries filtering by `owner_id` alone (leftmost column). The standalone index is redundant and wastes storage.

  **Suggestion:** Remove the standalone `index=True` from the `owner_id` column definition. The composite index `ix_schedules_owner` serves both single-column and multi-column lookups.

- **TD section 2.2 / Schedule model (line 133)** -- `next_run_at` has a standalone `index=True` on line 133, but the composite index `ix_schedules_tick(status, next_run_at)` already covers the tick query. The standalone index is only useful for queries filtering solely on `next_run_at` without status, which do not appear in the design.

  **Suggestion:** Remove the standalone `index=True` from `next_run_at` unless there is a known query pattern that filters on `next_run_at` alone.

- **TD section 5.6 / compute_next_run (lines 1740-1741)** -- The function uses `assert cadence_day is not None` for weekly and monthly cadences. `assert` statements are removed when Python runs with `-O` (optimize), which means this validation silently disappears in optimized production builds.

  **Suggestion:** Replace `assert` with an explicit `if cadence_day is None: raise ValueError(...)`. This is a code style concern per the project's error handling conventions: validation logic should not rely on debug assertions.

- **TD section 5.6 / compute_next_run (lines 1772-1787)** -- The DST handling logic uses a round-trip check (`convert to UTC, then back to local, then check if the time changed`). This is a correct approach, but the nonexistent-time branch (line 1785-1787) replaces the round-trip value's minute with 0 and then converts to UTC. The comment says "use the post-transition time," but the code sets minute to 0 explicitly. For a spring-forward from 2:00 to 3:00, this correctly gives 3:00, but if a timezone transitions at a different offset (e.g., 30-minute transition), this would give an incorrect result.

  **Suggestion:** Add a comment documenting that this logic assumes standard 1-hour DST transitions. All current IANA timezones that observe DST use 1-hour transitions, so this is safe in practice, but the assumption should be explicit. Add a test case comment noting this.

- **TD section 6.1 / Zod schemas (line 1823-1826)** -- The `runStatusSchema` values use snake_case (`partially_delivered`, `delivery_failed`, `generation_failed`), but the API conventions rule says to use camelCase for JSON field names. The enum values themselves are not JSON field names (they are string enum values), so this is arguably correct per convention. However, it is worth noting that the Pydantic response schemas serialize these enum values as their `.value` (which is snake_case), not camelCased.

  **Suggestion:** Document explicitly that enum values are intentionally snake_case strings (matching the Python enum definitions) and are not subject to the camelCase JSON field name convention. This prevents future implementers or reviewers from "fixing" them.

- **TD section 9 / Task 5 (line 2484)** -- Task 5 acknowledges touching 6 files (exceeding the 3-5 file limit per agent-workflow.md) but justifies it because "protocols are small interface files." This is reasonable, but the task also includes 4 Celery tasks and 2 protocol files with full implementations (including the SmtpEmailSender at ~60 lines). The combined scope is substantial.

  **Suggestion:** Consider splitting Task 5 into two sub-tasks: (5a) Service protocols and stub implementations (report_generator.py, email_sender.py), and (5b) Celery tasks (tick.py, execute.py, deliver.py, tasks/__init__.py). The protocols are a dependency for the tasks, so 5a must complete first. This keeps each sub-task at 3 files and reduces the risk of compound errors.

- **TD section 3.6 / Pause response (line 646-648)** -- The pause response returns `"nextRunAt": null`, but the TD notes that the database retains the stored value. This is a presentation-layer decision (the API hides the stored value when paused). This should be documented as an explicit design choice in the response schema comments so that implementers do not accidentally serialize the database value.

  **Suggestion:** Add a comment in the `StatusUpdateResponse` Pydantic schema or in the route handler indicating that `next_run_at` is explicitly set to `None` in the response when the schedule is paused, regardless of the database value.

- **TD section 3.4 / PATCH recipients replacement semantics (line 591)** -- The PATCH endpoint uses replacement semantics for recipients: "if `recipients` is present in the body, the entire list is replaced." This is documented, but it is a departure from typical PATCH semantics (which usually apply partial updates). A client sending `{"recipients": ["alice@example.com"]}` will remove all other recipients, which could be surprising.

  **Suggestion:** Add a note to the frontend implementation (Task 7 description or section 6.3) that the schedule edit form must always send the complete recipient list when modifying recipients, not just additions. This prevents accidental data loss from a frontend that assumes additive semantics.

### Positive

- **State machine design (section 2.8)** -- The explicit transition maps (`VALID_SCHEDULE_TRANSITIONS`, `VALID_RUN_TRANSITIONS`) with a shared validation function and a custom exception type are clean and enforceable. This prevents invalid state transitions from silently corrupting data and makes the valid transitions easy to review and test.

- **Security integration (sections 5.3, 5.4, 7.2)** -- The TD systematically addresses every finding from the Security Engineer review: CRLF injection prevention in email addresses and subjects, input validation at Celery task entry points, query-level authorization with pre-filtering (404 instead of 403), and email volume monitoring. Each mitigation is traced back to the specific Security Engineer finding (e.g., "Security Engineer WARNING-1").

- **Context packages (section 11)** -- The work area context packages are a valuable addition. Each work area defines exactly which files to read, what the binding contracts are, key decisions in scope, and what is out of scope. This gives implementing agents clear boundaries and prevents context poisoning.

- **Separation of `ScheduleRun` and `RunRecipient`** -- Storing per-recipient delivery outcomes in a separate `RunRecipient` table with email-as-snapshot (not FK) is the right design. It correctly preserves delivery history when recipients are soft-deleted and enables the "partially delivered" status without ambiguity.

- **Contract consistency across layers** -- The Pydantic response schemas (section 4.2), JSON API contracts (section 3), and Zod frontend schemas (section 6.1) are consistent in field names and types, with the exception of the recipients input format noted above. The use of Pydantic aliases for camelCase serialization is correctly applied throughout.

---

## Requirements Traceability

| User Story | Endpoints/Components | Tasks | Coverage |
|-----------|---------------------|-------|----------|
| US-001 (Report type selection) | GET /v1/report-types, ScheduleCreate.report_type_id validation | Task 3 (schema), Task 4 (report types endpoint), Task 7 (UI form) | Complete |
| US-002 (Cadence configuration) | POST /v1/schedules cadence fields, compute_next_run | Task 3 (schema + next_run), Task 7 (CadenceSelector) | Complete |
| US-003 (Recipient management) | POST/PATCH /v1/schedules recipients field, RecipientCreate schema | Task 3 (CRUD), Task 7 (RecipientInput) | Complete |
| US-004 (Email delivery) | tick_task, execute_schedule, deliver_report, EmailSender/ReportGenerator protocols | Task 5 (all Celery tasks + protocols) | Complete |
| US-005 (View/edit schedules) | GET /v1/schedules (list), GET /v1/schedules/{id}, PATCH /v1/schedules/{id} | Task 3 (endpoints), Task 7 (list/detail/edit pages) | Complete |
| US-006 (Pause/resume) | POST /v1/schedules/{id}/pause, POST /v1/schedules/{id}/resume | Task 3 (endpoints), Task 7 (UI controls) | Complete |
| US-007 (Delete schedules) | DELETE /v1/schedules/{id} | Task 3 (endpoint), Task 7 (ConfirmDialog) | Complete |
| US-008 (Delivery status) | GET /v1/schedules/{id}/runs, RunDetailResponse with RunRecipientResponse | Task 4 (runs endpoint), Task 7 (DeliveryHistory, RunDetail) | Complete |
| US-009 (Timezone support) | Schedule.timezone, compute_next_run DST handling, TimezoneSelector | Task 1 (model), Task 3 (next_run), Task 7 (TimezoneSelector) | Complete |
| US-010 (Authorization) | get_current_user, get_owned_schedule, owner-scoped queries | Task 2 (auth), Task 3 (route guards) | Complete |

All 10 user stories map to concrete endpoints, models, and implementation tasks. No gaps identified.

---

## Task Sizing Assessment

| Task | Files | Concerns | Verdict |
|------|-------|----------|---------|
| Task 1 (DB schema) | 4 + Alembic setup | Single concern (data layer) | OK |
| Task 2 (API core) | 5 | Single concern (infrastructure) | OK |
| Task 3 (Schedule CRUD) | 5 | Multiple concerns: schemas, routes, state machine, next_run | Borderline -- see Warning about PATCH ambiguity |
| Task 4 (Runs + report types) | 3 | Single concern (read endpoints) | OK |
| Task 5 (Celery tasks) | 6 | Multiple concerns: 3 tasks + 2 protocols | Split recommended (see Suggestion above) |
| Task 6 (FE schemas) | 5 | Single concern (API integration layer) | OK |
| Task 7 (FE pages) | 5 primary + supporting | Multiple UI concerns, but all are presentation | Acceptable for frontend work |

---

## Internal Consistency Check

- **Data model vs. API contracts**: Consistent. Pydantic schemas use `from_attributes=True` for ORM model compatibility. All model fields have corresponding response schema fields.
- **API contracts vs. Zod schemas**: Consistent (with the recipients input format exception noted as a Suggestion).
- **Enum values**: Consistent across Python enums, Pydantic schemas, JSON contracts, and Zod schemas.
- **Pagination format**: Consistent between schedule list and run list. Both use `{data: [...], pagination: {nextCursor, hasMore}}`.
- **Error format**: All error responses use RFC 7807 structure with `type`, `title`, `status`, `detail`, and `instance` fields.
- **State machine transitions**: The transition maps in `state_machine.py` (section 2.8) match the state machine diagrams in the architecture (section 6.5) and the requirements (section 6).

---

**End of Review**
