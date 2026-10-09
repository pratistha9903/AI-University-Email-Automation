# University XYZ — AI University Email Automation

An n8n-based AI email automation system that reads university emails, classifies them into departments, retrieves the right information from structured data and a RAG knowledge base, generates a concise professional reply, and sends the reply back to the original Gmail conversation.

> **Project status:** Working prototype / portfolio project  
> **University:** University XYZ (synthetic/sample data)  
> **Important:** University XYZ, its policies, applicants, students, fees, hostel records, and sample documents are fictional and were created for AI automation testing. They are not official university records.

---

## Overview

The system is designed around a simple principle:

**General university information comes from RAG. Personal/current information comes from the structured database.**

The AI agent does not answer applicant-specific questions from general knowledge. It first retrieves the relevant record from the database, verifies the available identifiers, and then uses the result as the source of truth.

For general policy questions, the agent searches the University XYZ knowledge base stored in Supabase as vector embeddings.

After the answer is prepared, the agent uses the Gmail Reply Tool to reply to the original email thread.

---

## Main Workflow

```text
Gmail Trigger
      |
      v
Edit Fields
      |
      v
Text Classifier
      |
      +-------------------+-------------------+
      |                   |                   |
      v                   v                   v
Admissions Agent      Fees Agent         Hostel Agent
      |                   |                   |
      |                   |                   |
      +---------+---------+---------+---------+
                |                   |
                v                   v
        Structured Database      RAG Knowledge Base
                |                   |
                +---------+---------+
                          |
                          v
                   Final Email Reply
                          |
                          v
                  Gmail Reply Tool
                          |
                          v
                     Email Sent
```

---

## Why the project uses both a database and RAG

These two sources have different responsibilities.

### Structured PostgreSQL / Supabase Database

Used for live or applicant/student-specific information such as:

- application status
- admission status
- missing documents
- fee balance
- payment status
- hostel allocation
- room details
- exam records
- scholarships
- placements
- technical tickets
- library records

### RAG / Supabase Vector Store

Used for university-wide information such as:

- eligibility criteria
- admission procedures
- required documents
- document-quality requirements
- fee policies and procedures
- hostel rules and procedures
- counselling rules
- verification procedures
- general university policies

This separation prevents the model from using a general policy document to answer a personal question such as "What is my current fee balance?".

---

# Workflow Details

## 1. Gmail Trigger

The workflow starts when a new email is received.

The Gmail trigger provides the information needed by downstream nodes, including the sender, email body, and Gmail message/thread context.

The original Gmail message ID is retained so the final response can be sent as a reply to the original conversation.

---

## 2. Edit Fields

The project uses **Edit Fields**, not an Information Extractor.

The Edit Fields node prepares and maps the incoming Gmail data for the rest of the workflow.

Typical values passed downstream include:

- sender email
- email body
- subject
- application number, when present
- registration number, when present
- student ID, when present
- roll number, when present
- original Gmail message ID
- original Gmail thread ID
- session information used by memory, when required

The purpose of this node is to create a clean, predictable input for the classifier and specialist agents.

---

## 3. Text Classifier

The Text Classifier routes the incoming email into exactly one of three categories:

### Admissions

Admission applications, application status, eligibility, admission documents, document verification, entrance requirements, counselling, and admission procedures.

### Fees

Tuition/academic fees, pending payments, payment status, fee receipts, refunds, transactions, and fee-related issues.

### Hostel

Hostel applications, hostel allocation, room information, hostel fees, hostel rules, facilities, room changes, and hostel issues.

The classifier routes by **main intent**, not just by keyword.

Examples:

```text
"What documents are required for B.Tech admission?"
→ Admissions

"How much fee is still pending on my account?"
→ Fees

"Have I been allocated a hostel room?"
→ Hostel
```

The specialist agents do not classify the email again.

---

# Specialist AI Agents

## Admissions Agent

The Admissions Agent handles admission-related emails.

### Main data source

`applicants` table

### Database tool

`admission_lookup_database`

### RAG source

University XYZ Admissions Knowledge Base

### Personal information it can retrieve

- application status
- admission status
- document status
- missing documents
- entrance-test status
- counselling status
- program
- application progress

### Identity verification

The agent prefers identifiers such as:

```text
sender_email + application_number
```

or:

```text
sender_email + registration_number
```

It must not combine records from different applicants.

If the identifiers do not match, the agent should treat the case as an identity mismatch and avoid revealing personal information.

---

## Fees Agent

The Fees Agent handles fee and payment-related emails.

### Main data source

`fees` table

### Database tool

`fees_lookup_database`

### RAG source

University XYZ Fees and Finance Knowledge Base

### Personal information it can retrieve

- total fee
- amount paid
- amount pending
- payment status
- receipt status
- refund status
- transaction information

### Identity verification

For enrolled students, the preferred identifiers are:

