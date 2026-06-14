

Fetched 3 resources

Good, I have strong context. Now let me deliver the comprehensive cheat sheet.

---

# Senior/Lead Full-Stack Interview Cheat Sheet
## Next.js · Node.js · AWS · PostgreSQL/SQL Server

---

## SECTION 1 — THE "TRAP" QUESTIONS

---

### TRAP 1: The Next.js App Router Streaming Race Condition

**Q:** You have a streaming Server Component using `<Suspense>`. A child async Server Component throws mid-stream after the HTTP response has already started (headers already sent). How does Next.js recover, and what are the failure modes you must account for?

**Deep Answer:**
Once the HTTP response stream begins, you **cannot change the status code**. Next.js ships an initial `200 OK` then streams the RSC Payload. If an async Server Component throws *after* the stream starts:
- The `error.js` boundary in the App Router **cannot** intercept it at the HTTP level — the browser has already received `200`.
- Instead, React serializes an error into the RSC Payload and triggers the nearest `error.tsx` boundary *in the client hydration phase*.
- The critical trap: **your monitoring/APM will see a `200` with an error inside the body.** Your alerting on 5xx will be blind to it.
- Production fix: Use `instrumentation.ts` + `onRequestError` hook (Next.js 15+) to capture server-side errors regardless of streaming state. Also wrap async RSC fetches in `try/catch` and return fallback UI rather than throwing.

```typescript
// instrumentation.ts
export function onRequestError(err, request, context) {
  // This fires even when status is already 200 mid-stream
  datadogRum.addError(err, { context });
}
```

---

### TRAP 2: Node.js EventEmitter Memory Leak with `async/await`

**Q:** You use `async/await` and believe you've avoided callback hell. You add an `EventEmitter` listener inside an `async` function called repeatedly. No explicit `removeListener` is called. What happens, and how does `async/await` mask it?

**Deep Answer:**
`async/await` does not help you with `EventEmitter` cleanup. Each call adds a new listener. Node's default `MaxListenersExceededWarning` fires at 11 listeners — but this is only a **warning**, not an error. The leak accumulates silently:

```js
// LEAKS - each call to initWebSocket adds a new 'data' listener
async function initWebSocket(emitter) {
  await setupConnection();
  emitter.on('data', handleData); // Never cleaned up
}
```

The trap: since `handleData` is an async function or captures a closure, the GC **cannot collect** the outer scope even after the logical operation completes. At scale (50k connections/hour) you exhaust memory without a single thrown exception.

**Fix pattern:**
```js
const controller = new AbortController();
emitter.on('data', handleData, { signal: controller.signal }); // Node 22+
// OR
const handler = handleData.bind(null, context);
emitter.once('close', () => emitter.off('data', handler));
```

Always pair `.on()` with a teardown path. Use `emitter.setMaxListeners(0)` only in tests, not production.

---

### TRAP 3: PostgreSQL N+1 in Prisma/TypeORM with `async/await`

**Q:** You have a GraphQL resolver that returns 100 `User` records, each with a nested `posts` relation. You `await` each `.findMany()` inside a loop. Why does this not just "run in parallel" and what's the actual DB impact?

**Deep Answer:**
`await` inside a `for...of` loop is **sequential**, not concurrent. You fire query #2 only after query #1 resolves. Even `Promise.all()` doesn't save you: 100 simultaneous DB connections hit your connection pool ceiling (default Prisma: 10). The remaining queries queue behind the pool.

Real impact:
- 100 `await posts.findMany({ where: { userId } })` = 100 round trips × ~2ms each = **200ms minimum** latency, plus pool contention.
- Under load, pool exhaustion causes `P2024: Timed out fetching a new connection`.

**Correct patterns:**
1. **DataLoader pattern** — batch & deduplicate: collect all `userIds`, fire one `WHERE userId IN (...)` query, distribute results.
2. **Prisma `include`** — single JOIN query.
3. **Raw SQL with `WITH` CTEs** — for complex cases.

```ts
// DataLoader approach
const userLoader = new DataLoader(async (userIds) => {
  const posts = await prisma.post.findMany({
    where: { userId: { in: userIds as string[] } }
  });
  // Group by userId and return in same order as input
  return userIds.map(id => posts.filter(p => p.userId === id));
});
```

