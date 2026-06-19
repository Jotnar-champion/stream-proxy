# Production-Grade AI-Powered HR Platform — Complete Architecture & Implementation Guide

---

## 1. System Architecture

### High-Level Service Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL CLIENTS                                    │
│   Next.js (Browser)  ←→  WebSocket / SSE / REST / FCM Push                 │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ HTTPS / WSS
┌─────────────────────────────▼───────────────────────────────────────────────┐
│                        API GATEWAY (Nginx / Traefik)                        │
│          Rate Limiting · TLS Termination · Load Balancing                   │
└────┬───────────────┬───────────────┬────────────────┬────────────────────────┘
     │               │               │                │
     ▼               ▼               ▼                ▼
┌─────────┐   ┌──────────┐   ┌────────────┐   ┌─────────────────┐
│  Auth   │   │  Core    │   │   RAG /    │   │  Notification   │
│ Service │   │  API     │   │  LLM       │   │  Service        │
│(FastAPI)│   │(FastAPI) │   │  Worker    │   │  (FastAPI +     │
│         │   │          │   │  (Consumer)│   │   Consumers)    │
└────┬────┘   └────┬─────┘   └─────┬──────┘   └────────┬────────┘
     │             │               │                    │
     └──────┬──────┘               │                    │
            │                      │                    │
     ┌──────▼──────────────────────▼────────────────────▼──────┐
     │                   Apache Kafka Cluster                    │
     │  resume-upload-events │ match-request-events             │
     │  match-result-events  │ notification-dispatch-events     │
     │  DLQ topics per above │                                  │
     └──────────────────────────────────────────────────────────┘
            │
     ┌──────▼──────────────────────────────────────────────────┐
     │                    Data Layer                            │
     │  PostgreSQL (RDS)  │  ChromaDB  │  Redis Cluster        │
     └──────────────────────────────────────────────────────────┘
            │
     ┌──────▼──────────────────────────────────────────────────┐
     │               AI / ML Layer                              │
     │  HuggingFace Embedding Model (sentence-transformers)    │
     │  HuggingFace LLM (Mistral / LLaMA via vLLM or TGI)    │
     │  LangChain / LlamaIndex RAG Orchestration               │
     └──────────────────────────────────────────────────────────┘
            │
     ┌──────▼──────────────────────────────────────────────────┐
     │           External Integrations                          │
     │  SendGrid (Email)  │  FCM (Push)  │  S3/MinIO (Files)  │
     └──────────────────────────────────────────────────────────┘
```

### Service Responsibilities

| Service | Responsibility |
|---|---|
| **Auth Service** | JWT issue/refresh/revoke, RBAC enforcement, user management |
| **Core API** | Resume upload, job CRUD, application management, REST endpoints |
| **RAG/LLM Worker** | Kafka consumer: chunk→embed→ChromaDB→match→LLM→result |
| **Notification Service** | Consumes `match-result-events`, threshold check, dispatches via WS/SSE/Email |
| **WebSocket Manager** | Per-user WS connections, Redis Pub/Sub fan-out across instances |

---

## 2. RBAC Design

### Role Hierarchy & Permission Matrix

```
SuperAdmin > HR > Applicant
```

| Route | Applicant | HR | SuperAdmin |
|---|---|---|---|
| POST /resume/upload | ✅ | ❌ | ✅ |
| GET /application/status | ✅ (own) | ❌ | ✅ |
| GET /applicants | ❌ | ✅ (assigned roles) | ✅ |
| GET /match/{applicant_id}/{job_id} | ❌ | ✅ (assigned) | ✅ |
| POST /jobs | ❌ | ✅ | ✅ |
| POST /roles | ❌ | ❌ | ✅ |
| PUT /users/{id}/role | ❌ | ❌ | ✅ |
| POST /hr/assign-role | ❌ | ❌ | ✅ |
| GET /notifications | ❌ | ✅ (own) | ✅ |
| PATCH /notifications/{id}/read | ❌ | ✅ (own) | ✅ |

### JWT Token Structure

```json
{
  "sub": "user_uuid",
  "email": "hr@company.com",
  "role": "HR",
  "assigned_job_role_ids": [1, 3, 7],
  "iat": 1718800000,
  "exp": 1718886400,
  "jti": "unique-token-id"
}
```

### FastAPI RBAC Guard

```python
# app/core/security.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import jwt, JWTError
from app.core.config import settings

bearer_scheme = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    redis: Redis = Depends(get_redis),
):
    token = credentials.credentials
    # Check token revocation in Redis
    if await redis.get(f"revoked_token:{token}"):
        raise HTTPException(status_code=401, detail="Token revoked")
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
        return payload
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_role(*roles: str):
    async def role_checker(current_user: dict = Depends(get_current_user)):
        if current_user["role"] not in roles:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return role_checker

def require_job_assignment(job_role_id: int):
    """Ensures HR can only access applicants for their assigned roles."""
    async def checker(current_user: dict = Depends(require_role("HR", "SuperAdmin"))):
        if current_user["role"] == "SuperAdmin":
            return current_user
        if job_role_id not in current_user.get("assigned_job_role_ids", []):
            raise HTTPException(status_code=403, detail="Not assigned to this job role")
        return current_user
    return checker

# Usage in routes:
@router.get("/match/{applicant_id}/{job_id}")
async def get_match(
    applicant_id: str,
    job_id: int,
    current_user: dict = Depends(require_job_assignment(job_id))
):
    ...
```

### HR ↔ Job Role Assignment Schema

```sql
-- hr_job_assignments table (see full schema in section 7)
CREATE TABLE hr_job_assignments (
    id          SERIAL PRIMARY KEY,
    hr_user_id  UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    job_role_id INT  NOT NULL REFERENCES job_roles(id) ON DELETE CASCADE,
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(hr_user_id, job_role_id)
);
```

**Enforcement**: On login, the Auth Service queries `hr_job_assignments` and embeds `assigned_job_role_ids` into the JWT. On protected endpoints, `require_job_assignment()` validates this claim.

### Next.js `middleware.ts` Route Protection

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server'
import { jwtVerify } from 'jose'

const ROLE_ROUTES: Record<string, string[]> = {
  '/hr':    ['HR', 'SuperAdmin'],
  '/admin': ['SuperAdmin'],
  '/applicant': ['Applicant', 'SuperAdmin'],
}

export async function middleware(req: NextRequest) {
  const token = req.cookies.get('access_token')?.value
  if (!token) return NextResponse.redirect(new URL('/login', req.url))

  try {
    const { payload } = await jwtVerify(
      token,
      new TextEncoder().encode(process.env.JWT_SECRET!)
    )
    const role = payload.role as string
    const path = req.nextUrl.pathname

    for (const [prefix, allowed] of Object.entries(ROLE_ROUTES)) {
      if (path.startsWith(prefix) && !allowed.includes(role)) {
        return NextResponse.redirect(new URL('/unauthorized', req.url))
      }
    }
    return NextResponse.next()
  } catch {
    return NextResponse.redirect(new URL('/login', req.url))
  }
}

export const config = { matcher: ['/hr/:path*', '/admin/:path*', '/applicant/:path*'] }
```

