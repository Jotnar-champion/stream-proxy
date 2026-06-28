# Senior Full-Stack Interview Prep — 10-Day Sprint Guide

**Date: June 15, 2026**  
**Target: Senior/Lead Full-Stack — Adobe CS2, Walmart, Uber, Flipkart, 
MakeMyTrip, Cars24, Nagarro**

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

#### React Fiber & Scheduler

- **Fiber unit of work**: Linked-list tree (not recursive call stack)
- **Double-buffering**: `current` tree (on screen) + `workInProgress` tree 
  (being built)
- **Render phase**: Pure, interruptible
- **Commit phase**: Effectful, synchronous, cannot be interrupted
- **Lanes model (React 18)**: Bitmask priority system replacing 
  `expirationTime`
  - SyncLane > InputContinuousLane > DefaultLane > TransitionLane > IdleLane

#### Concurrent Features (React 18)

- **`useTransition`**: Wraps state updates you control; marks them 
  low-priority
  - Returns `[isPending, startTransition]`
- **`useDeferredValue`**: For values you don't control (from props)
  - Deferred copy lags behind urgent renders
- **Automatic batching**: ALL updates batched inside `setTimeout`, 
  Promises, native events
- **`flushSync`**: Opt out of batching; forces synchronous flush
  - Blocks main thread — use sparingly

#### Next.js App Router Internals

- **RSC (React Server Components)**: Execute on server, zero client JS
  - Can't use hooks/browser APIs
  - Direct DB access possible
- **`"use client"` directive**: Marks component tree boundary
  - Only these components hydrate
- **Streaming**: `loading.tsx` wraps route in Suspense
  - React streams HTML in chunks as data resolves
- **Partial Prerendering (PPR)**: Static shell from CDN + dynamic holes 
  streamed at request time
  - Defined by Suspense boundaries
- **ISR**: `fetch(url, { next: { revalidate: 60 } })`
  - Stale-while-revalidate semantics
- **Route handlers**: Replace API routes in App Router
  - Run in Node.js or Edge runtime
- **Middleware**: Runs on Edge Runtime (V8 isolate, NOT Node.js)
  - No `fs`, no native modules

#### Optimization Beyond Memoization

- **`React.memo`**: Only prevents re-render if props referentially equal
  - Useless without stable references
- **Virtualization**: `react-window` / `tanstack-virtual` for lists > 100 items
- **Code splitting**: `React.lazy` + `Suspense`
  - `dynamic()` in Next.js with `{ ssr: false }` for client-only libs
- **Bundle analysis**: `@next/bundle-analyzer`
  - Look for duplicate packages, large moment/lodash imports
- **Core Web Vitals**:
  - LCP (largest contentful paint < 2.5s)
  - CLS (cumulative layout shift < 0.1)
  - INP (interaction to next paint < 200ms — replaced FID in 2024)
- **TTFB**: Reduce server compute, use edge caching, streaming, 
  DB query optimization

#### Memory Leaks in React

- **`useEffect` without cleanup**: Event listeners, subscriptions, 
  WebSockets, timers
- **Closures capturing stale references**: In async operations
- **`AbortController`**: For cancelling in-flight `fetch` on unmount
- **Selector issues**: Zustand/Redux selectors creating new object 
  references on every call

### 15 Hard Interview Questions

**Q1. React 17 vs React 18 batching: where exactly does React 17 NOT 
batch, and what does React 18 change?**

> React 17: Batching only in synthetic event handlers (React's own event 
> system). Inside `setTimeout`, `Promise.then`, `addEventListener` callbacks 
> — each `setState` triggers separate re-render. React 18 with `createRoot`: 
> automatic batching everywhere. Opt out with `flushSync(() => setState(...))`. 
> Interview trap: people say "React always batched" — wrong; it's new in 
> React 18 universally.

