# ADR-0003: Email Delivery Abstraction

## Status
Proposed

## Context
Report delivery via email is a P0 feature. For MVP, we need to send emails but do not yet have a production email provider selected. The system must support swapping email backends without changing business logic. The Security Engineer review flagged email as an attack surface requiring sender authentication (SPF/DKIM) and header encoding -- these are provider-level concerns that should not leak into the scheduling logic.

## Options Considered

### Option 1: Direct SMTP via Python stdlib (smtplib)
Use Python's built-in `smtplib` to send emails directly to an SMTP server.

- **Pros:** No external dependency. Works with any SMTP server. Simple for local development (use MailHog/Mailpit as a local SMTP sink).
- **Cons:** Low-level. Must handle connection pooling, retries, TLS, and encoding manually. No built-in templating. Production SMTP setup (SPF/DKIM) is operational work outside the application.

### Option 2: Email Service SDK (SendGrid, SES, etc.)
Use a specific email provider's SDK.

- **Pros:** Provider handles deliverability (SPF/DKIM, bounce handling). Rich APIs for tracking delivery status.
- **Cons:** Vendor lock-in. SDK dependency. License must be Fedora-allowed. Overkill for MVP when we do not have a provider selected.

### Option 3: Protocol Interface with SMTP Default
Define an `EmailSender` protocol. MVP implementation uses SMTP (pointed at a local mail sink in development). Production swaps in a provider-specific implementation. The protocol defines: recipients, subject, body (HTML), attachments.

- **Pros:** Same pattern as the report generator -- consistent abstraction strategy. Local dev uses MailHog/Mailpit (no external dependency). Production can use any provider without changing business logic. Testable with a mock sender.
- **Cons:** Must define the protocol shape. SMTP implementation still requires basic connection management.

## Decision
**Option 3: Protocol Interface with SMTP Default.** This mirrors the report generation boundary (ADR-0002) and keeps the architecture consistent. The `EmailSender` protocol is defined in `packages/api/src/services/`. The MVP SMTP implementation sends to a local mail sink (Mailpit in `compose.yml`). The protocol contract:
- Input: list of recipient emails, subject, HTML body, optional attachments (bytes + filename + content type)
- Output: per-recipient delivery result (sent/failed + error message if failed)
- The protocol handles individual recipient delivery -- the calling code fans out per-recipient so that one bounce does not block others.

## Consequences

### Positive
- Local development works without external email accounts -- Mailpit provides a web UI to inspect sent emails
- Production email provider is a configuration choice, not a code change
- Fan-out per recipient means one invalid address does not prevent delivery to others
- Consistent abstraction pattern across external integrations (generation + delivery)

### Negative
- SMTP implementation must handle basic TLS, connection reuse, and encoding
- Must define the email protocol shape before knowing the production provider's capabilities

### Neutral
- Mailpit is added to `compose.yml` for local development (lightweight, single container)
- The email protocol naturally supports adding delivery channels later (in-app notifications in Phase 2)
