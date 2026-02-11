# ADR-0002: Report Generation Boundary

## Status
Proposed

## Context
The product plan states "assumes report types already exist in the system" (Won't Have: Report builder or designer). The scheduling system must trigger report generation and receive the output (or a failure signal) to deliver via email. For the MVP, no real report generation system exists yet -- we need a clean integration boundary that can start with a stub and be replaced with a real implementation later.

The Architect review (W1) identified that generation and delivery are distinct concerns with different failure modes and retry semantics. This ADR defines the boundary between the scheduling system and the report generation system.

## Options Considered

### Option 1: Direct Function Call
The Celery execution task directly calls a report generation function. The function signature serves as the contract.

- **Pros:** Simple. No additional infrastructure. Fast for MVP.
- **Cons:** Tight coupling. If report generation is slow or blocking, it ties up the Celery worker. Hard to replace the implementation without changing the calling code. No clear contract boundary.

### Option 2: Protocol Interface (ABC/Protocol Class)
Define a Python Protocol (PEP 544) that specifies the report generation contract. The scheduling system calls through the protocol. MVP uses a stub implementation; production uses a real one. The protocol is defined in the `api` package and the implementation is injected at startup.

- **Pros:** Clean boundary. Implementation is swappable without changing the scheduling code. The protocol documents the contract explicitly. Testable -- tests inject a mock implementation. Supports async natively.
- **Cons:** Slightly more abstraction than a direct call. Must define the protocol shape upfront.

### Option 3: HTTP/Message Queue Integration
The scheduling system sends a message to an external report generation service via HTTP or a message queue. Fully decoupled.

- **Pros:** True service boundary. Report generation can be in a different language/process.
- **Cons:** Massive overkill for MVP. Adds infrastructure (another service, message format, error handling for network calls). The product plan does not indicate report generation is a separate service.

## Decision
**Option 2: Protocol Interface.** The scheduling system calls through a `ReportGenerator` protocol defined in `packages/api`. The protocol specifies:
- Input: report type identifier, schedule configuration (parameters, timezone)
- Output: generated report content (bytes + content type) or a structured error

MVP ships with a `StubReportGenerator` that returns a placeholder PDF/HTML. When the real report generation system is ready, a new implementation of the protocol is registered at startup. No scheduling code changes.

## Consequences

### Positive
- Clean, documented boundary between scheduling and generation
- MVP can proceed without a real report generation system
- Easy to test -- inject a mock generator that returns controlled outputs or raises controlled errors
- Supports async: the protocol can define `async def generate(...)` for non-blocking execution

### Negative
- Must define the protocol shape now, before the real generation system is known -- risk of mismatch
- Adds one layer of indirection vs. a direct function call

### Neutral
- The protocol lives in `packages/api/src/services/` alongside other service abstractions
- The stub implementation lives in the same package, selected via configuration
