# Integrations and System Responsibilities

The B2B WhatsApp system was built as a multi-service architecture rather than as a single chatbot.

Each integration had a specific responsibility.

The production stack included:

```text
Meta Ads
YCloud / WhatsApp
n8n
OpenAI
Supabase
PostgreSQL-backed chat memory
Google Drive
Google Sheets
```

The objective was to keep conversational AI, workflow control, application state, knowledge retrieval and commercial operations separated.

```text
Acquisition
    ↓
WhatsApp channel
    ↓
Workflow orchestration
    ↓
AI + routing + state + knowledge
    ↓
CRM / human sales
```

## High-level architecture

```text
Meta Ads
    ↓
WhatsApp
    ↓
YCloud
    ↓
n8n Webhook
    ↓
Input normalization
    │
    ├── Text
    └── Audio → transcription
    ↓
Clasificador F
    ↓
Supabase session lookup
    ↓
Router F
    ↓
Execution mode
    │
    ├── Conversation
    ├── RAG
    ├── Multimedia
    ├── Qualification
    ├── B2C redirect
    └── Human escalation
    ↓
AI Agent
    ↓
Structured data extraction
    ↓
Supabase persistence
    ↓
Google Sheets CRM
    ↓
Human review
    ↓
Commercial team
```

## Integration responsibilities

The production architecture intentionally avoids assigning every responsibility to the language model.

| Integration | Main responsibility |
|---|---|
| Meta Ads | Acquisition and traffic generation |
| YCloud | WhatsApp messaging transport |
| n8n | Workflow orchestration and business logic |
| OpenAI | Language reasoning, embeddings and audio transcription |
| Supabase | Persistent application state and vector knowledge |
| PostgreSQL chat memory | Conversational history |
| Google Drive | Knowledge / document source storage |
| Google Sheets | Operational CRM and commercial review |

## Meta Ads — acquisition layer

Meta Ads operated upstream from the automation.

Campaigns brought professional prospects into the WhatsApp channel.

```text
Paid media
    ↓
Ad creative
    ↓
Professional prospect
    ↓
WhatsApp
    ↓
B2B conversational system
```

The automation did not replace acquisition.

It handled what happened after the prospect entered WhatsApp.

This distinction is important because commercial performance depended on several components:

```text
Paid media
+
Creative
+
Conversational automation
+
Qualification
+
Human sales
```

Meta Ads execution was shared within the marketing team rather than being exclusively owned by the automation layer.

## YCloud — WhatsApp transport layer

YCloud provided the WhatsApp messaging interface used by the workflow.

The integration allowed n8n to receive and send WhatsApp interactions programmatically.

Its role included:

```text
Inbound WhatsApp events
        ↓
Webhook

Outbound text
        ↓
WhatsApp message

Outbound multimedia
        ↓
PDF / image / video / other material
```

YCloud therefore acted as the transport layer between the customer and the workflow.

It did not decide:

```text
what the customer meant
what product to recommend
whether qualification should begin
whether a human should take over
```

Those decisions were handled by the workflow.

## YCloud outbound actions

The production workflow contains multiple HTTP request nodes for WhatsApp delivery.

These are used for actions such as:

```text
Send text
Send welcome message
Send catalog / PDF
Send product material
Send additional conversational responses
```

The routing layer determines what should be sent.

YCloud executes the delivery.

Conceptually:

```text
Router decision
        ↓
Should material be sent?
        ↓
Select outbound action
        ↓
YCloud API
        ↓
WhatsApp
```

## YCloud credential management

Production authentication is handled through n8n credentials rather than relying on manually embedded API keys inside individual workflow requests.

This separation matters because secrets should be managed independently from workflow logic.

```text
Workflow logic
        ↓
references credential

Credential store
        ↓
contains authentication secret
```

This is safer and easier to maintain than duplicating authentication values across many HTTP nodes.

## n8n — orchestration layer

n8n is the central orchestration engine of the system.

It connects:

```text
WhatsApp
OpenAI
Supabase
Vector retrieval
PostgreSQL chat memory
Google Drive
Google Sheets
```

It also contains the business logic required to decide how each message should be processed.

Responsibilities include:

```text
Webhook reception
Input normalization
Audio handling
Intent classification
State retrieval
Routing
Qualification
RAG invocation
Multimedia decisions
Structured extraction
State persistence
CRM updates
Human-handoff logic
Outbound messaging
```

The system therefore uses n8n as an application orchestration layer rather than only as a simple automation connector.

## n8n Webhook

The workflow begins with an inbound webhook.

Conceptually:

