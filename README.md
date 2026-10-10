# University XYZ — AI University Email Automation

An n8n portfolio prototype that classifies incoming university emails into Admissions, Fees, or Hostel, retrieves relevant information from Supabase, drafts a concise response, and sends it using Gmail.

> **Demo project:** University XYZ, its policies, PDFs, applicants, students, payments, and hostel records are fictional sample data created for automation and RAG testing. They are not official university information.

## Overview

The workflow keeps two information sources separate:

- **Supabase/PostgreSQL:** applicant- and student-specific records, such as admission status, fee balances, and hostel allocation.
- **Supabase Vector Store (RAG):** general university policies and procedures retrieved from three knowledge-base PDFs.

The Text Classifier routes each email to a specialist AI Agent. The agent uses the relevant lookup workflow for personal records, the shared vector knowledge base for general policy questions, and the Gmail tool to send its response.

---

## Workflow

<img width="892" height="512" alt="Image" src="https://github.com/user-attachments/assets/3f82f42e-40b7-4238-8c50-8ddd9ccd26e8" />

---

## Architecture

### Incoming email path

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
              ├── Supabase Vector Store (general policy retrieval)
              └── Gmail sending tool
```

Each agent has an OpenRouter Chat Model and Postgres Chat Memory connected in n8n. The agents are routed by the classifier; they do not classify the email again.

### Knowledge-base ingestion path

```text
Google Drive Trigger
    ↓
Download File
    ↓
Default Data Loader + Text Splitter
    ↓
Google Gemini Embeddings
    ↓
Supabase Vector Store → documents
```

The same `documents` vector table is used for all three PDFs. The database lookup subworkflows use this pattern:

```text
When Executed by Another Workflow
    ↓
