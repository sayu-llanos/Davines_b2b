# Data Model

The production system stores conversation state, lead qualification data, chat history and retrieval knowledge outside the language model.

This separation allows the WhatsApp automation to preserve state across messages, avoid repeated qualification questions and make deterministic routing decisions.

## Supabase tables used by the B2B system

### `sesiones_bot`

Main conversation and lead-state table.

Key fields:

| Field | Type | Default / Purpose |
|---|---|---|
| `chat_id` | text | Conversation identifier |
| `tipo` | text | Lead type |
| `salon` | text | Salon name |
| `ciudad` | text | City |
| `distrito` | text | District |
| `estilistas` | text | Number / description of stylists |
| `telefono` | text | Contact phone |
| `correo` | text | Contact email |
| `contacto` | text | Contact name |
| `marca` | text | Professional brands currently used |
| `driver` | text | Commercial motivation |
| `horario` | text | Preferred contact channel / handoff mode |
| `lineas` | text | Product lines or needs already discussed |
| `lineas_pendientes` | text | Pending needs / product lines |
| `multimedia_enviado` | text | Tracks previously delivered media |
| `fase_actual` | text | Default: `exploracion` |
| `estado_conversacion` | varchar | Default: `bienvenida` |
| `estado` | text | Default: `activo` |
| `bot_pausado` | boolean | Default: `false` |
| `asesor_solicitado` | boolean | Default: `false` |
| `reminder_enviado` | boolean | Default: `false` |
| `lineas_enviadas` | integer | Default: `0` |
| `ultimo_estado_cambio` | timestamp | Default: `now()` |
| `updated_at` | timestamptz | Default: `now()` |

This table acts as the operational state store for the WhatsApp flow.

---

### `n8n_chat_histories`

Stores conversational memory used by the automation.

| Field | Type |
|---|---|
| `id` | integer |
| `session_id` | varchar |
| `message` | jsonb |

The language model can use chat history for conversational continuity, while business-critical state remains in `sesiones_bot`.

---

### `documents`

Knowledge table used by the retrieval layer.

| Field | Type |
|---|---|
| `id` | bigint |
| `content` | text |
| `source` | text |
| `tipo` | text |
| `embedding` | vector / user-defined |
| `metadata` | jsonb |

This table stores knowledge chunks and embeddings used for semantic retrieval.

The production workflow uses this knowledge layer for product and technical questions rather than relying only on the language model's internal knowledge.

---

### `salones_davines`

Stores salon records separately from the conversational session state.

| Field | Type |
|---|---|
| `id` | integer |
| `nombre` | text |
| `direccion` | text |
| `distrito` | text |
| `ciudad` | text |
| `region` | text |
| `red_social_url` | text |
| `activo` | boolean |
| `estado_revision` | text |
| `fuente` | text |
| `created_at` | timestamptz |
| `updated_at` | timestamptz |

This table is separate from `sesiones_bot`, which stores the live WhatsApp conversation state.

---

### `leads_notificados`

Tracks lead-notification state.

| Field | Type |
|---|---|
| `chat_id` | text |
| `ultimo_horario` | text |
| `notificado_at` | timestamp |

This supports notification control around qualified leads and handoff events.

## State vs conversational memory

The system intentionally separates two concepts:

```text
Natural conversation history
        ↓
n8n_chat_histories

Business-critical state
        ↓
sesiones_bot