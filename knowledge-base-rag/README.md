# 🧠 KnowledgeBase AI — RAG-Powered Document Intelligence Platform

A full-stack **Retrieval-Augmented Generation (RAG)** system that lets you upload documents and have an AI answer questions about them — with **streaming responses**, **source citations**, and a polished single-page web interface.

Built with **Sails.js**, **MongoDB**, **PostgreSQL + pgvector**, and **Ollama** (fully local AI — no OpenAI API key required).

---

## ✨ Key Features

- 📄 **Multi-format document ingestion** — PDF, TXT, Markdown, Images (OCR), URLs
- 🔍 **Semantic search** — vector similarity search, no keyword matching needed
- 💬 **Streaming RAG chat** — real-time token-by-token answers via SSE
- 📌 **Source citations** — every AI answer shows which document chunks it used
- ⭐ **Feedback system** — users rate AI responses 1–5 stars, editable anytime
- 📊 **Admin dashboard** — analytics for documents, chunks, sessions, ratings
- 🔐 **JWT authentication** — role-based access (admin / user)
- 🖥️ **100% local AI** — runs fully offline with Ollama (no API fees)
- 🗃️ **Dual-database architecture** — MongoDB for metadata + pgvector for embeddings

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser (SPA)                           │
│   app.html — Login / Dashboard / Docs / Search / Chat       │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP + SSE
┌──────────────────────────▼──────────────────────────────────┐
│                   Sails.js API Server                       │
│   AuthController  DocumentController  ChatController        │
│   SearchController  FeedbackController  AdminController     │
└────────┬──────────────────┬─────────────────────────────────┘
         │                  │
┌────────▼──────┐  ┌────────▼────────────────────────────────┐
│   MongoDB     │  │              Services Layer              │
│               │  │  Documentprocessor  Chunkingservice      │
│  • Users      │  │  Embeddingservice   Vectorstoreservice   │
│  • Documents  │  │  PgService          RagService           │
│  • Sessions   │  └────────────────┬────────────────────────-┘
│  • Messages   │                   │
│  • Feedback   │  ┌────────────────▼────────────────────────┐
└───────────────┘  │  PostgreSQL + pgvector                  │
                   │  document_chunks (768-dim vectors)       │
                   └────────────────┬────────────────────────-┘
                                    │
                   ┌────────────────▼────────────────────────┐
                   │            Ollama (Local LLM)           │
                   │  nomic-embed-text  →  embeddings        │
                   │  llama3.2          →  chat answers      │
                   └─────────────────────────────────────────┘
```

---

## 🔄 How the RAG Pipeline Works

### Phase 1 — Document Upload & Processing

```
User uploads PDF
        │
        ▼
DocumentController.upload()
        │
        ▼
Documentprocessor.extractText()    ← pdf-parse / tesseract.js / cheerio
        │
        ▼
Chunkingservice.splitIntoChunks()  ← 800 chars, 150 overlap
        │
        ▼
Embeddingservice.embedChunks()     ← Ollama nomic-embed-text → 768-dim vectors
        │
        ▼
Vectorstoreservice.indexDocument() ← Batch insert into PostgreSQL pgvector
        │
        ▼
KnowledgeDocument.status = 'indexed'
```

### Phase 2 — Semantic Search (No LLM)

```
User types query
        │
        ▼
Embeddingservice.embedText()       ← Same model as indexing (critical!)
        │
        ▼
PgService.similaritySearch()       ← pgvector cosine similarity
        │
        ▼
Return top-K matching chunks       ← With similarity score, doc title, page number
```

### Phase 3 — RAG Chat with Streaming

```
User sends question
        │
        ▼
RagService.chatStream()
        │
        ├── embedText()            ← Embed the question
        │
        ├── similaritySearch()     ← Find relevant chunks from pgvector
        │
        ├── ChatMessage.find()     ← Load last N turns from MongoDB
        │
        ├── Build LangChain prompt ← System prompt + context + history + question
        │
        └── ChatOllama.stream()    ← Stream tokens via SSE to browser
                │
                ▼
        EventSource in browser receives chunks in real time
                │
                ▼
        Full answer + sources saved to MongoDB
