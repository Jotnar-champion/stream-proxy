Here is the complete guide as a copy-ready Markdown document:

---

````md
# Senior Full-Stack Interview Prep — 10-Day Sprint Guide
**Date: June 15, 2026**
**Target: Senior/Lead Full-Stack — Adobe CS2, Walmart, Uber, Flipkart, MakeMyTrip, Cars24, Nagarro**

---

## Table of Contents
1. [React / Next.js](#1-react--nextjs)
2. [Node.js / Express](#2-nodejs--express)
3. [AWS](#3-aws)
4. [CI/CD, Docker, GitLab CI, Jenkins](#4-cicd-docker-gitlab-ci--jenkins)
5. [DSA / LeetCode — 10-Day Sprint](#5-dsa--leetcode--10-day-sprint)
6. [System Design — HLD + LLD](#6-system-design--hld--lld)
7. [Database Design — PostgreSQL / SQL Server](#7-database-design--postgresql--sql-server)
8. [Resources Index](#resources-index)

---

## 1. React / Next.js

### Key Concepts to Master

**React Fiber & Scheduler**
- Fiber = unit of work; a linked-list tree (not a recursive call stack)
- Double-buffering: `current` tree (on screen) + `workInProgress` tree (being built)
- Render phase = pure, interruptible; Commit phase = effectful, synchronous, cannot be interrupted
- Lanes model (React 18): bitmask priority system replacing `expirationTime`; SyncLane > InputContinuousLane > DefaultLane > TransitionLane > IdleLane

**Concurrent Features (React 18)**
- `useTransition`: wraps state updates you control; marks them low-priority; returns `[isPending, startTransition]`
- `useDeferredValue`: for values you don't control (from props/third-party); deferred copy lags behind urgent renders
- Automatic batching: ALL updates batched inside `setTimeout`, Promises, native events — not just synthetic events
- `flushSync`: opt out of batching; forces synchronous flush; blocks main thread — use sparingly

**Next.js App Router Internals**
- RSC (React Server Components): execute on server, zero client JS, can't use hooks/browser APIs, direct DB access
- `"use client"` directive: marks component tree boundary; only these components hydrate
- Streaming: `loading.tsx` wraps route in Suspense; React streams HTML in chunks as data resolves
- Partial Prerendering (PPR): static shell from CDN + dynamic holes streamed at request time; defined by Suspense boundaries
- ISR: `fetch(url, { next: { revalidate: 60 } })` — stale-while-revalidate semantics
- Route handlers replace API routes in App Router; run in Node.js or Edge runtime
- Middleware: runs on Edge Runtime (V8 isolate, NOT Node.js); no `fs`, no native modules

**Optimization Beyond Memoization**
- `React.memo` only prevents re-render if props are referentially equal — useless without stable references
- Virtualization: `react-window` / `tanstack-virtual` for lists > 100 items
- Code splitting: `React.lazy` + `Suspense`; `dynamic()` in Next.js with `{ ssr: false }` for client-only libs
- Bundle analysis: `@next/bundle-analyzer`; look for duplicate packages, large moment/lodash imports
- Core Web Vitals: LCP (largest contentful paint < 2.5s), CLS (cumulative layout shift < 0.1), INP (interaction to next paint < 200ms — replaced FID in 2024)
- TTFB: reduce server compute, use edge caching, streaming, DB query optimization

**Memory Leaks in React**
- `useEffect` without cleanup: event listeners, subscriptions, WebSockets, timers
- Closures capturing stale references in async operations
- `AbortController` for cancelling in-flight `fetch` on unmount
- Zustand/Redux selectors creating new object references on every call

---

### 15 Hard Interview Questions

**Q1. React 17 vs React 18 batching: where exactly does React 17 NOT batch, and what does React 18 change?**
> React 17: batching only in synthetic event handlers (React's own event system). Inside `setTimeout`, `Promise.then`, `addEventListener` callbacks — each `setState` triggers a separate re-render. React 18 with `createRoot`: automatic batching everywhere. Opt out with `flushSync(() => setState(...))`. Interview trap: people say "React always batched" — wrong; it's new in React 18 universally.

**Q2. Explain the stale closure problem in this code and provide two different fixes:**
```js
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => console.log(count), 1000);
    return () => clearInterval(id);
  }, []);
}
```
> **Problem**: `count` in the interval closure captures `0` at mount time (empty dep array). The interval always logs `0`. Fix 1: add `count` to deps array (creates new interval on every count change — acceptable for small intervals). Fix 2: use a ref to track latest value without causing re-render:
```js
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }, [count]);
useEffect(() => {
  const id = setInterval(() => console.log(countRef.current), 1000);
  return () => clearInterval(id);
}, []);
```
Fix 3 (if only incrementing): `setCount(c => c + 1)` functional updater doesn't need current count in closure.

**Q3. What is the React Fiber "double buffer" pattern? Why is it critical for concurrent rendering?**
> React maintains two trees simultaneously: `current` (what's visible on screen) and `workInProgress` (being built). During the render phase, React builds the WIP tree. On commit, it atomically swaps the pointers. This enables interruption: if a higher-priority update arrives mid-render, React can abandon the incomplete WIP tree and restart with the new priority without corrupting the visible UI. Without double buffering, a partially-rendered tree could be committed to the screen in a broken state.

**Q4. Explain hydration mismatches in Next.js. What exactly does React do when it detects one, and what are all the causes?**
> When React hydrates, it compares server-rendered HTML against the client's first render output. If they differ, React logs a warning and **discards the server HTML entirely**, doing a full client-side render — completely defeating SSR benefits. Causes: `new Date()` (different timezone/time between server and client), `Math.random()`, `typeof window` guards not applied correctly, browser extensions injecting DOM nodes, CSS-in-JS generating different class names, locale-dependent formatting. Fix: `suppressHydrationWarning` for intentional mismatches; `useEffect` for client-only rendering; ensure server and client render identical output on first pass.

**Q5. `useTransition` vs `useDeferredValue` — what's the fundamental difference and when do you use each?**
> `useTransition`: you OWN the state update. Wrap the update: `startTransition(() => setState(val))`. React marks this update as non-urgent and can interrupt it for urgent ones (typing, clicking). `useDeferredValue`: you DON'T own the state update (it comes from a prop, context, or library). You create a deferred copy: `const deferredQuery = useDeferredValue(query)`. The deferred copy lags behind while urgent renders complete. Use `useDeferredValue` for: rendering a filtered list from a prop you can't wrap in `startTransition`.

**Q6. You have 10,000 rows rendered with `React.memo`. Performance is still poor. Walk through your complete diagnosis.**
> Step 1: React DevTools Profiler — identify which components re-render and WHY (highlight "why did this render"). Step 2: check if parent state triggers context re-render (context value is a new object reference on every render). Step 3: verify `React.memo` comparison — if props include arrays/objects created inline, memo always fails. Step 4: check if the component itself calls expensive computations inside render without `useMemo`. Fixes in order: virtualize with `react-window` (render only visible rows), stabilize context with `useMemo`/split context, `useMemo` for expensive derived values, verify memo comparison with custom comparator. Bottom line: `React.memo` without stable prop references is useless.

**Q7. What is "tearing" in React concurrent mode and how does `useSyncExternalStore` solve it?**
> Tearing: during a concurrent render (which is interruptible), React may render different parts of the component tree at different moments in time. If an external store (Redux, Zustand) updates between those renders, some components see the old value and some see the new — the UI is in an inconsistent ("torn") state. `useSyncExternalStore` (React 18) solves this by: subscribing to the external store and forcing a synchronous re-render if the store changes during a concurrent render, ensuring all components see the same snapshot. All major state managers (Redux, Zustand) use this internally in React 18.

**Q8. Explain Next.js ISR: a user hits a page 500ms after `revalidate: 60` expires. What do they see? What happens next?**
> They see the **stale cached page immediately** (served from CDN/cache). Next.js simultaneously triggers a background revalidation request to regenerate the page. The user who triggered revalidation sees old data. The NEXT user after revalidation completes sees fresh data. This is stale-while-revalidate (SWR) semantics — prioritize availability/speed over freshness. Key interview point: if revalidation fails, the stale page continues to be served (no error shown to users).

**Q9. Why can a React Server Component not use `useState`, and what exactly happens at the serialization boundary?**
> RSCs run on the server once and produce a serializable React element tree (not HTML — a JSON-like wire format). They have no lifecycle, no client-side re-renders. `useState` requires client-side JS to persist state between renders — RSCs have no client runtime. At the serialization boundary (where RSC passes props to a Client Component): only serializable values can cross — strings, numbers, arrays, plain objects, Dates, JSX. Functions, class instances, closures, React context, and non-serializable values CANNOT cross. This is why "you can't pass a callback from Server to Client Component as a prop."

**Q10. Describe a memory leak in a React component using a WebSocket subscription. Write the buggy code and the fix.**
> Buggy:
```js
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (e) => setMessages(prev => [...prev, e.data]);
  // No cleanup!
}, [url]);
```
> On unmount (route change, conditional render), the WebSocket stays open and fires events, calling `setMessages` on an unmounted component — memory leak + React warning. Fix:
```js
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (e) => setMessages(prev => [...prev, e.data]);
  return () => {
    ws.close();
    ws.onmessage = null;
  };
}, [url]);
```

**Q11. What is Partial Prerendering (PPR) in Next.js and how is it architecturally different from ISR?**
> ISR: regenerates the ENTIRE page on a schedule; the whole page is either fresh or stale. PPR: a SINGLE route has a static shell (generated at build time, served from CDN instantly) and dynamic holes (Suspense-boundary-wrapped RSCs that stream in at request time). Different parts of the same page can be static or dynamic independently. PPR is more granular — your header, navigation, and footer are static; your user-specific feed streams in dynamically. Defined by wrapping dynamic components in `<Suspense>` boundaries.

**Q12. What causes high CLS (Cumulative Layout Shift) and walk through fixing it in a Next.js app?**
> Causes: images without explicit `width`/`height` (browser doesn't reserve space), web fonts causing FOUT (flash of unstyled text — elements shift when font loads), dynamic content injected above the fold (banners, cookie notices), embeds without fixed dimensions. Diagnosis: Chrome DevTools → Performance → Layout Shift regions; `web-vitals` library, Vercel Speed Insights. Fixes: always use Next.js `<Image>` (auto-reserves space), `font-display: optional` or preload critical fonts, CSS `aspect-ratio` for containers whose content loads async, `min-height` on skeleton placeholders that match final content dimensions.

**Q13. You have a large form (50+ fields) where every keystroke re-renders the entire form. What's your architecture?**
> Root cause: controlled inputs with top-level state cause full tree re-renders. Solutions in order of invasiveness: 1) `React Hook Form` — uncontrolled approach, no state per keystroke, only re-renders on validation/submit. 2) Split form into separate components with isolated state — changes in Section A don't re-render Section B. 3) `useReducer` at top + `React.memo` on each field with stable `dispatch` reference. 4) State management (Jotai atom per field) for maximum isolation. Never use a single `useState` object for 50+ fields.

**Q14. `useCallback` — when does it actually PREVENT re-renders and when does it FAIL to?**
> It PREVENTS re-renders when: 1) the child is wrapped in `React.memo`, AND 2) the `useCallback` deps haven't changed (so the function reference is stable). It FAILS when: 1) child is NOT wrapped in `React.memo` (doesn't matter — child re-renders anyway), 2) deps change every render (defeats purpose), 3) child receives OTHER unstable props alongside the stable callback. The most common mistake: adding `useCallback` everywhere "for performance" without `React.memo` on consumers — pure overhead with zero benefit.

**Q15. Explain Next.js Middleware vs API Route handlers. When does Middleware make it worse, not better?**
> Middleware runs on Edge Runtime (Cloudflare Workers-like V8 isolate) BEFORE the request hits cache or origin. Zero Node.js APIs. Use for: auth token validation, geo-routing, A/B testing, header injection. Middleware makes it WORSE when: you need to query a database (no native DB drivers in Edge), you need complex business logic (better in an API route), you need to read the request body of a POST (Middleware can't easily do this), or when adding latency to every request for logic that only applies to a few routes (use route-level middleware instead).

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | React Fiber Architecture — Andrew Clark | https://github.com/acdlite/react-fiber-architecture | Original design doc for Fiber; primary source on scheduler, lanes, double-buffering |
| 2 | React 18 Working Group (GitHub Discussions) | https://github.com/reactwg/react-18/discussions | RFC discussions from React core team; explains concurrent features from first principles |
| 3 | Next.js App Router Architecture (Vercel Blog) | https://nextjs.org/blog/next-13-4 | Official explanation of RSC, streaming, and PPR from the engineers who built it |

---

## 2. Node.js / Express

### Key Concepts to Master

**Event Loop Phases (Deep)**
- Order: timers → pending callbacks → idle/prepare → poll → check → close callbacks
- `process.nextTick`: NOT a phase; runs after current operation, before any I/O — can starve the event loop if recursive
- `Promise.then` (microtasks): runs after nextTick queue, before returning to event loop phases
- `setImmediate`: check phase; inside an I/O callback it ALWAYS runs before `setTimeout(fn, 0)`
- `setTimeout(fn, 0)`: timers phase; outside I/O, order vs `setImmediate` is non-deterministic (OS scheduler)
- Priority: `process.nextTick` > `Promise microtasks` > `setImmediate` > `setTimeout(fn,0)`

**Concurrency Models**
- Single-threaded event loop: I/O delegated to libuv thread pool (default 4 threads, set via `UV_THREADPOOL_SIZE`)
- **Worker Threads**: true parallel JS; shared memory via `SharedArrayBuffer` + `Atomics`; use for CPU-bound JS
- **Cluster**: `cluster.fork()` spawns multiple Node processes sharing a port; OS distributes connections; use for multi-core I/O scaling
- **Child Processes**: `spawn` (stream I/O), `exec` (buffer output), `fork` (IPC-enabled Node process); use for non-JS executables or isolated processes
- Decision tree: I/O-bound → event loop handles it. CPU-bound JS → Worker Threads. CPU-bound non-JS → `spawn`. Multiple cores for I/O → Cluster.

**Streams & Backpressure**
- `writable.write()` returns `false` when internal buffer exceeds `highWaterMark` — consumer can't keep up
- Correct backpressure: pause readable on `false`, resume on `drain` event
- `pipeline()` (Node 10+): automatically handles backpressure, cleanup, and error propagation — always prefer over manual `.pipe()`
- `objectMode: true`: streams work with arbitrary JS objects; `highWaterMark` = count of objects (default 16), not bytes
- Transform streams: both Readable and Writable; use for processing pipelines (compression, encryption, parsing)

**Express Production Patterns**
- Middleware order: `helmet()` → `cors()` → `express.json()` → `express.urlencoded()` → routes → 404 handler → error handler
- Error handler: 4-parameter signature `(err, req, res, next)` — must be registered LAST
- Async error propagation in Express 4: rejected promises in route handlers DO NOT automatically call error handler — must wrap or use Express 5
- `trust proxy`: required when behind nginx/ALB/ELB for correct `req.ip`, `req.protocol`, `X-Forwarded-*` headers
- `express.Router()`: mount per domain/feature; keeps route files isolated

**Security (OWASP)**
- `helmet()`: sets 11 security headers (HSTS, X-Frame-Options, CSP, X-Content-Type-Options)
- Input validation at boundaries with `zod` or `joi` — never trust `req.body` shape
- Rate limiting: `express-rate-limit` + Redis store for distributed environments
- Prototype pollution: `{"__proto__": {"isAdmin": true}}` — use `Object.create(null)` for untrusted data, `secure-json-parse`, or `zod` schema validation
- Never log `req.body` in production without sanitization (PII, secrets)

---

### 15 Hard Interview Questions

**Q1. Exact priority order: `process.nextTick` vs `Promise.then` vs `setImmediate` vs `setTimeout(fn,0)`. Explain with output prediction.**
```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
```
> Output: `sync` → `nextTick` → `promise` → (then either `timeout` or `immediate`, non-deterministic outside I/O). Inside an I/O callback: `nextTick` → `promise` → `immediate` → `timeout` (setImmediate always before setTimeout inside I/O).

**Q2. The 5th `fs.readFile` call hangs momentarily. Why? How do you fix it for high-concurrency production?**
> libuv thread pool defaults to 4 threads. `fs.readFile`, `dns.lookup`, `crypto` operations all share this pool. The 5th concurrent operation queues. Fix: `UV_THREADPOOL_SIZE=64` (environment variable, max 1024). Better: use `dns.resolve()` instead of `dns.lookup()` (uses real async DNS API, not thread pool). For file I/O at scale: use streams instead of buffering entire file, or offload to a dedicated microservice.

**Q3. This backpressure implementation causes OOM in production. Explain why and write the correct version.**
```js
readable.on('data', chunk => writable.write(chunk));
```
> `writable.write()` returning `false` is ignored — readable keeps emitting, chunks buffer in memory. Fix:
```js
readable.on('data', chunk => {
  if (!writable.write(chunk)) readable.pause();
});
writable.on('drain', () => readable.resume());
readable.on('end', () => writable.end());
```
> Better — use `pipeline`:
```js
const { pipeline } = require('stream/promises');
await pipeline(readable, writable); // handles backpressure + cleanup + errors
```

**Q4. Express 4 async middleware bug: why doesn't this error reach the error handler?**
```js
app.get('/users', async (req, res, next) => {
  const users = await db.getUsers(); // throws
  res.json(users);
});
```
> Express 4 route handlers are not promise-aware. An unhandled rejection in async function doesn't call `next(err)` — it becomes an unhandled promise rejection. Express never sees it. Fix:
```js
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users', asyncHandler(async (req, res) => {
  const users = await db.getUsers();
  res.json(users);
}));
```
> Express 5 (released) handles this natively — async route handlers' rejections auto-call `next(err)`.

**Q5. Explain `SharedArrayBuffer` + `Atomics` in Worker Threads. What happens without `Atomics`?**
> Worker Threads can share memory via `SharedArrayBuffer` — both threads read/write the same memory address. Without `Atomics`, operations aren't atomic: thread A reads value (5), thread B reads value (5), both add 1, both write 6 — you lose an increment. `Atomics.add(view, index, 1)` performs the read-modify-write atomically at the hardware level. `Atomics.wait()` / `Atomics.notify()` implement futex-like synchronization for coordination between threads.

**Q6. How do you perform a memory heap snapshot on a production Node.js service without downtime?**
> 1) Expose a protected admin endpoint: `app.get('/admin/heapdump', authMiddleware, (req, res) => { v8.writeHeapSnapshot(); res.json({ ok: true }); })`. 2) Use `--heap-prof` flag: `node --heap-prof app.js` — generates `.heapprofile` on exit. 3) `clinic heapprofiler` (NearForm Node Clinic) for detailed allocation tracking. 4) `--inspect=127.0.0.1:9229` on a staging instance with production-equivalent load → Chrome DevTools Memory tab → take 3 heap snapshots → compare for retained objects. 5) Monitor `process.memoryUsage().heapUsed` via custom CloudWatch metric — graph over time to confirm linear growth (leak pattern).