```text
sender_email + student_id
```

or:

```text
sender_email + roll_number
```

For an applicant fee record, `sender_email + application_number` can be used when that record exists.

---

## Hostel Agent

The Hostel Agent handles hostel-related emails.

### Main data source

`hostel_records` table

### Database tool

`hostel_lookup_database`

### RAG source

University XYZ Hostel Knowledge Base

### Personal information it can retrieve

- hostel status
- hostel name
- room number
- room type
- hostel fee
- allocation date

### Identity verification

The preferred identifiers are:

```text
sender_email + student_id
```

or:

```text
sender_email + roll_number
```

The agent must not invent a hostel name or room number when the database contains no such value.

---

# Gmail Reply Automation

Generating an email body is not considered a completed task.

The final action is explicitly:

```text
WRITE EMAIL
      ↓
CALL GMAIL REPLY TOOL
      ↓
EMAIL SENT
```

The Gmail Reply Tool receives:

```text
message_id = original_message_id
message = final email body
```

The reply is sent to the original Gmail conversation rather than creating an unrelated new email.

This ensures that the AI Agent is responsible for the decision and content, while Gmail performs the actual sending action.

---

# Email Style

The agents are instructed to produce professional plain-text emails that are:

- concise
- polite
- direct
- easy to scan
- based only on verified information

The project intentionally avoids unnecessary Markdown, long explanations, repeated information, and unsupported claims.

Typical structure:

```text
Dear [Name],

Thank you for contacting the [Department].

[Direct answer]

[Relevant status or next step]

Please reply to this email if you need further assistance.

Regards,
[Department]
University XYZ
```

---

# Database Design

The project uses **one Supabase project** containing both the structured PostgreSQL data and the RAG/vector data.

The structured database contains these tables:

```text
applicants
students
fees
exam_records
hostel_records
scholarships
placements
technical_tickets
library_records
```

### `applicants`

Stores applicants who are applying / not yet enrolled.

Important fields include:

- `application_number`
- `registration_number`
- `email`
- `applicant_name`
- `program`
- `application_status`
- `admission_status`
- `document_status`
- `missing_documents`
- `entrance_status`
- `counselling_status`

### `students`

Stores enrolled students.

Important fields include:

- `student_id`
- `roll_number`
- `email`
- `student_name`
- `program`
- `semester`
- `department`
- `academic_status`

### `fees`

Stores student/applicant-specific financial information such as:

- `student_id`
- `roll_number`
- `email`
- `application_number`
- `total_fee`
- `amount_paid`
- `amount_pending`
- `payment_status`
- `receipt_status`
- `refund_status`
- `transaction_id`

### `exam_records`

Stores examination information such as:

- semester
- exam name
- registration status
- admit-card status
- result
- marks

### `hostel_records`

Stores hostel information such as:

- hostel status
- hostel name
- room number
- room type
- hostel fee
- allocation date

### `scholarships`

Stores scholarship application, approval, and disbursement information.

### `placements`

Stores placement-drive registration, eligibility, interview, and selection information.

### `technical_tickets`

Stores support tickets, issue descriptions, priority, status, and resolution information.

### `library_records`

Stores book issue/return information, due dates, fines, and record status.

---

# Database Relationships

The tables are logically connected mainly through shared identifiers such as `student_id`, `roll_number`, `email`, and application identifiers.

The student-centric relationship is conceptually:

```text
students
   |
   +---- fees
   +---- exam_records
   +---- hostel_records
   +---- scholarships
   +---- placements
   +---- technical_tickets
   +---- library_records
```

The current n8n lookup workflows can operate using identifier-based filtering even without requiring every logical relationship to be represented as a foreign-key constraint.

---

# RAG / Knowledge Base

The RAG system uses a single vector table named:

```text
Documents
```

The table stores:

- document chunks
- metadata
- embedding vectors

The project uses Google Gemini embeddings with a **3072-dimensional vector configuration**, matching the observed output of the Gemini Embeddings node in the working setup.

Conceptually:

```text
PDF
 ↓
Document Loader
 ↓
Text Splitter
 ↓
Google Gemini Embeddings
 ↓
Supabase Vector Store
 ↓
documents
```

The vector search function uses cosine-distance similarity to retrieve the most relevant chunks.

---

# Knowledge Base Documents

Three synthetic university PDFs were created for the RAG system:

```text
University_XYZ_Admissions_Knowledge_Base.pdf
University_XYZ_Fees_and_Finance_Knowledge_Base.pdf
University_XYZ_Hostel_Knowledge_Base.pdf
```

### Admissions Knowledge Base

Contains information about:

- admission lifecycle
- program eligibility
- required documents
- document quality requirements
- application procedure
- application and registration numbers
- entrance assessment
- counselling
- status definitions
- deadlines and communication
- document verification
- general admissions FAQ

### Fees and Finance Knowledge Base