**Q2. Explain the stale closure problem in this code and provide two 
different fixes:**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => console.log(count), 1000);
    return () => clearInterval(id);
  }, []);
}
```

> **Problem**: `count` in the interval closure captures `0` at mount time 
> (empty dep array). The interval always logs `0`.
>
> **Fix 1**: Add `count` to deps array (creates new interval on every count 
> change — acceptable for small intervals).
>
> **Fix 2**: Use a ref to track latest value without causing re-render:
> ```javascript
> const countRef = useRef(count);
> useEffect(() => { countRef.current = count; }, [count]);
> useEffect(() => {
>   const id = setInterval(() => console.log(countRef.current), 1000);
>   return () => clearInterval(id);
> }, []);
> ```
>
> **Fix 3** (if only incrementing): `setCount(c => c + 1)` functional updater 
> doesn't need current count in closure.

**Q3. What is the React Fiber "double buffer" pattern? Why is it critical 
for concurrent rendering?**

> React maintains two trees simultaneously: `current` (what's visible on 
> screen) and `workInProgress` (being built). During the render phase, React 
> builds the WIP tree. On commit, it atomically swaps the pointers. This 
> enables interruption: if a higher-priority update arrives mid-render, React 
> can abandon the incomplete WIP tree and restart with the new priority 
> without corrupting the visible UI. Without double buffering, a 
> partially-rendered tree could be committed to the screen in a broken state.

**Q4. Explain hydration mismatches in Next.js. What exactly does React do 
when it detects one, and what are all the causes?**

> When React hydrates, it compares server-rendered HTML against the client's 
> first render output. If they differ, React logs a warning and **discards 
> the server HTML entirely**, doing a full client-side render — completely 
> defeating SSR benefits.
>
> **Causes**:
> - `new Date()` (different timezone/time between server and client)
> - `Math.random()`
> - `typeof window` guards not applied correctly
> - Browser extensions injecting DOM nodes
> - CSS-in-JS generating different class names
> - Locale-dependent formatting
>
> **Fix**: `suppressHydrationWarning` for intentional mismatches; `useEffect` 
> for client-only rendering; ensure server and client render identical 
> output on first pass.

**Q5. `useTransition` vs `useDeferredValue` — what's the fundamental 
difference and when do you use each?**

> **`useTransition`**: You OWN the state update. Wrap the update:
> ```javascript
> startTransition(() => setState(val))
> ```
> React marks this update as non-urgent and can interrupt it for urgent ones 
> (typing, clicking).
>
> **`useDeferredValue`**: You DON'T own the state update (it comes from a 
> prop, context, or library). You create a deferred copy:
> ```javascript
> const deferredQuery = useDeferredValue(query)
> ```
> The deferred copy lags behind while urgent renders complete. Use 
> `useDeferredValue` for: rendering a filtered list from a prop you can't 
> wrap in `startTransition`.

**Q6. You have 10,000 rows rendered with `React.memo`. Performance is still 
poor. Walk through your complete diagnosis.**

> **Step 1**: React DevTools Profiler — identify which components re-render 
> and WHY (highlight "why did this render").
>
> **Step 2**: Check if parent state triggers context re-render (context 
> value is a new object reference on every render).
>
> **Step 3**: Verify `React.memo` comparison — if props include 
> arrays/objects created inline, memo always fails.
>
> **Step 4**: Check if the component itself calls expensive computations 
> inside render without `useMemo`.
>
> **Fixes in order**:
> - Virtualize with `react-window` (render only visible rows)
> - Stabilize context with `useMemo` / split context
> - `useMemo` for expensive derived values
> - Verify memo comparison with custom comparator
>
> **Bottom line**: `React.memo` without stable prop references is useless.

**Q7. What is "tearing" in React concurrent mode and how does 
`useSyncExternalStore` solve it?**

> **Tearing**: During a concurrent render (which is interruptible), React 
> may render different parts of the component tree at different moments in 
> time. If an external store (Redux, Zustand) updates between those renders, 
> some components see the old value and some see the new — the UI is in an 
> inconsistent ("torn") state.
>
> **Solution**: `useSyncExternalStore` (React 18) solves this by:
> - Subscribing to the external store
> - Forcing a synchronous re-render if the store changes during a concurrent 
>   render
> - Ensuring all components see the same snapshot
>
> All major state managers (Redux, Zustand) use this internally in React 18.

**Q8. Explain Next.js ISR: a user hits a page 500ms after `revalidate: 60` 
expires. What do they see? What happens next?**

> They see the **stale cached page immediately** (served from CDN/cache). 
> Next.js simultaneously triggers a background revalidation request to 
> regenerate the page. The user who triggered revalidation sees old data. 
> The NEXT user after revalidation completes sees fresh data. This is 
> stale-while-revalidate (SWR) semantics — prioritize availability/speed 
> over freshness. Key interview point: if revalidation fails, the stale page 
> continues to be served (no error shown to users).

**Q9. Why can a React Server Component not use `useState`, and what exactly 
happens at the serialization boundary?**

> RSCs run on the server once and produce a serializable React element tree 
> (not HTML — a JSON-like wire format). They have no lifecycle, no 
> client-side re-renders. `useState` requires client-side JS to persist 
> state between renders — RSCs have no client runtime.
>
> **At the serialization boundary** (where RSC passes props to a Client 
> Component): Only serializable values can cross — strings, numbers, arrays, 
> plain objects, Dates, JSX. Functions, class instances, closures, React 
> context, and non-serializable values CANNOT cross. This is why "you can't 
> pass a callback from Server to Client Component as a prop."

**Q10. Describe a memory leak in a React component using a WebSocket 
subscription. Write the buggy code and the fix.**

> **Buggy**:
> ```javascript
> useEffect(() => {
>   const ws = new WebSocket(url);
>   ws.onmessage = (e) => setMessages(prev => [...prev, e.data]);
>   // No cleanup!
> }, [url]);
> ```
>
> On unmount (route change, conditional render), the WebSocket stays open 
> and fires events, calling `setMessages` on an unmounted component — memory 
> leak + React warning.
>
> **Fix**:
> ```javascript
> useEffect(() => {
>   const ws = new WebSocket(url);
>   ws.onmessage = (e) => setMessages(prev => [...prev, e.data]);
>   return () => {
>     ws.close();
>     ws.onmessage = null;
>   };
> }, [url]);
> ```

**Q11. What is Partial Prerendering (PPR) in Next.js and how is it 
architecturally different from ISR?**

> **ISR**: Regenerates the ENTIRE page on a schedule; the whole page is 
> either fresh or stale.
>
> **PPR**: A SINGLE route has a static shell (generated at build time, 
> served from CDN instantly) and dynamic holes (Suspense-boundary-wrapped 
> RSCs that stream in at request time). Different parts of the same page 
> can be static or dynamic independently.
>
> PPR is more granular — your header, navigation, and footer are static; 
> your user-specific feed streams in dynamically. Defined by wrapping 
> dynamic components in `<Suspense>` boundaries.

**Q12. What causes high CLS (Cumulative Layout Shift) and walk through 
fixing it in a Next.js app?**

> **Causes**:
> - Images without explicit `width`/`height` (browser doesn't reserve space)
> - Web fonts causing FOUT (flash of unstyled text — elements shift when 
>   font loads)
> - Dynamic content injected above the fold (banners, cookie notices)
> - Embeds without fixed dimensions
>
> **Diagnosis**: Chrome DevTools → Performance → Layout Shift regions; 
> `web-vitals` library, Vercel Speed Insights.
>
> **Fixes**:
> - Always use Next.js `<Image>` (auto-reserves space)
> - `font-display: optional` or preload critical fonts
> - CSS `aspect-ratio` for containers whose content loads async
> - `min-height` on skeleton placeholders that match final content dimensions

**Q13. You have a large form (50+ fields) where every keystroke 
re-renders the entire form. What's your architecture?**

> **Root cause**: Controlled inputs with top-level state cause full tree 
> re-renders.
>
> **Solutions in order of invasiveness**:
> 1. `React Hook Form` — uncontrolled approach, no state per keystroke, 
>    only re-renders on validation/submit
> 2. Split form into separate components with isolated state — changes in 
>    Section A don't re-render Section B
> 3. `useReducer` at top + `React.memo` on each field with stable 
>    `dispatch` reference
> 4. State management (Jotai atom per field) for maximum isolation
>
> **Never** use a single `useState` object for 50+ fields.

**Q14. `useCallback` — when does it actually PREVENT re-renders and when 
does it FAIL to?**

> **It PREVENTS re-renders when**:
> 1. The child is wrapped in `React.memo`, AND
> 2. The `useCallback` deps haven't changed (so the function reference 
>    is stable)
>
> **It FAILS when**:
> 1. Child is NOT wrapped in `React.memo` (doesn't matter — child re-renders 
>    anyway)
> 2. Deps change every render (defeats purpose)
> 3. Child receives OTHER unstable props alongside the stable callback
>
> **Most common mistake**: Adding `useCallback` everywhere "for performance" 
> without `React.memo` on consumers — pure overhead with zero benefit.

**Q15. Explain Next.js Middleware vs API Route handlers. When does 
Middleware make it worse, not better?**

> **Middleware** runs on Edge Runtime (Cloudflare Workers-like V8 isolate) 
> BEFORE the request hits cache or origin. Zero Node.js APIs.
>
> **Use for**:
> - Auth token validation
> - Geo-routing
> - A/B testing
> - Header injection
>
> **Middleware makes it WORSE when**:
> - You need to query a database (no native DB drivers in Edge)
> - You need complex business logic (better in an API route)
> - You need to read the request body of a POST (Middleware can't easily 
>   do this)
> - You're adding latency to every request for logic that only applies to 
>   a few routes (use route-level middleware instead)

### Study Resources

| # | Resource | URL |
|----|----------|-----|
| 1 | React Fiber Architecture — Andrew Clark | https://github.com/acdlite/react-fiber-architecture |
| 2 | React 18 Working Group (GitHub Discussions) | https://github.com/reactwg/react-18/discussions |
| 3 | Next.js App Router Architecture (Vercel Blog) | https://nextjs.org/blog/next-13-4 |

---

## 2. Node.js / Express

### Key Concepts to Master

#### Event Loop Phases (Deep)

- **Order**: timers → pending callbacks → idle/prepare → poll → check 
  → close callbacks
- **`process.nextTick`**: NOT a phase; runs after current operation, 
  before any I/O
  - Can starve the event loop if recursive
- **`Promise.then` (microtasks)**: Runs after nextTick queue, before 
  returning to event loop phases
- **`setImmediate`**: check phase; inside an I/O callback it ALWAYS runs 
  before `setTimeout(fn, 0)`
- **`setTimeout(fn, 0)`**: timers phase; outside I/O, order vs 
  `setImmediate` is non-deterministic (OS scheduler)
- **Priority**: 
  - `process.nextTick` > `Promise microtasks` > `setImmediate` 
    > `setTimeout(fn,0)`

#### Concurrency Models

- **Single-threaded event loop**: I/O delegated to libuv thread pool 
  (default 4 threads, set via `UV_THREADPOOL_SIZE`)
- **Worker Threads**: True parallel JS; shared memory via 
  `SharedArrayBuffer` + `Atomics`; use for CPU-bound JS
- **Cluster**: `cluster.fork()` spawns multiple Node processes sharing 
  a port; OS distributes connections; use for multi-core I/O scaling
- **Child Processes**: `spawn` (stream I/O), `exec` (buffer output), 
  `fork` (IPC-enabled Node process); use for non-JS executables or 
  isolated processes
- **Decision tree**:
  - I/O-bound → event loop handles it
  - CPU-bound JS → Worker Threads
  - CPU-bound non-JS → `spawn`
  - Multiple cores for I/O → Cluster

#### Streams & Backpressure

- **`writable.write()`** returns `false` when internal buffer exceeds 
  `highWaterMark` — consumer can't keep up
- **Correct backpressure**: Pause readable on `false`, resume on `drain` 
  event
- **`pipeline()`** (Node 10+): Automatically handles backpressure, cleanup, 
  and error propagation — always prefer over manual `.pipe()`
- **`objectMode: true`**: Streams work with arbitrary JS objects
  - `highWaterMark` = count of objects (default 16), not bytes
- **Transform streams**: Both Readable and Writable; use for processing 
  pipelines (compression, encryption, parsing)

#### Express Production Patterns

- **Middleware order**: `helmet()` → `cors()` → `express.json()` 
  → `express.urlencoded()` → routes → 404 handler → error handler
- **Error handler**: 4-parameter signature `(err, req, res, next)`
  - Must be registered LAST
- **Async error propagation in Express 4**: Rejected promises in route 
  handlers DO NOT automatically call error handler — must wrap or use 
  Express 5
- **`trust proxy`**: Required when behind nginx/ALB/ELB for correct 
  `req.ip`, `req.protocol`, `X-Forwarded-*` headers
- **`express.Router()`**: Mount per domain/feature; keeps route files 
  isolated

#### Security (OWASP)

- **`helmet()`**: Sets 11 security headers (HSTS, X-Frame-Options, CSP, 
  X-Content-Type-Options)
- **Input validation at boundaries** with `zod` or `joi`
  - Never trust `req.body` shape
- **Rate limiting**: `express-rate-limit` + Redis store for distributed 
  environments
- **Prototype pollution**: `{"__proto__": {"isAdmin": true}}`
  - Use `Object.create(null)` for untrusted data
  - Use `secure-json-parse`
  - Use `zod` schema validation
- **Never log `req.body`** in production without sanitization (PII, secrets)

### 15 Hard Interview Questions

**Q1. Exact priority order: `process.nextTick` vs `Promise.then` vs 
`setImmediate` vs `setTimeout(fn,0)`. Explain with output prediction.**

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
```

