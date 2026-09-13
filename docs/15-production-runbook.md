# Production Runbook

This document describes the operational troubleshooting process for the B2B WhatsApp conversational system.

The objective is to diagnose production failures systematically rather than modifying the workflow blindly.

The main principle is:

```text
Find the first layer where expected behavior stops.
```

The production path is:

```text
WhatsApp
    ↓
YCloud
    ↓
n8n Webhook
    ↓
Input normalization
    ↓
Clasificador F
    ↓
Supabase state
    ↓
Router F
    ↓
AI / RAG / qualification / multimedia
    ↓
State persistence
    ↓
Google Sheets CRM
    ↓
YCloud outbound response
```

A failure near the beginning of this chain can make every later component appear broken even when those components are healthy.

## Incident triage order

When a production conversation fails, inspect the system in this order:

```text
1. Did the WhatsApp event reach n8n?
2. Was the input normalized correctly?
3. What intent did Clasificador F produce?
4. What state existed in Supabase?
5. What route did Router F select?
6. Did the AI / RAG branch execute correctly?
7. Was structured data extracted correctly?
8. Was Supabase updated?
9. Was the CRM updated when required?
10. Did YCloud deliver the final response?
```

Avoid starting with the language model unless earlier layers have already been confirmed.

## First diagnostic question

The most useful first question is:

```text
Did the workflow execute?
```

If the answer is no:

```text
Inspect:
YCloud
Webhook
Inbound payload
Workflow activation
```

If the answer is yes:

```text
Inspect:
Execution data
Classifier
Router
State
Downstream integrations
```

## 1. No inbound execution

### Symptom

The customer sends a WhatsApp message but no new n8n execution appears.

### Check

```text
Is the workflow active?
Did YCloud receive the message?
Is YCloud pointing to the expected webhook?
Did the webhook URL change?
Is the incoming event type supported?
```

### Expected path

```text
Customer
    ↓
WhatsApp
    ↓
YCloud
    ↓
n8n Webhook
```

If there is no n8n execution, do not debug:

```text
Router F
Supabase
RAG
AI Agent
CRM
```

because the message has not reached those layers yet.

## 2. Webhook receives payload but message is empty

### Symptom

An execution exists, but later nodes do not contain the expected customer text.

### Inspect

```text
Webhook payload
Message type
Input normalization
Text extraction
Audio branch
```

WhatsApp events can represent different message formats.

Conceptually:

```text
Inbound event
    │
    ├── Text
    │     ↓
    │ extract text
    │
    └── Audio
          ↓
      download audio
          ↓
      transcription
```

Verify the normalized message before debugging routing.

## 3. Audio message fails

### Symptom

Text messages work but voice notes fail or return no useful response.

### Diagnostic path

```text
Was audio detected?
        ↓
Was the file downloaded?
        ↓
Was the audio accessible?
        ↓
Did transcription execute?
        ↓
Did transcription return text?
        ↓
Did normalized text reach classifier?
```

Possible failure points include:

```text
Invalid media URL
Expired media access
Authentication
Download failure
Unsupported payload
Transcription error
Empty transcript
```

After fixing the issue, retest both:

```text
Audio message
Text message
```

to ensure the normal text path was not affected.

## 4. YCloud authentication failure

### Example production symptom

```text
401
Header 'X-API-Key' is missing
```

This indicates an authentication-layer problem rather than a conversational AI problem.

### Diagnostic path

```text
Outbound HTTP node
        ↓
Authentication configuration
        ↓
Correct n8n credential selected?
        ↓
Header Auth configured?
        ↓
Credential valid?
```

The hardened production pattern is:

```text
HTTP Request
    ↓
Generic Credential Type
    ↓
Header Auth
    ↓
YCloud production credential
```

The API key should not be manually duplicated across request headers.

## 5. YCloud returns 400

### Symptom

Authentication succeeds, but the outbound request is rejected.

Typical causes include:

```text
Invalid JSON
Missing required field
Incorrect destination
Incorrect media structure
Malformed payload
Wrong endpoint
Incorrect Content-Type
```

### Diagnostic approach

Separate authentication from payload debugging.

```text
401
    ↓
Authentication problem

400
    ↓
Request / payload problem
```

