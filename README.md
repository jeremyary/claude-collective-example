# Claude Collective Scaffold — Workshop Example

This repository demonstrates the **Spec-Driven Development (SDD) process** provided by the [Claude Collective scaffold](https://github.com/jary-k/claude-collective). It walks a fictional SaaS application — a report scheduling platform — through every planning phase, from initial requirements gathering through to a ready-for-implementation work breakdown.

**No application code has been written.** The value here is the artifact trail: the plans, reviews, architecture decisions, and work breakdown that the scaffold's multi-agent system produces *before* a single line of implementation begins.

---

## What This Example Shows

The scaffold's SDD process moves through these phases, each driven by a specialist agent:

| Phase | Agent | Artifact |
|-------|-------|----------|
| 1. Requirements gathering | Requirements Analyst | [`plans/requirements.md`](plans/requirements.md) |
| 2. Requirements review | Product Manager, Architect | [`plans/reviews/requirements-review-*.md`](plans/reviews/) |
| 3. Product plan | Product Manager | [`plans/product-plan.md`](plans/product-plan.md) |
| 4. Product plan review | Architect, API Designer, Security Engineer | [`plans/reviews/product-plan-review-*.md`](plans/reviews/) |
| 5. Architecture design | Architect | [`plans/architecture.md`](plans/architecture.md) |
| 6. Architecture review | API Designer, Security Engineer | [`plans/reviews/architecture-review-*.md`](plans/reviews/) |
| 7. Architecture decision records | Architect | [`plans/adr/`](plans/adr/) |
| 8. Technical design (Phase 1) | Tech Lead | [`plans/technical-design-phase-1.md`](plans/technical-design-phase-1.md) |
| 9. Technical design review | Code Reviewer, Security Engineer | [`plans/reviews/technical-design-phase-1-review-*.md`](plans/reviews/) |
| 10. Work breakdown | Project Manager | [`plans/work-breakdown-phase-1.md`](plans/work-breakdown-phase-1.md) |

Each review phase includes a **validation summary** (`*-validation.md`) that aggregates findings and records the pass/fail decision before the process advances.

## Key Artifacts

### Plans

- **[requirements.md](plans/requirements.md)** — User stories, acceptance criteria, and non-functional requirements
- **[product-plan.md](plans/product-plan.md)** — PRD with personas, MoSCoW-prioritized features, user flows, and phasing
- **[architecture.md](plans/architecture.md)** — System architecture, component design, data flow, and technology choices
- **[technical-design-phase-1.md](plans/technical-design-phase-1.md)** — Interface contracts, file paths, data shapes, and machine-verifiable exit conditions for every task
- **[work-breakdown-phase-1.md](plans/work-breakdown-phase-1.md)** — 5 work units, 11 tasks, sized to scaffold constraints (3-5 files, single concern, ~1 hour each)

### Architecture Decision Records

- [ADR-0001: Scheduling Engine Approach](plans/adr/0001-scheduling-engine-approach.md)
- [ADR-0002: Report Generation Boundary](plans/adr/0002-report-generation-boundary.md)
- [ADR-0003: Email Delivery Abstraction](plans/adr/0003-email-delivery-abstraction.md)
- [ADR-0004: Timezone Strategy](plans/adr/0004-timezone-strategy.md)

### Reviews

The [`plans/reviews/`](plans/reviews/) directory contains 11 review artifacts spanning requirements, product plan, architecture, and technical design — each written by a different specialist agent. Reviews enforce the scaffold's anti-rubber-stamping governance: every review must include at least one finding.

## The Scaffold Itself

The scaffold configuration lives in `.claude/` and `CLAUDE.md`. Key pieces:

| File | Purpose |
|------|---------|
| [`CLAUDE.md`](CLAUDE.md) | Project identity, tech stack decisions, goals, constraints, and agent quick-reference |
| [`.claude/CLAUDE.md`](.claude/CLAUDE.md) | Agent routing matrix, orchestration patterns, and cost tier assignments |
| [`.claude/rules/`](.claude/rules/) | 15 convention rules enforced across all agents (code style, testing, security, AI compliance, etc.) |
| [`.claude/settings.json`](.claude/settings.json) | Tool permissions — safe commands pre-approved, dangerous operations blocked |
| [`START_HERE.md`](START_HERE.md) | Setup guide for customizing the scaffold on a new project |

## Getting the Scaffold

This repo is a completed example. To use the scaffold on your own project:

1. Go to [claude-collective](https://github.com/jary-k/claude-collective)
2. Click **"Use this template"** to create a new repository
3. Run `/setup` in Claude Code for an interactive walkthrough

## License

This project is licensed under the [Apache License 2.0](LICENSE).
