# Operational CRM and Human Review

The B2B WhatsApp system used Google Sheets as a lightweight operational CRM for commercial lead review and sales follow-up.

Supabase and Google Sheets served different purposes.

Supabase stored live conversation and application state, while the CRM transformed conversational information into structured commercial records that the marketing and sales teams could review and act on.

```text
WhatsApp
    ↓
AI conversation
    ↓
Product education + qualification
    ↓
Supabase
Live conversation state
    ↓
Structured commercial information
    ↓
Google Sheets CRM
    ↓
Human review and enrichment
    ↓
Sales follow-up