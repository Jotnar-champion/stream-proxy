---

# HR Venture — AI-Powered CV Analysis & Ranking Engine

## Project Overview

This is a **backend-only FastAPI application** that automates the end-to-end recruitment pipeline. It accepts bulk CV uploads alongside a Job Description (JD), a Scoring Framework, and an optional Evaluator Persona, then uses **Azure OpenAI (GPT-4o)** to score and rank every candidate. Results can be exported as CSV, DOCX reports, or polished PowerPoint presentations.

---

## High-Level Design (HLD)

### System Context Diagram

```mermaid
graph TD
    FE["Frontend (Next.js)\nlocalhost:3000"] -->|REST API| API["FastAPI Web Server\n:8080"]
    API -->|Enqueue task| Redis["Redis Broker\n:6379"]
    Redis -->|Dequeue| Worker["Celery Worker\n(2 concurrent)"]
    Worker -->|LLM scoring| AzureAI["Azure OpenAI\nGPT-4o + Embeddings"]
    Worker -->|Persist results| PG["PostgreSQL 15\n+ pgvector"]
    API -->|Read results| PG
    API -->|Email notifications| AzureEmail["Azure Communication\nEmail Service"]
```

### Request Flow — Batch CV Processing

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant API as FastAPI
    participant DB as PostgreSQL
    participant Q  as Redis Queue
    participant W  as Celery Worker
    participant AI as Azure OpenAI

    FE->>API: POST /api/role-data/upload-role-cv-batch\n(JD, Persona, Scoring Framework, CVs)
    API->>DB: Create Role record (status=queue)
    API->>Q: process_batch.delay(role_id, paths...)
    API-->>FE: {status: processing_started, role_id}
    W->>W: Extract JD/SF/Persona text
    loop For each CV (parallel chord)
        W->>AI: parse_cv() → structured JSON
        W->>W: Duplicate check (email/phone/LinkedIn)
        W->>AI: process_cv() → LLM scoring + dimensions
        W->>AI: generate_evidence_summary()
        W->>DB: Save CVFile (score, evidence, json_data)
    end
    W->>W: aggregate_batch_results() — rank & sort
    W->>DB: Update Role (status=completed, results=JSON)
    W->>AzureEmail: Send completion email (admin)
    FE->>API: GET /api/role-data/job-results/{role_id}
    API-->>FE: Ranked candidates + stats
