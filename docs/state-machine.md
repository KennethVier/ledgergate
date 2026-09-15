# Invoice State Machine

## 1. Purpose

The invoice lifecycle is persisted explicitly so that business state does not depend on an n8n execution remaining available forever.

The state machine separates:

- normal processing states;
- human-review/business exception states;
- terminal business outcomes;
- technical recovery states.

## 2. Primary states

| State | Meaning |
| --- | --- |
| `RECEIVED` | Intake event was accepted and durably identified |
| `EXTRACTING` | Document extraction is in progress |
| `VALIDATING` | Structured data is undergoing deterministic checks |
| `AWAITING_APPROVAL` | Required human approval is outstanding |
| `APPROVED` | Required approval policy has been satisfied |
| `READY_FOR_PAYMENT` | MVP automation completed successfully; safe handoff boundary reached |

## 3. Review / exception states

| State | Meaning |
| --- | --- |
| `EXTRACTION_REVIEW` | Extracted data is incomplete/ambiguous/low-confidence |
| `DUPLICATE_REVIEW` | Invoice appears to duplicate an existing business document |
| `VENDOR_REVIEW` | Vendor cannot be matched safely |
| `VALIDATION_REVIEW` | Deterministic business validation failed and requires operator action |
| `OPERATIONAL_EXCEPTION` | Technical processing exhausted automatic recovery and needs intervention |

## 4. Terminal states

| State | Meaning |
| --- | --- |
| `READY_FOR_PAYMENT` | Successful terminal state for MVP automation |
| `APPROVAL_REJECTED` | Human approval was explicitly denied |
| `CANCELLED` | Operator intentionally closed processing without completion |

`READY_FOR_PAYMENT` does not mean `PAID`. Payment execution is outside MVP scope.

## 5. High-level transitions

```text
RECEIVED
   |
   v
EXTRACTING
   |
   +---- extraction unsafe ------> EXTRACTION_REVIEW
   |                                    |
   |                               resolved/corrected
   |                                    |
   +------------------------------------+
   |
   v
VALIDATING
   |
   +---- duplicate -------------> DUPLICATE_REVIEW
   |                                  |
   |                             resolved/accepted
   |                                  |
   +----------------------------------+
   |
   +---- unknown vendor ---------> VENDOR_REVIEW
   |                                  |
   |                             resolved/matched
   |                                  |
   +----------------------------------+
   |
   +---- validation issue -------> VALIDATION_REVIEW
   |                                  |
   |                             resolved/corrected
   |                                  |
   +----------------------------------+
   |
   v
approval required?
   |
   +---- no ------------------------------+
   |                                      |
   +---- yes --> AWAITING_APPROVAL        |
                    |                     |
               approve|reject             |
                    |     +--> APPROVAL_REJECTED
                    v
                APPROVED
                    |
                    +---------------------+
                                          |
                                          v
                                  READY_FOR_PAYMENT
```

Any non-terminal processing state may move to `OPERATIONAL_EXCEPTION` after an unexpected technical failure exhausts its allowed automatic recovery.

## 6. Transition invariants

- A transition must be validated against an allowed transition table/policy.
- Every material transition emits an audit event.
- A transition is persisted before dependent downstream side effects where feasible.
- `APPROVAL_REJECTED`, `CANCELLED`, and `READY_FOR_PAYMENT` are terminal in the MVP.
- A review-state resolution must include an operator/system reason.
- Technical retry attempts do not invent new business states unless recovery is exhausted.
- Reprocessing the same input must not create a second invoice simply because a workflow execution was restarted.

## 7. Review-state resolution

Human review is modeled as a controlled transition, not as editing the state column arbitrarily.

Example: duplicate review

```text
DUPLICATE_REVIEW
    |
    +--> confirmed duplicate --> CANCELLED
    |
    +--> false positive -------> VALIDATING
```

Example: extraction review

```text
EXTRACTION_REVIEW
    |
    +--> corrected fields -----> VALIDATING
    |
    +--> unusable document ----> CANCELLED
```

Example: operational exception

```text
OPERATIONAL_EXCEPTION
    |
    +--> retry permitted ------> previous recoverable processing step
    |
    +--> corrected manually ---> appropriate validated state
    |
    +--> abandon -------------> CANCELLED
```

The implementation should persist enough exception metadata to determine the safe resume point rather than guessing from the latest workflow node.

## 8. State versus event

A state answers **where the invoice is now**.

An audit event answers **what happened**.

For example:

```text
Current state: AWAITING_APPROVAL

Audit history:
- INVOICE_RECEIVED
- EXTRACTION_COMPLETED
- VALIDATION_COMPLETED
- APPROVAL_REQUESTED
```

Keeping these concepts separate prevents business history from being lost when the current state changes.

## 9. Future extension states

Later phases may add states such as `SYNC_PENDING`, `SYNCED`, `RECONCILIATION_REVIEW`, or `PAID_EXTERNAL`, but none should be introduced until a real downstream integration is implemented.