**Q7. What is prototype pollution, give a real attack vector in an Express app, and list three mitigations.**
> Attack: POST body `{"__proto__": {"isAdmin": true}}` + `Object.assign({}, req.body)` → `({}).isAdmin === true` globally for all objects. Express app then grants admin access to every subsequent request. Mitigations: 1) Use `zod`/`joi` schema validation — both reject `__proto__` and `constructor` keys. 2) `const obj = Object.create(null)` for untrusted object accumulation — no prototype chain. 3) `npm install secure-json-parse` — replaces `JSON.parse` with pollution-safe version. 4) `Object.freeze(Object.prototype)` — prevents modification (may break some libraries). 5) `express.json({ reviver: ... })` to strip dangerous keys.

**Q8. Explain `AsyncLocalStorage` and write a request tracing implementation.**
> `AsyncLocalStorage` provides request-scoped storage that automatically propagates through all async operations in a request's execution tree without passing context explicitly.
```js
const { AsyncLocalStorage } = require('async_hooks');
const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  const store = { requestId: crypto.randomUUID(), userId: req.user?.id };
  requestContext.run(store, next);
});

// Anywhere in the codebase — no need to pass requestId as parameter:
function queryDatabase(sql) {
  const ctx = requestContext.getStore();
  logger.info({ requestId: ctx?.requestId, sql }, 'DB query');
  return db.query(sql);
}
```

**Q9. You have a Node.js service with random latency spikes every few minutes. No errors. What's your investigation?**
> First hypothesis: GC pause (major GC / mark-and-sweep). Verify: `node --trace-gc app.js` — logs GC events with duration. If 50-200ms pauses align with spikes → heap is growing too large. Second hypothesis: libuv thread pool saturation. Third: `setInterval` or cron job blocking event loop. Tool: `node --prof` → `node --prof-process` for V8 profiler flame graph. `clinic doctor` (NearForm) for automated diagnosis. Also check: DNS resolution (`dns.lookup` blocks thread pool), `bcrypt` calls in request path (CPU-bound crypto).

**Q10. Design graceful shutdown for an Express server that gets Kubernetes SIGTERM signals.**
```js
const server = app.listen(3000);
let isShuttingDown = false;

process.on('SIGTERM', () => {
  isShuttingDown = true;
  server.close(async () => {  // stop accepting new connections
    await db.pool.end();      // close DB connections
    await redisClient.quit(); // close Redis
    logger.info('Graceful shutdown complete');
    process.exit(0);
  });

  // Force exit if graceful shutdown takes too long (K8s SIGKILL = 30s)
  setTimeout(() => process.exit(1), 25000);
});

// Reject new requests during shutdown
app.use((req, res, next) => {
  if (isShuttingDown) return res.status(503).json({ error: 'Service unavailable' });
  next();
});
```

**Q11. `cluster` vs PM2 cluster for zero-downtime deploys. What does PM2 `reload` do that `restart` doesn't?**
> `pm2 restart`: kills ALL workers simultaneously, starts new ones — brief downtime. `pm2 reload`: rolling restart — sends `SIGINT` to ONE worker, waits for it to drain and exit (or timeout), starts a replacement, then moves to the next. Zero downtime because at least one worker is always alive. To implement manually with `cluster` module: iterate `cluster.workers`, send SIGINT one at a time, wait for `exit` event, `cluster.fork()` replacement. PM2 adds automatic restart on crash, log management, and monitoring on top.

**Q12. Explain Node.js `objectMode` streams with a database cursor → CSV export example.**
```js
const { Transform } = require('stream');
const { pipeline } = require('stream/promises');

const dbCursor = db.query('SELECT * FROM orders').stream(); // objectMode readable (rows)

const csvTransform = new Transform({
  objectMode: true, // input = objects
  writableObjectMode: true,
  readableObjectMode: false, // output = strings/Buffers
  transform(row, encoding, callback) {
    this.push(`${row.id},${row.total},${row.created_at}\n`);
    callback();
  }
});

await pipeline(dbCursor, csvTransform, res); // res is a writable stream
```
> Without `objectMode`: Transform would receive Buffer chunks, not row objects — can't access `row.id`.