Inspect the request body without exposing credentials.

Verify:

```text
Destination
Message type
Body structure
Media URL if applicable
Content-Type
Required API fields
```

## 6. Message reaches workflow but wrong intent is selected

### Symptom

A customer asks a valid product or commercial question but `Clasificador F` returns an unexpected intent.

### Inspect

```text
Normalized text
DEBUG_intencion
Detected product line
Detected hair need
Known commercial terms
Typos / short wording
```

Possible examples:

```text
Product need
classified as invalid

Commercial request
classified as generic information

B2C consumer
classified as professional B2B
```

Do not immediately modify the AI prompt.

First determine whether the error occurs in:

```text
Classifier
```

or:

```text
Router
```

These are separate layers.

## 7. Classifier is correct but final route is wrong

### Symptom

`Clasificador F` identifies the message correctly, but the system performs the wrong action.

### Inspect `Router F`

Useful debug fields include:

```text
DEBUG_intencion
DEBUG_intencionFinal
DEBUG_esPrimerMensaje
DEBUG_catalogoYaEnviado
DEBUG_lineaNombre
DEBUG_lineasPendientes
```

Also inspect current Supabase state:

```text
fase_actual
estado_conversacion
estado
bot_pausado
asesor_solicitado
multimedia_enviado
lineas
lineas_pendientes
```

A correct intent can intentionally produce a different route because of existing state.

Example:

```text
PRECIOS
+
pricing already sent
        ↓
CONVERSACION

instead of

MULTIMEDIA
```

This is expected state-aware behavior, not necessarily a routing bug.

## 8. Qualification restarts unexpectedly

### Symptom

The assistant asks questions that were already answered.

For example:

```text
User already provided city
        ↓
Bot asks city again
```

### Inspect

```text
Was ciudad persisted?
Did Supabase update succeed?
Did Router F receive previous state?
Did structured extraction return null?
Did merge logic overwrite a valid value?
```

Expected behavior:

```text
Existing value
+
No new replacement value
        ↓
Preserve existing value
```

Possible root causes:

```text
Persistence failure
Incorrect chat_id
AI extraction failure
Null overwrite
Wrong session loaded
State not passed into prompt
```

## 9. Short qualification answer is misinterpreted

### Symptom

During capture:

```text
Bot:
¿En qué ciudad estás?

User:
Lima
```

but the bot treats `Lima` as an unrelated message.

### Inspect

```text
fase_actual
```

Expected:

```text
fase_actual = captura
```

Persistent capture should have high priority.

Diagnostic path:

```text
Was capture state persisted?
        ↓
Did Router F read it?
        ↓
Did route remain APOSTOLADO?
        ↓
Was previous-question context available?
```

The fix should preserve contextual interpretation of short answers rather than adding special cases for every possible city or number.

## 10. Valid user correction does not update state

### Symptom

User says:

```text
"No, perdón, estoy en Iquitos"
```

but the old city remains.

### Inspect

```text
Structured [DATOS:] output
Field correction detection
Merge logic
Supabase update payload
```

Expected behavior:

```text
Old:
ciudad = Lima

Correction:
ciudad = Iquitos

Result:
ciudad = Iquitos
```

Unrelated fields should remain unchanged.

## 11. Profile correction creates inconsistent data

### Example

User was initially stored as:

```text
tipo = salon
salon = Studio Example
estilistas = 5
```

Then corrects:

```text
"En realidad soy independiente"
```

Expected behavior can require clearing fields that no longer apply:

```text
tipo = independiente
salon = null
estilistas = null
```

Inspect profile-correction and cleanup logic rather than only changing `tipo`.

The objective is to avoid contradictory state.

## 12. Multimedia is sent twice

### Symptom

A prospect receives the same catalog, pricing file or product material multiple times unnecessarily.

### Inspect

```text
multimedia_enviado
collectionHandle
lineaNombre
catalogoYaEnviado
previous Supabase update
```

Expected principle:

```text
Before delivery
        ↓
Check previous delivery state

Already sent?
    ├── YES → do not repeat unnecessarily
    └── NO  → send + persist delivery marker
```

Also verify that the Supabase update happened after the first delivery.

