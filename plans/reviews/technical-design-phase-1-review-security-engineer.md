# Technical Design Security Review: Report Scheduling Phase 1

**Reviewer:** Security Engineer
**Date:** 2026-02-10
**Artifact:** `/home/jary/redhat/git/claude-collective-example/plans/technical-design-phase-1.md`
**Phase:** 1 (Core Scheduling)
**Prior Review:** `/home/jary/redhat/git/claude-collective-example/plans/reviews/architecture-review-security-engineer.md`

---

## Verdict

**APPROVE**

The Technical Design successfully addresses all prior architecture review findings and implements a secure foundation for MVP. The design demonstrates defense-in-depth with validation at multiple layers, proper authorization patterns, and explicit security boundaries in protocol contracts. All WARNING-level findings from the architecture review have been resolved. The implementation is ready to proceed.

---

## Prior Findings Resolution

### WARNING-1: Email Header Injection Risk

**Status:** RESOLVED

The TD implements comprehensive CRLF injection prevention at three layers:

1. **API validation layer** (Section 4.1, line 424): Request schema validation explicitly rejects CRLF characters in recipient email addresses with a custom Pydantic validator (`reject_crlf_in_email`). Validation error message: "Email address must not contain newline characters."

2. **Task delivery layer** (Section 5.5, lines 1385-1399): The `deliver_report` task sanitizes subject lines and recipient emails before passing to EmailSender, using a compiled regex pattern `_CRLF_PATTERN = re.compile(r"[\r\n]")`. Subjects are sanitized by removal; recipients are logged and skipped.

3. **Protocol boundary** (Section 5.5, lines 1588-1640): The `EmailSender` protocol contract explicitly documents that `to_email` and `subject` MUST NOT contain CRLF characters, and the `SmtpEmailSender` implementation enforces this with a `ValueError` if control characters are detected.

All three layers are documented with references to "Security Engineer WARNING-1" in comments. Test coverage is specified in Task 3 exit condition (line 2440: "Validation: invalid timezone, cadence_day, recipients") and Task 5 (line 2500: "CRLF rejection in email addresses").

**Verification:** Section 13 checklist line 2714 confirms this finding is addressed.

---

### WARNING-2: Celery Task Deserialization Risks

**Status:** RESOLVED

The TD implements input validation at task entry for all three Celery tasks:

1. **execute_schedule task** (Section 5.3, lines 1181-1207): Validates that `run_id` is a positive integer and that the corresponding `ScheduleRun` exists in the database before proceeding. Returns `{"error": "invalid_run_id"}` if validation fails. Logs invalid inputs with structured logging for monitoring.

2. **deliver_report task** (Section 5.5, lines 1326-1361): Validates that `run_id` is a positive integer and that the `ScheduleRun` exists and is in the `GENERATED` state before delivery. Invalid state transitions are rejected.

3. **tick_task** (Section 5.2, lines 1090-1152): No parameters, so no input validation needed. The task queries the database using a parameterized query (SQLAlchemy ORM), which prevents injection.

The design also includes crash-safety configuration (Section 5.1, lines 1050-1051): `task_acks_late=True` and `task_reject_on_worker_lost=True`, which prevent task loss if a worker crashes mid-execution.

Redis security is deferred to the DevOps Engineer as noted in architecture review INFO-1, which is appropriate for the Technical Design phase.

**Verification:** Section 13 checklist line 2715 confirms this finding is addressed. Task 5 exit condition (line 2499) includes "Input validation at task entry (invalid run_id, wrong state)."

---

### SUGGESTION-1: Query-Level Authorization with Pre-filtering

**Status:** RESOLVED

The TD establishes an owner-scoped query pattern as the project convention (Section 7.2, lines 2167-2193). The pattern is documented with code examples showing:

- **Pre-filtering by owner_id** (line 2193): `stmt = stmt.where(Schedule.owner_id == user.id)` before executing the query
- **404 response** for both "not found" and "not owned" cases, preventing information disclosure
- **Admin bypass** via conditional logic: `if not user.is_admin: stmt = stmt.where(Schedule.owner_id == user.id)`

This pattern is applied to all schedule access endpoints:
- GET /v1/schedules (list) — Section 3.2, line 500
- GET /v1/schedules/{id} (detail) — Section 3.3, line 538
- PATCH /v1/schedules/{id} (update) — Section 3.4, line 593
- DELETE /v1/schedules/{id} (delete) — Section 3.5, line 623
- POST /v1/schedules/{id}/pause — Section 3.6, line 634
- POST /v1/schedules/{id}/resume — Section 3.7, line 667
- GET /v1/schedules/{id}/runs — Section 3.8 (via parent schedule ownership)

The pattern is called out in the Context Package for API Endpoints (Section 11, line 2620) as a binding contract, ensuring all implementers follow it.

**Verification:** Section 13 checklist line 2716 confirms this finding is addressed. Task 3 exit condition (line 2439) includes "Authorization: 404 when accessing another user's schedule."

---