```

### Key Architectural Decisions

| Decision | Choice | Reason |
|---|---|---|
| API Framework | FastAPI | Async, auto-docs, Pydantic validation |
| Task Queue | Celery + Redis | Async batch processing, parallel chord |
| LLM Provider | Azure OpenAI GPT-4o | Structured JSON output, enterprise compliance |
| DB | PostgreSQL + pgvector | Relational + vector similarity for RAG (future) |
| Migrations | Alembic | Versioned schema migrations |
| Containerization | Docker Compose | 4-service stack tuned for 2-CPU/8GB server |

---

## Low-Level Design (LLD)

### Database Schema

```mermaid
erDiagram
    roles ||--o| job_descriptions : "has"
    roles ||--o| persona_profiles : "has"
    roles ||--o| scoring_frameworks : "has"
    roles ||--o{ cv_files : "contains"
    cv_files ||--o{ document_chunks : "chunked into"

    roles {
        UUID id PK
        String title
        String company
        String status "queue|processing|completed|failed"
        JSON results
        Integer cv_count
        TIMESTAMP created_at
        TIMESTAMP started_at
        TIMESTAMP completed_at
    }

    cv_files {
        UUID id PK
        UUID role_id FK
        String filename
        JSONB json_data "parsed CV structure"
        Text text "raw extracted text"
        Float score
        Text evidence
        Boolean is_duplicate
        String fingerprint
    }

    document_chunks {
        UUID id PK
        UUID cv_file_id FK
        UUID role_id FK
        Text chunk_text
        Vector(1536) embedding "pgvector"
        JSONB chunk_metadata
        Integer chunk_index
    }
```

### Module Breakdown

#### role_data.py — REST Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/role-data/roles` | GET | List all roles with CV counts |
| `/role-data/role-status/{role_id}` | GET | Poll job processing status |
| `/role-data/job-results/{role_id}` | GET | Fetch ranked results |
| `/role-data/upload-role-cv-batch` | POST | Upload JD + CVs → triggers Celery |
| `/role-data/roles/{role_id}/export/candidate/{filename}` | GET | Single candidate CSV export |
| `/role-data/roles/{role_id}/export/all` | GET | All candidates CSV export |
| `/role-data/roles/{role_id}/export/report/{filename}` | GET | Candidate DOCX report |
| `/role-data/roles/{role_id}/generate-ppt` | POST | PowerPoint deck generation |

#### celery_app.py — Async Processing Pipeline

```mermaid
graph LR
    PB["process_batch\n(entry task)"] --> G["Celery group\nprocess_single_cv × N"]
    G --> AB["aggregate_batch_results\n(chord callback)"]
    
    subgraph process_single_cv
        E["Extract PDF text\n(PyMuPDF)"] --> P["parse_cv()\nLLM → structured JSON"]
        P --> D["Duplicate check\nemail/phone/LinkedIn"]
        D --> S["process_cv()\nPillar/dimension scoring"]
        S --> EV["generate_evidence_summary()"]
        EV --> DB2["Save to cv_files"]
    end
```

Three Celery tasks:
- **`process_batch`** — orchestrator: extracts JD/SF/Persona text, fans out using a `chord(group(...), aggregate_batch_results)`
- **`process_single_cv`** — worker: processes one CV end-to-end, returns a result dict
- **`aggregate_batch_results`** — chord callback: sorts by score, assigns ranks, saves final ranked JSON to `roles.results`, sends email

#### utils — Core Utility Modules

| Module | Responsibility |
|---|---|
| azure_ai.py | Two `AzureOpenAI` clients — `chat_client` (GPT-4o) and `embedding_client` |
| file_extractors.py | PDF → raw text via `PyMuPDFLoader` |
| structured_parser.py | Pydantic models for JD, Persona, Scoring Framework, CV; LLM JSON parsing via `parse_cv()` |
| pillar_dimensions.py | Core LLM scoring prompt + `process_cv()` → returns pillar/dimension scores, `get_relevant_cv_snippets()` using embeddings |
| evidence.py | `generate_evidence_summary()` — bullet-point fit summary via GPT-4o |
| duplicate_checker.py | In-memory `DuplicateChecker` (email/phone/LinkedIn/filename deduplication) |
| export_files.py | CSV and DOCX export builders |
| `file_util.py` | `save_temp()` — saves uploaded files to `/tmp/hr_uploads/{role_id}/` |
| azure_email.py | Azure Communication Services email sender |
| `email_templates.py` | Jinja2 template rendering for email subject/body |

#### ppt — PowerPoint Report Generator

```mermaid
graph TD
    PPT["ppt_generator.py\ngenerate_ppt()"] --> T["Load PPTX template\nconfidential_template.pptx"]
    PPT --> Tier["tier_renderer.py\nTier 1/2/3/4 candidate slides"]
    PPT --> Charts["chart_renderer.py"]
    Charts --> SD["score_distribution"]
    Charts --> GL["gender_location"]
    Charts --> LD["language_distribution"]
    Charts --> Seg["segment_diversity\n(AI-powered clustering)"]
    Charts --> HSC["high_score_cluster"]
    Charts --> PAI["pre_assessment_insight"]
    Charts --> JW["jewelry_watch (custom)"]
```

Candidates are split into **Tier 1** (top-X), **Tier 2** (score ≥ 65, up to 50), **Tier 3/4** (rest). Each tier has its own slide template and layout rules defined in template_config.py.

#### LLM Scoring Model — Pillar/Dimension Framework

The scoring prompt in pillar_dimensions.py implements a rigorous evaluation:

- Evidence MUST come verbatim from the CV
- Each dimension in the Scoring Framework is evaluated independently
- Scores follow a 1–5 behavioral anchor scale, mapped to 1–10 and then weighted
- Pillar weights from the Scoring Framework determine the `final_score` (out of 100)
- Persona file acts as an interpretive lens but cannot inflate scores without evidence

---

## Infrastructure (Docker Compose)

```mermaid
graph LR
    subgraph Docker Network
        WEB["web\nFastAPI :8080\n512MB RAM"]
        WORKER["worker\nCelery (2 concurrent)\n4GB RAM / 2 CPU"]
        PG["postgres:15\n1.5GB RAM\npgdata volume"]
        REDIS["redis:7\n512MB LRU\nno persistence"]
    end
    WEB --> PG
    WEB --> REDIS
    WORKER --> PG
    WORKER --> REDIS
```

Memory is carefully budgeted for a **2-core / 8GB RAM** server. The worker is given 4GB to handle large LLM context windows. Celery is configured with `max-tasks-per-child=30` to prevent memory leaks from repeated LLM calls.

---

## Data Flow Summary

```
[Upload API] → save files to disk
     ↓
[Celery: process_batch]
     ↓ extract JD/SF/Persona text
     ↓
[Celery chord: N × process_single_cv] (parallel)
     ↓ PDF → text → LLM parse → duplicate check → LLM score → evidence
     ↓
[Celery: aggregate_batch_results]
     ↓ sort by score → assign ranks → save to roles.results
     ↓ send admin email
     ↓
[GET job-results] → ranked candidates with scores, dimensions, evidence
     ↓
[Export] → CSV / DOCX / PowerPoint presentation
```