---

## 3. Atomicity & Transaction Design

### Atomic Resume Upload Flow (Saga Pattern)

```
Step 1: Save file to S3/MinIO              → Compensate: delete S3 object
Step 2: PDFLoader → extract text            → Compensate: (in-memory, no rollback needed)
Step 3: Chunk text (RecursiveCharacterSplit) → Compensate: (in-memory)
Step 4: Embed chunks (HuggingFace)          → Compensate: (in-memory)
Step 5: INSERT to ChromaDB                  → Compensate: delete collection entries by resume_id
Step 6: INSERT to PostgreSQL (applications) → Compensate: DELETE WHERE id=application_id
Step 7: Publish to resume-upload-events     → Compensate: publish to resume-rollback-events
```

```python
# app/services/resume_saga.py
import uuid
from contextlib import asynccontextmanager

class ResumeUploadSaga:
    def __init__(self, db, chroma, s3, kafka_producer):
        self.db = db
        self.chroma = chroma
        self.s3 = s3
        self.producer = kafka_producer
        self.compensations = []  # Stack of rollback callables

    async def execute(self, file: UploadFile, user_id: str, job_id: int):
        idempotency_key = f"resume_upload:{user_id}:{job_id}"
        
        # Idempotency check
        async with self.db.transaction():
            existing = await self.db.fetchrow(
                "SELECT id FROM applications WHERE idempotency_key = $1", idempotency_key
            )
            if existing:
                return {"status": "already_processed", "application_id": existing["id"]}

        application_id = str(uuid.uuid4())
        s3_key = None
        chroma_ids = []

        try:
            # Step 1: Upload to S3
            s3_key = f"resumes/{application_id}/{file.filename}"
            await self.s3.upload(file, s3_key)
            self.compensations.append(lambda: self.s3.delete(s3_key))

            # Step 2–4: Parse, chunk, embed (pure computation, no external state)
            text = await parse_pdf(file)
            chunks = chunk_text(text)
            embeddings = embed_chunks(chunks)

            # Step 5: ChromaDB insert
            chroma_ids = [str(uuid.uuid4()) for _ in chunks]
            await self.chroma.add(
                ids=chroma_ids,
                embeddings=embeddings,
                documents=chunks,
                metadatas=[{"resume_id": application_id, "job_id": job_id} for _ in chunks]
            )
            self.compensations.append(
                lambda: self.chroma.delete(where={"resume_id": application_id})
            )

            # Step 6: PostgreSQL insert (atomic with idempotency key)
            async with self.db.transaction():
                await self.db.execute("""
                    INSERT INTO applications
                        (id, user_id, job_id, s3_key, idempotency_key, status)
                    VALUES ($1, $2, $3, $4, $5, 'uploaded')
                """, application_id, user_id, job_id, s3_key, idempotency_key)
            self.compensations.append(
                lambda: self.db.execute("DELETE FROM applications WHERE id=$1", application_id)
            )

            # Step 7: Publish to Kafka
            await self.producer.send("resume-upload-events", {
                "application_id": application_id,
                "job_id": job_id,
                "user_id": user_id,
                "idempotency_key": idempotency_key
            })

            return {"status": "success", "application_id": application_id}

        except Exception as e:
            # Execute compensations in reverse order
            for compensate in reversed(self.compensations):
                try:
                    await compensate()
                except Exception as comp_err:
                    logger.error(f"Compensation failed: {comp_err}")
            raise HTTPException(status_code=500, detail=f"Upload saga failed: {str(e)}")
```

---

## 4. Kafka Pipeline Design

### Topic Architecture

```
resume-upload-events         partitions=12, replication=3, retention=7d
  └── DLQ: resume-upload-events.DLQ

match-request-events         partitions=12, replication=3, retention=7d
  └── DLQ: match-request-events.DLQ

match-result-events          partitions=6,  replication=3, retention=30d
  └── DLQ: match-result-events.DLQ

notification-dispatch-events partitions=6,  replication=3, retention=7d
  └── DLQ: notification-dispatch-events.DLQ
```

### Message Schemas

```python
# resume-upload-events
{
    "event_id": "uuid",
    "correlation_id": "uuid",       # Trace end-to-end
    "application_id": "uuid",
    "job_id": 42,
    "user_id": "uuid",
    "idempotency_key": "str",
    "timestamp": "ISO8601",
    "retry_count": 0
}

# match-request-events
{
    "event_id": "uuid",
    "correlation_id": "uuid",
    "application_id": "uuid",
    "job_id": 42,
    "triggered_by": "hr_user_id | system",
    "timestamp": "ISO8601"
}

# match-result-events
{
    "event_id": "uuid",
    "correlation_id": "uuid",
    "application_id": "uuid",
    "job_id": 42,
    "applicant_name": "Jane Doe",
    "job_role": "Backend Engineer",
    "match_score": 87,
    "top_skills": ["FastAPI", "PostgreSQL"],
    "gaps": ["Kubernetes"],
    "recommendation": "Strong hire",
    "review_url": "/hr/applicants/{applicant_id}",
    "timestamp": "ISO8601"
}
```

### Producer Setup

```python
# app/kafka/producer.py
from aiokafka import AIOKafkaProducer
import json, uuid
from datetime import datetime, timezone

class KafkaEventProducer:
    def __init__(self):
        self.producer = AIOKafkaProducer(
            bootstrap_servers=settings.KAFKA_BROKERS,
            value_serializer=lambda v: json.dumps(v).encode(),
            compression_type="gzip",
            enable_idempotence=True,          # Exactly-once semantics
            acks="all",
            retries=5,
        )

    async def publish(self, topic: str, payload: dict, key: str | None = None):
        payload.setdefault("event_id", str(uuid.uuid4()))
        payload.setdefault("timestamp", datetime.now(timezone.utc).isoformat())
        await self.producer.send_and_wait(
            topic,
            value=payload,
            key=key.encode() if key else None,  # Key=job_id ensures ordering per job
        )
```

### Consumer with DLQ & Retry

