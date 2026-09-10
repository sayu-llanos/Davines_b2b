# RAG and Knowledge Architecture

The B2B WhatsApp assistant was designed to answer both commercial and technical questions without relying only on the language model's general knowledge.

To achieve this, the production system used a Retrieval-Augmented Generation (RAG) layer built around curated Davines B2B knowledge.

The objective was to give the assistant access to information that was specific to the brand, professional products, commercial process and qualification model.

```text
Internal Davines knowledge
        ↓
Document preparation
        ↓
Text extraction
        ↓
Chunking
        ↓
Embeddings
        ↓
Supabase Vector Store
        ↓
Semantic retrieval
        ↓
AI Agent
        ↓
Context-aware response
```

## Knowledge sources

Three main knowledge sources were prepared for the B2B system.

### Technical knowledge base

The technical knowledge base contained product and professional-use information relevant to Davines.

Its purpose was to support questions related to topics such as:

```text
Product lines
Product purpose
Hair needs
Professional use
Technical characteristics
Recommendations
Usage context
```

This allowed the assistant to answer more specialized questions than a generic sales chatbot.

The source material was reviewed and prepared before being used by the retrieval layer.

### B2B Commercial Manual

The commercial knowledge source contained information relevant to the professional B2B relationship.

Its purpose was different from the technical knowledge base.

Instead of focusing primarily on product behavior, it supported the assistant's understanding of:

```text
Commercial context
Professional positioning
B2B value proposition
How Davines works with salons and professionals
Relevant commercial information
Sales-oriented explanations
```

This allowed the assistant to communicate with professional prospects using context specific to the business model.

### B2B Qualification System

A separate qualification framework was created to structure how professional prospects should be understood and qualified.

This knowledge included the concepts needed to distinguish and progressively capture information such as:

```text
Professional profile
Salon context
City and district
Number of stylists
Current professional brand
Commercial motivation
Contact information
Preferred follow-up channel
```

The qualification model complemented the deterministic routing and Supabase state architecture.

Together, these sources gave the system three different forms of knowledge:

```text
Technical knowledge
        +
Commercial knowledge
        +
Qualification knowledge
```

## Knowledge preparation

The source material was not treated as raw documents that could simply be given to the language model without preparation.

The knowledge was reviewed, cleaned and structured before retrieval.

The process included:

```text
Source document
        ↓
Review / cleanup
        ↓
Text extraction
        ↓
Structured content
        ↓
Chunking
        ↓
Embedding generation
        ↓
Vector storage
```

This preparation was necessary because the assistant needed to retrieve specific sections of knowledge rather than send complete documents into every conversation.

## Production ingestion workflow

The production n8n workflow contains the components required for document ingestion.

The architecture includes:

```text
Google Drive
        ↓
File retrieval
        ↓
Extract from File
        ↓
Default Data Loader
        ↓
Character Text Splitter
        ↓
OpenAI Embeddings
        ↓
Supabase Vector Store
```

This pipeline transforms source documents into smaller searchable knowledge units.

## Chunking

Long source documents are divided into smaller text chunks before embeddings are generated.

Conceptually:

```text
Large document
        ↓
Chunk 1
Chunk 2
Chunk 3
Chunk 4
...
        ↓
Each chunk becomes independently searchable
```

Chunking improves retrieval because a user usually needs one relevant section of a document rather than the entire source.

For example:

```text
User:
"¿Qué línea recomiendan para cabello muy seco?"
```

The system does not need to load the complete technical manual.

Instead, the retrieval layer can search for chunks semantically related to:

```text
dryness
nutrition
damage
relevant Davines line
professional recommendation
```

and provide that context to the AI Agent.

## Embeddings

After chunking, the production workflow generates embeddings using OpenAI embedding nodes.

An embedding represents the semantic meaning of a text chunk as a vector.

Conceptually:

```text
Text chunk
        ↓
Embedding model
        ↓
Vector representation
```

This allows retrieval to work by semantic similarity rather than requiring exact keyword matches.

For example:

```text
"cabello reseco"

"cabello deshidratado"

"necesito nutrición"

"mi cabello está muy seco"
```

