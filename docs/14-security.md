# Security, Secrets and Publication Hygiene

The B2B WhatsApp system handles production integrations, customer conversations and commercial data.

Security therefore involves more than hiding an API key.

The architecture needs to protect:

```text

Credentials

Customer identity

Conversation content

Commercial information

Private infrastructure

Internal documents

Production workflow configuration

```

This document describes the security practices applied to the project and the controls required before any part of the production system is published externally.

## Security objectives

The main security goals are:

```text

Do not expose production credentials

Do not publish customer PII

Do not leak private infrastructure identifiers

Separate workflow logic from secrets

Preserve private implementation evidence safely

Publish only sanitized technical artifacts

```

The objective is to make the engineering work auditable without making the production environment reproducible by an unauthorized third party.

## Sensitive assets

The system touches several classes of sensitive information.

### Credentials

Examples include:

```text

YCloud API credentials

OpenAI credentials

Supabase credentials

PostgreSQL credentials

Google integrations

Webhook authentication

```

These values must not be committed into Git.

### Personally identifiable information

Production systems may contain:

```text

Customer names

Phone numbers

Email addresses

Salon contact information

WhatsApp conversations

Salesperson contact information

```

These records are operational data, not portfolio material.

### Internal commercial information

Private artifacts can also contain:

```text

Pricing information

Internal commercial documents

Professional-customer databases

Lead qualification data

Internal knowledge sources

Commercial observations

```

Even when a value is not a password, that does not mean it belongs in a public repository.

## Credential management

Production secrets should be stored through credential-management mechanisms whenever the integration supports them.

For the n8n workflow, the preferred pattern is:

```text

Workflow node

        ↓

References credential

        ↓

n8n Credentials

        ↓

Authentication secret

```

rather than:

```text

Workflow node

        ↓

Hard-coded API key

```

This separates application logic from authentication material.

Benefits include:

```text

Easier credential rotation

Lower risk during workflow export

Less secret duplication

Cleaner version control

Safer collaboration

```

## YCloud authentication hardening

A real security issue was identified during production review around YCloud authentication.

An HTTP request produced an authentication failure equivalent to:

```text

401

X-API-Key missing

```

The integration was reviewed and the authentication configuration was migrated to a dedicated n8n credential rather than relying on manually managed authentication values across individual HTTP nodes.

The hardened pattern became:

```text

YCloud HTTP Request

        ↓

Generic Credential Type

        ↓

Header Auth credential

        ↓

X-API-Key managed outside workflow logic

```

For POST requests, payload configuration remains separate:

```text

Authentication

        ↓

n8n Credential

Request format

        ↓

Content-Type: application/json

```

The purpose of this change was not merely to fix the 401 error.

It also reduced the risk of accidentally exporting production secrets together with workflow logic.

## Credential rotation principle

When changing a production credential, the safe sequence is:

```text

Create replacement credential

        ↓

Configure affected nodes

        ↓

Test all relevant workflow paths

        ↓

Confirm production behavior

        ↓

Revoke old credential

```

Revoking the original secret before validating the replacement can create unnecessary downtime.

Testing should therefore precede final credential revocation.

## No secrets in workflow headers

A production workflow should not contain manually embedded secrets such as:

```text

X-API-Key: actual-secret

Authorization: Bearer actual-token

api_key = actual-secret

password = actual-password

```

Authentication headers that contain sensitive values belong in credential storage.

A sanitized workflow can still show that authentication exists without revealing the value.

For example:

```text

Authentication:

Header Auth credential

```

is appropriate.

This is not:

```text

X-API-Key:

abc123-real-production-secret

```

## Secret scanning before publication

Before making any workflow artifact public, the repository should be scanned for patterns associated with credentials.

Examples include:

```text

X-API-Key

Authorization

Bearer

api_key

apikey

secret

password

token

sk-

service_role

connection string

```

Finding one of these words does not automatically mean a secret exists.

The result should be manually inspected to distinguish:

```text

Safe documentation

```

from:

```text

Actual credential value

```

## Workflow sanitization

The production n8n JSON should not be published directly without review.

A public-safe workflow should preserve:

```text

Node structure

Routing architecture

Code logic

State transitions

RAG architecture

Integration relationships

```

while removing or replacing:

```text

Credentials

Credential identifiers when unnecessary

Private webhook URLs

Customer identifiers

Private database references

Internal file IDs

Internal Google Drive IDs

Internal Google Sheet IDs

Production phone numbers

Private service URLs

Personally identifiable information

```

Conceptually:

```text

Production workflow

        ↓

Security review

        ↓

Secret removal

        ↓

PII removal

        ↓

Identifier sanitization

        ↓

Public-safe workflow

```

The goal is not to hide the engineering.

The goal is to separate engineering evidence from operational access.

## Production artifact vs public artifact

The project distinguishes between two forms of evidence.

### Private production artifact

