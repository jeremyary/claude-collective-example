# Project Memory

## Project Identity
- **Name:** claude-collective-example
- **Domain:** SaaS report scheduling platform
- **Maturity:** MVP
- **Stack:** Python 3.12+/FastAPI + React 19/TypeScript + PostgreSQL 16 + Celery/Redis
- **Constraint:** All dependencies must be open-source (Fedora-allowed licenses)

## Setup Decisions (2026-02-10)
- Removed agents: SRE Engineer, Performance Engineer, Technical Writer (MVP — premature)
- Kept 14 active agents, expanded hybrid model tier (Opus for planning/review, Sonnet for implementation)
- Removed `python-style.md` in favor of merged `code-style.md` (full-stack project)
- Red Hat org-specific domains removed from `settings.local.json`; framework doc domains added
- Stakeholder prefers concise communication, most popular open-source defaults, quick delegation

## Feature Context
- First feature: scheduled report delivery (pick report type, set cadence, deliver to team)
- Product-driven requirement from stakeholder
