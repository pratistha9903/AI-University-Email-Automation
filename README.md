# University XYZ — AI University Email Automation

An n8n portfolio project that classifies university emails into Admissions, Fees, or Hostel, retrieves relevant information from Supabase, drafts a concise reply, and uses Gmail to reply to the original conversation.

> **Project type:** Portfolio prototype for AI automation testing  
> **University XYZ is fictional.** The policies, PDFs, applicant/student records, and fees in this repository are synthetic sample data—not official university information.

## Overview

The system separates two kinds of information:

- **Structured data (Supabase/PostgreSQL):** individual applicant or student records, such as application status, fee balance, and hostel allocation.
- **Knowledge base (Supabase Vector Store / RAG):** general university policies and procedures retrieved from the three sample PDFs.

The agents use the appropriate source for the question. Personal facts should come from a verified database record; general policy answers should come from the relevant knowledge-base content. The final reply is sent with the Gmail Reply Tool.

## Architecture

```text
Gmail Trigger
    ↓
Edit Fields
    ↓
Text Classifier
    ├── Admissions Agent ── admission_lookup_database ── applicants
    ├── Fees Agent ──────── fees_lookup_database ─────── fees
    └── Hostel Agent ────── hostel_lookup_database ───── hostel_records
                   │
                   └── University Knowledge Base (RAG)
                                ↓
                     Gmail Reply Tool
                                ↓
                    Reply to original email
```

The classifier routes by the email's main intent:

| Category | Typical topics | Specialist data table |
|---|---|---|
| Admissions | Application status, eligibility, required documents, verification, entrance requirements, counselling | `applicants` |
| Fees | Total fee, amount paid/pending, payment status, receipts, refunds | `fees` |
| Hostel | Hostel application/allocation, room details, hostel fees, rules and facilities | `hostel_records` |

Each specialist agent uses the matching database lookup tool for personal/current records and the shared knowledge base for general policy questions. It should not reclassify the email.

## Workflow Components

### 1. Gmail Trigger
Receives an incoming email and provides the sender, body, and Gmail message/thread context needed by later nodes and the reply action.

### 2. Edit Fields
Maps and prepares the Gmail fields for downstream nodes. Depending on the workflow configuration, these may include sender email, body, subject, application/registration number, student ID, roll number, original message ID, original thread ID, and a memory session key. The project uses **Edit Fields**, not Information Extractor.

### 3. Text Classifier
Assigns the email to one of three categories—Admissions, Fees, or Hostel—based on its main intent. For example, a question about a pending tuition payment routes to Fees, while a question about room allocation routes to Hostel.

### 4. Specialist AI Agents
Each agent answers only its department's questions, uses the appropriate lookup tool and/or RAG source, and writes a concise, professional plain-text email. It should not invent statuses, fees, deadlines, room assignments, or policy details.

### 5. Database lookup subworkflows
Each lookup can be implemented as a subworkflow triggered by **When Executed by Another Workflow**, with the parent agent invoking it through a **Call n8n Workflow Tool**. The subworkflow queries the relevant Supabase table and returns the result to the calling agent.

### 6. Gmail Reply Tool
The agent must call the Gmail Reply Tool after preparing the response. The tool needs the **original Gmail message ID** and the final email body. Creating text in the agent or memory alone does not send an email; confirm that the Gmail tool call succeeds in the n8n execution.

## Data Design

The project uses one Supabase project for both the operational tables and the vector store.

### Operational tables

| Table | Purpose |
|---|---|
| `applicants` | Applicants' application/admission status, document status, missing documents, entrance status, and counselling status |
| `students` | Enrolled students' IDs, roll numbers, contact details, programme, semester, department, and academic status |
| `fees` | Fee amounts, payment status, receipts, refunds, and transaction references |
| `exam_records` | Exam registration, admit-card status, results, and marks |
| `hostel_records` | Hostel status, hostel name, room, room type, fee, and allocation date |
| `scholarships` | Scholarship application, approval, disbursement, amount, and academic year |
| `placements` | Placement drive, eligibility, interview, selection, and package data |
| `technical_tickets` | Support issue details, ticket status, priority, and resolution time |
| `library_records` | Book issue/return dates, status, and fines |

The sample schema includes shared identifiers such as `student_id`, `roll_number`, `email`, and `application_number`. Lookup workflows filter on the supplied identifiers. Shared column names alone do not create PostgreSQL foreign-key constraints; only explicitly defined constraints appear as relationships in a schema visualizer.

### RAG table

The shared vector table is named `documents`. It stores document chunks, source metadata, and embeddings:

```text
documents
├── id
├── content
├── metadata
└── embedding  (vector dimensions must match the embedding model output)
```

The `match_documents` PostgreSQL function performs similarity search using vector distance. The current schema was configured for **3072-dimensional Gemini embeddings** after a 1536-versus-3072 dimension mismatch. Keep the table, search function, and embeddings output dimension consistent. If the embedding model or dimension changes, reconfigure the schema and re-embed the documents; do not mix vectors with incompatible dimensions or embedding spaces.