---

### TRAP 4: JWT `exp` Claim vs. Server-Side Revocation

**Q:** You set a 15-minute access token expiry. A compromised token is used at minute 14. Your `exp` check passes. The user's session was revoked server-side 10 minutes ago. What went wrong in your architecture?

**Deep Answer:**
JWTs are **stateless by design** — the `exp` check is purely cryptographic. If you only validate the signature + `exp`, you have a 15-minute revocation gap. This is the core JWT trap that junior architects miss.

**Enterprise pattern — Sliding Session with Blacklist:**

```
Access Token: 15-min TTL (stateless, in-memory validation)
Refresh Token: 7-day TTL, stored as httpOnly cookie (stateful, server-side)
Blacklist: Redis SET of jti (JWT ID) values of revoked tokens
```

Flow:
1. On logout/password change/suspicious activity → add `jti` to Redis blacklist with TTL matching remaining token lifetime.
2. On **every** resource request: check `jti` against Redis blacklist **before** processing.
3. Redis lookup is O(1) — ~0.1ms round trip. Performance impact is negligible.

```js
// Middleware check
const payload = jwt.verify(token, SECRET);
const isBlacklisted = await redis.sismember('token:blacklist', payload.jti);
if (isBlacklisted) return res.status(401).json({ error: 'Token revoked' });
```

**Replay attack prevention:** Include `jti` (UUID v4) + `iat` + bind to client fingerprint (hashed IP + User-Agent). Rotate `jti` on each refresh cycle.

---

### TRAP 5: AWS Lambda Cold Start in VPC + RDS Connection Exhaustion

**Q:** You moved your Node.js Lambda to a VPC to access RDS PostgreSQL directly. P99 latency spiked from 50ms to 4 seconds during scale-out. Why, and what's the architectural fix?

**Deep Answer:**
Two compounding problems:

1. **VPC cold start penalty**: Lambda must provision an ENI (Elastic Network Interface) in your VPC. This adds **500ms–10s** to cold start, not just function init time.
2. **RDS connection exhaustion**: PostgreSQL `max_connections` defaults to ~100 for `db.t3.medium`. Each Lambda invocation creates a new connection. At 200 concurrent Lambda instances you exceed the connection limit and get `FATAL: sorry, too many clients already`.

`async/await` does NOT pool connections across Lambda invocations — each invocation is a separate process.

**Fix:**
- **RDS Proxy** (AWS managed connection pooler) — sits between Lambda and RDS, maintains persistent connections. Reduces connection overhead by 95%+. Supports IAM authentication.
- **Lambda SnapStart** (Java) or pre-warming + provisioned concurrency (Node) to eliminate cold starts for critical paths.
- Architecture: `Lambda → RDS Proxy → RDS` instead of `Lambda → RDS directly`.

```
// Lambda: reuse connection across warm invocations
let db;
export const handler = async (event) => {
  if (!db) db = await createConnection(); // Only on cold start
  return db.query(...);
};
```

---

### TRAP 6: Next.js `fetch` Cache Poisoning in App Router

**Q:** Two users make requests to the same Next.js App Router route. User A is admin, User B is a regular user. Both hit the same `fetch()` call inside a Server Component. User B sees admin data. What happened?

**Deep Answer:**
Next.js App Router **automatically deduplicates and caches `fetch()` calls** with the same URL across requests in the same render pass — and by default uses a request-level **Data Cache** that can persist across requests.

```ts
// This is DANGEROUS in a Server Component with user-specific data
const data = await fetch('https://api.internal/user-dashboard', {
  // No cache: 'no-store' = response may be cached and shared!
});
```

The cache key is the URL + options. If two users hit the same URL with the same options, the cached response from User A's request can serve User B.

**Fix:**
```ts
// For user-specific or sensitive data:
const data = await fetch('https://api.internal/user-dashboard', {
  cache: 'no-store',          // Never cache
  // OR
  next: { tags: ['user-data'], revalidate: 0 }
});
```

Or use `cookies()` / `headers()` inside the route — Next.js automatically opts out of static caching when these are used, but only at the **route level**, not the fetch level.

---

## SECTION 2 — REAL-WORLD SCENARIO DILEMMAS & ARCHITECTURE

---

### 2A: JWT Security — Enterprise-Grade Compromise & Replay Defense

