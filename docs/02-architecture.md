# Architecture

```text
WhatsApp / YCloud
  ↓
Webhook
  ↓
Message normalization
  ├─ text
  └─ audio → transcription
  ↓
Intent classification
  ↓
Persistent conversation state
  ↓
State-aware router
  ├─ general conversation
  ├─ product / technical information
  ├─ B2B qualification
  ├─ B2C redirect
  ├─ existing customer
  ├─ commercial intent
  └─ human handoff
  ↓
AI Agent + guardrails
  ↓
RAG / vector retrieval
  ↓
Conversation memory
  ↓
CRM persistence
  ↓
WhatsApp response / multimedia
```
