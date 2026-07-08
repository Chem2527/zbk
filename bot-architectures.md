# System Architectures Reference Guide

High-level infrastructure and request routing overview for the **Banking Chatbot**, **KYC OCR Agents**, and **Compliance Agent** systems.

---

## 1. Banking Chatbot Architecture
### AWS Region: `us-east-1` (N. Virginia) | RDS Region: `ap-south-1` (Mumbai)

<img width="1707" height="1634" alt="banking_chatbot_arch" src="https://github.com/user-attachments/assets/2307752c-b582-4ec0-b7fb-c614fd2e72f8" />


### How It Works

The banking chatbot is an enterprise-grade AI assistant that lets customers query their banking data in plain English. It uses **Anthropic Claude** (hosted on Azure AI Foundry) as the AI brain and a dedicated **MCP (Model Context Protocol) Server** as the secure, read-only database gateway.

#### Request → Response Flow

| Step | What Happens |
| :---: | :--- |
| **1** | User sends a chat message via the browser (REST or WebSocket). |
| **2** | **API Gateway** routes the request to the **Chatbot Service Lambda** (`us-east-1`). |
| **3** | The Lambda loads session history from **DynamoDB** and checks the FAQ cache for a pre-computed answer. |
| **4** | If a database query is needed, the Chatbot Service (MCP Client) sends the question to **Claude** to translate it into a safe `SELECT` SQL statement. |
| **5** | The SQL is forwarded via **HMAC-SHA256 signed JSON-RPC 2.0** to the **MCP DB Middleware Lambda** (MCP Server). |
| **6** | The MCP Server validates the SQL (SELECT-only allowlist, no injections), enforces **Row-Level Security** (`user_id` / `company_id` scoping), and executes it against **RDS Aurora PostgreSQL** using a read-only database user. |
| **7** | Raw rows are returned to Claude, which formats a friendly natural-language reply streamed back to the user. |

#### Key Security Controls

| Control | Mechanism |
| :--- | :--- |
| **Customer Auth** | JWT Bearer tokens (HS256) |
| **Service-to-Service Auth** | HMAC-SHA256 signatures with 60-second replay window |
| **Payload Encryption** | AES-256-GCM end-to-end encryption on all frontend API calls |
| **SQL Safety** | SELECT/WITH only — `DROP`, `DELETE`, `INSERT`, `UPDATE` and 15+ dangerous patterns are blocked |
| **Data Scoping** | Every query must include `user_id` (banking) or `company_id` (invoicing) |
| **DB Access** | Read-only PostgreSQL user — write operations fail at the engine level |

#### DynamoDB Tables

| Table | Purpose |
| :--- | :--- |
| `sessions-banking-chatbot` | Active chat sessions and conversation history (up to 20 turns) |
| `connections-banking-chatbot` | Active WebSocket connection IDs |
| `faq-cache-banking-chatbot` | Cached AI responses with TTL (reduces LLM API costs) |
| `memories-banking-chatbot` | Cross-session user preferences and facts |
| `otps-banking-chatbot` | Time-limited OTPs for sensitive operations (payments) |

#### MCP Tools Exposed by the DB Middleware

| Tool | Parameters | Scope | Database |
| :--- | :--- | :--- | :--- |
| `execute_query` | `sql`, `userId` | Must contain `user_id` | `lmb_management` (Banking) |
| `execute_invoicing_query` | `sql`, `companyId` | Must contain `company_id` | `dev_invoice` (Invoicing) |

---

### Infrastructure Specifics

| Component | AWS Region | Detail |
| :--- | :--- | :--- |
| **API Gateway (REST + WS)** | `us-east-1` | Entry point for HTTP REST and real-time WebSocket connections |
| **Chatbot Service Lambda** | `us-east-1` | Node.js 20.x — orchestrates sessions, Claude, and MCP client calls |
| **MCP DB Middleware Lambda** | `us-east-1` | TypeScript — HMAC auth, SQL validation, RLS, read-only RDS executor |
| **DynamoDB** | `us-east-1` | Session, cache, memory, OTP storage (PAY_PER_REQUEST billing) |
| **RDS Aurora PostgreSQL** | `ap-south-1` | `lmb_management` (Banking Ledger) + `dev_invoice` (Invoicing) |
| **AI Provider** | Azure AI Foundry | Anthropic Claude 3.5 Sonnet/Opus via Azure AI Inference REST API |

---

## 2. KYC OCR Agents Architecture
### AWS Region: `ap-south-1` (Mumbai)

<img width="2057" height="1454" alt="kyc_ocr_agents_arch" src="https://github.com/user-attachments/assets/31886bee-0948-4900-9f3a-f5eca94bdc40" />


### How It Works

A production-grade, cloud-native document intelligence platform for automated KYC onboarding. It ingests identity and corporate PDFs, runs AI-powered OCR extraction using Vision LLMs, performs cross-document validation, scores compliance risk, and surfaces results to human reviewers via a backoffice case portal.

#### Processing Pipeline

