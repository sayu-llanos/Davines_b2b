# QA Torture Test Summary

This document summarizes a structured QA review performed against the B2B WhatsApp STAGING workflow.

The purpose of the test was to identify failures in conversational routing, state management, qualification, handoff and CRM persistence before production changes.

## Test scope

**Total conversational scenarios evaluated: 46**

| Result | Count |
|---|---:|
| PASS | 12 |
| PARTIAL | 13 |
| FAIL | 21 |

The review identified:

**7 critical P0 issues**

plus additional P1 and P2 findings related to routing, UX and state behavior.

## Critical P0 findings

| ID | Failure | Root cause |
|---|---|---|
| P0-1 | Independent professional incorrectly routed as B2C | Professional context was not sufficiently considered when detecting `"no tengo salón"` |
| P0-2 | Salon owner purchasing for business incorrectly routed as B2C | `"peluquería"` was missing from professional-context exceptions |
| P0-3 | Misspelled districts / cities could cause wrong advisor assignment or repeated questions | Geographic routing lacked consistent fuzzy normalization |
| P0-4 | Short product-line tokens generated false positives | Substring matching detected product keys inside unrelated words |
| P0-5 | User could be told that an advisor would contact them without persistent handoff state | Conversational output and state persistence were not fully synchronized |
| P0-6 | Advisor phone number could be persisted as the lead's phone number | Phone extraction inspected assistant output together with user input |
| P0-7 | CRM profile data could be inferred from the bot's own question | Fallback extraction analyzed assistant text instead of only customer-supported data |

## Main P1 findings

Important non-P0 issues included:

- first-message information could be discarded by the welcome branch
- post-handoff buttons could become unreachable
- common unaccented words such as `catalogo` or `capacitacion` could be treated as invalid
- `"ok"` / `"dale"` / `"perfecto"` could accidentally trigger qualification
- multi-intent messages could lose secondary intentions
- qualification state could be reset after a technical question
- existing clients were not always recognized from free text
- pronouns such as `"esa"` could lose product-line context
- common typos such as `friz`, `enerjizing` or `asezor` were not consistently recovered
- some customer corrections could result in stale routing information

## Examples of tested edge cases

The QA corpus included scenarios such as:

```text
Professional vs consumer ambiguity
Salon vs independent stylist
Existing customer vs new prospect
Pricing requests
Catalog requests
Technical product questions
Multiple hair needs
Misspelled districts
Misspelled product lines
Short one-word replies
Corrections to previous answers
Repeated price requests
Human handoff
Post-handoff interaction
CRM persistence
State persistence
```

## Root-cause methodology

The objective was not simply to record:

```text
"The bot answered incorrectly."
```

Each failure was traced through the relevant workflow layer.

```text
User input
    ↓
Classifier
    ↓
Router
    ↓
AI Agent
    ↓
Structured extraction
    ↓
Supabase state
    ↓
CRM transformation
```

The first incorrect layer was identified before proposing a fix.

## Deterministic code vs LLM behavior

The QA report intentionally distinguished between:

```text
[CÓDIGO]
Behavior directly determined by JavaScript,
regex, routing or workflow topology

[LLM]
Behavior dependent on model output
```

Critical regex and routing cases were executed directly during analysis rather than being inferred only from reading the workflow.

This distinction helped separate:

```text
Deterministic application bugs
```

from:

```text
Probabilistic model behavior
```

## Example: substring false positive

One critical bug occurred because short product-line identifiers were matched using substring logic.

Conceptually:

```text
"soi estilista"
```

could incorrectly match:

```text
OI
```

and:

```text
"espera un minuto"
```

could incorrectly match:

```text
MINU
```

The root cause was product detection equivalent to:

```text
q.includes(key)
```

without sufficient word-boundary protection.

This could cause the workflow to confidently deliver the wrong product material.

## Example: false handoff

Another critical issue involved the boundary between conversational output and application state.

A user could receive a message equivalent to:

```text
"Our advisor will contact you."
```

while the required state:

```text
bot_pausado = true
asesor_solicitado = true
estado = HUMANO
```

was not successfully persisted.

This demonstrated that human handoff needed to be treated as an application-state transition rather than only generated text.

## Example: CRM data contamination

QA also identified cases where downstream CRM extraction could inspect both:

```text
User message
+
Assistant response
```

This created the possibility of information from the assistant itself being interpreted as customer data.

For example:

```text
Bot asks:
"Do you work as an independent stylist?"
```

The downstream CRM logic could incorrectly infer:

```text
TipoPerfil = Independent stylist
```

before the customer had answered.

This finding reinforced the rule:

```text
Customer data must be supported by customer input
or validated application state.
```

## Structural findings

The test also identified workflow-level issues that were not isolated to a single prompt or regex.

Examples included:

```text
First-message branch ending before normal processing

and

Post-handoff state bypassing button routing
```

These findings demonstrated that conversational QA must inspect:

```text
Node logic
+
Workflow topology
+
Execution order
```

not only AI responses.

## QA philosophy

The purpose of the torture test was not to demonstrate that the assistant was perfect.

The objective was to intentionally search for assumptions that real WhatsApp users could break.

```text
Happy-path testing:
Does the designed journey work?

Torture testing:
What happens when users behave differently
from the designed journey?
```

This included:

```text
Typos
Informal Spanish
Ambiguous intent
Incomplete answers
Multiple intentions
Corrections
Unexpected wording
Repeated requests
State changes
```

## What the test established

The QA process demonstrated that:

- conversational correctness is different from workflow execution success
- AI outputs must be evaluated together with deterministic state
- root-cause analysis is more useful than fixing individual sentences
- CRM persistence requires its own validation layer
- STAGING testing reduces production risk
- stateful conversational systems require multi-turn testing
- model behavior and deterministic code should be evaluated separately

## Historical checkpoint

These results represent a **historical QA checkpoint** against the analyzed STAGING workflow.

They should not be interpreted as the failure rate of the later production system.

The findings were used to guide subsequent fixes, regression testing and workflow hardening.

## Privacy

The original QA artifact contains detailed internal traces and production-related implementation context.

This public summary intentionally excludes:

- customer PII
- phone numbers
- private workflow identifiers
- private local file paths
- internal URLs
- credentials
- raw customer conversations 