```python
# app/kafka/consumers/match_worker.py
from aiokafka import AIOKafkaConsumer
import asyncio, json

class MatchRequestConsumer:
    MAX_RETRIES = 3

    async def start(self):
        consumer = AIOKafkaConsumer(
            "match-request-events",
            bootstrap_servers=settings.KAFKA_BROKERS,
            group_id="match-worker-group",
            enable_auto_commit=False,
            auto_offset_reset="earliest",
            value_deserializer=lambda v: json.loads(v),
        )
        await consumer.start()
        try:
            async for msg in consumer:
                await self._process_with_retry(msg)
                await consumer.commit()
        finally:
            await consumer.stop()

    async def _process_with_retry(self, msg):
        payload = msg.value
        retry_count = payload.get("retry_count", 0)
        try:
            await self._handle_match_request(payload)
        except Exception as e:
            logger.error(f"[{payload['correlation_id']}] Match failed: {e}")
            if retry_count < self.MAX_RETRIES:
                payload["retry_count"] = retry_count + 1
                await producer.publish("match-request-events", payload)
            else:
                # Send to Dead Letter Queue
                await producer.publish("match-request-events.DLQ", {
                    **payload,
                    "failure_reason": str(e),
                    "failed_at": datetime.utcnow().isoformat()
                })

    async def _handle_match_request(self, payload: dict):
        # 1. Check Redis cache first
        cache_key = f"match:{payload['application_id']}:{payload['job_id']}"
        cached = await redis.get(cache_key)
        if cached:
            result = json.loads(cached)
        else:
            # 2. Run RAG + LLM
            result = await rag_pipeline.run(
                application_id=payload["application_id"],
                job_id=payload["job_id"],
                correlation_id=payload["correlation_id"]
            )
            # 3. Cache result
            await redis.setex(cache_key, 3600, json.dumps(result))

        # 4. Persist to PostgreSQL
        await db.execute("""
            INSERT INTO match_results (application_id, job_id, score, result_json, correlation_id)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (application_id, job_id) DO UPDATE SET
                score=EXCLUDED.score, result_json=EXCLUDED.result_json
        """, payload["application_id"], payload["job_id"],
            result["score"], json.dumps(result), payload["correlation_id"])

        # 5. Publish match-result-events
        await producer.publish("match-result-events", {
            "correlation_id": payload["correlation_id"],
            "application_id": payload["application_id"],
            "job_id": payload["job_id"],
            **result
        }, key=str(payload["job_id"]))
```

---

## 5. HR Notification System

### Notification Service Architecture

```
match-result-events (Kafka)
        │
        ▼
NotificationConsumer.consume()
        │
        ├─ score >= threshold? (e.g., 75)
        │      │ YES
        │      ▼
        ├─ lookup hr_job_assignments → get HR user IDs
        │
        ├─ lookup hr_notification_preferences per HR user
        │
        ├─ check digest mode:
        │      ├─ IMMEDIATE → dispatch now
        │      └─ DIGEST    → buffer in Redis list, schedule flush
        │
        └─ dispatch per channel:
               ├─ WebSocket  → Redis Pub/Sub channel "notif:{hr_user_id}"
               ├─ SSE        → Redis Pub/Sub (same channel, SSE consumers subscribed)
               ├─ Email      → SendGrid async task
               └─ In-App DB  → INSERT notifications + INCR Redis unread count
```

### Notification Service Consumer

```python
# app/notifications/service.py
class NotificationService:
    MATCH_THRESHOLD = 75

    async def consume(self):
        consumer = AIOKafkaConsumer(
            "match-result-events",
            bootstrap_servers=settings.KAFKA_BROKERS,
            group_id="notification-service-group",
            value_deserializer=lambda v: json.loads(v),
        )
        await consumer.start()
        async for msg in consumer:
            await self._handle_match_result(msg.value)

    async def _handle_match_result(self, event: dict):
        if event["match_score"] < self.MATCH_THRESHOLD:
            return

        hr_users = await self._get_assigned_hr_users(event["job_id"])

        for hr_user in hr_users:
            prefs = await self._get_preferences(hr_user["id"], event["job_id"])
            if prefs["muted"]:
                continue

            payload = self._build_payload(event, hr_user)
            await self._persist_notification(payload, hr_user["id"])

            if prefs["digest_mode"] == "immediate":
                await self._dispatch_all(payload, hr_user, prefs)
            else:
                await self._buffer_for_digest(payload, hr_user, prefs["digest_mode"])

    def _build_payload(self, event: dict, hr_user: dict) -> dict:
        return {
            "type": "NEW_MATCH",
            "job_role": event["job_role"],
            "applicant_name": event["applicant_name"],
            "match_score": event["match_score"],
            "top_skills": event["top_skills"],
            "gaps": event["gaps"],
            "review_url": f"/hr/applicants/{event['application_id']}",
            "timestamp": event["timestamp"],
            "hr_user_id": hr_user["id"],
            "application_id": event["application_id"],
            "job_id": event["job_id"],
        }

    async def _dispatch_all(self, payload: dict, hr_user: dict, prefs: dict):
        tasks = []
        if prefs["in_app_enabled"]:
            tasks.append(self._push_websocket(payload, hr_user["id"]))
        if prefs["email_enabled"]:
            tasks.append(self._send_email(payload, hr_user["email"]))
        await asyncio.gather(*tasks, return_exceptions=True)

    async def _push_websocket(self, payload: dict, hr_user_id: str):
        # Fan-out via Redis Pub/Sub (supports multiple FastAPI instances)
        await redis.publish(
            f"notif:{hr_user_id}",
            json.dumps(payload)
        )

    async def _persist_notification(self, payload: dict, hr_user_id: str):
        notif_id = str(uuid.uuid4())
        await db.execute("""
            INSERT INTO notifications
                (id, hr_user_id, type, payload, is_read, created_at)
            VALUES ($1, $2, $3, $4, false, NOW())
        """, notif_id, hr_user_id, "NEW_MATCH", json.dumps(payload))
        # Increment Redis unread count
        await redis.incr(f"notif:unread:{hr_user_id}")
```

### WebSocket Endpoint with Redis Pub/Sub Fan-Out

```python
# app/api/websocket.py
from fastapi import WebSocket, WebSocketDisconnect
import asyncio, json

@router.websocket("/ws/notifications/{hr_user_id}")
async def ws_notifications(websocket: WebSocket, hr_user_id: str):
    token = websocket.query_params.get("token")
    user = await verify_ws_token(token, hr_user_id)  # Validate JWT

    await websocket.accept()

    # Subscribe to Redis Pub/Sub channel for this user
    pubsub = redis.pubsub()
    await pubsub.subscribe(f"notif:{hr_user_id}")

    async def read_redis():
        async for message in pubsub.listen():
            if message["type"] == "message":
                await websocket.send_text(message["data"].decode())

    async def read_client():
        while True:
            data = await websocket.receive_text()
            # Handle client ping/ack
            if data == "ping":
                await websocket.send_text("pong")

    try:
        await asyncio.gather(read_redis(), read_client())
    except WebSocketDisconnect:
        await pubsub.unsubscribe(f"notif:{hr_user_id}")
```

### SSE Fallback Endpoint

```python
# app/api/sse.py
from fastapi.responses import StreamingResponse
import asyncio, json

@router.get("/notifications/stream/{hr_user_id}")
async def sse_notifications(hr_user_id: str, current_user = Depends(require_role("HR"))):
    async def event_stream():
        pubsub = redis.pubsub()
        await pubsub.subscribe(f"notif:{hr_user_id}")
        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    data = message["data"].decode()
                    yield f"data: {data}\n\n"
                    await asyncio.sleep(0)  # Yield control
        except asyncio.CancelledError:
            await pubsub.unsubscribe(f"notif:{hr_user_id}")

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Nginx: disable buffering
        }
    )
```

