# QA Deep Dive and Conversational Torture Testing

Quality assurance became a major part of the B2B WhatsApp system because conversational applications fail differently from traditional forms or deterministic software.

A workflow may execute successfully while still producing commercially incorrect behavior.

Examples include:

```text
Wrong customer classification
Repeated qualification questions
Incorrect product routing
Lost conversation state
False human handoff
Duplicate multimedia
Incorrect salesperson assignment
B2C users entering the B2B funnel
```

For this reason, QA evolved from manually trying conversations into a structured testing process involving:

```text
Conversation corpora
Severity levels
Root-cause analysis
STAGING isolation
Regression testing
Static code execution
Node-level tracing
End-to-end WhatsApp validation
```

## Why conversational QA was necessary

An n8n execution finishing successfully does not mean the user journey is correct.

A technically successful workflow could still behave like:

```text
HTTP 200
        ↓
All nodes green
        ↓
Wrong business decision
```

For example:

```text
Consumer asks to buy shampoo for personal use
        ↓
Workflow executes successfully
        ↓
Router incorrectly enters B2B qualification
```

From an infrastructure perspective, nothing crashed.

From a business perspective, the system failed.

QA therefore needed to evaluate:

```text
Technical execution
+
Conversation behavior
+
State transitions
+
Commercial routing
```

## Dedicated STAGING environment

Significant QA and fixes were performed against a dedicated STAGING workflow rather than modifying production directly.

The principle was:

```text
Identify issue
        ↓
Reproduce in STAGING
        ↓
Diagnose root cause
        ↓
Apply minimal fix
        ↓
Run targeted tests
        ↓
Run regression tests
        ↓
Validate behavior
        ↓
Only then consider production
```

This reduced the risk of debugging directly against live customer traffic.

Production and STAGING were intentionally treated as separate environments.

## Initial torture-test baseline

A structured torture-test baseline was created around **46 conversation scenarios**.

The baseline identified:

```text
7 P0 bugs
+
2 structural findings
```

These were not treated as seven unrelated customer complaints.

The findings were grouped by root cause so that one engineering fix could resolve several failing scenarios.

This changed the debugging approach from:

```text
Conversation fails
        ↓
Patch conversation
```

to:

```text
Several conversations fail
        ↓
Identify shared mechanism
        ↓
Fix root cause
        ↓
Regression-test all affected cases
```

## Severity model

The QA process used severity to distinguish commercially dangerous failures from smaller UX issues.

A simplified interpretation is:

### P0 — critical

Behavior capable of breaking an essential commercial path.

Examples:

```text
Professional prospect routed as B2C
B2C consumer routed into B2B capture
False salesperson handoff
Incorrect critical state transition
Critical qualification path blocked
```

### P1 — important

Behavior that significantly damages the experience or data quality but does not necessarily destroy the complete commercial journey.

Examples:

```text
Useful first-message information lost
Repeated qualification
Wrong intermediate response
Important context not persisted
```

### P2 — lower severity

Smaller quality, wording or edge-case problems that can be improved without blocking the core journey.

The objective was not to make every imperfection a P0.

Severity helped prioritize engineering effort around business risk.

## Root-cause clustering

The original P0 findings were grouped into engineering clusters instead of being fixed individually.

Examples of root causes discovered during QA included:

```text
Professional vs B2C classification
Geographic fuzzy matching
Product-line substring matching
False handoff behavior
State persistence
Routing interactions
Structural workflow ordering
```

This made the testing process closer to software debugging than manual chatbot reviewing.

## Example — professional prospect misclassified as B2C

One critical class of failures involved professional users whose language superficially resembled a consumer request.

Examples included messages conceptually similar to:

```text
"I don't have a salon, I work from home.
What do I need to work with Davines?"
```

or:

```text
"I want to buy 10 shampoos for my salon."
```

The initial classification rules could incorrectly interpret phrases such as:

```text
no tengo salón
para mí
```

as B2C signals without sufficiently considering professional context.

### Root cause

Professional-context vocabulary and B2C exception logic were not sufficiently aligned.

### Engineering response

Professional signals were strengthened around concepts such as:

```text
peluquería
barbería
spa
estética
estilista
trabajar con Davines
comprar para negocio
```

while preserving genuine B2C detection.

### Lesson

```text
Keyword alone
≠
intent
```

The same phrase can have different commercial meaning depending on the surrounding context.

## Example — fuzzy geographic input

Real WhatsApp conversations contain typos.

Examples observed during QA included inputs conceptually equivalent to:

```text
"sna borja"
```

instead of:

```text
"san borja"
```

and:

