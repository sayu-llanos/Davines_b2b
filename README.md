# B2B WhatsApp AI Sales System

**Production system designed and implemented by Sayuri Llanos.**

A WhatsApp-first B2B sales and qualification system combining deterministic routing, conversational AI, persistent state, retrieval, CRM persistence, multimedia delivery and human handoff.

## Stack
- n8n
- WhatsApp / YCloud
- OpenAI
- Supabase
- PostgreSQL
- Vector search / RAG
- Google Drive / Google Sheets

## What it does
- Normalizes inbound WhatsApp messages, including audio
- Classifies intent and routes key scenarios
- Maintains conversation state to avoid repeated questions
- Retrieves product knowledge through vector search
- Qualifies B2B leads
- Persists CRM data
- Sends relevant multimedia / documents
- Hands conversations off to a salesperson when appropriate

## Evidence policy
The original production export is **not committed** because it contains company-specific operational logic and identifiers.

A SHA-256 fingerprint of the original artifact is stored at:

`evidence/2026-09-08-original.sha256`

Original artifact SHA-256:

`f438ed58d419696112d2eb5605208eb7f505f06a2562e616023aa08bd2ef0668`

The workflow under `portfolio/` preserves only the system structure. Production prompts, executable routing code, credentials, URLs and private business rules are intentionally omitted.