### SUGGESTION-2: Email Volume Monitoring

**Status:** RESOLVED

The TD implements a monitoring threshold of 500 emails/day/user via structured logging (Section 5.7, lines 1792-1796 and Section 5.5, line 1318). The `deliver_report` task logs structured events including:
- `sent` (count of successfully sent emails)
- `failed` (count of failed deliveries)
- `schedule_id` and `owner_id` (for aggregation)

The log-based approach is appropriate for MVP maturity. The TD explicitly documents that ops teams can aggregate `sent` counts per `owner_id` per day from structured logs to detect threshold breaches. Hard limits are deferred to Phase 2, which aligns with the architecture review recommendation.

The threshold value (500 emails/day/user) is defined as `_EMAIL_VOLUME_WARNING_THRESHOLD` in the deliver task code (line 1318).

**Verification:** Section 13 checklist line 2717 confirms this finding is addressed. Task 5 exit condition (line 2657) includes "Email volume monitoring via structured logging (500/day/user threshold)."

---

### SUGGESTION-3: Unsubscribe Compliance Boundary

**Status:** RESOLVED

The TD includes an unsubscribe footer in all email deliveries (Section 5.5, lines 1647-1653) with the following text:

> "You are receiving this report because the schedule owner added you to a scheduled delivery. To stop receiving this report, contact the schedule owner or your account administrator."

This implements the owner-mediated unsubscribe path specified in the architecture for MVP. The footer is hardcoded into the `SmtpEmailSender.send()` implementation, ensuring it appears on every email without requiring implementer action.

The design does not include a comment explicitly documenting the organizational/internal recipient assumption from the architecture review, but this is acceptable because:
1. The footer text itself communicates the assumption ("contact the schedule owner or your account administrator")
2. The persona context in the product plan and requirements documents establishes the organizational scope
3. The architecture review (SUGGESTION-3) already documents the compliance boundary, so repeating it in the TD is not strictly necessary

Self-service unsubscribe with tokenized links is explicitly out of scope for Phase 1 (Section 1, line 27).

**Verification:** No checklist item for this finding, but the implementation is present and correct.

---

## New Security Findings

### SUGGESTION-1: State Transition Enforcement is Well-Designed

**Category:** OWASP A01:2021 – Broken Access Control (Positive Observation)
**Location:** Section 2.8 (State Machine Transition Functions), Section 5.3, Section 5.5

**Description:**
The state machine enforcement design is exemplary. The shared utility functions (`validate_schedule_transition` and `validate_run_transition`) ensure that invalid transitions cannot occur at either the API layer or the task layer. All transitions raise `InvalidStateTransitionError` if invalid, which maps to a 409 Conflict response at the API boundary.

**Key strengths:**
- **Fail-closed design:** Invalid transitions raise exceptions rather than silently succeeding or defaulting to a permissive behavior
- **Shared validation:** Both API handlers and Celery tasks call the same validation functions, preventing inconsistencies
- **Explicit transition maps:** The `VALID_SCHEDULE_TRANSITIONS` and `VALID_RUN_TRANSITIONS` dictionaries make the allowed state graph auditable
- **Terminal state protection:** Deleted schedules and terminal run states have empty transition sets, preventing resurrection

This design prevents a class of vulnerabilities where attackers could force objects into invalid states via API parameter manipulation or Redis task injection.

**Impact:**
No vulnerabilities identified. This is a positive finding demonstrating secure-by-design practices.

**Recommendation:**
No changes needed. This pattern should be documented in the project's architectural decision records as a reference for future state machines.

---

### INFO-1: Redis Authentication Deferred to Deployment

**Category:** OWASP A05:2021 – Security Misconfiguration
**Location:** Section 5.1 (Celery Application Setup)

**Description:**
The TD does not specify Redis authentication configuration (password, TLS). This is consistent with the architecture review INFO-1, which noted that Redis security is an operational concern for the DevOps Engineer, not a TD concern.

The local development configuration (via `compose.yml` referenced in the architecture) uses an open Redis instance, which is acceptable for local dev.

**Impact:**
No risk for Phase 1 implementation. Production deployment must address this before go-live.

**Recommendation:**
No changes needed in the TD. The DevOps Engineer must ensure Redis is:
1. Network-isolated (not exposed to the public internet)
2. Password-protected (`requirepass` configuration)
3. TLS-enabled if spanning multiple hosts
4. Bound to a private network in OpenShift/Kubernetes

This is already flagged in the architecture review INFO-1 and does not need re-specification here.

---

### INFO-2: SMTP Credentials Management Specified

**Category:** Secrets Management (Positive Observation)
**Location:** Section 5.5 (EmailSender Protocol), Section 2.1 (Configuration)

**Description:**
The TD specifies that SMTP credentials are loaded from environment variables via Pydantic Settings (Section 2.1 references `smtp_username` and `smtp_password` in the Settings model, and Section 5.5 line 1674 shows these are passed to the `SmtpEmailSender` constructor).