### Email Notification (SendGrid)

```python
# app/notifications/email.py
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail

MATCH_TEMPLATE_ID = "d-your-sendgrid-template-id"

async def send_match_email(payload: dict, recipient_email: str):
    message = Mail(
        from_email="noreply@hrplatform.com",
        to_emails=recipient_email,
    )
    message.template_id = MATCH_TEMPLATE_ID
    message.dynamic_template_data = {
        "applicant_name": payload["applicant_name"],
        "job_role": payload["job_role"],
        "match_score": payload["match_score"],
        "top_skills": ", ".join(payload["top_skills"]),
        "gaps": ", ".join(payload["gaps"]),
        "review_url": f"{settings.FRONTEND_URL}{payload['review_url']}",
        "cta_text": "Review Applicant",
    }
    sg = SendGridAPIClient(api_key=settings.SENDGRID_API_KEY)
    sg.client.mail.send.post(request_body=message.get())
```

### Digest Batching with Redis

```python
# app/notifications/digest.py
DIGEST_KEY = "digest:{hr_user_id}:{job_id}"

async def buffer_for_digest(payload: dict, hr_user_id: str, job_id: int, mode: str):
    key = f"digest:{hr_user_id}:{job_id}"
    await redis.rpush(key, json.dumps(payload))

    # Set TTL based on mode
    ttl_map = {"hourly": 3600, "daily": 86400}
    ttl = ttl_map.get(mode, 3600)
    await redis.expire(key, ttl)

# Celery or APScheduler task to flush digests
async def flush_digests():
    keys = await redis.keys("digest:*")
    for key in keys:
        items = await redis.lrange(key, 0, -1)
        if not items:
            continue
        payloads = [json.loads(i) for i in items]
        hr_user_id = key.split(":")[1]
        user = await db.fetchrow("SELECT email FROM users WHERE id=$1", hr_user_id)
        await send_digest_email(payloads, user["email"])
        await redis.delete(key)
```

---

## 6. Redis Caching Strategy

| Cache Key Pattern | Type | TTL | Strategy | Invalidation |
|---|---|---|---|---|
| `match:{resume_id}:{job_id}` | String (JSON) | 1h | Cache-aside | On re-match request |
| `jobs:active` | String (JSON list) | 5m | Write-through | On POST/PUT /jobs |
| `job:{job_id}` | String (JSON) | 10m | Cache-aside | On job update |
| `chroma:query:{hash}` | String (JSON) | 15m | Cache-aside | On new embeddings added |
| `notif:unread:{hr_user_id}` | Integer | No TTL | Write-through | DECR on read, INCR on new |
| `session:{jti}` | String | Token TTL | Write-through | On logout/revoke |
| `revoked_token:{jti}` | String | Remaining TTL | Write-through | Expires naturally |
| `digest:{hr_user_id}:{job_id}` | List | digest TTL | Append | On flush |

```python
# app/core/cache.py
class CacheService:
    async def get_or_set(self, key: str, ttl: int, fetch_fn):
        """Cache-aside pattern"""
        cached = await redis.get(key)
        if cached:
            return json.loads(cached)
        result = await fetch_fn()
        await redis.setex(key, ttl, json.dumps(result))
        return result

    async def invalidate_job_cache(self, job_id: int):
        """Called on job update — write-through invalidation"""
        await asyncio.gather(
            redis.delete(f"job:{job_id}"),
            redis.delete("jobs:active"),
        )
```

**Redis Eviction Policy**: `allkeys-lru` — ensures cache functions under memory pressure without data loss (LRU evicts least-recently-used keys globally).

---

## 7. Database Schema

### PostgreSQL

```sql
-- Users & Auth
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    hashed_pw   VARCHAR(255) NOT NULL,
    full_name   VARCHAR(255),
    role        VARCHAR(50) NOT NULL CHECK (role IN ('SuperAdmin','HR','Applicant')),
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Job Taxonomy
CREATE TABLE job_roles (
    id          SERIAL PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    department  VARCHAR(255),
    description TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE job_openings (
    id              SERIAL PRIMARY KEY,
    job_role_id     INT REFERENCES job_roles(id),
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    requirements    JSONB,                    -- ["Python","FastAPI","3+ years"]
    status          VARCHAR(50) DEFAULT 'open' CHECK (status IN ('open','closed','draft')),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    closed_at       TIMESTAMPTZ
);

-- Applications & Matching
CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    job_id          INT  REFERENCES job_openings(id),
    s3_key          VARCHAR(500),
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    status          VARCHAR(50) DEFAULT 'uploaded',
    submitted_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE match_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID REFERENCES applications(id) ON DELETE CASCADE,
    job_id          INT  REFERENCES job_openings(id),
    score           SMALLINT CHECK (score BETWEEN 0 AND 100),
    result_json     JSONB,    -- Full LLM output: strengths, gaps, recommendation
    correlation_id  UUID,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(application_id, job_id)
);

-- RBAC Assignments
CREATE TABLE hr_job_assignments (
    id          SERIAL PRIMARY KEY,
    hr_user_id  UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    job_role_id INT  NOT NULL REFERENCES job_roles(id) ON DELETE CASCADE,
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(hr_user_id, job_role_id)
);

-- Notifications
CREATE TABLE notifications (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hr_user_id  UUID REFERENCES users(id) ON DELETE CASCADE,
    type        VARCHAR(50) NOT NULL,
    payload     JSONB NOT NULL,
    is_read     BOOLEAN DEFAULT FALSE,
    read_at     TIMESTAMPTZ,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notifications_user_unread ON notifications(hr_user_id, is_read, created_at DESC);

-- Notification Preferences
CREATE TABLE hr_notification_preferences (
    id              SERIAL PRIMARY KEY,
    hr_user_id      UUID REFERENCES users(id) ON DELETE CASCADE,
    job_role_id     INT  REFERENCES job_roles(id) ON DELETE CASCADE,
    email_enabled   BOOLEAN DEFAULT TRUE,
    in_app_enabled  BOOLEAN DEFAULT TRUE,
    muted           BOOLEAN DEFAULT FALSE,
    digest_mode     VARCHAR(20) DEFAULT 'immediate'
                        CHECK (digest_mode IN ('immediate','hourly','daily')),
    score_threshold SMALLINT DEFAULT 75,
    UNIQUE(hr_user_id, job_role_id)
);
```

### ChromaDB Collection Design

```python
# Collection: "resumes"
# Each document = one text chunk from a resume
{
    "ids":        ["chunk_uuid_1", "chunk_uuid_2", ...],
    "embeddings": [[0.123, ...], ...],            # 384-dim MiniLM vectors
    "documents":  ["Experienced in FastAPI..."],   # Raw chunk text
    "metadatas":  [{
        "resume_id":       "application_uuid",
        "job_id":          42,
        "chunk_index":     0,
        "user_id":         "user_uuid",
        "source_filename": "jane_doe_cv.pdf",
        "page_number":     1,
    }]
}
```