> **Output**: `sync` → `nextTick` → `promise` → (then either `timeout` 
> or `immediate`, non-deterministic outside I/O).
>
> **Inside an I/O callback**: `nextTick` → `promise` → `immediate` 
> → `timeout` (setImmediate always before setTimeout inside I/O).

**Q2. The 5th `fs.readFile` call hangs momentarily. Why? How do you fix 
it for high-concurrency production?**

> libuv thread pool defaults to 4 threads. `fs.readFile`, `dns.lookup`, 
> `crypto` operations all share this pool. The 5th concurrent operation 
> queues.
>
> **Fix**: `UV_THREADPOOL_SIZE=64` (environment variable, max 1024).
>
> **Better**: Use `dns.resolve()` instead of `dns.lookup()` (uses real 
> async DNS API, not thread pool). For file I/O at scale: use streams 
> instead of buffering entire file, or offload to a dedicated microservice.

**Q3. This backpressure implementation causes OOM in production. Explain 
why and write the correct version.**

```javascript
readable.on('data', chunk => writable.write(chunk));
```

> `writable.write()` returning `false` is ignored — readable keeps emitting, 
> chunks buffer in memory.
>
> **Fix**:
> ```javascript
> readable.on('data', chunk => {
>   if (!writable.write(chunk)) readable.pause();
> });
> writable.on('drain', () => readable.resume());
> readable.on('end', () => writable.end());
> ```
>
> **Better** — use `pipeline`:
> ```javascript
> const { pipeline } = require('stream/promises');
> await pipeline(readable, writable);
> // handles backpressure + cleanup + errors
> ```

**Q4. Express 4 async middleware bug: why doesn't this error reach the 
error handler?**

```javascript
app.get('/users', async (req, res, next) => {
  const users = await db.getUsers(); // throws
  res.json(users);
});
```

> Express 4 route handlers are not promise-aware. An unhandled rejection 
> in async function doesn't call `next(err)` — it becomes an unhandled 
> promise rejection. Express never sees it.
>
> **Fix**:
> ```javascript
> const asyncHandler = fn => (req, res, next) =>
>   Promise.resolve(fn(req, res, next)).catch(next);
>
> app.get('/users', asyncHandler(async (req, res) => {
>   const users = await db.getUsers();
>   res.json(users);
> }));
> ```
>
> Express 5 (released) handles this natively — async route handlers' 
> rejections auto-call `next(err)`.

**Q5. Explain `SharedArrayBuffer` + `Atomics` in Worker Threads. What 
happens without `Atomics`?**

> Worker Threads can share memory via `SharedArrayBuffer` — both threads 
> read/write the same memory address. Without `Atomics`, operations aren't 
> atomic: thread A reads value (5), thread B reads value (5), both add 1, 
> both write 6 — you lose an increment.
>
> `Atomics.add(view, index, 1)` performs the read-modify-write atomically 
> at the hardware level. `Atomics.wait()` / `Atomics.notify()` implement 
> futex-like synchronization for coordination between threads.

**Q6. How do you perform a memory heap snapshot on a production Node.js 
service without downtime?**

> 1. Expose a protected admin endpoint:
> ```javascript
> app.get('/admin/heapdump', authMiddleware, (req, res) => {
>   v8.writeHeapSnapshot();
>   res.json({ ok: true });
> });
> ```
>
> 2. Use `--heap-prof` flag: `node --heap-prof app.js`
>    - Generates `.heapprofile` on exit
>
> 3. `clinic heapprofiler` (NearForm Node Clinic) for detailed allocation 
>    tracking
>
> 4. `--inspect=127.0.0.1:9229` on a staging instance with 
>    production-equivalent load → Chrome DevTools Memory tab
>    - Take 3 heap snapshots → compare for retained objects
>
> 5. Monitor `process.memoryUsage().heapUsed` via custom CloudWatch metric
>    - Graph over time to confirm linear growth (leak pattern)

**Q7. What is prototype pollution, give a real attack vector in an Express 
app, and list three mitigations.**

> **Attack**: 
> ```
> POST body {"__proto__": {"isAdmin": true}} + Object.assign({}, req.body)
> → ({}).isAdmin === true globally for all objects
> ```
> Express app then grants admin access to every subsequent request.
>
> **Mitigations**:
> 1. Use `zod`/`joi` schema validation — both reject `__proto__` and 
>    `constructor` keys
> 2. `const obj = Object.create(null)` for untrusted object accumulation
>    — no prototype chain
> 3. `npm install secure-json-parse` — replaces `JSON.parse` with 
>    pollution-safe version
> 4. `Object.freeze(Object.prototype)` — prevents modification (may break 
>    some libraries)
> 5. `express.json({ reviver: ... })` to strip dangerous keys

**Q8. Explain `AsyncLocalStorage` and write a request tracing 
implementation.**

> `AsyncLocalStorage` provides request-scoped storage that automatically 
> propagates through all async operations in a request's execution tree 
> without passing context explicitly.
>
> ```javascript
> const { AsyncLocalStorage } = require('async_hooks');
> const requestContext = new AsyncLocalStorage();
>
> app.use((req, res, next) => {
>   const store = {
>     requestId: crypto.randomUUID(),
>     userId: req.user?.id
>   };
>   requestContext.run(store, next);
> });
>
> // Anywhere in the codebase — no need to pass requestId as parameter:
> function queryDatabase(sql) {
>   const ctx = requestContext.getStore();
>   logger.info({ requestId: ctx?.requestId, sql }, 'DB query');
>   return db.query(sql);
> }
> ```

**Q9. You have a Node.js service with random latency spikes every few 
minutes. No errors. What's your investigation?**

> **First hypothesis**: GC pause (major GC / mark-and-sweep).
> - Verify: `node --trace-gc app.js` — logs GC events with duration
> - If 50-200ms pauses align with spikes → heap is growing too large
>
> **Second hypothesis**: libuv thread pool saturation
>
> **Third**: `setInterval` or cron job blocking event loop
>
> **Tools**:
> - `node --prof` → `node --prof-process` for V8 profiler flame graph
> - `clinic doctor` (NearForm) for automated diagnosis
>
> **Also check**:
> - DNS resolution (`dns.lookup` blocks thread pool)
> - `bcrypt` calls in request path (CPU-bound crypto)

**Q10. Design graceful shutdown for an Express server that gets 
Kubernetes SIGTERM signals.**

```javascript
const server = app.listen(3000);
let isShuttingDown = false;

process.on('SIGTERM', () => {
  isShuttingDown = true;
  server.close(async () => {
    // Stop accepting new connections
    await db.pool.end();      // Close DB connections
    await redisClient.quit(); // Close Redis
    logger.info('Graceful shutdown complete');
    process.exit(0);
  });

  // Force exit if graceful shutdown takes too long (K8s SIGKILL = 30s)
  setTimeout(() => process.exit(1), 25000);
});

// Reject new requests during shutdown
app.use((req, res, next) => {
  if (isShuttingDown) {
    return res.status(503).json({ error: 'Service unavailable' });
  }
  next();
});
```

**Q11. `cluster` vs PM2 cluster for zero-downtime deploys. What does PM2 
`reload` do that `restart` doesn't?**

> **`pm2 restart`**: Kills ALL workers simultaneously, starts new ones 
> — brief downtime.
>
> **`pm2 reload`**: Rolling restart — sends `SIGINT` to ONE worker, waits 
> for it to drain and exit (or timeout), starts a replacement, then moves 
> to the next. Zero downtime because at least one worker is always alive.
>
> **To implement manually with `cluster` module**:
> - Iterate `cluster.workers`, send SIGINT one at a time
> - Wait for `exit` event
> - `cluster.fork()` replacement
>
> PM2 adds automatic restart on crash, log management, and monitoring 
> on top.

**Q12. Explain Node.js `objectMode` streams with a database cursor 
→ CSV export example.**

