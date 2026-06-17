# Complete Technical Interview Preparation Guide
### AI-Inclined Full-Stack Engineering Role — UST Global

---

## TOPIC 1: Idempotency

---

### Common Interview Questions (20–30)

---

**Q1: What is idempotency? Why is it critical in distributed systems?**

**Answer:**
An operation is **idempotent** if applying it multiple times produces the same result as applying it once. Mathematically: `f(f(x)) = f(x)`.

In distributed systems, network failures, retries, and message duplication make idempotency essential. Without it, a retry of a failed request can cause duplicate payments, duplicate records, or data corruption.

**Real-world context:** In your NSE data ingestion pipeline, if a Lambda function consuming an SQS message fails midway after partially writing stock price records to DynamoDB, SQS will redeliver the message. Without idempotency, you get duplicate price entries.

---

**Q2: Which HTTP methods are idempotent by specification?**

**Answer:**

| Method | Idempotent | Safe | Notes |
|--------|-----------|------|-------|
| GET | ✅ | ✅ | No side effects |
| HEAD | ✅ | ✅ | Same as GET, no body |
| PUT | ✅ | ❌ | Replaces entire resource |
| DELETE | ✅ | ❌ | Deleting already-deleted resource returns 404 but state is same |
| OPTIONS | ✅ | ✅ | Metadata only |
| POST | ❌ | ❌ | Not idempotent by default |
| PATCH | ❌ | ❌ | Partial update, not idempotent unless conditional |

**Why DELETE is idempotent:** The state of the system after N deletions of the same resource is identical — the resource doesn't exist. The response code may differ (200 first time, 404 thereafter), but idempotency refers to **state**, not response.

---

**Q3: How do you make POST requests idempotent?**

**Answer:**
Use an **Idempotency Key** — a client-generated UUID sent in a request header. The server stores a mapping of `idempotency_key → response`. On retry, return the cached response without re-executing.

```typescript
// Express.js middleware for idempotency
import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const TTL_SECONDS = 86400; // 24 hours

export async function idempotencyMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const idempotencyKey = req.headers['idempotency-key'] as string;

  if (!idempotencyKey || req.method !== 'POST') {
    return next();
  }

  // Validate UUID format
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(idempotencyKey)) {
    return res.status(400).json({ error: 'Invalid idempotency key format' });
  }

  const cacheKey = `idempotency:${idempotencyKey}`;

  // Check for in-progress request (prevent concurrent duplicates)
  const existing = await redis.get(cacheKey);

  if (existing === 'PROCESSING') {
    return res.status(409).json({ error: 'Request is already being processed' });
  }

  if (existing) {
    const cached = JSON.parse(existing);
    return res.status(cached.statusCode).json(cached.body);
  }

  // Mark as in-progress
  await redis.set(cacheKey, 'PROCESSING', 'EX', 30); // 30s lock

  // Override res.json to capture the response
  const originalJson = res.json.bind(res);
  res.json = (body: any) => {
    // Store the response
    redis.set(
      cacheKey,
      JSON.stringify({ statusCode: res.statusCode, body }),
      'EX',
      TTL_SECONDS
    );
    return originalJson(body);
  };

  next();
}
```

---

**Q4: What is an Idempotency Key in payment APIs? How does Stripe implement it?**

**Answer:**
Stripe's idempotency key is sent as `Idempotency-Key: <uuid>` header. Stripe stores the key with the response for 24 hours. If a retry arrives with the same key, Stripe returns the original response — no new charge created.

Key design decisions:
- **Scope:** Per endpoint (same key for `/charges` and `/refunds` can coexist)
- **TTL:** 24 hours at Stripe; choose based on your retry window
- **Conflict detection:** If same key is used with different request body, return 422
- **Storage:** Redis (fast) with async persistence to DB for durability

```python
# FastAPI implementation for NSE order placement
from fastapi import FastAPI, Header, HTTPException, Depends
from redis.asyncio import Redis
import json, uuid

app = FastAPI()
redis = Redis.from_url("redis://localhost")

@app.post("/api/nse/orders")
async def place_order(
    order: OrderRequest,
    idempotency_key: str = Header(..., alias="Idempotency-Key")
):
    # Validate key format
    try:
        uuid.UUID(idempotency_key)
    except ValueError:
        raise HTTPException(400, "Invalid idempotency key")
    
    cache_key = f"idem:{idempotency_key}"
    cached = await redis.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    # Set lock with 10s TTL
    locked = await redis.set(cache_key, "PROCESSING", nx=True, ex=10)
    if not locked:
        raise HTTPException(409, "Concurrent request with same key")
    
    try:
        result = await process_order(order)
        # Store result with 24h TTL
        await redis.set(cache_key, json.dumps(result), ex=86400)
        return result
    except Exception as e:
        await redis.delete(cache_key)  # Release lock on failure
        raise
```

---

**Q5: How does idempotency differ from safety in HTTP?**

**Answer:**
- **Safe:** No side effects on the server (GET, HEAD, OPTIONS) — purely read operations
- **Idempotent:** May have side effects, but repeating produces identical state (GET, HEAD, PUT, DELETE)

All safe methods are idempotent, but not vice versa. PUT and DELETE are idempotent but not safe.

---

**Q6: What is the "at-least-once" vs "exactly-once" delivery semantics in message queues?**

**Answer:**

| Semantic | Description | Guarantee | Requires |
|----------|-------------|-----------|----------|
| At-most-once | May lose messages | No duplicates, possible loss | — |
| At-least-once | May duplicate | No loss, possible duplicates | Idempotent consumers |
| Exactly-once | No loss, no dup | Strongest guarantee | Transactional support |

**SQS defaults to at-least-once.** FIFO SQS with content-based deduplication provides exactly-once within 5 minutes. Kafka transactions + Streams API provide exactly-once.

**Practical advice:** Design consumers to be idempotent rather than relying on exactly-once delivery, because even with exactly-once queues, consumer crashes can cause reprocessing.

---

**Q7: How do you implement idempotency in AWS Lambda with SQS triggers?**

**Answer:**
SQS can deliver the same message multiple times (visibility timeout expiry, redrive). Lambda must handle duplicates.

```python
# Lambda handler with DynamoDB-based idempotency
import boto3
import json
import hashlib
from datetime import datetime, timezone
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
idempotency_table = dynamodb.Table('NSE_ProcessedMessages')
nse_table = dynamodb.Table('NSE_StockPrices')

def lambda_handler(event, context):
    for record in event['Records']:
        message_id = record['messageId']
        body = json.loads(record['body'])
        
        # Check if already processed
        if is_already_processed(message_id):
            print(f"Skipping duplicate message: {message_id}")
            continue
        
        # Process the NSE stock data
        try:
            process_stock_data(body)
            mark_as_processed(message_id, body)
        except Exception as e:
            # Do NOT catch here — let Lambda retry
            raise e

def is_already_processed(message_id: str) -> bool:
    try:
        response = idempotency_table.get_item(
            Key={'messageId': message_id}
        )
        return 'Item' in response
    except Exception:
        return False

def mark_as_processed(message_id: str, payload: dict):
    idempotency_table.put_item(
        Item={
            'messageId': message_id,
            'processedAt': datetime.now(timezone.utc).isoformat(),
            'symbol': payload.get('symbol'),
            'ttl': int(datetime.now(timezone.utc).timestamp()) + 86400  # 24h TTL
        },
        ConditionExpression='attribute_not_exists(messageId)'
        # ConditionExpression prevents race conditions
    )

def process_stock_data(data: dict):
    nse_table.put_item(
        Item={
            'symbol': data['symbol'],
            'timestamp': data['timestamp'],
            'price': str(data['price']),
            'volume': data['volume']
        }
    )
```

**Better approach: AWS Lambda Powertools Idempotency**

```python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent_function,
    IdempotencyConfig,
    DynamoDBPersistenceLayer
)

persistence_layer = DynamoDBPersistenceLayer(table_name="IdempotencyTable")
config = IdempotencyConfig(
    event_key_jmespath="messageId",  # Use SQS messageId as key
    raise_on_no_idempotency_key=True,
    expires_after_seconds=86400
)

@idempotent_function(data_keyword_argument="record", config=config, persistence_store=persistence_layer)
def process_record(record: dict):
    # This function is now automatically idempotent
    return ingest_nse_data(record)
```

---

**Q8: How do you use DynamoDB conditional writes for idempotency?**

**Answer:**
DynamoDB's `ConditionExpression` ensures an item is only written if a condition is met — atomically. This is the core primitive for idempotency.

```python
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('NSE_DailyPrices')

def upsert_stock_price_idempotent(symbol: str, date: str, price: float):
    """
    Idempotently insert a stock price.
    If record exists, it means we already processed this — skip.
    """
    try:
        table.put_item(
            Item={
                'PK': f'STOCK#{symbol}',
                'SK': f'DATE#{date}',
                'price': str(price),
                'updatedAt': datetime.utcnow().isoformat()
            },
            ConditionExpression='attribute_not_exists(PK) AND attribute_not_exists(SK)'
        )
        return {'status': 'created'}
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Record already exists — idempotent success
            return {'status': 'already_exists'}
        raise

def increment_trade_count_idempotent(symbol: str, date: str, trade_id: str):
    """
    Use SET with condition to ensure each trade is counted only once.
    Stores processed trade IDs in a set.
    """
    try:
        table.update_item(
            Key={'PK': f'STOCK#{symbol}', 'SK': f'DATE#{date}'},
            UpdateExpression='ADD tradeCount :one, processedTrades :trade_set',
            ConditionExpression='NOT contains(processedTrades, :trade_id)',
            ExpressionAttributeValues={
                ':one': 1,
                ':trade_set': {trade_id},
                ':trade_id': trade_id
            }
        )
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            print(f"Trade {trade_id} already counted — skipping")
```

---

**Q9: What is the "outbox pattern" and how does it enable idempotency?**

**Answer:**
The **Transactional Outbox Pattern** ensures that database writes and event publishing are atomic, preventing dual-write failures.

```
Problem: 
  1. Write to DB ✅
  2. Publish to SQS ❌ (crash) → event lost, downstream never notified

Solution: Outbox Pattern
  1. Write to DB + write to Outbox table in SAME transaction ✅
  2. Separate poller reads Outbox → publishes to SQS → marks as sent
```

```sql
-- PostgreSQL implementation
BEGIN;

-- Main operation
INSERT INTO nse_trades (id, symbol, price, quantity, traded_at)
VALUES ($1, $2, $3, $4, NOW())
ON CONFLICT (id) DO NOTHING;  -- Idempotent insert

-- Outbox event
INSERT INTO outbox (event_id, event_type, payload, created_at, processed)
VALUES ($1, 'TRADE_INGESTED', $5::jsonb, NOW(), false)
ON CONFLICT (event_id) DO NOTHING;

COMMIT;
```

