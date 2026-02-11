# Security Engineer Memory

## Project: Report Scheduling Feature

### Known Security Patterns (Positive)

#### State Machine Enforcement Pattern (2026-02-10)
The Report Scheduling TD demonstrates exemplary state machine security design:
- Shared validation functions (`validate_schedule_transition`, `validate_run_transition`) used by both API and Celery tasks
- Fail-closed design — invalid transitions raise exceptions rather than silently succeeding
- Explicit transition maps make the allowed state graph auditable
- Terminal states have empty transition sets, preventing resurrection attacks

**Reuse this pattern:** When reviewing future features with state machines, verify that validation is centralized, fail-closed, and shared across all code paths that mutate state.

#### Defense-in-Depth Input Validation (2026-02-10)
Report Scheduling implements multi-layer validation for CRLF injection prevention:
1. API request schema layer (Pydantic validators)
2. Business logic layer (task sanitization)
3. Protocol boundary layer (EmailSender contract enforcement)

This pattern means that if one layer is bypassed, downstream layers still prevent exploitation. **Preferred pattern for all external input handling.**

#### Owner-Scoped Query Pattern (2026-02-10)
Authorization pattern: Pre-filter queries by `owner_id` before executing, return 404 for "not found" and "not owned" cases (do not distinguish). This prevents IDOR vulnerabilities and information disclosure.

```python
# Good pattern
stmt = select(Schedule).where(Schedule.id == schedule_id)
if not user.is_admin:
    stmt = stmt.where(Schedule.owner_id == user.id)
schedule = await session.scalar(stmt)
if not schedule:
    raise HTTPException(status_code=404)
```

**Apply this pattern:** All resource access endpoints should follow this convention.

### Known Risks and Mitigations

#### Redis as a Trust Boundary
Redis is used as the Celery broker for task dispatch. Compromise of Redis could allow task injection attacks. Mitigations:
- All Celery tasks validate input arguments at task entry (not just at API boundary)
- Redis must be network-isolated and password-protected in production (DevOps responsibility)
- Use `task_acks_late=True` and `task_reject_on_worker_lost=True` for crash safety

**Review pattern:** When reviewing Celery task code, always verify that task functions validate their arguments before acting on them.

#### Email Header Injection (SMTP)
Email systems are vulnerable to CRLF injection in recipient addresses and subject lines. Require CRLF rejection at:
1. API request validation
2. Task-level sanitization
3. Protocol boundary (EmailSender contract)

**Test requirement:** Always include malicious input tests for email-related features (embedded `\r\n`, `%0d%0a` URL-encoded).

### Architectural Decisions

#### MVP Security Posture (2026-02-10)
Project maturity is MVP. Security expectations:
- Authentication and authorization are required
- Input validation at all boundaries
- Basic rate limiting
- No automatic retry logic for failed deliveries (avoid amplification attacks)
- Monitoring via structured logs, not hard enforcement (500 emails/day/user threshold is a warning, not a block)

Full OWASP audit, penetration testing, and security monitoring are expected at Production maturity.

### Review Checklist for Technical Designs

When reviewing a TD, verify:
- [ ] All external inputs validated at API boundary (Pydantic, zod, etc.)
- [ ] Authorization checks use pre-filtered queries (owner-scoped pattern)
- [ ] State machines enforce transitions with fail-closed validation
- [ ] Secrets loaded from environment variables, not hardcoded
- [ ] Error responses do not expose internal details (stack traces, DB constraints)
- [ ] Protocol contracts explicitly document security requirements (e.g., no CRLF)
- [ ] Celery tasks validate input arguments at task entry
- [ ] Test coverage includes security negative cases (injection attempts, IDOR)

### Stakeholder Preferences

- **Security rigor at MVP:** Stakeholder expects foundational security (auth, input validation, rate limiting) at MVP stage, but defers operational concerns (Redis hardening, secrets management) to DevOps Engineer. This aligns with the maturity expectations table in CLAUDE.md.
- **Defense-in-depth over single-point validation:** Stakeholder values multi-layer validation (as evidenced by CRLF prevention at 3 layers). When reviewing designs, highlight cases where validation is only at one layer.