```javascript
const { Transform } = require('stream');
const { pipeline } = require('stream/promises');

const dbCursor = db.query('SELECT * FROM orders').stream();
// objectMode readable (rows)

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

> Without `objectMode`: Transform would receive Buffer chunks, not row 
> objects — can't access `row.id`.

**Q13. What is the footgun with `process.nextTick` in recursive patterns?**

```javascript
function recursiveNextTick(n) {
  if (n === 0) return;
  process.nextTick(() => recursiveNextTick(n - 1));
}
recursiveNextTick(1000000);
```

> The nextTick queue is fully drained before the event loop advances. This 
> recursive pattern starves ALL I/O — HTTP requests, timers, file reads are 
> blocked until the million nextTick calls complete. This is why Node.js 
> docs say "prefer `setImmediate` over `process.nextTick` in recursive 
> patterns." `setImmediate` yields to the event loop between iterations; 
> `nextTick` does not.

**Q14. How does Node.js handle DNS resolution, and why does it matter for 
`http.request` performance?**

> `http.request` uses `dns.lookup()` by default, which goes through the 
> libuv thread pool (not the OS async DNS API). With 4 default threads and 
> multiple outgoing requests to different hosts, DNS lookups queue behind 
> each other and behind file I/O.
>
> **Symptoms**: Slow first requests, latency spikes under concurrent 
> outgoing calls.
>
> **Fixes**:
> 1. Set `UV_THREADPOOL_SIZE=16+`
> 2. Use `dns.resolve()` (actual async API) via custom `lookup` option 
>    in `http.agent`
> 3. Use a connection pool (http-agent with `keepAlive: true`) — reuses 
>    sockets, bypasses DNS for repeat connections

**Q15. What is `unhandledRejection` vs `uncaughtException` in Node.js? 
How should you handle each in production?**

> **`uncaughtException`**: A thrown error that propagated without a 
> `try/catch`. Process is in undefined state — log, flush metrics, 
> `process.exit(1)`. NEVER try to resume.
>
> **`unhandledRejection`**: A Promise that was rejected with no `.catch()` 
> handler. In Node 15+, this crashes the process by default (previously 
> just a warning).
>
> **Handle**:
> ```javascript
> process.on('uncaughtException', (err) => {
>   logger.fatal({ err }, 'Uncaught exception');
>   metrics.flush(() => process.exit(1));
> });
>
> process.on('unhandledRejection', (reason, promise) => {
>   logger.error({ reason }, 'Unhandled rejection');
>   // In production: treat same as uncaughtException — crash and let 
>   // PM2/K8s restart
>   process.exit(1);
> });
> ```
>
> **Key**: Always use `async/await` + `try/catch` or `.catch()` — don't 
> rely on global handlers as first defense.

### Study Resources

| # | Resource | URL |
|----|----------|-----|
| 1 | Node.js AsyncLocalStorage + async_hooks | https://nodejs.org/api/async_context.html |
| 2 | Node Clinic by NearForm | https://clinicjs.org/ |
| 3 | Node.js Design Patterns (Casciaro & Mammino) | https://www.nodejsdesignpatterns.com/ |

---

## 3. AWS

### Key Concepts to Master

#### Lambda

- **Cold start phases**: Download zip/image → start Firecracker micro-VM 
  → initialize runtime → run init code (outside handler)
- **Warm container**: Execution environment reused across invocations
  - Global scope persists (reuse DB connections, SDK clients)
- **Provisioned Concurrency**: Pre-warmed environments; eliminates cold 
  start; billed per hour
- **Reserved Concurrency**: Hard cap on function concurrency
  - `0` = function disabled (useful kill switch)
- **Concurrency limit**: 1000 per account per region by default
  - Request increase via Support
- **Lambda Layers**: Shared dependencies; up to 5 layers; max 250MB 
  unzipped total

#### S3

- **Strong consistency** since Dec 2020 (all operations: PUT, GET, LIST, 
  DELETE)
- **Storage classes**: Standard → Standard-IA (30-day minimum) 
  → One-Zone-IA → Glacier Instant → Glacier Flexible → Deep Archive
- **Multipart upload**: Required > 5GB, recommended > 100MB
  - Parts min 5MB except last
- **Pre-signed URLs**: Delegate time-limited access without exposing 
  credentials
  - `expires_in` parameter
- **Object Lock**: WORM (Write Once Read Many) compliance
  - Governance vs Compliance mode
- **S3 Transfer Acceleration**: Routes uploads through CloudFront edge 
  network

#### SQS

- **Standard**: At-least-once, best-effort ordering, nearly unlimited TPS
- **FIFO**: Exactly-once processing, strict ordering within message group
  - 300 TPS (3000 with batching)
- **Visibility timeout**: Time message is hidden after receipt
  - Set > Lambda max timeout + buffer
- **Long polling**: `WaitTimeSeconds=20`; reduces empty responses, saves 
  cost
- **DLQ**: Messages exceeding `maxReceiveCount` sent here
  - Monitor DLQ depth as alarm

#### SNS

- **Fan-out**: One publish → multiple SQS queues / Lambda functions 
  / HTTP endpoints / email
- **Message filtering**: Subscriber filter policy
  - `{"eventType": ["ORDER_CREATED"]}` — reduces Lambda invocations
- **FIFO SNS**: Works with FIFO SQS for ordered fan-out
  - Same group ID semantics

#### CloudWatch vs X-Ray

- **CloudWatch**: Metrics, logs, alarms, dashboards — aggregate view
- **X-Ray**: Distributed tracing — follows ONE request through Lambda 
  → API Gateway → DynamoDB → external calls
  - Shows per-segment timing; service map
- **X-Ray catches** what CloudWatch misses: Which specific downstream call 
  is slow, cold start vs handler execution breakdown, N+1 DB query 
  patterns in trace waterfall

#### EC2

- **Instance families**:
  - C (compute)
  - R (memory)
  - I (storage NVMe)
  - P/G (GPU)
  - T (burstable)
- **Placement Groups**:
  - Cluster (same AZ, low latency HPC)
  - Spread (max 7 instances/AZ, fault isolation)
  - Partition (Hadoop/Kafka, racks as partitions)
- **Spot Instances**: 2-minute termination notice
  - Use interruption handlers
  - Never for stateful single-instance workloads

#### API Gateway

- **REST API vs HTTP API**: HTTP API is 70% cheaper, lower latency, fewer 
  features (no request transformation, no usage plans)
- **WebSocket API**: Persistent connections; `connectionId` for targeting 
  specific clients
- **Throttling**: 10,000 RPS account limit, 5,000 burst
  - Per-stage/route overrides
- **Lambda Authorizer**: Custom JWT/token validation
  - Caches result by `authorizationToken` for TTL seconds

#### VPC

- **Public subnet**: Has 0.0.0.0/0 route to Internet Gateway
- **Private subnet**: No direct internet; NAT Gateway for outbound (charges 
  per GB processed)
- **Security Groups**: Stateful, instance-level (return traffic auto-allowed)
- **NACLs**: Stateless, subnet-level, evaluated in number order (lower 
  number = higher priority)
- **VPC Endpoints**:
  - Gateway (free, S3/DynamoDB)
  - Interface (priced, most other services)
  - Traffic stays in AWS network, bypasses NAT

### 15 Hard Interview Questions

**Q1. Lambda cold start: what are ALL the mitigation strategies and what 
are the tradeoffs of each?**

> 1. **Provisioned Concurrency**: Eliminates cold start completely; costs 
>    money per hour regardless of invocations
> 2. **Keep function small**: Smaller zip = faster download + 
>    initialization; tree-shake, use only required SDK v3 clients
> 3. **Minimize global init code**: Lazy-load heavy dependencies inside 
>    handler (trade-off: first invocation after warm-up has init cost)
> 4. **Scheduled warm-up ping**: Invoke every 5 min — anti-pattern, 
>    unreliable, wastes cost; avoid
> 5. **HTTP API over REST API**: Lighter runtime
> 6. **SnapStart** (Java/Python): Snapshots initialized state; 
>    sub-100ms cold start
> 7. **arm64 architecture**: 20% cheaper, often faster init than x86

**Q2. SQS visibility timeout = 30s. Lambda function takes 45s. Describe 
exactly what happens and the fix.**

> At T+0: Lambda receives message, message hidden.
> At T+30: Visibility timeout expires, message becomes visible again.
> A NEW Lambda invocation picks up the same message — concurrent duplicate 
> processing. Original Lambda is still running. Both complete successfully 
> → double processing.
>
> **Fix**: Set visibility timeout = (Lambda timeout × 6) per AWS 
> recommendation.
>
> Or use `ChangeMessageVisibility` API call within Lambda to extend 
> visibility timeout dynamically as work progresses.
>
> **Always**: Design consumers to be idempotent — duplicate processing 
> should be safe.

**Q3. SNS→Lambda direct vs SNS→SQS→Lambda fan-out. When does the direct 
pattern fail at scale?**

> **SNS→Lambda direct**: SNS invokes Lambda synchronously. If Lambda is 
> throttled (concurrency limit hit), SNS retries 2-3 times with exponential 
> backoff then **drops the message**. No persistence. Under traffic spikes: 
> lost messages.
>
> **SNS→SQS→Lambda**: SQS absorbs the spike (unlimited retention), Lambda 
> polls at its own pace via event source mapping. Concurrency controlled by 
> `batchSize` and `maxConcurrency`. Messages persist for up to 14 days.
>
> **Use direct** only for low-volume, non-critical notifications.
> **Use SQS buffer** for anything requiring guaranteed delivery.

**Q4. S3 event notifications vs EventBridge for S3: when do you choose 
each?**

> **S3 direct notifications**:
> - ~seconds latency
> - Limited event types (ObjectCreated, ObjectRemoved, Replication)
> - Single destination (SQS or SNS or Lambda)
>
> **EventBridge**:
> - Rich content-based filtering on S3 metadata (prefix, suffix, size)
> - Fan-out to 5+ targets simultaneously
> - Cross-account routing
> - Event archiving + replay capability (critical for debugging)
> - ~1s additional latency
>
> **Choose EventBridge when**: Multiple downstream consumers, complex 
> filtering (e.g., only `.jpg` files > 1MB in `uploads/` prefix), 
> cross-account, replay needed.
>
> **Choose direct for**: Single consumer, latency-sensitive pipeline, 
> simple triggers.

**Q5. You set Lambda reserved concurrency to 0 by mistake. What happens 
and how do you detect it?**

> Every invocation returns `TooManyRequestsException` (HTTP 429) immediately. 
> The function is effectively disabled. API Gateway callers receive 429 
> or 502 depending on integration.
>
> **Detection**: CloudWatch Lambda → `Throttles` metric spikes to 100%;
> `Errors` from API Gateway. If it's an async trigger (SQS, SNS), messages 
> pile up in SQS DLQ or SNS retries exhaust.
>
> **Fix**: Set reserved concurrency to desired value (or remove limit 
> entirely by setting to `null`).
>
> **Lesson**: Document this as a valid emergency kill switch but gate it 
> with IAM permissions.

**Q6. Explain the NAT Gateway cost footgun and how to fix it with VPC 
Endpoints.**

> **NAT Gateway charges**: $0.045/hour + $0.045/GB processed.
>
> A Lambda in a private VPC calling S3/DynamoDB routes through NAT Gateway 
> by default — every API call is billable data transfer. At scale (millions 
> of Lambda invocations calling S3): hundreds of dollars/month in NAT costs.
>
> **Fix**:
> 1. S3 Gateway VPC Endpoint: free, add route to route table, traffic stays 
>    in AWS network
> 2. DynamoDB Gateway VPC Endpoint: also free
> 3. For other services (SSM, Secrets Manager, STS): Interface VPC 
>    Endpoints (~$7.30/month/AZ — compare against NAT costs)
>
> **Most teams** save 60-80% on data transfer costs after adding 
> S3/DynamoDB endpoints.

**Q7. API Gateway 29-second timeout. You have a slow report generation 
endpoint. Design the async solution.**

> Never make the user wait 29s.
>
> **Pattern**:
> 1. POST `/reports` → Lambda queues job to SQS → returns `202 Accepted` 
>    + `{ jobId: "uuid" }`
> 2. SQS → Report Worker Lambda (can run up to 15 min)
> 3. Client polls GET `/reports/{jobId}` → returns `{ status: "processing" }` 
>    until done, then `{ status: "complete", url: "s3-presigned-url" }`
>
> Or: Step Functions callback pattern — worker calls `SendTaskSuccess` when 
> done, Step Functions notifies API.
>
> Or: WebSocket API Gateway — push completion event to client.
>
> **Rule**: Any operation > 5s should be async with status polling or push 
> notification.

**Q8. Explain how CloudWatch Alarms work with Auto Scaling and what 
"cooldown period" prevents.**

> **CloudWatch Alarm**: Metric breaches threshold → transitions to ALARM 
> state → triggers SNS/action (scale out policy).
>
> **Auto Scaling group** receives scale-out action → launches new instances.
>
> **Cooldown period** (default 300s): After a scaling activity, Auto 
> Scaling ignores further alarms for this duration.
>
> **Prevents**: Thrashing — new instances haven't started serving traffic 
> yet, metric still high, triggers another scale-out → over-provisioning.
>
> **Instance warmup**: Separate setting — new instance's metrics excluded 
> from aggregate until warmup period passes.
>
> **Target tracking policies** manage cooldown automatically; step scaling 
> requires manual tuning.

**Q9. Describe the SQS FIFO `MessageGroupId` pattern for parallel ordered 
processing.**

> FIFO queues provide ordering within a message group, parallelism across 
> groups.
>
> **Example**: Ride-sharing app where driver location updates for RIDE-123 
> must be processed in sequence, but RIDE-456 and RIDE-789 can process in 
> parallel.
>
> **Set** `MessageGroupId = rideId`.
>
> SQS FIFO ensures all messages for RIDE-123 go to a single Lambda 
> invocation at a time (per group serialization) while different rides 
> process concurrently. Without group IDs (all messages in one group): fully 
> sequential processing — FIFO becomes a bottleneck.

**Q10. X-Ray vs CloudWatch Logs Insights: what does X-Ray catch that you 
cannot see in logs?**

> **Logs**: What your code explicitly logs — application-level events.
>
> **X-Ray**: Distributed trace of one request's journey across ALL services 
> — Lambda function → API Gateway → two DynamoDB calls → one external HTTP 
> call.
>
> **Shows**:
> - Exact duration of EACH segment
> - Which specific DynamoDB query took 800ms (logs only show total handler 
>   time)
> - Cold start duration isolated from handler execution
> - Downstream service dependencies mapped visually
> - Error correlation across service boundaries
>
> **Logs Insights**: Powerful queries but only within one log group at a 
> time.
>
> **X-Ray** shows the full request waterfall across log groups.

**Q11. Lambda event source mapping for SQS: explain `batchSize`, 
`batchWindow`, and `maxConcurrency`.**

> - **`batchSize`**: Max messages per Lambda invocation (1-10,000 for 
>   standard, 1-10 for FIFO)
> - **`batchWindow`**: Wait up to N seconds to fill a batch before invoking 
>   Lambda (reduces Lambda invocations, increases latency)
> - **`maxConcurrency`**: Max simultaneous Lambda invocations from this SQS 
>   trigger (new in 2023)
>
> Without `maxConcurrency`: SQS scales Lambda to 1000 concurrent invocations 
> → may overwhelm downstream DB.
>
> **Set** `maxConcurrency` to protect downstream services.
>
> If any message in a batch fails: entire batch returned to queue by 
> default. Use `reportBatchItemFailures` to return only failed messages.

**Q12. Describe how EC2 Spot Instance interruptions work and design a 
fault-tolerant batch processing architecture.**

> **Spot Instance**: 2-minute interruption notice via EC2 metadata endpoint 
> (poll `http://169.254.169.254/latest/meta-data/spot/termination-time`) 
> and EventBridge event.
>
> **Architecture**: SQS queue of work items → Spot Instance fleet (Auto 
> Scaling with `SpotAllocationStrategy: capacity-optimized`) → workers 
> poll SQS.
>
> **On interruption notice**:
> - Worker stops polling
> - Finishes current item (< 2 min work units)
> - Deletes message from SQS
> - Gracefully exits
>
> **For long processing**: Checkpoint progress to S3/DynamoDB, extend SQS 
> visibility timeout.
>
> **Use** `mixed instances policy` with on-demand base capacity for 
> critical path.