Supabase: Get Many Rows
```

The specialist agent calls the corresponding lookup subworkflow through a Call n8n Workflow Tool.

## Email Categories and Agents

| Category | Example topics | Personal-data source |
|---|---|---|
| **Admissions** | Application/admission status, eligibility, required documents, verification, entrance requirements, counselling | `applicants` via `admission_lookup_database` |
| **Fees** | Total fee, amount paid or pending, payment status, receipts, refunds, transactions | `fees` via `fees_lookup_database` |
| **Hostel** | Hostel application/allocation, room details, hostel fee, rules, facilities, room changes | `hostel_records` via `hostel_lookup_database` |

The shared RAG knowledge base is used for general policy questions, such as the admission process, fee procedures, or hostel rules. For questions needing both a personal record and a general policy, the agent should use both sources.

## Data Design

The project uses one Supabase project for operational PostgreSQL tables and the vector store.

| Table | Purpose |
|---|---|
| `applicants` | Application number, registration number, contact details, program, application/admission/document status, missing documents, entrance status, and counselling status |
| `students` | Student ID, roll number, email, program, semester, department, and academic status |
| `fees` | Total fee, amount paid/pending, payment status, receipt/refund status, and transaction ID |
| `exam_records` | Semester, exam name, registration status, admit-card status, result, and marks |
| `hostel_records` | Hostel status, hostel name, room number/type, hostel fee, and allocation date |
| `scholarships` | Scholarship application, approval, disbursement, amount, and academic year |
| `placements` | Placement drive, eligibility, interview, selection, and package information |
| `technical_tickets` | Support issue, description, priority, status, and resolution time |
| `library_records` | Book issue/return dates, status, and fine amount |
| `documents` | Text chunks, metadata, and vector embeddings used for semantic search |

The operational tables contain lookup identifiers such as `student_id`, `roll_number`, `email`, and `application_number`. The sample SQL uses these fields for filtered lookups; shared column names alone do not create PostgreSQL foreign-key constraints.

### RAG vector schema

The `documents` table stores the extracted text chunks and their metadata. The `match_documents` function orders results by vector similarity using cosine distance.

```text
documents
├── id
├── content
├── metadata
└── embedding  (vector dimension must match Gemini output)
```

The setup was changed to **3072-dimensional vectors** after the Gemini node returned 3072 dimensions while the original table expected 1536. Keep the `documents.embedding` column and `match_documents` query parameter aligned with the dimension produced by the embeddings node. Re-embed documents if the model or embedding configuration changes; do not mix incompatible embeddings.

## Knowledge-Base PDFs

Ingest these synthetic documents into the shared vector table:

- `University_XYZ_Admissions_Knowledge_Base.pdf` — admission process, program eligibility, documents, document quality, entrance assessment, counselling, status definitions, deadlines, and FAQs.
- `University_XYZ_Fees_and_Finance_Knowledge_Base.pdf` — general fee and payment information.
- `University_XYZ_Hostel_Knowledge_Base.pdf` — general hostel information and procedures.

These PDFs provide general policy information only. They are not the source of truth for an individual applicant's live application status, fee balance, or room allocation.

## Identity and Response Handling

For personal queries, the agents are instructed to verify the sender against available identifiers before disclosing a record. Preferred examples are:

- **Applicant:** registered email + application number or registration number.
- **Enrolled student:** registered email + student ID or roll number.

If identifiers conflict or no unique record is found, the intended behavior is to avoid disclosing personal data and request verification or escalate the case. Agents should not infer admission decisions, payments, missing documents, deadlines, or hostel allocation from unsupported information.

Emails are intended to be concise, polite, plain text, and limited to information relevant to the question. The Gmail tool is responsible for actually sending the composed response. Confirm the Gmail node's operation and message/thread ID mapping to ensure replies go to the intended original conversation.

## Example Routing

| Incoming email | Expected source |
|---|---|
| “What is the status of my application?” | Admissions Agent → `admission_lookup_database` |
| “How much fee is still pending?” | Fees Agent → `fees_lookup_database` |
| “Have I been allocated a hostel room?” | Hostel Agent → `hostel_lookup_database` |
| “What documents are generally required for B.Tech admission?” | Admissions Agent → Supabase Vector Store |
| “What is the general fee payment procedure?” | Fees Agent → Supabase Vector Store |
| “What are the hostel application rules?” | Hostel Agent → Supabase Vector Store |

Useful tests include a verified personal query, a general question answered from a PDF, an unknown identifier, an identifier mismatch, and confirmation that the Gmail tool is actually called and completes successfully.

## Setup

1. **Create a Supabase project.** Use the same project for operational tables and RAG.
2. **Create the operational tables.** Run the sample PostgreSQL SQL and insert the synthetic sample records.
3. **Create the vector schema.** Create `documents` and `match_documents` with a vector dimension matching the Gemini Embeddings node (3072 in the current setup).
4. **Configure n8n credentials.** Connect Gmail, Supabase/PostgreSQL, Google Gemini Embeddings, and OpenRouter.
5. **Build the email path.** Configure `Gmail Trigger → Edit Fields → Text Classifier`, then connect the three classifier outputs to their specialist agents.
6. **Configure the agents and lookup tools.** Each agent needs its department-specific lookup workflow, the Supabase Vector Store tool, and the Gmail sending tool. The lookup subworkflows begin with `When Executed by Another Workflow` and then query the relevant table.
7. **Ingest the PDFs.** Use the Google Drive/document-loader path, split the documents into chunks, embed them with Gemini, and store them in `documents`.
8. **Test end to end.** Inspect n8n executions to confirm the right agent/tool runs, the retrieved data is grounded, and the Gmail node completes.

Exact node options depend on the n8n version and the way credentials and input fields are configured. Never commit API keys, credentials, OAuth tokens, private email data, or database secrets to GitHub.

## Limitations

- Uses synthetic sample records and fictional policies; it must not be used as a real university service.
- The workflow currently routes only Admissions, Fees, and Hostel categories.
- OpenRouter free-model limits or provider availability can interrupt testing; this is separate from the RAG database and vector dimensions.
- Successful document ingestion does not by itself prove retrieval quality; test with questions whose answers appear explicitly in the PDFs and inspect the tool calls.
- This is a portfolio prototype, not a production-validated or load-tested system. Production use would require stronger access controls, privacy review, retries, monitoring, idempotency, and throughput testing.

## Tech Stack

- **n8n** — workflow orchestration, classification, and AI agents
- **Gmail** — incoming email trigger and outbound response
- **Supabase / PostgreSQL** — structured operational data
- **pgvector / Supabase Vector Store** — semantic retrieval for RAG
- **Google Gemini Embeddings** — document and query embeddings
- **OpenRouter Chat Model** — LLM used by the classifier/agents, depending on node configuration
- **Postgres Chat Memory** — conversation context where configured

## Project Highlights

- Three-way email classification and specialist routing
- Separate lookup subworkflows for admissions, fees, and hostel records
- Shared RAG knowledge base for general university policies
- Database-backed personal responses with identity-verification instructions
- Concise professional email generation and Gmail sending
- Synthetic data and repeatable cases for testing RAG, lookups, and error handling