### Redis Key Naming Conventions

```
match:{application_id}:{job_id}       → JSON string, TTL 3600s
job:{job_id}                          → JSON string, TTL 600s
jobs:active                           → JSON array, TTL 300s
chroma:query:{sha256_of_query_params} → JSON string, TTL 900s
notif:unread:{hr_user_id}             → integer counter
session:{jti}                         → user payload JSON
revoked_token:{jti}                   → "1", TTL = remaining token lifetime
digest:{hr_user_id}:{job_id}          → list of JSON payloads
```

---

## 8. FastAPI Endpoint Design

```python
# app/api/auth.py
POST   /auth/register              # Public
POST   /auth/login                 # Public → returns access + refresh tokens
POST   /auth/refresh               # Refresh token rotation
POST   /auth/logout                # Revokes JTI in Redis

# app/api/applicant.py
POST   /resume/upload              # Applicant; triggers Saga + Kafka publish
GET    /application/status         # Applicant; own applications only

# app/api/hr.py
GET    /applicants                 # HR; filtered by assigned job roles
GET    /match/{applicant_id}/{job_id}  # HR; triggers/retrieves match
POST   /jobs                       # HR/Admin; create job opening
GET    /jobs                       # HR; list with Redis cache
GET    /notifications              # HR; paginated from PostgreSQL
PATCH  /notifications/{id}/read    # HR; mark read + Redis DECR
GET    /notifications/unread-count # HR; from Redis
PUT    /notifications/preferences  # HR; update hr_notification_preferences

# app/api/admin.py
POST   /admin/roles                # SuperAdmin; create role
PUT    /admin/users/{id}/role      # SuperAdmin; assign role
POST   /admin/hr/assign-role       # SuperAdmin; assign HR to job role
GET    /admin/users                # SuperAdmin; list all users

# app/api/websocket.py
WS     /ws/notifications/{hr_user_id}     # HR; real-time push

# app/api/sse.py
GET    /notifications/stream/{hr_user_id} # HR; SSE fallback
```

---

## 9. RAG Pipeline & Prompt Template

### Pipeline (LangChain)

```python
# app/rag/pipeline.py
from langchain.vectorstores import Chroma
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.llms import HuggingFacePipeline
from langchain.prompts import PromptTemplate
from langchain.chains import RetrievalQA

class RAGMatchPipeline:
    def __init__(self):
        self.embeddings = HuggingFaceEmbeddings(
            model_name="sentence-transformers/all-MiniLM-L6-v2"
        )
        self.vectorstore = Chroma(
            collection_name="resumes",
            embedding_function=self.embeddings,
            persist_directory=settings.CHROMA_PERSIST_DIR,
        )
        self.llm = HuggingFacePipeline.from_model_id(
            model_id="mistralai/Mistral-7B-Instruct-v0.2",
            task="text-generation",
            pipeline_kwargs={"max_new_tokens": 1024, "temperature": 0.1},
        )

    async def run(self, application_id: str, job_id: int, correlation_id: str) -> dict:
        job = await db.fetchrow("SELECT * FROM job_openings WHERE id=$1", job_id)
        resume_chunks = self.vectorstore.similarity_search(
            query=job["description"],
            k=6,
            filter={"resume_id": application_id}
        )
        context = "\n\n---\n\n".join([c.page_content for c in resume_chunks])
        prompt = MATCH_PROMPT_TEMPLATE.format(
            job_title=job["title"],
            job_description=job["description"],
            job_requirements=", ".join(job["requirements"]),
            resume_context=context
        )
        raw_output = await asyncio.to_thread(self.llm, prompt)
        return self._parse_llm_output(raw_output)

    def _parse_llm_output(self, raw: str) -> dict:
        import re, json
        match = re.search(r'\{.*\}', raw, re.DOTALL)
        if not match:
            raise ValueError("LLM did not return valid JSON")
        return json.loads(match.group())
```

### Prompt Template

```python
MATCH_PROMPT_TEMPLATE = """
You are an expert HR evaluation assistant. Analyze the resume excerpts below against the job requirements and provide a structured evaluation.

## Job Details
**Title**: {job_title}
**Description**: {job_description}
**Required Skills/Experience**: {job_requirements}

## Resume Excerpts (retrieved relevant sections)
{resume_context}

## Instructions
Evaluate the candidate's fit for this role. Consider:
1. Technical skills alignment
2. Years of relevant experience
3. Project/domain relevance
4. Critical gaps that would block success in this role

Respond ONLY with a valid JSON object in the following exact format — no markdown, no explanation outside the JSON:

{{
  "score": <integer 0-100>,
  "strengths": [<list of 3-5 specific matching skills or experiences>],
  "gaps": [<list of missing or weak areas>],
  "recommendation": "<one of: Strong Hire | Hire | Maybe | No Hire>",
  "reasoning": "<2-3 sentence justification of the score and recommendation>"
}}
"""
```

**LLM Output** (becomes the Kafka `match-result-events` body):
```json
{
  "score": 87,
  "strengths": ["FastAPI", "PostgreSQL", "Docker", "REST API design"],
  "gaps": ["Kubernetes", "Terraform"],
  "recommendation": "Strong Hire",
  "reasoning": "Candidate demonstrates 4+ years of Python backend development with direct FastAPI and PostgreSQL experience matching 80% of requirements. Lacks DevOps/infra experience but core engineering skills are excellent."
}
```

---

## 10. HuggingFace Integration

### Embedding vs. Generation — Model Roles

| Concern | Model | Purpose |
|---|---|---|
| **Embedding** | `sentence-transformers/all-MiniLM-L6-v2` | Convert resume chunks + job descriptions to 384-dim vectors for semantic similarity search in ChromaDB |
| **Generation** | `mistralai/Mistral-7B-Instruct-v0.2` | Instruction-following LLM for structured match analysis and reasoning |

```python
# app/ml/embeddings.py
from sentence_transformers import SentenceTransformer
import numpy as np

class EmbeddingService:
    """Singleton — load once, reuse across requests"""
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.model = SentenceTransformer(
                "sentence-transformers/all-MiniLM-L6-v2",
                device="cuda" if torch.cuda.is_available() else "cpu"
            )
        return cls._instance

    def embed(self, texts: list[str]) -> list[list[float]]:
        return self.model.encode(texts, normalize_embeddings=True).tolist()

# app/ml/llm.py
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline
import torch

class LLMService:
    def __init__(self):
        model_id = "mistralai/Mistral-7B-Instruct-v0.2"
        self.tokenizer = AutoTokenizer.from_pretrained(model_id)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_id,
            torch_dtype=torch.float16,
            device_map="auto",            # Auto GPU/CPU assignment
            load_in_4bit=True,            # QLoRA quantization for memory efficiency
        )
        self.pipe = pipeline(
            "text-generation",
            model=self.model,
            tokenizer=self.tokenizer,
            max_new_tokens=1024,
            temperature=0.1,
            do_sample=True,
        )

    def generate(self, prompt: str) -> str:
        result = self.pipe(f"[INST] {prompt} [/INST]")
        return result[0]["generated_text"].split("[/INST]")[-1].strip()
```