```typescript
// Outbox poller (runs on a schedule)
async function pollOutbox() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    
    // Lock rows for processing
    const { rows } = await client.query(`
      SELECT * FROM outbox
      WHERE processed = false
      ORDER BY created_at
      LIMIT 10
      FOR UPDATE SKIP LOCKED
    `);
    
    for (const event of rows) {
      await sqs.sendMessage({
        QueueUrl: process.env.SQS_URL!,
        MessageBody: JSON.stringify(event.payload),
        MessageDeduplicationId: event.event_id,  // FIFO SQS dedup
        MessageGroupId: event.payload.symbol
      }).promise();
      
      await client.query(
        'UPDATE outbox SET processed = true WHERE event_id = $1',
        [event.event_id]
      );
    }
    
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

---

**Q10: How does PUT differ from PATCH in terms of idempotency?**

**Answer:**
- **PUT** replaces the entire resource → always idempotent (same body → same state)
- **PATCH** applies a partial update → can be idempotent or not depending on implementation

```typescript
// Non-idempotent PATCH — increment operation
PATCH /api/portfolio/RELIANCE
{ "operation": "increment", "field": "quantity", "value": 10 }
// Two calls → quantity += 20 (NOT idempotent)

// Idempotent PATCH — set to specific value
PATCH /api/portfolio/RELIANCE  
{ "quantity": 110 }
// Two calls → quantity = 110 (idempotent)

// Making PATCH idempotent with ETags + conditional request
PATCH /api/portfolio/RELIANCE
If-Match: "abc123"   // ETag from last GET
{ "quantity": 110 }
// Second call fails with 412 Precondition Failed (ETag changed)
```

---

**Q11: What is optimistic locking and how does it ensure idempotency?**

**Answer:**
Optimistic locking uses a version number to detect concurrent modifications. Only the first writer succeeds; subsequent writers get a conflict error.

```python
# DynamoDB optimistic locking
def update_portfolio_price(symbol: str, new_price: float, expected_version: int):
    try:
        table.update_item(
            Key={'PK': f'PORTFOLIO#{symbol}'},
            UpdateExpression='SET currentPrice = :price, version = :new_version',
            ConditionExpression='version = :expected_version',
            ExpressionAttributeValues={
                ':price': str(new_price),
                ':new_version': expected_version + 1,
                ':expected_version': expected_version
            }
        )
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            raise OptimisticLockException("Version conflict — read and retry")
```

```sql
-- PostgreSQL optimistic locking
UPDATE nse_positions
SET quantity = $1, version = version + 1
WHERE symbol = $2 AND version = $3
RETURNING *;
-- If 0 rows returned → version mismatch → retry with fresh read
```

---

**Q12: How do you handle idempotency in GraphQL mutations?**

**Answer:**
GraphQL has no built-in idempotency. Approaches:

1. **Client-provided mutation ID** in the input
2. **Directive-based deduplication**
3. **Response caching** keyed on mutation variables

```graphql
mutation PlaceOrder($input: PlaceOrderInput!) {
  placeOrder(input: $input) {
    orderId
    status
  }
}

# Input type with idempotency key
input PlaceOrderInput {
  symbol: String!
  quantity: Int!
  price: Float!
  clientMutationId: ID!  # Client-generated UUID
}
```

```typescript
// Apollo Server resolver
const resolvers = {
  Mutation: {
    placeOrder: async (_, { input }) => {
      const { clientMutationId, ...orderData } = input;
      
      // Check idempotency store
      const cached = await redis.get(`mutation:${clientMutationId}`);
      if (cached) return JSON.parse(cached);
      
      const result = await processOrder(orderData);
      await redis.set(
        `mutation:${clientMutationId}`,
        JSON.stringify(result),
        'EX',
        3600
      );
      return result;
    }
  }
};
```

---

**Q13: Explain idempotency in the context of database migrations.**

**Answer:**
Migrations should be idempotent — running the same migration twice should not fail or corrupt data.

```sql
-- Non-idempotent (fails on second run)
CREATE TABLE nse_stocks (...);
ALTER TABLE nse_stocks ADD COLUMN sector VARCHAR(50);

-- Idempotent (safe to run multiple times)
CREATE TABLE IF NOT EXISTS nse_stocks (...);
ALTER TABLE nse_stocks ADD COLUMN IF NOT EXISTS sector VARCHAR(50);

-- For indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_stocks_symbol 
ON nse_stocks(symbol);

-- For stored procedures — use CREATE OR REPLACE
CREATE OR REPLACE FUNCTION get_stock_avg_price(p_symbol TEXT) ...
```

---

**Q14: What is the Two-Phase Commit (2PC) protocol and how does it relate to distributed idempotency?**

**Answer:**
2PC coordinates atomic transactions across multiple distributed resources:
- **Phase 1 (Prepare):** Coordinator asks all participants if they can commit
- **Phase 2 (Commit/Abort):** If all agree, coordinator sends commit; else abort

Problems: blocking (participants lock until coordinator responds), coordinator SPOF. Modern systems prefer **Saga pattern** with idempotent compensating transactions.

```
Saga for NSE Trade Settlement:
1. Reserve funds (debit account)          → Compensate: credit back
2. Place order at exchange               → Compensate: cancel order
3. Update portfolio                      → Compensate: revert portfolio

Each step MUST be idempotent because the orchestrator may retry.
```

---

**Q15: How does SQS FIFO with content-based deduplication work?**

**Answer:**
SQS FIFO queues support **exactly-once processing** via:
- **MessageDeduplicationId:** Explicit dedup key (per message)
- **Content-Based Deduplication:** SHA-256 hash of message body used as dedup key

Within a 5-minute dedup interval, duplicate messages are discarded.

```typescript
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';
import crypto from 'crypto';

const sqs = new SQSClient({ region: 'ap-south-1' });

async function publishStockData(stockData: NSEStockRecord) {
  const body = JSON.stringify(stockData);
  
  // Explicit deduplication key based on business logic
  const dedupId = `${stockData.symbol}-${stockData.timestamp}`;
  
  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.NSE_FIFO_QUEUE_URL,
    MessageBody: body,
    MessageGroupId: stockData.symbol,  // Ordering per symbol
    MessageDeduplicationId: dedupId,    // Explicit dedup
  }));
}
```

---

**Q16: How do you handle idempotency when using AWS Step Functions?**

**Answer:**
Step Functions automatically retries failed states. Each Lambda in the workflow must be idempotent, as any state can be retried.

```json
{
  "Comment": "NSE Data Ingestion Workflow",
  "StartAt": "ValidateData",
  "States": {
    "ValidateData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:ValidateNSEData",
      "Retry": [{
        "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2
      }],
      "Next": "StoreData"
    },
    "StoreData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:StoreNSEData",
      "Parameters": {
        "executionId.$": "$$.Execution.Id",  // Pass execution ID for idempotency
        "data.$": "$.validatedData"
      },
      "Retry": [...]
    }
  }
}
```

```python
# Lambda using Step Functions execution ID as idempotency key
def lambda_handler(event, context):
    execution_id = event['executionId']
    
    # Use execution ID as idempotency key
    if is_already_processed(execution_id):
        return get_existing_result(execution_id)
    
    result = store_nse_data(event['data'])
    save_result(execution_id, result)
    return result
```

---

**Q17: What is the "exactly-once semantics" challenge in event sourcing?**

**Answer:**
In event sourcing, every state change is an immutable event. Idempotency requires that replaying events doesn't corrupt state.

```python
# Event store with idempotent append
class NSEEventStore:
    def append_event(self, stream_id: str, event: dict, expected_version: int):
        """
        Optimistic concurrency: only append if stream is at expected_version
        """
        current_version = self.get_stream_version(stream_id)
        
        if current_version != expected_version:
            raise ConcurrencyException(
                f"Expected version {expected_version}, got {current_version}"
            )
        
        # Check if this exact event was already appended (by event_id)
        if self.event_exists(event['event_id']):
            return  # Idempotent — already stored
        
        self.store_event(stream_id, event, current_version + 1)
```

---

**Q18: How do you design an idempotent REST API for bulk operations?**

**Answer:**
```typescript
// Idempotent bulk upsert endpoint
app.put('/api/nse/prices/bulk', async (req, res) => {
  const { prices, batchId } = req.body;
  
  // Use batchId as idempotency key
  const existing = await redis.get(`batch:${batchId}`);
  if (existing) {
    return res.status(200).json({
      status: 'already_processed',
      result: JSON.parse(existing)
    });
  }
  
  // Upsert all prices atomically
  const result = await prisma.$transaction(
    prices.map(p =>
      prisma.stockPrice.upsert({
        where: { symbol_date: { symbol: p.symbol, date: p.date } },
        update: { price: p.price, volume: p.volume },
        create: { symbol: p.symbol, date: p.date, price: p.price, volume: p.volume }
      })
    )
  );
  
  await redis.set(`batch:${batchId}`, JSON.stringify(result), 'EX', 86400);
  res.status(200).json({ status: 'processed', result });
});
```

---

**Q19: How does Kubernetes handle idempotency in deployments?**

**Answer:**
K8s API is inherently idempotent — applying the same manifest multiple times converges to the same state. This is the **declarative model**. `kubectl apply` uses server-side apply with merge strategies to detect and apply only diffs.

---

**Q20: How would you test idempotency in an API?**

**Answer:**
```typescript
// Jest test for idempotent endpoint
describe('POST /api/nse/ingest - Idempotency', () => {
  const idempotencyKey = uuid();
  const payload = { symbol: 'NIFTY50', price: 22500.75, date: '2025-06-01' };
  
  it('should return same result on duplicate request', async () => {
    const first = await request(app)
      .post('/api/nse/ingest')
      .set('Idempotency-Key', idempotencyKey)
      .send(payload);
    
    const second = await request(app)
      .post('/api/nse/ingest')
      .set('Idempotency-Key', idempotencyKey)
      .send(payload);
    
    expect(first.status).toBe(201);
    expect(second.status).toBe(200); // Or 201 — same body matters
    expect(first.body.recordId).toBe(second.body.recordId);
    
    // Verify only ONE record in DB
    const count = await db.stockPrices.count({ where: { symbol: 'NIFTY50', date: '2025-06-01' } });
    expect(count).toBe(1);
  });
  
  it('should reject same key with different body', async () => {
    await request(app)
      .post('/api/nse/ingest')
      .set('Idempotency-Key', idempotencyKey)
      .send({ ...payload, price: 22600 }); // Different price
    
    const response = await request(app)
      .post('/api/nse/ingest')
      .set('Idempotency-Key', idempotencyKey)
      .send({ ...payload, price: 99999 }); // Changed body
    
    expect(response.status).toBe(422); // Unprocessable — body mismatch
  });
});
```

---

**Q21: What is the difference between idempotency and consistency?**

**Answer:**
- **Idempotency:** Property of an operation — repeating it N times = doing it once
- **Consistency:** Property of the system state — all nodes see the same data at the same time

They're orthogonal but related. A system can be idempotent but eventually consistent (SQS + DynamoDB). You can also have strong consistency without idempotency.

---

**Q22: How do you handle idempotency in WebSocket connections?**

**Answer:**
```typescript
// WebSocket server with message deduplication
const processedMessages = new Map<string, number>(); // messageId -> timestamp