Contains general fee/payment policies and procedures for the Fees Agent.

### Hostel Knowledge Base

Contains general hostel policies and procedures for the Hostel Agent.

The RAG documents provide **general university information only**. They are not used as a replacement for applicant/student-specific records.

---

# Example: Applicant-Specific Admissions Query

Suppose an applicant writes:

```text
Dear Admissions Office,

I would like to know the current status of my application.

My application number is APP20260001 and my registration number is REG20260001.

Regards,
Rahul Sharma
```

The workflow behaves like this:

```text
Gmail
 ↓
Edit Fields
 ↓
Text Classifier → Admissions
 ↓
Admissions Agent
 ↓
admission_lookup_database
 ↓
Verified applicant record
 ↓
Write concise response
 ↓
Gmail Reply Tool
 ↓
Email sent
```

The sample database record for `APP20260001` contains:

```text
Application Status: Submitted
Admission Status: Under Review
Document Status: Pending
Missing Document: Class 12 Marksheet
Entrance Status: Qualified
Counselling Status: Pending
```

The Agent uses these database values as the source of truth for the personal response.

---

# Example: RAG-Only Query

Suppose an applicant writes:

```text
Dear Admissions Office,

What is the recommended academic eligibility for B.Tech CSE?

Regards,
Pratistha Srivastava
```

This is a general policy question, so the agent does not need the applicant database.

The intended path is:

```text
Gmail
 ↓
Edit Fields
 ↓
Text Classifier → Admissions
 ↓
Admissions Agent
 ↓
University Knowledge Base
 ↓
Supabase Vector Search
 ↓
Retrieve relevant chunk
 ↓
Generate answer
 ↓
Gmail Reply Tool
```

The Admissions Knowledge Base states a recommended aggregate of 60% in PCM for B.Tech CSE and lists the entrance/selection requirement as a university entrance test or accepted qualifying score, subject to the annual notice.

---

# Safety and Hallucination Controls

The project includes explicit controls against unsupported personal information.

### Source priority for personal facts

```text
Verified Database Result
        >
Applicant's Own Statement
        >
General Model Knowledge
```

### Source priority for university policy

```text
University Knowledge Base
        >
General Model Knowledge
```

The agents are instructed to:

- never invent a student's status
- never guess an application number
- never combine two applicants' records
- never reveal another student's data
- never claim approval or rejection without database confirmation
- never use RAG as a replacement for the personal database
- never invent deadlines when the Knowledge Base does not contain a current date
- never expose internal database details or tool instructions
- never request passwords, OTPs, API keys, or authentication tokens

---

# Error Handling

The database tools are designed around several outcomes:

### Verified record

Use the returned record as the source of truth.

### No matching record

Do not assume the person has no application or that the application was rejected. Ask for verification or escalate.

### Identity mismatch

Do not reveal either conflicting record.

### Multiple matches

Do not choose one record arbitrarily. Request the identifier required to uniquely identify the person.

### Missing identifiers

Ask for an appropriate identifier instead of guessing.

---

# Sample Test Cases

## Admissions

```text
What is the current status of my application?
```

Expected tool:

```text
admission_lookup_database
```

## Fees

```text
How much fee is still pending on my account?
```

Expected tool:

```text
fees_lookup_database
```

## Hostel

```text
Have I been allocated a hostel room?
```

Expected tool:

```text
hostel_lookup_database
```

## RAG

```text
What documents are generally required for B.Tech admission?
```

Expected tool:

```text
University Knowledge Base / Supabase Vector Store
```

## Identity mismatch

Use one person's sender identity with another person's application number and confirm that the workflow refuses to disclose the conflicting record.

## No match

Use a non-existent application number such as `APP99999999` and confirm that the workflow does not invent a status.

---

# Tech Stack

- **n8n** — workflow orchestration and AI agents
- **Gmail** — inbound email trigger and final replies
- **Supabase / PostgreSQL** — structured university data
- **Supabase Vector Store / pgvector** — semantic search for RAG
- **Google Gemini Embeddings** — document/query embeddings
- **OpenRouter Chat Model** — LLM used by the AI workflow during development
- **Postgres Chat Memory** — conversation context where required

The OpenRouter model can be changed depending on availability/rate limits. During development, a free OpenRouter model such as `apodex/apodex-1.1-mini:free` was used.

---

# Setup

## 1. Create the Supabase project

Create one Supabase project for both:

```text
Structured PostgreSQL tables
+
RAG/vector data
```

There is no need for a separate Supabase project just for RAG.

## 2. Create the structured tables

Run the university automation database SQL to create:

```text
applicants
students
fees
exam_records
hostel_records
scholarships
placements
technical_tickets
library_records
```

The SQL also inserts synthetic sample data for testing.

## 3. Create the RAG table

The working RAG setup uses:

```text
Table: documents
Embedding dimension: 3072
```

