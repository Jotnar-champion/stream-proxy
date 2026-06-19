# AI-Powered HR Recruitment Platform — Design Plan

## Problem Statement
Build a production-grade AI-powered HR Recruitment Platform that:
- Accepts applicant PDF resumes
- Parses, embeds, and stores them
- Runs a RAG pipeline against active job openings in ChromaDB
- Scores candidates via an LLM (Mistral/LLaMA) using a structured rubric
- Notifies relevant HR users in real-time via WebSocket + email when a match ≥ threshold

## Tech Stack
- **Backend:** FastAPI (Python), Apache Kafka, Redis, PostgreSQL, ChromaDB
- **Frontend:** Next.js (React, TypeScript)
- **AI/ML:** HuggingFace sentence-transformers, LangChain RAG, HuggingFace LLM inference
- **Notifications:** FastAPI WebSockets + Redis Pub/Sub + SendGrid
- **Auth:** JWT + RBAC (SuperAdmin, HR, Applicant)

---

## Deliverables

### Section 1 — HLD
- 1.1 System Architecture Diagram (text)
- 1.2 Component Breakdown (single responsibility per service)
- 1.3 End-to-End Data Flow
- 1.4 Kafka Topic Design (4 topics + partitions + DLQ)
- 1.5 RBAC Design (roles, permission matrix, JWT claims)
- 1.6 Caching Strategy (Redis — 5 cache patterns, cache-aside vs write-through)
- 1.7 Atomicity & Saga Pattern for resume processing

### Section 2 — LLD
- 2.1 PostgreSQL Schema (7 tables, full DDL)
- 2.2 ChromaDB Collection Design
- 2.3 RAG Pipeline Design (LangChain, prompt template, weighted scoring)
- 2.4 LLM Scoring Output Contract (JSON schema)
- 2.5 Notification Service Design (Kafka consumer, digest modes)
- 2.6 WebSocket Architecture for horizontal scale (Redis Pub/Sub fan-out)

### Section 3 — Critical Code Snippets (9 snippets)
- 3.1 Resume Upload Endpoint (FastAPI + idempotency + Kafka)
- 3.2 Kafka Consumer — Resume Ingestion Worker (PDFLoader + embed + atomic write + DLQ)
- 3.3 RAG + LLM Scoring Pipeline (LangChain + structured JSON output + Kafka publish)
- 3.4 RBAC Middleware (JWT + `require_role` dependency factory)
- 3.5 WebSocket Notification Endpoint (FastAPI + Redis Pub/Sub)
- 3.6 Notification Service Kafka Consumer (score threshold, SendGrid email)
- 3.7 Next.js `useNotifications` Hook (WS reconnect, unread count)
- 3.8 Redis Caching Decorator (async cache-aside, parameterized TTL)
- 3.9 Job Opening Ingestion into ChromaDB (upsert + Redis invalidation)

### Section 4 — Additional Requirements
- Docker Compose (all 8 services)
- `.env` structure
- Scalability notes (Kafka consumer groups, WS horizontal scale, LLM inference)
- Security notes (PDF sanitization, JWT refresh rotation, rate limiting)
- Observability (structured logging with correlation IDs across Kafka events)

---

## Todos (tracked in SQL)
1. `hld-architecture` — Write HLD 1.1–1.3 (architecture diagram, components, data flow)
2. `hld-kafka-rbac` — Write HLD 1.4–1.5 (Kafka topics, RBAC)
3. `hld-cache-saga` — Write HLD 1.6–1.7 (Redis caching strategy, Saga pattern)
4. `lld-schema` — Write LLD 2.1–2.2 (PostgreSQL DDL, ChromaDB design)
5. `lld-rag-notif` — Write LLD 2.3–2.6 (RAG pipeline, LLM contract, notification service, WebSocket)
6. `code-backend` — Write code snippets 3.1–3.4 (resume upload, Kafka consumer, RAG pipeline, RBAC)
7. `code-notif-frontend` — Write code snippets 3.5–3.9 (WebSocket, notification consumer, hooks, caching, job ingestion)
8. `extras` — Docker Compose, .env, scalability, security, observability notes





================================================================================================================================================================================






# AI-Powered HR Recruitment Platform — HLD + LLD + Code

---

# SECTION 1 — HIGH-LEVEL DESIGN (HLD)

---

## 1.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL CLIENTS                                   │
│  ┌──────────────────────┐              ┌──────────────────────┐                 │
│  │   Applicant Portal   │              │      HR Portal       │                 │
│  │  (Next.js / React)   │              │  (Next.js / React)   │                 │
│  └──────────┬───────────┘              └──────────┬───────────┘                 │
└─────────────┼─────────────────────────────────────┼───────────────────────────-─┘
              │ HTTPS                                │ HTTPS + WebSocket
              ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             API GATEWAY (Nginx)                                 │
│           Rate Limiting │ SSL Termination │ JWT Header Forwarding               │
└──────┬────────────────────────────────────────────────────┬─────────────────────┘
       │                                                    │
       ▼                                                    ▼
