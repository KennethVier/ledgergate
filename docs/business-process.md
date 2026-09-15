# Business Process

## 1. Scenario

LedgerGate models accounts-payable automation for **Harborline Services Group (HSG)**, a fictional multi-department services company used only as a portfolio/demo business.

HSG receives vendor invoices through a shared finance inbox and occasional system-to-system submissions. The finance team currently performs repetitive intake, duplicate checks, data entry, approval routing, follow-up, and status tracking manually.

The project assumes approximately 150–300 invoices per month. These numbers are design assumptions, not claims about a real company.

## 2. Business problem

The manual process creates several operational risks:

- duplicate invoices can be entered twice;
- invoice fields can be copied incorrectly;
- approval routing depends on tribal knowledge;
- approvals can sit unnoticed in inboxes;
- exceptions are mixed with ordinary work;
- retrying an integration can create duplicate side effects;
- finance has limited visibility into where an invoice is blocked;
- audit evidence is reconstructed after the fact instead of recorded during processing.

LedgerGate is intended to reduce manual handling while preserving explicit human control over risky or ambiguous decisions.

## 3. Actors

| Actor | Responsibility |
| --- | --- |
| Vendor | Submits an invoice and supporting information |
| Finance operator | Owns intake exceptions and final operational review |
| Department approver | Approves invoices within assigned business authority |
| Finance approver | Reviews higher-value or finance-sensitive invoices |
| LedgerGate | Orchestrates processing and records state |
| AI extraction service | Suggests structured invoice fields from document content |
| Downstream accounting system | Future external destination for approved invoices; mocked in MVP |

## 4. In-scope MVP

The MVP covers one complete vertical slice:

1. Receive an invoice from a webhook or controlled email intake.
2. Create a correlation ID and derive an idempotency key.
3. Persist the intake record before performing non-repeatable side effects.
4. Store or reference the original document.
5. Extract structured invoice fields using an AI-assisted step.
6. Validate the extracted payload against deterministic rules.
7. Detect probable duplicate invoices.
8. Route ambiguous or unsafe cases to an exception queue.
9. Evaluate approval policy for valid invoices.
10. Pause for a human approval when policy requires it.
11. Mark the invoice `READY_FOR_PAYMENT` when all required controls pass.
12. Record audit events and notify the relevant operator.

## 5. Explicitly out of scope for the MVP

- executing real payments;
- storing banking credentials;
- autonomous vendor bank-detail changes;
- full purchase-order matching;
- ERP-specific production integration;
- tax/accounting advice;
- multi-tenant SaaS support;
- a custom React operations dashboard;
- advanced queue-mode scaling.

These may become later phases only when they serve a demonstrated requirement.

## 6. Happy path

```text
Vendor
  |
  v
Invoice received
  |
  v
Intake persisted
  |
  v
Document stored/referenced
  |
  v
AI extraction
  |
  v
Schema + business validation
  |
  v
Duplicate check
  |
  v
Approval policy
  |
  +--> no approval required ------+
  |                                |
  +--> approval required --> Human approval
                                   |
                                   v
                          READY_FOR_PAYMENT
                                   |
                                   v
                              Audit event
                                   |
                                   v
                              Notification
```

## 7. Exception paths

### Duplicate suspicion

If the same vendor, invoice number, and materially equivalent invoice identity already exist, the new invoice does not continue automatically. It moves to `DUPLICATE_REVIEW`.

### Unknown vendor

If the normalized vendor cannot be matched to an approved vendor record, processing moves to `VENDOR_REVIEW`.

### Low-confidence extraction

If required fields are absent, malformed, or below the configured confidence policy, processing moves to `EXTRACTION_REVIEW` rather than guessing.

### Validation failure

If deterministic rules fail—for example total calculations are inconsistent—the invoice moves to `VALIDATION_REVIEW`.

### Approval rejection

A rejected invoice moves to `APPROVAL_REJECTED`. Rejection is a business outcome, not a system failure.

### Integration failure

A transient external failure is retried according to policy. When retry limits are exhausted, the invoice or integration event moves to an operational exception state for safe recovery.

## 8. Initial approval policy

The exact thresholds are demo policy and are configurable.

| Condition | Routing |
| --- | --- |
| Total <= 500 USD equivalent and all controls pass | No additional approval in MVP demo |
| 500.01–2,500 | Department approver |
| 2,500.01–10,000 | Department approver + finance approver |
| > 10,000 | Finance approver; manual finance handling |
| Unknown vendor | Vendor review regardless of amount |
| Duplicate suspicion | Duplicate review regardless of amount |
| Low extraction confidence | Extraction review regardless of amount |
| Material vendor/payment-detail change | Block automatic progression |

The MVP stops at `READY_FOR_PAYMENT`; these thresholds do not authorize real financial transactions.

## 9. Service-level assumptions

These are portfolio design targets, not contractual SLAs.

- Intake acknowledgement should be recorded within one workflow execution after receipt.
- No invoice should disappear silently after a failed workflow.
- Every material state transition should have an associated audit event.
- A retry should not create a duplicate invoice record or duplicate external action.
- Human-review states must expose why automation stopped and what action is required.

## 10. Definition of success

The vertical slice is considered successful when the demo can prove all of the following with synthetic fixtures:

- a valid invoice reaches `READY_FOR_PAYMENT`;
- a duplicate is stopped safely;
- malformed/low-confidence extraction is routed to human review;
- a policy-controlled invoice waits for approval and resumes correctly;
- a rejected invoice terminates in a business state rather than an error state;
- a transient integration failure can be retried without duplicating prior work;
- an exhausted retry becomes a visible recoverable exception;
- the full lifecycle can be reconstructed from persisted state and audit events.