```text
WhatsApp event
        ↓
YCloud
        ↓
n8n Webhook
        ↓
Extract relevant message information
```

The workflow then normalizes the input before classification and routing.

This allows later nodes to work with a consistent representation of the customer message.

## Text and audio normalization

WhatsApp users do not always communicate through text.

The workflow includes audio handling.

```text
Incoming message
        │
        ├── Text
        │     ↓
        │ Normalize message
        │
        └── Audio
              ↓
          Download recording
              ↓
          Transcription
              ↓
          Normalized text
```

After normalization, both message types can continue through the same classification and routing architecture.

This prevents voice messages from requiring an entirely separate commercial process.

## OpenAI — conversational reasoning

OpenAI is used for the flexible language component of the system.

The production AI Agent is responsible for tasks such as:

```text
Natural conversation
Contextual interpretation
Technical explanations
Commercial communication
Structured field extraction
Handling messy user language
```

The language model does not independently control the complete application.

Its behavior is constrained by:

```text
Router F
Supabase state
System instructions
Retrieved knowledge
Previously captured data
```

This keeps the language model inside a broader deterministic workflow.

## OpenAI — embeddings

OpenAI embedding models are also used in the RAG ingestion and retrieval architecture.

```text
Document chunk
        ↓
OpenAI Embeddings
        ↓
Vector
        ↓
Supabase Vector Store
```

These vectors allow semantic retrieval from the curated Davines knowledge base.

Embeddings therefore solve a different problem from the conversational AI model.

```text
Chat model
        ↓
Generate / interpret language

Embedding model
        ↓
Represent semantic meaning for retrieval
```

## OpenAI — audio transcription

The workflow also uses transcription for audio messages.

Conceptually:

```text
WhatsApp audio
        ↓
Download file
        ↓
Transcription
        ↓
Text
        ↓
Normal routing pipeline
```

This allows a prospect using voice notes to enter the same stateful commercial system as a prospect typing messages.

## Supabase — application state

Supabase acts as the persistent state layer.

The main session table is:

```text
sesiones_bot
```

It stores information required to make deterministic decisions across independent WhatsApp messages.

Examples include:

```text
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

Conceptually:

```text
New WhatsApp message
        ↓
Read sesiones_bot
        ↓
Understand existing application state
        ↓
Route current message
        ↓
Extract new information
        ↓
Update sesiones_bot
```

Supabase therefore acts as application memory rather than only as a database destination.

## Supabase CRUD operations

The production workflow contains Supabase operations used to:

```text
Find existing session
Create new session
Update session
Persist qualification data
Persist routing state
Persist handoff state
Persist multimedia state
```

The workflow can therefore preserve commercial context between independent webhook executions.

Without this layer, each incoming WhatsApp message would effectively begin with much less structured application context.

## Supabase — vector knowledge

Supabase also serves a separate responsibility through its vector store.

The `documents` table stores:

```text
content
source
tipo
embedding
metadata
```

This is conceptually separate from:

```text
sesiones_bot
```

The distinction is:

```text
sesiones_bot
        ↓
What do we know about THIS conversation?


documents
        ↓
What does the SYSTEM know about Davines?
```

Both are stored in Supabase, but they solve fundamentally different problems.

## PostgreSQL-backed chat memory

The production workflow includes persistent chat memory through a PostgreSQL chat-memory node.

This layer stores natural conversational history.

Its purpose is different from structured Supabase session state.

```text
Chat memory
        ↓
What was said?


Supabase session state
        ↓
What business state should the application preserve?
```

For example:

```text
Conversation:
"I have a salon in Lima and I'm interested in OI."

Chat memory can preserve:
the conversational exchange.

Supabase can preserve:
ciudad = Lima
tipo = salon
lineas = OI
fase_actual = ...
```

This separation reduces the need to infer critical business state repeatedly from raw conversation history.

## Three different memory layers

The system therefore has at least three conceptually separate forms of persistent information:

```text
1. Conversational memory
   PostgreSQL chat history

2. Operational state
   Supabase sesiones_bot

3. Domain knowledge
   Supabase documents / Vector Store
```

Each answers a different question.

```text
Conversation memory:
"What have we talked about?"

Operational state:
"What does the workflow already know and what should happen?"

RAG:
"What relevant Davines knowledge should be retrieved?"
```

This separation is one of the core architectural decisions of the project.

## Google Drive — knowledge source layer

Google Drive is used as a source location for internal knowledge documents.

The ingestion flow includes:

```text
Google Drive
        ↓
Retrieve file
        ↓
Extract text
        ↓
Data Loader
        ↓
Text Splitter
        ↓