```text
"trujilo"
```

instead of:

```text
"trujillo"
```

Geographic information mattered because downstream commercial routing could depend on city or district.

### Failure mode

Exact matching could result in:

```text
Typo
        ↓
Location not recognized
        ↓
Fallback behavior
        ↓
Incorrect commercial routing
```

### QA principle

Fuzzy matching had to be tested not only for recognition but also for persistence.

A system that internally corrects:

```text
trujilo
→
Trujillo
```

but then rejects the corrected value when saving state has not actually solved the problem.

This led to tests spanning:

```text
Input interpretation
+
Router behavior
+
Persistence behavior
```

## Example — product-line substring false positive

Short product names introduced another edge case.

For example, a line such as:

```text
OI
```

should not be detected merely because those two letters appear inside an unrelated word.

Conceptually:

```text
"soi estilista"
```

must not accidentally produce:

```text
OI detected
```

Likewise, a product key should not activate inside unrelated words such as:

```text
"minuto"
```

simply because a short product token appears as a substring.

### Root cause

Substring matching without appropriate word boundaries.

### Engineering response

Testing forced product recognition to distinguish:

```text
Product token
```

from:

```text
Letters appearing inside another word
```

This is a good example of why real conversational inputs are more useful than only testing ideal product names.

## Example — false human handoff

Handoff QA was particularly important because conversational text and application state could disagree.

A dangerous scenario is:

```text
AI:
"Our advisor will contact you."

        ↓

Application:
handoff state not actually persisted
```

The user now believes a salesperson is coming, while the system does not.

The opposite is also problematic:

```text
Application marks conversation HUMAN
        ↓
Customer was not actually ready for handoff
```

QA therefore tested both:

```text
What the customer is told
```

and:

```text
What the application persists
```

A handoff is correct only when those layers agree.

## Structural workflow testing

Not every issue existed inside JavaScript.

Some failures came from workflow topology.

One historical structural issue involved the order of button handling and paused-bot logic.

The QA process compared workflow connections before and after the fix rather than relying only on comments inside nodes.

This is important because:

```text
Correct node code
+
Incorrect graph order
=
Broken application
```

Workflow QA therefore included:

```text
Node logic
+
Connections
+
Execution order
```

## Baseline → fix → re-execution

After the first P0 fixes, the original failing scenarios were executed again against the updated STAGING logic.

The re-execution verified that the **seven original P0 findings** were corrected at that checkpoint.

The validation did not rely only on comments such as:

```text
// FIXED
```

The relevant JavaScript logic was executed against the same test messages used by the baseline.

For several critical cases, dedicated harnesses were used to inspect the exact values that would be:

```text
returned by classifier
routed by Router F
persisted
written to CRM
```

## Why re-running the same cases matters

A fix should be tested against the exact input that originally failed.

```text
Original failure
        ↓
Engineering change
        ↓
Same input
        ↓
Expected behavior now passes
```

Otherwise, a developer may believe a bug was fixed simply because a different test case happens to work.

## Regression testing

Fixing one conversational rule can easily break another.

For example, expanding professional-context detection could accidentally cause true B2C consumers to enter the B2B funnel.

Regression testing therefore included:

```text
Original failing cases
+
New variations
+
Opposite / negative cases
+
Neighboring behaviors
```

Conceptually:

```text
Fix:
Recognize "peluquería"

Regression:
Confirm genuine consumer messages still route to B2C
```

The goal was not simply:

```text
Make failing test green
```

but:

```text
Make failing test green
without turning previously correct cases red
```

## Targeted regression suites

Some root-cause fixes were tested using dedicated groups of cases.

Historical QA artifacts include targeted runs such as:

```text
15 / 15
18 / 18
24 / 24
```

for specific P0 root-cause groups and regression variants.

These focused suites were useful because they made it possible to validate an isolated change before re-running the wider conversational benchmark.

The numbers refer to targeted test groups, not total production conversations.

## Expanding beyond the original baseline

The QA corpus evolved beyond the initial 46-conversation baseline.

A later reconciliation evaluated **51 conversational scenarios** and traced important cases more deeply through the workflow.

At that reconciliation checkpoint, the recorded classification was:

| Result | Conversations |
|---|---:|
| PASS | 18 |
| PARTIAL | 15 |
| FAIL | 18 |
| **Total** | **51** |

This table should be understood as a historical QA checkpoint, not as a metric of current production quality.

Its value is that the reconciliation challenged earlier assumptions rather than trying to produce an artificially perfect score.

## QA reconciliation

One of the strongest parts of the testing process was reconciling previous QA claims.