**Q13. What is the footgun with `process.nextTick` in recursive patterns?**
```js
function recursiveNextTick(n) {
  if (n === 0) return;
  process.nextTick(() => recursiveNextTick(n - 1));
}
recursiveNextTick(1000000);
```
> The nextTick queue is fully drained before the event loop advances. This recursive pattern starves ALL I/O — HTTP requests, timers, file reads are blocked until the million nextTick calls complete. This is why Node.js docs say "prefer `setImmediate` over `process.nextTick` in recursive patterns." `setImmediate` yields to the event loop between iterations; `nextTick` does not.

**Q14. How does Node.js handle DNS resolution, and why does it matter for `http.request` performance?**
> `http.request` uses `dns.lookup()` by default, which goes through the libuv thread pool (not the OS async DNS API). With 4 default threads and multiple outgoing requests to different hosts, DNS lookups queue behind each other and behind file I/O. Symptoms: slow first requests, latency spikes under concurrent outgoing calls. Fixes: 1) Set `UV_THREADPOOL_SIZE=16+`. 2) Use `dns.resolve()` (actual async API) via custom `lookup` option in `http.agent`. 3) Use a connection pool (http-agent with `keepAlive: true`) — reuses sockets, bypasses DNS for repeat connections.

**Q15. What is `unhandledRejection` vs `uncaughtException` in Node.js? How should you handle each in production?**
> `uncaughtException`: a thrown error that propagated without a `try/catch`. Process is in undefined state — log, flush metrics, `process.exit(1)`. NEVER try to resume. `unhandledRejection`: a Promise that was rejected with no `.catch()` handler. In Node 15+, this crashes the process by default (previously just a warning). Handle:
```js
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception');
  metrics.flush(() => process.exit(1));
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error({ reason }, 'Unhandled rejection');
  // In production: treat same as uncaughtException — crash and let PM2/K8s restart
  process.exit(1);
});
```
> Key: always use `async/await` + `try/catch` or `.catch()` — don't rely on global handlers as first defense.

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | Node.js AsyncLocalStorage + async_hooks | https://nodejs.org/api/async_context.html | Primary source for request context propagation; directly tested in senior interviews |
| 2 | Node Clinic by NearForm | https://clinicjs.org/ | Production-grade profiling suite used by Node core contributors; flame graphs + memory analysis |
| 3 | Node.js Design Patterns (Casciaro & Mammino) | https://www.nodejsdesignpatterns.com/ | Authoritative book on streams, patterns, and advanced Node architecture; referenced in official docs |

---

## 3. AWS

### Key Concepts to Master

**Lambda**
- Cold start phases: download zip/image → start Firecracker micro-VM → initialize runtime → run init code (outside handler)
- Warm container: execution environment reused across invocations; global scope persists (reuse DB connections, SDK clients)
- Provisioned Concurrency: pre-warmed environments; eliminates cold start; billed per hour
- Reserved Concurrency: hard cap on function concurrency; `0` = function disabled (useful kill switch)
- Concurrency limit: 1000 per account per region by default; request increase via Support
- Lambda Layers: shared dependencies; up to 5 layers; max 250MB unzipped total

**S3**
- Strong consistency since Dec 2020 (all operations: PUT, GET, LIST, DELETE)
- Storage classes: Standard → Standard-IA (30-day minimum) → One-Zone-IA → Glacier Instant → Glacier Flexible → Deep Archive
- Multipart upload: required > 5GB, recommended > 100MB; parts min 5MB except last
- Pre-signed URLs: delegate time-limited access without exposing credentials; `expires_in` parameter
- Object Lock: WORM (Write Once Read Many) compliance; Governance vs Compliance mode
- S3 Transfer Acceleration: routes uploads through CloudFront edge network

**SQS**
- Standard: at-least-once, best-effort ordering, nearly unlimited TPS
- FIFO: exactly-once processing, strict ordering within message group, 300 TPS (3000 with batching)
- Visibility timeout: time message is hidden after receipt — set > Lambda max timeout + buffer
- Long polling: `WaitTimeSeconds=20`; reduces empty responses, saves cost
- DLQ: messages exceeding `maxReceiveCount` sent here; monitor DLQ depth as alarm

**SNS**
- Fan-out: one publish → multiple SQS queues / Lambda functions / HTTP endpoints / email
- Message filtering: subscriber filter policy — `{"eventType": ["ORDER_CREATED"]}` — reduces Lambda invocations
- FIFO SNS: works with FIFO SQS for ordered fan-out; same group ID semantics

**CloudWatch vs X-Ray**
- CloudWatch: metrics, logs, alarms, dashboards — aggregate view
- X-Ray: distributed tracing — follows ONE request through Lambda → API Gateway → DynamoDB → external calls; shows per-segment timing; service map
- X-Ray catches what CloudWatch misses: which specific downstream call is slow, cold start vs handler execution breakdown, N+1 DB query patterns in trace waterfall

**EC2**
- Instance families: C (compute), R (memory), I (storage NVMe), P/G (GPU), T (burstable)
- Placement Groups: Cluster (same AZ, low latency HPC), Spread (max 7 instances/AZ, fault isolation), Partition (Hadoop/Kafka, racks as partitions)
- Spot Instances: 2-minute termination notice; use interruption handlers; never for stateful single-instance workloads

**API Gateway**
- REST API vs HTTP API: HTTP API is 70% cheaper, lower latency, fewer features (no request transformation, no usage plans)
- WebSocket API: persistent connections; `connectionId` for targeting specific clients
- Throttling: 10,000 RPS account limit, 5,000 burst; per-stage/route overrides
- Lambda Authorizer: custom JWT/token validation; caches result by `authorizationToken` for TTL seconds

**VPC**
- Public subnet: has 0.0.0.0/0 route to Internet Gateway
- Private subnet: no direct internet; NAT Gateway for outbound (charges per GB processed)
- Security Groups: stateful, instance-level (return traffic auto-allowed)
- NACLs: stateless, subnet-level, evaluated in number order (lower number = higher priority)
- VPC Endpoints: Gateway (free, S3/DynamoDB), Interface (priced, most other services) — traffic stays in AWS network, bypasses NAT

---

### 15 Hard Interview Questions

**Q1. Lambda cold start: what are ALL the mitigation strategies and what are the tradeoffs of each?**
> 1) **Provisioned Concurrency**: eliminates cold start completely; costs money per hour regardless of invocations. 2) **Keep function small**: smaller zip = faster download + initialization; tree-shake, use only required SDK v3 clients. 3) **Minimize global init code**: lazy-load heavy dependencies inside handler (trade-off: first invocation after warm-up has init cost). 4) **Scheduled warm-up ping**: invoke every 5 min — anti-pattern, unreliable, wastes cost; avoid. 5) **HTTP API over REST API**: lighter runtime. 6) **SnapStart** (Java/Python): snapshots initialized state; sub-100ms cold start. 7) **arm64 architecture**: 20% cheaper, often faster init than x86.

**Q2. SQS visibility timeout = 30s. Lambda function takes 45s. Describe exactly what happens and the fix.**
> At T+0: Lambda receives message, message hidden. At T+30: visibility timeout expires, message becomes visible again. A NEW Lambda invocation picks up the same message — concurrent duplicate processing. Original Lambda is still running. Both complete successfully → double processing. Fix: set visibility timeout = (Lambda timeout × 6) per AWS recommendation. Or use `ChangeMessageVisibility` API call within Lambda to extend visibility timeout dynamically as work progresses. Always: design consumers to be idempotent — duplicate processing should be safe.

**Q3. SNS→Lambda direct vs SNS→SQS→Lambda fan-out. When does the direct pattern fail at scale?**
> SNS→Lambda direct: SNS invokes Lambda synchronously. If Lambda is throttled (concurrency limit hit), SNS retries 2-3 times with exponential backoff then **drops the message**. No persistence. Under traffic spikes: lost messages. SNS→SQS→Lambda: SQS absorbs the spike (unlimited retention), Lambda polls at its own pace via event source mapping. Concurrency controlled by `batchSize` and `maxConcurrency`. Messages persist for up to 14 days. Use direct only for low-volume, non-critical notifications. Use SQS buffer for anything requiring guaranteed delivery.

**Q4. S3 event notifications vs EventBridge for S3: when do you choose each?**
> S3 direct notifications: ~seconds latency, limited event types (ObjectCreated, ObjectRemoved, Replication), single destination (SQS or SNS or Lambda). EventBridge: rich content-based filtering on S3 metadata (prefix, suffix, size), fan-out to 5+ targets simultaneously, cross-account routing, event archiving + replay capability (critical for debugging), ~1s additional latency. Choose EventBridge when: multiple downstream consumers, complex filtering (e.g., only `.jpg` files > 1MB in `uploads/` prefix), cross-account, replay needed. Choose direct for: single consumer, latency-sensitive pipeline, simple triggers.

**Q5. You set Lambda reserved concurrency to 0 by mistake. What happens and how do you detect it?**
> Every invocation returns `TooManyRequestsException` (HTTP 429) immediately. The function is effectively disabled. API Gateway callers receive 429 or 502 depending on integration. Detection: CloudWatch Lambda → `Throttles` metric spikes to 100%; `Errors` from API Gateway. If it's an async trigger (SQS, SNS), messages pile up in SQS DLQ or SNS retries exhaust. Fix: set reserved concurrency to desired value (or remove limit entirely by setting to `null`). Lesson: document this as a valid emergency kill switch but gate it with IAM permissions.

**Q6. Explain the NAT Gateway cost footgun and how to fix it with VPC Endpoints.**
> NAT Gateway charges: $0.045/hour + $0.045/GB processed. A Lambda in a private VPC calling S3/DynamoDB routes through NAT Gateway by default — every API call is billable data transfer. At scale (millions of Lambda invocations calling S3): hundreds of dollars/month in NAT costs. Fix: 1) S3 Gateway VPC Endpoint: free, add route to route table, traffic stays in AWS network. 2) DynamoDB Gateway VPC Endpoint: also free. 3) For other services (SSM, Secrets Manager, STS): Interface VPC Endpoints (~$7.30/month/AZ — compare against NAT costs). Most teams save 60-80% on data transfer costs after adding S3/DynamoDB endpoints.

**Q7. API Gateway 29-second timeout. You have a slow report generation endpoint. Design the async solution.**
> Never make the user wait 29s. Pattern: 1) POST `/reports` → Lambda queues job to SQS → returns `202 Accepted` + `{ jobId: "uuid" }`. 2) SQS → Report Worker Lambda (can run up to 15 min). 3) Client polls GET `/reports/{jobId}` → returns `{ status: "processing" }` until done, then `{ status: "complete", url: "s3-presigned-url" }`. Or: Step Functions callback pattern — worker calls `SendTaskSuccess` when done, Step Functions notifies API. Or: WebSocket API Gateway — push completion event to client. Rule: any operation > 5s should be async with status polling or push notification.

**Q8. Explain how CloudWatch Alarms work with Auto Scaling and what "cooldown period" prevents.**
> CloudWatch Alarm: metric breaches threshold → transitions to ALARM state → triggers SNS/action (scale out policy). Auto Scaling group receives scale-out action → launches new instances. Cooldown period (default 300s): after a scaling activity, Auto Scaling ignores further alarms for this duration. Prevents: thrashing — new instances haven't started serving traffic yet, metric still high, triggers another scale-out → over-provisioning. Instance warmup: separate setting — new instance's metrics excluded from aggregate until warmup period passes. Target tracking policies manage cooldown automatically; step scaling requires manual tuning.