wss.on('connection', (ws) => {
  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    const { messageId, payload } = message;
    
    // Check if already processed (within last 60 seconds)
    const lastProcessed = processedMessages.get(messageId);
    if (lastProcessed && Date.now() - lastProcessed < 60000) {
      ws.send(JSON.stringify({ type: 'ACK', messageId, duplicate: true }));
      return;
    }
    
    processMessage(payload);
    processedMessages.set(messageId, Date.now());
    ws.send(JSON.stringify({ type: 'ACK', messageId, duplicate: false }));
  });
});
```

---

**Q23: Explain token bucket / leaky bucket in the context of idempotency for rate-limited APIs.**

**Answer:**
When an idempotent API is rate-limited, retries must respect rate limits. The idempotency key allows clients to retry safely, but rate limiting must track by `(client_id, idempotency_key)` not just `client_id`, to not penalize legitimate retries.

```typescript
async function checkRateLimit(clientId: string, idempotencyKey?: string) {
  if (idempotencyKey) {
    // Check if this is a retry (key already seen)
    const isRetry = await redis.exists(`idem:${idempotencyKey}`);
    if (isRetry) return true; // Allow retries without consuming rate limit quota
  }
  
  // Regular rate limit check
  const key = `rate:${clientId}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 60); // 1 minute window
  return count <= 100; // 100 requests per minute
}
```

---

**Q24: How do you implement idempotent data ingestion for NSE stock data specifically?**

**Answer:**
NSE publishes BHAV copy (end-of-day prices) and tick data. The ingestion pipeline must handle:
1. Reprocessing on failure
2. Duplicate feeds from multiple sources
3. Out-of-order delivery

```python
# NSE BHAV copy ingestion — fully idempotent
import boto3
import pandas as pd
from datetime import datetime

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('NSE_BHAV_Copy')

def ingest_bhav_copy(s3_key: str, trading_date: str):
    """
    Idempotent ingestion of NSE BHAV copy data.
    Primary key: (symbol, trading_date) — natural dedup key.
    """
    # Download CSV from S3
    obj = s3.get_object(Bucket='nse-raw-data', Key=s3_key)
    df = pd.read_csv(obj['Body'])
    
    # Track ingestion job idempotency
    job_id = f"bhav-{trading_date}"
    
    with table.batch_writer() as batch:
        for _, row in df.iterrows():
            batch.put_item(
                Item={
                    'PK': f"STOCK#{row['SYMBOL']}",
                    'SK': f"DATE#{trading_date}",
                    'open': str(row['OPEN']),
                    'high': str(row['HIGH']),
                    'low': str(row['LOW']),
                    'close': str(row['CLOSE']),
                    'volume': int(row['TOTTRDQTY']),
                    'sourceFile': s3_key,
                    'ingestedAt': datetime.utcnow().isoformat(),
                    # put_item replaces — natural idempotency with same PK/SK
                }
            )
    
    # Mark job as complete
    mark_ingestion_complete(job_id, s3_key, len(df))
    return {'processed': len(df), 'date': trading_date}
```

---

### Deep-Dive Real-World Edge Case Questions (20)

---

**EC1: Your NSE data ingestion Lambda receives the same SQS message 3 times (two retries after failures). The first invocation partially wrote 500 out of 1000 records to DynamoDB. How do you design idempotency to ensure all 1000 records are written exactly once?**

**Answer:**
The key insight: **partial writes break naive idempotency**. You need checkpoint-based idempotency.

```python
import boto3
import json
from dataclasses import dataclass
from typing import List

dynamodb = boto3.resource('dynamodb')
prices_table = dynamodb.Table('NSE_StockPrices')
checkpoint_table = dynamodb.Table('IngestionCheckpoints')

@dataclass
class StockRecord:
    symbol: str
    date: str
    price: float
    volume: int

def process_batch_idempotent(message_id: str, records: List[StockRecord]):
    """
    Checkpoint-based idempotent batch processing.
    Uses DynamoDB to track which sub-batches were successfully written.
    """
    CHUNK_SIZE = 25  # DynamoDB batch write limit
    chunks = [records[i:i+CHUNK_SIZE] for i in range(0, len(records), CHUNK_SIZE)]
    
    for chunk_index, chunk in enumerate(chunks):
        checkpoint_key = f"{message_id}#CHUNK#{chunk_index}"
        
        # Skip already-processed chunks
        if is_chunk_processed(checkpoint_key):
            print(f"Skipping chunk {chunk_index} — already processed")
            continue
        
        # Write chunk atomically
        write_chunk_to_dynamodb(chunk)
        
        # Mark chunk as processed AFTER successful write
        mark_chunk_processed(checkpoint_key, len(chunk))
    
    # Mark entire message as done
    mark_message_complete(message_id, len(records))

def is_chunk_processed(checkpoint_key: str) -> bool:
    resp = checkpoint_table.get_item(Key={'checkpointId': checkpoint_key})
    return 'Item' in resp

def write_chunk_to_dynamodb(chunk: List[StockRecord]):
    with prices_table.batch_writer() as batch:
        for record in chunk:
            batch.put_item(Item={
                'PK': f"STOCK#{record.symbol}",
                'SK': f"DATE#{record.date}",
                'price': str(record.price),
                'volume': record.volume,
            })
            # put_item is naturally idempotent for same PK+SK

def mark_chunk_processed(checkpoint_key: str, count: int):
    from botocore.exceptions import ClientError
    try:
        checkpoint_table.put_item(
            Item={
                'checkpointId': checkpoint_key,
                'count': count,
                'processedAt': datetime.utcnow().isoformat(),
                'ttl': int(time.time()) + 86400
            },
            ConditionExpression='attribute_not_exists(checkpointId)'
        )
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            pass  # Another concurrent invocation wrote this — fine
```

---

**EC2: Two simultaneous API calls arrive with the same idempotency key. Your Redis check-and-set has a race window between GET and SET. How do you prevent both from executing the business logic?**

**Answer:**
Use `SET NX` (atomic set-if-not-exists) as a distributed lock:

```python
import asyncio
import redis.asyncio as aioredis

redis = aioredis.from_url("redis://localhost")

async def execute_idempotent(idempotency_key: str, operation):
    lock_key = f"lock:{idempotency_key}"
    result_key = f"result:{idempotency_key}"
    
    # Atomic check for completed result
    result = await redis.get(result_key)
    if result:
        return json.loads(result)
    
    # Atomic acquire lock — SET NX with 30s TTL
    acquired = await redis.set(lock_key, "1", nx=True, ex=30)
    
    if not acquired:
        # Another process is executing — poll for result
        for attempt in range(30):  # Poll for up to 30 seconds
            await asyncio.sleep(1)
            result = await redis.get(result_key)
            if result:
                return json.loads(result)
        raise TimeoutError("Timed out waiting for concurrent idempotent operation")
    
    try:
        # We hold the lock — execute the operation
        output = await operation()
        
        # Store result with 24h TTL
        await redis.set(result_key, json.dumps(output), ex=86400)
        return output
    finally:
        await redis.delete(lock_key)  # Always release lock
```

---

**EC3: Your idempotency key store (Redis) goes down. How do you degrade gracefully without causing duplicate operations or complete service outage?**

**Answer:**
**Circuit breaker + fallback strategy:**

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Redis down, fallback mode
    HALF_OPEN = "half_open"  # Testing recovery

class IdempotencyCircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.last_failure_time = 0
        self.recovery_timeout = recovery_timeout
    
    async def execute(self, idempotency_key: str, operation, db_fallback):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                # Redis down — use DB-level idempotency as fallback
                return await db_fallback(idempotency_key, operation)
        
        try:
            result = await self._execute_with_redis(idempotency_key, operation)
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
            return result
        except redis.RedisError:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
            # Fall back to DB-level idempotency
            return await db_fallback(idempotency_key, operation)

async def db_level_idempotency(idempotency_key: str, operation):
    """
    Fallback: use PostgreSQL advisory locks + idempotency table
    """
    async with db.transaction():
        # PostgreSQL advisory lock based on hash of key
        lock_id = hash(idempotency_key) % (2**31)
        await db.execute(f"SELECT pg_advisory_xact_lock({lock_id})")
        
        existing = await db.fetchrow(
            "SELECT result FROM idempotency_records WHERE key = $1",
            idempotency_key
        )
        if existing:
            return json.loads(existing['result'])
        
        result = await operation()
        await db.execute(
            "INSERT INTO idempotency_records (key, result, expires_at) VALUES ($1, $2, NOW() + INTERVAL '24 hours')",
            idempotency_key, json.dumps(result)
        )
        return result
```

---

**EC4: Your NSE pipeline runs an idempotent upsert, but a data correction requires updating an already-processed record. How do you allow authorized re-processing without breaking idempotency guarantees?**

**Answer:**
Use **versioned idempotency keys** and **correction workflows**:

```python
# Idempotency key includes version: "{base_key}:v{version}"
# Normal ingestion: "bhav-2025-06-01:v1"
# Correction run: "bhav-2025-06-01:v2"

async def ingest_with_correction_support(
    trading_date: str,
    data: List[dict],
    version: int = 1,
    override_previous: bool = False
):
    idempotency_key = f"bhav-{trading_date}:v{version}"
    
    if override_previous:
        # Authorized correction — invalidate previous version
        previous_key = f"bhav-{trading_date}:v{version - 1}"
        await redis.delete(f"result:{previous_key}")
        
        # Audit log the correction
        await audit_log.record({
            'action': 'IDEMPOTENCY_OVERRIDE',
            'trading_date': trading_date,
            'version': version,
            'reason': 'data_correction',
            'timestamp': datetime.utcnow().isoformat()
        })
    
    return await execute_idempotent(idempotency_key, lambda: upsert_prices(trading_date, data, version))
```

---

**EC5: A client sends two requests with identical idempotency keys but different request bodies (possible API misuse or bug). How do you handle this?**

**Answer:**
```python
async def validate_idempotency_key(
    idempotency_key: str,
    request_body: dict
) -> Optional[dict]:
    """
    Returns cached result if key+body match.
    Raises error if key exists with different body.
    """
    cache_key = f"idem:{idempotency_key}"
    cached_raw = await redis.get(cache_key)
    
    if not cached_raw:
        return None
    
    cached = json.loads(cached_raw)
    
    # Compute and compare body fingerprints
    incoming_fingerprint = hashlib.sha256(
        json.dumps(request_body, sort_keys=True).encode()
    ).hexdigest()
    
    if cached.get('request_fingerprint') != incoming_fingerprint:
        # SECURITY: Log this — potential API misuse
        await security_log.warn({
            'event': 'IDEMPOTENCY_KEY_BODY_MISMATCH',
            'key': idempotency_key,
            'incoming_fingerprint': incoming_fingerprint,
            'stored_fingerprint': cached['request_fingerprint']
        })
        raise HTTPException(
            status_code=422,
            detail={
                'error': 'IDEMPOTENCY_KEY_REUSE',
                'message': 'Idempotency key already used with different request body'
            }
        )
    
    return cached['response']
```

---

**EC6: How would you handle idempotency in a multi-region active-active DynamoDB Global Tables setup where the same message could be processed in two regions simultaneously?**

**Answer:**
DynamoDB Global Tables use **last-writer-wins** replication. For idempotency across regions:

```python
# Use a region-prefixed lock in DynamoDB with conditional writes
import os

REGION = os.environ['AWS_REGION']  # e.g., ap-south-1

def acquire_global_idempotency_lock(message_id: str, ttl_seconds: int = 300) -> bool:
    """
    Attempt to claim processing rights for this message globally.
    Returns True if this region "won" the race.
    """
    try:
        idempotency_table.put_item(
            Item={
                'messageId': message_id,
                'processingRegion': REGION,
                'lockedAt': datetime.utcnow().isoformat(),
                'ttl': int(time.time()) + ttl_seconds,
                'status': 'PROCESSING'
            },
            # Only write if not already claimed
            ConditionExpression='attribute_not_exists(messageId)'
        )
        return True  # This region will process the message
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Another region is processing — skip
            existing = idempotency_table.get_item(Key={'messageId': message_id})
            if existing['Item']['processingRegion'] != REGION:
                print(f"Message {message_id} being processed by {existing['Item']['processingRegion']}")
                return False
        raise

# Note: With Global Tables, there's still a ~1s replication lag.
# For true global deduplication, use a single-region DynamoDB as the lock table
# and accept the cross-region latency cost.
```

---

**EC7: Your idempotent NSE data processor crashes after writing to DynamoDB but BEFORE deleting the SQS message. SQS redelivers. What happens and how do you handle it?**

**Answer:**
This is the classic **at-least-once delivery** scenario. The solution:

1. DynamoDB `put_item` is idempotent (same PK+SK replaces same data)
2. The second invocation must detect the data already exists

```python
def process_nse_message_safe(message: dict) -> dict:
    """
    Safe to run multiple times. DynamoDB put_item is naturally idempotent
    because we use the business key (symbol + date) as the DynamoDB key.
    
    Crash scenarios:
    A) Crash BEFORE DynamoDB write → SQS redeliver → write succeeds
    B) Crash AFTER DynamoDB write, BEFORE SQS delete → SQS redeliver → 
       DynamoDB write is a no-op (same data), SQS delete succeeds
    C) DynamoDB write succeeds, enrichment fails → need separate state
    """
    symbol = message['symbol']
    date = message['date']
    
    # Step 1: Idempotent raw price write
    prices_table.put_item(Item={
        'PK': f"STOCK#{symbol}",
        'SK': f"DATE#{date}",
        'price': str(message['price']),
        'volume': message['volume'],
        # Deterministic computed field — safe to overwrite
        'priceChangePercent': str(
            calculate_price_change(symbol, date, message['price'])
        )
    })
    
    # Step 2: Conditional enrichment (only if not done)
    try:
        prices_table.update_item(
            Key={'PK': f"STOCK#{symbol}", 'SK': f"DATE#{date}"},
            UpdateExpression='SET enriched = :true, enrichedAt = :ts',
            ConditionExpression='attribute_not_exists(enriched)',
            ExpressionAttributeValues={
                ':true': True,
                ':ts': datetime.utcnow().isoformat()
            }
        )
        enrich_with_fundamentals(symbol, date)
    except ClientError as e:
        if e.response['Error']['Code'] != 'ConditionalCheckFailedException':
            raise
        # Already enriched — skip (idempotent)
    
    return {'processed': True, 'symbol': symbol, 'date': date}
```

---

**EC8: You need to implement idempotency for a streaming pipeline (Kinesis) where records can be replayed during shard resharding. How do you design this?**

**Answer:**
```python
def kinesis_consumer_handler(event, context):
    for record in event['Records']:
        # Kinesis sequence number is unique per shard, but resharding can replay
        sequence_number = record['kinesis']['sequenceNumber']
        shard_id = record['eventID'].split(':')[0]  # shard-000000000000:...
        
        # Idempotency key = shard_id + sequence_number
        # Survives resharding because sequence numbers are unique per shard
        idempotency_key = f"{shard_id}#{sequence_number}"
        
        if is_already_processed(idempotency_key):
            continue
        
        data = json.loads(
            base64.b64decode(record['kinesis']['data']).decode('utf-8')
        )
        
        process_market_data(data)
        mark_processed(idempotency_key)
    
    # Checkpoint — advance iterator to avoid replaying
    # (handled automatically by Lambda Kinesis trigger)
```

---

**EC9: How do you implement idempotency for file uploads to S3 in your NSE data pipeline, where the same file might be uploaded multiple times?**

**Answer:**
```python
import hashlib
import boto3

s3 = boto3.client('s3')

def upload_nse_file_idempotent(file_path: str, bucket: str) -> dict:
    """
    Content-addressed storage for idempotent file uploads.
    Files with the same content get the same S3 key.
    """
    with open(file_path, 'rb') as f:
        content = f.read()
    
    # Content-addressable key (SHA256 of file content)
    file_hash = hashlib.sha256(content).hexdigest()
    
    # Extract date from filename: NSE_BHAV_20250601.csv
    date_str = extract_date_from_filename(file_path)
    s3_key = f"nse/bhav/{date_str}/{file_hash}.csv"
    
    # Check if already uploaded (S3 HEAD request)
    try:
        s3.head_object(Bucket=bucket, Key=s3_key)
        print(f"File already uploaded: {s3_key}")
        return {'status': 'already_exists', 'key': s3_key}
    except s3.exceptions.ClientError as e:
        if e.response['Error']['Code'] != '404':
            raise
    
    # Upload with MD5 integrity check
    import base64
    md5 = base64.b64encode(hashlib.md5(content).digest()).decode()
    
    s3.put_object(
        Bucket=bucket,
        Key=s3_key,
        Body=content,
        ContentMD5=md5,  # S3 validates integrity
        Metadata={
            'source-file': file_path,
            'upload-timestamp': datetime.utcnow().isoformat(),
            'trading-date': date_str
        }
    )
    
    return {'status': 'uploaded', 'key': s3_key, 'hash': file_hash}
```

---

**EC10: A financial regulation audit requires you to prove that every NSE trade record was processed exactly once. How do you implement an audit trail for idempotent operations?**

**Answer:**
```python
from enum import Enum

class ProcessingStatus(str, Enum):
    RECEIVED = "RECEIVED"
    PROCESSING = "PROCESSING"
    COMPLETED = "COMPLETED"
    DUPLICATE_SKIPPED = "DUPLICATE_SKIPPED"
    FAILED = "FAILED"

class AuditableIdempotencyStore:
    def __init__(self):
        self.table = dynamodb.Table('IdempotencyAuditLog')
    
    async def record_processing(self, message_id: str, payload: dict, status: ProcessingStatus):
        await self.table.put_item(Item={
            'messageId': message_id,
            'status': status.value,
            'timestamp': datetime.utcnow().isoformat(),
            'payloadHash': hashlib.sha256(json.dumps(payload, sort_keys=True).encode()).hexdigest(),
            'lambdaRequestId': context.aws_request_id,
            'lambdaLogGroup': context.log_group_name,
            'ttl': int(time.time()) + (7 * 365 * 24 * 3600)  # 7-year retention for compliance
        })
    
    async def get_processing_history(self, message_id: str) -> List[dict]:
        """Returns full audit trail for a message ID"""
        response = await self.table.query(
            KeyConditionExpression='messageId = :id',
            ExpressionAttributeValues={':id': message_id}
        )
        return response['Items']

# Usage
async def lambda_handler(event, context):
    audit = AuditableIdempotencyStore()
    
    for record in event['Records']:
        message_id = record['messageId']
        payload = json.loads(record['body'])
        
        await audit.record_processing(message_id, payload, ProcessingStatus.RECEIVED)
        
        if await is_processed(message_id):
            await audit.record_processing(message_id, payload, ProcessingStatus.DUPLICATE_SKIPPED)
            continue
        
        try:
            await audit.record_processing(message_id, payload, ProcessingStatus.PROCESSING)
            await process_trade(payload)
            await audit.record_processing(message_id, payload, ProcessingStatus.COMPLETED)
        except Exception as e:
            await audit.record_processing(message_id, payload, ProcessingStatus.FAILED)
            raise
```

---

**EC11–EC20: [Condensed format for remaining edge cases]**

**EC11:** *Your Redis idempotency store grows unboundedly. How do you prevent memory exhaustion?*
- Use TTL on all keys (86400s for most operations). Implement a background job using `SCAN` + `TTL` to find keys without TTL. Use Redis `maxmemory-policy allkeys-lru` as a safety net. For long-lived operations (7-year audit), offload to DynamoDB with TTL.

**EC12:** *An idempotency key collision occurs between two different clients (UUID v4 collision probability ~10⁻³⁷). How do you namespace to prevent cross-tenant issues?*
- Always namespace keys: `{tenant_id}:{user_id}:{uuid}`. Validate UUID format at API boundary. Log and alert on any detected cross-tenant key conflicts.

**EC13:** *Your FastAPI NSE analysis endpoint calls Claude API. The Claude call succeeds but your DB write fails. On retry, Claude is called again wasting credits. How do you make the AI call idempotent?*
- Cache Claude responses keyed on `hash(prompt + model + temperature)` in Redis with 1-hour TTL. For deterministic prompts (same data → same analysis), this is valid. Expose a `force_refresh` parameter for explicit re-generation.

**EC14:** *SQS FIFO deduplication window is 5 minutes. A Lambda processes a message, crashes, and SQS doesn't redeliver for 6 minutes (past the dedup window). A new duplicate gets through. How do you handle this?*
- Don't rely on SQS deduplication alone. Always implement application-level idempotency in Lambda. SQS dedup is a first-line optimization, not a guarantee.

**EC15:** *You need idempotency across a saga with 5 steps. Step 3 fails on retry but steps 1 and 2 are non-reversible. How do you handle?*
- Each saga step stores its output keyed on `(saga_id, step_number)`. Retries replay each step but skip if output already stored. For non-reversible steps, compensating transactions must also be idempotent.

**EC16:** *A client generates a new UUID for every retry (bad client implementation). How do you detect and handle this at the server?*
- Implement **request fingerprinting**: hash of `(endpoint + user_id + normalized_body)`. If fingerprint matches an existing request within 60 seconds, treat as duplicate even without matching idempotency key. Log this as a client bug.

**EC17:** *Your idempotency check adds 30ms latency on every request due to Redis round-trip. How do you optimize?*
- Use Redis Cluster with read replicas for idempotency reads. Pipeline GET + SET commands. Use probabilistic early return: if the key clearly doesn't exist (new request), skip the lock dance. Async mark-as-processed (fire-and-forget) for low-risk operations.

**EC18:** *You're running a multi-threaded batch processor. Thread A and Thread B both read "not processed" for the same record simultaneously. Both proceed to write. How do you prevent this in a JVM/Node.js context?*
- In Node.js (single-threaded), async operations can interleave. Use `Promise` mutex patterns or `async-mutex` library. In multi-threaded JVM, use `ConcurrentHashMap.computeIfAbsent()` or database-level advisory locks.

**EC19:** *A downstream service your Lambda calls is not idempotent. Your Lambda retries and causes duplicate side effects in the downstream. How do you handle?*
- Wrap the downstream call in your own idempotency layer. Cache the downstream response keyed on your operation ID. On retry, return cached response without calling downstream again. If you can't cache (streaming response), use a separate DynamoDB record as a "downstream-called" flag.

**EC20:** *NSE data corrections are published as delta files (only changed records). Your idempotency store has the original records marked as processed. How do you ensure corrections are applied without treating them as duplicates?*
- Design your idempotency key to include a **content version** or **correction ID**. Original: `bhav-2025-06-01:original`. Correction: `bhav-2025-06-01:correction-1`. Each correction generates a new unique key and overrides the DynamoDB record with `put_item` (last writer wins for corrections).

---

### Must-Read Study Resources — Idempotency

1. **Stripe Idempotent Requests** — https://stripe.com/docs/api/idempotent_requests — Industry-standard implementation reference; explains the exact design Stripe uses including conflict handling
2. **AWS Lambda Powertools Idempotency** — https://docs.powertools.aws.dev/lambda/python/latest/utilities/idempotency/ — Production-grade idempotency for Lambda with DynamoDB persistence layer; directly applicable to your NSE pipeline
3. **"Idempotency is not a POST" — Marc's Blog** — https://www.mnot.net/blog/2013/05/22/idempotent — Deep RFC-level analysis of HTTP idempotency semantics; great for interview theory questions

---
---

## TOPIC 2: Knowledge Base (in AI/LLM Context)

---

### Common Interview Questions (20–30)

---

**Q1: What is a Knowledge Base in the context of LLMs and why is it needed?**

**Answer:**
An LLM's parametric knowledge is frozen at training time and limited to what was in the training corpus. A **Knowledge Base (KB)** is an external, updatable store of domain-specific information that the LLM can query at inference time to ground its responses.

**Why needed:**
- LLMs hallucinate on domain-specific facts not in training data
- Training data has a knowledge cutoff (Claude 3.5 Sonnet: early 2024)
- Private enterprise data (NSE filings, internal reports) cannot be in training data
- Knowledge can be updated without retraining

**Architecture:**

```
User Query → Retrieval System → Relevant Chunks → LLM + Context → Grounded Answer
                    ↑
            Knowledge Base
         (Vector DB + Documents)
```

---

**Q2: What is the difference between a Knowledge Base, RAG, and Fine-tuning?**

**Answer:**

| Aspect | Knowledge Base + RAG | Fine-tuning | Prompt Engineering |
|--------|---------------------|-------------|-------------------|
| Knowledge type | External, dynamic | Baked into weights | In-context |
| Update cost | Add documents | Retrain (~$$$) | Edit prompt |
| Factual accuracy | High (grounded) | Can hallucinate | Context-limited |
| Latency | Higher (retrieval) | Lower | Lowest |
| Best for | Private/dynamic data | Style, format, reasoning | Simple customization |
| Privacy | Data stays in your infra | Data sent for training | Prompt injection risk |

**When to use each:**
- **RAG/KB:** NSE real-time prices, SEBI filings, company financials
- **Fine-tuning:** Teaching a model to output in a specific JSON schema, or to reason like a financial analyst
- **Prompt engineering:** Customizing tone, output format, chain-of-thought style

---

**Q3: What is Amazon Bedrock Knowledge Base? How does it work?**

**Answer:**
Amazon Bedrock Knowledge Bases is a fully managed RAG service. You point it at an S3 bucket with your documents, it automatically:
1. Parses and chunks documents (PDF, HTML, CSV, Word)
2. Generates embeddings using Amazon Titan Embeddings or Cohere
3. Stores vectors in OpenSearch Serverless or Aurora PostgreSQL
4. Exposes a `retrieve` and `retrieve_and_generate` API

```python
import boto3

bedrock_agent = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

def query_nse_knowledge_base(question: str) -> dict:
    """
    Query Amazon Bedrock Knowledge Base for NSE market data.
    """
    response = bedrock_agent.retrieve_and_generate(
        input={'text': question},
        retrieveAndGenerateConfiguration={
            'type': 'KNOWLEDGE_BASE',
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': 'YOUR_KB_ID',
                'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20240620-v1:0',
                'retrievalConfiguration': {
                    'vectorSearchConfiguration': {
                        'numberOfResults': 10,
                        'overrideSearchType': 'HYBRID',  # Vector + BM25
                        'filter': {
                            'equals': {
                                'key': 'data_source',
                                'value': 'SEBI_FILINGS'
                            }
                        }
                    }
                },
                'generationConfiguration': {
                    'promptTemplate': {
                        'textPromptTemplate': """You are an NSE market analyst.
Answer the question based on the provided context about Indian financial markets.
Context: $search_results$
Question: $query$
Answer:"""
                    }
                }
            }
        }
    )
    
    return {
        'answer': response['output']['text'],
        'citations': response.get('citations', [])
    }