```

---

## 🗂️ Project Structure

```
knowledge-base-rag/
│
├── app.js                        # Entry point (Node.js polyfills + Sails boot)
│
├── assets/
│   ├── app.html                  # Full SPA — all 5 pages in one file
│   └── chat.html                 # Legacy standalone chat page
│
├── api/
│   ├── controllers/
│   │   ├── AuthController.js     # Register, login, logout, /me
│   │   ├── DocumentController.js # Upload, list, status, delete docs
│   │   ├── ChatController.js     # Create session, stream, history, clear
│   │   ├── SearchController.js   # Semantic search (no LLM)
│   │   ├── FeedbackController.js # Submit + get ratings
│   │   └── AdminController.js    # Analytics dashboard data
│   │
│   ├── models/
│   │   ├── User.js               # email, passwordHash, role, isActive
│   │   ├── KnowledgeDocument.js  # title, type, status, filePath, chunkCount
│   │   ├── ChatSession.js        # sessionId, userId, title
│   │   ├── ChatMessage.js        # sessionId, role, content, sources
│   │   └── Feedback.js           # messageId, userId, rating (1–5)
│   │
│   ├── services/
│   │   ├── Documentprocessor.js  # Extract text from PDF/TXT/MD/image/URL
│   │   ├── Chunkingservice.js    # Split text into overlapping chunks
│   │   ├── Embeddingservice.js   # Generate 768-dim vectors via Ollama
│   │   ├── Vectorstoreservice.js # Bridge: embeddings ↔ PostgreSQL
│   │   ├── PgService.js          # PostgreSQL pool, queries, similarity search
│   │   └── RagService.js         # Full RAG pipeline + streaming
│   │
│   └── policies/
│       ├── isAuthenticated.js    # Verify JWT token (header or query param)
│       └── isAdmin.js            # Restrict to admin role
│
├── config/
│   ├── routes.js                 # All API + static file routes
│   ├── policies.js               # Route → policy mapping
│   ├── datastores.js             # MongoDB connection (sails-mongo)
│   ├── bootstrap.js              # Startup checks: MongoDB, PostgreSQL, Ollama
│   ├── http.js                   # Custom body parser (bypass Skipper for uploads)
│   ├── custom.js                 # JWT_SECRET, token expiry
│   └── security.js               # CORS settings
│
└── uploads/                      # Uploaded files (gitignored)
```

---

## 🛠️ Tech Stack

| Layer                 | Technology                             | Purpose                                        |
| --------------------- | -------------------------------------- | ---------------------------------------------- |
| **Backend Framework** | [Sails.js v1.5](https://sailsjs.com)   | MVC, ORM, routing, policies                    |
| **Document DB**       | MongoDB + sails-mongo                  | Users, documents, sessions, messages, feedback |
| **Vector DB**         | PostgreSQL + pgvector                  | 768-dim embeddings, cosine similarity search   |
| **AI / LLM**          | [Ollama](https://ollama.ai) + llama3.2 | Chat completions (local, offline)              |
| **Embeddings**        | Ollama + nomic-embed-text              | Text → 768-dim vectors (local, offline)        |
| **LLM Framework**     | LangChain (JS)                         | Prompt templates, output parsers, streaming    |
| **File Upload**       | Multer v2                              | Multipart form handling                        |
| **PDF Parsing**       | pdf-parse                              | Extract text from PDF files                    |
| **OCR**               | Tesseract.js                           | Extract text from images                       |
| **Web Scraping**      | Cheerio                                | Extract text from URLs                         |
| **Authentication**    | JWT (jsonwebtoken) + bcryptjs          | Stateless auth, password hashing               |
| **Streaming**         | SSE (Server-Sent Events)               | Real-time token streaming to browser           |
| **Frontend**          | Pure HTML / CSS / JS                   | Zero-framework SPA                             |
| **Runtime**           | Node.js v20+                           | JavaScript runtime                             |

---

## ⚙️ Prerequisites

Before running this project, install the following:

| Tool               | Version | Install                                                              |
| ------------------ | ------- | -------------------------------------------------------------------- |
| Node.js            | v20+    | [nodejs.org](https://nodejs.org)                                     |
| MongoDB            | 5+      | [mongodb.com](https://www.mongodb.com/try/download/community)        |
| PostgreSQL         | 14+     | [postgresql.org](https://www.postgresql.org/download/)               |
| pgvector extension | latest  | [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector) |
| Ollama             | latest  | [ollama.ai/download](https://ollama.ai/download)                     |

> ⚠️ **Important**: Install Ollama from the **official website** (not Homebrew) to avoid the `llama-server` binary bug on macOS.

---

## 🚀 Installation & Setup

### 1. Clone the repository

```bash
git clone https://github.com/ztlab119/knowledge-base-rag.git
cd knowledge-base-rag
```

### 2. Install dependencies

```bash
npm install
```

### 3. Set up environment variables

Create a `.env` file in the project root:

```env
# MongoDB
MONGODB_URL=mongodb://localhost:27017/RAG_knowledge_base