## 13. Multimedia should send but does not

### Inspect

```text
Classifier intent
Router mode
Product-line detection
collectionHandle
Relevant delivery flag
YCloud HTTP request
Media URL / file source
```

Break the failure into two questions:

```text
Did Router F decide to send media?
```

and:

```text
Did YCloud successfully deliver it?
```

These are different problems.

## 14. Multiple product needs are lost

### Example

```text
"Busco algo para frizz, caída y rizos"
```

### Inspect

```text
Detected first need
lineas
lineas_pendientes
Persistence
Next-turn routing
```

Expected behavior:

```text
Need A
    ↓
Handle
    ↓
Pending:
Need B
Need C
```

The system should not silently discard additional needs after responding to the first one.

## 15. RAG answer is weak or incorrect

### Symptom

The workflow executes correctly, but the technical answer is incomplete, generic or poorly grounded.

Do not assume this is immediately an LLM problem.

Inspect the retrieval chain:

```text
Question
    ↓
Embedding / semantic query
    ↓
Retrieved chunks
    ↓
Source / tipo / metadata
    ↓
AI Agent context
```

Ask:

```text
Was RAG invoked?
Were relevant chunks retrieved?
Was the source content actually sufficient?
Was chunking appropriate?
Did routing select the right knowledge context?
```

Possible causes include:

```text
Weak source document
Poor chunk boundary
Irrelevant vector result
Missing knowledge
Incorrect metadata
Wrong routing
Prompt not using retrieved context effectively
```

## 16. RAG has no answer in source material

A system should not fabricate internal knowledge simply because the user asked a question.

If the curated documents do not contain sufficient information:

```text
Source does not support answer
        ↓
Do not invent internal facts
```

Depending on the commercial scenario, the assistant may:

```text
Provide a limited answer
Ask a clarifying question
Recommend human follow-up
```

Knowledge gaps should be treated as knowledge-maintenance issues rather than hidden through hallucination.

## 17. Existing customer enters new-lead qualification

### Symptom

A known Davines customer is asked new-prospect qualification questions or receives introductory material unnecessarily.

### Inspect

```text
CLIENTE_ACTUAL detection
Existing-customer data
Router interceptor
Session state
```

Expected principle:

```text
Known / existing client
        ↓
CONVERSACION

not

New-prospect APOSTOLADO
```

Also verify that automatic catalog or price delivery is not unnecessarily restarted.

## 18. B2C user enters B2B flow

### Symptom

A consumer looking for personal use is treated like a salon prospect.

### Inspect

```text
B2C classification
Router output
systemOverride
Switch branch
```

Expected:

```text
B2C intent
    ↓
B2C route / redirect
```

This protects both user experience and CRM quality.

## 19. Human handoff does not persist

### Symptom

The assistant indicates that a salesperson will follow up, but the next message restarts automated qualification.

### Inspect

```text
estado
asesor_solicitado
bot_pausado
```

Expected:

```text
estado = HUMANO
asesor_solicitado = true
bot_pausado = true
```

Also inspect downstream persistence logic after the handoff message.

A conversational phrase alone is not enough.

The state transition must be persisted.

## 20. Bot remains active after human handoff

### Symptom

A prospect has already been transferred to sales, but the automated acquisition flow continues as if nothing changed.

### Inspect

```text
postHandoff
bot_pausado
asesor_solicitado
estado
Router F
```

Expected detection:

```text
bot_pausado = true
OR
asesor_solicitado = true
OR
estado = HUMANO
        ↓
post-handoff handling
```

Do not fix this only through prompt wording.

The source of truth should remain persisted application state.

## 21. CRM record is missing

### Symptom

Qualification works but no expected Google Sheets record appears.

### Inspect

```text
Was structured data extracted?
Did Post Agente F5-F execute?
Did Sheets branch execute?
Was the target sheet available?
Was the row append/update successful?
```

Separate:

```text
No data generated
```

from:

```text
Data generated but Sheets write failed
```

## 22. CRM contains incomplete information

Incomplete CRM information is not automatically an error.

The production system intentionally supported progressive qualification.

A record such as:

```text
INTERÉS INICIAL
```

or:

```text
LEAD PARCIAL
```