**Q13. VPC Security Group vs NACL: when does the stateful vs stateless 
difference cause a real bug?**

> **Real bug scenario**: You allow inbound on port 443 in NACL. User makes 
> HTTPS request. Server responds on an ephemeral port (1024-65535) — the 
> return traffic.
>
> - **Security Group**: Stateful — return traffic automatically allowed
> - **NACL**: Stateless — outbound ephemeral port range (1024-65535) is NOT 
>   automatically allowed. Response packets are dropped.
>
> **Fix**: Add NACL outbound rule allowing 1024-65535.
>
> **Common mistake**: Developers set NACL inbound correctly but forget 
> outbound ephemeral ports.
>
> **Most teams** use Security Groups exclusively and set NACLs to 
> allow-all (layered defense via SGs only).

**Q14. Describe Lambda Destinations and how they differ from DLQs.**

> **DLQ**: Only for failed asynchronous invocations after all retries 
> exhausted.
>
> **Lambda Destinations**: Configurable for BOTH success and failure of 
> async invocations. Can route to SQS, SNS, Lambda, or EventBridge for 
> success (e.g., trigger next step on success) AND failure (e.g., alert 
> team on failure).
>
> - **DLQ** receives only the original event
> - **Destinations** receive: original event + function response/error 
>   + request context + invocation record
>
> **Destinations are more powerful**: You get success routing, richer 
> failure context, and EventBridge routing for complex workflows. DLQ is 
> simpler for basic failure capture.

**Q15. You're building a multi-region active-active architecture. What AWS 
services have cross-region complexity and how do you handle them?**