# Also available: retrieve-only (for custom generation)
def retrieve_only(question: str):
    response = bedrock_agent.retrieve(
        knowledgeBaseId='YOUR_KB_ID',
        retrievalQuery={'text': question},
        retrievalConfiguration={
            'vectorSearchConfiguration': {'numberOfResults': 5}
        }
    )
    return response['retrievalResults']
```

---

**Q4: How do you build a custom Knowledge Base with embeddings from scratch?**

**Answer:**
```python
# Full custom KB pipeline for NSE data
from anthropic import Anthropic
from sentence_transformers import SentenceTransformer
import psycopg2
from pgvector.psycopg2 import register_vector
import numpy as np
from typing import List, Dict

# 1. Embedding model
encoder = SentenceTransformer('BAAI/bge-large-en-v1.5')  # State-of-art as of 2025

# 2. PostgreSQL with pgvector
conn = psycopg2.connect(os.environ['DB_URL'])
register_vector(conn)

def setup_knowledge_base():
    with conn.cursor() as cur:
        cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
        cur.execute("""
            CREATE TABLE IF NOT EXISTS nse_knowledge_base (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                content TEXT NOT NULL,
                embedding vector(1024),  -- BGE-large embedding dim
                metadata JSONB,
                created_at TIMESTAMPTZ DEFAULT NOW()
            )
        """)
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_nse_kb_embedding 
            ON nse_knowledge_base USING ivfflat (embedding vector_cosine_ops)
            WITH (lists = 100)
        """)
        conn.commit()

# 3. Ingest documents
def ingest_sebi_circular(document_text: str, metadata: dict):
    chunks = chunk_document(document_text)  # See chunking section
    
    for chunk in chunks:
        embedding = encoder.encode(chunk, normalize_embeddings=True)
        
        with conn.cursor() as cur:
            cur.execute("""
                INSERT INTO nse_knowledge_base (content, embedding, metadata)
                VALUES (%s, %s, %s)
            """, (chunk, embedding.tolist(), json.dumps(metadata)))
        conn.commit()

# 4. Query the KB
def query_knowledge_base(question: str, top_k: int = 5) -> List[Dict]:
    query_embedding = encoder.encode(question, normalize_embeddings=True)
    
    with conn.cursor() as cur:
        cur.execute("""
            SELECT content, metadata, 
                   1 - (embedding <=> %s::vector) as cosine_similarity
            FROM nse_knowledge_base
            ORDER BY embedding <=> %s::vector
            LIMIT %s
        """, (query_embedding.tolist(), query_embedding.tolist(), top_k))
        
        return [
            {'content': row[0], 'metadata': row[1], 'score': float(row[2])}
            for row in cur.fetchall()
        ]

# 5. Generate answer with Claude
client = Anthropic()

def ask_knowledge_base(question: str) -> str:
    relevant_chunks = query_knowledge_base(question)
    
    context = "\n\n".join([
        f"[Source: {c['metadata'].get('source', 'Unknown')}]\n{c['content']}"
        for c in relevant_chunks
    ])
    
    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""Based on the following NSE/SEBI regulatory context, answer the question.
Only use information from the provided context.

Context:
{context}

Question: {question}

Answer:"""
        }]
    )
    return message.content[0].text