can represent a real prospect who stopped responding before completing every field.

Before treating missing fields as a bug, determine:

```text
Did the customer actually provide the information?
```

If no:

```text
Expected incomplete lead
```

If yes:

```text
Inspect extraction / persistence
```

## 23. CRM field is wrong

### Diagnostic path

```text
Original user statement
        ↓
AI structured extraction
        ↓
Post-processing
        ↓
Supabase
        ↓
CRM transformation
```

Find the first layer where the value becomes incorrect.

Do not manually patch the spreadsheet and assume the system is fixed.

If the same error can happen again, fix the upstream transformation.

## 24. Duplicate lead behavior

When a prospect returns after previous activity, inspect:

```text
chat_id
Existing Supabase session
Previous CRM record
Handoff state
Current lead status
```

The correct behavior may depend on whether the prospect is:

```text
continuing an active conversation
returning after abandonment
already handed to sales
an existing customer
```

Avoid assuming every new message represents a new lead.

## 25. State differs between Supabase and CRM

This is not automatically an error.

The systems serve different responsibilities.

```text
Supabase
    ↓
Live application state

Google Sheets
    ↓
Human-reviewed commercial operations
```

The CRM may contain:

```text
normalized values
human corrections
additional operational interpretation
```

while Supabase reflects application state at a particular workflow moment.

Investigate mismatches when they affect behavior, but do not require row-for-row equality.

## 26. Supabase state inspection

For a problematic session, relevant fields include:

```text
chat_id
fase_actual
estado_conversacion
estado
bot_pausado
asesor_solicitado

tipo
salon
ciudad
distrito
estilistas
telefono
correo
contacto
marca
driver
horario

lineas
lineas_pendientes
multimedia_enviado
```

The purpose is to reconstruct:

```text
What did the application believe before this message?
```

before analyzing the resulting route.

## 27. Debugging `Router F`

Useful router debug outputs include:

```text
DEBUG_intencion
DEBUG_intencionFinal
DEBUG_esPrimerMensaje
DEBUG_catalogoYaEnviado
DEBUG_lineaNombre
DEBUG_lineasPendientes
```

A routing bug should be explained as:

```text
Input
        ↓
Initial intent
        ↓
Previous state
        ↓
Final route
```

rather than only:

```text
"The bot answered wrong."
```

This makes regression testing possible.

## 28. Safe production changes

Avoid making several unrelated fixes simultaneously.

Preferred sequence:

```text
Reproduce bug
        ↓
Identify layer
        ↓
Make smallest reasonable change
        ↓
Test affected path
        ↓
Test neighboring paths
        ↓
Deploy
        ↓
Observe
```

Changing:

```text
classifier
router
prompt
Supabase logic
multimedia
```

all at the same time makes root-cause validation difficult.

## 29. Regression testing after a fix

Every important production fix should be tested against:

```text
Original failing case
        +
Normal expected case
        +
Nearby edge cases
```

Example:

```text
Bug:
"Lima" during capture was misrouted.

After fix test:

1. "Lima" during capture
2. "Lima" outside capture
3. city correction
4. short number response
5. normal technical question during capture
```

The goal is to fix the bug without damaging another route.

## 30. Test the complete user journey

A workflow can appear healthy when testing individual nodes while still failing end-to-end.

Important regression paths include:

```text
New prospect
Existing customer
B2C consumer
Product question
Technical question
Price request
Multiple needs
Audio
Qualification
Correction
Human handoff
Post-handoff message
```

Testing should occur from actual inbound behavior whenever possible rather than only executing isolated internal nodes.

## 31. Production authentication changes

When changing credentials:

```text
Create replacement credential
        ↓
Attach to affected nodes
        ↓
Test inbound / outbound paths
        ↓
Verify successful requests
        ↓
Only then revoke old credential
```

Avoid rotating authentication while simultaneously changing routing logic.

Separate infrastructure changes from conversational behavior changes when possible.

## 32. Do not expose secrets while debugging

Useful diagnostic evidence:

```text
Status code
Node name
Error category
Missing header name
Payload shape
Execution path
```

Unnecessary and unsafe evidence:

```text
Full API key
Bearer token
Database password
Customer phone
Full private webhook URL
```

