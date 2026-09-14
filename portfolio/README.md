# Public-safe workflow snapshot

`davines-b2b-public-safe.json` is a sanitized structural snapshot of the production B2B WhatsApp workflow.

It is intentionally **not a production-ready export**.

The file preserves enough structure to inspect the system architecture, including workflow topology, node types, routing layers, state persistence, RAG components, CRM integration, and human-handoff paths.

The following are intentionally removed or replaced:

- API credentials and credential identifiers
- private webhook URLs
- production phone numbers
- customer PII
- Google Drive / Sheet identifiers
- private database identifiers
- internal production URLs
- raw customer conversations
- private knowledge-source content
- executable private business logic inside code nodes

Code nodes contain portfolio-safe placeholders describing their responsibility rather than the original production implementation.

For deeper design context, see the repository documentation and architecture diagrams.
