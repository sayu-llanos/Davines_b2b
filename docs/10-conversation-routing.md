# Conversation Classification and Routing

The B2B assistant uses a hybrid routing architecture that separates conversational understanding from business-critical workflow decisions.

Instead of sending every incoming WhatsApp message directly to a language model and allowing the model to decide the complete process, the production workflow separates two responsibilities:

```text
Incoming WhatsApp message
        ↓
Message normalization
        ↓
Clasificador F
"What is the user trying to do?"
        ↓
Persisted Supabase state
        +
previous conversation context
        ↓
Router F
"What should the system do now?"
        ↓
Execution mode
        ↓
AI / RAG / multimedia / qualification / redirect
```

This distinction became important as the system evolved from a simple conversational bot into a stateful B2B sales application.

A user's current message alone is often not enough to determine the correct action.

For example:

```text
"Lima"
```

could be meaningless in isolation.

However, if the previous question asked for the prospect's city and the session is already in qualification mode, the same message is a valid commercial answer.

The routing layer therefore combines:

```text
Current message
+
detected intent
+
conversation phase
+
previously captured data
+
previously delivered material
+
commercial signals
+
handoff state
```

before deciding what should happen next.

## Layer 1 — Clasificador F

`Clasificador F` performs the first deterministic interpretation of the normalized user message.

The production classifier starts with a general fallback and then evaluates known patterns related to product needs, brand information, pricing, professional interest, B2C traffic and other important commercial situations.

Observed intent values in the production workflow include:

```text
INFORMACION
INFO_GENERAL
MULTIMEDIA
PRECIOS
TRABAJAR_MARCA
B2C
CLIENTE_ACTUAL
CADENA_DISTRIBUIDOR
SOPORTE
CONVERSACION_INVALIDA
```

The purpose of this layer is not to generate the final response.

Its purpose is to provide an initial interpretation that the router can later combine with state.

## Centralized detection of professional hair needs

The classifier contains a centralized detector for professional hair needs and common natural-language variations.

Examples include language related to:

```text
frizz
hair loss
dandruff
dryness
dehydration
damage
breakage
color-treated hair
decoloration
curls
waves
volume
oiliness
sensitive scalp
reconstruction
styling
shine
texture
```

This allows the system to recognize how real customers describe problems rather than requiring them to use exact Davines terminology.

For example:

```text
"Mis clientas tienen mucho frizz"
        ↓
Need detected
        ↓
Product context can be selected
        ↓
Relevant technical / commercial handling
```

## Product-line recognition

The routing logic also recognizes Davines product lines and common variations of their names.

Examples represented in the production workflow include:

```text
OI
MOMO
DEDE
MELU
MINU
NOUNOU
NOURISHING
REPLUMPING
REBALANCING
CALMING
LOVE CURL
LOVE SMOOTHING
NATURALTECH
MORE INSIDE
ENERGIZING
PURIFYING
DETOXIFYING
VIEW
THE CENTURY OF LIGHT
```

The prospect therefore does not need to navigate a rigid product menu.

The system can recognize either:

```text
Exact product line
```

or:

```text
Natural-language hair need
```

and use that information during routing.

## Classification is not routing

The architecture deliberately separates `Clasificador F` from `Router F`.

The classifier answers:

```text
What does this message appear to mean?
```

The router answers:

```text
Given what we already know about this conversation,
what should the system do now?
```

This distinction is important.

The same detected intent can produce different behavior depending on state.

```text
Intent
        +
Supabase state
        +
previous delivery state
        +
commercial context
        ↓
Final route
```

## Layer 2 — Router F

`Router F` is the main orchestration layer.

It receives the initial intent, reads persisted session information and applies additional business rules before producing the final execution mode.

The router can output modes such as:

| Mode | Role |
|---|---|
| `CONVERSACION` | General contextual conversation |
| `INFO_GENERAL` | General Davines brand information |
| `MULTIMEDIA` | Product / line information and related material |
| `APOSTOLADO` | Structured B2B qualification |
| `B2C` | Redirect consumer traffic away from the professional flow |
| `ALERTA_CADENA` | Separate handling for chains or large-scale distribution |

