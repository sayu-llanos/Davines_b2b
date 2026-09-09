# Supabase Conversation State

Supabase acts as the operational state layer of the B2B WhatsApp system.

The language model handles natural-language understanding and response generation, while structured business state is persisted separately so the workflow can make deterministic decisions across multiple WhatsApp messages.

This separation allows the system to remember what has already been captured, understand the current commercial phase and detect when a conversation has moved from automation to human follow-up.

## Production snapshot

The analyzed production snapshot contained **803 conversation sessions**.

Observed combinations of `fase_actual` and `estado` were:

| fase_actual | estado | Sessions |
|---|---|---:|
| exploracion | activo | 697 |
| captura | activo | 45 |
| captura | HUMANO | 35 |
| exploracion | HUMANO | 26 |

The conversation state is not represented by a single database field.

The workflow uses several persisted variables together:

```text
fase_actual
estado_conversacion
estado
asesor_solicitado
bot_pausado