| Stage | Service | What Happens |
| :---: | :--- | :--- |
| **1 — Ingestion** | `kyc-preprocessor` API | Client uploads PDF. The API validates consent, file type (<50 MB), creates a job row in PostgreSQL, uploads the raw PDF to S3, and enqueues a message to `kyc-preprocess-q`. |
| **2 — Classification** | Preprocessor Worker | SQS triggers the Lambda. It downloads the PDF, runs blank/blur/skew detection, and calls **Claude** to classify the document type (Passport, Trade License, Emirates ID, MOA, etc.). Result written to PostgreSQL. |
| **3 — Extraction** | `kyc-agents` Worker | A second SQS message triggers the Agents Worker. It converts PDF pages to JPEG at 300 DPI, sends them to the Claude **Vision LLM** via a specialized per-document-type agent, and parses the response into a typed schema. |
| **4 — Risk Scoring** | Agents Worker | Validated fields are passed through the Risk Scorer. A numeric risk score is calculated from flag weights; a final decision of `APPROVE`, `MANUAL_REVIEW`, or `REJECT` is derived and written to PostgreSQL. |
| **5 — RM Review** | `kyc-backoffice` | Relationship Managers log into the backoffice portal to inspect flagged cases, view OCR confidence, apply corrections with a full audit trail, and issue a final compliance decision. |

#### AI Agents by Document Type

| Document Type | Key Extracted Fields |
| :--- | :--- |
| Trade License | Company name, license number, expiry, activities, issuing authority |
| MOA (Memorandum of Association) | Company name, authorized capital, subscribers, legal structure |
| Incorporation Certificate | Company name, CIN, date of incorporation, jurisdiction |
| Emirates ID | Full name, ID number, DOB, expiry, nationality |
| Address Proof | Entity name, address, emirate, document date |
| Bank Statement | Account holder, account number (masked), bank name, statement period |
| Tax Registration | TRN number, entity name, registration date, tax authority |

#### Risk Decision Logic

| Decision | Condition |
| :--- | :--- |
| ✅ `APPROVE` | Risk score < 25 AND no hard-blocker flags |
| ⚠️ `MANUAL_REVIEW` | Risk score ≥ 25 AND no hard-blocker flags |
| ❌ `REJECT` | Any hard-blocker flag triggered OR risk score ≥ 80 |

Hard-blocker flags include: `TAMPER_SUSPECTED`, `ENTITY_INACTIVE`, `DOCUMENT_EXPIRED`, `NAME_MISMATCH`, `DATE_SEQUENCE_ANOMALY`.

#### SQS Queue Configuration

| Queue | Visibility Timeout | DLQ | Max Retries |
| :--- | :--- | :--- | :--- |
| `kyc-preprocess-q-{stage}` | 350s | `kyc-preprocess-dlq-{stage}` | 3 |
| `kyc-agents-q-{stage}` | 350s | `kyc-agents-dlq-{stage}` | 3 |

---

### Infrastructure Specifics

| Component | Detail |
| :--- | :--- |
| **AWS Region** | `ap-south-1` (Mumbai) — all compute, storage, queues, and database |
| **Compute** | AWS Lambda (Docker containers from ECR) — Python 3.12 runtime |
| **Document Storage** | AWS S3 (`kyc-documents-{stage}-{account-id}`) — private bucket, IAM-controlled access |
| **Task Queues** | AWS SQS Standard — batch size 1, `ReportBatchItemFailures` enabled |
| **Relational Database** | Aurora Serverless PostgreSQL (`kyc_db`) — Alembic-managed schema migrations |
| **Secrets** | AWS SSM Parameter Store (`/kyc/{stage}/...`) — no secrets in source code |
| **AI Provider** | Anthropic Claude 3.5 Sonnet via Azure AI Foundry (or Azure OpenAI GPT-4V) |
| **Backoffice** | FastAPI application — case management, RM overrides, analytics, audit trail |

---

## 3. Compliance Agent Architecture
### AWS Region: `me-central-1` (Middle East / Dubai)

<img width="1731" height="2056" alt="compliance_agent_arch" src="https://github.com/user-attachments/assets/7b65a615-f783-4742-bbfb-3659fd2cddd4" />


### How It Works

A serverless compliance screening engine that evaluates corporate entities for sanctions, UBO (Ultimate Beneficial Owner) checks, and regulatory risk. It uses **AWS S3 as a JSON document store** (no relational DB in production) and **Claude** for narrative generation.

#### Request → Evaluation Flow

| Step | What Happens |
| :---: | :--- |
| **1** | A client app (e.g., onboarding, KYC pipeline) calls the HTTP Gateway `/evaluate` endpoint. |
| **2** | The **Compliance API Lambda** writes the raw application JSON to S3 (`applications/`) and pushes a task to the **SQS Evaluation Queue**. |
| **3** | The **Compliance Worker Lambda** consumes the message, retrieves versioned compliance rules from S3 (`rules/`), and deterministically calculates risk scores. |
| **4** | The worker calls **Claude** to generate a plain-text narrative summary of findings. |
| **5** | Final findings, audit logs, and the compliance report are written to S3 (`findings/`, `reports/`, `audit_logs/`). |

#### S3 Folder Structure

| Folder | Contents |
| :--- | :--- |
| `applications/` | Raw incoming client metadata |
| `rules/` | Versioned JSON compliance rule definitions |
| `findings/` | Sanctions and UBO check results |
| `reports/` | Final compliance audit reports |
| `audit_logs/` | Maker/checker activity logs |

---

### Infrastructure Specifics

| Component | Detail |
| :--- | :--- |
| **AWS Region** | `me-central-1` (Dubai) — ensures Middle East data residency compliance |
| **Persistence** | AWS S3 (`compliance-agent-s3-staging-{account-id}`) — serverless JSON document store, no RDS |
| **Message Broker** | AWS SQS (`compliance-agent-sqs-staging`, VisibilityTimeout: 90s) |
| **AI Provider** | Azure OpenAI Foundry — Claude Opus/Sonnet for narrative and risk escalation |
