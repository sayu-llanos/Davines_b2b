# Conversation Flow and State Transitions

This diagram summarizes how the B2B WhatsApp system moves between exploration, qualification and human commercial follow-up.

The routing decision is based on:

```text
Current message
+
Intent classification
+
Previous Supabase state
+
Commercial context
+
Previously delivered material
+
Handoff state
```

```mermaid
flowchart TD

    A[Incoming WhatsApp Message]

    B[Normalize Input]
    C[Clasificador F]

    D{Existing Human Handoff?}

    E[Post-Handoff Context]

    F{Customer Type / Intent}

    G[B2C]
    H[Existing Davines Client]
    I[New / Active B2B Prospect]

    J{Commercial Intent?}

    K[Exploration State]
    L[Capture State]

    M{Question Type}

    N[General Information]
    O[Technical / Product Question]
    P[Price Request]
    Q[Multimedia Request]
    R[Multiple Product Needs]

    S[RAG Retrieval]
    T[AI Agent Response]

    U{Material Already Sent?}
    V[Send Relevant Material]
    W[Continue Conversation]

    X[Detect Professional Profile]
    Y[Collect Qualification Fields]

    Z{Enough Commercial Context?}

    AA[Continue Progressive Qualification]
    AB[Prepare Human Handoff]

    AC[Persist State]
    AD[(Supabase sesiones_bot)]

    AE[Operational CRM]
    AF[Human Review]
    AG[Sales Team]

    AH[B2C Redirect / Consumer Path]
    AI[Existing Client Conversation]

    A --> B
    B --> C
    C --> D

    D -->|Yes| E
    D -->|No| F

    E --> T

    F -->|B2C| G
    F -->|CLIENTE_ACTUAL| H
    F -->|B2B| I

    G --> AH
    H --> AI

    I --> J

    J -->|No / exploratory| K
    J -->|Yes| L

    K --> M

    M -->|General info| N
    M -->|Technical / product| O
    M -->|Prices| P
    M -->|Multimedia| Q
    M -->|Multiple needs| R

    N --> T

    O --> S
    S --> T

    P --> U
    Q --> U

    U -->|No| V
    U -->|Yes| W

    V --> T
    W --> T

    R --> T

    T --> AC
    AC --> AD

    L --> X
    X --> Y

    Y --> Z

    Z -->|Not yet| AA
    Z -->|Ready / advisor needed| AB

    AA --> AC

    AB --> AC
    AB --> AE

    AE --> AF
    AF --> AG
```

## Main conversation phases

The production workflow uses two main commercial phases:

```text
exploracion
captura
```

and a separate human-ownership state:

```text
HUMANO
```

These represent different responsibilities.

### Exploration

```text
fase_actual = exploracion
```

The user may still be:

- learning about Davines
- asking product questions
- comparing professional lines
- requesting technical information
- requesting pricing
- receiving catalogs or multimedia
- clarifying whether Davines fits their business

The goal is not to force qualification too early.

### Capture

```text
fase_actual = captura
```

The conversation has moved into structured commercial qualification.

Possible fields include:

```text
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
```

The assistant collects this information progressively across multiple turns.

### Human ownership

```text
estado = HUMANO
asesor_solicitado = true
bot_pausado = true
```

This represents commercial escalation.

The important distinction is:

```text
AI says "an advisor will contact you"
```

is not enough.

The workflow must also persist the handoff state.

---

## Exploration path

A new B2B prospect normally begins in exploration.

```mermaid
flowchart LR

    A[New Prospect]
    B[Brand / Product Question]
    C[Product Education]
    D[Commercial Interest]
    E[Qualification]

    A --> B
    B --> C
    C --> D
    D --> E
```

Not every conversation reaches qualification.

A prospect may:

```text
Ask one question
Receive information
Stop responding
```

That is still a valid conversational outcome.

---

## Qualification path

When commercial intent becomes stronger:

```text
TRABAJAR_MARCA
advisor request
visit request
professional onboarding interest
strong B2B buying intent
```

the system can move toward:

```text
APOSTOLADO
        ↓
captura
```

Conceptually:

```mermaid
flowchart TD

    A[Commercial Signal]
    B[APOSTOLADO]
    C[Read Existing State]
    D[Ask Only Missing Information]
    E[Persist New Data]
    F{Qualification Complete?}
    G[Continue Capture]
    H[Human Handoff]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F -->|No| G
    G --> D
    F -->|Yes / advisor needed| H
```

---

## Short-answer handling

During capture, short answers need to be interpreted using conversation state.

Example:

```text
Bot:
¿En qué ciudad estás?

User:
Lima
```

Outside qualification, `"Lima"` may have little meaning.

Inside capture:

```text
fase_actual = captura
+
previous question = city
        ↓
ciudad = Lima
```