A previous report could mark a scenario:

```text
PASS
```

but deeper tracing might reveal that only `Clasificador F` behaved correctly while `Router F` later changed the outcome.

The reconciliation therefore asked:

```text
Did the classifier pass?
```

and then:

```text
Did the complete relevant decision path pass?
```

This distinction exposed cases where an apparent fix at one layer did not produce the expected end-to-end behavior.

## Example — classifier fixed, router still blocks

A message can successfully pass:

```text
Clasificador F
```

while still failing later because:

```text
Router F
```

contains an independent gate.

This was observed during QA and changed the verdict of previously reported scenarios.

The lesson was:

```text
Component-level success
≠
End-to-end success
```

## Discovering new failures during regression

The reconciliation also uncovered a new critical case that had previously been reported as passing.

A B2C purchasing message could still be intercepted by an advisor-related router rule and enter the wrong B2B behavior.

Importantly, this issue was not introduced by the fix being tested.

It was a pre-existing defect revealed by deeper tracing.

This distinction matters.

A responsible QA report should differentiate:

```text
Regression caused by current change
```

from:

```text
Previously existing bug discovered during current testing
```

## No artificial "perfect QA" narrative

The QA process did not force every report toward:

```text
100% PASS
```

When deeper verification contradicted an earlier PASS, the result was downgraded.

For example:

```text
PASS
→
PARTIAL
```

or:

```text
PASS
→
FAIL
```

when the evidence required it.

This is more useful than preserving optimistic metrics that the implementation cannot support.

## F-INV robustness work

Another QA cycle focused on messages incorrectly falling into a generic:

```text
"I didn't understand"
```

type of response.

A classifier robustness fix improved the handling of natural messages that were valid but did not fit the earlier narrow rules.

A historical test corpus showed **26 of 87 tested messages** moving away from that invalid-message fallback after the fix, with no regressions attributed specifically to that change at the reconciliation checkpoint.

The important lesson was not the number itself.

It was that natural-language fallbacks also require regression testing.

## Testing messy language

The QA corpus intentionally included messages that do not look like clean demo inputs.

Examples of stress categories included:

```text
Typos
Short answers
Missing accents
Misspelled districts
Multiple needs
Corrections
Consumer/professional ambiguity
Advisor requests
Pricing language
Product names
Informal Spanish
One-word replies
```

This matters because a chatbot that only works for:

```text
"Hello, I own a salon and would like information about Davines."
```

is not production-ready for WhatsApp.

## Stateful conversation testing

Individual-message testing is not enough for a stateful assistant.

Some scenarios required multiple turns:

```text
Turn 1
Professional intent

Turn 2
Profile

Turn 3
City

Turn 4
District

Turn 5
Brand

Turn 6
Commercial driver

Turn 7
Preferred contact
```

QA therefore needed to verify:

```text
State after each turn
```

rather than evaluating only the final natural-language response.

## State contamination between tests

Because Supabase persists conversation state, independent QA cases should not unintentionally reuse the same user identity.

The later E2E checklist explicitly required a different WhatsApp test number for independent scenarios unless the test was intentionally continuing a previous conversation.

Conceptually:

```text
Case A
test number A

Case B
test number B
```

instead of:

```text
Case A
same number
        ↓
state persists

Case B
unexpected previous state
        ↓
false QA result
```

This is a critical testing principle for stateful conversational systems.

## Node-level observability

The E2E methodology required inspecting specific workflow outputs rather than judging only the final WhatsApp message.

Relevant nodes included:

```text
Clasificador F
Router F
Guardar Datos Parciales
Code in JavaScript
```

This allowed a failed conversation to be traced as:

```text
Input
        ↓
Classifier output
        ↓
Router output
        ↓
Extracted structured data
        ↓
Persisted state
```

instead of concluding only:

```text
"The bot was weird."
```

## CRM and Supabase verification

QA also extended beyond conversational output.

A scenario could sound correct to the customer but persist incorrect data.

Therefore relevant tests checked:

```text
Supabase session state
and/or
STAGING CRM row
```

Examples:

```text
Did city persist?
Did the professional profile persist?
Was the lead incorrectly marked human?
Did the CRM receive the correct field?
```

This turns conversational QA into application QA.

## Static execution vs real E2E

The project used more than one testing level.

### Static / code-level testing

JavaScript from workflow nodes could be extracted and executed with controlled mocks.

Useful for:

```text
Classifier rules
Router rules
Field transformations
Regression suites
Edge cases
```

Advantages:

```text
Fast
Repeatable
Precise
Easy to isolate root causes
```

But it cannot prove the complete WhatsApp journey.