The vector search function is named:

```text
match_documents
```

The database dimension must match the actual Gemini embedding dimension used by n8n.

## 4. Configure n8n credentials

Configure credentials for:

- Gmail
- Supabase/Postgres
- Google Gemini embeddings
- OpenRouter

## 5. Build the main email workflow

```text
Gmail Trigger
 → Edit Fields
 → Text Classifier
 → Specialist Agent
```

## 6. Connect the agent tools

Admissions Agent:

```text
admission_lookup_database
University Knowledge Base
Gmail Reply Tool
```

Fees Agent:

```text
fees_lookup_database
University Knowledge Base
Gmail Reply Tool
```

Hostel Agent:

```text
hostel_lookup_database
University Knowledge Base
Gmail Reply Tool
```

## 7. Build the lookup subworkflows

Each database lookup can be implemented as an n8n subworkflow using:

```text
When Executed by Another Workflow
        ↓
Supabase / PostgreSQL lookup
```

The parent specialist agent calls the subworkflow through a **Call n8n Workflow Tool**.

## 8. Load the three PDFs into RAG

Ingest each PDF through the document-loading, text-splitting, Gemini-embedding, and Supabase Vector Store workflow.

Use the same `documents` table for all three knowledge-base PDFs.

---

# Important Testing Note

The sample database contains synthetic addresses such as:

```text
rahul.sharma@example.com
rahul.enrolled@example.com
```

When testing from a real Gmail account, the sender address must match the address used by the corresponding sample database record if the identity verification logic checks `sender_email`.

For testing, it is easiest to replace one sample record's email with your own test Gmail address temporarily.

---

# Project Limitations

This project is a **portfolio prototype**, not a production university system.

Current limitations include:

- synthetic/sample university data
- limited department routing to Admissions, Fees, and Hostel
- sample policies rather than real university policies
- free-model/API rate limits can affect repeated testing
- no production-scale load test has been performed
- database update/write operations are intentionally restricted in the current design
- Gmail API permissions and thread/message handling must be configured correctly before production use

The system is designed to demonstrate the architecture and reasoning pattern rather than claim production readiness.

---

# Why this project is useful

This project demonstrates several practical AI automation concepts in one system:

- event-driven workflow automation
- email ingestion
- intent classification
- specialist AI agents
- tool calling
- PostgreSQL/Supabase integration
- database-backed identity verification
- RAG and vector search
- hallucination control through source separation
- conversation memory
- automated Gmail replies
- error handling and security rules

The most important design decision is separating **general knowledge** from **personal data**.

---

# Interview Explanation

A concise way to explain the project is:

> "I built an n8n-based university email automation system. Gmail receives an email, Edit Fields prepares the input, and a Text Classifier routes it to an Admissions, Fees, or Hostel agent. Each agent can use a structured Supabase/PostgreSQL database for student-specific information and a Supabase vector store for general university policies. The agents verify identifiers before returning personal data, and the final response is sent back through the original Gmail thread using a Gmail Reply Tool. I separated RAG from operational data so the model doesn't use static policy documents as a source of truth for live student information."

---

# Suggested Repository Structure

A clean repository can be organized like this:

```text
university-email-ai-automation/
│
├── README.md
│
├── database/
│   └── university_automation.sql
│
├── knowledge-base/
│   ├── University_XYZ_Admissions_Knowledge_Base.pdf
│   ├── University_XYZ_Fees_and_Finance_Knowledge_Base.pdf
│   └── University_XYZ_Hostel_Knowledge_Base.pdf
│
├── workflows/
│   ├── main-email-router.json
│   ├── admissions-db-lookup.json
│   ├── fees-db-lookup.json
│   └── hostel-db-lookup.json
│
└── screenshots/
    ├── main-workflow.png
    ├── admissions-agent.png
    ├── fees-agent.png
    ├── hostel-agent.png
    └── supabase-schema.png
```

Do not commit API keys, OAuth tokens, Supabase service-role keys, OpenRouter keys, or other secrets to the repository.

---

# Final Architecture Summary

```text
                       UNIVERSITY EMAIL
                              |
                              v
                         Gmail Trigger
                              |
                              v
                          Edit Fields
                              |
                              v
                       Text Classifier
                    /          |          \
                   /           |           \
                  v            v            v
           Admissions       Fees          Hostel
              Agent         Agent          Agent
                |             |              |
          applicants        fees      hostel_records
                |             |              |
                +-------------+--------------+
                              |
                    University Knowledge Base
                              |
                       Supabase Vector Store
                              |
                              v
                       Final Email Response
                              |
                              v
                       Gmail Reply Tool
                              |
                              v
                           EMAIL SENT
```

---

## Disclaimer

University XYZ and all related applicant/student records, policies, fees, hostel information, and knowledge-base content in this repository are synthetic examples created for demonstrating AI automation and RAG workflows. They should not be presented as real institutional policies or real student records.
# University XYZ — AI University Email Automation