Embeddings
        ↓
Supabase Vector Store
```

The source documents include separate forms of knowledge such as:

```text
Technical knowledge
Commercial B2B knowledge
Qualification framework
```

Google Drive therefore serves as source storage.

It is not the runtime semantic search engine.

Once processed, searchable chunks are stored in the vector knowledge layer.

## Google Sheets — operational CRM

Google Sheets functions as the lightweight operational CRM used after conversational processing.

Its role is distinct from Supabase.

```text
Supabase
        ↓
Machine / application state


Google Sheets
        ↓
Human-readable commercial operations
```

The CRM provides the team with structured information such as:

```text
TipoPerfil
EstadoLead
Nombre Salon
Distrito
Ciudad
Telefono
TelefonoFuente
Correo
Lineas
Prioridad
mensajeAsesor
Obs
CanalPreferido
MarcaActual
Driver
Estilistas
```

This gives marketing and sales an accessible commercial representation of the opportunity.

## Why Sheets was not replaced by Supabase

Technically, Supabase could store commercial data.

However, the operational requirement was not only storage.

The team needed to:

```text
inspect leads quickly
review information visually
correct small details
prioritize opportunities
share information internally
prepare records for sales
```

Google Sheets matched the team's existing operational habits.

The architecture therefore separates:

```text
machine-optimized application state
        ↓
Supabase

human-optimized operational review
        ↓
Google Sheets
```

The choice prioritized adoption and operational usability.

## Existing-customer data

The system also used structured information about existing Davines professional clients / salons.

This allowed the workflow to differentiate an existing relationship from a completely new prospect.

Conceptually:

```text
Incoming professional contact
        ↓
Existing-client context available?
        │
        ├── YES
        │     ↓
        │ CLIENTE_ACTUAL behavior
        │
        └── NO
              ↓
        Normal prospect journey
```

This information should be treated as operational customer data rather than public RAG material.

## End-to-end data movement

A simplified example illustrates how the integrations interact.

### Step 1 — acquisition

```text
Meta Ads
        ↓
Professional prospect
        ↓
WhatsApp
```

### Step 2 — channel

```text
WhatsApp
        ↓
YCloud
        ↓
n8n Webhook
```

### Step 3 — normalization

```text
Text
        ↓
Normalized message

OR

Audio
        ↓
OpenAI transcription
        ↓
Normalized message
```

### Step 4 — initial understanding

```text
Normalized message
        ↓
Clasificador F
        ↓
Initial intent
```

### Step 5 — state retrieval

```text
Supabase
        ↓
Existing session
        ↓
Current business state
```

### Step 6 — routing

```text
Intent
+
State
+
Commercial rules
        ↓
Router F
        ↓
Execution mode
```

### Step 7 — knowledge

If the message requires domain knowledge:

```text
Question
        ↓
Supabase Vector Store
        ↓
Relevant chunks
        ↓
AI Agent
```

### Step 8 — conversation

```text
AI Agent
        ↓
Natural-language response
        +
Structured [DATOS:]
```

### Step 9 — persistence

```text
Structured data
        ↓
Merge with previous state
        ↓
Supabase
```

### Step 10 — operational CRM

When appropriate:

```text
Structured commercial information
        ↓
Google Sheets
        ↓
Human review
```

### Step 11 — response

```text
Router / AI output
        ↓
n8n
        ↓
YCloud
        ↓
WhatsApp
```

### Step 12 — human sales

When escalation is appropriate:

```text
Qualified commercial context
        ↓
CRM / human review
        ↓
Sales team
```

## Integration boundaries

One of the strengths of the architecture is that responsibilities remain relatively separated.

```text
YCloud
does not decide qualification.

OpenAI
does not own application state.

Supabase
does not generate natural-language responses.

Google Sheets
does not control routing.

Google Drive
does not serve as live conversation memory.

n8n
coordinates all of them.
```

This makes the system easier to reason about and debug.

## Read / write responsibilities

A simplified responsibility matrix is:

| System | Reads | Writes |
|---|---|---|
| YCloud | n8n outbound request | WhatsApp messages |
| n8n | All connected sources | Workflow actions |
| OpenAI Chat | Prompt + context | Language / structured output |
| OpenAI Embeddings | Text chunks | Embedding vectors |
| Supabase sessions | Session identifier | Updated application state |
| Supabase Vector Store | Semantic query | Retrieved knowledge |
| Chat memory | Session history | New messages |
| Google Drive | Source documents | Source-file maintenance |
| Google Sheets | Structured lead data | Human edits / operational review |

This table is conceptual rather than a claim that every service directly communicates with every other service.

n8n acts as the primary orchestration layer between integrations.

## Failure boundaries

Separating integrations also creates clearer debugging boundaries.

For example:

```text
Message never reaches workflow
        ↓
