# n8n Workflows

Exported n8n workflow definitions will live here beginning in Phase 1.

Planned logical workflow IDs:

- `LG-WF-001` — Intake
- `LG-WF-002` — Extract
- `LG-WF-003` — Validate
- `LG-WF-004` — Approval
- `LG-WF-005` — Exceptions
- `LG-WF-006` — Notifications
- `LG-WF-900` — Error Handler

## Source-control rules

- Never commit credentials or usable access tokens.
- Export workflow JSON after meaningful, reviewed changes.
- Keep workflow names/IDs stable once referenced by documentation or sub-workflows.
- Avoid giant Code nodes; material business logic requires documentation and may require an ADR.
- Treat exported JSON as deployable configuration, not as the database/source of business state.