**Q9. Describe the SQS FIFO `MessageGroupId` pattern for parallel ordered processing.**
> FIFO queues provide ordering within a message group, parallelism across groups. Example: ride-sharing app where driver location updates for RIDE-123 must be processed in sequence, but RIDE-456 and RIDE-789 can process in parallel. Set `MessageGroupId = rideId`. SQS FIFO ensures all messages for RIDE-123 go to a single Lambda invocation at a time (per group serialization) while different rides process concurrently. Without group IDs (all messages in one group): fully sequential processing — FIFO becomes a bottleneck.

**Q10. X-Ray vs CloudWatch Logs Insights: what does X-Ray catch that you cannot see in logs?**
> Logs: what your code explicitly logs — application-level events. X-Ray: distributed trace of one request's journey across ALL services — Lambda function → API Gateway → two DynamoDB calls → one external HTTP call. Shows: exact duration of EACH segment, which specific DynamoDB query took 800ms (logs only show total handler time), cold start duration isolated from handler execution, downstream service dependencies mapped visually, error correlation across service boundaries. Logs Insights: powerful queries but only within one log group at a time. X-Ray shows the full request waterfall across log groups.

**Q11. Lambda event source mapping for SQS: explain `batchSize`, `batchWindow`, and `maxConcurrency`.**
> `batchSize`: max messages per Lambda invocation (1-10,000 for standard, 1-10 for FIFO). `batchWindow`: wait up to N seconds to fill a batch before invoking Lambda (reduces Lambda invocations, increases latency). `maxConcurrency`: max simultaneous Lambda invocations from this SQS trigger (new in 2023). Without `maxConcurrency`: SQS scales Lambda to 1000 concurrent invocations → may overwhelm downstream DB. Set `maxConcurrency` to protect downstream services. If any message in a batch fails: entire batch returned to queue by default. Use `reportBatchItemFailures` to return only failed messages.

**Q12. Describe how EC2 Spot Instance interruptions work and design a fault-tolerant batch processing architecture.**
> Spot Instance: 2-minute interruption notice via EC2 metadata endpoint (poll `http://169.254.169.254/latest/meta-data/spot/termination-time`) and EventBridge event. Architecture: SQS queue of work items → Spot Instance fleet (Auto Scaling with `SpotAllocationStrategy: capacity-optimized`) → workers poll SQS. On interruption notice: worker stops polling, finishes current item (< 2 min work units), deletes message from SQS, gracefully exits. In-progress items with long processing: checkpoint progress to S3/DynamoDB, extend SQS visibility timeout. Use `mixed instances policy` with on-demand base capacity for critical path.

**Q13. VPC Security Group vs NACL: when does the stateful vs stateless difference cause a real bug?**
> Real bug scenario: you allow inbound on port 443 in NACL. User makes HTTPS request. Server responds on an ephemeral port (1024-65535) — the return traffic. Security Group: stateful — return traffic automatically allowed. NACL: stateless — outbound ephemeral port range (1024-65535) is NOT automatically allowed. Response packets are dropped. Fix: add NACL outbound rule allowing 1024-65535. Common mistake: developers set NACL inbound correctly but forget outbound ephemeral ports. Most teams use Security Groups exclusively and set NACLs to allow-all (layered defense via SGs only).

**Q14. Describe Lambda Destinations and how they differ from DLQs.**
> DLQ: only for failed asynchronous invocations after all retries exhausted. Lambda Destinations: configurable for BOTH success and failure of async invocations. Can route to SQS, SNS, Lambda, or EventBridge for success (e.g., trigger next step on success) AND failure (e.g., alert team on failure). DLQ receives only the original event. Destinations receive: original event + function response/error + request context + invocation record. Destinations are more powerful: you get success routing, richer failure context, and EventBridge routing for complex workflows. DLQ is simpler for basic failure capture.

**Q15. You're building a multi-region active-active architecture. What AWS services have cross-region complexity and how do you handle them?**
> DynamoDB Global Tables: automatic multi-master replication; last-write-wins conflict resolution; eventual consistency across regions. S3 Cross-Region Replication: async, not real-time; use S3 Multi-Region Access Points for intelligent routing. Route 53 latency-based or geolocation routing + health checks for failover. API Gateway: deploy to each region, Route 53 routes to nearest. Lambda: deploy same function in each region, test for regional env var differences. The hardest part: database write conflicts (DynamoDB Global Tables last-writer-wins may not suit all apps), data residency compliance (some data can't leave region), and cost (CRR, Global Tables replication all cost extra).

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | AWS Well-Architected Framework | https://aws.amazon.com/architecture/well-architected/ | Primary source for all 5 pillars; every question about "best practice" maps here |
| 2 | AWS re:Invent Talks (YouTube) | https://www.youtube.com/@AWSEventsChannel | Engineers who built Lambda/SQS/S3 explaining design decisions; search "re:Invent SVS" for serverless |
| 3 | AWS Architecture Blog | https://aws.amazon.com/blogs/architecture/ | Real production architectures with diagrams and cost analysis from AWS solutions architects |

---

## 4. CI/CD, Docker, GitLab CI & Jenkins

### Key Concepts to Master

**Docker Internals**
- Image = read-only stack of layers; container = image + thin writable layer
- Each `RUN`, `COPY`, `ADD` = one layer; layer cached if content + all preceding layers unchanged
- Build context: entire directory sent to Docker daemon — `.dockerignore` is critical for performance
- Multi-stage builds: `FROM node:20 AS builder` ... `FROM gcr.io/distroless/nodejs20` ... `COPY --from=builder /app/dist ./dist`
- Distroless images (Google): no shell, no package manager, no OS utilities — minimal attack surface; `gcr.io/distroless/nodejs20-debian12`
- BuildKit: parallel layer building, `--mount=type=cache` (persist npm/pip cache across builds), `--mount=type=secret` (build-time secrets not in layers)
- Security: run as non-root (`USER node`), read-only filesystem (`--read-only`), no secrets in ENV or layers, scan with `trivy` or `grype`

**GitLab CI/CD**
- `.gitlab-ci.yml`: defines `stages`, `jobs`, `rules`, `needs`, `cache`, `artifacts`, `environment`
- `stages`: sequential execution blocks; all jobs in stage N complete before stage N+1 starts
- `needs`: DAG — job starts when its specific deps finish, regardless of stage completion
- `rules`: conditional job execution — `if`, `changes`, `exists`; evaluated in order (first match wins); preferred over deprecated `only/except`
- `cache`: persisted between pipeline runs; stored in runner cache; keyed by `key:` expression
- `artifacts`: files passed between jobs in same pipeline; uploaded to GitLab, downloaded by downstream jobs
- Environments: deployment tracking with approval gates (`when: manual`) for production
- GitLab Agent for Kubernetes: pull-based GitOps; cluster pulls from GitLab, not push-based deployment

**Jenkins**
- Declarative pipeline (`pipeline {}` block): structured, validated by Jenkins before running, limited to declarative syntax
- Scripted pipeline (`node {}` block): full Groovy, maximum flexibility, harder to maintain
- Shared Libraries: reusable Groovy code in separate Git repo; `@Library('lib@v1.0') _`; `vars/` for global functions
- Multibranch Pipeline: auto-discovers branches/PRs with Jenkinsfiles; creates pipeline per branch
- Still chosen when: on-prem with no internet access, complex Groovy orchestration needs, existing investment in plugins

**Deployment Strategies**
- **Blue-Green**: two identical environments; traffic switch at load balancer/DNS; instant rollback (point back); requires 2x resources
- **Canary**: gradual traffic shift (1% → 5% → 25% → 100%); monitor error rates + latency at each step; automated rollback trigger
- **Rolling**: replace instances one-at-a-time; less resource-intensive; can't instantly rollback all instances
- **Feature Flags**: deploy code without activating it; decouple deploy from release; LaunchDarkly, Unleash, AWS AppConfig
- **GitOps**: Git is single source of truth for infrastructure state; ArgoCD/Flux watches Git, applies changes to cluster; drift detection

---

### 15 Hard Interview Questions

**Q1. Explain why this Dockerfile is inefficient and rewrite it correctly for a production Node.js app.**
```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
> Problems: 1) `COPY . .` before `npm install` — any source code change busts the npm install cache. 2) `npm install` includes devDependencies. 3) Single stage — dev tools in production image. 4) Running as root. Fixed:
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --include=dev
COPY . .
RUN npm run build

# Stage 2: Production
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
# Distroless runs as nonroot by default (UID 65532)
EXPOSE 3000
CMD ["dist/server.js"]
```

**Q2. What is Docker BuildKit `--mount=type=secret` and why is it critical for private npm registries?**
> Without it: `RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && npm ci` — the `.npmrc` file with the token is baked into the image layer permanently. Anyone with image access can extract the token. With BuildKit secret mount:
```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```bash
docker build --secret id=npmrc,src=.npmrc .
```
> The `.npmrc` exists only during that `RUN` step — never in any layer. `docker history` shows nothing. Token is safe.

**Q3. Blue-green vs canary: for a database schema migration that adds a NOT NULL column, which strategy works and which breaks?**
> Blue-green breaks: v2 (new schema) is deployed and traffic switches. If rollback needed, v1 code can't read the NOT NULL column that v2 added (column doesn't exist in v1's expected schema). Canary breaks similarly. Correct approach: **Expand-Contract (Parallel Change) pattern**: 1) Expand: add column as nullable, deploy v1 (doesn't use it). 2) Backfill: populate the column. 3) Add constraint: add NOT NULL with DEFAULT. 4) Deploy v2: starts writing to the column. 5) Contract: remove old code paths. Both strategies are safe ONLY when migrations are backward-compatible.

**Q4. Production is down. Walk me through a rollback in under 5 minutes using GitLab CI.**
> Pre-requisite: this must be designed ahead of time. 1) GitLab Environments page → click "Rollback" on previous deployment → re-runs previous deployment job (30s if images are cached). 2) Feature flag: toggle off in LaunchDarkly — instant, zero deploy. 3) Blue-green: redirect load balancer to blue environment (< 30s DNS or ALB target group swap). 4) Helm: `helm rollback my-app 1` → previous chart revision (< 1min on K8s). 5) Manual GitLab pipeline trigger: create a `rollback` job with `when: manual` that runs `helm rollback` or swaps ALB target groups. Key lesson: rollback is a button press, practiced monthly in staging.

**Q5. Explain GitLab CI DAG with `needs:` and draw an example showing why it's faster.**
> Stage-based (without `needs`): all of `test-unit`, `test-integration`, `test-e2e` in Stage 2 must complete before `build-docker` in Stage 3 starts.

> DAG with `needs`: `build-docker` only needs `test-unit` to pass. It starts immediately after `test-unit` finishes, while `test-integration` and `test-e2e` are still running. `deploy-staging` needs both `build-docker` AND `test-e2e`. Time saving: if `test-unit` = 2min, `test-e2e` = 10min, `build-docker` = 3min — DAG pipeline: 13min total vs 15min stage-based. Real pipelines with 20+ jobs: 30-50% time savings.

**Q6. How do you handle secrets in Docker Compose (development) vs Kubernetes (production) without changing application code?**
> Application code always reads from files or environment variables — abstract the source. Development (Docker Compose): `secrets:` block reads from local files, mounted at `/run/secrets/db_password`. Production (K8s): External Secrets Operator or Vault Agent Sidecar fetches from Vault/AWS Secrets Manager, creates K8s Secret, mounted as file at same path. Or: K8s Secret (base64 encoded, encrypted at rest if KMS configured) as env var or volume mount. Application code: `fs.readFileSync('/run/secrets/db_password')` — identical in both environments. Never: hardcode, commit, or build secrets into images.

**Q7. Jenkins shared library: how do you version it and prevent a library update from breaking 50 pipelines?**
> Library referenced with tag/branch: `@Library('pipeline-lib@v2.1.0') _`. All 50 pipelines reference specific version tags. Library changes: create new tag → pipelines only upgrade when they explicitly change the `@Library` version. For breaking changes: semantic versioning, changelog in library repo, Slack notification to teams. Safe rollout: 1) Create `v3.0.0` tag. 2) Update 1 non-critical pipeline. 3) Monitor. 4) Gradual migration. CI for the library itself: run `jenkins-cli` to validate Jenkinsfiles against the library using the `lintJenkinsfile` command.

**Q8. Your Docker container fails in CI but works locally. Systematic debug process.**
> 1) **Pin base image**: local might have `node:20.11.1`, CI pulls `node:20` (latest) = different version. Fix: `node:20.11.1-alpine`. 2) **Platform mismatch**: M1/M2 Mac builds `linux/arm64`; CI is `linux/amd64`. Add `--platform linux/amd64` to build. 3) **Build context difference**: `.dockerignore` might exclude files locally that CI includes (or vice versa). Print what's in context. 4) **env vars**: CI has different values; print non-secret env vars in failing step. 5) **File permissions**: CI may run as different UID; check `RUN ls -la`. 6) **Reproduce CI locally**: `docker run -e CI=true --platform linux/amd64 your-image sh`.

**Q9. What does `gitlab-ci.yml` `cache` not restore, and how do you debug it?**
> Cache not restored when: 1) Different runner instance (not same machine) and cache is filesystem-based (not S3). 2) `key:` expression changed (branch name, lockfile hash changed). 3) Cache expired (default 30 days). 4) First run on a new branch (no cache for this key yet). 5) Cache upload failed on previous run (job killed). Debug: add `echo "Cache key: $CI_CACHE_KEY"` to job, check GitLab job log for "Checking cache for..." message. Fix for distributed runners: configure `[runners.cache]` in `config.toml` with S3 backend — shared cache across all runner instances.

**Q10. Explain zero-downtime deployment for a Node.js app with active WebSocket connections on Kubernetes.**
> WebSockets are long-lived — normal rolling deploy drops them. Strategy: 1) Client-side: always implement reconnection with exponential backoff + jitter (this is non-negotiable regardless of strategy). 2) K8s Pod `preStop` lifecycle hook: `sleep 15` before `SIGTERM` — allows load balancer to drain new connections away. 3) `terminationGracePeriodSeconds: 60`: give existing connections 60s to complete naturally. 4) App-level drain: on SIGTERM, stop accepting new WS upgrades, allow existing connections to idle-timeout. 5) Decouple state: use Redis pub/sub for WS state → any pod handles any connection → reconnection is transparent to the user.

**Q11. Implement a CI security scanning step that blocks deployment if critical vulnerabilities are found.**
```yaml
# .gitlab-ci.yml
container-scanning:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL,HIGH
        --ignore-unfixed
        --format template
        --template "@/contrib/gitlab.tpl"
        -o gl-container-scanning-report.json
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
  allow_failure: false  # blocks deployment on findings
```
> `--exit-code 1`: Trivy exits with code 1 on findings → GitLab marks job as failed → blocks downstream deploy job. `--ignore-unfixed`: skip vulns with no available fix (reduces noise). Add `--skip-dirs` for known false positives.

**Q12. What's the difference between `COPY` and `ADD` in Dockerfile? When is `ADD` dangerous?**
> `COPY`: copies files/dirs from build context to image. Explicit, predictable. `ADD`: does everything `COPY` does PLUS: auto-extracts tar archives, fetches remote URLs. Dangerous: `ADD https://example.com/file.tar.gz /app/` — fetches at build time, can't be cached properly, URL could change or be hijacked, introduces supply chain risk. Best practice: always use `COPY` unless you explicitly need tar extraction. For remote files: `curl` in `RUN` with checksum verification. Docker official docs recommend preferring `COPY` over `ADD`.

**Q13. GitLab CI `rules` vs `only/except`: concrete example showing where `only/except` fails and `rules` works.**
> Requirement: run job on merge request OR on `main` branch push, but only if `src/` files changed.
```yaml
# only/except — cannot express this:
job:
  only: [merge_requests, main]
  # No way to combine with 'changes' condition in only/except

# rules — expressive:
job:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes: [src/**/*]
    - if: '$CI_COMMIT_BRANCH == "main"'
      changes: [src/**/*]
    - when: never  # skip all other cases
```
> `only/except` can't combine branch/event conditions with `changes` (file changes) in an AND relationship. `rules` can express complex boolean logic. This is why `only/except` was deprecated.

**Q14. How do you implement GitOps with GitLab and ArgoCD for a Kubernetes deployment?**
> 1) Separate `infra` repo from `app` repo. 2) App CI pipeline builds Docker image, updates `infra` repo's Helm `values.yaml` with new image tag (git commit via CI job). 3) ArgoCD watches `infra` repo → detects git diff → automatically applies to K8s cluster (or requires manual sync for production). 4) ArgoCD shows drift: if someone manually changes K8s resource, ArgoCD shows it as "out of sync" with Git. 5) Rollback = `git revert` the image tag commit → ArgoCD applies previous state. Benefits: audit trail (Git blame shows who changed what when), PR reviews for infra changes, disaster recovery (recreate entire cluster from Git state).