**The full production pattern:**

```
┌────────────────────────────────────────────────────┐
│  ACCESS TOKEN (15 min, in Authorization header)    │
│  - Contains: sub, roles, jti (UUID v4), iat, exp   │
│  - Validated: signature + exp + Redis blacklist    │
│                                                    │
│  REFRESH TOKEN (7 days, httpOnly Secure SameSite   │
│  cookie, NOT in localStorage)                      │
│  - Stored: hashed in DB (bcrypt, not plaintext)    │
│  - Rotated: on every use (Refresh Token Rotation)  │
│  - Contains: family_id for reuse detection         │
└────────────────────────────────────────────────────┘
```

**Refresh Token Rotation (RTR) + Family Detection:**
```
1. Client sends refresh token RT₁
2. Server validates RT₁, issues RT₂ + new AT₂
3. Server invalidates RT₁ in DB
4. If RT₁ is used AGAIN → attacker has stolen it
   → Server detects reuse: RT₁ already invalidated
   → Invalidate ENTIRE token family (logout all sessions)
   → Alert security team
```

This means a stolen refresh token is detected the moment the legitimate user next refreshes — at worst, 7-day exposure window reduced to minutes in practice.

**Replay Attack Prevention:**
```
jti = crypto.randomUUID()          // Unique per token
Bind to: SHA256(IP + UserAgent)    // Stored in token as 'fgp' claim
On validation: recompute fingerprint, compare hashed
```

**Performance profile:**
- Redis blacklist: O(1), ~0.1ms
- DB refresh token lookup: indexed query, ~2ms
- Total overhead per request: ~2.1ms — negligible

**When token IS compromised:**
```
POST /auth/revoke
Body: { userId, revokeAll: true }

→ Add all active jti values for user to Redis blacklist
→ Delete all refresh tokens for user from DB
→ User is forced to re-authenticate
→ Issue security notification
```

---

### 2B: Distributed Caching — Production-Grade Strategy

**The five cache failure modes you must know:**

| Failure Mode | Definition | Solution |
|---|---|---|
| **Cache Stampede** | TTL expires → 1000s of requests hit DB simultaneously | Probabilistic early expiration + mutex lock |
| **Cache Avalanche** | Many keys expire at same time | Jitter on TTL: `TTL = base + random(0, 300s)` |
| **Cache Penetration** | Requests for non-existent keys bypass cache | Bloom filter OR cache null values with short TTL |
| **Hot Key** | Single key gets millions of reads/sec | Local in-process cache (LRU) + Redis replication |
| **Consistency Drift** | Cache and DB diverge | Write-through OR event-driven invalidation |

**Cache-aside vs. Write-through — the real trade-off:**

```
CACHE-ASIDE (Lazy Loading):
  Read: Check cache → miss → read DB → populate cache → return
  Write: Write DB only, cache becomes stale until next miss
  
  Pros: Only caches what's actually read, tolerant of cache failures
  Cons: Cache miss = 3 trips, stale data window = TTL duration
  Use when: Read-heavy, tolerate slight staleness (product catalog)

WRITE-THROUGH:
  Write: Write cache + DB synchronously (or near-sync)
  Read: Always cache hit (after first write)
  
  Pros: Zero stale data, low read latency
  Cons: Write latency doubles, cache fills with unread data
  Use when: Strong consistency required, financial data, user profiles

WRITE-BEHIND (Write-back):
  Write: Update cache → async DB write via queue
  
  Pros: Fastest write path
  Cons: Data loss if cache crashes before DB write
  Use when: High-throughput analytics, counters (views, likes)
```

**Cache Stampede prevention (production pattern):**
```js
async function getWithStampedeProtection(key, fetchFn, ttl) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  // Distributed lock (only one worker rebuilds)
  const lock = await redis.set(`lock:${key}`, 1, 'NX', 'EX', 10);
  if (!lock) {
    // Another process is rebuilding, wait briefly
    await sleep(50);
    return getWithStampedeProtection(key, fetchFn, ttl); // retry
  }
  
  try {
    const data = await fetchFn();
    // Jitter: prevent avalanche on neighboring keys
    await redis.setex(key, ttl + Math.floor(Math.random() * 300), JSON.stringify(data));
    return data;
  } finally {
    await redis.del(`lock:${key}`);
  }
}
```