# PostgreSQL
PGHOST=localhost
PGPORT=5432
PGDATABASE=RAG_knowledge_base
PGUSER=postgres
PGPASSWORD=your_postgres_password

# JWT
JWT_SECRET=your_super_secret_key_change_this_in_production
JWT_EXPIRES_IN=7d

# Ollama
OLLAMA_BASE_URL=http://localhost:11434

# Node environment
NODE_ENV=development
```

### 4. Set up PostgreSQL database

Connect to PostgreSQL and run:

```sql
-- Create the database
CREATE DATABASE "RAG_knowledge_base";

-- Connect to it
\c RAG_knowledge_base

-- Install pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create the chunks table
CREATE TABLE document_chunks (
    id          BIGSERIAL PRIMARY KEY,
    doc_id      VARCHAR(255) NOT NULL,
    doc_title   VARCHAR(500),
    doc_type    VARCHAR(50),
    chunk_index INTEGER DEFAULT 0,
    page_number INTEGER DEFAULT 1,
    content     TEXT NOT NULL,
    embedding   vector(768),
    metadata    JSONB,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for fast lookup
CREATE INDEX idx_chunks_doc_id ON document_chunks(doc_id);
CREATE INDEX idx_chunks_embedding ON document_chunks
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

### 5. Pull Ollama models

```bash
# Pull the LLM (for chat answers)
ollama pull llama3.2

# Pull the embedding model (for vector search)
ollama pull nomic-embed-text

# Verify both are downloaded
ollama list
```

### 6. Start Ollama server

```bash
ollama serve
```

### 7. Create your admin user

```bash
curl -X POST http://localhost:1337/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Admin",
    "email": "admin@test.com",
    "password": "Test1234",
    "role": "admin"
  }'
```

### 8. Start the development server

```bash
npm run dev
```

The server starts at **http://localhost:1337**

---

## 🌐 Accessing the Application

| URL                               | Description                |
| --------------------------------- | -------------------------- |
| `http://localhost:1337/app.html`  | **Main SPA** — all 5 pages |
| `http://localhost:1337/chat.html` | Standalone chat page       |

---

## 📡 API Reference

All API routes return JSON. Protected routes require `Authorization: Bearer <token>` header.

### Authentication

| Method | Route                | Auth   | Description           |
| ------ | -------------------- | ------ | --------------------- |
| `POST` | `/api/auth/register` | Public | Register new user     |
| `POST` | `/api/auth/login`    | Public | Login, returns JWT    |
| `POST` | `/api/auth/logout`   | User   | Logout                |
| `GET`  | `/api/auth/me`       | User   | Get current user info |

**Login request:**

```json
POST /api/auth/login
{
  "email": "admin@test.com",
  "password": "Test1234"
}
```

**Login response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "...",
    "name": "Admin",
    "email": "admin@test.com",
    "role": "admin"
  }
}
```

---

### Documents

| Method   | Route                       | Auth  | Description                       |
| -------- | --------------------------- | ----- | --------------------------------- |
| `POST`   | `/api/documents/upload`     | Admin | Upload a file (field: `document`) |
| `POST`   | `/api/documents/url`        | Admin | Index a URL                       |
| `GET`    | `/api/documents`            | Admin | List all documents                |
| `GET`    | `/api/documents/:id`        | Admin | Get document details              |
| `GET`    | `/api/documents/:id/status` | Admin | Get indexing status               |
| `DELETE` | `/api/documents/:id`        | Admin | Delete document + its chunks      |

**Upload a file:**

```bash
curl -X POST http://localhost:1337/api/documents/upload \
  -H "Authorization: Bearer <token>" \
  -F "document=@/path/to/file.pdf"