**Q15. What is a multi-arch Docker build and why is it required for teams with M-series Macs deploying to AWS?**
> M1/M2/M3 Macs are `arm64`. AWS EC2 default instances are `x86_64` (amd64). A Docker image built on Mac is `arm64` — it will fail or run via slow emulation on `amd64` EC2. Fix: multi-arch build with Docker Buildx:
```bash
docker buildx create --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t myrepo/myapp:latest .
```
> Creates a manifest list — `docker pull` on any architecture gets the right image. CI should always build for `linux/amd64` explicitly. Bonus: AWS Graviton (arm64) instances are 20% cheaper — worth building multi-arch to support both.

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | OWASP Docker Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html | Authoritative security hardening reference; maps to every production Docker question |
| 2 | GitLab CI Pipeline Efficiency Docs | https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html | GitLab's own guide on DAG, caching, artifacts optimization — primary source |
| 3 | Google SRE Book — Release Engineering | https://sre.google/sre-book/release-engineering/ | Google-scale CI/CD philosophy; the mental model senior interviewers expect you to have |

---

## 5. DSA / LeetCode — 10-Day Sprint

### Is Blind 75 Enough?

| Company | Verdict | Notes |
|---|---|---|
| Adobe (CS2) | Blind 75 + | Medium-hard focus; problem-solving process matters more than hard grinding |
| Walmart Labs | Grind 169 | Medium-heavy; some hard DP/graph; data structures + design expected |
| Flipkart / MakeMyTrip | Blind 75 ✓ | LeetCode Medium focus; articulate time/space complexity clearly |
| Uber | NeetCode 150 | Graph, advanced trees, system design-adjacent coding; check Uber company tag |
| Cars24 / Nagarro | Blind 75 ✓ | Emphasis on clean code and approach clarity |

**Recommendation**: Master Blind 75 first (solve each in < 20 min cold). Then fill gaps with NeetCode 150 additions, prioritizing graphs and DP. Use LeetCode company tags for Uber and Walmart for targeted prep.

---

### 10-Day DSA Sprint Plan

| Day | Topic | Problems | Goal |
|---|---|---|---|
| 1 | Arrays + Two Pointers | Two Sum, Three Sum, Container With Most Water, Trapping Rain Water | Learn two-pointer template |
| 2 | Sliding Window | Longest Substring Without Repeating Chars, Minimum Window Substring, Sliding Window Maximum | Learn SW template — shrink/expand |
| 3 | Trees — DFS/BFS | Binary Tree Level Order, Max Depth, Validate BST, LCA of Binary Tree, Diameter | Recursive + iterative for both |
| 4 | Graphs | Number of Islands, Clone Graph, Course Schedule (topological), Pacific Atlantic | BFS/DFS templates + topo sort |
| 5 | Dynamic Programming I | Climbing Stairs, House Robber, Coin Change, Longest Increasing Subsequence | Identify subproblem → recurrence |
| 6 | Dynamic Programming II | 0/1 Knapsack, Edit Distance, LCS, Word Break, Palindrome Partitioning | Bottom-up table traversal direction |
| 7 | Heaps + Binary Search | Top K Frequent, Merge K Sorted Lists, Median from Data Stream, Search in Rotated Array | Heap push/pop complexity; BS template |
| 8 | Stack + Monotonic Stack | Valid Parentheses, Min Stack, Daily Temperatures, Largest Rectangle in Histogram | Monotonic stack pattern |
| 9 | Linked Lists + Tries | Reverse LL, Detect Cycle, Merge K Sorted, Implement Trie, Word Search II | Fast/slow pointers; trie ops |
| 10 | Mixed Hard + Company Tags | 5 problems you struggled with, timed; Uber/Walmart company tag | Simulate real interview timing |

**Time budget per day**: 2-3 problems per hour; spend 25 min trying before looking at hint. Always explain your approach out loud before coding.

---

### Key Interview Patterns (Pattern → Problem Map)

| Pattern | Key Problems |
|---|---|
| Two Pointers | Three Sum, Container With Most Water, Remove Duplicates |
| Sliding Window | Min Window Substring, Longest Substring K Distinct |
| Fast/Slow Pointer | Linked List Cycle, Find Middle, Happy Number |
| BFS (shortest path) | Word Ladder, Shortest Path in Binary Matrix |
| DFS + Backtracking | Permutations, Subsets, N-Queens, Word Search |
| Topological Sort | Course Schedule, Alien Dictionary |
| Union-Find | Number of Connected Components, Redundant Connection |
| Monotonic Stack | Next Greater Element, Largest Rectangle |
| Binary Search | Search in Rotated Array, Find Minimum, Koko Eating Bananas |
| 0/1 Knapsack DP | Subset Sum, Partition Equal Subset |
| Unbounded Knapsack DP | Coin Change, Rod Cutting |
| Interval Scheduling | Merge Intervals, Meeting Rooms II |

---

### 10 Hard Interview Questions

**Q1. Sliding window maximum in O(n). Explain the monotonic deque approach.**
> Use a deque storing indices in decreasing order of values. For index `i`: pop from back while `nums[deque.back()] <= nums[i]` (current is better candidate for future windows). Pop from front if `deque.front() <= i - k` (outside window). Front is always the maximum.
```js
function maxSlidingWindow(nums, k) {
  const dq = [], result = [];
  for (let i = 0; i < nums.length; i++) {
    while (dq.length && nums[dq[dq.length-1]] <= nums[i]) dq.pop();
    dq.push(i);
    if (dq[0] <= i - k) dq.shift();
    if (i >= k - 1) result.push(nums[dq[0]]);
  }
  return result;
}
```
> O(n) — each element added/removed at most once.