**L1/L2 Cache Architecture (high-throughput):**
```
Request → Node.js in-process LRU (L1, 100ms TTL, ~1000 entries)
        → Redis cluster (L2, 60s TTL)
        → PostgreSQL read replica
        → PostgreSQL primary
```
L1 cache eliminates Redis round-trips for hot paths. For a viral product page: L1 handles 90% of reads, Redis handles 9%, DB handles 1%.

---

### 2C: Technology Trade-offs

#### AWS API Gateway vs. Nginx

| Dimension | AWS API Gateway | Nginx |
|---|---|---|
| **Use case** | Serverless/Lambda-backed APIs, public-facing with AWS auth | Container/VM-backed services, complex routing, high throughput |
| **Throughput ceiling** | ~10,000 RPS default (soft limit), adjustable | Millions of RPS, hardware-bound |
| **Latency** | +5–20ms per hop (region-dependent) | <1ms (same host), ~0.5ms (proxy) |
| **Auth/AuthZ** | Native Cognito, IAM, Lambda authorizers, API keys | Custom (nginx-plus or sidecar) |
| **Cost at scale** | $3.50/million requests + data transfer | Fixed infra cost, no per-request fee |
| **Operational overhead** | Zero (managed) | You own patching, HA, config |
| **WebSockets** | API Gateway WebSocket APIs (stateful) | Native, high performance |
| **Response size limit** | 10MB hard limit | Configurable, no practical limit |

**Decision rule:**
- Use **API Gateway** when: fully serverless, AWS ecosystem (Cognito/Lambda), rapid prototyping, traffic < 5M requests/day
- Use **Nginx** when: containers/VMs, > 5M req/day (cost), sub-millisecond routing, advanced L7 logic (A/B testing, canary), response transformation at scale

**Enterprise hybrid:** API Gateway as public ingress (handles rate limiting, auth, DDoS protection via WAF) → Nginx as internal service mesh proxy → application containers.

---

#### Next.js App Router vs. Standard Client-Side React (CRA/Vite SPA)

| Dimension | Next.js App Router | CRA/Vite SPA |
|---|---|---|
| **Rendering** | RSC: server-renders by default, streams HTML | CSR: ships empty HTML, renders in browser |
| **First Contentful Paint** | ~200–500ms (server renders HTML) | ~800–2000ms (JS parse + render) |
| **SEO** | Full HTML in response, crawlable immediately | Requires SSR workaround or prerendering |
| **JS bundle** | Server Components ship **zero JS** to client | Every component = JS bundle bytes |
| **Data fetching** | `async/await` in Server Components, no useEffect waterfalls | Client fetches = waterfall (component mounts → fetch → re-render) |
| **Secret handling** | API keys stay server-side natively | Must proxy through a backend to avoid exposure |
| **Complexity** | Higher: RSC mental model, `'use client'` boundary management | Lower: simpler mental model |
| **Ideal for** | Content-heavy, SEO-critical, large apps with authenticated data | Dashboards, internal tools, apps behind login (SEO irrelevant) |

**The architectural trap:** Using `'use client'` at too high a level (e.g., the layout) defeats the purpose of RSC — you ship the entire component tree as JS. The correct pattern is "push `'use client'` to the leaves" — only interactive leaf components get the directive.

---

## SECTION 3 — CRASH COURSE STUDY SOURCES (2-day plan)