### STAGING end-to-end testing

A later checklist was created specifically for real WhatsApp-to-STAGING validation.

That level verifies:

```text
WhatsApp
→ YCloud
→ n8n
→ state
→ response
→ CRM
```

This is slower, but validates integration behavior that isolated JavaScript tests cannot fully reproduce.

## E2E test design

The manual STAGING checklist used a repeatable structure.

For each case:

```text
Test objective
        ↓
Exact message(s)
        ↓
Expected user behavior
        ↓
Expected n8n nodes
        ↓
Expected Supabase / CRM state
        ↓
PASS / PARTIAL / FAIL
```

The tester was also instructed to record the approximate execution time so the corresponding n8n execution could be located easily.

This improves traceability between:

```text
WhatsApp test
and
workflow execution
```

## PASS / PARTIAL / FAIL

Conversational systems benefit from more nuance than a binary result.

### PASS

The critical expected behavior occurs correctly.

### PARTIAL

The main journey is usable but part of the expected behavior is incomplete or incorrect.

### FAIL

The relevant scenario does not satisfy the expected behavior.

This is particularly useful because:

```text
Correct response
+
wrong state
```

may deserve PARTIAL or FAIL depending on commercial impact.

## Testing first-message behavior

QA also documented structural behavior that was known to be suboptimal.

One example involved first messages containing useful commercial information.

A user might send:

```text
"Hi, I own a salon in Los Olivos,
we currently use another brand,
how much does Davines cost?"
```

but an early welcome path could prioritize:

```text
welcome
video
menu
```

without processing all information included in that same first message.

This was documented as a known structural behavior instead of pretending that every production path was optimal.

Known limitations are part of credible engineering documentation.

## Important QA categories

The complete QA strategy covered categories such as:

| Category | What can fail |
|---|---|
| B2B vs B2C | Wrong funnel |
| Product need | Wrong line / recommendation context |
| Pricing | Missing or duplicate price material |
| Short replies | Capture context lost |
| Geography | Wrong or rejected location |
| Corrections | Previous value not replaced correctly |
| Multiple needs | Secondary needs lost |
| Existing clients | New-lead flow incorrectly restarted |
| Handoff | False, missing or repeated escalation |
| Post-handoff | Qualification restarts |
| Multimedia | Wrong or duplicate material |
| State | Data lost or overwritten |
| CRM | Wrong / missing structured record |
| Buttons | Workflow branch unreachable |
| Audio | Voice path fails |
| Invalid messages | Valid text rejected |

## Torture testing philosophy

The objective of torture testing was not to demonstrate that the system never fails.

The objective was to intentionally search for the places where it fails.

```text
Normal QA:
Does the happy path work?

Torture test:
How can a real user break the assumptions?
```

Useful torture inputs include:

```text
Contradictory statements
Typos
Multiple intents
Incomplete answers
Corrections
Repeated requests
Unexpected wording
Consumer/professional ambiguity
Messages after handoff
```

This approach produced more useful engineering information than repeatedly testing ideal conversations.

## Fixing causes instead of examples

One of the most important QA lessons was to avoid creating a new rule for every failed sentence.

Bad pattern:

```text
"trujilo" failed
        ↓
Hard-code "trujilo"
```

Better pattern:

```text
"trujilo" failed
        ↓
Why are typos not handled?
        ↓
Improve geographic normalization / fuzzy logic
        ↓
Test other misspellings
```

The same principle applies to:

```text
Professional classification
Product matching
Handoff
Short responses
```

## Minimal-fix principle

Changes were intentionally scoped around the confirmed root cause.

Preferred process:

```text
Identify exact mechanism
        ↓
Change smallest relevant area
        ↓
Verify diff scope
        ↓
Run targeted tests
        ↓
Run neighboring regression suite
```

This reduces the risk of "fixing" one conversation by destabilizing unrelated paths.

## Workflow diff verification

Because n8n workflows contain large JSON documents, QA also included checking which nodes actually changed.

A fix could be expected to affect:

```text
Clasificador F
```

but an accidental workflow edit might change unrelated nodes or connections.

Diff inspection therefore helped answer:

```text
What changed?
Which nodes changed?
Did graph connections change?
Was the workflow still valid JSON?
```

This adds code-review discipline to a visual automation environment.

## Production protection during QA

An explicit rule during major QA rounds was:

```text
Do not modify production while diagnosing STAGING.
```

This allowed:

```text
Baseline
Fix
Regression
Reconciliation
```

to occur without risking the live customer flow.

Isolation between production and STAGING was therefore part of the testing methodology, not only infrastructure organization.