**Q2. Detect a cycle in directed vs undirected graph. Both approaches.**
> Undirected: DFS with parent tracking. If visited node ≠ parent → cycle.
> Directed: DFS with `visited` set (globally) AND `inStack` set (current DFS path). If DFS reaches an `inStack` node → cycle. Can also use Kahn's algorithm (BFS topological sort): if not all nodes in result → cycle exists.

**Q3. 0/1 Knapsack vs Unbounded Knapsack: why does table traversal direction differ?**
> 0/1 (each item used at most once): traverse capacity `j` from `W` down to `weight[i]`. Going backwards prevents using item `i` twice in same row update. Unbounded (unlimited use): traverse `j` from `weight[i]` up to `W`. Going forwards allows item `i` to be selected again in same row.

**Q4. Implement LRU Cache with O(1) get and put.**
```js
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map(); // key → node
    this.head = { key: 0, val: 0, prev: null, next: null }; // dummy
    this.tail = { key: 0, val: 0, prev: null, next: null }; // dummy
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }
  get(key) {
    if (!this.map.has(key)) return -1;
    const node = this.map.get(key);
    this._remove(node); this._addToFront(node);
    return node.val;
  }
  put(key, val) {
    if (this.map.has(key)) this._remove(this.map.get(key));
    const node = { key, val };
    this._addToFront(node);
    this.map.set(key, node);
    if (this.map.size > this.capacity) {
      const lru = this.tail.prev;
      this._remove(lru); this.map.delete(lru.key);
    }
  }
  _remove(node) { node.prev.next = node.next; node.next.prev = node.prev; }
  _addToFront(node) { node.next = this.head.next; node.prev = this.head; this.head.next.prev = node; this.head.next = node; }
}
```

**Q5. Given a list of intervals, merge overlapping ones. What's the key insight?**
> Sort by start time. Iterate: if current interval overlaps with last merged (`curr.start <= last.end`), extend last's end to `max(last.end, curr.end)`. Otherwise push new interval.
```js
function merge(intervals) {
  intervals.sort((a, b) => a[0] - b[0]);
  const result = [intervals[0]];
  for (const [s, e] of intervals.slice(1)) {
    const last = result[result.length - 1];
    if (s <= last[1]) last[1] = Math.max(last[1], e);
    else result.push([s, e]);
  }
  return result;
}
```

**Q6. What's the difference between Dijkstra's and Bellman-Ford? When does Dijkstra fail?**
> Dijkstra: greedy, min-heap, O((V+E)logV). Doesn't work with negative edges (greedy assumption broken — a longer path could become shorter via negative edge). Bellman-Ford: DP, O(VE), handles negative edges, detects negative cycles (if V-th relaxation improves a path → negative cycle). Use Bellman-Ford for: graphs with negative weights, detecting negative cycles (currency arbitrage).

**Q7. Word Ladder: why is it a BFS problem and not DFS/DP?**
> We need the SHORTEST transformation sequence — BFS guarantees shortest path in an unweighted graph (each transformation = 1 step = equal edge weight). DFS would find A path, not necessarily the shortest. DP doesn't naturally apply to shortest path in graph with equal weights. Key insight: treat each word as a node; edge exists if words differ by exactly one character. BFS from `beginWord` to `endWord`.

**Q8. Top K frequent elements: heap approach vs bucket sort approach. When is each better?**
> Heap (max-heap): O(n log k) time, O(n) space. Count frequencies → push all to min-heap of size k → heap contains top-k. Bucket sort: O(n) time, O(n) space. Count frequencies → create buckets indexed by frequency (max = n) → fill buckets → read from highest bucket backwards. Bucket sort wins for large n when k is small. Heap wins when you need EXACTLY k elements from a sorted order perspective.

**Q9. Explain Union-Find with path compression and union by rank. What's the amortized complexity?**
> `find(x)`: traverse to root, compress path (make every node point to root directly). `union(x, y)`: find both roots, attach smaller rank tree under larger. Amortized: O(α(n)) ≈ O(1) for all practical purposes (inverse Ackermann function). Without compression: O(log n). Without rank: O(n) worst case (linear chain).

**Q10. When should you choose a Segment Tree vs Fenwick Tree (BIT)?**
> Both: O(log n) point update, O(log n) range query. Fenwick (BIT): simpler to implement (10 lines), only for prefix-reducible operations (sum, XOR, product). Segment Tree: more complex but supports range updates with lazy propagation, range min/max (non-invertible operations), more query types. Choose Fenwick for: range sum with point updates (simpler, less memory). Choose Segment Tree for: range min/max, range updates, or when you need multiple query types on the same structure.

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | NeetCode.io | https://neetcode.io/ | Best structured coverage of Blind 75 + 150 with pattern-based video explanations |
| 2 | Competitive Programmer's Handbook (free PDF) | https://cses.fi/book/book.pdf | Covers advanced algorithms (Segment Trees, Fenwick, advanced graphs) tested at senior levels |
| 3 | LeetCode Company Tags | https://leetcode.com/company/ | Filter by Uber, Walmart, Adobe — see actual reported questions from interviews |

---

## 6. System Design — HLD + LLD

### Key Concepts to Master (Non-Negotiable)

**CAP Theorem**
- You cannot have all three: Consistency, Availability, Partition Tolerance
- Network partitions are unavoidable in distributed systems → real choice is CP vs AP
- CP: consistent but may be unavailable during partition (HBase, ZooKeeper, Etcd)
- AP: available but may serve stale data (Cassandra, CouchDB, DynamoDB with eventual consistency)
- PACELC extension: even without partition, tradeoff between Latency (L) and Consistency (C)

**Consistent Hashing**
- Hash ring: nodes placed by hash(nodeId); keys assigned to next clockwise node
- Adding/removing nodes: only `k/n` keys remapped (vs 100% in modulo hashing)
- Virtual nodes: each physical node has multiple positions on ring for even distribution
- Used by: DynamoDB, Cassandra, Redis Cluster, Memcached, CDN edge routing

**Database Sharding**
- Range-based: shard by ID range (1M-2M, 2M-3M) — simple range queries, hot shard risk
- Hash-based: `hash(userId) % numShards` — even distribution, range queries suffer
- Directory-based: lookup table (shard metadata service) — flexible, SPOF risk
- Key challenge: cross-shard JOINs don't exist; choose shard key to keep related data together
- Hotspot: user_id as shard key → celebrity user causes hot shard; solution: add random salt

**Caching Patterns**
- Cache-aside (lazy loading): app checks cache on read; miss → fetch DB → populate cache
- Write-through: write to cache AND DB synchronously; no stale reads; more write latency
- Write-behind (write-back): write to cache; async flush to DB; risk of data loss on cache failure
- Cache invalidation: TTL (simplest, accepts staleness), event-driven invalidation on write, cache-busting with versioned keys

**Redis Patterns**
- String: simple key-value, counters (`INCR`), rate limiting
- Sorted Set: leaderboard (`ZADD`, `ZRANGE`), time-series events
- Hash: user session storage, partial updates
- Pub/Sub: real-time notifications, presence, WebSocket fan-out
- Streams: append-only log; consumer groups; use over Pub/Sub when message durability needed

---

### HLD: Dropbox / Google Drive

**Functional Requirements**: upload/download, sync across devices, share files/folders, versioning, conflict resolution.