An n8n-based AI email automation system that reads university emails, classifies them into departments, retrieves the right information from structured data and a RAG knowledge base, generates a concise professional reply, and sends the reply back to the original Gmail conversation.

> **Project status:** Working prototype / portfolio project  
> **University:** University XYZ (synthetic/sample data)  
> **Important:** University XYZ, its policies, applicants, students, fees, hostel records, and sample documents are fictional and were created for AI automation testing. They are not official university records.

---

## Overview

The system is designed around a simple principle:

**General university information comes from RAG. Personal/current information comes from the structured database.**

The AI agent does not answer applicant-specific questions from general knowledge. It first retrieves the relevant record from the database, verifies the available identifiers, and then uses the result as the source of truth.

For general policy questions, the agent searches the University XYZ knowledge base stored in Supabase as vector embeddings.

After the answer is prepared, the agent uses the Gmail Reply Tool to reply to the original email thread.

---

## Main Workflow

```text
Gmail Trigger
      |
      v
Edit Fields
      |
      v
Text Classifier
      |
      +-------------------+-------------------+
      |                   |                   |
      v                   v                   v
Admissions Agent      Fees Agent         Hostel Agent
      |                   |                   |
      |                   |                   |
      +---------+---------+---------+---------+
                |                   |
                v                   v
        Structured Database      RAG Knowledge Base
                |                   |
                +---------+---------+
                          |
                          v
                   Final Email Reply
                          |
                          v
                  Gmail Reply Tool
                          |
                          v
                     Email Sent
```

---

## Why the project uses both a database and RAG

These two sources have different responsibilities.

### Structured PostgreSQL / Supabase Database

Used for live or applicant/student-specific information such as:

- application status
- admission status
- missing documents
- fee balance
- payment status
- hostel allocation
- room details
- exam records
- scholarships
- placements
- technical tickets
- library records

### RAG / Supabase Vector Store

Used for university-wide information such as:

- eligibility criteria
- admission procedures
- required documents
- document-quality requirements
- fee policies and procedures
- hostel rules and procedures
- counselling rules
- verification procedures
- general university policies

This separation prevents the model from using a general policy document to answer a personal question such as "What is my current fee balance?".

---

# Workflow Details

## 1. Gmail Trigger

The workflow starts when a new email is received.

The Gmail trigger provides the information needed by downstream nodes, including the sender, email body, and Gmail message/thread context.

The original Gmail message ID is retained so the final response can be sent as a reply to the original conversation.

---

## 2. Edit Fields

The project uses **Edit Fields**, not an Information Extractor.

The Edit Fields node prepares and maps the incoming Gmail data for the rest of the workflow.

Typical values passed downstream include:

- sender email
- email body
- subject
- application number, when present
- registration number, when present
- student ID, when present
- roll number, when present
- original Gmail message ID
- original Gmail thread ID
- session information used by memory, when required

The purpose of this node is to create a clean, predictable input for the classifier and specialist agents.

---

## 3. Text Classifier

The Text Classifier routes the incoming email into exactly one of three categories:

### Admissions

Admission applications, application status, eligibility, admission documents, document verification, entrance requirements, counselling, and admission procedures.

### Fees

Tuition/academic fees, pending payments, payment status, fee receipts, refunds, transactions, and fee-related issues.

### Hostel

Hostel applications, hostel allocation, room information, hostel fees, hostel rules, facilities, room changes, and hostel issues.

The classifier routes by **main intent**, not just by keyword.

Examples:

```text
"What documents are required for B.Tech admission?"
→ Admissions

"How much fee is still pending on my account?"
→ Fees

"Have I been allocated a hostel room?"
→ Hostel
```

The specialist agents do not classify the email again.

---

# Specialist AI Agents

## Admissions Agent

The Admissions Agent handles admission-related emails.

### Main data source

`applicants` table

### Database tool

`admission_lookup_database`

### RAG source

University XYZ Admissions Knowledge Base

### Personal information it can retrieve

- application status
- admission status
- document status
- missing documents
- entrance-test status
- counselling status
- program
- application progress

### Identity verification

The agent prefers identifiers such as:

```text
sender_email + application_number
```

or:

```text
sender_email + registration_number
```

It must not combine records from different applicants.

If the identifiers do not match, the agent should treat the case as an identity mismatch and avoid revealing personal information.

---

## Fees Agent

The Fees Agent handles fee and payment-related emails.

### Main data source

`fees` table

### Database tool

`fees_lookup_database`

### RAG source

University XYZ Fees and Finance Knowledge Base

### Personal information it can retrieve

- total fee
- amount paid
- amount pending
- payment status
- receipt status
- refund status
- transaction information

### Identity verification

For enrolled students, the preferred identifiers are:

```text
sender_email + student_id
```

or:

```text
sender_email + roll_number
```

For an applicant fee record, `sender_email + application_number` can be used when that record exists.

---

## Hostel Agent

The Hostel Agent handles hostel-related emails.

### Main data source

`hostel_records` table

### Database tool

`hostel_lookup_database`

### RAG source

University XYZ Hostel Knowledge Base

### Personal information it can retrieve

- hostel status
- hostel name
- room number
- room type
- hostel fee
- allocation date

### Identity verification

The preferred identifiers are:

```text
sender_email + student_id
```

or:

```text
sender_email + roll_number
```

The agent must not invent a hostel name or room number when the database contains no such value.

---

# Gmail Reply Automation

Generating an email body is not considered a completed task.

The final action is explicitly:

```text
WRITE EMAIL
      ↓
CALL GMAIL REPLY TOOL
      ↓
EMAIL SENT
```

The Gmail Reply Tool receives:

```text
message_id = original_message_id
message = final email body
```

The reply is sent to the original Gmail conversation rather than creating an unrelated new email.

This ensures that the AI Agent is responsible for the decision and content, while Gmail performs the actual sending action.

---

# Email Style

The agents are instructed to produce professional plain-text emails that are:

- concise
- polite
- direct
- easy to scan
- based only on verified information

The project intentionally avoids unnecessary Markdown, long explanations, repeated information, and unsupported claims.

Typical structure:

```text
Dear [Name],

Thank you for contacting the [Department].

[Direct answer]

[Relevant status or next step]

Please reply to this email if you need further assistance.

Regards,
[Department]
University XYZ
```

---

# Database Design

The project uses **one Supabase project** containing both the structured PostgreSQL data and the RAG/vector data.

The structured database contains these tables:

```text
applicants
students
fees
exam_records
hostel_records
scholarships
placements
technical_tickets
library_records
```

### `applicants`

Stores applicants who are applying / not yet enrolled.

Important fields include:

- `application_number`
- `registration_number`
- `email`
- `applicant_name`
- `program`
- `application_status`
- `admission_status`
- `document_status`
- `missing_documents`
- `entrance_status`
- `counselling_status`

### `students`

Stores enrolled students.

Important fields include:

- `student_id`
- `roll_number`
- `email`
- `student_name`
- `program`
- `semester`
- `department`
- `academic_status`

### `fees`

Stores student/applicant-specific financial information such as:

- `student_id`
- `roll_number`
- `email`
- `application_number`
- `total_fee`
- `amount_paid`
- `amount_pending`
- `payment_status`
- `receipt_status`
- `refund_status`
- `transaction_id`

### `exam_records`

Stores examination information such as:

- semester
- exam name
- registration status
- admit-card status
- result
- marks

### `hostel_records`

Stores hostel information such as:

- hostel status
- hostel name
- room number
- room type
- hostel fee
- allocation date

### `scholarships`

Stores scholarship application, approval, and disbursement information.

### `placements`

Stores placement-drive registration, eligibility, interview, and selection information.

### `technical_tickets`

Stores support tickets, issue descriptions, priority, status, and resolution information.

### `library_records`

Stores book issue/return information, due dates, fines, and record status.

---

# Database Relationships

The tables are logically connected mainly through shared identifiers such as `student_id`, `roll_number`, `email`, and application identifiers.

The student-centric relationship is conceptually:

```text
students
   |
   +---- fees
   +---- exam_records
   +---- hostel_records
   +---- scholarships
   +---- placements
   +---- technical_tickets
   +---- library_records
```

The current n8n lookup workflows can operate using identifier-based filtering even without requiring every logical relationship to be represented as a foreign-key constraint.

---

# RAG / Knowledge Base

The RAG system uses a single vector table named:

```text
Documents
```

The table stores:

- document chunks
- metadata
- embedding vectors

The project uses Google Gemini embeddings with a **3072-dimensional vector configuration**, matching the observed output of the Gemini Embeddings node in the working setup.

Conceptually:

```text
PDF
 ↓
Document Loader
 ↓
Text Splitter
 ↓
Google Gemini Embeddings
 ↓
Supabase Vector Store
 ↓
documents
```

The vector search function uses cosine-distance similarity to retrieve the most relevant chunks.

---

# Knowledge Base Documents

Three synthetic university PDFs were created for the RAG system:

```text
University_XYZ_Admissions_Knowledge_Base.pdf
University_XYZ_Fees_and_Finance_Knowledge_Base.pdf
University_XYZ_Hostel_Knowledge_Base.pdf
```

### Admissions Knowledge Base

Contains information about:

- admission lifecycle
- program eligibility
- required documents
- document quality requirements
- application procedure
- application and registration numbers
- entrance assessment
- counselling
- status definitions
- deadlines and communication
- document verification
- general admissions FAQ

### Fees and Finance Knowledge Base

Contains general fee/payment policies and procedures for the Fees Agent.