The resulting `systemOverride` is then used by the downstream workflow to select the appropriate behavior.

## Simplified intent-to-route mapping

The production routing logic contains mappings such as:

```text
B2C
        → B2C

CADENA_DISTRIBUIDOR
        → ALERTA_CADENA

INFO_GENERAL + first interaction
        → INFO_GENERAL

INFO_GENERAL after initial interaction
        → CONVERSACION

SOPORTE
        → CONVERSACION

TRABAJAR_MARCA
        → APOSTOLADO

MULTIMEDIA + strong commercial intent
        → APOSTOLADO

MULTIMEDIA without commercial escalation
        → MULTIMEDIA

INFORMACION
        → CONVERSACION
```

This means product interest and commercial qualification are intentionally separated.

A person can learn about Davines without immediately being forced into a lead form.

## First-message awareness

The router checks whether the conversation already exists in the stored session data.

This allows the system to distinguish an initial interaction from an established conversation.

For example, general brand information can be treated differently on the first message than later in the conversation.

```text
First interaction
        ↓
Brand introduction may be useful

Later interaction
        ↓
Continue conversation without restarting introduction
```

This prevents the assistant from repeatedly behaving as if every message were the beginning of a new chat.

## Price requests receive high-priority handling

Pricing is treated as a special case in the production router.

The system checks whether pricing material has already been delivered by reading `multimedia_enviado`.

Conceptually:

```text
User asks for prices
        ↓
Check Supabase
        ↓
Was pricing material already sent?
        │
        ├── NO
        │    ↓
        │ MULTIMEDIA
        │    ↓
        │ Deliver pricing material
        │
        └── YES
             ↓
          CONVERSACION
```

This prevents the workflow from repeatedly sending the same pricing document whenever the user mentions prices again.

The design therefore combines intent detection with delivery memory.

## Existing-customer interceptor

The production workflow includes an early interceptor for:

```text
CLIENTE_ACTUAL
```

An existing customer is routed to:

```text
CONVERSACION
```

rather than being treated as a new B2B prospect.

The interceptor explicitly avoids restarting behavior such as:

```text
new-prospect qualification
automatic catalog delivery
automatic price delivery
```

This is important because acquisition logic should not be blindly applied to an established client.

## Persistent capture mode

One of the most important production routing mechanisms is persistent qualification state.

When Supabase shows:

```text
fase_actual = captura
```

the workflow gives qualification behavior high priority.

The production implementation explicitly describes this as:

```text
MODO CAPTURA PERSISTENTE
```

The reason is simple: qualification often happens across many short WhatsApp messages.

For example:

```text
Bot:
¿En qué ciudad estás?

User:
Lima
```

Without state-aware routing:

```text
"Lima"
        ↓
Standalone classification
        ↓
Potential ambiguity
```

With persistent capture:

```text
"Lima"
        ↓
Read Supabase
        ↓
fase_actual = captura
        ↓
Interpret answer in qualification context
        ↓
Continue APOSTOLADO
```

This prevents valid qualification responses from being misclassified as unrelated conversation or invalid input.

## Contextual interpretation of short answers

During qualification, users frequently answer with only one word or a short phrase.

Examples include:

```text
"Sayuri"
"Lima"
"Miraflores"
"5"
"Wella"
"diferenciarme"
"sí"
"WhatsApp"
```

The qualification prompt explicitly instructs the conversational layer to interpret short responses according to the previous question.

Examples:

```text
Previous question:
What is your name?

"Sayuri"
        ↓
contacto
```

```text
Previous question:
Which city are you in?

"Lima"
        ↓
ciudad
```

```text
Previous question:
How many stylists work with you?

"5"
        ↓
estilistas
```

```text
Previous question:
Which professional brand do you currently use?

"Wella"
        ↓
marca
```

```text
Previous question:
What are you looking for from Davines?

"diferenciarme"
        ↓
driver
```