may express related needs even though the wording is different.

Vector retrieval makes it possible to search by meaning.

## Supabase Vector Store

The generated chunks and embeddings are stored in Supabase.

The production `documents` table contains fields including:

| Field | Role |
|---|---|
| `id` | Document/chunk identifier |
| `content` | Searchable text content |
| `source` | Source information |
| `tipo` | Knowledge/content classification |
| `embedding` | Semantic vector representation |
| `metadata` | Additional structured metadata |

Conceptually:

```text
documents
│
├── content
├── source
├── tipo
├── embedding
└── metadata
```

The `embedding` field allows the Supabase Vector Store to perform semantic similarity search.

## Retrieval during conversation

The retrieval layer is used when the conversation requires knowledge that benefits from the curated document base.

Conceptually:

```text
User question
        ↓
Routing identifies technical / product context
        ↓
Semantic search
        ↓
Relevant document chunks
        ↓
AI Agent receives retrieved context
        ↓
Response grounded in Davines knowledge
```

This separates two responsibilities:

```text
Router
"What kind of interaction is this?"

RAG
"What relevant knowledge should the model receive?"
```

The router controls workflow behavior.

The retrieval layer provides domain knowledge.

## RAG is not the same as conversation memory

The architecture intentionally separates three different forms of stored information.

```text
n8n_chat_histories
        ↓
What has been said in the conversation


sesiones_bot
        ↓
What the application knows about the lead and workflow state


documents
        ↓
What the system knows about Davines products,
commercial context and domain knowledge
```

These systems solve different problems.

Conversation memory cannot replace a product knowledge base.

A vector database cannot replace qualification state.

Supabase session state cannot replace natural conversational history.

The production system therefore uses each storage layer for a specific responsibility.

## Why RAG was used

A general-purpose language model may know broad information about hair care, but the B2B assistant needed to operate using business-specific material.

RAG allows the system to use curated context when answering questions related to Davines.

Without retrieval:

```text
User question
        ↓
LLM general knowledge
        ↓
Possible generic answer
```

With retrieval:

```text
User question
        ↓
Relevant Davines knowledge retrieved
        ↓
LLM receives brand-specific context
        ↓
More grounded response
```

The goal was not to make the language model memorize the documents.

The goal was to retrieve the right information when it was needed.

## Technical and commercial knowledge together

One of the important characteristics of the system is that it was not designed only as a technical FAQ assistant.

It also participated in the commercial journey.

A prospect could ask questions such as:

```text
What is this product for?
Which line is appropriate for this need?
How is the product used professionally?
What options does Davines offer?
How can I work with the brand?
What commercial information should I know?
```

The architecture therefore combined product education and commercial qualification inside the same conversational experience.

```text
Technical question
        ↓
Knowledge retrieval
        ↓
Product education


Commercial signal
        ↓
Routing
        ↓
Qualification


Both can occur
inside the same conversation
```

## Relationship with multimedia

The RAG layer and multimedia delivery serve related but different purposes.

RAG provides textual knowledge context to the AI.

Multimedia delivery provides concrete commercial or educational assets to the prospect.

```text
RAG
        ↓
Explain relevant information


Multimedia
        ↓
Show / send relevant material
```

Depending on conversation context, the system could combine:

```text
Natural-language explanation
+
Technical knowledge
+
Commercial information
+
Relevant catalog / PDF / image / video
```

This made the assistant more useful than a text-only FAQ bot.

## Existing-salon data

The project also used a structured B2B salon dataset containing information about existing Davines salons.

This dataset served a different operational purpose from the vector knowledge base.

It helped provide context for recognizing or working with existing professional relationships.

It should therefore be conceptually separated from the RAG documents:

```text
RAG documents
        ↓
Knowledge retrieval


Existing salon dataset
        ↓
Operational / customer context
```

This distinction is also relevant to the `CLIENTE_ACTUAL` routing behavior documented in the conversation-routing architecture.

## Knowledge source vs production data

Several files used during development should not be confused with RAG source material.

For example:

