# Code Reviewer Memory

## Project Context

- **Project**: Report Scheduling feature for an internal reporting platform
- **Maturity**: Proof-of-concept (per CLAUDE.md template, not yet filled in)
- **Tech Stack**: FastAPI (Python), React 19 + TypeScript, PostgreSQL 16, Celery + Redis, SQLAlchemy 2.0
- **Architecture**: Monorepo with packages/db, packages/api, packages/ui

## Review History

### 2026-02-10: Technical Design Phase 1 Review
- **Verdict**: APPROVE WITH SUGGESTIONS
- **Key findings**:
  - database.py imports from api package's config (circular dependency)
  - Tick task commit-then-dispatch gap (crash between commit and delay leaves orphaned pending runs)
  - PATCH partial update ambiguity with Pydantic optional fields (null vs absent)
  - Redis client created per request in rate limiter (no connection pooling)
  - Recipients input format mismatch: JSON contract says string[], Pydantic expects [{email: str}], frontend interface says string[]
  - Redundant standalone indexes on columns already covered by composite indexes
  - assert used for validation (removed by -O flag)

## Recurring Patterns to Watch

- **Cross-package dependency direction**: db package must not import from api package
- **Pydantic PATCH semantics**: Always check for null vs absent distinction using model_fields_set
- **Connection pooling**: Redis/DB clients should be initialized once, not per-request
- **assert vs raise**: Production validation must use explicit raise, not assert