This behavior was added because production WhatsApp conversations do not resemble structured forms.

## Profile-aware qualification

The qualification flow adapts to the professional profile.

Observed profile types include:

```text
salon
independiente
apertura_proxima
cadena
```

The required questions are not identical for every profile.

For example, an independent stylist should not be repeatedly asked for:

```text
salon name
number of salon stylists
```

if those fields do not apply.

The routing and conversational instructions therefore adapt qualification according to the type of professional prospect.

## Progressive lead capture

The workflow does not require all commercial information to be obtained in one message.

Relevant fields can accumulate progressively:

```text
contacto
tipo
salon
ciudad
distrito
estilistas
correo
marca
driver
telefono
horario
```

Conceptually:

```text
Turn 1
city captured
        ↓
Turn 2
profile captured
        ↓
Turn 3
email captured
        ↓
Turn 4
current brand captured
        ↓
Turn 5
commercial motivation captured
```

The router supplies already captured information to the qualification layer so completed fields do not need to be requested again.

## Commercial-intent escalation

A technical product question and a commercial request are not treated as the same thing.

For example:

```text
"¿Qué recomiendan para frizz?"
```

can remain a product-education interaction.

However:

```text
"Quiero trabajar con la marca"
"Quiero que me contacte un asesor"
"Quiero que visiten mi salón"
```

contains stronger commercial intent.

Conceptually:

```text
Product interest
        ↓
MULTIMEDIA / CONVERSACION

Strong commercial intent
        ↓
APOSTOLADO
        ↓
fase_actual = captura
        ↓
Structured qualification
```

This design avoids turning every product question into an aggressive sales form while still recognizing moments when a prospect is ready to progress commercially.

## Product education during qualification

Qualification does not completely disable useful product conversation.

The production prompt includes explicit handling for cases where a prospect asks a technical or product question while already in capture mode.

Instead of losing the qualification context, the system can:

```text
Answer relevant technical question
        ↓
Preserve already captured data
        ↓
Return to the next missing qualification field
```

This allows the conversation to remain useful rather than forcing the prospect through a rigid questionnaire.

## Multimedia routing

The router can determine which product context should be handled and which material should be delivered.

Relevant routing variables include concepts such as:

```text
lineaNombre
collectionHandle
multimedia_enviado
lineas
lineas_pendientes
```

The intended pattern is:

```text
User mentions need / product line
        ↓
Identify relevant product context
        ↓
Check persistent delivery state
        ↓
Respond with product information
        ↓
Deliver relevant material when appropriate
        ↓
Record delivery state
```

The system therefore acts as both:

```text
Conversational sales assistant
+
Product-education layer
```

rather than simply replying with generic text.

## Preventing repeated multimedia

The system stores already delivered material in:

```text
multimedia_enviado
```

When a piece of material is sent, the Supabase update logic adds the corresponding line identifier only if it is not already present.

Conceptually:

```text
Existing:
["OI", "NOUNOU"]

New delivery:
"OI"

        ↓

Already exists
        ↓
Do not duplicate state
```

This gives the router memory about what the prospect has already received.

## Multiple product needs

Real customers often mention several problems at once.

For example:

```text
"Necesito algo para frizz, caída y rizos"
```

The workflow includes:

```text
lineas_pendientes
```

to preserve additional needs and handle them sequentially.

Conceptually:

```text
frizz + caída + rizos
        ↓
Handle first need
        ↓
Store remaining needs
        ↓
Offer next relevant topic
        ↓
Continue sequentially
```

This prevents one response from becoming an overloaded list of unrelated product recommendations.

## Invalid-message recovery

The production router contains a recovery mechanism for cases where the initial classifier returns:

```text
CONVERSACION_INVALIDA
```

but the message still contains a strong known product or hair-need keyword.

Conceptually:

```text
Initial classification
CONVERSACION_INVALIDA
        ↓
Known strong keyword detected?
        │
        ├── YES
        │     ↓
        │ Recover as MULTIMEDIA
        │
        └── NO
              ↓
        Conversational clarification
```