> - **DynamoDB Global Tables**: Automatic multi-master replication; 
>   last-write-wins conflict resolution; eventual consistency across regions
> - **S3 Cross-Region Replication**: Async, not real-time; use S3 
>   Multi-Region Access Points for intelligent routing
> - **Route 53**: Latency-based or geolocation routing + health checks for 
>   failover
> - **API Gateway**: Deploy to each region, Route 53 routes to nearest
> - **Lambda**: Deploy same function in each region, test for regional env 
>   var differences
>
> **The hardest part**: Database write conflicts (DynamoDB Global Tables 
> last-writer-wins may not suit all apps), data residency compliance (some 
> data can't leave region), and cost (CRR, Global Tables replication all 
> cost extra).

### Study Resources

| # | Resource | URL |
|----|----------|-----|
| 1 | AWS Well-Architected Framework | https://aws.amazon.com/architecture/well-architected/ |
| 2 | AWS re:Invent Talks (YouTube) | https://www.youtube.com/@AWSEventsChannel |
| 3 | AWS Architecture Blog | https://aws.amazon.com/blogs/architecture/ |

---

## 4. CI/CD, Docker, GitLab CI & Jenkins

### Key Concepts to Master

#### Docker Internals

- **Image** = read-only stack of layers
- **Container** = image + thin writable layer
- **Each `RUN`, `COPY`, `ADD`** = one layer; layer cached if content + all 
  preceding layers unchanged
- **Build context**: Entire directory sent to Docker daemon
  - `.dockerignore` is critical for performance
- **Multi-stage builds**: `FROM node:20 AS builder` ... `FROM 
  gcr.io/distroless/nodejs20` ... `COPY --from=builder /app/dist ./dist`
- **Distroless images** (Google): No shell, no package manager, no OS 
  utilities
  - Minimal attack surface; `gcr.io/distroless/nodejs20-debian12`
- **BuildKit**: Parallel layer building, `--mount=type=cache` (persist 
  npm/pip cache across builds), `--mount=type=secret` (build-time secrets 
  not in layers)
- **Security**: Run as non-root (`USER node`), read-only filesystem 
  (`--read-only`), no secrets in ENV or layers, scan with `trivy` or 
  `grype`

#### GitLab CI/CD

- **`.gitlab-ci.yml`**: Defines `stages`, `jobs`, `rules`, `needs`, 
  `cache`, `artifacts`, `environment`
- **`stages`**: Sequential execution blocks; all jobs in stage N complete 
  before stage N+1 starts
- **`needs`**: DAG — job starts when its specific deps finish, regardless 
  of stage completion
- **`rules`**: Conditional job execution — `if`, `changes`, `exists`
  - Evaluated in order (first match wins)
  - Preferred over deprecated `only/except`
- **`cache`**: Persisted between pipeline runs; stored in runner cache
  - Keyed by `key:` expression
- **`artifacts`**: Files passed between jobs in same pipeline; uploaded to 
  GitLab, downloaded by downstream jobs
- **Environments**: Deployment tracking with approval gates 
  (`when: manual`) for production
- **GitLab Agent for Kubernetes**: Pull-based GitOps; cluster pulls from 
  GitLab, not push-based deployment

#### Jenkins

- **Declarative pipeline** (`pipeline {}` block): Structured, validated by 
  Jenkins before running, limited to declarative syntax
- **Scripted pipeline** (`node {}` block): Full Groovy, maximum 
  flexibility, harder to maintain
- **Shared Libraries**: Reusable Groovy code in separate Git repo
  - `@Library('lib@v1.0') _`
  - `vars/` for global functions
- **Multibranch Pipeline**: Auto-discovers branches/PRs with Jenkinsfiles
  - Creates pipeline per branch
- **Still chosen when**: On-prem with no internet access, complex Groovy 
  orchestration needs, existing investment in plugins

#### Deployment Strategies

- **Blue-Green**: Two identical environments; traffic switch at load 
  balancer/DNS; instant rollback (point back); requires 2x resources
- **Canary**: Gradual traffic shift (1% → 5% → 25% → 100%); monitor error 
  rates + latency at each step; automated rollback trigger
- **Rolling**: Replace instances one-at-a-time; less resource-intensive; 
  can't instantly rollback all instances
- **Feature Flags**: Deploy code without activating it; decouple deploy 
  from release; LaunchDarkly, Unleash, AWS AppConfig
- **GitOps**: Git is single source of truth for infrastructure state
  - ArgoCD/Flux watches Git, applies changes to cluster
  - Drift detection

### 15 Hard Interview Questions

**Q1. Explain why this Dockerfile is inefficient and rewrite it correctly 
for a production Node.js app.**

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

> **Problems**:
> 1. `COPY . .` before `npm install` — any source code change busts the 
>    npm install cache
> 2. `npm install` includes devDependencies
> 3. Single stage — dev tools in production image
> 4. Running as root
>
> **Fixed**:
> ```dockerfile
> # Stage 1: Build
> FROM node:20-alpine AS builder
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci --include=dev
> COPY . .
> RUN npm run build
>
> # Stage 2: Production
> FROM gcr.io/distroless/nodejs20-debian12
> WORKDIR /app
> COPY --from=builder /app/dist ./dist
> COPY --from=builder /app/node_modules ./node_modules
> # Distroless runs as nonroot by default (UID 65532)
> EXPOSE 3000
> CMD ["dist/server.js"]
> ```

**Q2. What is Docker BuildKit `--mount=type=secret` and why is it critical 
for private npm registries?**

> Without it: `RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" 
> > .npmrc && npm ci` — the `.npmrc` file with the token is baked into 
> the image layer permanently. Anyone with image access can extract the 
> token.
>
> **With BuildKit secret mount**:
> ```dockerfile
> # syntax=docker/dockerfile:1
> FROM node:20-alpine AS builder
> WORKDIR /app
> COPY package*.json ./
> RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
> ```
> ```bash
> docker build --secret id=npmrc,src=.npmrc .
> ```
>
> The `.npmrc` exists only during that `RUN` step — never in any layer.
> `docker history` shows nothing. Token is safe.

**Q3. Blue-green vs canary: for a database schema migration that adds a 
NOT NULL column, which strategy works and which breaks?**

