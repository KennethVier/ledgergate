# Failure and Exception Model

## 1. Why failures are modeled explicitly

LedgerGate distinguishes between **business exceptions** and **technical failures**.

A business exception means the system is working correctly but cannot safely continue without a decision. A technical failure means an expected system operation did not complete as intended.

Treating both as generic workflow errors would make operations harder to understand and recover.

## 2. Failure classes

### A. Business exceptions

Expected states requiring review rather than automatic retry.

Examples:

- duplicate invoice suspicion;
- unknown vendor;
- low-confidence extraction;
- inconsistent totals;
- approval rejection;
- unsupported currency or document shape.

Behavior:

- persist a review/terminal business state;
- record a human-readable reason;
- create an audit event;
- notify the responsible operator when action is required;
- do not repeatedly retry an outcome that requires human judgment.

### B. Transient technical failures

Failures reasonably expected to succeed later.

Examples:

- HTTP 429 rate limit;
- HTTP 502/503/504 from an external service;
- temporary network timeout;
- short-lived AI provider outage;
- notification service unavailable.

Behavior:

- retry only when the operation is safe to retry;
- use bounded attempts and increasing delay/jitter where appropriate;
- preserve idempotency keys across attempts;
- record attempt metadata;
- escalate to `OPERATIONAL_EXCEPTION` after the retry budget is exhausted.

### C. Permanent technical failures

Failures unlikely to succeed without configuration/code/data changes.

Examples:

- invalid credentials;
- HTTP 400 caused by a malformed request contract;
- missing required configuration;
- database constraint violation indicating a programming/invariant error;
- unsupported API version.

Behavior:

- fail fast rather than burning retry attempts;
- record diagnostic context without secrets;
- create an operational exception;
- alert the operator/developer.

### D. Ambiguous side-effect failures

The most dangerous category: the caller cannot tell whether an external side effect occurred.

Example:

```text
LedgerGate sends request
        |
external system commits change
        |
network drops before response arrives
        |
LedgerGate sees timeout
```

Blind retry could duplicate the side effect.

Behavior:

1. Prefer external APIs that support idempotency keys.
2. Persist an integration-event record before the call.
3. On ambiguous timeout, query/reconcile remote state when possible.
4. Retry only when duplicate execution is provably safe.
5. Otherwise route to `OPERATIONAL_EXCEPTION` for manual recovery.

## 3. Retry policy

Retry policy is defined per operation, not globally.

Initial guidance:

| Failure | Automatic retry? | Notes |
| --- | --- | --- |
| 429 | Yes | Respect provider retry hints/rate limits |
| 502/503/504 | Yes | Bounded backoff |
| Network timeout before any side effect | Usually | Only if idempotent/safe |
| Ambiguous timeout after possible side effect | Conditional | Reconcile first |
| 400/validation error | No | Fix request/data |
| 401/403 | No automatic loop | Credentials/permissions need intervention |
| Duplicate business invoice | No | Human/business review |
| Low AI confidence | No blind retry | Human review or alternate extraction policy |
| Approval rejection | No | Terminal business decision |

Exact delays and attempt counts are deferred until integration characteristics are known.

## 4. Central error workflow

Unexpected n8n workflow failures should route into a dedicated error-handling workflow rather than duplicating alert logic everywhere.

Minimum captured context:

- workflow name/id;
- n8n execution reference;
- correlation ID when available;
- invoice ID when available;
- failing operation/node;
- sanitized error category/message;
- timestamp;
- retryability classification where known.

The error workflow must never log credentials, access tokens, full secret-bearing headers, or unnecessary invoice contents.

## 5. Exception record

A durable exception record should eventually contain fields equivalent to:

```text
id
invoice_id
correlation_id
category
code
status
summary
retryable
resume_from
attempt_count
last_error_at
resolved_at
resolution
created_at
updated_at
```

This is conceptual Phase 0 modeling; the exact SQL schema is Phase 1 work.

## 6. Recovery principles

### Never recover from UI memory alone

A user should not need to remember which n8n node failed. Persist enough recovery context to determine the safe next step.

### Prefer resume from a durable checkpoint

Example:

```text
RECEIVED -> extraction persisted -> validation failed technically
```

Recovery should reuse the persisted extraction rather than call the AI provider again unnecessarily.

### Preserve business identity

A recovered execution reuses the same invoice identity and correlation context. Recovery must not create a fresh business object unless explicitly intended.

### Make operator action explicit

Operational exceptions should answer:

- What failed?
- Was anything already committed?
- Is retry safe?
- What input/configuration needs correction?
- Where will processing resume?

## 7. Failure-injection scenarios for the portfolio demo

The project will intentionally test failures instead of demoing only a happy path.

Required scenarios:

1. Submit the same inbound event twice.
2. Submit two different documents with the same vendor + invoice number.
3. Make the AI response omit a required field.
4. Force low-confidence extraction.
5. Fail a mock external API with 503 and then recover.
6. Exhaust retries and verify an exception record/alert is created.
7. Simulate an ambiguous timeout and prove no duplicate side effect occurs.
8. Reject a human approval and confirm it is handled as a business outcome.
9. Restart/re-run a workflow execution and prove durable state prevents duplicate invoice creation.

## 8. Definition of reliable enough for MVP

The MVP is considered reliability-complete when:

- duplicate delivery cannot create duplicate invoice records;
- retryable failures are bounded;
- non-retryable failures do not loop;
- exhausted failures become visible operational exceptions;
- business review states are distinct from crashes;
- every accepted invoice can be traced with a correlation ID;
- recovery can continue from a documented durable checkpoint;
- audit history remains comprehensible after retries and manual resolutions.
