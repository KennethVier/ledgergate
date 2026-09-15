# ADR-0001: Use n8n as the Orchestration Layer

- **Status:** Accepted for MVP
- **Date:** 2026-09-16

## Context

LedgerGate is intended to demonstrate production-oriented workflow automation, not to become another large custom application competing with the developer's primary software-engineering projects.

The system still needs to coordinate multiple concerns:

- inbound webhook/email events;
- document handling;
- AI-assisted extraction;
- database reads/writes;
- deterministic routing;
- human approval and wait/resume behavior;
- retries and failure handling;
- notifications;
- future external accounting integrations.

A custom Spring Boot orchestration service could implement these requirements, but doing so would move the project away from its purpose: learning and demonstrating n8n/integration engineering while minimizing custom-code cost.

## Decision

Use **n8n as the orchestration layer** for the MVP.

Use **PostgreSQL/Supabase as the durable business source of truth**.

Use custom code only when a requirement cannot be expressed maintainably with standard n8n nodes, SQL constraints/functions, or small bounded transformation logic.

Do not place authoritative long-lived business state only inside n8n execution data or workflow static memory.

## Consequences

### Positive

- Directly develops the n8n skills relevant to automation/integration work.
- Visual workflows make business process behavior easy to demonstrate to clients.
- Built-in integrations reduce boilerplate application code.
- Human wait/approval, webhook, HTTP, database, and error-handling patterns can be modeled without a custom service layer.
- The project can reach a deployable vertical slice with substantially less implementation effort than a separate backend/frontend application.

### Negative

- Large workflows can become difficult to maintain if responsibilities are not decomposed.
- Excessive Code nodes can turn n8n into an opaque custom application.
- Workflow JSON can be noisy in source control.
- Some advanced domain logic may eventually justify a dedicated service.
- Runtime/execution history must not be confused with durable business state.

## Guardrails

1. Keep workflows responsibility-oriented and bounded.
2. Prefer standard nodes and explicit mappings over large Code nodes.
3. Persist business state in PostgreSQL before relying on downstream continuation.
4. Use database uniqueness/constraints for invariants that must survive retries.
5. Export workflow JSON to Git without credentials.
6. Introduce a custom service only through a future ADR that identifies a concrete limitation and measurable benefit.

## Alternatives considered

### Spring Boot orchestration service

Rejected for MVP because it would duplicate capabilities n8n is meant to demonstrate and materially increase implementation/token/time cost.

### Pure database automation

Rejected because SQL alone is not an appropriate orchestration surface for external APIs, human waits, email/webhook handling, and visual operational workflows.

### Serverless functions as primary orchestrator

Deferred. Functions may later complement n8n for isolated compute or webhook verification, but they do not provide the same workflow visibility and learning value for this project's goal.

## Revisit criteria

Reconsider this decision if one or more of the following becomes true:

- domain rules become too complex to express/read safely in n8n;
- transaction boundaries require logic that must execute atomically outside workflow orchestration;
- performance/latency targets exceed practical n8n behavior;
- a downstream integration requires a reusable service API used by multiple non-n8n clients;
- Code nodes begin accumulating substantial business logic.