**System Design:**
- [github.com/donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer) — Read: Cache section, Database section, Load Balancer. Skip everything else. (~45 min)
- [highscalability.com](http://highscalability.com/blog/category/example) — Pick 2 "Real World Architecture" posts (Netflix, Stripe). (~20 min each)

**Next.js / React:**
- [nextjs.org/docs/app/building-your-application/caching](https://nextjs.org/docs/app/building-your-application/caching) — The entire caching page. This is the #1 source of App Router interview questions. (~30 min)
- [nextjs.org/docs/app/building-your-application/rendering](https://nextjs.org/docs/app/building-your-application/rendering) — RSC vs Client Components. (~20 min)
- [**Theo's "RSC From Scratch"** on YouTube](https://www.youtube.com/watch?v=MaebEqhZR84) — 20 min, deeply clarifies the RSC mental model

**JWT & Auth:**
- [auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/) — ~10 min
- [hasura.io/blog/best-practices-of-using-jwt-with-graphql](https://hasura.io/blog/best-practices-of-using-jwt-with-graphql/) — Senior-level JWT patterns, ~15 min

**AWS:**
- [AWS Well-Architected Framework — Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html) — Skim the key principles (~15 min)
- [AWS RDS Proxy docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html) — Know *why* it exists, the connection pooling model (~10 min)

**Production Boilerplates:**
- [github.com/vercel/next.js/tree/canary/examples](https://github.com/vercel/next.js/tree/canary/examples) — `with-auth`, `with-redis` examples
- [github.com/t3-oss/create-t3-app](https://github.com/t3-oss/create-t3-app) — Production Next.js + Prisma + tRPC opinionated stack

**SQL:**
- [use-the-index-luke.com](https://use-the-index-luke.com/) — Chapter 1-3 only. The best senior SQL indexing resource. (~30 min)

---

## SECTION 4 — 20 MUST-DO JS & UI CODING ROUND REFRESHERS

| # | Problem | Core Trap |
|---|---|---|
| 1 | `debounce(fn, wait, { leading, trailing })` | Leading vs trailing edge, cancellation of inflight timers |
| 2 | `throttle(fn, interval)` with last-call guarantee | Time tracking without `setInterval`, argument forwarding |
| 3 | `Promise.all` polyfill | Counter pattern, early reject, empty array edge case |
| 4 | `Promise.allSettled` polyfill | Never rejects, always resolves with status objects |
| 5 | `Promise.race` polyfill | First settler wins, others still execute (not cancelled) |
| 6 | Async task queue with concurrency limit `asyncPool(limit, tasks)` | Sliding window of promises, refilling as slots open |
| 7 | LRU Cache (get/put in O(1)) | `Map` preserves insertion order in JS — use as doubly-linked list substitute |
| 8 | Deep clone without `JSON.parse/stringify` | Circular refs, `Date`, `RegExp`, `undefined`, Symbol |
| 9 | `EventEmitter` class (on/off/once/emit) | `once` wrapper must remove itself after first call |
| 10 | Virtualized list (windowing) | `scrollTop / itemHeight` for start index, padding trick for scroll height |
| 11 | `curry(fn)` with arbitrary arity | `fn.length` for arity detection, accumulator closure |
| 12 | `memoize(fn)` with WeakMap for objects | Primitive keys: Map. Object keys: WeakMap (prevents leak) |
| 13 | Flatten deeply nested array without `.flat(Infinity)` | Recursive vs iterative (stack-based to avoid call stack overflow at depth >10k) |
| 14 | Build `Observable` (subscribe/unsubscribe pattern) | Teardown function from subscriber, synchronous vs async emission |
| 15 | Recursive DOM tree traversal (BFS + DFS) | DFS: recursive or explicit stack. BFS: queue. Handle text nodes. |
| 16 | Multi-step form with validation state machine | Each step validates independently; `isValid` derived not stored |
| 17 | Infinite scroll with `IntersectionObserver` | Sentinel element pattern; cleanup observer on unmount |
| 18 | `groupBy(array, key)` — like lodash | `reduce` with object accumulator, handle missing keys |
| 19 | Custom `useDebounce` hook | `useEffect` cleanup cancels previous timer; return debounced value |
| 20 | `scheduler` — execute tasks by priority without starving | Min-heap or sorted queue, starvation prevention via aging |

---

## SECTION 5 — DSA PATTERN QUICK-REFRESH

---

### Sliding Window

**Q1 (Medium):** Longest substring with at most K distinct characters.
- **Time:** O(n) | **Space:** O(k)
- **Trick:** `Map` to track character counts. Expand right pointer, shrink left when `map.size > k`. Window never contracts unnecessarily — advance left only when constraint violated.

**Q2 (Hard):** Minimum window substring containing all chars of pattern T.
- **Time:** O(n + m) | **Space:** O(1) since charset is fixed
- **Trap:** Two pointer + frequency map. Track `formed` count (chars at required frequency). When `formed == required`, try contracting. Many candidates miss that you need *exact* frequency matching, not just presence.

**Sources:** [LeetCode #3](https://leetcode.com/problems/longest-substring-without-repeating-characters/), [#76](https://leetcode.com/problems/minimum-window-substring/)

---

### Two Pointer

**Q1 (Medium):** 3Sum — find all triplets summing to zero, no duplicates in output.
- **Time:** O(n²) | **Space:** O(1) excluding output
- **Trick:** Sort first. Fix `i`, use left/right pointers. Skip duplicates by advancing past equal values. The sort is what makes O(n²) possible; naive is O(n³).

**Q2 (Hard):** Trapping Rain Water
- **Time:** O(n) | **Space:** O(1)
- **Trap:** Two-pointer approach: maintain `leftMax` and `rightMax`. Water at position `i` = `min(leftMax, rightMax) - height[i]`. Process from the lower side. Naive two-pass with extra arrays is O(n) time but O(n) space — the trap is missing the O(1) space solution.

**Sources:** [LeetCode #15](https://leetcode.com/problems/3sum/), [#42](https://leetcode.com/problems/trapping-rain-water/)

---

### Hashmap / Frequency Counter

**Q1 (Medium):** Subarray Sum Equals K — count subarrays with sum exactly K.
- **Time:** O(n) | **Space:** O(n)
- **Trick:** Prefix sum + hashmap. `prefixSum[i] - prefixSum[j] = K` → look for `currentSum - K` in map. The off-by-one trap: initialize map with `{0: 1}` to handle subarrays starting at index 0.

**Q2 (Hard):** Longest Consecutive Sequence in O(n).
- **Time:** O(n) | **Space:** O(n)
- **Trap:** Use a `Set`. For each number, only start counting if `num - 1` is NOT in the set (i.e., it's a sequence start). This prevents O(n²) — the interview trap is implementing this with nested loops.

**Sources:** [LeetCode #560](https://leetcode.com/problems/subarray-sum-equals-k/), [#128](https://leetcode.com/problems/longest-consecutive-sequence/)

---

### Kadane's Algorithm

**Q1 (Medium):** Maximum Subarray Sum (classic Kadane's).
- **Time:** O(n) | **Space:** O(1)
- **Trick:** `currentMax = max(num, currentMax + num)`. Reset when adding current element is worse than starting fresh. `globalMax = max(globalMax, currentMax)`. The trap: all-negative arrays — answer is the single largest element, not 0.

**Q2 (Hard):** Maximum Sum Circular Subarray.
- **Time:** O(n) | **Space:** O(1)
- **Trap:** Two cases: (1) max subarray doesn't wrap = standard Kadane's. (2) max subarray wraps = `totalSum - minSubarraySum` (Kadane's for minimum). Answer = `max(case1, case2)`. Edge case: if all elements are negative, `case2 = 0` which is wrong — return `case1` only.

**Sources:** [LeetCode #53](https://leetcode.com/problems/maximum-subarray/), [#918](https://leetcode.com/problems/maximum-sum-circular-subarray/)

**General DSA resource for senior-level depth:** [neetcode.io/roadmap](https://neetcode.io/roadmap) — do the "Advanced" section only (you know the basics).

---

## QUICK-REFERENCE SUMMARY FOR DAY-OF

**JWT Rotation Flow in 30 seconds:**
> Issue AT (15min, contains `jti`) + RT (7-day, httpOnly cookie, hashed in DB). On every AT use: validate sig + exp + Redis blacklist check on `jti`. On RT use: rotate RT (invalidate old, issue new), detect reuse by checking if RT is already invalidated → if yes, nuke entire token family. On logout/compromise: blacklist all `jti`s + delete all RTs for user.

**Cache Consistency in 30 seconds:**
> Cache-aside for read-heavy with tolerable staleness (product catalog). Write-through for strong consistency requirements (user wallet balance). Write-behind for high-throughput counters where occasional data loss is acceptable (view counts). Always add TTL jitter (±10–20% of TTL) to prevent avalanche. Use Redis `SET key value NX EX 10` for stampede locks.

**Next.js App Router in 30 seconds:**
> Server Components render on server, ship zero JS, can be `async`. Client Components get `'use client'`, ship JS, can use hooks/events. RSC Payload is the binary wire format. `fetch()` is cached by default — add `cache: 'no-store'` for user-specific data. Errors mid-stream don't change the 200 status code — monitor via `onRequestError`.
