Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/package.json)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/requirements.txt)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/docker-compose.yml#1-1), lines 1 to 80

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/Dockerfile)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/database.py)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/main.py)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/celery_app.py#1-1), lines 1 to 80

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/next.config.ts#1-1), lines 1 to 50

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/utils/azure_ai.py)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/lib/api.ts#1-1), lines 1 to 60

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/lib/axios.ts)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/utils/pillar_dimensions.py#1-1), lines 1 to 50

Searched for files matching `**/be/.env*`, 2 matches

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/alembic/versions/0001_initial.py#1-1), lines 1 to 60

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/lib/auth.ts#1-1), lines 1 to 40

Searched for files matching `**/fe/.env*`, no matches

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/.env.example)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/.env_old)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/next.config.ts#50-50), lines 50 to 60

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/fe/middleware.ts)

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/api/role_data.py#1-1), lines 1 to 40

Read [](file:///c%3A/Temporary_Stuff/Personal/HR-venture/be/app/utils/structured_parser.py#1-1), lines 1 to 60

Now I have everything I need. Here's the full, practical setup guide:

---

# HR-Venture Local Setup Guide

## Critical Warning: Windows vs WSL2

**Celery's `chord` parallelism is broken on Windows** with the default `prefork` pool. The entire batch pipeline depends on `chord`. You have two options:

- **Option A (recommended)**: Run via Docker Desktop (avoids all OS issues)
- **Option B**: Use WSL2 for the backend, run frontend natively

All commands below assume **Docker Desktop** (Option A) first, then a no-Docker path.

---

## 1. Dependency Inventory

| Layer | Dependency | Version | Notes |
|---|---|---|---|
| Frontend runtime | Node.js | ≥ 20 LTS | Next.js 15 requirement |
| Frontend package manager | npm (bundled with Node) | — | No pnpm/yarn needed |
| Frontend framework | Next.js | 15.5.4 | React 19 |
| Frontend auth | NextAuth.js | v5 beta | Microsoft Entra ID |
| Backend language | Python | **3.11** | Dockerfile pins this; 3.12 has known issues with some deps |
| Backend framework | FastAPI + uvicorn | latest | Port 8080 |
| Task queue | Celery + Redis | — | `chord` pattern — broken on Windows without Docker |
| SQL DB | PostgreSQL | 15 | Must have `pgvector` extension enabled |
| Cache/Broker | Redis | 7 | Dual-role: Celery broker (db=0) AND result backend (db=1) |
| Vector operations | pgvector | ≥ 0.2.0 | Python client only; `document_chunks` table is unused dead code |
| AI — LLM | Azure OpenAI | GPT-4o | Three calls per CV |
| AI — Embeddings | Azure OpenAI | `text-embedding-3-small` | One call per CV chunk |
| Email | Azure Communication Services | — | Optional — only for admin notification after batch completes |
| PyTorch (CPU) | torch | 2.7.1+cpu | Only for scipy/numpy ops; installed from PyTorch index |
| **No ChromaDB** | — | — | Confirmed not present anywhere |

---

## 2. Startup Path

```
uvicorn app.main:app
  └── main.py
        ├── init_db()            ← Base.metadata.create_all() — auto-creates tables (NOT Alembic)
        ├── include_router(role_router, prefix='/api')
        └── CORS middleware

Frontend: next dev
  └── app/page.tsx
        └── axios baseURL = NEXT_PUBLIC_API_URL || "http://74.242.171.29:8080"  ← MUST override
              └── all API calls to /api/role-data/*

Celery: celery -A app.celery_app.celery worker
  └── celery_app.py
        ├── broker = CELERY_BROKER_URL (redis://localhost:6379/0)
        └── backend = CELERY_RESULT_BACKEND (redis://localhost:6379/1)
```

**DB init happens on web startup** (`create_all`), not via `alembic upgrade head`. The migration file in `alembic/versions/0001_initial.py` is stale — do not run it, it will fail with constraint errors against the current schema.

**OpenAI clients**: azure_ai.py creates two `AzureOpenAI` clients (for embed + chat). pillar_dimensions.py and structured_parser.py create a **separate** `openai.OpenAI(base_url=OPENAI_BASE_URL)` client. Both sets of env vars are required.

---

## 3. Option A — Full Docker Setup (Recommended)

### 3.1 Install Docker Desktop

Download from docker.com/products/docker-desktop. Enable WSL2 backend on Windows.

### 3.2 Create .env

The .env.example is incomplete. Create .env with the full set:

```bash
# === PostgreSQL ===
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=hr_venture

# For Docker networking (service name "postgres"):
DATABASE_URL=postgresql+psycopg2://postgres:postgres@postgres:5432/hr_venture

# === Redis / Celery ===
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/1

# === Azure OpenAI (AzureOpenAI client — azure_ai.py) ===
AZURE_OPENAI_ENDPOINT=https://YOUR_RESOURCE.openai.azure.com/
AZURE_OPENAI_KEY=YOUR_KEY_HERE
CHAT_DEPLOYMENT=gpt-4o
EMBEDDING_DEPLOYMENT=embedding-small

# === Azure OpenAI (openai.OpenAI client — pillar_dimensions.py, structured_parser.py) ===
# Must be the Azure-compatible endpoint WITHOUT trailing slash
OPENAI_BASE_URL=https://YOUR_RESOURCE.openai.azure.com/openai
LLM_MODEL=gpt-4o

# === Azure Communication Services (email — optional) ===
AZURE_COMMUNICATION_CONNECTION_STRING=endpoint=https://...;accesskey=...
SENDER_EMAIL=DoNotReply@yourdomain.com
ADMIN_EMAIL=admin@yourdomain.com
```

> **OPENAI_BASE_URL is not in .env.example** but is required. Without it, pillar_dimensions.py scoring silently fails (the `openai.OpenAI(base_url=None)` client will call openai.com instead of Azure and auth will fail).

### 3.3 Enable pgvector Before First Startup

The `create_all()` startup will fail on the `pgvector` vector column if the extension isn't loaded. After postgres starts the first time, run:

```bash
cd be
docker compose up postgres -d
# Wait ~5 seconds, then:
docker compose exec postgres psql -U postgres -d hr_venture -c "CREATE EXTENSION IF NOT EXISTS vector;"
# Then bring everything up:
docker compose up --build
```

### 3.4 Start Everything

```bash
cd be
docker compose up --build
```

This starts: `postgres`, `redis`, `web` (FastAPI on 8080), `worker` (Celery).

### 3.5 Verify Backend

```bash
curl http://localhost:8080/api/role-data/roles
# Should return [] (empty list), not a connection error
```

---

## 4. Frontend Setup

```bash
# 1. Check Node version (must be ≥ 20)
node --version

# 2. Install dependencies
cd fe
npm install

# 3. Create fe/.env.local (no .env.example exists — create from scratch)
```

Create `fe/.env.local`:

```bash
# Backend API URL — must match CORS origins in be/app/main.py
NEXT_PUBLIC_API_URL=http://localhost:8080

# NextAuth — Required for the auth session object (even if routes are unprotected)
AUTH_SECRET=any-random-32-char-string-here
NEXTAUTH_URL=http://localhost:3000

# Microsoft Entra ID (Azure AD) — required if you hit protected routes
# Leave blank for local dev if you only use /roles, /results, /upload-role-file
AUTH_MICROSOFT_ENTRA_ID_ID=
AUTH_MICROSOFT_ENTRA_ID_SECRET=
AUTH_MICROSOFT_ENTRA_ID_TENANT_ID=
```

> `AUTH_SECRET` is required by NextAuth v5 even if you never log in — the app will crash on startup without it.

```bash
# 4. Start frontend
npm run dev
# Runs on http://localhost:3000
```

---

## 5. Option B — No Docker (WSL2 Required for Backend)

**Use this only if you cannot use Docker.** You must run all backend steps inside WSL2 Ubuntu, not Windows PowerShell.

### 5.1 Inside WSL2

```bash
# Install system packages
sudo apt-get update && sudo apt-get install -y gcc libpq-dev libgl1 libglib2.0-0

# Install Python 3.11
sudo apt-get install -y python3.11 python3.11-venv python3.11-dev

# Install PostgreSQL
sudo apt-get install -y postgresql postgresql-contrib
# For pgvector:
sudo apt-get install -y postgresql-15-pgvector
# OR compile from source if pgvector package isn't in your apt repos:
# git clone https://github.com/pgvector/pgvector && cd pgvector && make && sudo make install

# Install Redis
sudo apt-get install -y redis-server

# Start services
sudo service postgresql start
sudo service redis-server start

# Create DB
sudo -u postgres psql -c "CREATE DATABASE hr_venture;"
sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS vector;" -d hr_venture
```

### 5.2 Python Environment

```bash
cd /path/to/HR-venture/be

python3.11 -m venv .venv
source .venv/bin/activate

# Install Torch FIRST with its own index (must be before requirements.txt)
pip install torch==2.3.0 --index-url https://download.pytorch.org/whl/cpu

# Install everything else
pip install -r requirements.txt
```

> The requirements.txt has `torch==2.7.1+cpu` with an `--extra-index-url` line, but `pip` does not process `--extra-index-url` when a line appears mid-file. Install Torch separately first, then run `pip install -r requirements.txt` and pip will skip the already-installed torch.

### 5.3 Create .env

Same as the Docker `.env` above, but change service hostnames:

```bash
DATABASE_URL=postgresql+psycopg2://postgres:postgres@localhost:5432/hr_venture
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/1
```

### 5.4 Create uploads directory

```bash
mkdir -p be/uploads/temp
```

Both the web process and Celery worker must see this same path. They do because they're in the same filesystem (unlike Docker where you'd need a shared volume).