**Key Design Decisions**:
- **Chunking**: split files into 4MB chunks; upload only changed chunks on sync (delta sync). Content-defined chunking (variable size based on content hash) for better deduplication across similar files.
- **Deduplication**: SHA-256 each chunk; if hash exists in block store → reference it (don't re-upload). Saves storage; protects against re-uploading identical content.
- **Metadata vs Block Storage**: Metadata service (PostgreSQL — ACID, relational structure for file trees, permissions) separate from block store (S3 — cheap, durable, scalable).
- **Sync protocol**: client maintains local sync DB. On file change: detect delta → hash changed chunks → upload missing chunks → update metadata service → notify other devices via WebSocket/long-poll.
- **Conflict**: last-write-wins by default; or create "conflict copy" (Google Drive approach) when simultaneous edits detected via vector clock comparison.

**Architecture**:
```
Client
  ↓ HTTPS
Load Balancer
  ↓
API Servers (stateless)
  ├── Metadata Service → PostgreSQL (file metadata, chunk lists, sharing)
  │                   → Redis (session cache, recently accessed metadata)
  ├── Upload Service → S3 (chunk storage) + SQS (async processing)
  └── Notification Service → WebSocket gateway → Redis Pub/Sub (fan-out)

Background:
  Block Processor (deduplication, compression, virus scan)
  Thumbnail Generator (Lambda triggered by S3 event)
```

**Follow-up Questions Interviewers Ask**:
- "Large file upload fails at 80% — what happens?" → Resumable upload: client tracks uploaded chunks, retries from last checkpoint (multipart upload resumption)
- "Two users edit the same Google Doc simultaneously?" → Operational Transformation (OT) or CRDT; this is Google Docs, not Drive — real-time co-editing is a separate harder problem
- "How do you scale to 1 billion files?" → Shard metadata DB by `userId`; use S3 directly for block storage; add CDN for download acceleration
- "How do you handle deleted files?" → Soft delete (mark as deleted, 30-day recovery bin); hard delete runs as scheduled cleanup job; S3 lifecycle policy removes blocks with zero references

---

### HLD: Google Search

**Key Components**:
- **Web Crawler**: distributed, polite (robots.txt, crawl-delay, politeness per domain), deduplication via URL fingerprint (SimHash), priority queue (PageRank-based crawl frequency — high-value pages re-crawled more often)
- **Inverted Index**: term → [docId, TF, positions]. Built by MapReduce/distributed pipeline. Stored in distributed file system. TF-IDF + PageRank for relevance.
- **Query Processing**: tokenize → lowercase → stopword removal → stemming → inverted index lookup → boolean/vector space ranking → return top K
- **PageRank**: iterative algorithm; score = sum of referring pages' scores / their out-degree. Computed offline (batch job), stored per document.
- **Serving tier**: index tiered — hot index (recent crawls, memory), warm index (SSD), cold index (HDD). Latency target < 200ms P99.

**Follow-up Questions**:
- "How do you handle near-duplicate content?" → SimHash — compute hash for document; similar documents have low Hamming distance; deduplicate by canonicalizing to one URL
- "How do you index a new page in < 1 hour?" → Separate "freshness index" for recent crawls; merged with main index; Google's real-time indexing pipeline
- "How would you design spell correction?" → N-gram language model + edit distance; "did you mean?" based on high-frequency query corrections from query logs
- "How do you serve personalized results?" → User query history + click-through data; learning-to-rank models; safe to mention this exists without needing to fully design it

---

### HLD: WhatsApp / Messenger

**Key Requirements**: 1:1 messaging, group messaging (< 256 members), delivery status, media, online presence, end-to-end encryption.

**Key Design Decisions**:
- **WebSocket connections**: persistent TCP per client to gateway server; enables server push without polling. Connection state stored in Redis (`userId → gatewayServerId`).
- **Message flow**: Sender → Gateway A → Kafka (message topic) → Consumer → Gateway B → Recipient's WebSocket. If recipient offline → Kafka → Push Notification Service → FCM/APNS.
- **Message storage**: Cassandra. Schema: `(conversationId, messageTimestamp, messageId, senderId, content)`. Partition by `conversationId`, cluster by `messageTimestamp DESC`. Reads: `SELECT ... WHERE conversationId = ? LIMIT 50`.
- **Delivery receipts**: Recipient ACKs receipt → updates message status in Cassandra → notify sender's gateway → single-tick to double-tick.
- **Group messages**: Fan-out at write time for small groups (< 256): copy message to each member's inbox (fast reads). Fan-out at read time for large groups (> 256): one copy, compute membership on read.
- **Presence**: Heartbeat every 5s → `SETEX userId:presence "online" 10` in Redis. Subscribe to presence via WebSocket.
- **Media**: Upload to S3 via pre-signed URL; send S3 URL as message content; thumbnail generation via Lambda.
- **E2E Encryption**: Signal Protocol; keys stored on device only; server stores encrypted blobs only.

**Follow-up Questions**:
- "How do you guarantee message ordering?" → Sequence number per conversation; client-side reordering; server assigns monotonically increasing seq# at gateway
- "Group with 1000 members sends a message — fan-out latency?" → Async fan-out via Kafka; notifications batched; use `MessageGroupId = userId` for parallel per-user delivery
- "Cassandra goes down — do you lose messages?" → Replication factor = 3; quorum writes (`QUORUM` consistency) — 2/3 nodes must ACK; AP model means brief unavailability, not data loss

---

### HLD: Reddit

**Key Design**:
- **Vote counting**: Redis counters (`INCR vote:postId:up`) for real-time score; async sync to PostgreSQL every 30s via Kafka
- **Feed ranking**: Wilson score (accounts for vote ratio + count) + time decay; pre-computed for top subreddits; stored in Redis sorted sets (`ZREVRANGEBYSCORE`)
- **Comment threads**: Materialized Path encoding — `1/4/7/22` means comment 22 is child of 7, child of 4, child of 1. Efficient prefix queries for subtrees; bounded depth assumption.
- **Hot posts**: background job re-ranks every 5 min; pushes to Redis sorted set; CDN caches top posts
- **Scaling reads**: PostgreSQL read replicas; Redis caches hot posts; CDN for all static assets; r/AskReddit front page served from cache — DB barely touched

---

### LLD: Rate Limiter

**Algorithms Compared**:

| Algorithm | Burst Handling | Memory | Accuracy |
|---|---|---|---|
| Token Bucket | Yes (burst up to bucket size) | O(1) per key | Good |
| Leaky Bucket | No (fixed output rate) | O(1) per key | Good |
| Fixed Window Counter | Boundary burst (2x) | O(1) per key | Poor at boundaries |
| Sliding Window Log | Perfect | O(requests) | Perfect |
| Sliding Window Counter | Good | O(1) per key | Good (approx) |

**Implementation with Redis (Sliding Window Counter)**:
```lua
-- Lua script for atomic execution
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

local windowStart = now - window
redis.call('ZREMRANGEBYSCORE', key, '-inf', windowStart)
local count = redis.call('ZCARD', key)

if count < limit then
  redis.call('ZADD', key, now, now .. math.random())
  redis.call('EXPIRE', key, window / 1000)
  return 1  -- allowed
end
return 0  -- rate limited
```

**Headers to return**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1718438400
Retry-After: 30
```

---

### LLD: Notification System

**Architecture**:
```
API (create notification)
  ↓
Notification Service
  ├── User preference check (opt-out, quiet hours, channel preference)
  ├── Template Engine (personalize message)
  └── Priority Router
       ├── Critical Queue (SQS FIFO) → Lambda → Email/SMS adapter
       ├── Transactional Queue (SQS Standard) → Lambda → FCM/APNS adapter
       └── Marketing Queue (SQS Standard, lower priority) → Lambda → Email batch adapter

Delivery Tracker (Kafka):
  SENT → DELIVERED → OPENED events → ClickHouse (analytics)
```

**Key Design Points**:
- Idempotency key: `notificationId` — deduplicate via Redis `SET nx` before sending; prevents double sends on retry
- Retry: exponential backoff (1s, 2s, 4s, 8s) → DLQ after 5 failures → alert on-call
- User preferences stored in Redis (hot path) + backed by PostgreSQL

---

### LLD: Autocomplete System

**Core Data Structure**: Trie with top-K suggestions cached at each node.

**At Scale**:
- Shard trie by first 2 characters: "ab-az" on server 1, "ba-bz" on server 2
- Store in Redis as `Hash: prefix → JSON array of [{suggestion, score}]`
- Update pipeline: Kafka (search query events) → Flink (count frequencies, sliding window) → Redis update (batch, every 10 min)
- Personalization: global trie result + user query history; merge and re-rank by personalized score
- Latency requirement: < 50ms → entire hot prefix tree in memory; Redis cluster for distribution
- Fuzzy matching for typos: DFA-based error-tolerant automaton; or: pre-index common misspellings

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | Designing Data-Intensive Applications — Kleppmann | https://dataintensive.net/ | Most cited system design book; replication, partitioning, consensus chapters are essential |
| 2 | High Scalability Blog | http://highscalability.com/ | Real architecture teardowns (WhatsApp, Twitter, Uber) with actual numbers from engineering teams |
| 3 | System Design Primer (GitHub) | https://github.com/donnemartin/system-design-primer | Comprehensive templates, patterns, and company-specific designs; 250k+ stars |

---

## 7. Database Design — PostgreSQL / SQL Server

### Key Concepts to Master

**Query Optimization**
- `EXPLAIN ANALYZE`: actual execution plan with real timing; use `EXPLAIN (ANALYZE, BUFFERS)` to see shared hits vs reads
- Seq Scan vs Index Scan vs Index-Only Scan: planner chooses based on cost model + statistics; low-selectivity queries → seq scan is cheaper (costs rows × page_cost)
- Join strategies: Nested Loop (small tables, indexed inner), Hash Join (large unsorted), Merge Join (pre-sorted inputs)
- `pg_stat_statements`: tracks slow queries in production; `mean_exec_time` + `calls` = identify N+1 patterns
- Stale statistics → wrong plan: run `ANALYZE table_name` to update; `autovacuum` does this automatically

**Index Strategies**
- **B-Tree** (default): equality, range, `LIKE 'prefix%'`, `ORDER BY`, `BETWEEN`
- **Hash**: equality only; slightly faster than B-Tree for pure equality; PostgreSQL WAL-logged since 10
- **Partial Index**: `CREATE INDEX ON orders(user_id) WHERE status = 'active'`; smaller and faster for filtered queries
- **Covering Index**: `CREATE INDEX ON orders(user_id) INCLUDE (status, total, created_at)`; index-only scan (no heap fetch)
- **Composite Index**: column order matters: `(a, b, c)` supports queries on `a`, `(a, b)`, `(a, b, c)`; NOT `b` alone; equality columns first, range column last
- **GIN Index**: for JSONB (`@>`, `?` operators), array columns, full-text search (`tsvector`)
- **Expression Index**: `CREATE INDEX ON users(lower(email))`; enables case-insensitive index lookups

**Advanced SQL**
- Window functions: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `SUM() OVER (PARTITION BY ... ORDER BY ...)`; evaluated after `WHERE`/`GROUP BY`, before `ORDER BY`/`LIMIT`
- CTEs: optimization fence in PG < 12 (always materialized); inlined by default in PG 12+; use `MATERIALIZED` keyword to force
- Lateral joins: subquery references outer query columns; returns multiple rows; essential for "top N per group"
- JSONB: binary storage, GIN-indexed, supports operators `@>` (contains), `?` (has key), `->>` (extract text)
- Partitioning: range/list/hash; partition pruning; use for time-series (monthly partitions on `created_at`)

**PostgreSQL Internals**
- MVCC: every UPDATE creates new row version; old version marked dead; `VACUUM` reclaims space
- WAL (Write-Ahead Log): all changes written to WAL before data files; enables crash recovery, replication, PITR
- Autovacuum: monitor `pg_stat_user_tables.n_dead_tup`; tune `autovacuum_vacuum_scale_factor` for hot tables
- Connection pooling: PostgreSQL spawns process per connection; PgBouncer pools; transaction mode incompatible with session-scoped prepared statements

---

### 15 Hard Interview Questions

**Q1. Explain MVCC in PostgreSQL and why it causes table bloat. How do you diagnose and fix it?**
> MVCC: every UPDATE writes a new row version, marks old as dead (not physically deleted). Every DELETE marks row as dead. Dead tuples accumulate. Table bloat = large percentage dead tuples = wasted space + slower sequential scans (must scan dead tuples). Diagnose: `SELECT relname, n_dead_tup, n_live_tup, last_autovacuum FROM pg_stat_user_tables WHERE n_dead_tup > 10000`. Fix: `VACUUM ANALYZE tablename`; tune autovacuum: `autovacuum_vacuum_scale_factor = 0.01` (trigger at 1% dead tuples for hot tables, default is 20%).

**Q2. You have index `(user_id, status)` and `(status, user_id)`. Query: `WHERE user_id = 5 AND status = 'pending'`. Which does the planner use and why?**
> For equality predicates on both columns, both indexes produce equally small result sets. Planner uses statistics from `pg_statistic` (updated by `ANALYZE`) to estimate selectivity. If `user_id` has high cardinality (many distinct values), `user_id = 5` is highly selective → `(user_id, status)` chosen because it immediately narrows to one user's rows. Run `EXPLAIN (ANALYZE, BUFFERS)` to see actual plan. Note: for this specific query with equality on both, the planner might even prefer a covering index that includes the `SELECT` columns to enable index-only scan.

**Q3. Demonstrate the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()` with actual output.**
```sql
SELECT name, score,
  ROW_NUMBER() OVER (ORDER BY score DESC) as row_num,
  RANK()       OVER (ORDER BY score DESC) as rank,
  DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank
FROM scores;
```
| name | score | row_num | rank | dense_rank |
|---|---|---|---|---|
| Alice | 100 | 1 | 1 | 1 |
| Bob | 90 | 2 | 2 | 2 |
| Carol | 90 | 3 | 2 | 2 |
| Dave | 80 | 4 | 4 | 3 |

> `RANK()` skips 3 (gap after tie). `DENSE_RANK()` never gaps. `ROW_NUMBER()` is always unique.

**Q4. Explain how a covering index eliminates heap fetches. Show with EXPLAIN output.**
```sql
-- Without covering index:
CREATE INDEX idx_orders_user ON orders(user_id);
EXPLAIN SELECT status, total FROM orders WHERE user_id = 5;
-- Output: "Index Scan using idx_orders_user" + "Heap Fetches: 150"

-- With covering index:
CREATE INDEX idx_orders_user_cover ON orders(user_id) INCLUDE (status, total);
EXPLAIN SELECT status, total FROM orders WHERE user_id = 5;
-- Output: "Index Only Scan using idx_orders_user_cover" + "Heap Fetches: 0"
```
> Normal index scan: finds matching rows in index → fetch full row from heap (random I/O). Index-only scan: all needed columns in index → no heap access. Massive performance gain for high-frequency queries on large tables.

**Q5. N+1 problem in raw SQL: what is it, how do you detect it in production, and show the fix.**
> N+1: fetch list of N users, then for each user run `SELECT * FROM orders WHERE user_id = $id` = N+1 queries. Detection: `pg_stat_statements` — find statements with high `calls` and similar query patterns. APM (Datadog, New Relic) shows repeated identical queries per request. Fix:
```sql
-- N+1 (bad):
-- SELECT * FROM users WHERE id IN (1,2,3)
-- SELECT * FROM orders WHERE user_id = 1
-- SELECT * FROM orders WHERE user_id = 2
-- SELECT * FROM orders WHERE user_id = 3

-- Fixed with JOIN:
SELECT u.*, o.id, o.total, o.status
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id IN (1,2,3);

-- Or with IN batch:
SELECT * FROM orders WHERE user_id IN (1,2,3);
```

**Q6. CTE as optimization fence (PG < 12): demonstrate with a query where it causes a full table scan.**
```sql
-- PG < 12: this CTE forces a seq scan on users, even with index on email
WITH active_users AS (
  SELECT * FROM users WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT * FROM active_users WHERE email = 'test@example.com';
-- Planner: scans ALL rows in CTE first (can't push email = '...' into CTE)

-- PG 12+: inlined by default (same as subquery); planner pushes predicate in
-- Or force materialization:
WITH active_users AS MATERIALIZED (
  SELECT * FROM users WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT * FROM active_users WHERE email = 'test@example.com';
```
> Always `EXPLAIN` CTEs in PG versions before 12.

**Q7. Write a query using a Lateral join to get the top 3 most recent orders per user.**
```sql
SELECT u.id, u.name, o.id AS order_id, o.created_at, o.total
FROM users u
CROSS JOIN LATERAL (
  SELECT id, created_at, total
  FROM orders
  WHERE user_id = u.id
  ORDER BY created_at DESC
  LIMIT 3
) o;
```
> LATERAL allows the subquery to reference `u.id` from the outer query. Returns multiple rows per user (up to 3). Cannot do this with a regular subquery. `CROSS JOIN LATERAL` with LIMIT = efficient "top N per group" pattern.

**Q8. Design a time-series table for sensor readings with PostgreSQL partitioning. Include maintenance.**
```sql
-- Parent table
CREATE TABLE sensor_readings (
  sensor_id   INT NOT NULL,
  recorded_at TIMESTAMPTZ NOT NULL,
  value       FLOAT NOT NULL
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions
CREATE TABLE sensor_readings_2026_06
  PARTITION OF sensor_readings
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Index on each partition
CREATE INDEX ON sensor_readings_2026_06 (sensor_id, recorded_at DESC);

-- Maintenance: drop old partitions (fast — no VACUUM needed)
DROP TABLE sensor_readings_2024_01;

-- Automate with pg_partman extension
```
> Partition pruning: `WHERE recorded_at > '2026-06-01'` only scans June partition. Dropping old partition = instant DDL (vs `DELETE` which leaves dead tuples).

**Q9. What is `EXPLAIN (ANALYZE, BUFFERS)` telling you? Explain "shared hit" vs "shared read".**
> `shared hit`: blocks served from PostgreSQL `shared_buffers` (RAM) — fast (~0.1ms). `shared read`: blocks read from OS page cache or disk — slower (1-10ms disk). High `shared read` ratio means working set doesn't fit in `shared_buffers` (default is 128MB — almost always insufficient; set to 25% of RAM). Also look for: rows estimate vs actual rows (large gap → stale stats → run `ANALYZE`); `Sort Method: external merge` (work_mem too low for in-memory sort → increase `work_mem`); `Loops:` count on Nested Loop (high loops on large tables = index needed on inner relation).

**Q10. How does PgBouncer transaction mode break prepared statements, and what's the fix?**
> PostgreSQL prepared statements (`PREPARE stmt AS SELECT...`, `EXECUTE stmt`) are session-scoped — exist for the life of a backend connection. PgBouncer transaction mode: after each transaction, client's connection is returned to pool; next transaction may use a different backend. Backend B doesn't have the prepared statement that was created on backend A. Error: `prepared statement "stmt_name" does not exist`. Fixes: 1) Use PgBouncer session mode (less pooling efficiency). 2) Disable prepared statements in your driver (`?prepareThreshold=0` for JDBC, `prepare: false` for node-postgres). 3) Use pgBouncer's built-in `prepared_statement_cache_size`.