Inspect YCloud / webhook


Workflow receives message but wrong route
        ↓
Inspect classifier / router / Supabase state


Correct route but weak technical answer
        ↓
Inspect RAG retrieval / source knowledge


Correct response generated but not delivered
        ↓
Inspect outbound HTTP / YCloud


Qualification works but CRM missing data
        ↓
Inspect structured extraction / Sheets branch


Conversation repeats questions
        ↓
Inspect Supabase session state / merge logic
```

This is significantly easier to troubleshoot than treating the entire system as one opaque AI agent.

## Why a multi-service architecture was appropriate

No single service was ideal for every responsibility.

The architecture uses specialized tools:

```text
WhatsApp transport
        ↓
YCloud

Workflow orchestration
        ↓
n8n

Natural language
        ↓
OpenAI

Operational state
        ↓
Supabase

Semantic knowledge
        ↓
Supabase Vector Store

Conversation history
        ↓
PostgreSQL-backed memory

Source-document storage
        ↓
Google Drive

Human commercial operations
        ↓
Google Sheets
```

The system therefore follows a composable architecture rather than trying to force one platform to perform every task.

## Engineering tradeoff

A multi-service system provides flexibility but introduces integration complexity.

Potential failure points include:

```text
Authentication
API availability
Webhook payload changes
Data format mismatches
State synchronization
Rate limits
Incorrect routing
Credential configuration
```

This creates a need for:

```text
QA
debugging
observability
credential management
clear responsibility boundaries
```

The architecture should therefore be evaluated not only by how many services it connects but by whether each integration has a justified role.

## Security principles

Production integrations should not expose credentials inside public workflow artifacts.

Sensitive values include:

```text
API keys
authorization headers
database credentials
service tokens
private webhook URLs
customer identifiers
internal document IDs
```

Credentials should remain inside controlled credential stores such as n8n Credentials whenever supported.

Workflow logic can then reference the credential without hardcoding the secret.

```text
Public architecture
        ↓
can describe integration

Public workflow
        ↓
must remove secrets

Private production environment
        ↓
stores credentials
```

## Private operational data

Some integrations contain information that should remain private even when the architecture itself is documented publicly.

Examples include:

```text
WhatsApp conversation exports
Supabase session rows
Existing salon datasets
CRM customer records
Internal source documents
Customer phone numbers
Emails
Private Google Drive files
```

These artifacts can be preserved privately as implementation evidence while remaining outside the public repository.

## Final integration architecture

The complete integration map can be summarized as:

```text
                       META ADS
                           ↓
                       WHATSAPP
                           ↓
                        YCLOUD
                           ↓
                      N8N WEBHOOK
                           ↓
                 INPUT NORMALIZATION
                    │            │
                   TEXT        AUDIO
                                  ↓
                              OPENAI
                           TRANSCRIPTION
                    │            │
                    └──────┬─────┘
                           ↓
                     CLASIFICADOR F
                           ↓
                   SUPABASE SESSION
                           ↓
                       ROUTER F
                           ↓
             ┌─────────────┼─────────────┐
             │             │             │
         CONVERSATION     RAG       QUALIFICATION
             │             │             │
             │       SUPABASE VECTOR     │
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                       AI AGENT
                           ↓
                   STRUCTURED DATA
                           ↓
                 SUPABASE PERSISTENCE
                           ↓
                GOOGLE SHEETS CRM
                           ↓
                     HUMAN REVIEW
                           ↓
                     SALES TEAM

Response path:

AI / Router
     ↓
n8n
     ↓
YCloud
     ↓
WhatsApp
```

## Engineering lesson

The most important integration lesson from the project is that building a production conversational system is not primarily about connecting many APIs.

It is about assigning the correct responsibility to each component.

The architecture separates:

```text
Acquisition
Channel transport
Workflow orchestration
Natural-language intelligence
Application state
Conversation memory
Domain knowledge
Operational CRM
Human sales
```

This separation allowed the system to combine AI flexibility with deterministic business operations.

The result is not simply an AI chatbot connected to WhatsApp.

It is a multi-layer B2B conversational system connecting acquisition, product education, qualification, structured state, knowledge retrieval and human commercial execution.

## Privacy and publication

This document describes system responsibilities and data movement without exposing production credentials, customer records or private infrastructure identifiers.

Any workflow, screenshot or configuration published externally should be sanitized before release.