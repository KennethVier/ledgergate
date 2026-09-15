# LedgerGate

Production-oriented invoice and exception automation platform built around **n8n**, **PostgreSQL/Supabase**, AI-assisted extraction, human approvals, and resilient workflow patterns.

> **Status:** Phase 0 — Business Architecture. No production automation is implemented yet.

## Why this project exists

LedgerGate is a portfolio project for learning and demonstrating business automation as an engineering discipline rather than as a collection of simple app-to-app integrations.

The system models a realistic accounts-payable workflow for a fictional small-to-medium business. Vendor invoices enter through email or webhook intake, are extracted and validated, routed through approval rules, persisted as auditable state, and moved into an exception path whenever automation cannot safely continue.

The implementation is intentionally **n8n-first**. n8n owns orchestration; PostgreSQL owns durable business state; AI is limited to probabilistic extraction/classification; deterministic rules and explicit human approvals protect consequential decisions.

## Design goals

- Model a real business process with explicit states and ownership.
- Keep workflows modular instead of building one oversized n8n canvas.
- Make retries safe through idempotency and duplicate detection.
- Treat exception handling and recovery as first-class behavior.
- Preserve an audit trail for material business decisions.
- Use AI only where uncertainty is expected and validate its output.
- Require human approval for risk-sensitive paths.
- Keep credentials and secrets outside exported workflow definitions.
- Produce documentation that another engineer could operate and extend.

## Planned MVP flow

```text
Invoice intake
    |
    v
Persist intake event + idempotency key
    |
    v
Store document
    |
    v
AI-assisted structured extraction
    |
    v
Deterministic validation
    |
    +--> duplicate / unknown vendor / low confidence --> Exception queue
    |
    v
Approval policy
    |
    +--> human approval when required
    |
    v
Ready for payment / downstream sync
    |
    v
Audit event + notification
```

The MVP stops at a safe `READY_FOR_PAYMENT` boundary. It does **not** autonomously move money or connect to a real banking system.

## Architecture boundaries

| Component | Responsibility |
| --- | --- |
| n8n | Workflow orchestration, integrations, branching, waits, retries, notifications |
| PostgreSQL / Supabase | Source of truth for invoices, processing state, idempotency, approvals, exceptions, audit events |
| AI provider | Structured extraction/classification only; never the source of truth |
| Email / webhook | Invoice intake channels |
| Object/document storage | Original invoice documents and immutable references |
| Human approver | Approval and exception resolution where policy requires judgment |

## Repository structure

```text
ledgergate/
├── workflows/                 # Exported n8n workflows (Phase 1+)
├── database/
│   ├── migrations/            # PostgreSQL schema changes (Phase 1+)
│   └── seed/                  # Safe demo data
├── fixtures/                  # Synthetic invoice/test inputs
├── docs/
│   ├── architecture.md
│   ├── business-process.md
│   ├── state-machine.md
│   ├── failure-model.md
│   └── decisions/             # Architecture Decision Records
├── .env.example               # Non-secret configuration contract (Phase 1+)
└── README.md
```

## Phase plan

### Phase 0 — Business Architecture

- [x] Define the fictional business context and operational problem.
- [x] Define system boundaries and responsibilities.
- [x] Define invoice lifecycle/state machine.
- [x] Define failure and exception taxonomy.
- [x] Record the first architecture decision.
- [ ] Review and lock MVP scope.

### Phase 1 — Automation Foundation

- [ ] Local n8n setup.
- [ ] PostgreSQL/Supabase schema.
- [ ] Credentials/configuration strategy.
- [ ] Correlation ID and idempotency conventions.
- [ ] Base error workflow.

### Phase 2 — Vertical Slice

- [ ] Invoice intake.
- [ ] Document persistence.
- [ ] AI structured extraction.
- [ ] Validation and duplicate checks.
- [ ] Approval routing.
- [ ] Exception queue and recovery.
- [ ] Audit events and notifications.

### Phase 3 — Production Hardening & Portfolio Evidence

- [ ] Retry policy and recovery tests.
- [ ] Security review.
- [ ] Synthetic fixtures and failure scenarios.
- [ ] Deployment/runbook documentation.
- [ ] Architecture diagram and recorded demo.

## Engineering principles

1. **Orchestrate, do not hide a monolith inside n8n.** Complex responsibilities become bounded sub-workflows.
2. **Persist before side effects.** Durable state is recorded before downstream actions that may need recovery.
3. **Assume retries happen.** Every externally triggered operation must be safe to execute more than once.
4. **AI output is untrusted input.** Schema validation and deterministic checks are mandatory.
5. **Exceptions are business states, not just errors.** A valid invoice needing a human review is different from a crashed workflow.
6. **No invisible decisions.** Material state transitions produce audit events.
7. **Automation stops at unsafe boundaries.** The MVP never autonomously executes a real payment.

## Documentation

- [Business process](docs/business-process.md)
- [Architecture](docs/architecture.md)
- [State machine](docs/state-machine.md)
- [Failure model](docs/failure-model.md)
- [ADR-0001: n8n as the orchestration layer](docs/decisions/0001-n8n-as-orchestrator.md)

## License

A license will be selected before the project is presented as a reusable/open-source template.