**Text Chunking Strategy** (preserves semantic coherence):
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ". ", " ", ""]
)
```

---

## 11. Next.js Frontend Structure

### Directory Structure

```
src/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx
│   ├── applicant/
│   │   ├── dashboard/page.tsx
│   │   └── upload/page.tsx
│   ├── hr/
│   │   ├── dashboard/page.tsx
│   │   ├── jobs/page.tsx
│   │   ├── applicants/
│   │   │   └── [applicant_id]/page.tsx
│   │   ├── notifications/page.tsx
│   │   └── settings/
│   │       └── notifications/page.tsx
│   └── admin/
│       └── users/page.tsx
├── components/
│   ├── NotificationBell.tsx      ← Real-time bell with unread badge
│   ├── NotificationDropdown.tsx
│   └── MatchScoreCard.tsx
├── hooks/
│   └── useNotifications.ts       ← WebSocket hook
├── lib/
│   ├── api.ts                    ← Axios instance with interceptors
│   └── auth.ts                   ← Token helpers
└── middleware.ts                  ← Route protection by role
```

### `useNotifications` WebSocket Hook

```typescript
// hooks/useNotifications.ts
import { useEffect, useRef, useState, useCallback } from 'react'

interface Notification {
  type: string
  job_role: string
  applicant_name: string
  match_score: number
  top_skills: string[]
  gaps: string[]
  review_url: string
  timestamp: string
}

export function useNotifications(hrUserId: string) {
  const [notifications, setNotifications] = useState<Notification[]>([])
  const [unreadCount, setUnreadCount] = useState(0)
  const [connected, setConnected] = useState(false)
  const ws = useRef<WebSocket | null>(null)

  const connect = useCallback(() => {
    const token = localStorage.getItem('access_token')
    const url = `${process.env.NEXT_PUBLIC_WS_URL}/ws/notifications/${hrUserId}?token=${token}`
    ws.current = new WebSocket(url)

    ws.current.onopen = () => setConnected(true)

    ws.current.onmessage = (event) => {
      const notif: Notification = JSON.parse(event.data)
      setNotifications(prev => [notif, ...prev])
      setUnreadCount(prev => prev + 1)
    }

    ws.current.onclose = () => {
      setConnected(false)
      // Exponential backoff reconnect
      setTimeout(connect, 3000)
    }
  }, [hrUserId])

  useEffect(() => {
    connect()
    // Fetch initial unread count from REST
    fetch(`/api/notifications/unread-count`)
      .then(r => r.json())
      .then(d => setUnreadCount(d.count))

    return () => ws.current?.close()
  }, [connect])

  const markAsRead = async (notifId: string) => {
    await fetch(`/api/notifications/${notifId}/read`, { method: 'PATCH' })
    setUnreadCount(prev => Math.max(0, prev - 1))
  }

  return { notifications, unreadCount, connected, markAsRead }
}
```

### Notification Bell Component

```tsx
// components/NotificationBell.tsx
'use client'
import { Bell } from 'lucide-react'
import { useState } from 'react'
import { useNotifications } from '@/hooks/useNotifications'

export function NotificationBell({ hrUserId }: { hrUserId: string }) {
  const { notifications, unreadCount, markAsRead } = useNotifications(hrUserId)
  const [open, setOpen] = useState(false)

  return (
    <div className="relative">
      <button onClick={() => setOpen(!open)} className="relative p-2">
        <Bell className="h-6 w-6" />
        {unreadCount > 0 && (
          <span className="absolute -top-1 -right-1 bg-red-500 text-white text-xs
                           rounded-full h-5 w-5 flex items-center justify-center">
            {unreadCount > 99 ? '99+' : unreadCount}
          </span>
        )}
      </button>

      {open && (
        <div className="absolute right-0 mt-2 w-96 bg-white shadow-xl rounded-lg z-50 max-h-[480px] overflow-y-auto">
          <div className="p-4 border-b font-semibold text-gray-700">Notifications</div>
          {notifications.length === 0 ? (
            <p className="p-4 text-gray-500 text-sm">No notifications yet</p>
          ) : (
            notifications.map((n, i) => (
              <div key={i} onClick={() => markAsRead((n as any).id)}
                className="p-4 border-b hover:bg-gray-50 cursor-pointer">
                <div className="flex justify-between items-start">
                  <p className="font-medium text-sm">{n.applicant_name}</p>
                  <span className={`text-xs px-2 py-1 rounded-full font-bold
                    ${n.match_score >= 85 ? 'bg-green-100 text-green-700' :
                      n.match_score >= 70 ? 'bg-yellow-100 text-yellow-700' :
                                            'bg-red-100 text-red-700'}`}>
                    {n.match_score}%
                  </span>
                </div>
                <p className="text-xs text-gray-500 mt-1">{n.job_role}</p>
                <p className="text-xs text-gray-400 mt-1">
                  Skills: {n.top_skills.join(', ')}
                </p>
                <a href={n.review_url}
                   className="text-xs text-blue-600 hover:underline mt-1 block">
                  Review →
                </a>
              </div>
            ))
          )}
        </div>
      )}
    </div>
  )
}
```

---

## 12. End-to-End Data Flow

```
1. APPLICANT uploads resume PDF via POST /resume/upload
   │
2. ResumeUploadSaga begins:
   ├── S3/MinIO: store raw PDF
   ├── PDFLoader (LangChain/PyMuPDF): extract text
   ├── RecursiveCharacterTextSplitter: create 512-token chunks
   ├── EmbeddingService (MiniLM): generate 384-dim vectors
   ├── ChromaDB: store chunks + vectors + metadata
   └── PostgreSQL: INSERT applications (idempotency_key)
   │
3. Kafka PRODUCER publishes to resume-upload-events
   │
4. match-request-events triggered (either manually by HR or auto-scored):
   Kafka PRODUCER publishes to match-request-events
   │
5. RAG/LLM CONSUMER reads match-request-events:
   ├── Check Redis cache: match:{application_id}:{job_id}
   │   └── CACHE HIT → skip to step 8
   ├── ChromaDB: similarity_search(job_description, k=6, filter=resume_id)
   ├── Build prompt: resume_chunks + job_description
   ├── Mistral LLM: generate structured JSON
   ├── Parse result: {score, strengths, gaps, recommendation}
   └── Redis: SETEX match:{id}:{job_id} 3600 <result_json>
   │
6. Result PERSISTED to PostgreSQL match_results
   │
7. Kafka PRODUCER publishes to match-result-events:
   {application_id, job_id, match_score: 87, top_skills, gaps, ...}
   │
