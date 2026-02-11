# Product Manager Memory

## Project Context
- **Project:** SaaS report scheduling platform
- **Maturity:** MVP
- **Domain:** Business intelligence / reporting
- **Stakeholder style:** Concise, delegates quickly, prefers proven open-source, prefers defaults over lengthy discussion

## Decisions Made
- Report Scheduling product plan written to `plans/product-plan.md`
- Three personas defined: Report Owner (Maya), Report Recipient (Sam), Account Admin (Jordan)
- Three-phase roadmap: Core Scheduling -> Reliability/Flexibility -> Administration/Convenience
- P0 scope: report type selection, cadence (daily/weekly/monthly), email delivery, schedule CRUD, delivery status, timezone, auth

## Open Questions (unresolved)
- What report types exist today and do they all support automated generation?
- Email sending constraints (volume, sender domain)?
- Orphaned schedule policy when owner leaves?
- Report generation duration variation across report types?
- Recipient limits per schedule/account?
- Monthly schedules on nonexistent days (e.g., 31st of Feb)?

## Validation Status
- Product plan validated 2026-02-10 after Phase 2 reviews (Architect, API Designer, Security Engineer)
- All three reviews: APPROVE WITH SUGGESTIONS
- Stakeholder deferred all suggestions to downstream phases
- Validation verdict: VALIDATED -- proceed to Architecture
- Borderline deferred items: orphan schedule policy (Architect may escalate), unsubscribe compliance (if external recipients needed)
- Validation written to `plans/reviews/product-plan-validation.md`

## Scope Discipline Reminders
- Never embed technology names in feature descriptions -- record stakeholder mandates in Constraints section
- NFRs must be user-facing outcomes, not implementation targets (e.g., "feels responsive" not "< 200ms")
- No epic/story breakout or dependency maps -- that is the Project Manager's job
- Phasing describes capability milestones, not deliverables or agent assignments

## Lessons Learned
- Persona data (e.g., "3-20 recipients") serves as implicit sizing guidance for the Architect even when open questions are unresolved
- When reviewers suggest splitting a feature (e.g., generation vs. delivery), check whether it is product scope or architecture decomposition -- most splits are architecture
- "Borderline deferred" items should have explicit escalation paths documented in the validation so downstream agents know when to come back
