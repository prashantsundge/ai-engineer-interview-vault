# SharePoint Knowledge Assistant
### Enterprise Multimodal RAG Chatbot with Microsoft Entra ID SSO

> Built by PrashantKumar Sundge — 
> Stack: React 19 · FastAPI · LangGraph · Qdrant · GPT-4o-mini · MSAL v5

---

## What This Is

An enterprise-grade AI chatbot that lets your team query SharePoint documents using natural language. It retrieves runbooks, LLDs, RCAs, configuration files, and more — returning grounded, sourced answers with confidence scoring and clickable SharePoint links.

**It is not a generic chatbot.** Every answer comes exclusively from your organisation's SharePoint content. The system cites the exact file, sheet, and page the answer came from.

---

## Architecture Overview

```
User (Browser)
    │
    ▼
React SPA (localhost:3000)
├── Microsoft Entra ID SSO (MSAL v5 + PKCE)
├── Conversation Memory (session-scoped)
└── SSE Streaming UI with Markdown / Table rendering
    │
    ▼ Bearer Token + Query + History
FastAPI Backend (localhost:8000)
├── JWT Token Validation (Entra ID)
├── RBAC — Entra ID Security Groups
└── LangGraph Agent Pipeline
        │
        ├── Intent Node          → retrieval / comparison / follow-up detection
        ├── Retrieval Node       → Hybrid Vector + BM25 + Fuzzy filename boost
        ├── Reranker Node        → Cross-Encoder reranking (ms-marco-MiniLM-L-6-v2)
        ├── Diagnostics Node     → Spread, diversity, stability scoring
        ├── Context Node         → Adaptive top-K, spreadsheet-aware
        ├── Generation Node      → GPT-4o-mini with enterprise prompt
        ├── Confidence Node      → Calibrated scoring, hallucination detection
        └── Follow-up Node       → Contextual suggestion chips
            │
            ▼
    Qdrant Vector DB (localhost:6333)
    ├── Dense vectors: all-MiniLM-L6-v2 (local)
    └── Sparse index: BM25 (rank_bm25)

SharePoint Ingestion Pipeline (Celery + Redis)
├── Delta-based sync (only changed files)
├── Multi-modal extraction: PDF, XLSX, DOCX, PPTX, PNG/JPG (OCR), TXT, MSG
├── Folder-level security filter (Finance, HR, Legal excluded at ingestion)
└── SHA-256 deduplication
```

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Entra ID SSO** | PKCE auth code flow via MSAL v5. Single-page application platform |
| **Hybrid Retrieval** | Vector (semantic) + BM25 (lexical) + fuzzy filename boosting |
| **Cross-Encoder Reranking** | ms-marco-MiniLM-L-6-v2 for precision scoring |
| **Conversation Memory** | Follow-up questions stay within the active document context |
| **Multimodal Ingestion** | PDF, Excel, Word, PowerPoint, Images (OCR), plain text, email |
| **Delta Sync** | SharePoint delta API — only re-ingests changed files |
| **Security Filtering** | Restricted folders blocked at both ingestion and retrieval time |
| **Confidence Scoring** | Composite score: similarity + rerank + grounding + hallucination detection |
| **Source Cards** | Clickable SharePoint links with file name, sheet/page, modality |
| **Markdown Tables** | GFM table rendering for spreadsheet data |
| **Follow-up Chips** | Suggested next questions as clickable buttons |
| **Streaming** | Word-by-word typewriter effect via SSE |

---

## Project Structure