```

**Document status values:**

| Status     | Meaning                                |
| ---------- | -------------------------------------- |
| `pending`  | Just uploaded, waiting to be processed |
| `indexing` | Currently extracting and embedding     |
| `indexed`  | Ready for search and chat              |
| `failed`   | Processing failed (check logs)         |

---

### Chat

| Method   | Route                  | Auth | Description                     |
| -------- | ---------------------- | ---- | ------------------------------- |
| `POST`   | `/api/chat/session`    | User | Create a new chat session       |
| `GET`    | `/api/chat/sessions`   | User | List all user's sessions        |
| `GET`    | `/api/chat/stream`     | User | Stream a RAG answer (SSE)       |
| `GET`    | `/api/chat/:sessionId` | User | Get chat history                |
| `DELETE` | `/api/chat/:sessionId` | User | Clear all messages in a session |

**Streaming chat (SSE):**

```
GET /api/chat/stream?sessionId=abc&question=What+is+the+refund+policy&token=<jwt>
```

SSE events received by the client:

```
data: {"chunk": "The "}
data: {"chunk": "refund "}
data: {"chunk": "policy..."}
data: {"sources": [{"title": "Policy Doc", "page": 2}]}
data: [DONE]
```

---

### Search

| Method | Route                      | Auth | Description              |
| ------ | -------------------------- | ---- | ------------------------ |
| `GET`  | `/api/search?q=...&topK=5` | User | Semantic search (no LLM) |

**Response:**

```json
{
  "query": "refund policy",
  "results": [
    {
      "docId": "...",
      "docTitle": "Company Policy.pdf",
      "docType": "pdf",
      "pageNumber": 2,
      "chunkIndex": 5,
      "content": "...matching passage...",
      "similarity": 0.9241
    }
  ],
  "total": 5,
  "tookMs": 23
}
```

---

### Feedback

| Method | Route                      | Auth | Description                       |
| ------ | -------------------------- | ---- | --------------------------------- |
| `POST` | `/api/feedback`            | User | Submit a rating for an AI message |
| `GET`  | `/api/feedback/:messageId` | User | Get ratings for a message         |

```json
POST /api/feedback
{
  "messageId": "abc123",
  "rating": 5
}
```

---

### Admin Analytics

| Method | Route                  | Auth  | Description       |
| ------ | ---------------------- | ----- | ----------------- |
| `GET`  | `/api/admin/analytics` | Admin | System-wide stats |

**Response shape:**

```json
{
  "documents": {
    "total": 12,
    "byStatus": { "indexed": 10, "failed": 1, "pending": 1 }
  },
  "chunks": { "total": 3847 },
  "chat": {
    "totalSessions": 45,
    "totalMessages": 312,
    "avgMessagesPerSession": 6.93
  },
  "feedback": {
    "totalRatings": 87,
    "averageRating": 4.2,
    "distribution": { "5": 40, "4": 28 }
  },
  "users": { "total": 5, "byRole": { "admin": 1, "user": 4 } }
}
```

---

## 🖥️ Frontend SPA — `app.html`

A single HTML file with **5 views** — no page reloads, no framework needed.

### Page 1 — Login

- Email + password form → `POST /api/auth/login`
- Saves JWT to `localStorage`, auto-restores on refresh

### Page 2 — Dashboard _(Admin only)_

- Stat cards: Documents, Chunks, Sessions, Avg Rating
- Documents by status + Feedback distribution charts
- Data from `GET /api/admin/analytics`

### Page 3 — Documents _(Admin only)_

- Drag-and-drop or click-to-upload (`.pdf`, `.txt`, `.md`, `.jpg`, `.png`)
- Live table with status badges (pending → indexing → indexed)
- Delete documents with confirmation

### Page 4 — Search

- Instant semantic vector search — no LLM, very fast
- Shows similarity %, page number, document name, matching passage
- **"Ask AI about this"** → pre-fills chat with the passage

### Page 5 — Chat

- Session sidebar, create/switch conversations
- Real-time streaming — words appear as LLM generates them
- Source citations under every AI response
- Star rating (1–5) on each response — editable, golden glow on selection

---

## 🔐 Authentication Flow

```
Client                          Server
  │                               │
  │── POST /api/auth/login ──────►│
  │                               │── bcrypt.compare(password, hash)
  │                               │── jwt.sign({ id, role }, JWT_SECRET)
  │◄─ { token, user } ────────────│
  │                               │
  │── GET /api/chat/sessions ────►│
  │   Authorization: Bearer <jwt> │
  │                               │── isAuthenticated policy
  │                               │── jwt.verify(token, JWT_SECRET)
  │◄─ { sessions: [...] } ────────│