This is why persistent state has higher importance than isolated-message interpretation.

---

## Correction handling

Users can change previously supplied information.

Example:

```text
User:
Estoy en Lima.

Later:
Perdón, estoy en Trujillo.
```

Expected transition:

```text
ciudad = Lima
        ↓
correction detected
        ↓
ciudad = Trujillo
```

The new value should replace the old value without deleting unrelated qualification fields.

---

## Profile correction

Changing professional profile may require clearing incompatible information.

Example:

```text
Old state:

tipo = salon
salon = Studio Example
estilistas = 6
```

User correction:

```text
"En realidad trabajo de manera independiente."
```

Expected:

```text
tipo = independiente
salon = null
estilistas = null
```

The goal is consistent state rather than preserving contradictory fields.

---

## Existing client path

Existing Davines clients should not be treated as brand-new acquisition prospects.

```mermaid
flowchart LR

    A[Incoming User]
    B{Existing Client?}
    C[CLIENTE_ACTUAL]
    D[Continue Client Conversation]
    E[New Prospect Flow]

    A --> B
    B -->|Yes| C
    C --> D
    B -->|No| E
```

This prevents unnecessary:

```text
introductory qualification
catalog repetition
price repetition
new-lead treatment
```

---

## B2C separation

The B2B workflow also needs to detect consumer traffic.

```mermaid
flowchart LR

    A[Incoming User]
    B{Professional B2B?}
    C[B2B Journey]
    D[B2C Path / Redirect]

    A --> B
    B -->|Yes| C
    B -->|No| D
```

This protects CRM quality and prevents consumers from entering professional qualification.

---

## Price behavior

Price requests are state-aware.

```mermaid
flowchart TD

    A[PRECIOS]
    B{Price material already sent?}
    C[Send price material]
    D[Continue conversational response]
    E[Persist multimedia state]

    A --> B
    B -->|No| C
    C --> E
    E --> D
    B -->|Yes| D
```

This reduces unnecessary repeated file delivery.

---

## Multimedia behavior

The same principle applies to catalogs, PDFs, images and videos.

```text
User request
        ↓
Relevant material?
        ↓
Already sent?
        │
        ├── No → send
        └── Yes → continue without unnecessary repetition
```

Delivery state is persisted using:

```text
multimedia_enviado
```

---

## RAG path

Technical and product questions can invoke knowledge retrieval.

```mermaid
flowchart LR

    A[Technical Question]
    B[Detect Product / Hair Need]
    C[Semantic Retrieval]
    D[Relevant Knowledge Chunks]
    E[AI Agent]
    F[Grounded Response]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
```

RAG provides domain context.

It does not decide the commercial state machine.

---

## Multiple needs

A customer can express more than one product need in the same message.

Example:

```text
"I need something for frizz, curls and hair loss."
```

The workflow can track:

```text
current need
+
lineas_pendientes
```

Conceptually:

```text
Need 1
    ↓
Answer / recommend
    ↓
Pending Need 2
    ↓
Answer / recommend
    ↓
Pending Need 3
```

This prevents secondary needs from disappearing after the first recommendation.

---

## Human handoff

Handoff occurs when the conversation should move from automated qualification toward commercial ownership.

```mermaid
stateDiagram-v2

    [*] --> Exploration

    Exploration --> Exploration: Product / brand questions
    Exploration --> Capture: Strong commercial intent

    Capture --> Capture: Progressive qualification
    Capture --> Human: Advisor handoff

    Exploration --> Human: Direct advisor request

    Human --> Human: Post-handoff messages
```

Observed production data showed both:

```text
captura → HUMANO
```

and:

```text
exploracion → HUMANO
```

which means some conversations escalated directly without completing the idealized capture path.

The documentation preserves this observed behavior rather than forcing every session into a theoretical funnel.

---

## State transition summary

```text
NEW
 ↓
EXPLORACION
 │
 ├── General information
 ├── Product education
 ├── RAG
 ├── Pricing
 ├── Multimedia
 └── Commercial intent
          ↓
       CAPTURA
          │
          ├── profile
          ├── location
          ├── contact
          ├── current brand
          ├── commercial driver
          └── product interests
                   ↓
                HUMANO
                   ↓
             CRM + Sales
```

Parallel exits:

```text
B2C
    ↓
Consumer path

CLIENTE_ACTUAL
    ↓
Existing-client conversation
```

---

## Core engineering principle

Conversation flow is not controlled by the language model alone.

```text
Message meaning
        +
Persistent state
        +
Deterministic rules
        +
Commercial context
        ↓
Behavior
```

This prevents the application from depending entirely on whether the model "remembers" what should happen next.

---

## Privacy

This diagram uses synthetic examples and architectural labels only.

No customer conversations, phone numbers, emails, credentials or private production identifiers are included.