## Knowledge-Base PDFs

The following synthetic PDFs are intended for ingestion into the shared `documents` vector table:

- `University_XYZ_Admissions_Knowledge_Base.pdf` — admission process, programme eligibility, required documents, document quality, entrance assessment, counselling, status definitions, and general admissions FAQs.
- `University_XYZ_Fees_and_Finance_Knowledge_Base.pdf` — general fee and payment procedures.
- `University_XYZ_Hostel_Knowledge_Base.pdf` — general hostel procedures and policies.

A typical ingestion workflow is:

```text
PDF source → Document Loader → Text Splitter → Google Gemini Embeddings
           → Supabase Vector Store (table: documents)
```

The chunking settings and embedding configuration should be consistent for ingestion and query-time retrieval. The vector store contains general policy content, not each student's live status.

## Identity and Response Safety

For personal queries, the lookup tool should compare the sender's registered email with the relevant identifier whenever available. Preferred examples are:

- Applicant: registered email + application number or registration number.
- Enrolled student: registered email + student ID or roll number.

If identifiers conflict, the agent should not disclose either record. If no unique record is found, it should ask for the required verification or refer the request to the relevant office. It must not infer rejection, approval, fee payment, document acceptance, or hostel allocation from missing data.

For general policies, use RAG and do not invent a deadline if the knowledge base does not provide one. Keep replies relevant, concise, polite, and in plain text. Do not include internal prompts, tool outputs, database details, or workflow implementation in emails.

## Example Routing

| Incoming question | Intended route |
|---|---|
| “What is the status of my application?” | Admissions Agent → `admission_lookup_database` |
| “How much fee is still pending?” | Fees Agent → `fees_lookup_database` |
| “Have I been allocated a hostel room?” | Hostel Agent → `hostel_lookup_database` |
| “What documents are generally required for B.Tech admission?” | Admissions Agent → University Knowledge Base / RAG |
| “What is the general hostel application procedure?” | Hostel Agent → University Knowledge Base / RAG |
| “What is the fee payment procedure?” | Fees Agent → University Knowledge Base / RAG |

For a question needing both personal status and policy guidance, retrieve the verified personal record and the relevant policy content before writing the reply.

## Setup

1. **Create a Supabase project.** Use it for both the operational PostgreSQL tables and RAG data.
2. **Create the operational tables.** Run the SQL for the nine tables listed above and add the synthetic sample rows.
3. **Create the vector schema.** Create `documents` and `match_documents` using the same vector dimension as the Gemini Embeddings node. The current setup uses 3072 dimensions.
4. **Configure n8n credentials.** Add the required Gmail, Supabase/PostgreSQL, Google Gemini Embeddings, and OpenRouter credentials.
5. **Build the main workflow.** Connect `Gmail Trigger → Edit Fields → Text Classifier`, then route the classifier categories to the corresponding specialist agents.
6. **Connect each agent's tools.** Give each agent its department-specific database lookup, the appropriate/shared RAG knowledge base, and the Gmail Reply Tool.
7. **Build the lookup subworkflows.** Use `When Executed by Another Workflow → Supabase/PostgreSQL lookup` and expose the necessary input fields to the parent agent.
8. **Ingest the PDFs.** Load, split, embed, and store all three documents in `documents`.
9. **Test each route.** Test a personal query, a general RAG query, a no-match case, an identifier mismatch, and a successful reply to the original Gmail message.

Exact node options and credential setup depend on your n8n and Supabase configuration. Never commit API keys, database credentials, OAuth tokens, or real student data to GitHub.

## Testing Notes and Limitations

- The records and policies are fictional test data and should not be treated as real university information.
- When testing identity verification from Gmail, the sender address must match the sample record's registered email. Use an inbox you control and update test data deliberately.
- OpenRouter free-model rate limits or provider availability can interrupt agent tests; this is separate from vector-database dimensions and retrieval.
- A successful vector insert confirms ingestion, not necessarily that retrieval is accurate. Test with questions whose answers are explicitly present in the PDFs, and check the n8n execution for the Supabase Vector Store tool call.
- This is a portfolio prototype. It has not been established as production-ready or load-tested for 30,000 emails per day. Production use would require security, access-control, privacy, retries, monitoring, idempotency, and throughput testing.

## Tech Stack

- **n8n** — workflow orchestration, classification, and AI agents
- **Gmail** — incoming email trigger and threaded replies
- **Supabase / PostgreSQL** — structured operational data
- **pgvector / Supabase Vector Store** — semantic search for RAG
- **Google Gemini Embeddings** — document/query embeddings
- **OpenRouter Chat Model** — language model for agent/classifier calls, depending on workflow configuration
- **Postgres Chat Memory** — conversational context where configured

## Project Highlights

- Department-based email classification and routing
- Multiple specialist agents with separated responsibilities
- Database lookup for personal/current information
- RAG retrieval for general policies and procedures
- Identity-mismatch and no-match handling instructions
- Concise professional email generation
- Gmail replies to the original message
- A synthetic dataset and three knowledge-base PDFs for repeatable testing