┌──────────────────────┐                    ┌──────────────────────────────────┐
│  FastAPI Service A   │                    │      FastAPI Service B           │
│  (Auth + Applicant)  │                    │   (HR Portal + WebSocket)        │
│  - POST /auth/*      │                    │   - GET  /jobs/*                 │
│  - POST /resume/     │                    │   - POST /jobs/ (HR post)        │
│    upload            │                    │   - GET  /matches/*              │
│  - GET  /matches/    │                    │   - WS   /ws/notifications/      │
│    my-results        │                    │     {hr_user_id}                 │
└────────┬─────────────┘                    └──────────────────┬───────────────┘
         │                                                     │
         │  publish: resume-uploaded                           │ subscribe: ws:hr:{id}
         ▼                                                     ▼
┌─────────────────────────────┐              ┌────────────────────────────────┐
│       Apache Kafka          │              │         Redis Cluster          │
│  Topics:                    │              │  - Pub/Sub channels            │
│  ├─ resume-uploaded         │              │  - Cache (match, jobs, JWT)    │
│  ├─ match-requested         │              │  - Unread counters             │
│  ├─ match-completed         │              │  - Idempotency keys            │
│  └─ notification-dispatch   │              │  - Digest notification buffers │
│  DLQ: *.dlq per topic       │              └────────────────────────────────┘
└──────┬──────────────────────┘
       │
       ├───────────────────────────────────────────────────────┐
       │                                                       │
       ▼                                                       ▼
┌──────────────────────────┐                    ┌─────────────────────────────┐
│  Resume Ingestion Worker │                    │  Notification Service       │
│  (Kafka Consumer Group:  │                    │  (Kafka Consumer Group:     │
│   resume-ingestion-cg)   │                    │   notification-cg)          │
│                          │                    │                             │
│  1. PDFLoader → text     │                    │  Consumes: match-completed  │
│  2. Chunk + Embed        │  publish:          │  - Score threshold check    │
│     (HuggingFace)        │  match-requested   │  - HR preference lookup     │
│  3. Write → ChromaDB     ├──────────────────► │  - Publish Redis Pub/Sub    │
│  4. Write → PostgreSQL   │                    │  - Send SendGrid email      │
│  (atomic saga)           │                    │  - Store notification in PG │
└──────────────────────────┘                    │  - Increment Redis counter  │
                                                └─────────────────────────────┘
       │
       ▼
┌──────────────────────────┐
│  LLM Scoring Worker      │
│  (Kafka Consumer Group:  │
│   llm-scoring-cg)        │
│                          │
│  Consumes: match-requested│
│  1. Load ChromaDB        │
│     retriever (top-K=5)  │
│  2. Build LangChain RAG  │
│     chain                │
│  3. LLM inference        │
│     (HuggingFace API)    │
│  4. Parse JSON output    │
│  5. Write match_results  │
│     → PostgreSQL         │
│  6. Publish              │
│     match-completed      │
└──────────────────────────┘

Data Stores:
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  PostgreSQL  │   │   ChromaDB   │   │     S3 /     │
│  - users     │   │  Collections │   │  Local FS    │
│  - hr_prof.  │   │  - job_idx   │   │  (PDF files) │
│  - job_open. │   │  - resume_idx│   └──────────────┘
│  - applicat. │   └──────────────┘
│  - match_res.│
│  - notifs    │
│  - hr_prefs  │
└──────────────┘
```

---

## 1.2 Component Breakdown

| Component | Responsibility |
|---|---|
| **API Gateway (Nginx)** | SSL termination, rate limiting, JWT header forwarding, route to FastAPI services |
| **FastAPI Service A — Auth + Applicant** | User registration/login (JWT issue), resume upload (idempotency + Kafka publish), applicant match result retrieval |
| **FastAPI Service B — HR + WebSocket** | HR job posting (PG + ChromaDB + cache invalidation), job listing, match result viewing, WebSocket notification endpoint |
| **Resume Ingestion Worker** | Consumes `resume-uploaded`; runs PDFLoader → chunk → HuggingFace embed → atomic write to ChromaDB + PostgreSQL; Saga rollback on failure; publishes `match-requested` |
| **LLM Scoring Worker** | Consumes `match-requested`; builds LangChain RAG chain; queries ChromaDB for top-K job matches; runs LLM inference; parses structured JSON; writes to `match_results`; publishes `match-completed` |
| **Notification Service** | Consumes `match-completed`; checks score threshold vs HR preference; dispatches WebSocket via Redis Pub/Sub + email via SendGrid; stores notification record; manages digest buffer |
| **ChromaDB** | Vector store for job opening embeddings (semantic retrieval) and optionally resume embeddings |
| **PostgreSQL** | Source of truth for users, job openings, applications, match results, notifications, HR preferences |
| **Redis** | Caching (match results, active jobs, JWT), WebSocket Pub/Sub channels, idempotency keys, unread counters, digest notification lists |
| **Apache Kafka** | Async event bus decoupling all pipeline stages; durable event log; DLQ for retry failures |
| **Applicant Portal (Next.js)** | Resume upload UI, match results dashboard, application status tracking |
| **HR Portal (Next.js)** | Job posting UI, matched applicant cards, real-time notification bell, applicant profile viewer |

---

## 1.3 End-to-End Data Flow

```
Step 1 — Resume Upload
  Applicant → POST /resume/upload (multipart/form-data, PDF)
  FastAPI Service A:
    a. Verify JWT (role=Applicant)
    b. Check idempotency key in Redis (key: idem:{sha256_of_file})
    c. If duplicate → return 200 (cached response)
    d. Save PDF to file storage (S3 or /uploads)
    e. INSERT application row (status=PENDING) into PostgreSQL
    f. SET idempotency key in Redis (TTL=24h)
    g. Publish to Kafka topic `resume-uploaded`:
       { application_id, applicant_user_id, resume_path, file_hash }

Step 2 — Resume Ingestion (Worker)
  Resume Ingestion Worker consumes `resume-uploaded`:
    a. Load PDF via PDFLoader (LangChain)
    b. Extract raw text
    c. Chunk text (RecursiveCharacterTextSplitter)
    d. Embed chunks via HuggingFace sentence-transformer
    e. BEGIN saga:
       - Write embeddings to ChromaDB collection `resume_index`
       - Write raw_text + metadata to PostgreSQL (UPDATE application)
       COMMIT or ROLLBACK:
       - If ChromaDB fails → skip PG update, delete uploaded file, push to DLQ
       - If PG fails → delete ChromaDB vectors, delete file, push to DLQ
    f. On success → publish to Kafka `match-requested`:
       { application_id, resume_id, chroma_doc_id }

Step 3 — RAG Matching + LLM Scoring (Worker)
  LLM Scoring Worker consumes `match-requested`:
    a. Load resume text from PostgreSQL
    b. Build LangChain RAG chain:
       - Retriever: ChromaDB `job_openings_index`, top-K=5, cosine similarity
       - Query: resume full text
    c. Retrieve top-5 job opening documents with metadata
    d. Construct prompt (structured template, see 2.3)
    e. Call HuggingFace LLM inference API (Mistral/LLaMA)
    f. Parse response as JSON array of rankings
    g. For each ranking:
       - INSERT into PostgreSQL `match_results`
       - SET Redis cache: key `match:{resume_id}:{job_id}` (TTL=24h)
    h. Publish to Kafka `match-completed`:
       { application_id, rankings: [{ job_id, score, hr_user_id }] }

Step 4 — Notification Dispatch
  Notification Service consumes `match-completed`:
    a. For each ranking in rankings:
       i.  Lookup HR user assigned to job_opening (from match_results)
       ii. Fetch HR notification preferences from PostgreSQL (with Redis cache)
       iii.If score >= hr.score_threshold:
           - INSERT notification row into PostgreSQL `notifications`
           - INCR Redis key `notif:unread:{hr_user_id}`
           - Check digest_mode:
             IMMEDIATE:
               - PUBLISH to Redis channel `ws:hr:{hr_user_id}` (JSON payload)
               - Send SendGrid email (async)
             HOURLY / DAILY:
               - LPUSH notification JSON to Redis list `notif:digest:{hr_user_id}`
               - APScheduler flush job sends batch email + WS burst

Step 5 — WebSocket Delivery to HR
  FastAPI Service B WebSocket handler for hr_user_id:
    a. HR client connects to WS /ws/notifications/{hr_user_id}
    b. Handler subscribes to Redis Pub/Sub channel `ws:hr:{hr_user_id}`
    c. On Redis message received → forward JSON payload to WS client
    d. On WS disconnect → unsubscribe from Redis channel

Step 6 — HR Reviews Match
  HR Portal → GET /matches/{job_id}
    a. Check Redis cache: `match:{resume_id}:{job_id}`
    b. Cache hit → return cached result
    c. Cache miss → query PostgreSQL `match_results` → cache + return
```

---

## 1.4 Kafka Topic Design

### Topics

| Topic | Purpose | Producers | Consumers |
|---|---|---|---|
| `resume-uploaded` | Triggers ingestion pipeline | FastAPI Service A | Resume Ingestion Worker |
| `match-requested` | Triggers RAG + LLM scoring | Resume Ingestion Worker | LLM Scoring Worker |
| `match-completed` | Triggers notification dispatch | LLM Scoring Worker | Notification Service |
| `notification-dispatch` | Fan-out to HR channels | Notification Service | (internal, direct to Redis) |

### DLQ (Dead Letter Queue)

Each topic has a companion DLQ:
- `resume-uploaded.dlq`
- `match-requested.dlq`
- `match-completed.dlq`

A failed message is published to the `.dlq` topic after **3 retries with exponential backoff** (1s → 2s → 4s). A separate DLQ monitor service alerts ops and can replay messages.

### Partition Strategy

```
resume-uploaded:
  partitions: 6
  key: applicant_user_id  (ensures ordered processing per applicant)
  replication: 3

match-requested:
  partitions: 6
  key: application_id
  replication: 3

match-completed:
  partitions: 6
  key: application_id
  replication: 3

notification-dispatch:
  partitions: 12          (higher throughput — many HR users)
  key: hr_user_id         (ensures ordered notification per HR)
  replication: 3
```

### Consumer Group Design

```
resume-ingestion-cg:
  topic:     resume-uploaded
  instances: 3–6 workers (scale with partitions)
  isolation: read_committed (for exactly-once semantics)

llm-scoring-cg:
  topic:     match-requested
  instances: 2–4 workers (LLM inference is compute-heavy)
  isolation: read_committed

notification-cg:
  topic:     match-completed
  instances: 3–6 workers
  isolation: read_committed
```

---

## 1.5 RBAC Design

### Roles

| Role | Description |
|---|---|
| `SuperAdmin` | Platform administrator; manages users, job openings globally |
| `HR` | Posts job openings; receives match notifications for their postings only |
| `Applicant` | Uploads resume; views their own match results |

### Permission Matrix

| Endpoint | SuperAdmin | HR | Applicant |
|---|---|---|---|
| POST `/auth/register` | ✓ | ✓ | ✓ |
| POST `/auth/login` | ✓ | ✓ | ✓ |
| POST `/resume/upload` | ✗ | ✗ | ✓ |
| GET `/matches/my-results` | ✗ | ✗ | ✓ |
| POST `/jobs/` | ✓ | ✓ | ✗ |
| GET `/jobs/` | ✓ | ✓ | ✓ |
| GET `/jobs/{id}` | ✓ | ✓ | ✓ |
| PUT `/jobs/{id}` | ✓ | HR owner only | ✗ |
| DELETE `/jobs/{id}` | ✓ | HR owner only | ✗ |
| GET `/matches/{job_id}` | ✓ | HR owner only | ✗ |
| GET `/admin/users` | ✓ | ✗ | ✗ |
| WS `/ws/notifications/{hr_id}` | ✓ | self only | ✗ |
| GET `/notifications/` | ✓ | self only | ✗ |
| PUT `/notifications/preferences` | ✓ | self only | ✗ |

### JWT Claims Structure

```json
{
  "sub": "user_uuid",
  "email": "user@example.com",
  "role": "HR",
  "assigned_job_ids": ["job-uuid-1", "job-uuid-2"],
  "iat": 1718000000,
  "exp": 1718003600,
  "jti": "unique-token-id"
}
```

- `assigned_job_ids` is populated for HR role — contains job IDs the HR user owns.
- `jti` (JWT ID) enables token revocation via Redis blocklist.
- Access token TTL: 1 hour. Refresh token TTL: 7 days (stored in Redis, rotated on use).

---

## 1.6 Caching Strategy (Redis)

### Cache Patterns Summary

| Cache Key | Pattern | TTL | Invalidation |
|---|---|---|---|
| `match:{resume_id}:{job_id}` | Cache-aside | 24 h | Never (immutable result) |
| `jobs:active` | Write-through | ∞ | On HR POST/PUT/DELETE job |
| `chroma:topk:{resume_hash}` | Cache-aside | 6 h | On new job posting |
| `notif:unread:{hr_user_id}` | Write-through | ∞ | On notification read |
| `jwt:blocklist:{jti}` | Write-through | = token TTL | On logout/revoke |

### Detailed Policies

**`match:{resume_id}:{job_id}` — Cache-aside (read-heavy, write-once)**
```
Read:  GET key → miss → query PG → SET key (TTL 24h) → return
Write: LLM worker writes result → SET key after PG insert
```

**`jobs:active` — Write-through (frequently queried list)**
```
Read:  GET key → miss → query PG for status=ACTIVE → SET key → return
Write: HR posts/updates/deletes job → update PG → DEL key (invalidate)
       Next read rebuilds cache
```

**`chroma:topk:{resume_hash}` — Cache-aside (duplicate resume queries)**
```
Read:  GET key → miss → call ChromaDB retriever → SET key (TTL 6h) → return
Write: On new job posting → DEL all chroma:topk:* keys (since job set changed)
```

**`notif:unread:{hr_user_id}` — Write-through (counter)**
```
Increment: Notification Service → INCR key after inserting notification
Decrement: HR reads notification(s) → DECR or reset via GET /notifications/mark-read
```

**`jwt:blocklist:{jti}` — Write-through (token revocation)**
```
On logout: SET key "revoked" (TTL = remaining token lifetime)
Middleware: for every request, check if jti is in blocklist → reject if present
```

---

## 1.7 Atomicity & Saga Pattern

### Resume Ingestion Saga

```
SAGA: ResumeIngestionSaga
Participants: FileStorage, ChromaDB, PostgreSQL

Step 1: Save PDF to file storage
  Compensate: delete file from storage

Step 2: Parse + embed text, write to ChromaDB
  Compensate: delete ChromaDB vectors by doc_id

Step 3: Update PostgreSQL application row (raw_text, status=INGESTED)
  Compensate: revert application status to PENDING

Failure Handling:
  If Step 2 fails → execute Compensate(Step 1) → push to DLQ
  If Step 3 fails → execute Compensate(Step 2) → Compensate(Step 1) → push to DLQ

Retry Policy:
  Max retries: 3
  Backoff: 1s, 2s, 4s (exponential)
  After 3 failures: publish to resume-uploaded.dlq with failure_reason
```

### Idempotency

```
Key: idem:{sha256(file_content)}:{applicant_user_id}
TTL: 24 hours

On upload:
  SETNX idem_key "processing"
  If key existed: return 200 (already processing or processed)
  After saga success: SET idem_key "completed"
  After saga failure: DEL idem_key (allow retry by user)
```

### LLM Scoring Retry

```
Retry policy on LLM API failure:
  Max retries: 3
  Backoff: 2s, 4s, 8s
  On HuggingFace API timeout: retry with reduced context
  After 3 failures: push to match-requested.dlq
  DLQ alert: trigger ops PagerDuty/Slack webhook
```

---

---

# SECTION 2 — LOW-LEVEL DESIGN (LLD)

---

## 2.1 PostgreSQL Schema (Full DDL)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- for text search

-- ── users ──────────────────────────────────────────────────────────────────
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255)  NOT NULL,
    email           VARCHAR(255)  UNIQUE NOT NULL,
    hashed_password TEXT          NOT NULL,
    role            VARCHAR(20)   NOT NULL CHECK (role IN ('SuperAdmin', 'HR', 'Applicant')),
    is_active       BOOLEAN       NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role  ON users(role);

-- ── hr_profiles ────────────────────────────────────────────────────────────
CREATE TABLE hr_profiles (
    user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    company_name    VARCHAR(255) NOT NULL,
    department      VARCHAR(255),
    linkedin_url    TEXT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- ── job_openings ───────────────────────────────────────────────────────────
CREATE TABLE job_openings (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    hr_user_id       UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title            VARCHAR(255) NOT NULL,
    company          VARCHAR(255) NOT NULL,
    jd_text          TEXT         NOT NULL,
    required_skills  TEXT[]       NOT NULL DEFAULT '{}',
    experience_years SMALLINT     NOT NULL DEFAULT 0,
    status           VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                                  CHECK (status IN ('ACTIVE', 'PAUSED', 'CLOSED')),
    chroma_doc_id    TEXT,        -- ID in ChromaDB collection
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_job_openings_hr_user_id ON job_openings(hr_user_id);
CREATE INDEX idx_job_openings_status     ON job_openings(status);
CREATE INDEX idx_job_openings_skills     ON job_openings USING GIN(required_skills);

-- ── applications ───────────────────────────────────────────────────────────
CREATE TABLE applications (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    applicant_user_id UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    resume_path       TEXT         NOT NULL,   -- S3 URI or local path
    raw_text          TEXT,                    -- populated after ingestion
    file_hash         VARCHAR(64)  NOT NULL,   -- SHA-256 of uploaded PDF
    status            VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                                   CHECK (status IN ('PENDING','INGESTED','SCORED','FAILED')),
    uploaded_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_applications_applicant   ON applications(applicant_user_id);
CREATE INDEX idx_applications_file_hash   ON applications(file_hash);
CREATE INDEX idx_applications_status      ON applications(status);

-- ── match_results ──────────────────────────────────────────────────────────
CREATE TABLE match_results (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    application_id  UUID         NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    job_opening_id  UUID         NOT NULL REFERENCES job_openings(id) ON DELETE CASCADE,
    score           SMALLINT     NOT NULL CHECK (score BETWEEN 0 AND 100),
    strengths       TEXT[]       NOT NULL DEFAULT '{}',
    gaps            TEXT[]       NOT NULL DEFAULT '{}',
    persona_fit     TEXT,
    recommendation  TEXT,
    reasoning       TEXT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (application_id, job_opening_id)
);
CREATE INDEX idx_match_results_application ON match_results(application_id);
CREATE INDEX idx_match_results_job         ON match_results(job_opening_id);
CREATE INDEX idx_match_results_score       ON match_results(score DESC);

-- ── notifications ──────────────────────────────────────────────────────────
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    hr_user_id      UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    application_id  UUID        NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    job_opening_id  UUID        NOT NULL REFERENCES job_openings(id) ON DELETE CASCADE,
    score           SMALLINT    NOT NULL,
    is_read         BOOLEAN     NOT NULL DEFAULT FALSE,
    dispatched_via  TEXT[],     -- ['websocket', 'email']
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_notifications_hr_user    ON notifications(hr_user_id);
CREATE INDEX idx_notifications_is_read    ON notifications(hr_user_id, is_read);
CREATE INDEX idx_notifications_created    ON notifications(created_at DESC);

-- ── hr_notification_preferences ────────────────────────────────────────────
CREATE TABLE hr_notification_preferences (
    hr_user_id      UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    channel_email   BOOLEAN     NOT NULL DEFAULT TRUE,
    channel_inapp   BOOLEAN     NOT NULL DEFAULT TRUE,
    score_threshold SMALLINT    NOT NULL DEFAULT 75
                                CHECK (score_threshold BETWEEN 0 AND 100),
    digest_mode     VARCHAR(10) NOT NULL DEFAULT 'immediate'
                                CHECK (digest_mode IN ('immediate', 'hourly', 'daily')),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── refresh_tokens ─────────────────────────────────────────────────────────
CREATE TABLE refresh_tokens (
    jti         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id     UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT        NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked     BOOLEAN     NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);

-- ── Triggers: auto-update updated_at ───────────────────────────────────────
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
  BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_job_openings_updated_at
  BEFORE UPDATE ON job_openings FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_applications_updated_at
  BEFORE UPDATE ON applications FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## 2.2 ChromaDB Collection Design

### Collection: `job_openings_index`

```python
# Document format per job opening
Document(
    page_content = f"""
        Job Title: {job.title}
        Company: {job.company}
        Job Description: {job.jd_text}
        Required Skills: {', '.join(job.required_skills)}
        Experience Required: {job.experience_years} years
    """,
    metadata = {
        "job_id":           str(job.id),           # UUID
        "hr_user_id":       str(job.hr_user_id),   # UUID
        "company":          job.company,
        "title":            job.title,
        "required_skills":  ",".join(job.required_skills),
        "experience_years": job.experience_years,
        "status":           job.status,            # ACTIVE / PAUSED
    }
)

# Embedding: HuggingFace sentence-transformers/all-MiniLM-L6-v2
# Similarity: cosine
# ID: job.id (UUID as string) — enables upsert/delete by job ID
```

### Collection: `resume_index` (optional — future applicant-side search)

```python
Document(
    page_content = resume_text_chunk,   # chunked resume text
    metadata = {
        "application_id": str(application.id),
        "applicant_user_id": str(application.applicant_user_id),
        "chunk_index": int,
    }
)
```

### ChromaDB Client Setup

```python
import chromadb
from chromadb.config import Settings

client = chromadb.HttpClient(
    host=settings.CHROMADB_HOST,
    port=settings.CHROMADB_PORT,
    settings=Settings(anonymized_telemetry=False)
)

job_collection = client.get_or_create_collection(
    name="job_openings_index",
    metadata={"hnsw:space": "cosine"}
)
```

---

## 2.3 RAG Pipeline Design (LangChain)

### Embedding Setup

```python
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_chroma import Chroma

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True}
)

vectorstore = Chroma(
    client=chroma_client,
    collection_name="job_openings_index",
    embedding_function=embeddings
)

retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5}
)
```

### Prompt Template

```python
from langchain_core.prompts import PromptTemplate

SCORING_PROMPT = PromptTemplate.from_template("""
You are an expert HR analyst. Analyze the candidate's resume against the provided job openings.

## Candidate Resume:
{resume_text}

## Retrieved Job Openings:
{job_openings}

## Scoring Rubric (weights):
- Skills Match:       40% — exact and related technical/domain skills
- Work Experience:    25% — years, relevance, seniority alignment
- Projects:           20% — project complexity, relevant tech, outcomes
- Achievements:       15% — awards, publications, measurable impact

## Instructions:
For EACH job opening, produce a score from 0–100 based on the rubric.
Be objective. Penalize gaps. Reward exceptional alignment.

## Output Format (strict JSON, no extra text):
{{
  "rankings": [
    {{
      "job_id": "<uuid>",
      "job_title": "<title>",
      "company": "<company>",
      "score": <0-100>,
      "strengths": ["<strength1>", "<strength2>"],
      "gaps": ["<gap1>", "<gap2>"],
      "persona_fit": "<one sentence about culture/team fit>",
      "recommendation": "<Highly recommended|Recommended|Borderline|Not recommended>"
    }}
  ]
}}

Output ONLY valid JSON. No markdown, no preamble.
""")
```

### RAG Chain Construction

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import JsonOutputParser
from langchain_huggingface import HuggingFaceEndpoint

llm = HuggingFaceEndpoint(
    repo_id="mistralai/Mistral-7B-Instruct-v0.2",
    huggingfacehub_api_token=settings.HUGGINGFACE_API_TOKEN,
    max_new_tokens=2048,
    temperature=0.1,
)

def format_jobs(docs) -> str:
    return "\n\n---\n\n".join([
        f"Job ID: {d.metadata['job_id']}\n"
        f"Title: {d.metadata['title']}\n"
        f"Company: {d.metadata['company']}\n"
        f"JD: {d.page_content}\n"
        f"Skills: {d.metadata['required_skills']}\n"
        f"Experience: {d.metadata['experience_years']} years"
        for d in docs
    ])

rag_chain = (
    {
        "job_openings": retriever | format_jobs,
        "resume_text": RunnablePassthrough()
    }
    | SCORING_PROMPT
    | llm
    | JsonOutputParser()
)
```

---

## 2.4 LLM Scoring Output Contract

```json
{
  "rankings": [
    {
      "job_id": "550e8400-e29b-41d4-a716-446655440001",
      "job_title": "Computer Scientist",
      "company": "Adobe",
      "score": 87,
      "strengths": [
        "Python expertise with NumPy/PyTorch",
        "ML research experience (3 published papers)",
        "Computer vision background matches team focus"
      ],
      "gaps": [
        "No Kubernetes experience (required)",
        "3 years experience vs 5 required"
      ],
      "persona_fit": "Strong independent researcher; suits Adobe's ML team culture and publication cadence",
      "recommendation": "Highly recommended"
    },
    {
      "job_id": "550e8400-e29b-41d4-a716-446655440002",
      "job_title": "Backend Engineer",
      "company": "Stripe",
      "score": 62,
      "strengths": [
        "REST API experience",
        "PostgreSQL proficiency"
      ],
      "gaps": [
        "No Go experience (primary language)",
        "No payments domain knowledge",
        "Limited distributed systems exposure"
      ],
      "persona_fit": "Solid generalist; may lack Stripe's high-reliability engineering culture fit",
      "recommendation": "Borderline"
    }
  ]
}
```

**Validation rules:**
- `score` must be integer 0–100
- `strengths` and `gaps` must be non-empty arrays
- `recommendation` must be one of: `Highly recommended`, `Recommended`, `Borderline`, `Not recommended`
- `job_id` must match a known UUID from retrieved documents (validate before DB insert)

---

## 2.5 Notification Service Design

```
Kafka Consumer: notification-cg
Topic: match-completed
Event payload: {
  application_id, applicant_user_id, applicant_name, applicant_email,
  rankings: [{ job_id, job_title, company, score, hr_user_id }]
}

For each ranking:
  1. Fetch HR notification preferences (Redis cache → PG fallback)
     Cache key: prefs:{hr_user_id}, TTL: 5 min

  2. if score < hr.score_threshold: SKIP

  3. INSERT into notifications table

  4. INCR Redis key notif:unread:{hr_user_id}

  5. Based on hr.digest_mode:
     IMMEDIATE:
       a. Build notification payload:
          {
            type: "new_match",
            notification_id: uuid,
            applicant_name, score, job_title, company,
            match_url: "/matches/{job_id}?app={application_id}"
          }
       b. PUBLISH to Redis Pub/Sub channel: ws:hr:{hr_user_id}
       c. If hr.channel_email:
            send_email_via_sendgrid(hr_email, applicant_summary, score)

     HOURLY / DAILY:
       a. RPUSH to Redis list: notif:digest:{hr_user_id}
       b. APScheduler job (runs hourly/daily):
            notifications = LRANGE notif:digest:{hr_user_id} 0 -1
            DEL notif:digest:{hr_user_id}
            Send digest email with all buffered notifications
            PUBLISH burst of WS notifications
```

### Digest Scheduler (APScheduler)

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()

@scheduler.scheduled_job('interval', hours=1, id='hourly_digest')
async def flush_hourly_digests():
    hr_keys = await redis.keys("notif:digest:*")
    for key in hr_keys:
        hr_user_id = key.split(":")[-1]
        prefs = await get_hr_preferences(hr_user_id)
        if prefs.digest_mode == "hourly":
            await flush_digest(hr_user_id)

@scheduler.scheduled_job('cron', hour=8, id='daily_digest')
async def flush_daily_digests():
    # similar, filter for daily mode
    ...
```

---

## 2.6 WebSocket Architecture for Scale

```
Problem: Multiple FastAPI instances — a WebSocket client connects to instance A,
         but Notification Service runs on instance B.
Solution: Redis Pub/Sub as the message bus between all instances.

Architecture:

  [Notification Service]
       │
       └── PUBLISH redis:// ws:hr:{hr_user_id}  ← notification JSON
                              │
               ┌──────────────┴───────────────┐
               │                              │
  [FastAPI Instance A]              [FastAPI Instance B]
  WS Handler: hr_user_123           WS Handler: hr_user_456
       │                                       │
       SUBSCRIBE ws:hr:hr_user_123             SUBSCRIBE ws:hr:hr_user_456
       │                                       │
  [HR Client 123]                        [HR Client 456]
  (WebSocket connected)                  (WebSocket connected)

Redis fan-out ensures delivery regardless of which FastAPI instance holds the WS.
If HR is not connected: notification is stored in PG + unread counter incremented.
On next HR login: load unread notifications from PG.
```

---

---

# SECTION 3 — CRITICAL CODE SNIPPETS

---

## 3.1 Resume Upload Endpoint (FastAPI)

```python
# app/api/v1/resume.py
import hashlib
import uuid
from fastapi import APIRouter, Depends, File, HTTPException, UploadFile, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.deps import get_db, get_current_user, get_redis, get_kafka_producer
from app.core.rbac import require_role
from app.models.user import User
from app.schemas.application import ApplicationResponse
from app.services.storage import save_upload
from app.repositories.application import ApplicationRepository

router = APIRouter(prefix="/resume", tags=["resume"])

MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB


@router.post(
    "/upload",
    response_model=ApplicationResponse,
    status_code=status.HTTP_202_ACCEPTED,
    dependencies=[Depends(require_role(["Applicant"]))],
)
async def upload_resume(
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
    redis=Depends(get_redis),
    producer=Depends(get_kafka_producer),
):
    # ── Validate file type & size ────────────────────────────────────────────
    if file.content_type != "application/pdf":
        raise HTTPException(status.HTTP_415_UNSUPPORTED_MEDIA_TYPE, "Only PDF files accepted")

    content = await file.read()
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(status.HTTP_413_REQUEST_ENTITY_TOO_LARGE, "File exceeds 10 MB limit")

    # ── Sanitize filename ────────────────────────────────────────────────────
    safe_filename = f"{uuid.uuid4()}.pdf"

    # ── Idempotency check ────────────────────────────────────────────────────
    file_hash = hashlib.sha256(content).hexdigest()
    idem_key = f"idem:{file_hash}:{current_user.id}"

    existing = await redis.get(idem_key)
    if existing:
        # Return existing application record
        app_repo = ApplicationRepository(db)
        application = await app_repo.get_by_file_hash_and_user(file_hash, current_user.id)
        if application:
            return ApplicationResponse.from_orm(application)

    # ── Mark idempotency key as processing ──────────────────────────────────
    await redis.set(idem_key, "processing", ex=86400)  # 24h TTL

    try:
        # ── Save file to storage ─────────────────────────────────────────────
        resume_path = await save_upload(safe_filename, content)

        # ── Create application record in PostgreSQL ──────────────────────────
        app_repo = ApplicationRepository(db)
        application = await app_repo.create(
            applicant_user_id=current_user.id,
            resume_path=resume_path,
            file_hash=file_hash,
        )

        # ── Publish to Kafka ─────────────────────────────────────────────────
        event = {
            "application_id": str(application.id),
            "applicant_user_id": str(current_user.id),
            "resume_path": resume_path,
            "file_hash": file_hash,
            "correlation_id": str(uuid.uuid4()),  # for distributed tracing
        }
        await producer.send_and_wait(
            topic="resume-uploaded",
            key=str(current_user.id).encode(),
            value=event,
        )

        # ── Mark idempotency key as completed ────────────────────────────────
        await redis.set(idem_key, "completed", ex=86400)

        return ApplicationResponse.from_orm(application)

    except Exception as exc:
        # Rollback idempotency key on failure so user can retry
        await redis.delete(idem_key)
        raise HTTPException(status.HTTP_500_INTERNAL_SERVER_ERROR, str(exc)) from exc
```

---

## 3.2 Kafka Consumer — Resume Ingestion Worker

```python
# workers/resume_ingestion_worker.py
import asyncio
import json
import logging
import uuid
from typing import Any

from aiokafka import AIOKafkaConsumer, AIOKafkaProducer
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_huggingface import HuggingFaceEmbeddings

from app.core.config import settings
from app.db.session import async_session_factory
from app.repositories.application import ApplicationRepository
from app.services.chroma import get_job_collection, get_resume_collection

logger = logging.getLogger(__name__)

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    encode_kwargs={"normalize_embeddings": True}
)

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)


async def process_resume(event: dict[str, Any], producer: AIOKafkaProducer) -> None:
    application_id = event["application_id"]
    resume_path    = event["resume_path"]
    correlation_id = event.get("correlation_id", str(uuid.uuid4()))

    log = logger.bind(application_id=application_id, correlation_id=correlation_id)
    log.info("Starting resume ingestion")

    chroma_doc_ids: list[str] = []

    async with async_session_factory() as db:
        app_repo = ApplicationRepository(db)

        try:
            # ── Step 1: PDF → text ───────────────────────────────────────────
            loader = PyPDFLoader(resume_path)
            pages  = loader.load()
            full_text = " ".join(p.page_content for p in pages)

            # ── Step 2: Chunk + embed ────────────────────────────────────────
            chunks = splitter.split_text(full_text)
            vectors = embeddings.embed_documents(chunks)

            # ── Step 3: Write to ChromaDB ────────────────────────────────────
            resume_collection = get_resume_collection()
            chroma_doc_ids = [f"{application_id}_chunk_{i}" for i in range(len(chunks))]
            resume_collection.add(
                ids=chroma_doc_ids,
                documents=chunks,
                embeddings=vectors,
                metadatas=[
                    {
                        "application_id": application_id,
                        "chunk_index": i,
                        "correlation_id": correlation_id,
                    }
                    for i in range(len(chunks))
                ],
            )

            # ── Step 4: Update PostgreSQL ────────────────────────────────────
            await app_repo.update_ingested(
                application_id=application_id,
                raw_text=full_text,
                status="INGESTED",
            )
            await db.commit()

            # ── Step 5: Publish match-requested ─────────────────────────────
            await producer.send_and_wait(
                topic="match-requested",
                key=application_id.encode(),
                value={
                    "application_id": application_id,
                    "applicant_user_id": event["applicant_user_id"],
                    "correlation_id": correlation_id,
                },
            )
            log.info("Ingestion complete, match-requested published")

        except Exception as exc:
            # ── Saga rollback ────────────────────────────────────────────────
            log.error("Ingestion failed, rolling back", error=str(exc))

            # Rollback ChromaDB
            if chroma_doc_ids:
                try:
                    resume_collection.delete(ids=chroma_doc_ids)
                except Exception:
                    log.warning("ChromaDB rollback failed — manual cleanup needed")

            # Revert PostgreSQL status
            await app_repo.update_status(application_id, "FAILED")
            await db.commit()

            raise  # bubble up for retry logic


async def main() -> None:
    producer = AIOKafkaProducer(
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        value_serializer=lambda v: json.dumps(v).encode(),
    )
    consumer = AIOKafkaConsumer(
        "resume-uploaded",
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        group_id="resume-ingestion-cg",
        value_deserializer=lambda v: json.loads(v.decode()),
        enable_auto_commit=False,
        isolation_level="read_committed",
    )
    dlq_producer = AIOKafkaProducer(
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        value_serializer=lambda v: json.dumps(v).encode(),
    )

    await asyncio.gather(producer.start(), consumer.start(), dlq_producer.start())

    try:
        async for msg in consumer:
            event = msg.value
            retry_count = event.get("_retry_count", 0)

            try:
                await process_resume(event, producer)
                await consumer.commit()

            except Exception as exc:
                if retry_count < 3:
                    # Exponential backoff retry
                    await asyncio.sleep(2 ** retry_count)
                    event["_retry_count"] = retry_count + 1
                    await producer.send_and_wait(
                        "resume-uploaded",
                        key=msg.key,
                        value=event,
                    )
                    logger.warning("Retry %d for %s", retry_count + 1, event["application_id"])
                else:
                    # Push to DLQ
                    event["_failure_reason"] = str(exc)
                    await dlq_producer.send_and_wait("resume-uploaded.dlq", value=event)
                    logger.error("Sent to DLQ: %s", event["application_id"])
                await consumer.commit()
    finally:
        await asyncio.gather(producer.stop(), consumer.stop(), dlq_producer.stop())


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 3.3 RAG + LLM Scoring Pipeline (LangChain)

```python
# workers/llm_scoring_worker.py
import asyncio
import json
import logging
import uuid

from aiokafka import AIOKafkaConsumer, AIOKafkaProducer
from langchain_chroma import Chroma
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_huggingface import HuggingFaceEmbeddings, HuggingFaceEndpoint

from app.core.config import settings
from app.db.session import async_session_factory
from app.repositories.application import ApplicationRepository
from app.repositories.match_result import MatchResultRepository
from app.services.chroma import get_chroma_client

logger = logging.getLogger(__name__)

# ── Embedding + Vectorstore ───────────────────────────────────────────────────
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    encode_kwargs={"normalize_embeddings": True},
)

vectorstore = Chroma(
    client=get_chroma_client(),
    collection_name="job_openings_index",
    embedding_function=embeddings,
)

retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5, "filter": {"status": "ACTIVE"}},
)

# ── LLM ──────────────────────────────────────────────────────────────────────
llm = HuggingFaceEndpoint(
    repo_id="mistralai/Mistral-7B-Instruct-v0.2",
    huggingfacehub_api_token=settings.HUGGINGFACE_API_TOKEN,
    max_new_tokens=2048,
    temperature=0.1,
)

SCORING_PROMPT = PromptTemplate.from_template("""
You are an expert HR analyst. Analyze the candidate resume against the job openings below.

## Candidate Resume:
{resume_text}

## Retrieved Job Openings:
{job_openings}

## Scoring Rubric:
- Skills Match: 40%
- Work Experience: 25%
- Projects: 20%
- Achievements: 15%

Output ONLY valid JSON matching this schema exactly:
{{
  "rankings": [
    {{
      "job_id": "<uuid>",
      "job_title": "<string>",
      "company": "<string>",
      "score": <int 0-100>,
      "strengths": ["<string>"],
      "gaps": ["<string>"],
      "persona_fit": "<string>",
      "recommendation": "<Highly recommended|Recommended|Borderline|Not recommended>"
    }}
  ]
}}
""")


def format_jobs(docs) -> str:
    return "\n\n---\n\n".join([
        f"Job ID: {d.metadata['job_id']}\n"
        f"Title: {d.metadata['title']}\n"
        f"Company: {d.metadata['company']}\n"
        f"JD: {d.page_content}\n"
        f"Skills: {d.metadata['required_skills']}\n"
        f"Experience Required: {d.metadata['experience_years']} years"
        for d in docs
    ])


rag_chain = (
    {
        "job_openings": retriever | format_jobs,
        "resume_text": RunnablePassthrough(),
    }
    | SCORING_PROMPT
    | llm
    | JsonOutputParser()
)


async def process_scoring(event: dict, producer: AIOKafkaProducer, redis) -> None:
    application_id = event["application_id"]
    correlation_id = event.get("correlation_id", str(uuid.uuid4()))

    log = logger.bind(application_id=application_id, correlation_id=correlation_id)
    log.info("Starting LLM scoring")

    async with async_session_factory() as db:
        app_repo    = ApplicationRepository(db)
        result_repo = MatchResultRepository(db)

        application = await app_repo.get_by_id(application_id)
        if not application or not application.raw_text:
            raise ValueError(f"No raw_text for application {application_id}")

        # Check Redis cache first (duplicate resume)
        cache_key = f"scoring:in_progress:{application_id}"
        if await redis.get(cache_key):
            log.info("Scoring already in progress, skipping")
            return
        await redis.setex(cache_key, 3600, "1")

        try:
            # ── RAG + LLM ────────────────────────────────────────────────────
            result = await asyncio.get_event_loop().run_in_executor(
                None, rag_chain.invoke, application.raw_text
            )

            rankings = result.get("rankings", [])

            # ── Validate + persist ───────────────────────────────────────────
            persisted_rankings = []
            for r in rankings:
                if not (0 <= r["score"] <= 100):
                    log.warning("Invalid score %s for job %s", r["score"], r["job_id"])
                    continue

                await result_repo.create(
                    application_id=application_id,
                    job_opening_id=r["job_id"],
                    score=r["score"],
                    strengths=r.get("strengths", []),
                    gaps=r.get("gaps", []),
                    persona_fit=r.get("persona_fit"),
                    recommendation=r.get("recommendation"),
                )

                # Cache match result
                match_cache_key = f"match:{application_id}:{r['job_id']}"
                await redis.setex(match_cache_key, 86400, json.dumps(r))

                persisted_rankings.append({
                    "job_id":     r["job_id"],
                    "job_title":  r["job_title"],
                    "company":    r["company"],
                    "score":      r["score"],
                })

            await app_repo.update_status(application_id, "SCORED")
            await db.commit()

            # ── Publish match-completed ──────────────────────────────────────
            await producer.send_and_wait(
                topic="match-completed",
                key=application_id.encode(),
                value={
                    "application_id":    application_id,
                    "applicant_user_id": event["applicant_user_id"],
                    "rankings":          persisted_rankings,
                    "correlation_id":    correlation_id,
                },
            )
            log.info("Scoring complete, match-completed published")

        finally:
            await redis.delete(cache_key)


async def main() -> None:
    import aioredis
    redis = await aioredis.from_url(settings.REDIS_URL)

    producer = AIOKafkaProducer(
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        value_serializer=lambda v: json.dumps(v).encode(),
    )
    consumer = AIOKafkaConsumer(
        "match-requested",
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        group_id="llm-scoring-cg",
        value_deserializer=lambda v: json.loads(v.decode()),
        enable_auto_commit=False,
    )
    dlq_producer = AIOKafkaProducer(
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        value_serializer=lambda v: json.dumps(v).encode(),
    )

    await asyncio.gather(producer.start(), consumer.start(), dlq_producer.start())

    try:
        async for msg in consumer:
            event = msg.value
            retry_count = event.get("_retry_count", 0)
            try:
                await process_scoring(event, producer, redis)
                await consumer.commit()
            except Exception as exc:
                if retry_count < 3:
                    await asyncio.sleep(2 ** (retry_count + 1))
                    event["_retry_count"] = retry_count + 1
                    await producer.send_and_wait("match-requested", key=msg.key, value=event)
                else:
                    event["_failure_reason"] = str(exc)
                    await dlq_producer.send_and_wait("match-requested.dlq", value=event)
                    logger.error("DLQ: scoring failed for %s", event["application_id"])
                await consumer.commit()
    finally:
        await asyncio.gather(producer.stop(), consumer.stop(), dlq_producer.stop())
        await redis.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 3.4 RBAC Middleware (FastAPI)

```python
# app/core/rbac.py
from functools import lru_cache
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

from app.core.config import settings
from app.core.deps import get_redis

bearer_scheme = HTTPBearer()


class CurrentUser:
    def __init__(self, user_id: str, email: str, role: str, assigned_job_ids: list[str], jti: str):
        self.user_id         = user_id
        self.email           = email
        self.role            = role
        self.assigned_job_ids = assigned_job_ids
        self.jti             = jti


async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(bearer_scheme)],
    redis=Depends(get_redis),
) -> CurrentUser:
    token = credentials.credentials
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired token",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            settings.JWT_SECRET_KEY,
            algorithms=[settings.JWT_ALGORITHM],
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token expired")
    except jwt.PyJWTError:
        raise credentials_exception

    # Check blocklist (logout/revoke)
    jti = payload.get("jti")
    if jti and await redis.exists(f"jwt:blocklist:{jti}"):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token has been revoked")

    return CurrentUser(
        user_id=payload["sub"],
        email=payload["email"],
        role=payload["role"],
        assigned_job_ids=payload.get("assigned_job_ids", []),
        jti=jti,
    )


def require_role(allowed_roles: list[str]):
    """
    Dependency factory: gates endpoint access to specified roles.

    Usage:
        @router.get("/admin/users", dependencies=[Depends(require_role(["SuperAdmin"]))])
        @router.post("/jobs/", dependencies=[Depends(require_role(["HR", "SuperAdmin"]))])
    """
    async def _check_role(current_user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
        if current_user.role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{current_user.role}' is not permitted. Required: {allowed_roles}",
            )
        return current_user

    return _check_role


def require_job_ownership():
    """
    Dependency: ensures the HR user owns the requested job opening.
    Inject as a dependency alongside a path param `job_id`.
    """
    async def _check_ownership(
        job_id: str,
        current_user: CurrentUser = Depends(get_current_user),
    ) -> CurrentUser:
        if current_user.role == "SuperAdmin":
            return current_user
        if job_id not in current_user.assigned_job_ids:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="You do not own this job opening",
            )
        return current_user

    return _check_ownership


# ── Usage examples ────────────────────────────────────────────────────────────
# from app.core.rbac import require_role, require_job_ownership

# @router.post("/jobs/", dependencies=[Depends(require_role(["HR", "SuperAdmin"]))])
# async def create_job(...): ...

# @router.get("/matches/{job_id}", dependencies=[Depends(require_job_ownership())])
# async def get_matches(job_id: str): ...

# @router.delete("/jobs/{job_id}", dependencies=[Depends(require_job_ownership())])
# async def delete_job(job_id: str): ...
```

---

## 3.5 WebSocket Notification Endpoint (FastAPI + Redis Pub/Sub)

```python
# app/api/v1/websocket.py
import asyncio
import json
import logging

import aioredis
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends, HTTPException, status

from app.core.config import settings
from app.core.deps import get_redis
from app.core.rbac import get_current_user, CurrentUser

router = APIRouter(tags=["websocket"])
logger = logging.getLogger(__name__)


@router.websocket("/ws/notifications/{hr_user_id}")
async def websocket_notifications(
    websocket: WebSocket,
    hr_user_id: str,
    token: str,  # passed as query param: ?token=<jwt>
):
    # ── Authenticate via query-param token ───────────────────────────────────
    # (WS clients cannot set headers; token is passed as query param)
    redis = await aioredis.from_url(settings.REDIS_URL)

    try:
        from app.core.rbac import verify_token
        current_user: CurrentUser = await verify_token(token, redis)
    except HTTPException:
        await websocket.close(code=4001)
        return

    # ── Authorization: HR can only listen to their own channel ───────────────
    if current_user.role not in ("HR", "SuperAdmin"):
        await websocket.close(code=4003)
        return
    if current_user.role == "HR" and current_user.user_id != hr_user_id:
        await websocket.close(code=4003)
        return

    await websocket.accept()
    logger.info("WS connected: hr_user_id=%s", hr_user_id)

    pubsub = redis.pubsub()
    channel = f"ws:hr:{hr_user_id}"
    await pubsub.subscribe(channel)

    # ── Send unread count on connect ──────────────────────────────────────────
    unread = await redis.get(f"notif:unread:{hr_user_id}") or 0
    await websocket.send_json({"type": "unread_count", "count": int(unread)})

    async def redis_listener():
        async for message in pubsub.listen():
            if message["type"] == "message":
                payload = message["data"]
                if isinstance(payload, bytes):
                    payload = payload.decode()
                await websocket.send_text(payload)

    async def ws_keepalive():
        while True:
            await asyncio.sleep(30)
            try:
                await websocket.send_json({"type": "ping"})
            except Exception:
                break

    try:
        await asyncio.gather(redis_listener(), ws_keepalive())
    except (WebSocketDisconnect, asyncio.CancelledError):
        logger.info("WS disconnected: hr_user_id=%s", hr_user_id)
    finally:
        await pubsub.unsubscribe(channel)
        await pubsub.close()
        await redis.close()
```

---

## 3.6 Notification Service — Kafka Consumer

```python
# workers/notification_service.py
import asyncio
import json
import logging
from typing import Any

import aioredis
import sendgrid
from aiokafka import AIOKafkaConsumer
from sendgrid.helpers.mail import Mail

from app.core.config import settings
from app.db.session import async_session_factory
from app.repositories.notification import NotificationRepository
from app.repositories.hr_preferences import HRPreferencesRepository
from app.repositories.user import UserRepository

logger = logging.getLogger(__name__)
sg = sendgrid.SendGridAPIClient(api_key=settings.SENDGRID_API_KEY)


async def get_hr_preferences(hr_user_id: str, redis, db):
    """Fetch HR notification preferences with Redis cache."""
    cache_key = f"prefs:{hr_user_id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    prefs_repo = HRPreferencesRepository(db)
    prefs = await prefs_repo.get_by_user_id(hr_user_id)
    if not prefs:
        # Default preferences
        return {"score_threshold": 75, "channel_email": True, "channel_inapp": True, "digest_mode": "immediate"}

    result = {
        "score_threshold": prefs.score_threshold,
        "channel_email":   prefs.channel_email,
        "channel_inapp":   prefs.channel_inapp,
        "digest_mode":     prefs.digest_mode,
    }
    await redis.setex(cache_key, 300, json.dumps(result))  # 5 min cache
    return result


async def send_email_notification(
    hr_email: str,
    hr_name: str,
    applicant_name: str,
    job_title: str,
    company: str,
    score: int,
    match_url: str,
) -> None:
    message = Mail(
        from_email=settings.SENDGRID_FROM_EMAIL,
        to_emails=hr_email,
        subject=f"🎯 New Match: {applicant_name} scored {score}/100 for {job_title} at {company}",
        html_content=f"""
        <h2>New Candidate Match Alert</h2>
        <p>Hi {hr_name},</p>
        <p>A new candidate has matched your job opening with a score of <strong>{score}/100</strong>.</p>
        <table>
          <tr><td><b>Candidate:</b></td><td>{applicant_name}</td></tr>
          <tr><td><b>Position:</b></td><td>{job_title} at {company}</td></tr>
          <tr><td><b>Match Score:</b></td><td>{score}/100</td></tr>
        </table>
        <p><a href="{settings.FRONTEND_URL}{match_url}">Review Candidate →</a></p>
        """,
    )
    try:
        sg.send(message)
        logger.info("Email sent to %s", hr_email)
    except Exception as exc:
        logger.error("SendGrid email failed: %s", exc)


async def process_notification(event: dict[str, Any], redis) -> None:
    application_id    = event["application_id"]
    applicant_user_id = event["applicant_user_id"]
    rankings          = event.get("rankings", [])
    correlation_id    = event.get("correlation_id", "")

    log = logger.bind(application_id=application_id, correlation_id=correlation_id)

    async with async_session_factory() as db:
        notif_repo = NotificationRepository(db)
        user_repo  = UserRepository(db)

        applicant = await user_repo.get_by_id(applicant_user_id)
        applicant_name = applicant.name if applicant else "Unknown Applicant"

        for ranking in rankings:
            job_id     = ranking["job_id"]
            score      = ranking["score"]
            hr_user_id = ranking.get("hr_user_id")
            job_title  = ranking.get("job_title", "")
            company    = ranking.get("company", "")

            if not hr_user_id:
                continue

            prefs = await get_hr_preferences(hr_user_id, redis, db)

            if score < prefs["score_threshold"]:
                log.debug("Score %d below threshold %d for HR %s", score, prefs["score_threshold"], hr_user_id)
                continue

            # ── Store notification ───────────────────────────────────────────
            notification = await notif_repo.create(
                hr_user_id=hr_user_id,
                application_id=application_id,
                job_opening_id=job_id,
                score=score,
            )

            # ── Increment unread counter ────────────────────────────────────
            await redis.incr(f"notif:unread:{hr_user_id}")

            match_url = f"/matches/{job_id}?app={application_id}"

            notification_payload = json.dumps({
                "type":            "new_match",
                "notification_id": str(notification.id),
                "applicant_name":  applicant_name,
                "job_title":       job_title,
                "company":         company,
                "score":           score,
                "match_url":       match_url,
                "correlation_id":  correlation_id,
            })

            if prefs["digest_mode"] == "immediate":
                # ── WebSocket via Redis Pub/Sub ──────────────────────────────
                if prefs["channel_inapp"]:
                    await redis.publish(f"ws:hr:{hr_user_id}", notification_payload)

                # ── Email via SendGrid ───────────────────────────────────────
                if prefs["channel_email"]:
                    hr_user = await user_repo.get_by_id(hr_user_id)
                    await send_email_notification(
                        hr_email=hr_user.email,
                        hr_name=hr_user.name,
                        applicant_name=applicant_name,
                        job_title=job_title,
                        company=company,
                        score=score,
                        match_url=match_url,
                    )
            else:
                # ── Buffer for digest ────────────────────────────────────────
                await redis.rpush(f"notif:digest:{hr_user_id}", notification_payload)

        await db.commit()
        log.info("Notification processing complete for application %s", application_id)


async def main() -> None:
    redis = await aioredis.from_url(settings.REDIS_URL)

    consumer = AIOKafkaConsumer(
        "match-completed",
        bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
        group_id="notification-cg",
        value_deserializer=lambda v: json.loads(v.decode()),
        enable_auto_commit=False,
    )

    await consumer.start()
    try:
        async for msg in consumer:
            try:
                await process_notification(msg.value, redis)
                await consumer.commit()
            except Exception as exc:
                logger.error("Notification processing failed: %s", exc, exc_info=True)
                await consumer.commit()  # don't re-process indefinitely
    finally:
        await consumer.stop()
        await redis.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 3.7 Next.js `useNotifications` Hook

```typescript
// hooks/useNotifications.ts
import { useCallback, useEffect, useRef, useState } from "react";

export interface Notification {
  notification_id: string;
  type: string;
  applicant_name: string;
  job_title: string;
  company: string;
  score: number;
  match_url: string;
  timestamp: string;
}

interface UseNotificationsOptions {
  hrUserId: string;
  token: string;
  wsBaseUrl?: string;
}

const MAX_RECONNECT_ATTEMPTS = 5;
const RECONNECT_BASE_DELAY_MS = 1000;

export function useNotifications({
  hrUserId,
  token,
  wsBaseUrl = process.env.NEXT_PUBLIC_WS_URL ?? "ws://localhost:8000",
}: UseNotificationsOptions) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [unreadCount, setUnreadCount]     = useState(0);
  const [connected, setConnected]         = useState(false);

  const wsRef             = useRef<WebSocket | null>(null);
  const reconnectAttempts = useRef(0);
  const reconnectTimer    = useRef<NodeJS.Timeout | null>(null);
  const isMounted         = useRef(true);

  const connect = useCallback(() => {
    if (!hrUserId || !token) return;

    const url = `${wsBaseUrl}/ws/notifications/${hrUserId}?token=${encodeURIComponent(token)}`;
    const ws  = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setConnected(true);
      reconnectAttempts.current = 0;
    };

    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);

        if (data.type === "ping") return;

        if (data.type === "unread_count") {
          setUnreadCount(data.count);
          return;
        }

        if (data.type === "new_match") {
          const notification: Notification = {
            ...data,
            timestamp: new Date().toISOString(),
          };
          setNotifications((prev) => [notification, ...prev]);
          setUnreadCount((prev) => prev + 1);
        }
      } catch {
        // ignore malformed messages
      }
    };

    ws.onclose = () => {
      setConnected(false);
      if (!isMounted.current) return;

      if (reconnectAttempts.current < MAX_RECONNECT_ATTEMPTS) {
        const delay = RECONNECT_BASE_DELAY_MS * 2 ** reconnectAttempts.current;
        reconnectAttempts.current += 1;
        reconnectTimer.current = setTimeout(connect, delay);
      }
    };

    ws.onerror = () => {
      ws.close();
    };
  }, [hrUserId, token, wsBaseUrl]);

  useEffect(() => {
    isMounted.current = true;
    connect();
    return () => {
      isMounted.current = false;
      if (reconnectTimer.current) clearTimeout(reconnectTimer.current);
      wsRef.current?.close();
    };
  }, [connect]);

  const markAllRead = useCallback(() => {
    setUnreadCount(0);
    setNotifications((prev) => prev.map((n) => ({ ...n })));
    // Also call PATCH /notifications/mark-read API
  }, []);

  const dismissNotification = useCallback((notificationId: string) => {
    setNotifications((prev) => prev.filter((n) => n.notification_id !== notificationId));
  }, []);

  return { notifications, unreadCount, connected, markAllRead, dismissNotification };
}
```

---

## 3.8 Redis Caching Decorator (Python)

```python
# app/core/cache.py
import asyncio
import functools
import hashlib
import json
import logging
from typing import Any, Callable

from app.core.deps import get_redis

logger = logging.getLogger(__name__)


def redis_cache(
    ttl: int = 300,
    key_prefix: str = "cache",
    key_builder: Callable[..., str] | None = None,
):
    """
    Async cache-aside decorator for FastAPI route handlers.

    Args:
        ttl:         Time-to-live in seconds.
        key_prefix:  Prefix for the Redis key.
        key_builder: Optional callable that receives (*args, **kwargs) and
                     returns a string cache key. Defaults to hash of args.

    Usage:
        @router.get("/matches/{job_id}")
        @redis_cache(ttl=86400, key_prefix="match")
        async def get_match_result(job_id: str, app_id: str):
            ...
    """

    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            # Resolve Redis from function kwargs (FastAPI Depends injected)
            redis = kwargs.get("redis") or await get_redis().__anext__()

            # Build cache key
            if key_builder:
                cache_key = f"{key_prefix}:{key_builder(*args, **kwargs)}"
            else:
                raw = json.dumps(
                    {"args": [str(a) for a in args], "kwargs": {k: str(v) for k, v in kwargs.items()}},
                    sort_keys=True,
                )
                hash_suffix = hashlib.sha256(raw.encode()).hexdigest()[:16]
                cache_key = f"{key_prefix}:{func.__name__}:{hash_suffix}"

            # ── Cache hit ─────────────────────────────────────────────────────
            cached = await redis.get(cache_key)
            if cached:
                logger.debug("Cache hit: %s", cache_key)
                return json.loads(cached)

            # ── Cache miss: call original function ────────────────────────────
            logger.debug("Cache miss: %s", cache_key)
            result = await func(*args, **kwargs)

            # Serialize and store
            try:
                serialized = json.dumps(result if not hasattr(result, "__dict__") else result.dict())
                await redis.setex(cache_key, ttl, serialized)
            except (TypeError, AttributeError):
                logger.warning("Could not serialize result for cache key %s", cache_key)

            return result

        wrapper._cache_decorator = True  # marker for testing/inspection
        return wrapper

    return decorator


# ── Usage example ─────────────────────────────────────────────────────────────
# @router.get("/matches/{job_id}")
# @redis_cache(
#     ttl=86400,
#     key_prefix="match",
#     key_builder=lambda *args, job_id, application_id, **kw: f"{application_id}:{job_id}",
# )
# async def get_match_result(
#     job_id: str,
#     application_id: str,
#     db: AsyncSession = Depends(get_db),
#     redis = Depends(get_redis),
# ):
#     return await MatchResultRepository(db).get(application_id, job_id)
```

---

## 3.9 Job Opening Ingestion into ChromaDB

```python
# app/services/job_ingestion.py
import json
import logging
import uuid
from typing import Any

from langchain_huggingface import HuggingFaceEmbeddings
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import settings
from app.models.job_opening import JobOpening
from app.repositories.job_opening import JobOpeningRepository
from app.services.chroma import get_job_collection

logger = logging.getLogger(__name__)

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    encode_kwargs={"normalize_embeddings": True},
)


def build_job_document(job: JobOpening) -> tuple[str, list[float], dict[str, Any]]:
    """Build document text, embedding, and metadata for a job opening."""
    doc_text = (
        f"Job Title: {job.title}\n"
        f"Company: {job.company}\n"
        f"Job Description: {job.jd_text}\n"
        f"Required Skills: {', '.join(job.required_skills)}\n"
        f"Experience Required: {job.experience_years} years"
    )
    vector = embeddings.embed_query(doc_text)
    metadata = {
        "job_id":           str(job.id),
        "hr_user_id":       str(job.hr_user_id),
        "company":          job.company,
        "title":            job.title,
        "required_skills":  ",".join(job.required_skills),
        "experience_years": job.experience_years,
        "status":           job.status,
    }
    return doc_text, vector, metadata


async def ingest_job_opening(
    job: JobOpening,
    db: AsyncSession,
    redis,
) -> str:
    """
    Embeds a job opening and upserts it into ChromaDB.
    Invalidates the active jobs Redis cache.
    Returns the ChromaDB doc_id.
    """
    collection = get_job_collection()
    doc_id     = str(job.id)  # Use job UUID as stable ChromaDB document ID

    doc_text, vector, metadata = build_job_document(job)

    # ── Upsert into ChromaDB (idempotent — same ID = update) ─────────────────
    collection.upsert(
        ids=[doc_id],
        documents=[doc_text],
        embeddings=[vector],
        metadatas=[metadata],
    )
    logger.info("Upserted job %s into ChromaDB", doc_id)

    # ── Persist chroma_doc_id in PostgreSQL ───────────────────────────────────
    job_repo = JobOpeningRepository(db)
    await job_repo.set_chroma_doc_id(job.id, doc_id)
    await db.commit()

    # ── Invalidate caches ─────────────────────────────────────────────────────
    await redis.delete("jobs:active")

    # Invalidate all cached ChromaDB top-K results (job set has changed)
    async for key in redis.scan_iter("chroma:topk:*"):
        await redis.delete(key)

    logger.info("Cache invalidated for jobs:active and chroma:topk:*")
    return doc_id


async def remove_job_from_chroma(
    job_id: str,
    db: AsyncSession,
    redis,
) -> None:
    """Removes a job opening from ChromaDB and invalidates caches."""
    collection = get_job_collection()
    collection.delete(ids=[job_id])
    logger.info("Deleted job %s from ChromaDB", job_id)

    await redis.delete("jobs:active")
    async for key in redis.scan_iter("chroma:topk:*"):
        await redis.delete(key)


# ── FastAPI Route Integration ─────────────────────────────────────────────────
# @router.post("/jobs/", response_model=JobResponse)
# async def create_job(
#     payload: JobCreateRequest,
#     db: AsyncSession = Depends(get_db),
#     redis = Depends(get_redis),
#     current_user: CurrentUser = Depends(require_role(["HR", "SuperAdmin"])),
# ):
#     job_repo = JobOpeningRepository(db)
#     job = await job_repo.create(hr_user_id=current_user.user_id, **payload.dict())
#     await ingest_job_opening(job, db, redis)
#     return JobResponse.from_orm(job)
```

---

---

# SECTION 4 — ADDITIONAL REQUIREMENTS

---

## Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:

  # ── PostgreSQL ─────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       ${POSTGRES_DB}
      POSTGRES_USER:     ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      retries: 5

  # ── Redis ──────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      retries: 5

  # ── Zookeeper ─────────────────────────────────────────────────────────────
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME:   2000
    volumes:
      - zk_data:/var/lib/zookeeper/data

  # ── Kafka ─────────────────────────────────────────────────────────────────
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID:                        1
      KAFKA_ZOOKEEPER_CONNECT:                zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS:             PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE:        "true"
    ports:
      - "9092:9092"
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
      interval: 15s
      retries: 5

  # ── ChromaDB ──────────────────────────────────────────────────────────────
  chromadb:
    image: chromadb/chroma:latest
    environment:
      CHROMA_SERVER_AUTH_CREDENTIALS_FILE: ""
      ANONYMIZED_TELEMETRY:                "false"
    volumes:
      - chroma_data:/chroma/chroma
    ports:
      - "8001:8000"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/api/v1/heartbeat || exit 1"]
      interval: 10s
      retries: 5

  # ── FastAPI (API Service) ──────────────────────────────────────────────────
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL:              ${DATABASE_URL}
      REDIS_URL:                 ${REDIS_URL}
      KAFKA_BOOTSTRAP_SERVERS:   kafka:9092
      CHROMADB_HOST:             chromadb
      CHROMADB_PORT:             8000
      JWT_SECRET_KEY:            ${JWT_SECRET_KEY}
      HUGGINGFACE_API_TOKEN:     ${HUGGINGFACE_API_TOKEN}
      SENDGRID_API_KEY:          ${SENDGRID_API_KEY}
      SENDGRID_FROM_EMAIL:       ${SENDGRID_FROM_EMAIL}
      FRONTEND_URL:              ${FRONTEND_URL}
    ports:
      - "8000:8000"
    depends_on:
      postgres:  { condition: service_healthy }
      redis:     { condition: service_healthy }
      kafka:     { condition: service_healthy }
      chromadb:  { condition: service_healthy }
    volumes:
      - uploads:/app/uploads
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4

  # ── Resume Ingestion Worker ────────────────────────────────────────────────
  resume-worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    command: python -m workers.resume_ingestion_worker
    environment:
      DATABASE_URL:            ${DATABASE_URL}
      REDIS_URL:               ${REDIS_URL}
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      CHROMADB_HOST:           chromadb
      CHROMADB_PORT:           8000
    depends_on:
      postgres: { condition: service_healthy }
      kafka:    { condition: service_healthy }
      chromadb: { condition: service_healthy }
    volumes:
      - uploads:/app/uploads
    deploy:
      replicas: 3

  # ── LLM Scoring Worker ────────────────────────────────────────────────────
  llm-worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    command: python -m workers.llm_scoring_worker
    environment:
      DATABASE_URL:            ${DATABASE_URL}
      REDIS_URL:               ${REDIS_URL}
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      CHROMADB_HOST:           chromadb
      CHROMADB_PORT:           8000
      HUGGINGFACE_API_TOKEN:   ${HUGGINGFACE_API_TOKEN}
    depends_on:
      postgres: { condition: service_healthy }
      kafka:    { condition: service_healthy }
      chromadb: { condition: service_healthy }
    deploy:
      replicas: 2

  # ── Notification Service ──────────────────────────────────────────────────
  notification-service:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    command: python -m workers.notification_service
    environment:
      DATABASE_URL:            ${DATABASE_URL}
      REDIS_URL:               ${REDIS_URL}
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SENDGRID_API_KEY:        ${SENDGRID_API_KEY}
      SENDGRID_FROM_EMAIL:     ${SENDGRID_FROM_EMAIL}
      FRONTEND_URL:            ${FRONTEND_URL}
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
      kafka:    { condition: service_healthy }
    deploy:
      replicas: 2

  # ── Next.js Frontend ──────────────────────────────────────────────────────
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    environment:
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL}
      NEXT_PUBLIC_WS_URL:  ${NEXT_PUBLIC_WS_URL}
    ports:
      - "3000:3000"
    depends_on:
      - api

volumes:
  postgres_data:
  redis_data:
  zk_data:
  chroma_data:
  uploads:
```

---

## Environment Configuration (`.env`)

```dotenv
# ── PostgreSQL ────────────────────────────────────────────────────────────────
POSTGRES_DB=hr_recruitment
POSTGRES_USER=hr_admin
POSTGRES_PASSWORD=<strong-password>
DATABASE_URL=postgresql+asyncpg://hr_admin:<password>@postgres:5432/hr_recruitment

# ── Redis ─────────────────────────────────────────────────────────────────────
REDIS_PASSWORD=<strong-redis-password>
REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0

# ── Kafka ─────────────────────────────────────────────────────────────────────
KAFKA_BOOTSTRAP_SERVERS=kafka:9092

# ── ChromaDB ─────────────────────────────────────────────────────────────────
CHROMADB_HOST=chromadb
CHROMADB_PORT=8000

# ── JWT Authentication ────────────────────────────────────────────────────────
JWT_SECRET_KEY=<256-bit-random-secret>
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=60
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# ── HuggingFace ───────────────────────────────────────────────────────────────
HUGGINGFACE_API_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx
HUGGINGFACE_LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2
HUGGINGFACE_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2

# ── SendGrid ──────────────────────────────────────────────────────────────────
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxx
SENDGRID_FROM_EMAIL=noreply@yourplatform.com

# ── Application ───────────────────────────────────────────────────────────────
FRONTEND_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000

# ── File Storage ──────────────────────────────────────────────────────────────
UPLOAD_DIR=/app/uploads
MAX_UPLOAD_SIZE_MB=10

# ── Observability ─────────────────────────────────────────────────────────────
LOG_LEVEL=INFO
SENTRY_DSN=https://xxx@sentry.io/xxx   # optional
```

---

## Scalability Notes

### Kafka Consumer Scaling
```
Rule: one consumer instance per partition (no benefit exceeding partition count).

resume-ingestion-cg:  scale 1 → 6 replicas (6 partitions)
llm-scoring-cg:       scale 1 → 4 replicas (LLM is I/O-bound; HuggingFace API calls)
                      Add GPU-backed replicas for self-hosted LLM inference
notification-cg:      scale 1 → 12 replicas (12 partitions for notification-dispatch)

Auto-scaling trigger: Kafka consumer lag > 1000 messages (Kubernetes HPA via KEDA)
```

### WebSocket Horizontal Scaling
```
Challenge: sticky sessions needed for WS connections.
Solution:  Redis Pub/Sub fan-out eliminates sticky session requirement.
           Any FastAPI instance can receive a Pub/Sub message and forward to its WS.

Nginx upstream:
  upstream api_ws {
    least_conn;
    server api_1:8000;
    server api_2:8000;
    server api_3:8000;
  }
  # WS connections auto-distributed — Redis handles cross-instance delivery
```

### LLM Inference Scaling
```
Option A (HuggingFace Inference API):
  Scale via API key rotation + rate limit management.
  Use asyncio.Semaphore to limit concurrent LLM requests per worker.

Option B (Self-hosted Mistral / vLLM):
  Deploy vLLM server on GPU nodes (A100/H100).
  Use OpenAI-compatible API endpoint.
  Horizontal scale: 1 vLLM replica per GPU.
  Load balance via Nginx round-robin.

Option C (Cost optimization):
  Queue LLM requests during peak → batch inference on vLLM for 5–10x throughput.
```

---

## Security Notes

### PDF Upload Security
```python
import magic  # python-magic

def validate_pdf(content: bytes) -> None:
    mime = magic.from_buffer(content, mime=True)
    if mime != "application/pdf":
        raise HTTPException(415, "File content is not a valid PDF")
    # Additional: scan with ClamAV antivirus in production
    # Max size enforced at Nginx level (client_max_body_size 10m)
```

### JWT Security
```
- Access token TTL:  60 minutes
- Refresh token TTL: 7 days, stored in PostgreSQL + Redis
- Refresh rotation:  issue new refresh token on each use, revoke old one
- Blocklist:         jti stored in Redis on logout (TTL = remaining token lifetime)
- Signing:           HS256 with 256-bit secret (rotate periodically with key versioning)
- Transport:         HTTPS only; tokens never in URLs (use Authorization header)
```

### Rate Limiting
```nginx
# Nginx rate limiting per role (set via JWT claim in header)
limit_req_zone $http_x_user_role zone=applicant:10m rate=10r/m;   # 10 uploads/min
limit_req_zone $http_x_user_role zone=hr:10m        rate=60r/m;   # 60 req/min
limit_req_zone $binary_remote_addr zone=global:10m  rate=100r/s;

# FastAPI-level rate limiting (slowapi)
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/resume/upload")
@limiter.limit("5/minute")  # per IP
async def upload_resume(...): ...
```

### Input Sanitization
```
- PDF filename:   UUID-generated, never user-provided filename used on disk
- Text extracted: strip null bytes, limit to 100K chars before embedding
- SQL:            SQLAlchemy ORM with parameterized queries (no raw SQL with user input)
- JSON output:    Pydantic v2 models validate all LLM output before DB insert
```

---

## Observability — Structured Logging with Correlation IDs

```python
# app/core/logging.py
import logging
import uuid
from contextvars import ContextVar

import structlog

correlation_id_var: ContextVar[str] = ContextVar("correlation_id", default="")


def get_correlation_id() -> str:
    return correlation_id_var.get() or str(uuid.uuid4())


# Configure structlog
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.BoundLogger,
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()


# FastAPI middleware: inject correlation_id into every request
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class CorrelationIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        cid = request.headers.get("X-Correlation-ID") or str(uuid.uuid4())
        correlation_id_var.set(cid)
        structlog.contextvars.bind_contextvars(correlation_id=cid)
        response = await call_next(request)
        response.headers["X-Correlation-ID"] = cid
        return response
```

### Correlation ID Flow Through Kafka Events

```
HTTP Request → FastAPI (correlation_id = uuid4)
  → Kafka event: resume-uploaded  { correlation_id }
  → Resume Ingestion Worker logs: { correlation_id, application_id, stage: "ingesting" }
  → Kafka event: match-requested  { correlation_id }
  → LLM Scoring Worker logs:      { correlation_id, application_id, stage: "scoring" }
  → Kafka event: match-completed  { correlation_id }
  → Notification Service logs:    { correlation_id, application_id, stage: "notifying" }

All log lines for one resume upload share the same correlation_id.
Grep / Loki query: { correlation_id="abc-123" } → full trace in seconds.
```

### Log Aggregation Stack (recommended)
```
FastAPI + Workers → stdout (JSON) → Fluent Bit → Loki / Elasticsearch
                                               → Grafana dashboards
                                               → Alertmanager (PagerDuty/Slack on DLQ events)
```

---

*End of Document*