```

---

**Q5: What are the different chunking strategies and when do you use each?**

**Answer:**

**1. Fixed-size chunking:**
```python
def fixed_size_chunk(text: str, chunk_size: int = 512, overlap: int = 50) -> List[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        if chunk:
            chunks.append(chunk)
    return chunks
# Pros: Simple, predictable. Cons: Breaks mid-sentence, loses semantic coherence.
```

**2. Recursive character text splitting (LangChain default):**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],  # Priority order
    length_function=len,
)
# Tries to split at paragraph → sentence → word boundaries
chunks = splitter.split_text(sebi_circular_text)
```

**3. Semantic chunking (best quality, higher cost):**
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_community.embeddings import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(model_name='BAAI/bge-large-en-v1.5')
semantic_splitter = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile",  # Split where semantic shift > 95th percentile
    breakpoint_threshold_amount=95
)
chunks = semantic_splitter.split_text(document_text)
# Pros: Semantically coherent chunks. Cons: Expensive (N² embedding comparisons)
```

**4. Document-structure-aware chunking (for NSE structured data):**
```python
def chunk_nse_annual_report(pdf_text: str) -> List[Dict]:
    """
    Chunk based on document structure: sections, tables, footnotes separately
    """
    sections = extract_sections(pdf_text)  # regex + heuristics
    chunks = []
    
    for section_title, section_content in sections.items():
        if len(section_content) < 100:
            continue
        
        # Prepend section title to every chunk (context injection)
        for sub_chunk in recursive_split(section_content, 800, 100):
            chunks.append({
                'content': f"Section: {section_title}\n\n{sub_chunk}",
                'metadata': {
                    'section': section_title,
                    'document_type': 'annual_report',
                    'source': 'NSE'
                }
            })
    
    return chunks
```

**Comparison table:**

| Strategy | Quality | Speed | Cost | Best For |
|----------|---------|-------|------|----------|
| Fixed-size | Low | Fast | Low | Quick prototypes |
| Recursive | Medium | Fast | Low | General documents |
| Semantic | High | Slow | High | Narrative text |
| Structure-aware | High | Medium | Medium | PDFs, reports |

---

**Q6: What is the difference between sparse and dense embeddings?**

**Answer:**
- **Dense embeddings (neural):** 768–4096 dimensional float vectors from transformer encoders. Capture semantic meaning. Ex: "NSE" and "National Stock Exchange" are similar.
- **Sparse embeddings (BM25/TF-IDF):** High-dimensional (vocabulary-size) vectors with mostly zeros. Term frequency–based. Ex: exact keyword matches.

For financial data (NSE tickers like "NIFTY50", "RELIANCE.NS"), **hybrid search** is critical because:
- Dense: finds semantically related concepts
- Sparse: finds exact ticker symbols and regulatory codes (SEBI/2024/CIRCULAR/001)

---

**Q7: How does knowledge base retrieval work with metadata filtering?**

**Answer:**
```python
# pgvector with metadata filtering for NSE KB
async def search_with_filters(
    query: str,
    filters: dict,
    top_k: int = 10
) -> List[dict]:
    """
    Search NSE knowledge base with metadata pre-filtering.
    Example: Only search SEBI circulars from 2024 for a specific sector.
    """
    embedding = await generate_embedding(query)
    
    # Build metadata filter SQL
    where_clauses = []
    params = [embedding, top_k]
    
    if filters.get('year'):
        where_clauses.append(f"(metadata->>'year')::int = ${len(params)+1}")
        params.append(filters['year'])
    
    if filters.get('document_type'):
        where_clauses.append(f"metadata->>'document_type' = ${len(params)+1}")
        params.append(filters['document_type'])
    
    if filters.get('sector'):
        where_clauses.append(f"metadata->'sectors' ? ${len(params)+1}")
        params.append(filters['sector'])
    
    where_sql = "WHERE " + " AND ".join(where_clauses) if where_clauses else ""
    
    query_sql = f"""
        SELECT content, metadata,
               1 - (embedding <=> $1::vector) as score
        FROM nse_knowledge_base
        {where_sql}
        ORDER BY embedding <=> $1::vector
        LIMIT $2
    """
    
    return await db.fetch(query_sql, *params)