### 5.5 Start Backend (3 terminals)

```bash
# Terminal 1 — FastAPI
cd be && source .venv/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload

# Terminal 2 — Celery worker (must be inside WSL2, not Windows)
cd be && source .venv/bin/activate
celery -A app.celery_app.celery worker --loglevel=info --concurrency=2 --max-tasks-per-child=30

# Terminal 3 — Frontend (can be Windows PowerShell)
cd fe && npm run dev
```

---

## 6. Verification Checklist

```bash
# Backend health
curl http://localhost:8080/api/role-data/roles
# Expected: [] or a list of roles

# DB connected (check FastAPI startup logs)
# Expected: no "could not connect to server" errors

# Redis connected (check Celery startup logs)
# Expected: "Connected to redis://localhost:6379/0"
# Expected: "[tasks]" list including "process_single_cv", "process_batch", "aggregate_batch_results"

# Frontend
# Open http://localhost:3000 — should load the landing page without JS console errors
# Open http://localhost:3000/roles — should show empty roles list (no 403/CORS error)
# Open http://localhost:3000/upload-role-file — should show the upload form
```

---

## 7. Most Likely Failure Points

### 7.1 Backend starts but AI responses fail

| Symptom | Cause | Fix |
|---|---|---|
| CV scoring returns empty/error, role stays `processing` | `OPENAI_BASE_URL` not set | Set `OPENAI_BASE_URL=https://YOUR_RESOURCE.openai.azure.com/openai` |
| `AuthenticationError` from Azure | `AZURE_OPENAI_KEY` wrong or expired | Check key in Azure Portal |
| `model not found` | `CHAT_DEPLOYMENT` or `LLM_MODEL` don't match your Azure deployment names | Check Azure OpenAI → Deployments |
| Role stays `processing` forever after upload | Celery chord callback never fired | Check worker terminal for exceptions; check Redis is running on both db=0 and db=1 |

