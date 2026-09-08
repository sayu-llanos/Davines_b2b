# Architecture Decisions

## ADR-001 — Deterministic routing before LLM reasoning
Critical commercial paths need predictable behavior.

## ADR-002 — Persistent conversation state
State is stored outside the LLM to prevent repeated questions and support multi-turn qualification.

## ADR-003 — Human handoff as a state transition
Once a salesperson takes over, automation should not restart qualification or compete with the human conversation.

## ADR-004 — Secrets outside workflow exports
API keys are stored in n8n Credentials rather than plaintext HTTP headers.