This creates an additional safety layer against false-negative classification.

A typo or unusual phrasing should not automatically prevent the prospect from receiving useful product information.

## Correction handling

Users frequently correct information they provided earlier.

Examples:

```text
"Perdón, no era Lima, estoy en Iquitos"
```

```text
"Me confundí, soy independiente"
```

```text
"Son 3 estilistas, no 5"
```

The production qualification rules explicitly instruct the system to:

```text
Identify corrected field
        ↓
Update only that field
        ↓
Preserve unrelated information
        ↓
Continue qualification
```

This is important because real conversations are not clean database forms.

## Profile correction and state cleanup

The downstream persistence layer contains explicit logic for a profile correction from salon to independent professional.

If the final type becomes:

```text
independiente
```

the workflow can clear incompatible fields such as:

```text
salon
estilistas
```

instead of preserving values that no longer make sense for the corrected profile.

This prevents internal state from becoming contradictory.

## Anti-overwrite protection

The workflow includes anti-overwrite logic when new information is merged with existing Supabase state.

If the AI does not return a new value for a field that already contains valid information, the stored value can be preserved.

Conceptually:

```text
Existing Supabase state
        +
New extracted values
        ↓
Merge
        ↓
Do not replace valid data with empty values
```

This is particularly important for multi-turn qualification.

Without this protection, one incomplete AI response could erase information captured earlier.

## Product-line merge protection

The persistence layer also contains logic to preserve previously captured product-line information.

If new product interests appear later, existing and new values can be merged rather than blindly overwritten.

Conceptually:

```text
Existing lines
        +
New line
        ↓
Deduplicate
        ↓
Persist combined product context
```

The production workflow also contains specific sanitization logic for incorrectly inferred product lines, showing that product-state quality was adjusted through production testing rather than assumed to be perfect.

## Human-handoff detection

The workflow determines post-handoff state using persisted values including:

```text
bot_pausado = true
```

or:

```text
asesor_solicitado = true
```

or:

```text
estado = HUMANO
```

This creates a durable signal that the opportunity has already moved toward human commercial ownership.

## Handoff persistence

The downstream persistence logic can recognize language indicating that a salesperson will contact the prospect.

When handoff is detected, the workflow persists:

```text
bot_pausado = true
asesor_solicitado = true
estado = HUMANO
```

If the conversation was already handed off, the previous handoff state is preserved.

Conceptually:

```text
Advisor handoff detected
        ↓
Persist HUMANO state
        ↓
Next WhatsApp execution
        ↓
Router reads existing state
        ↓
Do not restart acquisition flow
```

This makes handoff resilient across independent webhook executions.

## Post-handoff conversation

A human handoff does not necessarily mean the customer will never send another WhatsApp message.

The workflow therefore contains dedicated post-handoff behavior.

The assistant can preserve previously captured information while avoiding a restart of the qualification sequence.

```text
Lead already handed off
        ↓
Customer sends new message
        ↓
Router detects postHandoff
        ↓
Preserve existing commercial data
        ↓
Do not restart qualification
```

The post-handoff conversational prompt also explicitly instructs the system not to erase previously registered fields.

## Structured data extraction after the AI response

The AI Agent returns conversational text together with a structured `[DATOS:]` block.

The downstream workflow parses this block and extracts commercial values such as:

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
lineas_pendientes
```

This converts natural-language conversation back into structured application state.

Conceptually:

```text
Natural-language response
        +
[DATOS: ...]
        ↓
Parser
        ↓
Merge with previous Supabase values
        ↓
Persist updated session
```

The language model therefore participates in information extraction, while deterministic code controls how that information is stored and protected.

## Router output and downstream execution

`Router F` returns structured routing information including:

```text
intencion
systemOverride
collectionHandle
lineaNombre
tieneLineaEnMensaje
hayPreguntaTecnica
intencionComercial
lineasPendientes
datosCapturados
telefonoWhatsApp
postHandoff
```

It also exposes debugging fields used during development and QA.

The router then connects to `Switch1`, which directs the execution toward the appropriate downstream behavior.

```text
Router F
        ↓