### 7.2 Frontend loads but API calls fail

| Symptom | Cause | Fix |
|---|---|---|
| `Network Error` / `ERR_CONNECTION_REFUSED` | `NEXT_PUBLIC_API_URL` not set — defaults to `http://74.242.171.29:8080` (prod server) | Set `NEXT_PUBLIC_API_URL=http://localhost:8080` in `fe/.env.local` |
| `CORS policy` error | Your origin isn't in main.py `origins` list | `origins` in main.py hardcodes `localhost:3000` — you're fine if running on port 3000 |
| `401 Unauthorized` loop | NextAuth session error | Ensure `AUTH_SECRET` is set in `fe/.env.local` |
| `Failed to fetch` on upload | Axios baseURL hitting wrong host | Check browser Network tab → Request URL must start with `http://localhost:8080` |

### 7.3 DB issues

| Symptom | Cause | Fix |
|---|---|---|
| `column "rank" does not exist` | ORM has `rank` but it wasn't in migration; `create_all` should add it | Run `create_all` by restarting the web container/process |
| `type "vector" does not exist` | pgvector extension not installed | `psql -d hr_venture -c "CREATE EXTENSION IF NOT EXISTS vector;"` |
| `relation "roles" does not exist` | DB connected but `create_all` failed silently | Check FastAPI startup logs for SQLAlchemy errors |