# Usage
results = await search_with_filters(
    query="insider trading regulations",
    filters={'year': 2024, 'document_type': 'SEBI_CIRCULAR', 'sector': 'BANKING'},
    top_k=5
)
```

---

**Q8: What is the difference between knowledge bases for structured vs unstructured data?**

**Answer:**

**Unstructured (documents, PDFs, text):** Traditional RAG with embeddings + vector search.

**Structured (NSE price tables, financial ratios):**
```python
# Option 1: Text2SQL — convert natural language to SQL
async def structured_kb_query(question: str) -> dict:
    """
    For structured NSE data, generate SQL instead of vector search.
    """
    schema = """
    Tables:
    - nse_daily_prices (symbol, date, open, high, low, close, volume)
    - nse_company_info (symbol, name, sector, market_cap, pe_ratio)
    - sebi_filings (company_id, filing_date, filing_type, document_url)
    """
    
    response = anthropic_client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=500,
        system=f"You are a SQL expert for Indian stock market data. Schema: {schema}",
        messages=[{
            "role": "user",
            "content": f"Generate a PostgreSQL query for: {question}\nReturn only the SQL, no explanation."
        }]
    )
    
    sql = response.content[0].text.strip()
    
    # SECURITY: Validate SQL before execution (read-only, no DDL)
    if not is_safe_query(sql):
        raise ValueError("Generated SQL failed security validation")
    
    results = await db.fetch(sql)
    return {'sql': sql, 'results': results}

def is_safe_query(sql: str) -> bool:
    """Whitelist-based SQL safety check"""
    sql_upper = sql.upper().strip()
    forbidden = ['DROP', 'DELETE', 'UPDATE', 'INSERT', 'ALTER', 'CREATE', 'TRUNCATE', '--', ';--']
    return (
        sql_upper.startswith('SELECT') and
        not any(word in sql_upper for word in forbidden)
    )
```

**Option 2: Hybrid KB — route by query type:**
```python
async def intelligent_kb_router(question: str) -> str:
    """Route to structured or unstructured KB based on query classification"""
    
    classification = await classify_query(question)
    
    if classification == 'QUANTITATIVE':
        # Price queries, ratios, comparisons → SQL
        return await structured_kb_query(question)
    elif classification == 'REGULATORY':
        # Policy, circular, regulation → RAG
        return await rag_kb_query(question)
    else:
        # Mixed → both, combine results
        structured = await structured_kb_query(question)
        unstructured = await rag_kb_query(question)
        return await merge_and_synthesize(question, structured, unstructured)
```

---

**Q9: What is a knowledge graph and how does it differ from a vector knowledge base?**

**Answer:**
- **Vector KB:** Stores chunks with embeddings. Retrieves by similarity. Best for "find me relevant passages."
- **Knowledge Graph:** Stores entities and relationships as nodes/edges. Retrieves by traversal. Best for "how are these entities related?"

```python
# GraphRAG — combining both (Microsoft Research, 2024)
# Knowledge graph + vector search for complex multi-hop questions

# Example: "What is the relationship between SEBI's 2024 FII regulations 
#           and NIFTY50 volatility?"
# 
# Vector search: finds relevant SEBI circulars AND volatility research
# Graph traversal: FII_REGULATION → affects → FOREIGN_INVESTMENT → 
#                  influences → NIFTY50 → measured_by → VOLATILITY_INDEX

# For NSE platform: Build a graph of
# COMPANY → listed_on → NSE
# COMPANY → filed_with → SEBI
# COMPANY → belongs_to → SECTOR
# SECTOR → correlated_with → SECTOR (correlation matrix)
```

---

**Q10: How do you handle knowledge base freshness and updates?**

**Answer:**
```python
# Incremental KB updates for NSE data
class NSEKnowledgeBaseManager:
    
    async def update_document(self, doc_id: str, new_content: str, metadata: dict):
        """
        Update a document in the KB:
        1. Delete old chunks for this document
        2. Rechunk and re-embed new content  
        3. Insert new chunks
        """
        async with db.transaction():
            # Delete old chunks
            await db.execute(
                "DELETE FROM nse_knowledge_base WHERE metadata->>'document_id' = $1",
                doc_id
            )
            
            # Rechunk and re-embed
            chunks = self.chunk_document(new_content)
            embeddings = await self.batch_embed(chunks)
            
            # Bulk insert
            await db.executemany("""
                INSERT INTO nse_knowledge_base (content, embedding, metadata)
                VALUES ($1, $2, $3)
            """, [
                (chunk, emb.tolist(), json.dumps({**metadata, 'document_id': doc_id}))
                for chunk, emb in zip(chunks, embeddings)
            ])
    
    async def schedule_nse_refresh(self):
        """
        Daily job: pull new BHAV copy, SEBI circulars, and update KB
        """
        today = datetime.now().strftime('%Y-%m-%d')
        
        # Fetch new SEBI circulars published today
        new_circulars = await fetch_sebi_circulars(date=today)
        for circular in new_circulars:
            await self.update_document(
                doc_id=circular['id'],
                new_content=circular['text'],
                metadata={
                    'type': 'SEBI_CIRCULAR',
                    'date': today,
                    'circular_number': circular['number']
                }
            )
```

---

**Q11: What embedding models are best for financial/domain-specific knowledge bases?**

**Answer:**

| Model | Dimensions | MTEB Score (2025) | Best For |
|-------|-----------|------------------|----------|
| `text-embedding-3-large` (OpenAI) | 3072 | 64.6 | General, multilingual |
| `BAAI/bge-large-en-v1.5` | 1024 | 63.9 | English, open-source |
| `Cohere embed-v3` | 1024 | 64.5 | Production, multilingual |
| `voyage-finance-2` (Voyage AI) | 1024 | ~65+ | **Finance-specific** |
| `Amazon Titan Embeddings v2` | 1024 | ~62 | AWS-native |

For your NSE platform: **`voyage-finance-2`** is trained specifically on financial documents (10-K, earnings reports, regulatory filings) and outperforms general embeddings on financial retrieval tasks by ~8-12% on domain-specific benchmarks.

```python
# Using Voyage AI finance embeddings
import voyageai

voyage_client = voyageai.Client(api_key=os.environ['VOYAGE_API_KEY'])

def embed_financial_document(text: str) -> List[float]:
    result = voyage_client.embed(
        [text],
        model="voyage-finance-2",
        input_type="document"  # vs "query" for search queries
    )
    return result.embeddings[0]

def embed_financial_query(query: str) -> List[float]:
    result = voyage_client.embed(
        [query],
        model="voyage-finance-2",
        input_type="query"  # Asymmetric embeddings — query vs document differ
    )
    return result.embeddings[0]
```

---

**Q12: What is the difference between a knowledge base and a vector store?**

**Answer:**
- **Vector store:** Storage layer — stores and retrieves embeddings (Pinecone, pgvector, Chroma, Weaviate)
- **Knowledge base:** Higher-level abstraction — includes documents, metadata, chunking logic, access control, and the retrieval pipeline built on top of a vector store

A knowledge base = vector store + document management + retrieval logic + (optionally) generation.

---

**Q13: How does Anthropic's Claude handle knowledge through system prompts vs retrieval?**

**Answer:**
Claude supports:
1. **System prompt:** Static context injected at conversation start (~200k token context for Claude 3.5 Sonnet). Good for small KBs.
2. **Tool use (function calling):** Claude calls a `search_knowledge_base` tool mid-conversation to retrieve relevant chunks on-demand.
3. **Retrieved context in messages:** Pre-retrieved chunks injected into the user message.

```python
# Claude with tool-use for dynamic KB retrieval
import anthropic

client = anthropic.Anthropic()

tools = [{
    "name": "search_nse_knowledge_base",
    "description": "Search the NSE/SEBI knowledge base for regulatory information, company filings, and market data",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "The search query"},
            "filters": {
                "type": "object",
                "properties": {
                    "year": {"type": "integer"},
                    "document_type": {"type": "string", "enum": ["SEBI_CIRCULAR", "ANNUAL_REPORT", "BHAV_COPY"]}
                }
            }
        },
        "required": ["query"]
    }
}]

def claude_with_kb(user_question: str) -> str:
    messages = [{"role": "user", "content": user_question}]
    
    while True:
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )
        
        if response.stop_reason == "tool_use":
            # Claude wants to search the KB
            tool_use = next(b for b in response.content if b.type == "tool_use")
            search_results = search_nse_knowledge_base(**tool_use.input)
            
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": json.dumps(search_results)
                }]
            })
        else:
            # Final text response
            return response.content[0].text
```

---

**Q14: What is the context window limit challenge in knowledge base retrieval?**

**Answer:**
Even with large context windows (200k tokens for Claude 3.5 Sonnet), stuffing too many chunks degrades performance — "lost in the middle" problem (Liu et al., 2023): LLMs perform worst on information in the middle of long contexts.

**Strategies:**
1. **Limit chunks** to top-5 to top-10 most relevant
2. **Re-rank** and keep only the most relevant
3. **Map-reduce:** Process large KBs in chunks, summarize each, then synthesize
4. **Hypothetical Document Embeddings (HyDE):** Generate a hypothetical answer, embed it, use that for retrieval

---

**Q15: Explain vector index types: IVFFlat vs HNSW in pgvector.**

**Answer:**

| Index | Build Time | Query Time | Memory | Accuracy | Best For |
|-------|-----------|-----------|--------|----------|----------|
| IVFFlat | Fast | Medium | Low | ~95% | Large datasets, memory-constrained |
| HNSW | Slow | **Very fast** | High | ~99% | Production retrieval, speed-critical |

```sql
-- IVFFlat (requires training data first)
CREATE INDEX ON nse_knowledge_base USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- ~sqrt(row_count)

-- HNSW (build once, fast queries, recommended for production)
CREATE INDEX ON nse_knowledge_base USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Query with probe tuning
SET ivfflat.probes = 10;  -- Higher = more accurate but slower
SET hnsw.ef_search = 100; -- Higher = more accurate but slower
```

---

**Q16: How does Amazon Bedrock Knowledge Base handle automatic syncing?**

**Answer:**
Bedrock KB can be configured to sync automatically when S3 objects change (via EventBridge triggers). You can also trigger sync via API.

```python
import boto3

bedrock_agent = boto3.client('bedrock-agent')

def trigger_kb_sync(knowledge_base_id: str, data_source_id: str):
    response = bedrock_agent.start_ingestion_job(
        knowledgeBaseId=knowledge_base_id,
        dataSourceId=data_source_id,
        description=f"Sync triggered at {datetime.utcnow().isoformat()}"
    )
    return response['ingestionJob']['ingestionJobId']

def check_sync_status(knowledge_base_id: str, data_source_id: str, job_id: str):
    response = bedrock_agent.get_ingestion_job(
        knowledgeBaseId=knowledge_base_id,
        dataSourceId=data_source_id,
        ingestionJobId=job_id
    )
    return response['ingestionJob']['status']  # STARTING, IN_PROGRESS, COMPLETE, FAILED
```

---

**Q17: What is a parent-child chunking strategy?**

**Answer:**
Store small chunks for precise retrieval, but return their larger parent chunks for richer context.

```python
# LangChain ParentDocumentRetriever equivalent
class ParentChildChunker:
    def __init__(self, parent_chunk_size=2000, child_chunk_size=400):
        self.parent_splitter = RecursiveCharacterTextSplitter(chunk_size=parent_chunk_size)
        self.child_splitter = RecursiveCharacterTextSplitter(chunk_size=child_chunk_size)
    
    def chunk_document(self, doc_id: str, text: str) -> dict:
        parent_chunks = self.parent_splitter.split_text(text)
        
        result = {'parents': [], 'children': []}
        
        for parent_idx, parent_chunk in enumerate(parent_chunks):
            parent_id = f"{doc_id}#parent#{parent_idx}"
            result['parents'].append({
                'id': parent_id,
                'content': parent_chunk  # Stored in docstore, not vector DB
            })
            
            child_chunks = self.child_splitter.split_text(parent_chunk)
            for child_chunk in child_chunks:
                result['children'].append({
                    'content': child_chunk,  # Stored in vector DB
                    'parent_id': parent_id,  # Reference to parent
                    'metadata': {'parent_id': parent_id, 'doc_id': doc_id}
                })
        
        return result
    
    def retrieve(self, query: str, top_k: int = 5) -> List[str]:
        # Search in child chunks (small, precise)
        matching_children = vector_search(query, top_k=top_k * 3)
        
        # Return parent chunks (large, context-rich)
        parent_ids = list(set(c['metadata']['parent_id'] for c in matching_children))
        return [self.get_parent(pid) for pid in parent_ids[:top_k]]