**Q11. Design a schema for a multi-tenant SaaS. Compare the three isolation models.**
> **Row-Level Security (RLS)**:
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::INT);
```
> Simple ops; all tenants in same tables; noisy neighbor risk; GDPR isolation concerns.

> **Schema-per-tenant**: `SET search_path = tenant_42; SELECT * FROM orders;` — strong isolation; migration tooling complexity (Flyway × 10,000 schemas).

> **Database-per-tenant**: full isolation; independent scaling; expensive ops at scale; use for enterprise/compliance. Choose RLS for SMB SaaS. Schema-per-tenant for mid-market. Database-per-tenant for enterprise with strict data residency.

**Q12. What are the tradeoffs of using JSONB columns vs normalized tables in PostgreSQL?**
> JSONB pros: flexible schema (no migration for new fields), stores arbitrary nested structures, GIN-indexed for key/value lookup. JSONB cons: no foreign keys, no joins on JSON fields, query syntax verbose (`data->>'price'`), type coercion issues, harder to enforce constraints, storage overhead (binary). Use JSONB for: semi-structured data (product attributes that vary by category), audit logs, external API data you don't control. Use normalized tables for: anything you JOIN, filter, aggregate, or enforce constraints on. Hybrid: normalized columns for hot query paths + JSONB for flexible attributes.

**Q13. Explain WAL (Write-Ahead Log) and how it enables Point-in-Time Recovery (PITR).**
> All changes written to WAL (sequential append-only log) BEFORE data files are modified. On crash: replay WAL from last checkpoint to restore data files. PITR: 1) Take base backup (`pg_basebackup`). 2) Archive WAL files continuously to S3. 3) To restore to T-minus-2-hours: restore base backup + replay archived WAL files up to target timestamp. WAL streaming replication: standby continuously receives WAL stream from primary — no data loss if standby is caught up. `synchronous_commit = off`: improves write performance; risk: last ~1 transaction lost on crash (WAL written async).

**Q14. Your PostgreSQL query with a perfect index is still slow. What are the non-obvious causes?**
> 1) **Statistics out of date**: planner estimates 10 rows, actual = 10,000 → wrong join strategy. Run `ANALYZE`. 2) **Low `work_mem`**: sort operations spill to disk. Set `SET work_mem = '256MB'` for the session and re-run. 3) **Index bloat**: high dead tuples in index → `REINDEX TABLE tablename`. 4) **Correlation**: index on `created_at` but physical row order doesn't match (correlation ≈ 0) → index scan slower than seq scan. Check `pg_stats.correlation`. 5) **Connection overhead**: hundreds of short-lived connections → add PgBouncer. 6) **Lock contention**: `SELECT * FROM pg_locks` — blocked by concurrent writes.

**Q15. Write a query to find the second highest salary per department without using `LIMIT`.**
```sql
-- Option 1: Window function (preferred)
SELECT department_id, employee_id, salary
FROM (
  SELECT department_id, employee_id, salary,
    DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as rnk
  FROM employees
) ranked
WHERE rnk = 2;

-- Option 2: Correlated subquery (less efficient)
SELECT department_id, MAX(salary) as second_highest
FROM employees e1
WHERE salary < (
  SELECT MAX(salary) FROM employees e2
  WHERE e2.department_id = e1.department_id
)
GROUP BY department_id;
```
> Window function approach is O(n log n) for sorting; correlated subquery is O(n²). Always prefer window functions.

---

### Study Resources

| # | Resource | URL | Why It's Valuable |
|---|---|---|---|
| 1 | Use the Index, Luke — Markus Winand | https://use-the-index-luke.com/ | Definitive free resource on SQL indexing; covers B-Tree internals and query plan analysis |
| 2 | PostgreSQL Docs — Query Planning | https://www.postgresql.org/docs/current/performance-tips.html | Primary source for EXPLAIN, planner settings, statistics; no secondary source is more accurate |
| 3 | Cybertec PostgreSQL Blog | https://www.cybertec-postgresql.com/en/blog/ | Advanced content from core contributors; covers VACUUM, partitioning, JSONB — exactly the depth expected at senior level |

---

## Resources Index

### React / Next.js
| Resource | URL |
|---|---|
| React Fiber Architecture (Andrew Clark) | https://github.com/acdlite/react-fiber-architecture |
| React 18 Working Group Discussions | https://github.com/reactwg/react-18/discussions |
| Next.js App Router (Vercel Blog) | https://nextjs.org/blog/next-13-4 |
| React.dev Official Docs | https://react.dev/ |
| Airbnb Performance Engineering | https://medium.com/airbnb-engineering/recent-web-performance-fixes-on-airbnb-listing-pages-6cd8d93df6f4 |
| Google web.dev Core Web Vitals | https://web.dev/vitals/ |

### Node.js / Express
| Resource | URL |
|---|---|
| Node.js AsyncLocalStorage Docs | https://nodejs.org/api/async_context.html |
| Node.js Streams Official Docs | https://nodejs.org/api/stream.html |
| Node Clinic (NearForm) | https://clinicjs.org/ |
| Node.js Design Patterns Book | https://www.nodejsdesignpatterns.com/ |
| OWASP Node.js Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html |

### AWS
| Resource | URL |
|---|---|
| AWS Well-Architected Framework | https://aws.amazon.com/architecture/well-architected/ |
| AWS re:Invent Talks (YouTube) | https://www.youtube.com/@AWSEventsChannel |
| AWS Architecture Blog | https://aws.amazon.com/blogs/architecture/ |
| AWS Lambda Power Tuning (GitHub) | https://github.com/alexcasalboni/aws-lambda-power-tuning |
| AWS Serverless Land | https://serverlessland.com/ |

### CI/CD / Docker
| Resource | URL |
|---|---|
| OWASP Docker Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html |
| GitLab CI Pipeline Efficiency | https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html |
| Google SRE Book — Release Engineering | https://sre.google/sre-book/release-engineering/ |
| Docker BuildKit Docs | https://docs.docker.com/build/buildkit/ |
| Trivy Container Scanner | https://github.com/aquasecurity/trivy |

### DSA
| Resource | URL |
|---|---|
| NeetCode.io | https://neetcode.io/ |
| Competitive Programmer's Handbook (free PDF) | https://cses.fi/book/book.pdf |
| LeetCode Company Tags | https://leetcode.com/company/ |
| r/cscareerquestions Interview Wiki | https://www.reddit.com/r/cscareerquestions/wiki/index |

### System Design
| Resource | URL |
|---|---|
| Designing Data-Intensive Applications | https://dataintensive.net/ |
| High Scalability Blog | http://highscalability.com/ |
| System Design Primer (GitHub) | https://github.com/donnemartin/system-design-primer |
| ByteByteGo | https://bytebytego.com/ |
| Martin Fowler's Blog | https://martinfowler.com/ |
| Netflix TechBlog | https://netflixtechblog.com/ |
| Uber Engineering Blog | https://www.uber.com/blog/engineering/ |
| InfoQ Architecture | https://www.infoq.com/architecture-design/ |

### Database
| Resource | URL |
|---|---|
| Use the Index, Luke | https://use-the-index-luke.com/ |
| PostgreSQL Performance Tips (official) | https://www.postgresql.org/docs/current/performance-tips.html |
| Cybertec PostgreSQL Blog | https://www.cybertec-postgresql.com/en/blog/ |
| pganalyze Blog | https://pganalyze.com/blog |

---

*Save as `interview-prep.md` · Export to PDF via [stackedit.io](https://stackedit.io) or [dillinger.io](https://dillinger.io)*
*Date: June 15, 2026 · Target: 10-Day Sprint*
````

---

**To save this:**
- Copy everything between the ```` ```md ```` fences above
- Paste into VS Code → save as `interview-prep.md`
- For PDF: paste into [stackedit.io](https://stackedit.io) → File → Export as PDF```md ```` fences above
- Paste into VS Code → save as `interview-prep.md`
- For PDF: paste into [stackedit.io](https://stackedit.io) → File → Export as PDF
