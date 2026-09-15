# Database

PostgreSQL/Supabase will be the durable source of truth for LedgerGate business state.

Phase 1 will introduce:

- `migrations/` — ordered SQL migrations;
- `seed/` — synthetic demo data only.

Initial domain concepts expected to require persistence:

- vendors;
- invoices;
- extracted invoice data;
- approval requests/events;
- idempotency records;
- exceptions;
- integration events;
- audit events.

The exact schema is intentionally deferred until Phase 1 so that tables are derived from the locked business process and state machine rather than guessed in advance.

## Rules

- Prefer database constraints for durable uniqueness/invariants.
- Never commit real vendor, banking, invoice, or employer data.
- Migrations must be repeatable in a clean development environment.
- Business state must not depend on n8n execution history remaining available.