### Hostel Knowledge Base

Contains general hostel policies and procedures for the Hostel Agent.

The RAG documents provide **general university information only**. They are not used as a replacement for applicant/student-specific records.

---

# Example: Applicant-Specific Admissions Query

Suppose an applicant writes:

```text
Dear Admissions Office,

I would like to know the current status of my application.

My application number is APP20260001 and my registration number is REG20260001.

Regards,
Rahul Sharma
```

The workflow behaves like this:

```text
Gmail
 ↓
Edit Fields
 ↓
Text Classifier → Admissions
 ↓
Admissions Agent
 ↓
admission_lookup_database
 ↓
Verified applicant record
 ↓
Write concise response
 ↓
Gmail Reply Tool
 ↓
Email sent
```

The sample database record for `APP20260001` contains:

```text
Application Status: Submitted
Admission Status: Under Review
Document Status: Pending
Missing Document: Class 12 Marksheet
Entrance Status: Qualified
Counselling Status: Pending
```

The Agent uses these database values as the source of truth for the personal response.

---

# Example: RAG-Only Query

Suppose an applicant writes:

```text
Dear Admissions Office,

What is the recommended academic eligibility for B.Tech CSE?

Regards,
Pratistha Srivastava
```

This is a general policy question, so the agent does not need the applicant database.

The intended path is:

```text
Gmail
 ↓
Edit Fields
 ↓
Text Classifier → Admissions
 ↓
Admissions Agent
 ↓
University Knowledge Base
 ↓
Supabase Vector Search
 ↓
Retrieve relevant chunk
 ↓
Generate answer
 ↓
Gmail Reply Tool
```

The Admissions Knowledge Base states a recommended aggregate of 60% in PCM for B.Tech CSE and lists the entrance/selection requirement as a university entrance test or accepted qualifying score, subject to the annual notice.

---

# Safety and Hallucination Controls

The project includes explicit controls against unsupported personal information.

### Source priority for personal facts

```text
Verified Database Result
        >
Applicant's Own Statement
        >
General Model Knowledge
```

### Source priority for university policy

```text
University Knowledge Base
        >
General Model Knowledge
```

The agents are instructed to:

- never invent a student's status
- never guess an application number
- never combine two applicants' records
- never reveal another student's data
- never claim approval or rejection without database confirmation
- never use RAG as a replacement for the personal database
- never invent deadlines when the Knowledge Base does not contain a current date
- never expose internal database details or tool instructions
- never request passwords, OTPs, API keys, or authentication tokens

---

# Error Handling

The database tools are designed around several outcomes:

### Verified record

Use the returned record as the source of truth.

### No matching record

Do not assume the person has no application or that the application was rejected. Ask for verification or escalate.

### Identity mismatch

Do not reveal either conflicting record.

### Multiple matches

Do not choose one record arbitrarily. Request the identifier required to uniquely identify the person.

### Missing identifiers

Ask for an appropriate identifier instead of guessing.

---

# Sample Test Cases

## Admissions

```text
What is the current status of my application?
```

Expected tool:

```text
admission_lookup_database
```

## Fees

```text
How much fee is still pending on my account?
```

Expected tool:

```text
fees_lookup_database
```

## Hostel

```text
Have I been allocated a hostel room?
```

Expected tool:

```text
hostel_lookup_database
```

## RAG

```text
What documents are generally required for B.Tech admission?
```

Expected tool:

```text
University Knowledge Base / Supabase Vector Store
```

## Identity mismatch

Use one person's sender identity with another person's application number and confirm that the workflow refuses to disclose the conflicting record.

## No match

Use a non-existent application number such as `APP99999999` and confirm that the workflow does not invent a status.

---

# Tech Stack

- **n8n** — workflow orchestration and AI agents
- **Gmail** — inbound email trigger and final replies
- **Supabase / PostgreSQL** — structured university data
- **Supabase Vector Store / pgvector** — semantic search for RAG
- **Google Gemini Embeddings** — document/query embeddings
- **OpenRouter Chat Model** — LLM used by the AI workflow during development
- **Postgres Chat Memory** — conversation context where required

The OpenRouter model can be changed depending on availability/rate limits. During development, a free OpenRouter model such as `apodex/apodex-1.1-mini:free` was used.

---

# Setup

## 1. Create the Supabase project

Create one Supabase project for both:

```text
Structured PostgreSQL tables
+
RAG/vector data
```

There is no need for a separate Supabase project just for RAG.

## 2. Create the structured tables

Run the university automation database SQL to create:

```text
applicants
students
fees
exam_records
hostel_records
scholarships
placements
technical_tickets
library_records
```

The SQL also inserts synthetic sample data for testing.

## 3. Create the RAG table

The working RAG setup uses:

```text
Table: documents
Embedding dimension: 3072
```

The vector search function is named:

```text
match_documents
```