> **Blue-green breaks**: v2 (new schema) is deployed and traffic switches. 
> If rollback needed, v1 code can't read the NOT NULL column that v2 added 
> (column doesn't exist in v1's expected schema).
>
> **Canary breaks similarly**.
>
> **Correct approach**: **Expand-Contract (Parallel Change) pattern**:
> 1. Expand: Add column as nullable, deploy v1 (doesn't use it)
> 2. Backfill: Populate the column
> 3. Add constraint: Add NOT NULL with DEFAULT
> 4. Deploy v2: Starts writing to the column
> 5. Contract: Remove old code paths
>
> Both strategies are safe ONLY when migrations are backward-compatible.

**Q4. Production is down. Walk me through a rollback in under 5 minutes 
using GitLab CI.**

> **Pre-requisite**: This must be designed ahead of time.
>
> 1. GitLab Environments page → click "Rollback" on previous deployment 
>    → re-runs previous deployment job (30s if images are cached)
> 2. Feature flag: Toggle off in LaunchDarkly — instant, zero deploy
> 3. Blue-green: Redirect load balancer to blue environment (< 30s DNS or 
>    ALB target group swap)
> 4. Helm: `helm rollback my-app 1` → previous chart revision (< 1min on K8s)
> 5. Manual GitLab pipeline trigger: Create a `rollback` job with 
>    `when: manual` that runs `helm rollback` or swaps ALB target groups
>
> **Key lesson**: Rollback is a button press, practiced monthly in staging.

**Q5. Explain GitLab CI DAG with `needs:` and draw an example showing why 
it's faster.**

> **Stage-based (without `needs`)**: All of `test-unit`, 
> `test-integration`, `test-e2e` in Stage 2 must complete before 
> `build-docker` in Stage 3 starts.
>
> **DAG with `needs`**: `build-docker` only needs `test-unit` to pass.
> It starts immediately after `test-unit` finishes, while 
> `test-integration` and `test-e2e` are still running.
> `deploy-staging` needs both `build-docker` AND `test-e2e`.
>
> **Time saving**: If `test-unit` = 2min, `test-e2e` = 10min, 
> `build-docker` = 3min — DAG pipeline: 13min total vs 15min stage-based.
> Real pipelines with 20+ jobs: 30-50% time savings.

**Q6. How do you handle secrets in Docker Compose (development) vs 
Kubernetes (production) without changing application code?**

> Application code always reads from files or environment variables — 
> abstract the source.
>
> **Development (Docker Compose)**:
> - `secrets:` block reads from local files
> - Mounted at `/run/secrets/db_password`
>
> **Production (K8s)**:
> - External Secrets Operator or Vault Agent Sidecar fetches from Vault/AWS 
>   Secrets Manager
> - Creates K8s Secret, mounted as file at same path
> - Or: K8s Secret (base64 encoded, encrypted at rest if KMS configured) as 
>   env var or volume mount
>
> **Application code**: `fs.readFileSync('/run/secrets/db_password')` — 
> identical in both environments.
>
> **Never**: Hardcode, commit, or build secrets into images.

**Q7. Jenkins shared library: how do you version it and prevent a library 
update from breaking 50 pipelines?**

> **Library referenced with tag/branch**: `@Library('pipeline-lib@v2.1.0') 
> _`
>
> All 50 pipelines reference specific version tags. Library changes: create 
> new tag → pipelines only upgrade when they explicitly change the 
> `@Library` version.
>
> **For breaking changes**: Semantic versioning, changelog in library repo, 
> Slack notification to teams.
>
> **Safe rollout**:
> 1. Create `v3.0.0` tag
> 2. Update 1 non-critical pipeline
> 3. Monitor
> 4. Gradual migration
>
> **CI for the library itself**: Run `jenkins-cli` to validate Jenkinsfiles 
> against the library using the `lintJenkinsfile` command.

**Q8. Your Docker container fails in CI but works locally. Systematic debug 
process.**

> 1. **Pin base image**: Local might have `node:20.11.1`, CI pulls 
>    `node:20` (latest) = different version
>    - Fix: `node:20.11.1-alpine`
>
> 2. **Platform mismatch**: M1/M2 Mac builds `linux/arm64`; CI is 
>    `linux/amd64`
>    - Add `--platform linux/amd64` to build
>
> 3. **Build context difference**: `.dockerignore` might exclude files 
>    locally that CI includes (or vice versa)
>    - Print what's in context
>
> 4. **env vars**: CI has different values
>    - Print non-secret env vars in failing step
>
> 5. **File permissions**: CI may run as different UID
>    - Check `RUN ls -la`
>
> 6. **Reproduce CI locally**: `docker run -e CI=true --platform 
>    linux/amd64 your-image sh`

**Q9. What does `gitlab-ci.yml` `cache` not restore, and how do you debug 
it?**

> **Cache not restored when**:
> 1. Different runner instance (not same machine) and cache is filesystem-based (not S3)
> 2. `key:` expression changed (branch name, lockfile hash changed)
> 3. Cache expired (default 30 days)
> 4. First run on a new branch (no cache for this key yet)
> 5. Cache upload failed on previous run (job killed)
>
> **Debug**: Add `echo "Cache key: $CI_CACHE_KEY"` to job, check GitLab 
> job log for "Checking cache for..." message.
>
> **Fix for distributed runners**: Configure `[runners.cache]` in 
> `config.toml` with S3 backend — shared cache across all runner instances.

**Q10. Explain zero-downtime deployment for a Node.js app with active 
WebSocket connections on Kubernetes.**

> **WebSockets are long-lived** — normal rolling deploy drops them.
>
> **Strategy**:
> 1. **Client-side**: Always implement reconnection with exponential 
>    backoff + jitter (this is non-negotiable regardless of strategy)
> 2. **K8s Pod `preStop` lifecycle hook**: `sleep 15` before `SIGTERM` 
>    — allows load balancer to drain new connections away
> 3. **`terminationGracePeriodSeconds: 60`**: Give existing connections 
>    60s to complete naturally
> 4. **App-level drain**: On SIGTERM, stop accepting new WS upgrades, 
>    allow existing connections to idle-timeout
> 5. **Decouple state**: Use Redis pub/sub for WS state → any pod handles 
>    any connection → reconnection is transparent to the user

**Q11. Implement a CI security scanning step that blocks deployment if 
critical vulnerabilities are found.**

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

> `--exit-code 1`: Trivy exits with code 1 on findings → GitLab marks job 
> as failed → blocks downstream deploy job.
>
> `--ignore-unfixed`: Skip vulns with no available fix (reduces noise).
>
> Add `--skip-dirs` for known false positives.

**Q12. What's the difference between `COPY` and `ADD` in Dockerfile? When 
is `ADD` dangerous?**

> - **`COPY`**: Copies files/dirs from build context to image
>   - Explicit, predictable
> - **`ADD`**: Does everything `COPY` does PLUS: auto-extracts tar 
>   archives, fetches remote URLs
>
> **Dangerous**: `ADD https://example.com/file.tar.gz /app/`
> - Fetches at build time
> - Can't be cached properly
> - URL could change or be hijacked
> - Introduces supply chain risk
>
> **Best practice**: Always use `COPY` unless you explicitly need tar 
> extraction. For remote files: `curl` in `RUN` with checksum verification.
>
> Docker official docs recommend preferring `COPY` over `ADD`.

**Q13. GitLab CI `rules` vs `only/except`: concrete example showing where 
`only/except` fails and `rules` works.**

> **Requirement**: Run job on merge request OR on `main` branch push, but 
> only if `src/` files changed.
>
> ```yaml
> # only/except — cannot express this:
> job:
>   only: [merge_requests, main]
>   # No way to combine with 'changes' condition in only/except
>
> # rules — expressive:
> job:
>   rules:
>     - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
>       changes: [src/**/*]
>     - if: '$CI_COMMIT_BRANCH == "main"'
>       changes: [src/**/*]
>     - when: never  # skip all other cases
> ```
>
> `only/except` can't combine branch/event conditions with `changes` (file 
> changes) in an AND relationship. `rules` can express complex boolean 
> logic. This is why `only/except` was deprecated.

**Q14. How do you implement GitOps with GitLab and ArgoCD for a Kubernetes 
deployment?**

> 1. Separate `infra` repo from `app` repo
> 2. App CI pipeline builds Docker image, updates `infra` repo's Helm 
>    `values.yaml` with new image tag (git commit via CI job)
> 3. ArgoCD watches `infra` repo → detects git diff → automatically applies 
>    to K8s cluster (or requires manual sync for production)
> 4. ArgoCD shows drift: if someone manually changes K8s resource, ArgoCD 
>    shows it as "out of sync" with Git
> 5. Rollback = `git revert` the image tag commit → ArgoCD applies 
>    previous state
>
> **Benefits**:
> - Audit trail (Git blame shows who changed what when)
> - PR reviews for infra changes
> - Disaster recovery (recreate entire cluster from Git state)

**Q15. What is a multi-arch Docker build and why is it required for teams 
with M-series Macs deploying to AWS?**

> **M1/M2/M3 Macs** are `arm64`. **AWS EC2 default instances** are 
> `x86_64` (amd64).
>
> A Docker image built on Mac is `arm64` — it will fail or run via slow 
> emulation on `amd64` EC2.
>
> **Fix**: Multi-arch build with Docker Buildx:
> ```bash
> docker buildx create --use
> docker buildx build \
>   --platform linux/amd64,linux/arm64 \
>   --push \
>   -t myrepo/myapp:latest .
> ```
>
> Creates a manifest list — `docker pull` on any architecture gets the 
> right image.
>
> **CI should** always build for `linux/amd64` explicitly.
>
> **Bonus**: AWS Graviton (arm64) instances are 20% cheaper — worth 
> building multi-arch to support both.

### Study Resources

| # | Resource | URL |
|----|----------|-----|
| 1 | OWASP Docker Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html |
| 2 | GitLab CI Pipeline Efficiency Docs | https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html |
| 3 | Google SRE Book — Release Engineering | https://sre.google/sre-book/release-engineering/ |

---

## 5. DSA / LeetCode — 10-Day Sprint

### Is Blind 75 Enough?

| Company | Verdict | Notes |
|---------|---------|-------|
| Adobe (CS2) | Blind 75 + | Medium-hard focus; problem-solving process matters more than hard grinding |
| Walmart Labs | Grind 169 | Medium-heavy; some hard DP/graph; data structures + design expected |
| Flipkart / MakeMyTrip | Blind 75 ✓ | LeetCode Medium focus; articulate time/space complexity clearly |
| Uber | NeetCode 150 | Graph, advanced trees, system design-adjacent coding; check Uber company tag |
| Cars24 / Nagarro | Blind 75 ✓ | Emphasis on clean code and approach clarity |

**Recommendation**: Master Blind 75 first (solve each in < 20 min cold).
Then fill gaps with NeetCode 150 additions, prioritizing graphs and DP.
Use LeetCode company tags for Uber and Walmart for targeted prep.

### 10-Day DSA Sprint Plan