```

---

**Q18: How do you handle multi-modal knowledge bases (text + tables + images in financial reports)?**

**Answer:**
```python
# Multi-modal KB for NSE annual reports with charts and tables
from anthropic import Anthropic
import base64

client = Anthropic()

async def extract_table_from_image(image_bytes: bytes) -> str:
    """Use Claude Vision to extract table data from financial report images"""
    image_b64 = base64.standard_b64encode(image_bytes).decode('utf-8')
    
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": image_b64
                    }
                },
                {
                    "type": "text",
                    "text": "Extract all data from this financial table in JSON format. Include all row and column headers."
                }
            ]
        }]
    )
    return response.content[0].text

async def ingest_annual_report_multimodal(pdf_path: str, company: str):
    """
    1. Extract text pages → chunk → embed → store
    2. Extract tables (as structured JSON) → embed table description → store
    3. Extract charts (as Claude Vision description) → embed → store
    """
    pages = extract_pdf_pages(pdf_path)  # [(text, images_on_page)]
    
    for page_num, (text, images) in enumerate(pages):
        # Text chunks
        if text.strip():
            for chunk in semantic_chunk(text):
                await store_chunk(chunk, {
                    'company': company, 'page': page_num, 'type': 'text'
                })
        
        # Tables extracted from images
        for img in images:
            if is_table_image(img):
                table_json = await extract_table_from_image(img)
                table_description = f"Financial table from {company} annual report, page {page_num}:\n{table_json}"
                await store_chunk(table_description, {
                    'company': company, 'page': page_num, 'type': 'table',
                    'raw_json': table_json
                })
```

---

**Q19: What is the LLM-as-judge approach for evaluating knowledge base quality?**

**Answer:**
```python
# Use Claude to evaluate retrieval quality (RAGAS-style)
async def evaluate_kb_retrieval(
    question: str,
    retrieved_chunks: List[str],
    ground_truth: str
) -> dict:
    
    context = "\n\n".join(retrieved_chunks)
    
    # Faithfulness: Is the answer supported by the context?
    faithfulness_prompt = f"""
    Context: {context}
    Ground truth answer: {ground_truth}
    
    Rate on a scale of 1-5: How well is the ground truth answer supported by the context?
    Return JSON: {{"score": int, "explanation": str}}
    """
    
    # Context Precision: Are the retrieved chunks relevant?
    precision_prompt = f"""
    Question: {question}
    Retrieved chunks: {context}
    
    For each chunk, rate relevance (1-5) and explain.
    Return JSON: {{"chunk_scores": [int], "overall_precision": float}}
    """
    
    results = await asyncio.gather(
        call_claude(faithfulness_prompt),
        call_claude(precision_prompt)
    )
    
    return {
        'faithfulness': json.loads(results[0]),
        'context_precision': json.loads(results[1])
    }
```

---

**Q20: How does knowledge base caching work to reduce latency and cost?**

**Answer:**
```python
# Semantic caching — cache KB answers for similar queries
class SemanticCache:
    def __init__(self, similarity_threshold: float = 0.95):
        self.threshold = similarity_threshold
    
    async def get(self, query: str) -> Optional[str]:
        query_embedding = await embed(query)
        
        # Find cached queries with high similarity
        result = await db.fetchrow("""
            SELECT answer, 1 - (query_embedding <=> $1::vector) as similarity
            FROM query_cache
            WHERE 1 - (query_embedding <=> $1::vector) > $2
            ORDER BY similarity DESC
            LIMIT 1
        """, query_embedding, self.threshold)
        
        return result['answer'] if result else None
    
    async def set(self, query: str, answer: str, ttl_hours: int = 24):
        embedding = await embed(query)
        await db.execute("""
            INSERT INTO query_cache (query, query_embedding, answer, expires_at)
            VALUES ($1, $2, $3, NOW() + INTERVAL '$4 hours')
        """, query, embedding, answer, ttl_hours)

# Anthropic prompt caching (for large system prompts / KB context)
# Cache a large NSE context in Claude's prompt cache
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_nse_context,  # Up to 200k tokens
            "cache_control": {"type": "ephemeral"}  # Cache this prefix
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)
# First call: ~$0.003/1k input tokens (writes cache)
# Subsequent calls: ~$0.0003/1k input tokens (reads cache — 10x cheaper)
```

---

### Deep-Dive Real-World Edge Case Questions (20)

---

**EC1: Your NSE knowledge base has 10 million chunks. Vector search takes 800ms — too slow for real-time queries. How do you optimize?**

**Answer:**
Multi-level optimization strategy:

```python
# 1. Pre-filtering with metadata (reduce search space)
# Instead of searching ALL 10M chunks:
# Filter to relevant document type + date range first
async def optimized_search(query: str, filters: dict) -> List[dict]:
    
    # Step 1: Metadata pre-filter (reduces 10M → ~100K chunks)
    filtered_ids = await db.fetch("""
        SELECT id FROM nse_knowledge_base
        WHERE metadata->>'document_type' = $1
          AND (metadata->>'year')::int >= $2
    """, filters['doc_type'], filters['min_year'])
    
    # Step 2: Vector search only within filtered set
    filtered_id_list = [r['id'] for r in filtered_ids]
    
    # Step 3: Use HNSW index with pre-filter
    results = await db.fetch("""
        SELECT content, metadata,
               1 - (embedding <=> $1::vector) as score
        FROM nse_knowledge_base
        WHERE id = ANY($2)
        ORDER BY embedding <=> $1::vector
        LIMIT $3
    """, query_embedding, filtered_id_list, 10)
    
    return results

# 2. Approximate Nearest Neighbor tuning
# Reduce HNSW ef_search for speed/accuracy tradeoff
await db.execute("SET hnsw.ef_search = 40")  # Default 40, max 1000

# 3. Read replicas for search workload
# Route search queries to read replica
search_db = create_engine(READ_REPLICA_URL)