The database dimension must match the actual Gemini embedding dimension used by n8n.

## 4. Configure n8n credentials

Configure credentials for:

- Gmail
- Supabase/Postgres
- Google Gemini embeddings
- OpenRouter

## 5. Build the main email workflow

```text
Gmail Trigger
 → Edit Fields
 → Text Classifier
 → Specialist Agent
```

## 6. Connect the agent tools

Admissions Agent:

```text
admission_lookup_database
University Knowledge Base
Gmail Reply Tool
```

Fees Agent:

```text
fees_lookup_database
University Knowledge Base
Gmail Reply Tool
```

Hostel Agent:

```text
hostel_lookup_database
University Knowledge Base
Gmail Reply Tool
```

## 7. Build the lookup subworkflows

Each database lookup can be implemented as an n8n subworkflow using:

```text
When Executed by Another Workflow
        ↓
Supabase / PostgreSQL lookup
```

The parent specialist agent calls the subworkflow through a **Call n8n Workflow Tool**.

## 8. Load the three PDFs into RAG

Ingest each PDF through the document-loading, text-splitting, Gemini-embedding, and Supabase Vector Store workflow.

Use the same `documents` table for all three knowledge-base PDFs.

---

# Important Testing Note

The sample database contains synthetic addresses such as:

```text
rahul.sharma@example.com
rahul.enrolled@example.com
```

When testing from a real Gmail account, the sender address must match the address used by the corresponding sample database record if the identity verification logic checks `sender_email`.

For testing, it is easiest to replace one sample record's email with your own test Gmail address temporarily.

---

# Project Limitations

This project is a **portfolio prototype**, not a production university system.

Current limitations include:

- synthetic/sample university data
- limited department routing to Admissions, Fees, and Hostel
- sample policies rather than real university policies
- free-model/API rate limits can affect repeated testing
- no production-scale load test has been performed
- database update/write operations are intentionally restricted in the current design
- Gmail API permissions and thread/message handling must be configured correctly before production use

The system is designed to demonstrate the architecture and reasoning pattern rather than claim production readiness.

---

# Why this project is useful

This project demonstrates several practical AI automation concepts in one system:

- event-driven workflow automation
- email ingestion
- intent classification
- specialist AI agents
- tool calling
- PostgreSQL/Supabase integration
- database-backed identity verification
- RAG and vector search
- hallucination control through source separation
- conversation memory
- automated Gmail replies
- error handling and security rules

The most important design decision is separating **general knowledge** from **personal data**.

---

# Interview Explanation

A concise way to explain the project is:

> "I built an n8n-based university email automation system. Gmail receives an email, Edit Fields prepares the input, and a Text Classifier routes it to an Admissions, Fees, or Hostel agent. Each agent can use a structured Supabase/PostgreSQL database for student-specific information and a Supabase vector store for general university policies. The agents verify identifiers before returning personal data, and the final response is sent back through the original Gmail thread using a Gmail Reply Tool. I separated RAG from operational data so the model doesn't use static policy documents as a source of truth for live student information."

---

# Suggested Repository Structure

A clean repository can be organized like this:

```text
university-email-ai-automation/
│
├── README.md
│
├── database/
│   └── university_automation.sql
│
├── knowledge-base/
│   ├── University_XYZ_Admissions_Knowledge_Base.pdf
│   ├── University_XYZ_Fees_and_Finance_Knowledge_Base.pdf
│   └── University_XYZ_Hostel_Knowledge_Base.pdf
│
├── workflows/
│   ├── main-email-router.json
│   ├── admissions-db-lookup.json
│   ├── fees-db-lookup.json
│   └── hostel-db-lookup.json
│
└── screenshots/
    ├── main-workflow.png
    ├── admissions-agent.png
    ├── fees-agent.png
    ├── hostel-agent.png
    └── supabase-schema.png
```

Do not commit API keys, OAuth tokens, Supabase service-role keys, OpenRouter keys, or other secrets to the repository.

---

# Final Architecture Summary

```text
                       UNIVERSITY EMAIL
                              |
                              v
                         Gmail Trigger
                              |
                              v
                          Edit Fields
                              |
                              v
                       Text Classifier
                    /          |          \
                   /           |           \
                  v            v            v
           Admissions       Fees          Hostel
              Agent         Agent          Agent
                |             |              |
          applicants        fees      hostel_records
                |             |              |
                +-------------+--------------+
                              |
                    University Knowledge Base
                              |
                       Supabase Vector Store
                              |
                              v
                       Final Email Response
                              |
                              v
                       Gmail Reply Tool
                              |
                              v
                           EMAIL SENT
```

---

## Disclaimer

University XYZ and all related applicant/student records, policies, fees, hostel information, and knowledge-base content in this repository are synthetic examples created for demonstrating AI automation and RAG workflows. They should not be presented as real institutional policies or real student records.