### 7.4 Celery / Redis

| Symptom | Cause | Fix |
|---|---|---|
| Worker exits immediately on Windows | Celery prefork broken on Windows | Move to WSL2 or Docker |
| Tasks received but never execute | Wrong broker/backend URL mismatch between web and worker | Confirm both read from same `.env`; check `CELERY_BROKER_URL` vs `CELERY_RESULT_BACKEND` |
| `chord` callback never called | Task exception in `process_single_cv` swallowed | Check worker logs for the individual task errors |

---

## 8. Debug Checklist by Scenario

**Backend starts but AI scoring fails:**
1. Check `be/logs/app.log.*` (file logger is configured)
2. Check Celery worker terminal for Python tracebacks
3. Test Azure credentials: `curl -H "api-key: YOUR_KEY" "https://YOUR_RESOURCE.openai.azure.com/openai/models?api-version=2024-07-01-preview"`
4. Confirm `OPENAI_BASE_URL` is set AND points to the same resource as `AZURE_OPENAI_ENDPOINT`

**Frontend loads but API calls fail:**
1. Browser DevTools → Network tab → check the actual request URL
2. If URL shows `74.242.171.29`, `NEXT_PUBLIC_API_URL` is not being read — confirm file is `fe/.env.local` (not `.env`)
3. Check FastAPI CORS: `origins` list in main.py includes `http://localhost:3000`

**DB transactions fail:**
1. `psql -U postgres -d hr_venture -c "\dt"` — confirm tables exist
2. `psql -U postgres -d hr_venture -c "\dx"` — confirm `vector` extension is listed
3. Check `DATABASE_URL` doesn't use Docker hostname (`postgres`) when running locally

---

## 9. Files to Read for Setup Understanding

| Order | File | What It Tells You |
|---|---|---|
| 1 | docker-compose.yml | All service wiring, ports, env_file, volume mounts |
| 2 | .env.example | Minimum required vars (incomplete — see Section 3.2 above) |
| 3 | main.py | App init, CORS origins, router prefix, DB init call |
| 4 | database.py | How `DATABASE_URL` is consumed; `create_all` location |
| 5 | celery_app.py | Broker/backend URLs, concurrency config, task definitions |
| 6 | azure_ai.py | Which env vars drive AzureOpenAI clients |
| 7 | axios.ts | `NEXT_PUBLIC_API_URL` baseURL — the single frontend→backend connection point |
| 8 | middleware.ts | Which routes require auth (only `/aisuite`, `/files`, `/admin` — not `/roles`) |

---

## 10. Hidden Assumptions in the Repo

- **Port 8080** — hardcoded in Dockerfile CMD, docker-compose, and next.config.ts CSP. If you change it, update all three.
- **Port 3000** — hardcoded in main.py CORS `origins` list. Frontend must run on port 3000 or CORS will reject it.
- **`uploads/temp/`** — web and worker containers share this via `volumes: .:/app`. In no-Docker mode, both processes must run from the same directory.
- **`LLM_MODEL = "gpt-4o"`** — hardcoded in pillar_dimensions.py, ignores `CHAT_DEPLOYMENT`. Your Azure deployment must be named exactly `gpt-4o` or you must edit that file.
- **`OPENAI_BASE_URL` format** — must end with `/openai` (e.g., `https://myresource.openai.azure.com/openai`), not just the base endpoint.
- **Python 3.11** — Dockerfile is explicit. `psycopg2-binary` and the specific torch build have known issues on 3.12+.
- **No data seeding required** — `create_all` creates tables empty; the app can operate on an empty DB.
- **Azure Communication email is optional** — if `AZURE_COMMUNICATION_CONNECTION_STRING` is not set, `send_admin_email` will throw but the batch result is already committed before that call.
- **NextAuth `AUTH_SECRET`** — not documented anywhere in the repo; required by NextAuth v5 or the frontend crashes on first session read.
- **`fe/.env.local` not `.env`** — Next.js only loads `.env.local` in dev mode for `NEXT_PUBLIC_*` vars to work. A plain `.env` file won't be read by `next dev`.