Example:

```text
GOOD:
401 — X-API-Key missing

BAD:
401 — attempted key abc123-real-secret
```

## 33. Production evidence

Useful evidence when documenting a bug includes:

```text
Date
Expected behavior
Observed behavior
Affected route
Root cause
Fix
Regression tests
Result
```

For example:

```text
Issue:
YCloud outbound request returned 401

Expected:
Message delivered

Observed:
X-API-Key missing

Root cause:
Authentication configuration

Fix:
Move authentication to correct n8n Header Auth credential

Regression:
Test affected outbound paths

Result:
Successful delivery
```

This format is much more useful than storing random screenshots without context.

## 34. Incident template

Use the following structure for significant production issues:

```text
Incident:
[Short name]

Date:
[YYYY-MM-DD]

User scenario:
[What the customer was doing]

Expected:
[Correct behavior]

Observed:
[Failure]

Layer:
[Channel / classifier / routing / state / AI / RAG / CRM / outbound]

Root cause:
[Confirmed cause]

Fix:
[What changed]

Regression tests:
[Cases tested]

Result:
[Pass / remaining issue]
```

## 35. Monitoring mindset

The system should be observed across both technical and commercial signals.

Technical indicators:

```text
Webhook failures
HTTP errors
Authentication errors
Execution failures
Supabase write failures
Sheets write failures
Transcription failures
```

Behavioral indicators:

```text
Repeated questions
Wrong qualification route
Duplicate multimedia
Unexpected B2C traffic
Lost state
Poor RAG answer
Broken handoff
```

Commercial indicators:

```text
Incomplete qualification
Unexpected lead quality
Missing advisor summaries
CRM inconsistencies
Handoff problems
```

A technically successful HTTP response does not guarantee commercially correct behavior.

## 36. Troubleshooting decision tree

```text
Customer reports problem
        ↓
Did n8n execute?
        │
        ├── NO
        │    ↓
        │ YCloud / webhook
        │
        └── YES
              ↓
       Was input normalized?
              │
              ├── NO
              │    ↓
              │ Text / audio preprocessing
              │
              └── YES
                    ↓
             Correct intent?
                    │
                    ├── NO
                    │    ↓
                    │ Clasificador F
                    │
                    └── YES
                          ↓
                   Correct state?
                          │
                          ├── NO
                          │    ↓
                          │ Supabase / persistence
                          │
                          └── YES
                                ↓
                         Correct route?
                                │
                                ├── NO
                                │    ↓
                                │ Router F
                                │
                                └── YES
                                      ↓
                              Correct knowledge?
                                      │
                                      ├── NO
                                      │    ↓
                                      │ RAG / source docs
                                      │
                                      └── YES
                                            ↓
                                   Correct output generated?
                                            │
                                            ├── NO
                                            │    ↓
                                            │ AI / prompt
                                            │
                                            └── YES
                                                  ↓
                                         Response delivered?
                                                  │
                                                  ├── NO
                                                  │    ↓
                                                  │ YCloud outbound
                                                  │
                                                  └── YES
                                                        ↓
                                                Check CRM / handoff
```

## 37. Recovery philosophy

The production system should prefer:

```text
Diagnose
        ↓
Fix
        ↓
Test
        ↓
Observe
```

rather than:

```text
Something failed
        ↓
Add more prompt
        ↓
Hope
```

Not every conversational failure is an AI problem.

Many failures belong to:

```text
State
Routing
Authentication
Data transformation
Integration configuration
```

Identifying the correct layer is one of the most important operational skills in a multi-service AI system.

## Engineering lesson

The main production lesson from this project is that debugging a conversational system requires visibility across the entire application.

A customer sees:

```text
"The bot answered incorrectly."
```

but the actual root cause could be:

```text
Wrong inbound data
Wrong intent
Stale state
Incorrect route
Weak retrieval
Extraction failure
Failed database update
Failed API request
Broken handoff
```

Production reliability therefore depends on tracing the full decision path rather than treating the AI model as a black box.

## Privacy

Production debugging evidence should not expose customer conversations, phone numbers, emails, credentials or private infrastructure in public documentation.

Public incident examples should use sanitized or synthetic data.