| Day | Topic | Goal |
|-----|-------|------|
| 1 | Arrays + Two Pointers | Learn two-pointer template |
| 2 | Sliding Window | Learn SW template — shrink/expand |
| 3 | Trees — DFS/BFS | Recursive + iterative for both |
| 4 | Graphs | BFS/DFS templates + topo sort |
| 5 | Dynamic Programming I | Identify subproblem → recurrence |
| 6 | Dynamic Programming II | Bottom-up table traversal direction |
| 7 | Heaps + Binary Search | Heap push/pop complexity; BS template |
| 8 | Stack + Monotonic Stack | Monotonic stack pattern |
| 9 | Linked Lists + Tries | Fast/slow pointers; trie ops |
| 10 | Mixed Hard + Company Tags | Simulate real interview timing |

**Time budget per day**: 2-3 problems per hour; spend 25 min trying before 
looking at hint. Always explain your approach out loud before coding.

---

## 6. System Design — HLD + LLD

### Key Concepts to Master (Non-Negotiable)

#### CAP Theorem

- You cannot have all three: Consistency, Availability, Partition 
  Tolerance
- Network partitions are unavoidable in distributed systems → real choice 
  is CP vs AP
- **CP**: Consistent but may be unavailable during partition (HBase, 
  ZooKeeper, Etcd)
- **AP**: Available but may serve stale data (Cassandra, CouchDB, DynamoDB 
  with eventual consistency)
- **PACELC extension**: Even without partition, tradeoff between Latency 
  (L) and Consistency (C)

#### Consistent Hashing

- **Hash ring**: Nodes placed by hash(nodeId); keys assigned to next 
  clockwise node
- **Adding/removing nodes**: Only `k/n` keys remapped (vs 100% in modulo 
  hashing)
- **Virtual nodes**: Each physical node has multiple positions on ring for 
  even distribution
- **Used by**: DynamoDB, Cassandra, Redis Cluster, Memcached, CDN edge 
  routing

#### Database Sharding

- **Range-based**: Shard by ID range (1M-2M, 2M-3M) — simple range 
  queries, hot shard risk
- **Hash-based**: `hash(userId) % numShards` — even distribution, range 
  queries suffer
- **Directory-based**: Lookup table (shard metadata service) — flexible, 
  SPOF risk
- **Key challenge**: Cross-shard JOINs don't exist; choose shard key to 
  keep related data together
- **Hotspot**: user_id as shard key → celebrity user causes hot shard
  - Solution: Add random salt

#### Caching Patterns

- **Cache-aside (lazy loading)**: App checks cache on read; miss → fetch 
  DB → populate cache
- **Write-through**: Write to cache AND DB synchronously; no stale reads; 
  more write latency
- **Write-behind (write-back)**: Write to cache; async flush to DB; risk 
  of data loss on cache failure
- **Cache invalidation**: TTL (simplest, accepts staleness), event-driven 
  invalidation on write, cache-busting with versioned keys

#### Redis Patterns

- **String**: Simple key-value, counters (`INCR`), rate limiting
- **Sorted Set**: Leaderboard (`ZADD`, `ZRANGE`), time-series events
- **Hash**: User session storage, partial updates
- **Pub/Sub**: Real-time notifications, presence, WebSocket fan-out
- **Streams**: Append-only log; consumer groups; use over Pub/Sub when 
  message durability needed

---

## 7. Database Design — PostgreSQL / SQL Server

### Key Concepts to Master

#### Query Optimization

- **`EXPLAIN ANALYZE`**: Actual execution plan with real timing
  - Use `EXPLAIN (ANALYZE, BUFFERS)` to see shared hits vs reads
- **Seq Scan vs Index Scan vs Index-Only Scan**: Planner chooses based on 
  cost model + statistics
  - Low-selectivity queries → seq scan is cheaper (costs rows × page_cost)
- **Join strategies**: Nested Loop (small tables, indexed inner), Hash Join 
  (large unsorted), Merge Join (pre-sorted inputs)
- **`pg_stat_statements`**: Tracks slow queries in production
  - `mean_exec_time` + `calls` = identify N+1 patterns
- **Stale statistics → wrong plan**: Run `ANALYZE table_name` to update
  - `autovacuum` does this automatically

#### Index Strategies

- **B-Tree** (default): Equality, range, `LIKE 'prefix%'`, `ORDER BY`, 
  `BETWEEN`
- **Hash**: Equality only; slightly faster than B-Tree for pure equality
  - PostgreSQL WAL-logged since 10
- **Partial Index**: `CREATE INDEX ON orders(user_id) WHERE status = 
  'active'`
  - Smaller and faster for filtered queries
- **Covering Index**: `CREATE INDEX ON orders(user_id) INCLUDE (status, 
  total, created_at)`
  - Index-only scan (no heap fetch)
- **Composite Index**: Column order matters: `(a, b, c)` supports queries 
  on `a`, `(a, b)`, `(a, b, c)`
  - NOT `b` alone
  - Equality columns first, range column last
- **GIN Index**: For JSONB (`@>`, `?` operators), array columns, 
  full-text search (`tsvector`)
- **Expression Index**: `CREATE INDEX ON users(lower(email))`
  - Enables case-insensitive index lookups

#### Advanced SQL

- **Window functions**: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, 
  `LEAD()`, `SUM() OVER (...)`
  - Evaluated after `WHERE`/`GROUP BY`, before `ORDER BY`/`LIMIT`
- **CTEs**: Optimization fence in PG < 12 (always materialized)
  - Inlined by default in PG 12+
  - Use `MATERIALIZED` keyword to force
- **Lateral joins**: Subquery references outer query columns
  - Returns multiple rows; essential for "top N per group"
- **JSONB**: Binary storage, GIN-indexed, supports operators `@>` 
  (contains), `?` (has key), `->>` (extract text)
- **Partitioning**: range/list/hash; partition pruning
  - Use for time-series (monthly partitions on `created_at`)

#### PostgreSQL Internals

- **MVCC**: Every UPDATE creates new row version
  - Old version marked dead; `VACUUM` reclaims space
- **WAL (Write-Ahead Log)**: All changes written to WAL before data files
  - Enables crash recovery, replication, PITR
- **Autovacuum**: Monitor `pg_stat_user_tables.n_dead_tup`
  - Tune `autovacuum_vacuum_scale_factor` for hot tables
- **Connection pooling**: PostgreSQL spawns process per connection
  - PgBouncer pools; transaction mode incompatible with session-scoped 
    prepared statements

---

## Resources Index

### React / Next.js

| Resource | URL |
|----------|-----|
| React Fiber Architecture (Andrew Clark) | https://github.com/acdlite/react-fiber-architecture |
| React 18 Working Group Discussions | https://github.com/reactwg/react-18/discussions |
| Next.js App Router (Vercel Blog) | https://nextjs.org/blog/next-13-4 |
| React.dev Official Docs | https://react.dev/ |

### Node.js / Express

| Resource | URL |
|----------|-----|
| Node.js AsyncLocalStorage Docs | https://nodejs.org/api/async_context.html |
| Node.js Streams Official Docs | https://nodejs.org/api/stream.html |
| Node Clinic (NearForm) | https://clinicjs.org/ |
| Node.js Design Patterns Book | https://www.nodejsdesignpatterns.com/ |

### AWS

| Resource | URL |
|----------|-----|
| AWS Well-Architected Framework | https://aws.amazon.com/architecture/well-architected/ |
| AWS re:Invent Talks (YouTube) | https://www.youtube.com/@AWSEventsChannel |
| AWS Architecture Blog | https://aws.amazon.com/blogs/architecture/ |

### CI/CD / Docker

| Resource | URL |
|----------|-----|
| OWASP Docker Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html |
| GitLab CI Pipeline Efficiency | https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html |
| Google SRE Book — Release Engineering | https://sre.google/sre-book/release-engineering/ |

### DSA

| Resource | URL |
|----------|-----|
| NeetCode.io | https://neetcode.io/ |
| Competitive Programmer's Handbook (free PDF) | https://cses.fi/book/book.pdf |
| LeetCode Company Tags | https://leetcode.com/company/ |

### System Design

| Resource | URL |
|----------|-----|
| Designing Data-Intensive Applications | https://dataintensive.net/ |
| High Scalability Blog | http://highscalability.com/ |
| System Design Primer (GitHub) | https://github.com/donnemartin/system-design-primer |
| ByteByteGo | https://bytebytego.com/ |

### Database

| Resource | URL |
|----------|-----|
| Use the Index, Luke | https://use-the-index-luke.com/ |
| PostgreSQL Performance Tips (official) | https://www.postgresql.org/docs/current/performance-tips.html |
| Cybertec PostgreSQL Blog | https://www.cybertec-postgresql.com/en/blog/ |

---

**Date: June 15, 2026 · Target: 10-Day Sprint**

*Save this file as `interview-prep.md` in your Git repo.*