# 4. Async batch embeddings
# Pre-compute embeddings for common queries
```

---

**EC2: A user asks about a recent SEBI circular published yesterday, but your KB was last synced 48 hours ago. How do you handle stale knowledge?**

**Answer:**
```python
class FreshnessAwareKB:
    MAX_STALENESS_HOURS = 24
    
    async def query(self, question: str, require_fresh: bool = False) -> dict:
        # Check KB freshness
        last_sync = await self.get_last_sync_time()
        hours_since_sync = (datetime.utcnow() - last_sync).total_seconds() / 3600
        
        if hours_since_sync > self.MAX_STALENESS_HOURS:
            if require_fresh:
                # Trigger sync and wait
                await self.sync_and_wait(timeout=120)
            else:
                # Add staleness warning to response
                warning = f"⚠️ Knowledge base last updated {int(hours_since_sync)}h ago. Recent developments may not be reflected."
        
        # Try KB first
        kb_results = await self.vector_search(question)
        
        # If low confidence, supplement with live search
        if not kb_results or max(r['score'] for r in kb_results) < 0.7:
            live_data = await self.fetch_live_sebi_data(question)
            return self.merge_results(kb_results, live_data, warning)
        
        return {'results': kb_results, 'warning': warning if hours_since_sync > self.MAX_STALENESS_HOURS else None}
    
    async def fetch_live_sebi_data(self, query: str) -> List[dict]:
        """
        Fallback: fetch from SEBI API directly for real-time regulatory data
        """
        # SEBI Open Data API
        response = await httpx.get(
            'https://www.sebi.gov.in/api/circulars',
            params={'q': query, 'date_from': (datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d')}
        )
        return response.json()['results']
```

---

**EC3: Two very different questions produce the same top-3 retrieved chunks due to a poorly calibrated embedding model. How do you diagnose and fix this?**

**Answer:**
```python
# Diagnosis: Embedding collapse analysis
async def diagnose_retrieval_quality(test_questions: List[dict]) -> dict:
    """
    test_questions: [{'question': str, 'expected_topic': str}]
    """
    results = {'collisions': [], 'avg_similarity': 0, 'issues': []}
    
    all_retrievals = []
    for item in test_questions:
        chunks = await retrieve(item['question'], top_k=3)
        all_retrievals.append({
            'question': item['question'],
            'expected': item['expected_topic'],
            'retrieved_ids': [c['id'] for c in chunks],
            'scores': [c['score'] for c in chunks]
        })
    
    # Check for retrieval collisions (different questions → same chunks)
    chunk_id_sets = [frozenset(r['retrieved_ids']) for r in all_retrievals]
    for i, set_a in enumerate(chunk_id_sets):
        for j, set_b in enumerate(chunk_id_sets[i+1:], i+1):
            overlap = len(set_a & set_b) / len(set_a | set_b)
            if overlap > 0.5:
                results['collisions'].append({
                    'q1': test_questions[i]['question'],
                    'q2': test_questions[j]['question'],
                    'overlap': overlap
                })
    
    return results

# Fixes:
# 1. Switch to domain-specific embedding model (voyage-finance-2)
# 2. Add query-type prefix: "financial_query: {query}" vs "regulation_query: {query}"
# 3. Implement MMR (Maximal Marginal Relevance) for diverse results
from langchain.vectorstores import PGVector
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={'k': 5, 'fetch_k': 20, 'lambda_mult': 0.5}
    # lambda_mult: 0=max diversity, 1=max relevance
)
```

---

**EC4: Your KB contains contradictory information — an older SEBI circular says X, a newer one supersedes it with Y. The LLM gets confused. How do you handle temporal knowledge conflicts?**

**Answer:**
```python
# Temporal-aware retrieval with conflict detection
async def temporally_aware_retrieve(query: str) -> dict:
    
    # Retrieve with timestamps
    chunks = await db.fetch("""
        SELECT content, metadata,
               1 - (embedding <=> $1::vector) as score,
               (metadata->>'date')::date as doc_date
        FROM nse_knowledge_base
        ORDER BY embedding <=> $1::vector
        LIMIT 20
    """, query_embedding)
    
    # Group by topic, keep most recent
    topic_groups = group_by_topic(chunks)  # Cluster similar chunks
    
    deduplicated = []
    for group in topic_groups:
        # Keep the most recent document for each topic
        most_recent = max(group, key=lambda c: c['doc_date'])
        
        # Flag if older versions exist (for transparency)
        if len(group) > 1:
            most_recent['supersedes'] = [c['metadata']['doc_id'] for c in group[1:]]
        
        deduplicated.append(most_recent)
    
    # Prompt engineering for temporal reasoning
    context = format_context_with_dates(deduplicated[:5])
    
    prompt = f"""You are analyzing NSE/SEBI regulatory documents.
The following context includes documents with their publication dates.
When documents contradict each other, ALWAYS follow the more recent document.
If you identify a conflict, explicitly state which document supersedes the other.

Context:
{context}

Question: {query}"""
    
    return await call_claude(prompt)
```

---

**EC5: You need to ingest 50,000 NSE/SEBI PDF documents into your knowledge base. The embedding API has rate limits. How do you handle bulk ingestion efficiently?**

**Answer:**
```python
import asyncio
from asyncio import Semaphore
import aiofiles
import httpx

class BulkIngestionPipeline:
    def __init__(self, max_concurrent: int = 10, rate_limit_rpm: int = 1000):
        self.semaphore = Semaphore(max_concurrent)
        self.rate_limiter = AsyncRateLimiter(rate_limit_rpm)
    
    async def ingest_all(self, pdf_paths: List[str]) -> dict:
        # Process in batches with progress tracking
        total = len(pdf_paths)
        processed = 0
        failed = []
        
        tasks = [self.ingest_single(path) for path in pdf_paths]
        
        for coro in asyncio.as_completed(tasks):
            try:
                result = await coro
                processed += 1
                if processed % 100 == 0:
                    print(f"Progress: {processed}/{total}")
            except Exception as e:
                failed.append(str(e))
        
        return {'processed': processed, 'failed': len(failed), 'errors': failed}
    
    async def ingest_single(self, pdf_path: str):
        async with self.semaphore:  # Limit concurrency
            text = await extract_text(pdf_path)
            chunks = self.chunk(text)
            
            # Batch embed (more efficient than one-by-one)
            batch_size = 100
            for i in range(0, len(chunks), batch_size):
                batch = chunks[i:i + batch_size]
                
                await self.rate_limiter.acquire()  # Respect rate limits
                embeddings = await self.embed_batch(batch)
                await self.store_batch(batch, embeddings, pdf_path)
    
    async def embed_batch(self, texts: List[str]) -> List[List[float]]:
        """Voyage AI supports batch embedding"""
        result = voyage_client.embed(
            texts,
            model="voyage-finance-2",
            input_type="document",
            truncation=True
        )
        return result.embeddings

class AsyncRateLimiter:
    def __init__(self, rpm: int):
        self.rpm = rpm
        self.tokens = rpm
        self.last_refill = time.time()
        self.lock = asyncio.Lock()
    
    async def acquire(self):
        async with self.lock:
            now = time.time()
            elapsed = now - self.last_refill
            # Refill tokens
            self.tokens = min(self.rpm, self.tokens + elapsed * (self.rpm / 60))
            self.last_refill = now
            
            if self.tokens < 1:
                wait_time = (1 - self.tokens) * 60 / self.rpm
                await asyncio.sleep(wait_time)
                self.tokens = 0
            else:
                self.tokens -= 1
```

---

**EC6–EC20:** *(Condensed for remaining edge cases)*

**EC6:** *A user queries in Hindi about NSE regulations. Your KB only has English documents. How do you handle multilingual queries?*
- Translate query to English before embedding (using a multilingual model or Claude). OR use a multilingual embedding model like `multilingual-e5-large`. Return answer in the user's language by instructing Claude.

**EC7:** *Your pgvector instance runs out of memory during HNSW index build on 10M vectors. What do you do?*
- Switch to IVFFlat (lower memory), or build on a larger instance temporarily. Consider partitioning the table by `document_type` and building separate indexes. Use streaming ingestion with batch index updates rather than full rebuilds.

**EC8:** *The retrieval keeps returning the same large document because it dominates the corpus. How do you ensure diversity?*
- Implement MMR (Maximal Marginal Relevance). Add a `document_weight` column and penalize documents that are over-represented. Use namespace/namespace isolation in vector store.

**EC9:** *A security audit finds that users can retrieve confidential SEBI insider trading investigation documents by clever query crafting. How do you implement access control on your KB?*
- Implement document-level ACLs. Tag every chunk with `access_level`. Filter by `access_level <= user.clearance_level` in every query. Never expose chunk IDs to users (they could enumerate). Use row-level security (RLS) in PostgreSQL.

**EC10:** *Your KB embedding model is updated to a new version (bge-large-en-v1.5 → v2.0). Embeddings from old and new models are incompatible. How do you migrate?*
- Keep both models active during migration. Re-embed all documents with new model in background. Use `model_version` metadata column. Query against both indexes until migration complete (AB-style), then cut over and drop old embeddings. Expect ~1-2 weeks for 10M chunks.

**EC11:** *How do you handle very short queries like "NIFTY PE" that lack sufficient context for meaningful vector search?*
- Implement **query expansion**: use Claude to generate a more detailed version of the query before embedding. Or use HyDE (generate a hypothetical answer, embed that). Short queries often benefit from keyword (BM25) search more than vector search.

**EC12:** *Your knowledge base returns hallucinated citations — claiming to reference a document that doesn't exist. How do you prevent this?*
- Never let the LLM generate citations. Only inject retrieved chunk metadata as facts. Post-process responses to verify all cited document IDs exist in your KB. Use structured output (JSON) with explicit citation fields tied to retrieved chunks.

**EC13:** *A regulatory document has been revoked/repealed. How do you ensure the KB reflects this and doesn't return invalidated information?*
- Add `is_revoked: bool` and `revoked_date` fields to chunk metadata. Exclude revoked documents from retrieval by default (`WHERE metadata->>'is_revoked' != 'true'`). For research queries, allow revoked documents with a clear `[REVOKED]` label.

**EC14:** *Your NSE KB query latency spikes during market hours (9:15 AM–3:30 PM IST). How do you handle load?*
- Pre-warm connection pools. Use read replicas with auto-scaling. Cache popular queries (semantic cache). Pre-compute answers for top-100 most frequently asked questions during off-hours. Use CloudFront/CDN for static regulatory content.

**EC15:** *The same regulatory concept appears in 200 different documents with slightly different wording. How do you prevent the retrieval from returning 5 chunks all saying the same thing?*
- Post-retrieval diversity filter: compute pairwise cosine similarity between retrieved chunks, filter out pairs with similarity > 0.9, keeping only the highest-scored unique chunk. This is part of MMR.

**EC16:** *Your KB is used for both regulatory compliance queries and trading strategy queries. The same query can legitimately return different content depending on intent. How do you handle query intent?*
- Classify query intent first (compliance vs strategy vs data vs education). Route to separate KB namespaces or apply different metadata filters based on intent. Use different prompt templates per intent.

**EC17:** *A SEBI circular references 15 other circulars. How do you build cross-reference retrieval?*
- Build a knowledge graph alongside the vector KB. When ingesting, parse cross-references and create graph edges. Use GraphRAG: combine vector search with graph traversal. LlamaIndex's `KnowledgeGraphIndex` supports this natively.

**EC18:** *The LLM confidently answers a question using retrieved context, but the answer is wrong because the retrieved chunk was truncated mid-sentence and lost the negation ("...is NOT required" became "...is required"). How do you prevent this?*
- Increase chunk overlap to prevent boundary truncation. Use sentence-aware splitting (never break in mid-sentence). Store larger parent chunks and retrieve those (parent-child strategy). Post-process: if a chunk ends mid-sentence, extend it to the next sentence boundary.

**EC19:** *Your knowledge base costs $500/month in embedding API calls for daily syncs. How do you reduce cost without sacrificing quality?*
- Only re-embed **changed** documents (content hash comparison). Use cheaper embedding models for low-importance documents and expensive models for high-priority regulatory content. Batch all embedding calls. Use Amazon Bedrock Titan Embeddings (cheaper) for initial ingestion and Voyage Finance for production queries.

**EC20:** *Two users ask the same question but one is a retail investor and one is a professional fund manager. Should the KB return different content?*
- Yes. Implement **persona-aware retrieval**: tag chunks with `audience_level` (retail/professional/institutional). Filter or re-rank based on user profile. Use different system prompts that adjust explanation depth. Claude can naturally adjust complexity based on system prompt persona.

---

### Must-Read Study Resources — Knowledge Base

1. **Anthropic Claude Tool Use (for KB integration)** — https://docs.anthropic.com/en/docs/build-with-claude/tool-use — Official docs for building Claude-powered KB systems with tool use / function calling
2. **Amazon Bedrock Knowledge Bases** — https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html — Fully managed KB service; directly relevant if deploying on AWS
3. **LlamaIndex Knowledge Base Guide** — https://docs.llamaindex.ai/en/stable/understanding/putting_it_all_together/q_and_a/ — Comprehensive patterns for building production KBs with structured and unstructured data

---
---

## TOPIC 3: Transformer Model

---

### Common Interview Questions (20–30)

---

**Q1: Explain the Transformer architecture at a high level. What problem did it solve?**

**Answer:**
Transformers (Vaswani et al., "Attention Is All You Need", 2017) replaced RNNs/LSTMs which processed tokens **sequentially** — making them slow and poor at long-range dependencies (gradients vanish over long sequences).

Transformers process **all tokens in parallel** using **attention** to learn which tokens are relevant to each other, regardless of distance.

```
Input: "NIFTY rose sharply because FII buying increased"
         ↕        ↕         ↕       ↕    ↕
Every token attends to every other token simultaneously
(RNN would process left-to-right, forgetting "NIFTY" by the time it reads "increased")
```

**Architecture:**
```
Input Embeddings
    +
Positional Encoding
    ↓
[Encoder Block × N]              [Decoder Block × N]
  - Multi-Head Self-Attention       - Masked Multi-Head Self-Attention
  - Feed-Forward Network            - Multi-Head Cross-Attention (→ encoder)
  - Add & Norm (residual)           - Feed-Forward Network
                                    - Add & Norm

Encoder output → Decoder cross-attention → Output logits → Softmax → Token
```

---

**Q2: What is self-attention and how does it work mathematically?**

**Answer:**
Self-attention computes a weighted sum of values, where weights are determined by query-key dot products.

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

Where:
- **Q (Query):** What am I looking for?
- **K (Key):** What do I offer?
- **V (Value):** What information do I carry?
- $d_k$: Key dimension (scaling factor to prevent softmax saturation)

```python
import numpy as np

def scaled_dot_product_attention(Q, K, V, mask=None):
    """
    Q: (batch, heads, seq_len, d_k)
    K: (batch, heads, seq_len, d_k)  
    V: (batch, heads, seq_len, d_v)
    """
    d_k = Q.shape[-1]
    
    # Compute attention scores
    scores = np.matmul(Q, K.transpose(-2, -1)) / np.sqrt(d_k)
    # scores: (batch, heads, seq_len_q, seq_len_k)
    
    # Apply mask (for decoder — prevent attending to future tokens)
    if mask is not None:
        scores = np.where(mask == 0, -1e9, scores)
    
    # Softmax to get attention weights
    attention_weights = np.exp(scores) / np.sum(np.exp(scores), axis=-1, keepdims=True)
    
    # Weighted sum of values
    output = np.matmul(attention_weights, V)
    # output: (batch, heads, seq_len, d_v)
    
    return output, attention_weights

# Intuition for NSE context:
# Query: "What caused NIFTY to fall?"
# Token "fall" queries all other tokens
# High attention scores for tokens: "FII", "selling", "US Fed", "rate hike"
# Low attention scores for: "the", "was", "at"
```

---

**Q3: What is Multi-Head Attention? Why use multiple heads?**

**Answer:**
Multi-Head Attention runs $h$ parallel attention mechanisms with different learned projections, then concatenates and projects the results.

$$\text{MultiHead}(Q,K,V) = \text{Concat}(\text{head}_1,...,\text{head}_h)W^O$$
$$\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$$

**Why multiple heads:**
- Each head can