8. NOTIFICATION SERVICE consumes match-result-events:
   ├── score (87) >= threshold (75)? → YES
   ├── Query PostgreSQL hr_job_assignments → get HR user IDs [hr_A, hr_B]
   ├── For each HR user:
   │   ├── Query hr_notification_preferences
   │   ├── muted? → SKIP
   │   ├── digest_mode = "immediate"
   │   ├── INSERT notifications table (PostgreSQL)
   │   ├── Redis INCR notif:unread:{hr_user_id}
   │   ├── Redis PUBLISH notif:{hr_user_id} → <payload JSON>
   │   │     └── WebSocket/SSE subscribers receive push instantly
   │   └── SendGrid: async email with match summary + CTA
   │
9. HR sees real-time alert:
   ├── NotificationBell badge increments (unreadCount++)
   ├── Dropdown shows: "Jane Doe — Backend Engineer — 87% match"
   └── HR clicks → /hr/applicants/{applicant_id} review page
   │
10. HR marks as read: PATCH /notifications/{id}/read
    ├── PostgreSQL: UPDATE notifications SET is_read=true
    └── Redis: DECR notif:unread:{hr_user_id}
```

---

## 13. Non-Functional Considerations

### Horizontal Scaling

```yaml
# docker-compose.yml sketch (production: use Kubernetes)
services:
  api:
    image: hr-api
    replicas: 4                           # Scale API horizontally
    environment:
      - REDIS_URL=redis://redis-cluster:6379

  rag-worker:
    image: hr-rag-worker
    replicas: 3                           # One consumer group: match-worker-group
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: 1, capabilities: [gpu]}]

  notification-worker:
    image: hr-notification-worker
    replicas: 2                           # notification-service-group
```

**WebSocket Fan-Out at Scale**: All FastAPI instances subscribe to the same Redis Pub/Sub channel `notif:{hr_user_id}`. When Notification Service publishes, ALL instances receive it — whichever instance holds the user's WS connection will forward it. This is the correct pattern for multi-instance WS.

### ChromaDB Persistence & Backup

```python
# Use persistent mode (not in-memory)
chroma_client = chromadb.PersistentClient(path="/data/chroma")

# Backup: nightly tar of /data/chroma to S3
# OR use ChromaDB Cloud (managed) for production
```

### Rate Limiting (per role)

```python
# app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# Different limits per role
@router.post("/resume/upload")
@limiter.limit("10/hour")   # Applicants: 10 uploads/hour
async def upload_resume(...): ...

@router.get("/match/{applicant_id}/{job_id}")
@limiter.limit("100/hour")  # HR: 100 match requests/hour
async def get_match(...): ...
```

### Structured Logging with Correlation IDs

```python
# app/core/logging.py
import structlog

log = structlog.get_logger()

# Middleware: inject correlation_id into every request
@app.middleware("http")
async def correlation_middleware(request: Request, call_next):
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    structlog.contextvars.bind_contextvars(correlation_id=correlation_id)
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response

# Usage in Kafka consumers:
log.info("match_completed",
    correlation_id=event["correlation_id"],
    application_id=event["application_id"],
    job_id=event["job_id"],
    score=result["score"],
    duration_ms=elapsed
)
```

---

## Project Scaffolding Reference

```
hr-platform/
├── backend/
│   ├── app/
│   │   ├── api/          # Route handlers
│   │   ├── core/         # Config, security, logging
│   │   ├── db/           # PostgreSQL models, migrations (Alembic)
│   │   ├── kafka/        # Producers + consumers
│   │   ├── ml/           # Embedding + LLM services
│   │   ├── notifications/ # Notification service + channels
│   │   ├── rag/          # RAG pipeline
│   │   └── services/     # Business logic (saga, cache, etc.)
│   ├── alembic/          # DB migration files
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── app/          # Next.js App Router pages
│   │   ├── components/
│   │   ├── hooks/
│   │   └── lib/
│   ├── middleware.ts
│   └── package.json
├── infra/
│   ├── docker-compose.yml
│   ├── kafka/            # docker-compose for Kafka + ZooKeeper/KRaft
│   └── nginx/            # Reverse proxy config
└── docs/
    └── architecture.md
```

---

This guide covers all 13 sections with production-ready patterns. Key integration points to implement first:
1. **Auth + RBAC** (everything depends on this)
2. **Resume upload Saga** (core data pipeline)
3. **Kafka topics + RAG worker** (async processing backbone)
4. **Notification Service + WebSocket** (the high-value HR feature)


==============================================================================================================================================

Here are the most impactful improvements, organized by severity:

---

## 🔴 Critical Gaps

### 1. LLM Output is Fragile
The regex-based JSON parser `re.search(r'\{.*\}', raw)` will break on any LLM deviation. Use **[Instructor](https://github.com/jxnl/instructor)** or **Outlines** instead — they enforce structured output via Pydantic validation at generation time:

```python
import instructor
from pydantic import BaseModel, Field

class MatchResult(BaseModel):
    score: int = Field(ge=0, le=100)
    strengths: list[str] = Field(min_length=1, max_length=5)
    gaps: list[str]
    recommendation: Literal["Strong Hire", "Hire", "Maybe", "No Hire"]
    reasoning: str

client = instructor.from_transformers(pipeline)
result = client.chat.completions.create(
    response_model=MatchResult,
    messages=[{"role": "user", "content": prompt}],
    max_retries=3,   # Auto-retries if LLM output is invalid
)
```

### 2. JWT Contains Mutable State
Embedding `assigned_job_role_ids` in the JWT is dangerous — if an admin changes an HR user's assignments, the old token is still valid until expiry. **Fix**: Move to a lookup-at-request-time pattern with short token TTLs (15 min) + Redis cache for assignments:

```python
# On every protected request, validate assignments from Redis/DB, not JWT claims
async def get_hr_assignments(user_id: str, redis, db) -> list[int]:
    cached = await redis.get(f"hr_assignments:{user_id}")
    if cached:
        return json.loads(cached)
    rows = await db.fetch("SELECT job_role_id FROM hr_job_assignments WHERE hr_user_id=$1", user_id)
    result = [r["job_role_id"] for r in rows]
    await redis.setex(f"hr_assignments:{user_id}", 300, json.dumps(result))
    return result
```

### 3. Notification Deduplication Missing
Kafka guarantees **at-least-once** delivery. The same `match-result-events` message can be processed twice → HR gets duplicate notifications. Add idempotency at the notification insert:

```sql
-- Add unique constraint
ALTER TABLE notifications
    ADD COLUMN event_id UUID UNIQUE;

