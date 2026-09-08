# Build Log

> Historical development reconstructed on 2026-09-08 from production artifacts, QA history and project records.

## Phase 1 — Initial WhatsApp automation

Goal:
Create a B2B WhatsApp assistant capable of answering inbound leads and guiding conversations toward a sales advisor.

Initial architecture:

WhatsApp → YCloud → n8n → OpenAI

Early problems discovered:

- Conversations could lose context
- Different customer intents followed similar paths
- The AI could repeat questions
- Commercial routing needed more predictable behavior

## Phase 2 — Intent classification

Added deterministic classification for different conversation types, including:

- B2B prospects
- B2C users
- Existing customers
- Product information
- Pricing requests
- Commercial intent
- Requests for a human advisor

Reason:

Important commercial decisions needed predictable routing instead of relying only on the language model.

## Phase 3 — Persistent conversation state

Added Supabase persistence to maintain customer and conversation state across messages.

This allowed the system to remember information such as:

- Customer type
- City and district
- Business information
- Qualification progress
- Previously shared material
- Human handoff state

## Phase 4 — Product knowledge and RAG

Added product knowledge retrieval using vector search and OpenAI embeddings.

This allowed the assistant to retrieve relevant product information instead of depending only on the model's general knowledge.

## Phase 5 — Human handoff

Added a structured handoff flow:

AI conversation → lead qualification → CRM persistence → salesperson handoff → bot pause

The system can recognize when automation should stop and a human sales advisor should continue the conversation.

## Phase 6 — Production QA

Real conversational testing exposed edge cases such as:

- One-word answers
- Typos and informal language
- Multiple needs in one message
- Corrections to previously captured information
- Repeated qualification questions
- Existing-customer conversations
- Post-handoff messages
- Audio messages

These cases led to iterative fixes and hotfixes in the production routing logic.

## Key lesson

The final system became a hybrid architecture:

Deterministic routing and state management for critical business logic, combined with AI for flexible language understanding and natural conversation.