# Architecture

## 1. Architectural intent

LedgerGate is an **automation/orchestration system**, not a conventional full-stack application. The MVP deliberately minimizes custom application code so the project can demonstrate n8n, integration engineering, persistent workflow state, human approval, and recoverability without competing with larger coding projects.

The system follows four primary boundaries:

1. **n8n orchestrates** workflows and external integrations.
2. **PostgreSQL/Supabase persists** durable business state and audit history.
3. **AI assists** extraction/classification but never becomes the source of truth.
4. **Humans approve or resolve** ambiguous and consequential business states.

## 2. Context diagram

```text
                    +----------------------+
                    | Vendor / Test Client |
                    +----------+-----------+
                               |
                     Email / HTTP Webhook
                               |
                               v
+----------------+    +--------------------+    +---------------------+
| Human Approver |<-->|        n8n         |<-->| AI Extraction Model |
+----------------+    | Orchestration Layer|    +---------------------+
                      +----+-----------+----+
                           |           |
                           |           +-----------> Notification channel
                           |
                           v
                    +------------------+
                    | PostgreSQL /     |
                    | Supabase         |
                    | Source of Truth  |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | Document Storage |
                    | / Reference      |
                    +------------------+
```

## 3. Responsibility boundaries

### n8n

Owns:

- intake triggers;
- orchestration order;
- calls to AI and external APIs;
- transformation/mapping needed at integration boundaries;
- conditional routing;
- wait/resume behavior for approvals;
- bounded retry behavior;
- invoking reusable sub-workflows;
- operator notifications;
- centralized workflow error routing.

Does not own:

- the authoritative invoice lifecycle;
- hidden business state stored only inside execution history;
- long-lived business truth that would disappear if executions were pruned;
- real payment authorization.

### PostgreSQL / Supabase

Owns:

- invoice identity and current lifecycle state;
- vendor records used by the demo;
- idempotency keys;
- structured extracted data accepted for processing;
- approval requests/outcomes;
- exception records;
- audit events;
- integration-event status where safe retries are required.

Database constraints should enforce invariants where practical rather than relying only on workflow branches.

### AI provider

May:

- extract invoice number/date/vendor/amount/currency/line items;
- normalize noisy document text;
- provide per-field or overall confidence metadata where supported;
- classify document type if needed later.

Must not:

- approve an invoice;
- decide that a duplicate can be ignored;
- mutate vendor payment details;
- mark an invoice as paid;
- overwrite durable business truth without deterministic validation.

### Human approver/operator

Owns:

- approval decisions required by policy;
- resolving duplicate/vendor/extraction/validation review states;
- deciding whether an operational exception should be retried, corrected, or closed;
- any real-world financial action outside the demo system.

## 4. Planned workflow decomposition

The exact node implementation belongs to later phases, but the MVP should preserve these logical boundaries:

| Workflow | Responsibility |
| --- | --- |
| `LG-WF-001 Intake` | Receive invoice, correlation ID, initial persistence, idempotency check |
| `LG-WF-002 Extract` | Document retrieval + AI structured extraction |
| `LG-WF-003 Validate` | Deterministic validation + duplicate/vendor checks |
| `LG-WF-004 Approval` | Approval policy, wait/resume, approval persistence |
| `LG-WF-005 Exceptions` | Operator-facing exception creation/recovery path |
| `LG-WF-006 Notifications` | Email/Slack-style operational notifications |
| `LG-WF-900 Error Handler` | Unexpected workflow failures and diagnostic capture |

A workflow should not become a generic utility dumping ground. Reuse is introduced only where a responsibility is stable and meaningful.

## 5. Data and control flow

### Intake

1. Trigger receives an invoice event.
2. Generate/propagate `correlation_id`.
3. Normalize stable identifiers needed to derive `idempotency_key`.
4. Attempt durable intake persistence.
5. If idempotency conflict proves the event was already accepted, return the previous outcome or stop safely.
6. Continue only after durable identity exists.

### Extraction

1. Retrieve the immutable document reference.
2. Invoke the AI provider with a strict structured-output contract.
3. Treat the returned payload as untrusted input.
4. Validate required fields/types before persistence as candidate extraction data.
5. Route low-confidence/incomplete output to review.

### Validation

Deterministic checks include, at minimum:

- required field presence;
- recognized currency format;
- amount invariants;
- known vendor match;
- duplicate candidate detection;
- status transition validity.

### Approval

1. Determine approval policy from validated invoice attributes.
2. Persist the approval requirement before sending the external request.
3. Wait for human response where necessary.
4. Persist the human outcome.
5. Continue only if the persisted outcome permits it.

## 6. Identity conventions

Every top-level business flow should carry:

- `correlation_id`: traces one business processing attempt across workflow boundaries;
- `invoice_id`: durable database identity once created;
- `idempotency_key`: identifies the same inbound business event/invoice attempt;
- `execution_id`: optional n8n execution reference for diagnostics, not business identity.

These values must not be treated interchangeably.

## 7. Idempotency strategy

The workflow is designed around **at-least-once delivery assumptions**: webhook senders, users, or integrations may retry.

Initial invoice idempotency will use a normalized business key built from available stable attributes, for example:

```text
normalized_vendor_identity + invoice_number
```

When invoice number is unavailable at intake, a provisional intake key may use immutable source metadata such as sender/message ID/document hash. Once extraction completes, duplicate detection uses stronger business identity.

The database—not an in-memory n8n variable—must enforce uniqueness for durable idempotency records.

## 8. Audit model

Material transitions generate append-only audit events containing enough information to reconstruct:

- what entity changed;
- previous state;
- new state;
- event type;
- correlation ID;
- actor (`system`, workflow, or human identity when available);
- timestamp;
- compact reason/metadata.

Audit events are not a replacement for normal application logs. Logs explain execution diagnostics; audit events explain business history.

## 9. Security boundaries

- No API keys or OAuth secrets are committed to Git.
- Exported n8n JSON must not contain usable credentials.
- Webhook endpoints must use appropriate authentication/signature controls before production exposure.
- Database operations must use scoped credentials and parameterized inputs.
- Synthetic invoices must contain no real personal, banking, employer, or vendor data.
- AI prompts must not receive secrets or unrelated sensitive information.
- Execution data retention should be reviewed before using real business documents.

## 10. Deployment direction

Phase 1 starts locally so workflow behavior can be learned and tested cheaply.

Likely progression:

```text
Local development
    n8n via Docker
    PostgreSQL/Supabase dev project
    synthetic fixtures

        ↓

Portfolio deployment
    managed or self-hosted n8n
    Supabase PostgreSQL
    configured external integrations
    demo-only data
```

The project will choose an actual hosting provider only after the vertical slice works locally. Deployment choice is therefore intentionally not locked in Phase 0.

## 11. Observability direction

Minimum portfolio-grade observability:

- searchable n8n execution history;
- correlation ID on business records and diagnostic messages;
- persisted exception records;
- append-only audit events;
- failure notifications for exhausted/unexpected failures;
- enough context to replay/recover an operation safely.

Metrics, OpenTelemetry, queue mode, and centralized log infrastructure are deferred until justified by workload or portfolio value.

## 12. Architectural quality bar

A workflow is not considered complete merely because the happy path runs. Each implemented vertical slice must answer:

1. What happens if the trigger is delivered twice?
2. What happens if an external API times out after accepting the request?
3. What is persisted before the next side effect?
4. Can a human understand why automation stopped?
5. Can the operation be retried safely?
6. Can the business history be reconstructed later?
7. Are AI outputs validated before they affect durable state?