```

| Role    | Access                            |
| ------- | --------------------------------- |
| `user`  | Search, Chat, Feedback            |
| `admin` | Everything + Documents, Dashboard |

---

## 🗄️ Database Schema

### MongoDB Collections

| Collection           | Key Fields                                          |
| -------------------- | --------------------------------------------------- |
| `users`              | name, email, passwordHash, role, isActive           |
| `knowledgedocuments` | title, type, status, filePath, chunkCount           |
| `chatsessions`       | sessionId (uuid), userId, title                     |
| `chatmessages`       | sessionId, role (user\|assistant), content, sources |
| `feedbacks`          | messageId, userId, rating (1–5)                     |

### PostgreSQL Table — `document_chunks`

```sql
id          BIGSERIAL PRIMARY KEY
doc_id      VARCHAR(255)      -- links to KnowledgeDocument
doc_title   VARCHAR(500)
chunk_index INTEGER
page_number INTEGER
content     TEXT
embedding   vector(768)       -- nomic-embed-text output
metadata    JSONB
created_at  TIMESTAMP
```

---

## 🐛 Troubleshooting

| Problem                                     | Cause                       | Fix                                                              |
| ------------------------------------------- | --------------------------- | ---------------------------------------------------------------- |
| `ReadableStream is not defined`             | Node.js < v18               | `nvm use 20`                                                     |
| `401 Unauthorized` on all calls             | Stale token in localStorage | Run `localStorage.clear(); location.reload()` in browser console |
| Upload not saving file                      | Sails Skipper conflict      | Already fixed in `config/http.js`                                |
| `ECONNREFUSED` on embeddings                | Ollama not running          | `ollama serve`                                                   |
| Chat returns empty / errors                 | Model not pulled            | `ollama pull llama3.2 && ollama pull nomic-embed-text`           |
| `relation "document_chunks" does not exist` | DB not set up               | Run the SQL setup script above                                   |
| `uuid` fails                                | Node.js v16                 | Upgrade to Node.js v20                                           |

---

## 📦 NPM Scripts

| Script        | Command               | Description                             |
| ------------- | --------------------- | --------------------------------------- |
| `dev`         | `npm run dev`         | Copy assets + start Sails (development) |
| `start`       | `npm start`           | Production start                        |
| `copy-assets` | `npm run copy-assets` | Copy `assets/` → `.tmp/public/`         |
| `test`        | `npm test`            | Lint + custom tests                     |

---

## 🌍 Environment Variables

| Variable          | Required | Default                  | Description                   |
| ----------------- | -------- | ------------------------ | ----------------------------- |
| `MONGODB_URL`     | ✅       | —                        | MongoDB connection string     |
| `PGHOST`          | ✅       | `localhost`              | PostgreSQL host               |
| `PGPORT`          | ✅       | `5432`                   | PostgreSQL port               |
| `PGDATABASE`      | ✅       | —                        | PostgreSQL database name      |
| `PGUSER`          | ✅       | —                        | PostgreSQL username           |
| `PGPASSWORD`      | ✅       | —                        | PostgreSQL password           |
| `JWT_SECRET`      | ✅       | —                        | Secret key for signing JWTs   |
| `JWT_EXPIRES_IN`  | ❌       | `7d`                     | JWT expiry duration           |
| `OLLAMA_BASE_URL` | ❌       | `http://localhost:11434` | Ollama API URL                |
| `NODE_ENV`        | ❌       | `development`            | `development` or `production` |

---

## 👤 Author

**ztlab119** — [@ztlab119](https://github.com/ztlab119)

---

## 🙏 Acknowledgements

- [Sails.js](https://sailsjs.com) — Node.js MVC framework
- [Ollama](https://ollama.ai) — Run LLMs locally
- [LangChain JS](https://js.langchain.com) — LLM orchestration
- [pgvector](https://github.com/pgvector/pgvector) — Vector similarity search for PostgreSQL
- [nomic-embed-text](https://ollama.ai/library/nomic-embed-text) — Open-source embedding model

---

> 💡 **Tip**: Start with a small PDF (2–5 pages) to test the full pipeline end-to-end before uploading large documents.

### Links

- [Sails framework documentation](https://sailsjs.com/get-started)
- [Version notes / upgrading](https://sailsjs.com/documentation/upgrading)
- [Deployment tips](https://sailsjs.com/documentation/concepts/deployment)
- [Community support options](https://sailsjs.com/support)
- [Professional / enterprise options](https://sailsjs.com/enterprise)

### Version info

This app was originally generated on Thu May 28 2026 17:16:46 GMT+0530 (India Standard Time) using Sails v1.5.14.

<!-- Internally, Sails used [`sails-generate@2.0.13`](https://github.com/balderdashy/sails-generate/tree/v2.0.13/lib/core-generators/new). -->

<!--
Note:  Generators are usually run using the globally-installed `sails` CLI (command-line interface).  This CLI version is _environment-specific_ rather than app-specific, thus over time, as a project's dependencies are upgraded or the project is worked on by different developers on different computers using different versions of Node.js, the Sails dependency in its package.json file may differ from the globally-installed Sails CLI release it was originally generated with.  (Be sure to always check out the relevant [upgrading guides](https://sailsjs.com/upgrading) before upgrading the version of Sails used by your app.  If you're stuck, [get help here](https://sailsjs.com/support).)
-->