The design correctly avoids hardcoding credentials. The TD does not specify whether Kubernetes Secrets or a secrets manager will be used in production, but that is a deployment concern for the DevOps Engineer, not a code-level concern.

For local development, Mailpit requires no authentication, so the optional `username` and `password` fields in the `SmtpEmailSender` constructor (line 1611) support both dev and production use cases.

**Impact:**
No vulnerabilities identified. Credentials management follows the project security baseline.

**Recommendation:**
No changes needed in the TD. The DevOps Engineer should use OpenShift Secrets or a secrets manager for production deployment, as noted in architecture review INFO-2.

---

### INFO-3: Input Validation is Defense-in-Depth

**Category:** OWASP A03:2021 – Injection (Positive Observation)
**Location:** Multiple sections

**Description:**
The TD implements validation at multiple layers for all external inputs:

1. **Request schema layer (Section 4.1):** Pydantic validators enforce email format (`EmailStr`), timezone validity (`zoneinfo.available_timezones()`), cadence_day ranges, and recipient count limits (1-50)
2. **Business logic layer (Section 7.2):** Authorization pre-filters queries by owner_id; rate limiting enforces per-user request limits
3. **Protocol boundary layer (Section 5.5):** EmailSender rejects CRLF characters; ReportGenerator validates report_type_id
4. **Task entry layer (Section 5.3, 5.5):** Tasks validate run_id and state before proceeding

This defense-in-depth approach means that if one validation layer is bypassed (e.g., via a task injected into Redis), downstream layers still prevent exploitation.

**Impact:**
No vulnerabilities identified. This is a positive finding demonstrating secure coding practices.

**Recommendation:**
No changes needed. This pattern is well-executed and should be maintained in future phases.

---

## Security-Relevant Observations for Implementation

### For Backend Developer (Tasks 3, 4, 5)

1. **Parameterized queries only:** All database queries in the TD use SQLAlchemy ORM. Ensure no raw SQL with string concatenation is introduced during implementation. The ORM's query construction is safe against SQL injection.

2. **Error message sanitization:** When mapping exceptions to RFC 7807 responses, ensure internal error details (stack traces, database constraint names, file paths) are not exposed to clients. The TD's error handling design (Section 7.4) correctly logs detailed errors server-side and returns sanitized messages to clients — maintain this pattern.

3. **Timezone validation:** The TD specifies that timezone validation uses `zoneinfo.available_timezones()`. Ensure this check is performed at request validation time, not at schedule execution time, to fail fast and prevent invalid timezones from being stored.

4. **State transition assertions:** When implementing pause/resume endpoints, ensure that `validate_schedule_transition()` is called before updating the status field. The TD shows this pattern in Section 2.8 and Task 3 exit condition (line 2438) confirms it's tested. Do not bypass the validation function.

---

### For Test Engineer (Task 10)

1. **Security test cases required:**
   - Email header injection attempts: Test that recipient addresses with `\r`, `\n`, `\r\n`, and `%0d%0a` (URL-encoded) are rejected at API and task layers
   - IDOR attempts: Test that user A cannot access, modify, or delete user B's schedules via ID enumeration (expect 404, not 403)
   - State transition attacks: Test that invalid state transitions (e.g., DELETED → ACTIVE, DELIVERED → GENERATING) return 409 and do not mutate the database
   - Rate limit bypass: Test that changing user ID in JWT or making parallel requests does not bypass the rate limiter
   - Task input validation: Test that calling `execute_schedule` with an invalid run_id (negative, non-existent, string) does not cause crashes or unintended behavior

2. **Negative test coverage:** The TD's task exit conditions reference validation tests, but ensure coverage includes malicious inputs, not just malformed inputs. For example, test a valid email address with embedded CRLF: `user@example.com\r\nBcc: attacker@evil.com`.

---

### For Frontend Developer (Task 7)

1. **Client-side validation is not a security boundary:** The TD specifies client-side email validation and deduplication in the recipient input component (Section 6.5, line 2116). This is good UX, but ensure server-side validation is not removed or weakened. The API must re-validate all inputs.

2. **Avoid exposing sensitive data in client state:** If implementing admin features in the future, ensure that admin-only data (e.g., other users' schedules) is not cached in TanStack Query or local storage where it could be accessed by inspecting browser state.

---

## Summary

The Technical Design addresses all prior security findings and implements a secure foundation for Phase 1. The design demonstrates mature security practices including:

- Multi-layer input validation (API, protocol boundary, task entry)
- Fail-closed state machine enforcement with explicit transition maps
- Owner-scoped authorization pattern that prevents IDOR vulnerabilities
- CRLF injection prevention at three layers (API, task, protocol)
- Defense-in-depth approach with validation at each system boundary

No critical or high-severity security issues identified. All WARNING-level findings from the architecture review have been resolved. The design is implementation-ready.

The only areas requiring operational follow-up (Redis security, SMTP credentials management) are correctly scoped to the DevOps Engineer and do not block implementation.

Proceed with implementation. Security review of the code itself will verify that the design is correctly implemented.