-- Insert with ON CONFLICT DO NOTHING
INSERT INTO notifications (id, event_id, hr_user_id, ...)
VALUES (...)
ON CONFLICT (event_id) DO NOTHING;
```

### 4. Embedding Model Versioning
If you ever update from `all-MiniLM-L6-v2` to a better model, **all existing ChromaDB vectors are incompatible** — different dimensional space. There's no migration strategy. Fix:

```python
# Tag every document with the embedding model version
metadatas=[{
    "resume_id": application_id,
    "embedding_model": "all-MiniLM-L6-v2",
    "embedding_model_version": "1.0",
    ...
}]
```
And version your ChromaDB collections (`resumes_v1`, `resumes_v2`) with a migration job.

---

## 🟠 Important Improvements

### 5. No Kafka Schema Registry
Raw JSON Kafka events break silently when producers/consumers evolve independently. Use **Confluent Schema Registry with Avro or Protobuf**:

```python
# Schema is versioned, validated on produce AND consume
# Incompatible schema changes are rejected at the registry
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer
```

### 6. Redis is a Single Point of Failure
The architecture uses Redis for unread counts, WS fan-out, and token revocation — all critical. A Redis crash silently breaks notifications and auth. Fix:
- Use **Redis Sentinel** (HA) or **Redis Cluster** (HA + sharding)
- For token revocation fallback: if Redis is down, default to DB validation

### 7. No Hybrid Search in RAG
Pure vector similarity misses exact keyword matches (e.g., specific certifications like "AWS SAP-C02"). Production RAG should use **hybrid search** — combine dense vector search with sparse BM25:

```python
# LlamaIndex supports this natively
from llama_index.retrievers import BM25Retriever, VectorIndexRetriever
from llama_index.retrievers import QueryFusionRetriever

retriever = QueryFusionRetriever(
    [vector_retriever, bm25_retriever],
    similarity_top_k=6,
    mode="reciprocal_reranking",   # RRF fusion
)
```

### 8. No Feedback Loop to Improve Matches
When an HR user rejects a high-scoring applicant, that signal is thrown away. A production system should:
- Store HR decisions (`hired`, `rejected`, `interviewed`) in the DB
- Periodically analyze score vs. outcome correlation
- Use this to calibrate the `score_threshold` per job role dynamically

```sql
ALTER TABLE match_results ADD COLUMN hr_decision VARCHAR(20);  -- hired/rejected/interviewed
ALTER TABLE match_results ADD COLUMN hr_feedback TEXT;
```

### 9. localStorage Token Storage is XSS-Vulnerable
The frontend hook uses `localStorage.getItem('access_token')` — any XSS attack immediately steals tokens. **Fix**: Use `httpOnly` cookies exclusively:

```typescript
// Instead of localStorage, tokens set by server as httpOnly cookies
// api.ts — no manual token attachment needed, browser sends cookies automatically
const api = axios.create({
    baseURL: process.env.NEXT_PUBLIC_API_URL,
    withCredentials: true,   // Send cookies on every request
})
```

### 10. PDF Parsing is a Security Risk
Untrusted PDFs can exploit parser vulnerabilities (PyMuPDF had CVEs). Recommendations:
- Sandbox PDF parsing in an **isolated container/subprocess** with no network access
- Validate file type by magic bytes, not just extension
- Set a file size limit (e.g., 10MB)
- Scan with ClamAV before parsing

```python
import magic
async def validate_pdf(file: UploadFile):
    content = await file.read(2048)  # Read only magic bytes
    mime = magic.from_buffer(content, mime=True)
    if mime != "application/pdf":
        raise HTTPException(400, "Invalid file type")
    if file.size > 10 * 1024 * 1024:
        raise HTTPException(400, "File too large")
    await file.seek(0)
```

---

## 🟡 Quality Improvements

### 11. No Observability Stack
Correlation IDs in logs alone aren't enough. Add:
- **OpenTelemetry** → traces across FastAPI + Kafka consumers → Jaeger/Tempo
- **Prometheus + Grafana** → metrics: Kafka consumer lag, LLM inference time, cache hit rate
- **Sentry** → exception tracking with Kafka message context
- **Kafka consumer lag alert**: if `match-worker-group` lag > 1000, page on-call

### 12. The Saga Compensations Can Fail Silently
The current design logs compensation failures and moves on. A failed compensation (e.g., ChromaDB delete failed during rollback) leaves **orphaned vectors** forever. Fix: persist saga state to a `saga_log` table so failed compensations can be replayed:

```sql
CREATE TABLE saga_log (
    saga_id         UUID PRIMARY KEY,
    type            VARCHAR(50),
    state           JSONB,           -- completed steps + their compensation data
    status          VARCHAR(20),     -- running / completed / compensating / failed
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

Or use **[Temporal](https://temporal.io/)** — a purpose-built workflow orchestrator that handles saga durability natively.

### 13. No Rate Limiting on WebSocket Connections
A malicious user can open thousands of WS connections to `/ws/notifications/{hr_user_id}`. Add connection limits:

```python
# Track active connections in Redis
async def accept_websocket(hr_user_id: str, websocket: WebSocket):
    conn_count = await redis.incr(f"ws_conn:{hr_user_id}")
    if conn_count > 3:   # Max 3 concurrent connections per user
        await redis.decr(f"ws_conn:{hr_user_id}")
        await websocket.close(code=1008, reason="Connection limit exceeded")
        return False
    await websocket.accept()
    return True
```

### 14. No GDPR / Data Retention Strategy
Resumes contain PII. There's no:
- **Right to erasure**: `DELETE /users/{id}/data` must cascade to S3, ChromaDB, and PostgreSQL
- **Data retention policy**: resumes auto-deleted after N days if no active application
- **Consent tracking**: when applicant uploaded and agreed to terms

### 15. Unread Count Can Drift from Reality
`notif:unread:{hr_user_id}` in Redis can drift from the actual DB count if Redis restarts or a DECR is missed. Add a periodic reconciliation:

```python
# Nightly job or on-demand endpoint
async def sync_unread_count(hr_user_id: str):
    real_count = await db.fetchval(
        "SELECT COUNT(*) FROM notifications WHERE hr_user_id=$1 AND is_read=false",
        hr_user_id
    )
    await redis.set(f"notif:unread:{hr_user_id}", real_count)
```

---

## Summary Priority Matrix

| Improvement | Impact | Effort |
|---|---|---|
| Structured LLM output (Instructor) | 🔴 Critical | Low |
| Notification deduplication | 🔴 Critical | Low |
| JWT mutable state fix | 🔴 Critical | Medium |
| Embedding model versioning | 🔴 Critical | Medium |
| Hybrid RAG search | 🟠 High | Medium |
| httpOnly cookie tokens | 🟠 High | Low |
| PDF sandboxing | 🟠 High | Medium |
| Redis HA | 🟠 High | Low (config) |
| OpenTelemetry + Grafana | 🟡 Medium | Medium |
| Saga durability (Temporal) | 🟡 Medium | High |
| GDPR erasure flow | 🟡 Medium | Medium |
| Feedback loop / HR decisions | 🟡 Medium | Low |

The three items to fix **before writing another line of feature code**: structured LLM output validation, notification deduplication, and httpOnly cookie auth — they're low effort and prevent hard-to-debug production incidents.