The original workflow can be retained privately as historical evidence.

It demonstrates the actual production implementation.

It should not be published if it exposes operational or customer information.

### Public technical artifact

A sanitized workflow can be created for recruiters, engineers or portfolio review.

It can retain:

```text

Architecture

Routing logic

State logic

Code structure

RAG relationships

Handoff behavior

```

without exposing:

```text

Secrets

PII

Private infrastructure

Commercial identifiers

```

This separation makes it possible to demonstrate technical depth without weakening production security.

## Repository protection

The repository `.gitignore` intentionally excludes sensitive or unnecessary local artifacts.

Examples of files that should remain outside Git tracking include:

```text

Original production workflow exports

Environment files

Private key files

Secret files

Private evidence directories

Raw internal knowledge documents

Production database exports

Conversation exports

```

A critical operational rule is:

```text

Do not use:

git add .

without first reviewing git status.

```

Instead, files should be staged intentionally.

For example:

```text

git add docs/14-security.md

```

This reduces the chance of accidentally committing a newly downloaded private dataset.

## Private RAG evidence

The project retains private source material used during development of the RAG system.

Examples include:

```text

Technical knowledge documents

B2B commercial material

Qualification documentation

Existing-salon datasets

Supabase exports

Conversation-history exports

```

These artifacts are useful as implementation evidence but should remain outside the public repository.

The repository documents:

```text

What type of knowledge existed

How it was prepared

How it was chunked

How embeddings were created

How retrieval worked

```

without publishing the underlying confidential files.

## CRM privacy

The operational CRM contains real commercial information.

Potentially sensitive fields include:

```text

Name

Salon

Phone

Email

City

District

Current brand

Commercial motivation

Observations

Advisor summary

```

Public documentation therefore uses:

```text

Schema

Aggregated counts

Field names

Architecture

Synthetic examples

```

rather than real rows.

For example:

```text

55 / 55 ready-to-contact records contained an advisor summary

```

can be published.

A real customer's phone number should not be.

## Supabase privacy

The same principle applies to Supabase.

Public documentation can describe:

```text

sesiones_bot

documents

n8n_chat_histories

state fields

aggregate counts

```

without exposing:

```text

individual session rows

chat IDs

phone numbers

emails

conversation messages

private metadata

```

The production snapshot is used analytically, but only aggregate results are included in documentation.

## Conversation-history privacy

Chat history requires especially careful handling because free-form messages can contain information that was never intended for public distribution.

Examples include:

```text

Names

Phone numbers

Business information

Personal questions

Commercial negotiations

Addresses

Free-form customer messages

```

Raw `n8n_chat_histories` exports should therefore remain private.

Synthetic conversations should be used for public demonstrations.

## Synthetic demo data

A public demo should use fictional information.

Example:

```text

Nombre:

María Demo

Salón:

Studio Aurora

Ciudad:

Lima

Distrito:

Miraflores

Marca actual:

Example Professional

Estilistas:

5

```

The workflow behavior can then be demonstrated without exposing an actual customer.

Synthetic data is particularly useful for:

```text

Portfolio screenshots

Architecture walkthroughs

Demo videos

Recruiter presentations

Public workflow tests

```

## Screenshot sanitization

Screenshots require the same security review as code.

Before publishing an image, inspect it for:

```text

Phone numbers

Email addresses

Names

Chat IDs

Credential names

API values

URLs

Database identifiers

Google Drive IDs

Sheet IDs

Browser tabs containing private information

Sidebars showing customer data

```

Cropping a screenshot is not always sufficient.

Sensitive information should be removed or replaced before publication.

### Sanitized production evidence

Public portfolio material may include real production screenshots when sensitive information has been removed.

The publication boundary is:

```text

Real interface / workflow evidence

        +

PII removed

        +

credentials removed

        +

private identifiers removed

        ↓

Public-safe production evidence
```

Raw customer conversations, raw operational exports and unsanitized production records remain private.

Synthetic examples are preferred when real data is unnecessary, but sanitized production excerpts may be used when they materially demonstrate system behavior without identifying customers or exposing operational access.

## URLs and identifiers

Not every identifier is a secret, but unnecessary production identifiers should still be removed from public artifacts.

Examples include:

```text

Private webhook URL

Supabase project reference

Google Sheet ID

Google Drive file ID

Production WhatsApp number

Internal document identifier

Credential ID

```

Removing these values reduces unnecessary exposure and makes the public artifact easier to reuse safely.

## Principle of minimum public exposure

A useful publication rule is:

```text

Publish what demonstrates engineering.

Keep private what enables access or identifies people.

```

Examples:

| Information | Public? |

|---|---|

| Architecture | Yes |

| State machine | Yes |

| Routing methodology | Yes |

| Sanitized code | Yes |

| Aggregate metrics | Yes |

| Synthetic demo conversations | Yes |

| API secret | No |

| Customer phone | No |

| Real chat history | No |

