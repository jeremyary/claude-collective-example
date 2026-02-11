# claude-collective-example

> **SaaS platform that lets users schedule reports, set delivery cadence, and automatically distribute them to their teams.**

## Project Context

<!-- Fill in these values when starting a new project. Every agent reads this file. -->

| Attribute | Value |
|-----------|-------|
| Maturity | `mvp` |
| Domain | SaaS / business intelligence / reporting |
| Primary Users | End customers (teams receiving scheduled reports) |
| Compliance | None |

### Maturity Expectations

| Concern | MVP |
|---------|-----|
| Testing | Happy path + critical edges |
| Error handling | Basic error responses |
| Security | Auth + input validation |
| Documentation | README + API basics |
| Performance | Profile obvious bottlenecks |
| Code review | Light review |
| Infrastructure | Basic CI + single deploy target |

## Goals

1. Enable users to schedule reports with configurable cadence (daily, weekly, monthly, custom cron)
2. Support multiple report types with team-based delivery (email, in-app notifications)
3. Provide a self-service UI for managing report schedules, recipients, and delivery preferences

## Non-Goals

- None identified yet

## Constraints

- All dependencies must be open-source with Fedora-allowed licenses
- Modern browsers only (last 2 versions)

## Stakeholder Preferences

<!-- Record stakeholder decision patterns and preferences so agents can anticipate rather than re-ask. -->
<!-- These accumulate over time as agents learn from interactions. -->

| Preference Area | Observed Pattern |
|-----------------|-----------------|
| Review thoroughness | <!-- To be observed --> |
| Risk tolerance | Prefers most popular, proven open-source options over cutting-edge |
| Scope decisions | <!-- To be observed --> |
| Communication style | Concise, delegates details quickly, prefers defaults over lengthy discussion |
| Technology biases | Prefers most popular open-source stack for each layer |
| Testing expectations | <!-- To be observed --> |
| Documentation level | <!-- To be observed --> |

## Red Hat AI Compliance

All AI-assisted work in this project must comply with Red Hat's internal AI policies. The full machine-enforceable rules are in `.claude/rules/ai-compliance.md`. Summary of obligations:

1. **Human-in-the-Loop** — All AI-generated code must be reviewed, tested, and validated by a human before merge
2. **Sensitive Data Prohibition** — Never input confidential data, PII, credentials, or internal hostnames into AI tools
3. **AI Marking** — Include `// This project was developed with assistance from AI tools.` (or language equivalent) at the top of AI-assisted files, and use `Assisted-by:` / `Generated-by:` commit trailers
4. **Copyright & Licensing** — Verify generated code doesn't reproduce copyrighted implementations; all dependencies must use [Fedora Allowed Licenses](https://docs.fedoraproject.org/en-US/legal/allowed-licenses/)
5. **Upstream Contributions** — Check upstream project AI policies before contributing AI-generated code; default to disclosure
6. **Security Review** — Treat AI-generated code with the same or higher scrutiny as human-written code, especially for auth, crypto, and input handling

See `docs/ai-compliance-checklist.md` for the developer quick-reference checklist.

## Key Decisions

<!-- Record major technology choices here so all agents stay aligned. -->
<!-- Move detailed trade-off analysis to docs/adr/ as the project matures. -->

- **Language:** TypeScript 5.x (frontend), Python 3.12+ (backend)
- **Runtime:** Node.js 22 LTS (frontend), Python 3.12+ (backend)
- **Backend:** FastAPI
- **Frontend:** React 19 + Vite
- **Database:** PostgreSQL 16
- **ORM:** SQLAlchemy 2.0 (async)
- **Task Queue:** Celery + Redis
- **Testing:** Vitest + React Testing Library (frontend), pytest (backend)
- **Package Manager:** pnpm (frontend), uv (backend)

---

## Agent System

This project uses a multi-agent system with specialized Claude Code agents. The main session handles routing and orchestration using the routing matrix in `.claude/CLAUDE.md`. Each agent has a defined role, model tier, and tool set optimized for its task.

### Quick Reference — "I need to..."

| Need | Agent | Command |
|------|-------|---------|
| Plan a feature or large task | **Main session** | Describe what you need; routing matrix and workflow-patterns skill guide orchestration |
| Shape a product idea into a plan | **Product Manager** | `@product-manager` |
| Gather requirements | **Requirements Analyst** | `@requirements-analyst` |
| Design system architecture | **Architect** | `@architect` |
| Design feature-level implementation approach | **Tech Lead** | `@tech-lead` |
| Break work into epics & stories | **Project Manager** | `@project-manager` |
| Write backend/API code | **Backend Developer** | `@backend-developer` |
| Build UI components | **Frontend Developer** | `@frontend-developer` |
| Design database schema | **Database Engineer** | `@database-engineer` |
| Design API contracts | **API Designer** | `@api-designer` |
| Review code quality | **Code Reviewer** | `@code-reviewer` |
| Write or fix tests | **Test Engineer** | `@test-engineer` |
| Audit security | **Security Engineer** | `@security-engineer` |
| Set up CI/CD or infra | **DevOps Engineer** | `@devops-engineer` |
| Debug a problem | **Debug Specialist** | `@debug-specialist` |

### How It Works

1. **Describe what you need** — for non-trivial tasks, the main session uses the routing matrix and workflow-patterns skill to select agents and sequence work.
2. **Use a specialist directly** when you know exactly which agent you need (e.g., `@backend-developer`).
3. **Rules files** (imported below) enforce project conventions automatically across all agents.
4. **Spec-Driven Development** is the default for non-trivial features — plan review before code review, machine-verifiable exit conditions, and anti-rubber-stamping governance.
5. **Skills** provide workflow templates and project convention references.

## Project Conventions

@.claude/rules/ai-compliance.md
@.claude/rules/code-style.md
@.claude/rules/git-workflow.md
@.claude/rules/testing.md
@.claude/rules/security.md
@.claude/rules/error-handling.md
@.claude/rules/observability.md
@.claude/rules/api-conventions.md
@.claude/rules/agent-workflow.md
@.claude/rules/review-governance.md
@.claude/rules/architecture.md
@.claude/rules/api-development.md
@.claude/rules/ui-development.md
@.claude/rules/database-development.md

## Project Commands

### Build
```bash
make build
```

### Test
```bash
make test
```

### Lint
```bash
make lint
```

### Type Check
```bash
pnpm type-check
```

### Database
```bash
make db-start
make db-upgrade
```

### Dev Server
```bash
make dev
```