```
SHAREPOINT_CHATBOT/
├── app/
│   ├── agents/                  # LangGraph nodes
│   │   ├── state.py             # AgentState TypedDict
│   │   ├── graph_builder.py     # Graph wiring
│   │   ├── intent_node.py       # Intent + follow-up detection
│   │   ├── retrieval_node.py    # Hybrid retrieval + security filter
│   │   ├── reranker_node.py     # Cross-encoder reranking
│   │   ├── diagnostics_node.py  # Retrieval quality metrics
│   │   ├── context_node.py      # Context assembly (adaptive top-K)
│   │   ├── generation_node.py   # LLM answer generation
│   │   ├── confidence_node.py   # Confidence scoring
│   │   ├── followup_node.py     # Follow-up suggestions
│   │   └── comparison_node.py   # Multi-document comparison
│   ├── api/
│   │   ├── routes.py            # FastAPI endpoints (/chat, /chat/stream)
│   │   └── dependencies.py      # JWT validation, service injection
│   ├── core/
│   │   ├── config.py            # Settings (env vars)
│   │   ├── logger.py            # Structured logging
│   │   ├── sanitizer.py         # PII/credential scrubbing
│   │   └── exceptions.py        # Custom exceptions
│   ├── embeddings/
│   │   ├── local_provider.py    # all-MiniLM-L6-v2 (runs locally)
│   │   └── service.py           # Embedding service wrapper
│   ├── graph/
│   │   ├── crawler.py           # SharePoint file crawler
│   │   └── client.py            # Microsoft Graph API client
│   ├── ingestion/
│   │   ├── orchestrator.py      # Ingestion coordinator
│   │   ├── tasks.py             # Celery task definitions
│   │   ├── extractor_router.py  # Routes files to correct extractor
│   │   ├── hashing.py           # SHA-256 deduplication
│   │   └── delta_store.py       # Delta token persistence
│   ├── llm/
│   │   ├── openai_service.py    # GPT-4o-mini (sync + async streaming)
│   │   └── base.py              # Base LLM interface
│   ├── retrieval/
│   │   └── bm25_index.py        # BM25 sparse index
│   ├── security/
│   │   ├── content_guard.py     # Output content filtering
│   │   └── scrubbing.py         # Regex PII redaction engine
│   ├── vectorstore/
│   │   ├── qdrant_client.py     # Qdrant connection manager
│   │   └── repository.py        # Vector search operations
│   └── main.py                  # FastAPI app + CORS + router
│
├── frontend/
│   └── src/
│       ├── App.tsx              # Auth shell + React Router
│       ├── index.tsx            # MSAL init + BrowserRouter
│       ├── authConfig.ts        # MSAL configuration
│       ├── components/
│       │   └── ChatContainer.tsx  # Full chat UI (sidebar, markdown, SSE)
│       ├── pages/
│       │   └── ChatPage.tsx     # Chat page wrapper
│       └── types/
│           └── chat.ts          # TypeScript interfaces
│
├── scripts/
│   ├── ingest.py                # Manual ingestion trigger
│   ├── purge_restricted_chunks.py  # Remove restricted content from Qdrant
│   └── reset_delta_token.py     # Reset SharePoint delta sync
│
├── tests/
├── logs/
├── .env                         # Environment variables (never commit)
├── requirements.txt
└── README.md
```

---

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| Python | 3.11+ | Backend runtime |
| Node.js | 18+ | Frontend build |
| Qdrant | Latest | Vector database |
| Redis | Latest | Celery broker |
| Tesseract OCR | 5.x | Image text extraction |
| Azure App Registration | — | SSO + Graph API access |

---

## Environment Variables

Create `.env` in the project root:

```env
# SharePoint / Microsoft Graph
TENANT_ID=your-tenant-id
CLIENT_ID=your-app-client-id
CLIENT_SECRET=your-app-client-secret
SHAREPOINT_HOSTNAME=yourcompany.sharepoint.com
SHAREPOINT_SITE_PATH=/sites/your-site-name

# OpenAI
OPENAI_API_KEY=sk-...

# Qdrant
QDRANT_HOST=localhost
QDRANT_PORT=6333

# Redis / Celery
CELERY_BROKER_URL=redis://localhost:6379/0

# LangSmith (optional — for tracing)
LANGCHAIN_TRACING_V2=false
LANGCHAIN_API_KEY=
LANGCHAIN_PROJECT=sharepoint-multimodal-rag

# OCR
TESSERACT_PATH=C:\Program Files\Tesseract-OCR\tesseract.exe
```

---

## Azure App Registration Setup

1. Go to **Azure Portal → Azure Active Directory → App registrations → New registration**

2. **Authentication → Add a platform → Single-page application** (NOT Web)
   - Redirect URI: `http://localhost:3000`
   - ✅ Must be SPA platform — Web platform blocks PKCE and causes auth loops

3. **API permissions → Add permission**
   - Microsoft Graph → Delegated → `User.Read`
   - Click **Grant admin consent**

4. **Certificates & secrets → New client secret**
   - Copy the value to `.env` as `CLIENT_SECRET`

5. **Overview** — copy Application (client) ID → `CLIENT_ID`, Directory (tenant) ID → `TENANT_ID`

---

## Installation

### Backend