Switch1
        ↓
Execution branch
        ↓
AI Agent
        ↓
Post-processing
        ↓
Supabase / CRM / WhatsApp actions
```

## Debug-oriented routing

The production router contains explicit debug outputs such as:

```text
DEBUG_intencion
DEBUG_intencionFinal
DEBUG_esPrimerMensaje
DEBUG_catalogoYaEnviado
DEBUG_lineaNombre
DEBUG_lineasPendientes
```

These values make routing decisions easier to inspect during testing.

This is important because a conversational system can fail even when the final text looks plausible.

Debugging requires visibility into:

```text
What was classified?
What route was selected?
What state existed?
What product line was detected?
What pending needs existed?
What had already been delivered?
```

## Why deterministic routing was necessary

A pure language-model architecture would be flexible but could make important commercial behavior unpredictable.

For example, the model should not independently decide every time whether to:

```text
send prices again
restart qualification
erase a stored field
route a B2C consumer into B2B sales
treat an existing client as a new prospect
continue after human handoff
```

These decisions are better controlled through explicit state and business rules.

## Why the language model was still necessary

A completely hard-coded decision tree would be deterministic but poor at handling natural WhatsApp language.

Customers use:

```text
typos
short answers
slang
incomplete phrases
multiple needs
corrections
questions outside a fixed menu
```

The production architecture therefore combines both approaches.

```text
Deterministic code
        ↓
routing
state transitions
commercial rules
handoff
anti-repetition
data protection

        +

Language model
        ↓
natural conversation
contextual interpretation
technical explanations
commercial communication
structured extraction
```

## Final routing architecture

The production flow can be summarized as:

```text
WhatsApp inbound message
        ↓
Normalize input / handle audio when needed
        ↓
Clasificador F
        ↓
Initial intent
        ↓
Read existing session state
        ↓
Router F
        │
        ├── Is this an existing client?
        ├── Is persistent capture active?
        ├── Has a human already been requested?
        ├── Is this B2C traffic?
        ├── Is this a chain / distributor?
        ├── Is there strong commercial intent?
        ├── Is this a technical or product need?
        ├── Is this a pricing request?
        ├── Was this material already sent?
        ├── Are there pending product needs?
        └── Is recovery from invalid classification possible?
        ↓
systemOverride
        ↓
Switch1
        ↓
CONVERSACION
INFO_GENERAL
MULTIMEDIA
APOSTOLADO
B2C
ALERTA_CADENA
        ↓
AI / RAG / qualification / material delivery
        ↓
Parse structured data
        ↓
Merge with previous state
        ↓
Supabase persistence
        ↓
CRM / WhatsApp / human handoff as required
```

## Engineering evolution

The architecture evolved away from a simple pattern:

```text
Message
        ↓
AI
        ↓
Response
```

toward:

```text
Message
        ↓
Intent classification
        ↓
Persistent state
        ↓
Business rules
        ↓
Execution mode
        ↓
AI where flexibility is useful
        ↓
Structured extraction
        ↓
State update
```

This is the main architectural difference between a basic chatbot and the production B2B conversational system.

The assistant behaves as a stateful application with conversational AI inside it, rather than treating the language model as the entire application.

## Engineering lesson

The most important lesson from the routing layer was that conversational intelligence and workflow control solve different problems.

Natural-language understanding is valuable for interpreting messy human conversation.

Commercial routing requires greater determinism.

Combining both allowed the production system to remain conversational while protecting critical behaviors such as:

```text
qualification continuity
data persistence
product delivery
anti-repetition
customer-type routing
human handoff
post-handoff behavior
```

## Privacy and publication

This document describes the architecture and behavior of the routing system without exposing production secrets, customer conversations or personally identifiable information.

The production workflow itself should be sanitized before any public release.

A public artifact can preserve routing logic and engineering decisions while excluding:

```text
API credentials
customer identifiers
private operational URLs
internal document IDs
production contact information
personally identifiable customer data
```