| Database password | No |

| Private webhook URL | No |

| Raw CRM export | No |

## Integrity evidence

Security also includes preserving evidence of what the original artifact was.

A SHA-256 fingerprint can be retained for the private production workflow.

Conceptually:

```text

Original private workflow

        ↓

SHA-256

        ↓

Immutable fingerprint stored in repository

```

This allows the existence of a specific artifact to be documented without committing the artifact itself.

The repository contains an evidence fingerprint for the original production export.

A hash does not prove authorship or automatically establish intellectual-property ownership.

It provides integrity evidence that a particular file corresponds to the recorded fingerprint.

## Why hashes are useful

Suppose a private file is retained separately.

Later:

```text

Current file

        ↓

Calculate SHA-256

        ↓

Compare with stored fingerprint

```

If the hashes match, the file content has not changed.

This is useful for preserving historical technical evidence while keeping sensitive production material private.

## Security vs attribution

Security documentation and authorship evidence solve different problems.

```text

Security

        ↓

Who should be able to access this information?


Attribution

        ↓

What work was performed and how is it documented?

```

Protecting private production data does not weaken technical attribution.

In many cases, intentionally sanitizing customer and credential information demonstrates stronger engineering judgment than publishing everything.

## Human review before public release

Automated scanning alone is not enough.

Before changing repository visibility or publishing artifacts, a manual review should inspect:

```text

README

Docs

Workflow JSON

Screenshots

Architecture diagrams

Git history

Evidence files

Example payloads

Code blocks

Local configuration accidentally committed

```

The reviewer should explicitly search for:

```text

Secrets

PII

Private URLs

Internal IDs

Customer conversations

Commercially sensitive source material

```

## Git history matters

Deleting a secret from the latest version of a file does not necessarily remove it from Git history.

If a real credential was ever committed:

```text

Commit A

contains secret

Commit B

removes secret

```

the credential may still exist inside:

```text

Commit A

```

For a credential exposed through Git history, the correct response is:

```text

Rotate credential

        ↓

Treat old credential as compromised

        ↓

Remove sensitive history if appropriate

```

Simply deleting the visible text is not sufficient.

## Current repository strategy

The repository is being developed privately first.

The intended publication sequence is:

```text

Complete technical documentation

        ↓

Prepare sanitized workflow

        ↓

Prepare synthetic screenshots / demo

        ↓

Scan repository for secrets

        ↓

Review Git history

        ↓

Review PII and private identifiers

        ↓

Final manual inspection

        ↓

Only then consider public visibility

```

This prevents publication pressure from overriding security hygiene.

## Security review checklist

Before a public release:

```text

[ ] No API keys

[ ] No passwords

[ ] No Bearer tokens

[ ] No database credentials

[ ] No customer phone numbers

[ ] No customer email addresses

[ ] No real conversation transcripts

[ ] No private webhook URLs

[ ] No unnecessary production IDs

[ ] No raw Supabase exports

[ ] No raw CRM exports

[ ] No private RAG source documents

[ ] No internal customer databases

[ ] Workflow JSON sanitized

[ ] Screenshots sanitized

[ ] Git history reviewed

[ ] Synthetic data used in demos

```

## Production troubleshooting without exposing secrets

Debugging should avoid copying real credentials into documentation or chat logs.

A useful diagnostic pattern is:

```text

Authentication failed

        ↓

Record status code

        ↓

Record missing header / credential type

        ↓

Fix credential configuration

```

rather than:

```text

Paste full API secret into troubleshooting document

```

For example:

```text

401

Missing X-API-Key

```

is useful technical evidence.

The API key value itself is not.

## Security as part of engineering quality

A technically impressive workflow can still be poor engineering if publishing it leaks:

```text

Customer data

Production credentials

Private infrastructure

Commercial information

```

Security hygiene is therefore part of the portfolio quality of this project.

A recruiter or engineer should be able to inspect:

```text

Architecture

Business logic

State design

RAG design

QA methodology

Handoff architecture

Integration decisions

```

without receiving access to the production environment.

## Final security model

```text

PRIVATE ENVIRONMENT

│

├── Production credentials

├── Original workflow

├── Customer conversations

├── CRM records

├── Supabase exports

├── RAG source documents

└── Operational identifiers

                ↓

        Sanitization boundary

PUBLIC TECHNICAL EVIDENCE

│

├── Architecture

├── State model

├── Routing design

├── RAG methodology

├── Aggregated metrics

├── Sanitized workflow

├── Synthetic demos

└── Engineering decisions

```

## Engineering lesson

The key security lesson from the project is that production evidence and public evidence do not need to be the same artifact.

A strong technical portfolio does not require exposing the production environment.

The better approach is:

```text

Preserve original evidence privately

        +

Publish sanitized technical evidence

        +

Document the security decisions

```

This protects customers and infrastructure while still demonstrating the engineering work behind the system.
