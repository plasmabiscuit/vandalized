# Awesome.ai — Tennessee Tech University Deployment Guide

> **Platform:** Awesome.ai (forked from the University of Idaho's Vandalizer project)
> **Target Environment:** Tennessee Tech University HPC
> **Date:** April 2026

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack](#3-technology-stack)
4. [How the System Works](#4-how-the-system-works)
5. [Prerequisites](#5-prerequisites)
6. [Infrastructure Services](#6-infrastructure-services)
7. [Environment Variables Reference](#7-environment-variables-reference)
8. [Deployment Steps](#8-deployment-steps)
9. [Post-Deployment Configuration](#9-post-deployment-configuration)
10. [Institutional Knowledge Bases](#10-institutional-knowledge-bases)
11. [Authentication & SSO](#11-authentication--sso)
12. [Branding Changes Required](#12-branding-changes-required)
13. [Backup & Recovery](#13-backup--recovery)
14. [Scaling & HPC Considerations](#14-scaling--hpc-considerations)
15. [Operational Runbook](#15-operational-runbook)
16. [Security Considerations](#16-security-considerations)
17. [Appendix A: Complete File Structure](#appendix-a-complete-file-structure)
18. [Appendix B: All Celery Task Queues](#appendix-b-all-celery-task-queues)
19. [Appendix C: Database Collections](#appendix-c-database-collections)

---

## 1. Platform Overview

Awesome.ai is an open-source, self-hosted AI-powered document intelligence platform built for research administration. It enables users to:

- **Upload documents** (PDF, DOCX, XLSX, PPTX, HTML) and extract structured data using LLM-powered workflows
- **Chat with documents** via retrieval-augmented generation (RAG) backed by ChromaDB vector search
- **Build extraction workflows** — multi-step pipelines that pull specific fields from research documents (proposals, awards, subawards, budgets)
- **Collaborate in teams** with role-based access control (owner/admin/member), organizational hierarchy, and multi-tenancy
- **Automate intake** from Microsoft 365 (Outlook shared mailboxes, OneDrive folders) with AI-powered triage
- **Verify and quality-gate** extracted data through human-in-the-loop verification workflows
- **Track certification progress** through the Awesome.ai Architect Workflow Certification program

The platform was originally developed at the University of Idaho under the NSF GRANTED program. This deployment adapts it for Tennessee Tech.

---

## 2. Architecture

```
                 ┌─────────────┐
   Browser ──────│  nginx/TLS  │
                 │  :443       │
                 └──────┬──────┘
                        │
              ┌─────────┴──────────┐
              │                    │
       /api/* │             /* (SPA)
              │                    │
       ┌──────▼──────┐    ┌───────▼────────┐
       │   FastAPI   │    │  React static  │
       │   :8001     │    │  (nginx files) │
       └──────┬──────┘    └────────────────┘
              │
     ┌────────┼──────────┬──────────┐
     │        │          │          │
  ┌──▼──┐ ┌──▼───┐ ┌────▼───┐ ┌───▼────┐
  │Mongo│ │Redis │ │ChromaDB│ │Celery  │
  │27017│ │:6379 │ │:8000   │ │workers │
  └─────┘ └──────┘ └────────┘ └────────┘
```

### Component Roles

| Component | Role |
|-----------|------|
| **FastAPI (API server)** | REST API, authentication, WebSocket/SSE streaming, request routing |
| **React SPA (frontend)** | Single-page application served as static files via nginx |
| **MongoDB** | Primary data store — users, teams, documents, workflows, audit logs, system config |
| **Redis** | Celery task broker/result backend, session token storage, rate limiting, OAuth state |
| **ChromaDB** | Vector database for RAG — stores document/knowledge base embeddings |
| **Celery workers** | Async background tasks — document processing, workflow execution, email, M365 sync |
| **Celery Beat** | Cron-like scheduler for periodic tasks (demo expiry, retention, engagement emails) |
| **nginx** | Reverse proxy, TLS termination, static file serving, SPA routing |

### Data Flow

1. **Document upload:** Browser → nginx → FastAPI → filesystem (or S3) → Celery task → text extraction → ChromaDB embedding
2. **Extraction workflow:** User configures steps → Celery executes → LLM API calls → structured JSON output → MongoDB storage
3. **RAG chat:** User sends message → FastAPI queries ChromaDB for relevant chunks → constructs prompt with context → streams LLM response
4. **M365 intake:** Graph API webhook → Celery task → download attachment → triage with LLM → trigger workflow

---

## 3. Technology Stack

### Backend

| Technology | Version | Purpose |
|-----------|---------|---------|
| Python | 3.11–3.12 | Runtime |
| FastAPI | >= 0.115 | Web framework (async) |
| Beanie | >= 1.27 | Async MongoDB ODM (Pydantic v2) |
| Motor | >= 3.6 | Async MongoDB driver |
| Celery | >= 5.4 | Distributed task queue |
| pydantic-ai-slim | >= 1.56 | LLM agent framework |
| ChromaDB | >= 0.5 | Vector database |
| PyJWT | >= 2.12 | JWT authentication |
| httpx | >= 0.27 | Async HTTP client |
| Uvicorn | >= 0.34 | ASGI server |
| slowapi | >= 0.1 | Rate limiting |
| aiosmtplib | >= 3.0 | Async SMTP |
| msal | >= 1.28 | Microsoft authentication |
| python3-saml | >= 1.16 | SAML 2.0 SSO |
| sentry-sdk | >= 2.0 | Error tracking (optional) |
| prometheus-fastapi-instrumentator | >= 7.0 | Metrics |

**Document processing:** PyPDF2, PyMuPDF, markitdown (DOCX/PPTX/XLSX), BeautifulSoup4, pypandoc, reportlab, fpdf2

**Package manager:** `uv` (not pip)

### Frontend

| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 19 | UI framework |
| Vite | 7.3+ | Build tool & dev server |
| TypeScript | 5.9+ | Type safety |
| Tailwind CSS | 4.2+ | Utility-first CSS |
| TanStack Router | 1.161+ | File-based routing with URL state |
| TanStack React Query | 5.90+ | Server state management |
| Recharts | 3.7+ | Charts and data visualization |
| Lucide React | 0.574+ | Icons |
| Vitest | 4.1+ | Unit testing |
| Playwright | 1.49+ | E2E testing |

### Infrastructure

| Service | Image/Version | Purpose |
|---------|--------------|---------|
| MongoDB | mongo:7.0.16 | Document database |
| Redis | redis-stack:7.4.0-v3 | Broker, cache, session store |
| ChromaDB | chromadb/chroma:0.6.3 | Vector embeddings |
| nginx | nginx-alpine | Reverse proxy & static files |

---

## 4. How the System Works

### 4.1 Document Processing Pipeline

When a user uploads a document:

1. **File storage:** Saved to `UPLOAD_DIR` (local filesystem or S3)
2. **Text extraction:** Celery task extracts raw text using format-specific readers:
   - PDF: PyPDF2 (digital) or OCR endpoint (scanned)
   - DOCX/PPTX/XLSX: markitdown library
   - HTML: BeautifulSoup4
3. **Validation:** Optional format and content checks
4. **Classification:** Auto-classifies sensitivity level (unrestricted, internal, FERPA, CUI, ITAR)
5. **Semantic ingestion:** Text is chunked (1000 chars, 200 overlap) and embedded into ChromaDB per-user collection
6. **Metadata stored:** SmartDocument record created in MongoDB with text, token count, page count

### 4.2 Extraction Engine

The extraction engine uses LLMs to pull structured data from documents:

- **Two-pass mode (default):**
  - Pass 1: "Thinking" pass — LLM analyzes document with extraction keys, returns raw reasoning
  - Pass 2: "Structured" pass — LLM converts reasoning into validated JSON matching the schema
- **One-pass mode:** Single LLM call with optional thinking and structured output
- **Multimodal support:** Can send PDF pages as images to vision-capable models
- **Chunking:** For documents with many extraction fields, splits into manageable chunks

### 4.3 Workflow Engine

Workflows chain multiple operations:

1. **Steps:** Ordered sequence of operations (extraction, transformation, output)
2. **Tasks:** Each step contains one or more tasks
3. **Attachments:** Documents bound to the workflow
4. **Execution:** Celery task runs steps sequentially, tracking progress via WorkflowResult
5. **Approval gates:** Workflows can pause at a step for human approval
6. **Batch execution:** Run a workflow across multiple documents
7. **Validation planning:** Auto-generated test plans to verify workflow accuracy

### 4.4 RAG Chat

The chat system provides document-aware Q&A:

1. Load conversation history from MongoDB
2. If documents are selected, include their text as context
3. If a knowledge base is active, query ChromaDB for top-8 semantically similar chunks
4. Construct system prompt + context + user message
5. Stream LLM response with thinking tag extraction
6. Store message in ChatConversation

### 4.5 Knowledge Bases

Knowledge bases are curated collections of documents and URLs for RAG:

- **Document sources:** Link existing SmartDocuments; text is chunked and embedded in a dedicated ChromaDB collection (`kb_{uuid}`)
- **URL sources:** Fetch web pages, extract text, chunk and embed; optional crawling with configurable depth
- **Quality gates:** Minimum sources, chunks, source health, and retrieval precision thresholds
- **Sharing:** Share with team or across organizations
- **Verification:** Verified KBs can be published to the library

### 4.6 Multi-Tenancy & Access Control

- **Users** belong to one or more **Teams** via **TeamMembership** (roles: owner, admin, member)
- **Organizations** provide hierarchy (university > college > department)
- Documents, workflows, folders, and knowledge bases are scoped by `team_id`
- Users have a `current_team` that controls their active workspace
- A default team can be configured for auto-enrollment of new users
- SAML SSO can auto-map users to organizations based on department attribute

### 4.7 Automation & M365 Integration

- **Folder watch:** Trigger workflows when documents are uploaded to a specific folder
- **M365 intake:** Monitor Outlook shared mailboxes or OneDrive folders via Microsoft Graph webhooks
- **Scheduled automations:** Cron-based execution via Celery Beat
- **API triggers:** External systems can trigger workflows via API key authentication
- **AI triage:** Incoming work items are classified by LLM for routing

### 4.8 Certification Program

The Awesome.ai Architect Workflow Certification is an interactive 8-module training program:

- Progressive modules covering platform usage, workflow design, extraction techniques
- Knowledge checks and self-assessments
- XP-based progression with level tracking
- Completion awards the Authorized Awesome.ai Architect badge

---

## 5. Prerequisites

### 5.1 Hardware Requirements

| Deployment Size | CPU | RAM | Storage | Users |
|----------------|-----|-----|---------|-------|
| Small team | 4 cores | 8 GB | 50 GB | < 50 |
| Department / college | 8 cores | 16 GB | 100 GB+ | 50–500 |

The server does **not** need a GPU — LLM inference happens externally via API calls. Storage needs scale with uploaded document volume.

**Docker resource limits from compose.yaml:**

| Service | Memory | CPU |
|---------|--------|-----|
| Redis | 512 MB | 1.0 |
| MongoDB | 1 GB | 2.0 |
| ChromaDB | 2 GB | 2.0 |
| API (FastAPI) | 4 GB | 4.0 |
| Celery workers | 8 GB | 4.0 |

### 5.2 Software Requirements

- **Docker** and **Docker Compose** (v2+)
- **Git** for source management
- **Python 3.11–3.12** (if running outside Docker)
- **Node.js 18+** (if building frontend outside Docker)
- **uv** package manager (for Python, if running outside Docker)
- **pandoc** system package (for document conversion, included in Docker image)

### 5.3 Network Requirements

- Port 80/443 for web traffic
- Port 8001 for API (internal, behind reverse proxy)
- Port 27017/27018 for MongoDB (internal)
- Port 6379 for Redis (internal)
- Port 8000 for ChromaDB (internal)
- Outbound HTTPS to LLM provider APIs (OpenAI, Azure, etc.)
- Outbound HTTPS to Microsoft Graph API (if using M365 integration)
- Outbound SMTP to email relay (if sending emails)

### 5.4 LLM Provider Access

At least one LLM provider must be accessible. Supported protocols:

| Provider Type | Protocol | Example |
|--------------|----------|---------|
| OpenAI | `openai` | api.openai.com |
| Azure OpenAI | `openai` | your-resource.openai.azure.com |
| Ollama (local) | `ollama` | localhost:11434 |
| vLLM (local) | `vllm` | localhost:8000 |
| OpenRouter | `openai` | openrouter.ai/api/v1 |
| Any OpenAI-compatible | `openai` | Custom endpoint |

LLM models are configured in the Admin UI, **not** in environment variables. API keys are encrypted at rest in MongoDB using the `CONFIG_ENCRYPTION_KEY`.

### 5.5 OCR Service (Optional but Recommended)

For processing scanned PDFs, configure an OCR endpoint in the Admin UI. Any HTTP service that accepts a multipart PDF upload and returns plain text will work:

- Self-hosted: Marker, Surya, Tesseract wrapper
- Cloud: Azure Document Intelligence, AWS Textract, Google Document AI

Without OCR, the platform falls back to PyPDF2 text extraction (poor results on scanned documents).

### 5.6 SMTP Server (Optional but Recommended)

Required for:
- Password reset emails
- Team invitation emails
- Demo account lifecycle emails
- Onboarding drip campaigns
- Support ticket notifications
- Daily activity digest

TTU should provide an institutional SMTP relay or use a service like SendGrid, Mailgun, or Amazon SES.

---

## 6. Infrastructure Services

### 6.1 MongoDB

**Purpose:** Primary application database. Stores all users, teams, documents (metadata + extracted text), workflows, extraction results, chat conversations, audit logs, and system configuration.

**Version:** 7.0.16
**Default database name:** `vandalizer` (configurable via `MONGO_DB` — consider renaming to `awesomeai` for TTU)
**Port mapping:** 27018 (host) → 27017 (container)

**Key collections (43 Beanie document models):**

| Category | Collections |
|----------|------------|
| Users & Auth | User, Team, TeamMembership, TeamInvite, Organization |
| Documents | SmartDocument, SmartFolder, SearchSet, SearchSetItem |
| Workflows | Workflow, WorkflowStep, WorkflowStepTask, WorkflowResult, WorkflowArtifact, WorkflowAttachment |
| Chat | ChatConversation, ChatMessage, FileAttachment, UrlAttachment, ChatFeedback |
| Knowledge | KnowledgeBase, KnowledgeBaseSource, KnowledgeBaseReference, KBTestQuery, KBSuggestion |
| Automation | Automation, Verification, ExtractionTestCase, ValidationRun, QualityAlert |
| Activity | ActivityEvent, AuditLog, AdminAuditLog, Notification |
| Config | SystemConfig (singleton), UserModelConfig |
| M365 | IntakeConfig, WorkItem, GraphSubscription, TriggerEvent |
| Other | DemoApplication, CertificationProgress, ApprovalRequest, FeedbackPrompt, Library, LibraryItem, LibraryFolder |

**SystemConfig** is a singleton document that stores all runtime configuration:
- LLM endpoints and available models (with encrypted API keys)
- OCR endpoint
- Extraction strategy (one-pass/two-pass)
- Quality verification gates and thresholds
- Data classification levels and retention policies
- Authentication methods and OAuth/SAML provider configs
- M365 integration settings
- UI customization (highlight color, border radius)
- Support contacts and default team

Indexes are auto-created by Beanie on startup. No manual index creation is needed.

**For TTU HPC:** If MongoDB Atlas or an existing institutional MongoDB instance is available, set `MONGO_HOST` to point to it. Otherwise, run the containerized MongoDB with a persistent volume backed by reliable storage.

### 6.2 Redis

**Purpose:** Celery task broker and result backend; also used for password reset tokens (1-hour TTL), OAuth CSRF state (10-minute TTL), and rate limiting counters.

**Version:** redis-stack:7.4.0-v3
**Port:** 6379

Redis is **ephemeral** — no backup needed. Losing Redis drops in-flight Celery tasks (which can be resubmitted) but no application data is lost. All persistent state is in MongoDB.

**Celery broker:** `redis://redis:6379/0`
**Celery backend:** `redis://redis:6379/1`

### 6.3 ChromaDB

**Purpose:** Vector database for document embeddings and semantic search (RAG). Stores chunked and embedded document text for both per-user document collections and knowledge base collections.

**Version:** 0.6.3
**Port:** 8000
**Persistence:** `IS_PERSISTENT=TRUE`, data stored in `chroma-data` volume

**Collection naming:**
- Per-user documents: `user_{user_id}_docs`
- Per-knowledge-base: `kb_{kb_uuid}`

**Chunking defaults:** 1000 characters per chunk, 200 character overlap, paragraph-aware splitting.

ChromaDB can be **rebuilt** from source documents if lost (via re-ingestion), but this is time-consuming for large deployments.

---

## 7. Environment Variables Reference

### 7.1 Root `.env` (project root)

```env
# Application mode: development | staging | production
ENVIRONMENT=production

# Custom LLM endpoint (OpenAI-compatible). Leave empty to use provider endpoints from Admin UI.
INSIGHT_ENDPOINT=

# Redis host
redis_host=redis

# ChromaDB configuration
USE_CHROMA_SERVER=false
CHROMA_HOST=chromadb
CHROMA_PORT=8000
CHROMADB_PERSIST_DIR=../app/static/db

# Config Encryption — Fernet key for encrypting LLM API keys and OAuth secrets in MongoDB.
# Generate: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# REQUIRED for production. Without this, secrets are stored in plaintext.
CONFIG_ENCRYPTION_KEY=

# M365 Integration (Graph API) — only needed if using Microsoft 365 intake
GRAPH_TOKEN_KEY=                                            # Fernet key for Graph API tokens
GRAPH_NOTIFICATION_URL=https://awesomeai.tntech.edu/office/webhooks/graph
GRAPH_CLIENT_STATE_SECRET=<generate-random-secret>
VANDALIZER_BASE_URL=https://awesomeai.tntech.edu
```

### 7.2 Backend `.env` (`backend/.env`)

```env
# --- Database ---
MONGO_HOST=mongodb://mongo:27017/       # Docker service name, or external MongoDB URI
MONGO_DB=vandalizer                      # Database name (consider changing to 'awesomeai')

# --- Redis ---
REDIS_HOST=redis                         # Docker service name, or external Redis host

# --- Auth ---
# REQUIRED. Generate: python -c "import secrets; print(secrets.token_urlsafe(64))"
# Signs all JWT tokens. Must be at least 64 characters. Keep secret.
JWT_SECRET_KEY=<generate-strong-secret>

# --- File Storage ---
UPLOAD_DIR=/app/static/uploads           # Persistent volume mount point

# --- Frontend ---
FRONTEND_URL=https://awesomeai.tntech.edu   # Public URL for CORS and redirects

# --- Environment ---
ENVIRONMENT=production

# --- LLM ---
# Models are configured in Admin UI, not here. This is only for a default custom endpoint.
INSIGHT_ENDPOINT=

# --- ChromaDB ---
CHROMADB_PERSIST_DIR=/app/static/db

# --- Config Encryption ---
# Same Fernet key as root .env. REQUIRED for production.
CONFIG_ENCRYPTION_KEY=

# --- SMTP Email ---
SMTP_HOST=smtp.tntech.edu               # TTU institutional SMTP relay
SMTP_PORT=587
SMTP_USER=                               # If relay requires auth
SMTP_PASSWORD=
SMTP_USE_TLS=false
SMTP_START_TLS=true
SMTP_FROM_EMAIL=noreply@tntech.edu
SMTP_FROM_NAME=Awesome.ai               # Was "Vandalizer"

# --- Observability (optional) ---
SENTRY_DSN=                              # Sentry error tracking
LOG_FORMAT=json                          # json or text

# --- S3 Storage (optional, alternative to local filesystem) ---
# STORAGE_BACKEND=s3
# S3_BUCKET=
# S3_REGION=us-east-1
# S3_ACCESS_KEY_ID=
# S3_SECRET_ACCESS_KEY=
# S3_ENDPOINT_URL=                       # For MinIO or other S3-compatible storage
```

### 7.3 Frontend `.env` (`frontend/.env`)

```env
# Optional support link shown in the header
# VITE_SUPPORT_URL=https://support.tntech.edu/awesomeai
```

The frontend is largely configured at **runtime** via API calls to the backend (`/api/config/theme`, `/api/auth/config`), not through environment variables.

### 7.4 Critical Variables Summary

| Variable | Required | Where | Purpose |
|----------|----------|-------|---------|
| `JWT_SECRET_KEY` | **Yes** | backend/.env | Signs all auth tokens. Generate 64+ char secret. |
| `CONFIG_ENCRYPTION_KEY` | **Yes** (production) | both .env files | Encrypts LLM API keys in MongoDB. Fernet key. |
| `MONGO_HOST` | **Yes** | backend/.env | MongoDB connection URI |
| `MONGO_DB` | Yes | backend/.env | Database name (default: `vandalizer`) |
| `REDIS_HOST` | Yes | backend/.env | Redis hostname |
| `FRONTEND_URL` | Yes | backend/.env | Public URL for CORS and email links |
| `UPLOAD_DIR` | Yes | backend/.env | Document storage path |
| `CHROMADB_PERSIST_DIR` | Yes | backend/.env | ChromaDB data path |
| `ENVIRONMENT` | Yes | both | `production` for deployment |
| `SMTP_*` | Recommended | backend/.env | Email sending |
| `GRAPH_*` | Optional | root .env | M365 integration |
| `SENTRY_DSN` | Optional | backend/.env | Error tracking |
| `S3_*` | Optional | backend/.env | S3 file storage |

---

## 8. Deployment Steps

### 8.1 Clone and Configure

```bash
git clone <repo-url>
cd Awesome.ai

# Copy environment templates
cp .env.example .env
cp backend/.env.example backend/.env
```

### 8.2 Generate Secrets

```bash
# JWT secret (required)
python3 -c "import secrets; print(secrets.token_urlsafe(64))"

# Config encryption key (required for production)
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Graph client state secret (if using M365)
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Paste the generated values into the appropriate `.env` files.

### 8.3 Interactive Setup (Recommended)

```bash
./setup.sh
```

The setup wizard will:
1. Run pre-flight checks (Docker, Compose, etc.)
2. Walk through environment configuration
3. Generate any missing secrets
4. Build Docker images
5. Start all containers
6. Create the initial admin account
7. Seed the database with default data

### 8.4 Manual Docker Compose Setup

```bash
# Build and start all services
docker compose up --build -d

# Bootstrap admin account and default team
docker compose exec \
  -e ADMIN_EMAIL=admin@tntech.edu \
  -e ADMIN_PASSWORD='<strong-password>' \
  -e ADMIN_NAME='TTU Admin' \
  -e DEFAULT_TEAM_NAME='Research Administration' \
  api python bootstrap_install.py

# Verify health
curl http://localhost:8001/api/health
# Expected: {"status":"ok","checks":{"mongodb":"ok","redis":"ok","chromadb":"ok"}}
```

### 8.5 Development Setup (Without Docker)

```bash
# Start infrastructure only
docker compose up -d redis mongo chromadb

# Backend
cd backend
uv sync
uvicorn app.main:app --reload --port 8001

# Celery workers (separate terminal)
cd backend
./run_celery.sh start

# Frontend (separate terminal)
cd frontend
npm install
npm run dev
```

### 8.6 TLS/HTTPS Configuration

Place a reverse proxy in front of the application. Example nginx config:

```nginx
server {
    listen 80;
    server_name awesomeai.tntech.edu;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name awesomeai.tntech.edu;

    ssl_certificate     /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;

    client_max_body_size 200M;

    location /api/ {
        proxy_pass http://api:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;       # Required for SSE/streaming chat
        proxy_cache off;
        proxy_read_timeout 300s;
    }

    location /static/uploads/ {
        alias /app/static/uploads/;
    }

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 9. Post-Deployment Configuration

After the application is running, log in as the bootstrap admin and configure via the Admin UI.

### 9.1 LLM Models (Admin → System Config → Models)

Add at least one LLM model. Each entry requires:

| Field | Description |
|-------|-------------|
| Name | Display label (e.g., "GPT-4o", "Claude Sonnet") |
| API Key | Provider API key (encrypted at rest) |
| Endpoint URL | API base URL |
| Protocol | `openai`, `ollama`, or `vllm` |
| Multimodal | Whether the model supports vision/images |
| Supports PDF | Whether the model can process PDF input |
| Thinking | Whether the model supports extended thinking |

Models can be added, removed, or rotated without restarting the application.

### 9.2 Extraction Strategy (Admin → System Config → Extraction)

Configure how the extraction engine operates:

- **Mode:** `two_pass` (recommended) or `one_pass`
- **Model selection:** Which model to use for each pass
- **Chunking:** Enable for documents with many extraction fields
- **Use images:** Enable multimodal extraction for vision-capable models

### 9.3 OCR Endpoint (Admin → System Config → Endpoints)

Set the OCR Endpoint URL to an HTTP service that accepts multipart PDF uploads and returns text.

### 9.4 Quality Gates (Admin → System Config → Quality)

Configure verification thresholds:

- Minimum extraction accuracy: 0.7 (default)
- Minimum extraction consistency: 0.8
- Minimum workflow grade: "C"
- Knowledge base: minimum sources (3), chunks (50), source health (0.8), retrieval precision (0.6)

### 9.5 Data Classification & Retention (Admin → System Config)

Classification levels with default retention periods:

| Level | Retention | Soft-Delete Grace |
|-------|-----------|-------------------|
| Unrestricted | 365 days | 30 days |
| Internal | 730 days | 30 days |
| FERPA | 2,555 days (7 years) | 60 days |
| CUI | 1,825 days (5 years) | 60 days |
| ITAR | 1,825 days (5 years) | 90 days |

Adjust these to comply with TTU data retention policies.

### 9.6 Authentication Methods (Admin → System Config → Auth)

Enable one or more: `password`, `oauth` (Azure AD), `saml` (enterprise SSO).

### 9.7 UI Customization (Admin → System Config)

- **Highlight color:** Primary accent color (default: #eab308 yellow — consider TTU purple/gold)
- **UI radius:** Border radius for UI elements (default: 12px)

### 9.8 Verification Checklist

- [ ] Login works with admin credentials
- [ ] At least one LLM model is configured and responding
- [ ] OCR endpoint is configured (if processing scanned PDFs)
- [ ] File upload completes successfully
- [ ] Extraction workflow runs to completion (confirms Celery workers are connected)
- [ ] Chat with a document works (confirms RAG pipeline end-to-end)
- [ ] Email sending works (test password reset)
- [ ] Run `./status.sh` — all checks pass

---

## 10. Institutional Knowledge Bases

### 10.1 Seeded Knowledge Bases

The repository includes seed data for institutional knowledge bases in `backend/seeds/knowledge_bases/`. These are currently configured for University of Idaho and **must be replaced** with Tennessee Tech equivalents:

| Seed File | Original Content | TTU Replacement Needed |
|-----------|-----------------|----------------------|
| `ui_sponsored_programs.json` | University of Idaho Office of Sponsored Programs policies and procedures | TTU Office of Research policies, procedures, and guidelines |

### 10.2 Creating TTU Knowledge Bases

Knowledge bases should be created for TTU-specific institutional knowledge:

1. **TTU Office of Research** — Sponsored programs policies, proposal submission guidelines, award management procedures
2. **TTU Compliance** — IRB, IACUC, IBC protocols and requirements
3. **TTU Financial Services** — Budget templates, cost-sharing policies, indirect cost rates
4. **Federal Sponsor Guidelines** — NSF PAPPG, NIH Grants Policy Statement, DoD grant guidelines
5. **TTU Forms & Templates** — Standard institutional forms, routing sheets, approval workflows

### 10.3 Knowledge Base Architecture

Each knowledge base:
- Has a dedicated ChromaDB collection for vector search
- Can include documents (uploaded files) and URLs (web pages with optional crawling)
- Goes through quality gates before verification
- Can be shared with teams or across organizations
- Verified KBs can be published to the shared library

### 10.4 Seed Workflow Templates

The repository also includes seed workflows in `backend/seeds/workflows/`. These contain extraction templates for common research administration documents. Review and adapt for TTU-specific document formats.

---

## 11. Authentication & SSO

### 11.1 Password Authentication (Default)

Standard username/password with:
- JWT cookie-based sessions (access: 30 min, refresh: 60 days)
- HttpOnly, Secure, SameSite=Lax cookies in production
- Rate limiting: 5 login attempts/minute, 3 registration attempts/minute
- Password reset via email with 1-hour token

### 11.2 Azure AD OAuth (Recommended for TTU)

If TTU uses Azure AD / Entra ID:

1. Register an application in Azure AD
2. Set redirect URI: `https://awesomeai.tntech.edu/api/auth/oauth/azure/callback`
3. Configure in Admin → System Config → Auth → OAuth Providers:
   - `client_id`: Azure app client ID
   - `client_secret`: Azure app client secret
   - `tenant_id`: TTU tenant ID
   - `label`: "Sign in with TTU" (replaces "Sign in with U of I")

OAuth users are auto-created on first login with no password (OAuth-only accounts). They automatically get a personal team and join the default team.

### 11.3 SAML 2.0 SSO

For enterprise SSO via TTU's identity provider:

1. Configure in Admin → System Config → Auth → OAuth Providers (provider type: `saml`):
   - `idp_entity_id`: TTU IdP entity ID
   - `idp_sso_url`: TTU IdP SSO URL
   - `idp_x509_cert`: TTU IdP signing certificate
   - `sp_entity_id`: (optional) Service provider entity ID
   - `acs_url`: (optional) Assertion consumer service URL

2. Provide TTU IdP administrators with the SP metadata: `GET /api/auth/saml/metadata`

**Attribute mapping:**
- `uid` → User ID
- `email` → Email address
- `display_name` → Full name
- `department` → Auto-maps to Organization if matching org exists

### 11.4 API Key Authentication

For programmatic access (scripts, external systems):
- Users generate API tokens via Account settings
- Tokens expire after 365 days
- Sent via `X-API-Key` header
- Supports extraction and workflow execution endpoints

---

## 12. Branding Changes Required

The following plain-text branding references need to be updated for the TTU deployment. Variable names, code identifiers, and internal technical references (like `vandalizer_export` JSON flags) do **not** need to change unless desired.

### 12.1 Replacement Map

| Original | Replacement |
|----------|-------------|
| University of Idaho | Tennessee Tech |
| Vandalizer | Awesome.ai |
| Vandal Workflow Architect certification | Awesome.ai Architect Workflow Certification |
| Vandal Workflow Architect | Authorized Awesome.ai Architect |
| U of I | TTU |

### 12.2 Files Requiring Updates

#### Frontend UI Components

| File | Status | References |
|------|--------|-----------|
| `frontend/src/pages/Landing.tsx` | Done | "University of Idaho" (4), product description, alt text |
| `frontend/src/pages/Certification.tsx` | Done | "University of Idaho", "Vandal Workflow Architect" (multiple), "Vandalizer" |
| `frontend/src/pages/Docs.tsx` | Done | "University of Idaho" |
| `frontend/src/pages/Admin.tsx` | Done | "University of Idaho" (org hierarchy examples) |
| `frontend/src/pages/Demo.tsx` | Done | "University of Idaho" placeholder |
| `frontend/src/pages/Office.tsx` | Done | "/Vandalizer/Intake" placeholder |
| `frontend/src/components/chat/ChatPanel.tsx` | Done | "Vandalizing" -> "Awesomizing", mascot alt text |
| `frontend/src/components/chat/ChatMessage.tsx` | Done | "Vandalizing" -> "Awesomizing" |
| `frontend/src/components/chat/ChatInput.tsx` | Done | "Ask Vandalizer anything" placeholder |
| `frontend/src/components/chat/WelcomeExperience.tsx` | Done | "Vandalizer" in feature descriptions |
| `frontend/src/components/certification/CelebrationOverlay.tsx` | Done | "University of Idaho", "Vandal Workflow Architect" |
| `frontend/src/components/certification/CertifiedBanner.tsx` | Done | "University of Idaho", "Vandal Workflow Architect" |
| `frontend/src/components/workspace/ActivityRail.tsx` | Done | "Vandal Workflow Architect" |
| `frontend/src/components/shared/VerificationSubmitDialog.tsx` | Done | "University of Idaho" placeholder |
| `frontend/src/components/layout/Header.tsx` | Done | Mascot alt text updated |
| `frontend/src/hooks/useOnboarding.ts` | Done | "Vandal Workflow Architect" certification pill |
| `frontend/index.html` | Done | `<title>Awesome.ai</title>` |

#### Backend Services

| File | Status | References |
|------|--------|-----------|
| `backend/app/main.py` | Done | Logging and FastAPI title |
| `backend/app/config.py` | Done | `smtp_from_name` default |
| `backend/app/services/llm_service.py` | Done | All system prompts, certification references |
| `backend/app/services/email_service.py` | Done | All email templates (~40 occurrences) |
| `backend/app/services/chat_service.py` | Done | Help context comment |
| `backend/app/services/feedback_prompt_service.py` | Done | Team name, welcome text, feedback prompts |
| `backend/app/services/support_service.py` | Done | Support HTML templates |
| `backend/app/services/output_handlers.py` | Done | OneDrive path default |
| `backend/app/services/export_import_service.py` | Done | Error message text |
| `backend/app/services/certification_service.py` | Done | Docstring |
| `backend/app/models/certification.py` | Done | Docstring |
| `backend/app/routers/certification.py` | Done | Docstring |
| `backend/app/routers/auth.py` | Done | "Sign in with TTU" OAuth label |
| `backend/app/exceptions.py` | Done | Docstring |
| `backend/bootstrap_install.py` | Done | Docstring |
| `backend/create_admin.py` | Done | Docstring |

#### Certification & Seed Data

| File | Status | References |
|------|--------|-----------|
| `backend/certification-data/exercises.json` | Done | Institution fields |
| `backend/certification-data/generate_pdfs.py` | Done | Sample document institution fields |
| `backend/seeds/knowledge_bases/ui_sponsored_programs.json` | Done | Display name and description |

#### Deployment Scripts

| File | Status | References |
|------|--------|-----------|
| `setup.sh` | Done | Header comment, SMTP_FROM_NAME default |
| `deploy.sh` | Done | Deployment wizard banner, all user-facing text |
| `status.sh` | Done | Status check banner |
| `scripts/test_automation_api.sh` | Done | Header, banner, error message |
| `scripts/reset_db.sh` | Done | Header comment |

#### Environment & Config

| File | Status | References |
|------|--------|-----------|
| `.env.example` | Done | Comment header |
| `backend/.env.example` | Done | Comment header, SMTP_FROM_NAME |
| `backend/pyproject.toml` | Done | Project description |

#### Gamification

| File | Status | References |
|------|--------|-----------|
| `testing.html` | Done | "Vandal Legend" -> "Awesome Legend" |

#### Not Changed (Intentional)

| File | Reason |
|------|--------|
| `README.md`, `CLAUDE.md`, `DEPLOY.md` | Upstream project documentation — describes origin history |
| `CONTRIBUTING.md`, `SECURITY.md` | Upstream community files |
| `WORKFLOW_ARCHITECT.md` | Upstream design document |
| `CHANGELOG.md` | Historical record |
| `Makefile` | Docker image names are technical identifiers (`vandalizer-backend`) |
| `compose.yaml` | Docker network name `vandalizer` is a technical identifier |
| `chrome-extension/` | `window.VandalizerRecorder` etc. are JavaScript class names — renaming breaks functionality |
| `.github/ISSUE_TEMPLATE/` | Upstream templates |
| `backend/app/services/export_import_service.py` | `vandalizer_export` JSON flag is a technical format identifier checked in code |
| `.vandalizer.json` file format references | Actual file extension used in export code — must match |

#### Assets to Replace (Manual)

| File | Description |
|------|-------------|
| `frontend/public/images/joevandal.png` | Replace with Awesome Eagle mascot/logo |
| `frontend/public/images/Vandalizer_Wordmark_Color_RGB+W.png` | Replace with Awesome.ai logo |
| `frontend/public/images/Vandalizer_Wordmark_RGB.png` | Replace with Awesome.ai logo variant |

#### Gamification

| File | Reference |
|------|-----------|
| `testing.html` | "Vandal Legend" badge name |

---

## 13. Backup & Recovery

### 13.1 Persistent Data Map

| Component | Location | Content |
|-----------|----------|---------|
| MongoDB | `mongo-data` volume | All application state (users, documents, workflows, config, audit) |
| Uploads | `uploads` volume | Source PDFs and generated files |
| ChromaDB | `chroma-data` volume | Vector embeddings (rebuildable from source) |
| Secrets | `backend/.env` | JWT secret, encryption keys, SMTP credentials |

### 13.2 Backup Procedure

```bash
STAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$PWD/backups/$STAMP"
mkdir -p "$BACKUP_DIR"

# Record state
git rev-parse HEAD > "$BACKUP_DIR/git-revision.txt"
docker compose config > "$BACKUP_DIR/compose.resolved.yaml"
cp backend/.env "$BACKUP_DIR/backend.env"

# MongoDB
docker compose exec -T mongo \
  sh -lc 'mongodump --archive --gzip --db="${MONGO_DB:-vandalizer}"' \
  > "$BACKUP_DIR/mongo.archive.gz"

# Uploaded files
docker compose exec -T api \
  sh -lc 'tar czf - -C /app/static/uploads .' \
  > "$BACKUP_DIR/uploads.tgz"

# ChromaDB
docker compose exec -T api \
  sh -lc 'tar czf - -C /app/static/db .' \
  > "$BACKUP_DIR/chroma.tgz"
```

**Recommended cadence:** MongoDB and uploads daily + before every upgrade. ChromaDB at minimum before upgrades.

### 13.3 Restore Procedure

```bash
# 1. Stop application services
docker compose stop frontend api celery

# 2. Ensure data services are running
docker compose up -d mongo redis chromadb

# 3. Restore secrets
cp "$BACKUP_DIR/backend.env" backend/.env

# 4. Restore MongoDB
cat "$BACKUP_DIR/mongo.archive.gz" | docker compose exec -T mongo \
  sh -lc 'mongorestore --drop --archive --gzip --db="${MONGO_DB:-vandalizer}"'

# 5. Restore uploads
cat "$BACKUP_DIR/uploads.tgz" | docker compose run --rm -T api \
  sh -lc 'mkdir -p /app/static/uploads && rm -rf /app/static/uploads/* && tar xzf - -C /app/static/uploads'

# 6. Restore ChromaDB
cat "$BACKUP_DIR/chroma.tgz" | docker compose run --rm -T api \
  sh -lc 'mkdir -p /app/static/db && rm -rf /app/static/db/* && tar xzf - -C /app/static/db'

# 7. Start everything
docker compose up -d
```

**Recovery order:** MongoDB first (all app state) → uploads → ChromaDB → Redis (no restore needed) → services.

### 13.4 Redis

No backup needed. Redis is ephemeral — used only for Celery broker/results, session tokens, and rate limiting counters. All are regenerated on restart.

---

## 14. Scaling & HPC Considerations

### 14.1 Celery Worker Scaling

Workers are divided by queue with default concurrency:

| Queue | Default Workers | Tasks |
|-------|----------------|-------|
| `documents` | 8 | Document processing, text extraction, ChromaDB ingestion |
| `workflows` | 6 | Workflow execution, extraction runs, evaluation |
| `uploads` | 4 | File upload processing |
| `passive` | 2 | M365 intake triggers, scheduled automations, Graph subscriptions |
| `default` | 2 | Demo management, activity logging, retention, engagement emails |

On HPC, scale the `documents` and `workflows` queues for heavier workloads. Workers can be run on separate nodes.

### 14.2 FastAPI Worker Scaling

```bash
# Default: 2 workers (set via WEB_WORKERS env var)
uvicorn app.main:app --host 0.0.0.0 --port 8001 --workers 4
```

Increase for higher API concurrency.

### 14.3 Externalizing Services

For HPC deployment, consider:

- **MongoDB:** Use an institutional MongoDB instance or MongoDB Atlas
- **Redis:** Use institutional Redis or AWS ElastiCache
- **ChromaDB:** Run as a dedicated server with `USE_CHROMA_SERVER=true`

Update `MONGO_HOST`, `REDIS_HOST`, `CHROMA_HOST`/`CHROMA_PORT` accordingly.

### 14.4 S3 Storage

For shared filesystem or object storage on HPC:

```env
STORAGE_BACKEND=s3
S3_BUCKET=awesomeai-uploads
S3_REGION=us-east-1
S3_ENDPOINT_URL=https://s3.tntech.edu  # For MinIO or institutional S3
```

### 14.5 Task Timeouts

Celery defaults:
- Soft time limit: 3,600 seconds (1 hour)
- Hard time limit: 3,660 seconds
- Result expiration: 86,400 seconds (1 day)

Adjust for long-running LLM workflows on HPC if needed.

---

## 15. Operational Runbook

### 15.1 Health Checks

```bash
# Full status check
./status.sh

# API health endpoint
curl http://localhost:8001/api/health

# Service logs
docker compose logs --tail=100 api
docker compose logs --tail=100 celery
```

The `/api/health` endpoint checks MongoDB, Redis, and ChromaDB connectivity. Returns 200 (all ok) or 503 (degraded) with per-service status.

### 15.2 Upgrade Procedure

```bash
# Recommended: interactive upgrade
./setup.sh --upgrade

# Manual:
git fetch --tags
git checkout <release-tag>
docker compose build api celery frontend
docker compose up -d
./status.sh
```

### 15.3 Database Reset (Development Only)

```bash
./scripts/reset_db.sh          # Interactive
./scripts/reset_db.sh --force  # Skip confirmation
```

Drops MongoDB database, clears uploads directory, clears ChromaDB.

### 15.4 Common Maintenance

```bash
# Restart application services (preserves data)
docker compose restart api celery frontend

# Rebuild from current code
./setup.sh --redeploy

# Diagnose and fix
./setup.sh --repair

# Update seed catalog
./setup.sh --seed

# Re-ingest knowledge bases into ChromaDB
./setup.sh --reingest
```

### 15.5 Periodic Tasks (Celery Beat)

These run automatically when Celery Beat is active:

| Task | Schedule | Purpose |
|------|----------|---------|
| Process demo waitlist | Every 5 min | Activate pending demo accounts |
| Check demo expirations | Every hour | Lock expired demo accounts |
| Process pending triggers | Every 60 sec | Execute queued workflow triggers |
| Process scheduled automations | Every 60 sec | Run scheduled automations |
| Renew Graph subscriptions | Every 12 hours | Renew M365 webhook subscriptions |
| Send daily digest | Daily 8:00 AM | Email activity summary to users |
| Cleanup trigger events | Daily 3:00 AM | Prune old trigger event records |
| Quality monitor | Daily | Check extraction quality metrics |
| Schedule deletions | Daily 2:00 AM | Mark documents for soft deletion per retention policy |
| Execute soft deletes | Daily 3:00 AM | Soft-delete scheduled documents |
| Execute hard deletes | Daily 4:00 AM | Permanently delete soft-deleted items past grace period |
| Cleanup ancillary data | Daily 5:00 AM | Clean up related chat/activity data |
| Onboarding drips | Daily 10:00 AM | Send onboarding email sequence |
| Inactivity nudges | Daily 10:30 AM | Send re-engagement emails |

---

## 16. Security Considerations

### 16.1 Middleware Stack

The application applies these security layers in order:

1. **Security Headers:** X-Content-Type-Options, X-Frame-Options (DENY), Referrer-Policy, Permissions-Policy, X-XSS-Protection, Content-Security-Policy, HSTS (production)
2. **CORS:** Configured to allow only `FRONTEND_URL` origin
3. **CSRF:** Double-submit cookie pattern — `X-CSRF-Token` header required on state-changing requests
4. **Rate Limiting:** Per-endpoint limits (5/min login, 3/min register, 30/min chat, etc.)

### 16.2 Credential Management

- LLM API keys encrypted at rest in MongoDB using Fernet (`CONFIG_ENCRYPTION_KEY`)
- OAuth client secrets encrypted in SystemConfig
- Graph API tokens encrypted with separate Fernet key (`GRAPH_TOKEN_KEY`)
- JWT signing with strong secret (`JWT_SECRET_KEY`)
- No secrets in environment variables for LLM providers — all managed via Admin UI

### 16.3 Data Classification

Documents can be auto-classified on upload:
- **Unrestricted** — public/general documents
- **Internal** — institutional use only
- **FERPA** — student records (Family Educational Rights and Privacy Act)
- **CUI** — Controlled Unclassified Information
- **ITAR** — International Traffic in Arms Regulations

Each level has configurable retention policies with soft-delete grace periods.

### 16.4 Audit Logging

- All user actions logged to `AuditLog` (actor, action, resource, IP address, timestamp)
- Admin actions logged to `AdminAuditLog` with redacted payloads
- Immutable, append-only audit trail

### 16.5 Non-Root Containers

The backend Docker image runs as `appuser` (UID 1000), not root.

---

## Appendix A: Complete File Structure

```
Awesome.ai/
├── compose.yaml              # Docker Compose orchestration (6 services)
├── setup.sh                  # Interactive setup wizard
├── deploy.sh                 # Deployment wizard
├── status.sh                 # Health check script
├── Makefile                  # CI/CD build targets
├── .env.example              # Root environment template
│
├── backend/
│   ├── Dockerfile            # Multi-stage Python 3.12 build
│   ├── pyproject.toml        # Python dependencies (uv)
│   ├── .env.example          # Backend environment template
│   ├── run_celery.sh         # Celery worker management
│   ├── start_server.sh       # Dev server start
│   ├── bootstrap_install.py  # First-run database setup
│   ├── create_admin.py       # Admin account creation
│   ├── app/
│   │   ├── main.py           # FastAPI app, middleware, lifespan
│   │   ├── config.py         # Pydantic Settings
│   │   ├── database.py       # Beanie/MongoDB initialization
│   │   ├── dependencies.py   # Auth dependency injection
│   │   ├── celery_app.py     # Celery configuration
│   │   ├── exceptions.py     # Custom exception hierarchy
│   │   ├── rate_limit.py     # slowapi rate limiter
│   │   ├── middleware/        # CSRF middleware
│   │   ├── models/           # 31 Beanie Document model files
│   │   ├── routers/          # 28 FastAPI router modules
│   │   ├── services/         # 48 business logic service files
│   │   ├── tasks/            # 15 Celery task modules
│   │   └── schemas/          # 11 request/response Pydantic schemas
│   ├── seeds/                # Default workflow & KB templates
│   ├── certification-data/   # Certification exercise PDFs & data
│   ├── scripts/              # Utility scripts
│   └── tests/                # Backend test suite
│
├── frontend/
│   ├── Dockerfile            # Multi-stage Node 22 + nginx build
│   ├── nginx.conf            # Production nginx configuration
│   ├── package.json          # NPM dependencies and scripts
│   ├── vite.config.ts        # Vite build configuration
│   ├── .env.example          # Frontend environment template
│   ├── src/
│   │   ├── App.tsx           # Root component, provider hierarchy
│   │   ├── router.tsx        # TanStack Router route definitions
│   │   ├── index.css         # Tailwind CSS + custom theme
│   │   ├── api/              # 33 API client modules
│   │   ├── pages/            # 15 page components
│   │   ├── components/       # 97 components in 12 feature dirs
│   │   ├── contexts/         # Auth, Team, Workspace, Toast, Cert
│   │   ├── hooks/            # 21 custom React hooks
│   │   └── lib/              # Utility functions, query client
│   ├── public/images/        # Logos, mascot, favicon
│   └── e2e/                  # Playwright E2E tests
│
├── scripts/
│   ├── backup_mongo.sh       # MongoDB backup with retention
│   ├── reset_db.sh           # Development database reset
│   └── test_automation_api.sh # API trigger testing
│
└── docs/                     # Documentation
```

---

## Appendix B: All Celery Task Queues

### Queue Routing

| Pattern | Queue |
|---------|-------|
| `tasks.document.*` | documents |
| `tasks.workflow.*` | workflows |
| `tasks.upload.*` | uploads |
| `tasks.extraction.*` | workflows |
| `tasks.evaluation.*` | workflows |
| `tasks.passive.*` | passive |
| `tasks.activity.*` | default |
| `tasks.demo.*` | default |
| `tasks.retention.*` | default |

### Task Modules

| Module | Key Tasks |
|--------|-----------|
| `document_tasks` | classify_document, perform_extraction_and_update, perform_semantic_ingestion, cleanup_document |
| `workflow_tasks` | execute_workflow_task, approval notifications |
| `extraction_tasks` | perform_extraction_task, ingest_extraction_recommendation |
| `upload_tasks` | File upload processing |
| `upload_validation_tasks` | File format validation |
| `evaluation_tasks` | generate_evaluation_plan, run_validation |
| `knowledge_base_tasks` | kb_ingest_document, kb_ingest_url |
| `activity_tasks` | generate_activity_description |
| `classification_tasks` | Document sensitivity classification |
| `quality_tasks` | Quality monitoring, alert generation |
| `passive_tasks` | M365 ingestion, triage, Graph subscriptions, daily digest, trigger processing |
| `demo_tasks` | Demo waitlist, expiration, warnings |
| `retention_tasks` | Soft/hard deletion scheduling and execution |
| `engagement_tasks` | Onboarding drips, inactivity nudges |
| `m365_tasks` | Microsoft 365 integration tasks |

---

## Appendix C: Database Collections

### Core Authentication & Multi-Tenancy

- **User** — user_id, email, password_hash, roles (admin/staff/examiner), current_team, organization_id, demo tracking, M365 flags, API token, engagement metrics
- **Team** — uuid, name, owner_user_id, organization_id
- **TeamMembership** — team ref, user_id, role (owner/admin/member)
- **TeamInvite** — team ref, email, invite token, accepted flag
- **Organization** — uuid, name, org_type (university/college/department/unit), parent hierarchy

### Documents & Storage

- **SmartDocument** — uuid, path, raw_text, title, extension, token_count, num_pages, classification level, retention tracking, folder ref, processing state
- **SmartFolder** — uuid, title, parent_id, user_id, team_id

### Workflows & Extraction

- **Workflow** — uuid, name, steps (ordered), attachments, execution count, version, validation plan
- **WorkflowStep** — name, tasks list, config data, is_output flag
- **WorkflowStepTask** — name, task config data
- **WorkflowResult** — session_id, status (running/completed/failed), progress tracking, outputs, timing, batch_id, approval state
- **WorkflowArtifact** — result ref, artifact type, file path, extracted data
- **SearchSet** — uuid, title, extraction config, domain (nsf/nih/dod), cross-field rules, tuning results
- **SearchSetItem** — search phrase, type, constraints, PDF bindings

### Chat & Knowledge

- **ChatConversation** — uuid, title, messages list, file/URL attachments
- **ChatMessage** — role (user/assistant), content, thinking data
- **KnowledgeBase** — uuid, title, status, sources, chunk counts, sharing, verification
- **KnowledgeBaseSource** — source type (document/url), processing status, chunk count
- **KnowledgeBaseReference** — user's reference to a verified KB

### Automation & Verification

- **Automation** — trigger type (folder_watch/m365/api/schedule), action config, enabled flag
- **ApprovalRequest** — workflow pause point, data for review, reviewer assignment, decision
- **QualityAlert** — quality monitoring alerts
- **ValidationRun** — extraction validation test runs
- **ExtractionTestCase** — gold-standard test cases for validation

### System Configuration

- **SystemConfig** (singleton) — LLM models, OCR endpoint, extraction config, quality gates, classification levels, retention policies, auth methods, OAuth/SAML providers, M365 config, UI theme, support contacts
- **UserModelConfig** — per-user model preferences

### Activity & Audit

- **ActivityEvent** — type, status, user/team, timing, token usage
- **AuditLog** — actor, action, resource, detail, IP address (append-only)
- **AdminAuditLog** — admin-specific actions with redacted payloads
- **Notification** — in-app notifications

### M365 Integration

- **IntakeConfig** — intake source type, mailbox/folder config, default workflow, triage settings
- **WorkItem** — ingested items with triage results, sensitivity flags, routing

### Other

- **CertificationProgress** — module completion, XP, level, streak tracking
- **DemoApplication** — demo waitlist, approval, expiry lifecycle
- **Library/LibraryItem/LibraryFolder** — shared template management
- **FeedbackPrompt/FeedbackPromptResponse** — user feedback collection

---

*This document was generated from a comprehensive review of the Awesome.ai codebase for Tennessee Tech University deployment planning.*