```bash
cd D:\CONNX 2026\SHAREPOINT_CHATBOT

# Create virtual environment
python -m venv venv
venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start Qdrant (Docker)
docker run -p 6333:6333 qdrant/qdrant

# Start Redis (Docker)
docker run -p 6379:6379 redis

# Start FastAPI backend
uvicorn app.main:app --reload --port 8000
```

### Frontend

```bash
cd frontend

# Install dependencies
npm install

# Start React dev server
npm start
```

### Ingestion

```bash
# Run SharePoint ingestion (delta sync)
cd D:\CONNX 2026\SHAREPOINT_CHATBOT
python scripts/ingest.py

# Start Celery worker (in separate terminal)
celery -A app.core.celery_app worker --loglevel=info -P threads
```

---

## Running the Full Stack

```
Terminal 1: docker run -p 6333:6333 qdrant/qdrant
Terminal 2: docker run -p 6379:6379 redis
Terminal 3: venv\Scripts\activate && uvicorn app.main:app --reload --port 8000
Terminal 4: cd frontend && npm start
Terminal 5: celery -A app.core.celery_app worker --loglevel=info -P threads
```

Then open `http://localhost:3000` and sign in with your corporate Microsoft account.

---

## Security Model

### Authentication
- MSAL v5 with PKCE Authorization Code Flow
- Tokens validated in `dependencies.py` against your tenant ID
- Session storage (clears on browser close — no persistent credentials)

### Authorisation
- Entra ID security group GUIDs extracted from JWT claims
- Passed as `user_groups` through entire LangGraph pipeline
- RBAC filtering applied in retrieval node (folder → group mapping)

### Content Security
Two layers of restricted content protection:

1. **Ingestion time** (`orchestrator.py`) — folders matching restricted keywords are never ingested into Qdrant
2. **Retrieval time** (`retrieval_node.py`) — even if content exists in Qdrant, chunks from restricted paths are stripped before reaching the LLM

Restricted keywords: `finance`, `salary`, `payroll`, `invoice`, `hr`, `legal`, `confidential`, `accounts`, `compensation`

### PII Scrubbing
`sanitizer.py` and `scrubbing.py` apply regex patterns to redact passwords, credentials, and sensitive values in both ingested content and LLM output.

---

## API Reference

### POST /chat/stream

Streaming SSE endpoint for the chat interface.

**Request:**
```json
{
  "query": "nutreco spectralink runbook",
  "conversation_history": [
    {"role": "user", "content": "previous question"},
    {"role": "assistant", "content": "previous answer"}
  ],
  "active_document": "Nutreco Spectralink Runbook New1.xlsx"
}
```

**Headers:**
```
Authorization: Bearer <access_token>
Content-Type: application/json
```

**Response (SSE stream):**
```
data: {"token": "The"}
data: {"token": " Spectralink"}
...
event: metadata
data: {"confidence_score": 0.87, "sources": [...], "followups": [...]}
event: done
data: end
```

### POST /chat

Blocking (non-streaming) endpoint for programmatic use.

### GET /

Health check — returns `{"status": "running"}`

---

## Maintenance

### Reset Delta Sync
If SharePoint delta token becomes stale:
```bash
python scripts/reset_delta_token.py
```

### Purge Restricted Content from Qdrant
```bash
python scripts/purge_restricted_chunks.py
```

### Check Qdrant Collection Stats
```bash
python scripts/Checkqdrant.py
```

---

## Known Limitations

| Limitation | Workaround |
|------------|-----------|
| Graph token signature unverifiable | Tenant ID check in place; upgrade to Chat.Access scope for full verification |
| Conversation history is session-only | Clears on page refresh or new session |
| Celery workers require Redis | Can run synchronous ingestion without Celery for small datasets |
| OCR quality depends on image resolution | Low-resolution images score lower via OCR penalty in retrieval |
| BM25 index rebuilt on startup | Adds ~10s cold start for large collections (2260+ chunks) |

---

## Roadmap

- [ ] Expose `Chat.Access` scope in Azure for full JWT signature verification
- [ ] Persistent conversation history (database-backed)
- [ ] RBAC mapping: Entra ID groups → SharePoint folder ACLs
- [ ] Streaming directly from OpenAI (true token-by-token, not word replay)
- [ ] Docker Compose for one-command startup
- [ ] Admin dashboard for ingestion monitoring
- [ ] Multi-tenant support

---

## Author

**Pravin Kumar Sundge**  
Senior Technical Lead | Informatica MDM | IDMC | Enterprise Data Management  
CONNX 2026 — SharePoint AI Chatbot Project  
*Built June 2026*