```text
n8n_chat_histories export
        ↓
Conversation history / debugging evidence

sesiones_bot export
        ↓
Application-state evidence

CRM export
        ↓
Commercial operations

Technical / commercial / qualification documents
        ↓
Knowledge sources
```

This separation is important because training or retrieval data, operational state and customer records have different security and privacy requirements.

## Human curation

The knowledge layer was not created by automatically uploading arbitrary files.

The source information had to be reviewed, selected and organized around the actual B2B use case.

The implementation process included:

```text
Understand the business material
        ↓
Identify useful information
        ↓
Prepare the documents
        ↓
Structure knowledge for retrieval
        ↓
Test questions
        ↓
Observe weak answers
        ↓
Improve source material / routing
        ↓
Test again
```

This iterative process was particularly important because product knowledge and sales language had to remain useful to professional salon prospects.

## Why not put everything inside the system prompt?

A large static prompt could contain some product and commercial information, but this approach becomes difficult to maintain as knowledge grows.

Conceptually:

```text
Huge system prompt
        ↓
More tokens
Harder updates
Harder source separation
Less targeted context
```

RAG instead allows:

```text
Large knowledge base
        ↓
Retrieve only relevant chunks
        ↓
Provide targeted context
```

This makes the knowledge architecture easier to extend and maintain.

## Updating knowledge

The ingestion architecture also makes the knowledge base updateable.

When source material changes, the intended process is:

```text
Update source document
        ↓
Reprocess content
        ↓
Rechunk
        ↓
Generate new embeddings
        ↓
Update vector store
```

This separates knowledge maintenance from the core conversational routing logic.

The assistant can therefore evolve as product or commercial information changes without redesigning the entire workflow.

## Engineering tradeoff

RAG improves grounding but does not automatically guarantee a correct response.

Retrieval quality depends on several factors:

```text
Source quality
Chunk quality
Embedding quality
Search relevance
Prompt instructions
Conversation context
Routing accuracy
```

For this reason, RAG was treated as one component of the larger architecture rather than as a complete solution by itself.

The system combines:

```text
Deterministic routing
+
Persistent state
+
Conversation memory
+
RAG
+
Human review
```

## Security and publication

The original source documents and production exports are not intended to be committed directly to the public repository.

Private project material can contain:

```text
Internal commercial information
Customer or salon information
Conversation history
Phone numbers
Email addresses
Internal documents
Operational identifiers
```

The public repository should document the architecture and engineering methodology without publishing confidential source material.

The production knowledge sources can therefore be represented by descriptions such as:

```text
Technical knowledge base
B2B Commercial Manual
B2B Qualification Framework
```

without exposing the underlying private documents.

## Private evidence retained separately

Original project artifacts can be retained privately as evidence of the implementation while remaining outside Git tracking.

Examples include:

```text
Technical source documents
Commercial source documents
Qualification documentation
CRM exports
Supabase state exports
Conversation-history exports
Existing-salon datasets
```

These artifacts support the historical and technical record of the project but should remain separate from the public codebase.

## Final knowledge architecture

The production knowledge architecture can be summarized as:

```text
Davines internal knowledge
        │
        ├── Technical knowledge
        ├── B2B commercial knowledge
        └── Qualification knowledge
                ↓
        Document preparation
                ↓
        Google Drive / source storage
                ↓
        Text extraction
                ↓
        Character-based chunking
                ↓
        OpenAI embeddings
                ↓
        Supabase Vector Store
                ↓
        Semantic retrieval
                ↓
        AI Agent
                ↓
        Contextual product / commercial response
                ↓
        Router continues workflow
                ↓
        State persisted in Supabase
```

## Engineering lesson

The key lesson from the knowledge layer was that a useful commercial AI assistant needs more than access to a language model.

The system needed separate mechanisms for:

```text
Understanding conversation
        ↓
LLM

Knowing the business
        ↓
RAG

Remembering workflow state
        ↓
Supabase

Remembering conversation history
        ↓
Postgres chat memory

Executing commercial rules
        ↓
Deterministic routing

Escalating important opportunities
        ↓
Human sales team
```

RAG therefore became one part of a larger stateful B2B conversational architecture rather than the entire system.