## QA artifact evolution

The historical QA process produced several types of artifacts:

```text
Baseline torture-test report
P0 fix plan
After-P0 report
Reconciled report
E2E STAGING checklist
Workflow snapshots
Targeted test scripts
```

These artifacts represent different moments in the engineering process.

They should not be collapsed into a single "final score."

Each artifact answers a different question:

```text
Baseline:
What is broken?

Fix plan:
Why is it broken?

After-P0:
Did the targeted fixes work?

Reconciliation:
Were previous conclusions actually correct end-to-end?

E2E checklist:
Does the integrated WhatsApp system behave correctly in STAGING?
```

## Interpreting historical QA metrics

Historical test counts should always be labeled with their testing stage.

For example:

```text
46 conversations
```

refers to the original torture-test baseline.

```text
51 conversations
```

refers to a later expanded/reconciled QA corpus.

```text
87 messages
```

belongs to a classifier-focused robustness corpus.

They are not interchangeable.

They should not be added together and described as:

```text
184 conversations tested
```

because they represent overlapping test methods and scopes.

## What QA proves

The QA artifacts support claims such as:

```text
The system was tested against non-happy-path conversations.

Critical bugs were categorized and investigated by root cause.

Fixes were regression-tested.

Workflow behavior was traced beyond the LLM response.

State and CRM persistence were included in QA.

STAGING was used to reduce production risk.

Earlier PASS claims were revised when deeper evidence contradicted them.
```

## What QA does NOT prove

The testing does not prove:

```text
The system can never fail.

Every possible Spanish sentence was tested.

Every historical issue was absent from every later version.

A static unit-style test is equivalent to WhatsApp E2E.

An old QA score describes the current production workflow.
```

This distinction is important for technical credibility.

## Recommended public QA evidence

The public repository should summarize the methodology and selected sanitized examples.

Useful public evidence includes:

```text
Test categories
Severity definitions
Example synthetic failures
Root-cause methodology
Regression strategy
Historical aggregate counts
Sanitized screenshots of test reports
```

Raw customer conversations should remain private.

## Private QA evidence

Private historical material can retain:

```text
Full torture-test reports
STAGING workflow snapshots
Detailed internal test scripts
Conversation-specific debugging
Execution screenshots
```

provided that customer PII and credentials remain protected.

These artifacts can support implementation history without needing to be published.

## Example QA record

A public-safe bug record can look like:

```text
ID:
QA-B2B-014

Scenario:
Professional stylist without physical salon asks how to work with Davines.

Expected:
Recognize professional B2B context.

Observed:
Routed as B2C.

Severity:
P0

Layer:
Clasificador F

Root cause:
Professional-context exceptions were incomplete.

Fix:
Centralize and expand professional-context signals.

Regression:
Test true B2C + salon + independent stylist variants.

Result:
Targeted suite passes.
```

This communicates much more engineering depth than:

```text
"Fixed chatbot bug."
```

## QA loop

The operational QA loop can be summarized as:

```text
Observe conversation
        ↓
Reproduce
        ↓
Classify severity
        ↓
Trace workflow
        ↓
Identify first incorrect layer
        ↓
Confirm root cause
        ↓
Implement minimal fix
        ↓
Targeted test
        ↓
Regression suite
        ↓
E2E validation when required
        ↓
Document result
```

## Engineering lesson

The most important QA lesson from the project was that conversational AI quality is not only a prompt-engineering problem.

Failures can originate from:

```text
Classifier rules
Router rules
State
Persistence
Graph topology
Data transformation
RAG
Integration behavior
Prompt behavior
```

A reliable debugging process must identify the first incorrect layer.

The project therefore evolved from:

```text
Try chatbot manually
        ↓
Edit prompt
        ↓
Try again
```

toward:

```text
Structured scenario
        ↓
Expected behavior
        ↓
Node-level trace
        ↓
Root cause
        ↓
Minimal fix
        ↓
Regression testing
        ↓
End-to-end validation
```

That change in testing methodology was as important as many of the individual workflow fixes.

## Final QA principle

A production conversational system should not be judged by whether it can complete one perfect demo.

It should be judged by how it behaves when users:

```text
make mistakes
change their mind
use slang
write incomplete messages
send voice notes
ask multiple things
provide information out of order
return after handoff
behave differently from the designed happy path
```

The purpose of QA is to make those failures observable, reproducible and progressively less dangerous to the business.

## Privacy

Public QA examples should use synthetic or anonymized conversations.

Raw customer messages, phone numbers, emails, internal salesperson data, credentials and production identifiers should remain private.