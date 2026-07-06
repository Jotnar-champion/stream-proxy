# Apollo/GraphQL Lead Engineer — Interview Study Guide

*Target: Wednesday. Read in order 1→7, then drill the 50 Q&A. Time-box: sections 1–3 are where "lead-level" is judged hardest since you have real Apollo history — expect the deepest cross-questions there.*

---

## 0. Folder Structure — Next.js + Node (TS) + Apollo Server/Client + Real-time Notifications

```
crm-platform/
├── apps/
│   ├── web/                          # Next.js (App Router) - TypeScript
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── providers.tsx     # ApolloProvider wrapper (client component)
│   │   │   │   ├── (dashboard)/
│   │   │   │   │   ├── notifications/
│   │   │   │   │   │   ├── page.tsx           # Server Component (SSR fetch)
│   │   │   │   │   │   └── NotificationBell.tsx # Client Component (subscription)
│   │   │   │   │   └── leads/page.tsx
│   │   │   │   └── api/
│   │   │   │       └── graphql-proxy/route.ts  # optional BFF proxy (auth cookie -> Bearer)
│   │   │   ├── lib/
│   │   │   │   ├── apollo/
│   │   │   │   │   ├── apolloClient.ts         # browser singleton
│   │   │   │   │   ├── apolloServerClient.ts   # per-request RSC client
│   │   │   │   │   ├── cacheTypePolicies.ts
│   │   │   │   │   └── links.ts                # httpLink, wsLink, authLink, errorLink, splitLink
│   │   │   │   └── auth/
│   │   │   │       └── getToken.ts
│   │   │   ├── graphql/
│   │   │   │   ├── operations/
│   │   │   │   │   ├── notifications.graphql
│   │   │   │   │   └── leads.graphql
│   │   │   │   └── generated/                  # graphql-codegen output (typed hooks)
│   │   │   └── components/
│   │   ├── codegen.ts
│   │   ├── next.config.js
│   │   └── package.json
│   │
│   └── api/                          # Apollo Server (Node) - TypeScript
│       ├── src/
│       │   ├── index.ts                        # HTTP + WS server bootstrap
│       │   ├── schema/
│       │   │   ├── typeDefs/
│       │   │   │   ├── notification.graphql
│       │   │   │   ├── lead.graphql
│       │   │   │   └── directives/auth.graphql # @auth, @hasRole directive defs
│       │   │   ├── resolvers/
│       │   │   │   ├── notification.resolvers.ts
│       │   │   │   ├── lead.resolvers.ts
│       │   │   │   └── index.ts
│       │   │   └── directives/
│       │   │       └── authDirective.ts        # SchemaDirectiveVisitor/mapSchema impl
│       │   ├── context/
│       │   │   └── createContext.ts            # JWT verify, dataloaders, user injection
│       │   ├── datasources/
│       │   │   ├── loaders/
│       │   │   │   ├── userLoader.ts           # DataLoader batch fns
│       │   │   │   └── leadLoader.ts
│       │   │   └── repositories/
│       │   │       ├── notificationRepository.ts  # DB access, no GraphQL types
│       │   │       └── leadRepository.ts
│       │   ├── services/
│       │   │   ├── notificationService.ts      # business logic, calls repo + pubsub.publish
│       │   │   └── leadService.ts
│       │   ├── pubsub/
│       │   │   ├── pubsub.ts                   # RedisPubSub instance (prod) / PubSub (dev)
│       │   │   └── events.ts                   # NOTIFICATION_ADDED etc. constants
│       │   ├── middleware/
│       │   │   ├── authMiddleware.ts           # verify JWT -> req.user
│       │   │   ├── rateLimiter.ts
│       │   │   └── depthLimit.ts / complexity.ts
│       │   ├── errors/
│       │   │   ├── AppError.ts                 # base class
│       │   │   ├── AuthenticationError.ts
│       │   │   └── formatError.ts              # maps AppError -> GraphQLFormattedError
│       │   ├── plugins/
│       │   │   ├── persistedQueriesPlugin.ts
│       │   │   └── loggingPlugin.ts
│       │   └── db/
│       │       ├── prismaClient.ts (or knex/pg pool)
│       │       └── migrations/
│       ├── tests/
│       ├── Dockerfile
│       └── package.json
│
├── packages/                         # shared monorepo packages (pnpm/turborepo/nx)
│   ├── graphql-schema/                # shared typeDefs/types if federated or codegen source
│   └── shared-types/
│
├── docker-compose.yml                # postgres, redis, api, web
├── turbo.json / pnpm-workspace.yaml
└── README.md
```

**Notification flow (server → client) in this structure:**
`leadService` (e.g., lead assigned) → `notificationService.create()` writes DB row → `pubsub.publish(NOTIFICATION_ADDED, {...})` → Apollo Server subscription resolver on `graphql-ws` pushes over WebSocket → Next.js client component (`NotificationBell.tsx`) has `useSubscription` → on receipt, writes into Apollo Client's `InMemoryCache` via the subscription's own cache write **and/or** an explicit `update()` that pushes into a paginated `notifications` list field — no polling needed.

---

## 1. Apollo Client Caching Deep Dive

### 1.1 Normalization mechanics
`InMemoryCache` flattens the response tree into a flat map: `Cache["ROOT_QUERY"]`, `Cache["Lead:123"]`, `Cache["User:45"]`, etc. Each object gets a cache key computed as `__typename + ":" + id` (or `_id`) by default via `InMemoryCache`'s built-in `keyFields` heuristic (`dataIdFromObject` under the hood, now expressed via `typePolicies[Type].keyFields`).

- If a type has **no `id`/`_id`**, Apollo can't normalize it — it gets stored **inline** as a nested object under its parent's field, meaning **two queries returning "the same" embedded object won't share cache identity**, causing UI desync (update one, other stays stale) — a classic interview trap.
- Fix: custom `keyFields` (composite keys, e.g. `keyFields: ['orgId', 'sku']`) for entities without a single `id`, or `keyFields: false` to explicitly force embedding for value-objects (e.g., `Money { amount, currency }`) where identity doesn't matter and you *want* it colocated with the parent to avoid needless cache entries.

```ts
new InMemoryCache({
  typePolicies: {
    Lead: { keyFields: ['id'] },
    Address: { keyFields: false }, // always embedded, no normalization
    Money: { keyFields: ['currency', 'amount'] },
    Query: {
      fields: {
        leads: {
          keyArgs: ['filter', 'orgId'], // pagination args excluded from cache key
          merge(existing = { edges: [] }, incoming, { args }) {
            // Relay-style merge — see 1.5
          },
        },
      },
    },
  },
});
```

### 1.2 `typePolicies` deep dive
Three main things you configure per type:
1. **`keyFields`** — identity/normalization key.
2. **`fields`** — per-field `read`/`merge` functions, local-only computed fields, and `keyArgs` for field-level cache splitting by arguments.
3. Type-level `merge: true` shortcut for shallow merge of non-normalized nested objects.

`read` function lets you compute derived/client-only state without hitting network (e.g., `isOverdue` computed from `dueDate` at read time) — critical for "local-first" fields and integrating REST-derived data into the same cache (`@client` fields).

**Cross-question bait:** *"What happens if two different queries request the same field with different arguments?"* → Without `keyArgs`, Apollo stores each arg-combination as a separate cache entry under a serialized field key like `leads({"filter":"active"})`. If you don't declare `keyArgs`, Apollo auto-includes **all** arguments in the key by default, which is usually fine — the risk is when you *want* argument-based cache merging (e.g., paginated lists) and forget `keyArgs`/`merge`, leading to lost pages on every fetch of a new page.

### 1.3 Fetch policies — decision matrix

| Policy | Reads cache? | Hits network? | Use case |
|---|---|---|---|
| `cache-first` (default) | Yes, if complete | Only on miss | Static/rarely-changing reference data |
| `cache-and-network` | Yes (instant) | Always, updates in bg | Dashboards — instant paint + freshness |
| `network-only` | No | Always | Data must be fresh, still writes to cache |
| `no-cache` | No | Always | Doesn't touch cache at all (mutations-like reads, sensitive data) |
| `cache-only` | Yes | Never (errors if missing) | Local/derived-only reads, offline fallback |
| `standby` | Like cache-first | No active fetch | Used internally for polling pause / lazy activation |

**Nuance interviewers probe:** `cache-and-network` at the **query** level re-renders twice (stale-then-fresh) — fine for top-level `useQuery`, but if you set it as a **default** in `ApolloClient({ defaultOptions })`, it silently makes *every* query double-fetch, hurting perf. Lead-level answer: set fetch policy **per query**, not globally, and pair `cache-and-network` with `nextFetchPolicy: 'cache-first'` to avoid repeated network calls on cache-hit re-renders (e.g., component remount).

### 1.4 `evict()`, `gc()`, `modify()`
- **`cache.evict({ id, fieldName })`** — removes a normalized entity (or one field of it) from the cache. Does **not** immediately clean up dangling references (e.g., a list field still holding a `Reference` to the evicted object).
- **`cache.gc()`** — garbage collection pass that removes now-unreachable entities (nothing in `ROOT_QUERY`/`ROOT_MUTATION` points to them anymore). **You must call `gc()` after `evict()`** to actually shrink memory — a very common gotcha ("I called evict but memory didn't drop").
- **`cache.modify({ id, fields })`** — surgical, imperative field-level mutation of cache without needing the full new object shape; commonly used to add/remove an item from a list (append a `Reference` via `toReference()`) without refetching.

```ts
cache.modify({
  id: cache.identify({ __typename: 'Organization', id: orgId }),
  fields: {
    leads(existingRefs = [], { readField }) {
      return existingRefs.filter(ref => readField('id', ref) !== deletedLeadId);
    },
  },
});
cache.evict({ id: cache.identify({ __typename: 'Lead', id: deletedLeadId }) });
cache.gc();
```

**Cross-question:** *"Why not just refetch the query after delete?"* → Refetch = round-trip latency + wasted bandwidth for data you already mostly have; `modify`/`evict` is O(1) local, instant UI, no server load — but it's manual and error-prone (must handle every list/field referencing the entity, including paginated ones across different `keyArgs` buckets). Real lead answer: use `modify`/`evict` for hot-path mutations (delete, toggle), `refetchQueries` for complex cross-entity side effects where correctness > micro-perf.

### 1.5 Pagination — Relay-style connections
```graphql
type LeadConnection {
  edges: [LeadEdge!]!
  pageInfo: PageInfo!
}
type LeadEdge { cursor: String!, node: Lead! }
type PageInfo { hasNextPage: Boolean!, endCursor: String }
```
`merge` function concatenates `edges` across pages keyed by the *same* `keyArgs` (e.g., `filter`) while ignoring `after`/`first` so pagination args don't fragment the cache bucket:
```ts
leads: {
  keyArgs: ['filter'],
  merge(existing, incoming, { args }) {
    if (!existing || !args?.after) return incoming;
    return { ...incoming, edges: [...existing.edges, ...incoming.edges] };
  },
}
```

### 1.6 TTL / LRU-style eviction
Apollo Client's `InMemoryCache` has **no built-in TTL** — it's a persistent-until-evicted store, unlike React Query which has `staleTime`/`cacheTime` (gcTime) natively. Lead-level pattern for TTL in Apollo:
- Store a `fetchedAt` timestamp per entity via `read`/`merge`, and use a `read` function to return `undefined` (triggering refetch) if `Date.now() - fetchedAt > TTL`.
- Or use `apollo3-cache-persist` with a manual periodic sweep calling `evict` + `gc` on stale keys.
- For LRU/memory bounding at scale (large apps with thousands of entities), periodically call `cache.gc()` after route changes, and consider `possibleTypes`/normalization discipline to avoid unbounded growth from list-heavy screens (infinite scroll notification feeds are the classic culprit — cap retained pages).

### 1.7 Apollo Cache vs React Query vs Redis — when/how they coexist

| | Apollo InMemoryCache | React Query | Redis (server-side) |
|---|---|---|---|
| Location | Browser, per-tab | Browser, per-tab | Server, shared across instances |
| Shape | Normalized graph | Per-query-key, unnormalized | Whatever you serialize |
| Invalidation | Manual (`evict`/`modify`) or field policies | `staleTime`/`invalidateQueries`, TTL-native | TTL native (`EXPIRE`), pub/sub invalidation |
| Best for | GraphQL clients where entity-level consistency across queries matters | REST/any-async-fn caching, simpler mental model, great devtools | Cross-instance shared cache, session data, rate-limit counters, PubSub backplane |
| Consistency | Strong within client (normalized single source of truth) | Query-key scoped, can desync across keys unless invalidated together | Shared truth across all server instances |

**They coexist, don't compete:**
- **Apollo Client cache** = client-side, per-user, reduces client→server round-trips and drives optimistic UI.
- **Redis** = server-side, shared, reduces server→DB round-trips (e.g., cache resolver results, session store, and doubles as **PubSub backplane** for horizontally scaled subscriptions — see §3).
- **React Query** is Apollo's non-GraphQL cousin — if part of the app calls REST endpoints (e.g., file upload, legacy service), React Query manages that cache while Apollo manages GraphQL data; keep them in separate domains, don't try to merge normalization models. If migrating incrementally REST→GraphQL, it's common to run both simultaneously.
- Multi-layer caching in one CRM request: **Browser (Apollo)** → **CDN (persisted query GET caching)** → **Redis (resolver-level/DataLoader-level cache)** → **DB read replica**. Each layer has a different TTL/invalidation story; the failure mode to call out: **cache stampede** when Redis entry expires under high concurrency — mitigate with request coalescing/locking or stale-while-revalidate.

---

## 2. Mutation Data Flow

### 2.1 Optimistic response
```ts
const [addLead] = useMutation(ADD_LEAD, {
  optimisticResponse: {
    addLead: {
      __typename: 'Lead',
      id: `temp-${uuid()}`,     // must look real; __typename REQUIRED for normalization
      name: input.name,
      status: 'NEW',
    },
  },
  update(cache, { data }) {
    cache.modify({
      fields: {
        leads(existing = []) {
          const ref = cache.writeFragment({ data: data.addLead, fragment: LEAD_FRAGMENT });
          return [...existing, ref];
        },
      },
    });
  },
});
```
Apollo writes the optimistic data into the cache **immediately** on a separate "optimistic layer", triggers re-render, then **rolls back and replays** with the real server response (or an error) when it arrives — the optimistic layer is discarded and the real `update()` runs against the base cache. UI never "flashes" if optimistic and real shapes match.

**Failsafe/edge case interviewers dig into:**
- **Temp ID mismatch**: if your optimistic `id` doesn't get reconciled with the server's real ID, you get a duplicate entity (`Lead:temp-xxx` and `Lead:actual-id` both in cache). Fix: server should be queried with a `clientMutationId` or the `update()` function should explicitly evict the temp entity once real data lands, or design ID generation so client can predict the real ID (UUID generated client-side, sent to server, server persists that exact ID) — this is the clean, common lead-level pattern.
- **Optimistic response on error**: if the mutation fails, Apollo automatically rolls back the optimistic layer — but any **manual side effects** in `update()` that aren't purely cache writes (e.g., triggering analytics) will have already fired optimistically and won't roll back — don't put side effects in `update()`.

### 2.2 `update()` vs `refetchQueries` vs `cache.writeQuery`
- **`update(cache, mutationResult)`**: most efficient, manual, runs synchronously after mutation resolves — you decide exactly which cache entries change. Best when you know precisely what changed.
- **`refetchQueries: [{ query: GET_LEADS }]` or `refetchQueries: 'active'`**: simplest, guarantees correctness, costs a network round trip; use when the mutation's blast radius is hard to model manually (e.g., server-computed aggregates, counts, complex derived fields) or when several unrelated queries need to reflect the change.
- **`awaitRefetchQueries: true`**: mutation's promise won't resolve until refetches complete — needed if you navigate away right after and want fresh data guaranteed before the next screen reads cache.
- Normalization means **most of the time you don't need either** — if the mutation response includes the updated entity with matching `__typename + id`, Apollo **automatically merges it into every query that already references that entity**, no `update()` needed. This is the single most important "senior" insight: **write your mutation response to request the same fields your queries use**, and normalization handles UI sync for free. `update()`/`refetchQueries` are only needed for **structural** changes (add/remove from a list, changed relationships) that normalization can't infer.

### 2.3 Cross-questions
- *"Mutation succeeds but UI doesn't update — why?"* → Missing `id`/`__typename` in mutation response (can't normalize/merge), or entity added to a list but list field has no `merge` function so incoming reference isn't appended, or stale `keyArgs` bucket (list under different filter args untouched by design).
- *"How do you avoid double-toast/double-refetch when both optimistic UI and a subscription push the same change?"* → Dedup by mutation-originated `clientMutationId` — if the subscription event's payload matches the client's own recent mutation, skip re-processing (or rely on normalization idempotency: writing the same entity twice with the same ID is a no-op merge).

---

## 3. Pub/Sub & Subscriptions

### 3.1 Transport
Modern Apollo Server uses **`graphql-ws`** (not the legacy `subscriptions-transport-ws`) over a raw WebSocket upgrade on the same HTTP server:
```ts
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';

const wsServer = new WebSocketServer({ server: httpServer, path: '/graphql' });
const serverCleanup = useServer(
  {
    schema,
    context: async (ctx) => {
      const token = ctx.connectionParams?.authToken;
      const user = await verifyToken(token);   // auth happens at CONNECTION time
      return { user, pubsub, loaders: createLoaders() };
    },
    onDisconnect() { /* cleanup subscriptions, decrement metrics */ },
  },
  wsServer
);
```
Client:
```ts
const wsLink = new GraphQLWsLink(createClient({
  url: 'wss://api.example.com/graphql',
  connectionParams: () => ({ authToken: getToken() }),
  retryAttempts: 5,
  shouldRetry: () => true,
}));
const splitLink = split(
  ({ query }) => {
    const def = getMainDefinition(query);
    return def.kind === 'OperationDefinition' && def.operation === 'subscription';
  },
  wsLink,
  httpLink,
);
```

### 3.2 Resolver pattern
```ts
Subscription: {
  notificationAdded: {
    subscribe: withFilter(
      (_, __, ctx) => ctx.pubsub.asyncIterator(NOTIFICATION_ADDED),
      (payload, _, ctx) => payload.notificationAdded.userId === ctx.user.id, // per-user filtering
    ),
  },
},
```
`withFilter` prevents broadcasting every event to every connected client — critical, otherwise every user's browser evaluates every notification in the system (security leak + perf disaster at scale).

### 3.3 In-memory `PubSub` vs Redis/Kafka-backed
- **`graphql-subscriptions` in-memory `PubSub`**: fine for single-instance dev, **broken in production** the moment you run >1 server instance (horizontal scaling / rolling deploys) — an event published on instance A never reaches a WebSocket connected to instance B. This is *the* classic scaling gotcha to call out proactively.
- **`graphql-redis-subscriptions` (`RedisPubSub`)**: publishes to a Redis channel; every instance subscribes to Redis, so instance A's publish reaches WS clients connected on instance B, C, etc. Requires Redis (or Redis Cluster) as the backplane. Good default choice for moderate scale.
- **Kafka-backed**: for very high throughput / durability / replay requirements (e.g., audit trail of every notification event, or fan-out to multiple downstream consumers beyond just WS pushes — analytics, email digests). Adds operational complexity (consumer groups, partitioning by userId/tenant for ordering) — only justify this over Redis when you need durable replay or multi-consumer fan-out, not just pub/sub delivery.
- **Sticky sessions caveat**: WebSockets are long-lived and stateful at the connection level — your load balancer needs either sticky sessions (not required for pub/sub correctness since Redis brokers cross-instance, but relevant for reconnection/backpressure) or you accept clients reconnecting to a different instance on LB rebalancing (client should auto-reconnect via `retryAttempts`).

### 3.4 Real-time notification system design (what to say in system design segment)
```
[DB write (notification)] 
     → [Service publishes to RedisPubSub channel "notifications:{userId}"]
     → [Every API instance subscribed to Redis relays to its local in-process asyncIterator]
     → [graphql-ws pushes to matching WS connections filtered by userId]
     → [Apollo Client subscription updates cache -> UI bell badge increments]
```
Failsafes to mention unprompted:
- **Missed events on disconnect**: subscriptions are fire-and-forget over the wire — if the client is offline when an event fires, it's lost. Mitigate: on reconnect/mount, run a **query** (`unreadNotifications`) to reconcile ("subscribe for live updates, query for catch-up") — never rely on subscriptions as the sole source of truth.
- **Backpressure**: a burst of events (bulk import triggers 10k notifications) can overwhelm a client — batch/debounce on the server or coalesce into a single "N new notifications" event.
- **At-least-once vs exactly-once**: Redis pub/sub is at-most-once (no delivery guarantee, no persistence) — if you need guaranteed delivery, that's a Kafka/durable-queue argument, or store notifications in DB (source of truth) and treat the subscription purely as a "hey, go refetch/merge" signal rather than carrying full payload.

---

## 4. API Design for a CRM with RBAC

### 4.1 GraphQL vs REST — same system, when to use which
- **GraphQL**: client-driven aggregation screens (dashboards pulling lead + contact + activity + notification counts in one round trip), mobile clients needing to minimize over-fetching, rapidly evolving front-end schemas.
- **REST**: file uploads/downloads (binary, streaming — GraphQL multipart spec exists but REST is simpler/more cacheable via HTTP semantics), webhooks (third parties expect REST/HTTP callbacks, not GraphQL clients), simple CRUD internal service-to-service calls where GraphQL's flexibility is unneeded overhead, and anything needing **native HTTP caching** (CDN-cacheable GET endpoints) — GraphQL POST requests bypass HTTP caching unless you adopt persisted queries + GET.
- Common lead-level answer: **coexist** — GraphQL as the primary client-facing API/BFF, REST for file/webhook/health-check/service-to-service edges, sometimes exposed from the *same* Node process (Express app with both `/graphql` and `/api/webhooks/*`).

### 4.2 Schema-first vs code-first
- **Schema-first** (SDL `.graphql` files + resolver maps): schema is the contract, reviewable independently of implementation, easier cross-team (FE can write against schema before BE resolvers exist), what Apollo Server historically encourages.
- **Code-first** (e.g., `type-graphql`/`Pothos`/`Nexus` — decorators/builders generate schema from TS classes): single source of truth in code, strong TS inference without codegen mismatch risk, better for teams that want compile-time safety over schema-first drift risk.
- Lead-level take: schema-first + `graphql-codegen` for types gives you the contract-first governance benefit *and* type safety — most CRM-scale teams land here; code-first shines when the team is small/TS-only and wants zero SDL/resolver drift.

### 4.3 Nullability discipline
Default to **non-null (`!`) output fields** unless a field can genuinely be absent — over-using nullable fields forces every client to null-check everywhere and hides real invariants. But: **never mark a list item type non-null if a single failing resolver should not null out the entire list** (partial failure semantics — `[Lead]` vs `[Lead!]!` vs `[Lead!]`: if one `Lead` resolver throws, `[Lead!]` degrades that one element to `null` while the array of items around it survives; `[Lead!]!` nulls the *entire* list up to the nearest nullable ancestor — a very common "gotcha" question about GraphQL error propagation ("null bubbling")).

### 4.4 Pagination — Relay connections (recap from §1.5) as the input-type pattern
```graphql
input LeadFilterInput { status: LeadStatus, ownerId: ID, search: String }
type Query {
  leads(filter: LeadFilterInput, first: Int, after: String): LeadConnection!
}
```
Input types group related args, version cleanly (add optional fields without breaking clients), and keep resolver signatures stable as filters grow.

### 4.5 DataLoader for N+1
```ts
const userLoader = new DataLoader(async (ids: readonly string[]) => {
  const users = await db.user.findMany({ where: { id: { in: [...ids] } } });
  const map = new Map(users.map(u => [u.id, u]));
  return ids.map(id => map.get(id) ?? new Error(`User ${id} not found`));
});
```
- Batches all `.load(id)` calls within a single event-loop tick into one `WHERE id IN (...)` query.
- **Must be created per-request** (in `context`), never as a module-level singleton — otherwise you leak cached results/identity across users and requests (a serious security/correctness bug: user A's DataLoader cache could serve user B's request if shared).
- Caching within DataLoader is **per-request only** by default (in-memory `Map`) — doesn't replace Redis; DataLoader solves N+1 *batching*, Redis solves *cross-request* caching. They compose: DataLoader's batch function can itself read-through Redis.

### 4.6 Repository/service layer separation
```
Resolver (thin, GraphQL-shape only)
   → Service (business logic, authorization checks, orchestration, publishes events)
      → Repository (pure data access, no GraphQL/business knowledge, swappable ORM/DB)
```
Why: resolvers stay testable/thin, business logic is reusable from REST endpoints or background jobs (not trapped inside a resolver), repository swap (Postgres→different DB, or adding a read replica) doesn't touch business logic.

### 4.7 Federation/stitching
- **Schema stitching** (legacy): manually merges multiple schemas at the gateway, brittle with type conflicts, mostly superseded.
- **Apollo Federation**: each subgraph (e.g., `leads-service`, `notifications-service`, `users-service`) declares entities via `@key(fields: "id")`, gateway composes a supergraph and routes fields to the owning subgraph, with `__resolveReference` to fetch entity data across service boundaries. Use when the CRM outgrows a single monolithic GraphQL server organizationally (separate teams/deploys own separate domains) — the classic trigger is "team ownership boundaries," not just "system is big."

---

## 5. Auth & Security

### 5.1 JWT flow
```
Login (REST or mutation) → verify credentials → sign JWT (short-lived access + refresh token)
→ client stores access token (memory, not localStorage ideally) + refresh in httpOnly cookie
→ every GraphQL request: Authorization: Bearer <token> header via authLink
→ Apollo Server context: verify JWT signature/expiry → attach `user` to context
→ resolvers/directives check `context.user` for authn, `context.user.roles` for authz
```
```ts
const authLink = setContext((_, { headers }) => ({
  headers: { ...headers, authorization: token ? `Bearer ${token}` : '' },
}));
```
**Refresh flow edge case**: on 401/expired token mid-session, use `onError` link to trigger silent refresh (call refresh endpoint) then retry the original operation via `apollo-link-token-refresh` or a custom retry link — must handle **concurrent requests all hitting 401 simultaneously** (dedupe refresh calls so you don't fire 10 refresh requests at once — single in-flight refresh promise shared across callers).

### 5.2 Context injection & authorization middleware
```ts
context: async ({ req, connectionParams }) => {
  const token = req?.headers.authorization?.split(' ')[1] ?? connectionParams?.authToken;
  const user = token ? await verifyJwt(token) : null;
  return { user, loaders: createLoaders(user), pubsub };
},
```
Resolver-level guard pattern:
```ts
function requireRole(role: string) {
  return (next) => (parent, args, ctx, info) => {
    if (!ctx.user) throw new AuthenticationError('Not authenticated');
    if (!ctx.user.roles.includes(role)) throw new ForbiddenError('Insufficient role');
    return next(parent, args, ctx, info);
  };
}
```

### 5.3 Directive-based authorization
```graphql
directive @auth on FIELD_DEFINITION
directive @hasRole(role: Role!) on FIELD_DEFINITION | OBJECT

type Mutation {
  deleteLead(id: ID!): Boolean @hasRole(role: ADMIN)
}
type Lead {
  ssn: String @hasRole(role: ADMIN)   # field-level authz — hides sensitive fields per-role
}
```
Implemented via `mapSchema`/`getDirective` (graphql-tools) wrapping the field resolver — checks run **before** the underlying resolver executes, throwing (or returning `null` for field-level, non-blocking style) as appropriate. **Field-level > object/query-level** because it lets one query shape serve multiple roles (e.g., `Lead.ssn` hidden for non-admins while the rest of `Lead` still resolves) — avoids N+1 duplicate queries per role.

**Cross-question**: *"Directive throws vs returns null for unauthorized field — which do you pick?"* → Throwing on a nullable field nulls just that field (good, minimal blast radius) but on a **non-null** field it bubbles the null up to the nearest nullable ancestor, potentially nulling out the whole `Lead` object for an unrelated missing permission on one field — so sensitive fields you may want hidden-per-role should be **nullable in the schema**, and the directive should return `null` rather than throw when the reason is "just not authorized to see this field" (reserve throwing for true auth *errors*, not routine role-based hiding).

### 5.4 Centralized error handling
```ts
export class AppError extends GraphQLError {
  constructor(message: string, code: string, extra?: object) {
    super(message, { extensions: { code, ...extra } });
  }
}
export class AuthenticationError extends AppError {
  constructor(msg = 'Not authenticated') { super(msg, 'UNAUTHENTICATED'); }
}
export class ForbiddenError extends AppError {
  constructor(msg = 'Forbidden') { super(msg, 'FORBIDDEN'); }
}
```
```ts
new ApolloServer({
  schema,
  formatError: (formattedError, error) => {
    if (error instanceof AppError) return formattedError;
    logger.error(error); // don't leak internals
    return { message: 'Internal server error', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
  },
});
```
**GraphQLError vs REST error middleware**: REST uses HTTP status codes (404/401/403/500) as the primary error signal, caught by generic middleware wrapping the whole request. GraphQL always returns **HTTP 200** (even on resolver errors) — the error signal lives in the `errors[]` array with an `extensions.code`, because a single response can be **partial success** (some fields resolved, some errored) — there's no single "status code" that fits a response with mixed success/failure. This is a top interview trap: *"why does GraphQL return 200 on errors?"* → because errors are field-level, not request-level; some GraphQL servers/gateways *do* use non-200 for **request-level** failures (malformed query, failed auth at the transport layer) but resolver errors are always 200 + `errors[]`.

### 5.5 Query depth/complexity limiting
```ts
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

new ApolloServer({
  schema,
  validationRules: [depthLimit(7), createComplexityLimitRule(1000)],
});
```
Depth limiting stops naive nested-relation DoS (`lead { contacts { leads { contacts { ... } } } }`); complexity limiting is more precise — assigns cost per field (list fields cost more, multiplied by pagination `first` arg) so a shallow-but-wide query (`leads(first: 10000)`) is also caught, which depth-limiting alone misses.

### 5.6 Persisted queries
Client sends a query **hash** instead of full query text (`APQ` — Automatic Persisted Queries): first request sends hash, server 404s ("not found"), client resends hash+full query once, server caches hash→query mapping (in-memory or Redis for multi-instance), subsequent requests just send the hash. Benefits: smaller request payloads, **enables GET requests → CDN cacheable**, and can be combined with an **allow-list** (only pre-registered query hashes accepted) to prevent arbitrary/malicious query execution in production — a strong security posture for public-facing GraphQL APIs.

### 5.7 Rate limiting
Per-user/per-IP at the gateway (Redis-backed token bucket/sliding window, e.g., `graphql-rate-limit` directive `@rateLimit(max: 10, window: "1m")` on expensive mutations), plus **cost-based** rate limiting (combine with complexity score so "1 expensive query" counts more than "1 cheap query") rather than flat request counting.

### 5.8 Input validation
Validate at the **resolver/service boundary**, not just relying on GraphQL's type system (GraphQL validates *shape*, not *business rules* — e.g., `email: String!` doesn't validate email format). Use a schema validation lib (Zod/Yup/class-validator) inside services, return validation errors as a structured `UserInputError`/`AppError` with field-level details in `extensions`.

### 5.9 CSRF considerations
GraphQL over POST with `Content-Type: application/json` is **not** vulnerable to classic form-based CSRF (browsers can't set that content-type via a simple form submission) — but if you support **GET** requests (for persisted queries/CDN caching) or accept `Content-Type: text/plain`/`multipart/form-data`, you reopen the CSRF surface. Mitigations: require a custom header (e.g., `X-Requested-With` or `Apollo-Require-Preflight: true`) that forces a CORS preflight (can't be set by a simple cross-origin form), enforce strict CORS `Access-Control-Allow-Origin`, and if using cookie-based auth (not Bearer tokens), add `SameSite=Strict/Lax` cookies + CSRF tokens on top.

---

## 6. Concurrency & Data Availability

### 6.1 Race conditions & optimistic locking
Classic CRM race: two reps edit the same Lead simultaneously → last-write-wins silently discards one edit. Fix with **optimistic locking**: add a `version` (or `updatedAt`) column; mutation requires client to pass the version it read; `UPDATE ... WHERE id = ? AND version = ?` — if 0 rows affected, throw a `ConflictError` (HTTP 409 equivalent / GraphQL `extensions.code: 'CONFLICT'`) and let client re-fetch + merge/prompt user. Contrast with **pessimistic locking** (`SELECT ... FOR UPDATE`) — heavier, holds DB locks across the request, generally avoided in web APIs due to connection/latency risk; optimistic locking is the standard GraphQL mutation pattern.

### 6.2 Idempotency in mutations
Non-idempotent mutations (e.g., `createLead`) retried by flaky networks/client retry logic can create duplicates. Fix: client generates an **idempotency key** (UUID) sent with the mutation; server stores `(idempotencyKey → result)` for a TTL window (Redis) and returns the cached result on retry instead of re-executing. Critical for `graphql-ws` reconnect scenarios and for mutations triggered by webhooks that may redeliver.

### 6.3 Read replicas / sharding
Route `Query` resolvers to a read replica connection pool, `Mutation` resolvers to primary — but beware **read-your-write consistency** lag (user mutates, immediate refetch on replica may not reflect it yet due to replication delay). Mitigations: pin the *same request* to primary right after a mutation (session affinity for a few seconds), or have the mutation resolver return the updated entity directly (client relies on the mutation response + normalization instead of an immediate refetch — ties back to §2.2). Sharding (by tenant/orgId in a multi-tenant CRM) requires resolvers/DataLoader batch functions to be shard-aware — DataLoader batching across shards needs care not to issue one giant cross-shard `IN` query.

### 6.4 CDN/persisted query caching
GET + APQ (§5.6) makes GraphQL responses cacheable by a CDN using standard `Cache-Control` headers set per-operation (via a plugin inspecting the operation name/complexity) — only safe for **queries with no user-specific data** or when cache key includes auth context (Vary on Authorization, or separate cache per user segment) — a common mistake is caching an "authenticated" response at the CDN and leaking it across users; must vary cache key by identity/role when responses differ per user.

### 6.5 Horizontal scaling of subscription servers
Stateful WS connections don't scale like stateless HTTP — options: (a) Redis-backed PubSub (§3.3) so any instance can serve any connection's events, (b) sticky sessions at LB so a client's reconnect lands predictably (helps operationally, not required for correctness with Redis backplane), (c) connection draining on deploy (graceful shutdown: stop accepting new WS, let existing ones finish or force-reconnect clients via a "server restarting" message so they reconnect to a new instance rather than erroring), (d) monitor per-instance connection count for autoscaling triggers (WS connection count, not just CPU, since idle WS connections are cheap on CPU but consume memory/file descriptors).

---

## 7. Next.js + TypeScript Integration

### 7.1 SSR/SSG with Apollo in Next.js App Router
- **Server Components**: fetch data directly (no `useQuery` — that's a client hook). Use a lightweight per-request Apollo Client (or plain `graphql-request`/fetch) in RSC, since Apollo's React hooks require client-side context. Pattern:
```ts
// lib/apollo/apolloServerClient.ts
export function getServerClient() {
  return new ApolloClient({
    ssrMode: true,
    link: new HttpLink({ uri: process.env.API_URL, fetch }),
    cache: new InMemoryCache(),
  });
}
```
```tsx
// app/(dashboard)/leads/page.tsx  (Server Component)
export default async function LeadsPage() {
  const client = getServerClient();
  const { data } = await client.query({ query: GET_LEADS, context: { headers: authHeaders() } });
  return <LeadsList initialData={data.leads} />;
}
```
- **Client Components** (`'use client'`) use the browser singleton `ApolloClient` + `ApolloProvider`, needed for anything with `useQuery`/`useMutation`/`useSubscription` (subscriptions **cannot** run on the server at all — WS only makes sense client-side).
- Avoid the old `getDataFromTree`/`renderToStringWithData` SSR hydration dance from pages-router; App Router's RSC model makes that unnecessary — fetch in the server component, pass serialized data down as props, optionally **seed** the client cache on mount (`client.writeQuery` or `restore()`) so the client Apollo cache isn't empty for subsequent client-side navigations/refetches.

### 7.2 Server vs client data fetching split
- Initial page load / SEO-relevant data → Server Component `client.query()`.
- Anything needing interactivity, subscriptions, or user-triggered mutations → Client Component with real `useQuery`/`useMutation`/`useSubscription`.
- Common pitfall: instantiating a **new** Apollo Client per Server Component render is correct (avoids cross-request cache leakage — server-side state must not be shared between users, unlike the browser singleton) — don't reuse a module-level singleton client for RSC fetching, that would leak one user's cached/authenticated data into another user's request in a shared Node process.

### 7.3 GraphQL Codegen
```yaml
# codegen.ts
schema: ${API_URL}/graphql   # or a local .graphql SDL file
documents: 'src/graphql/operations/**/*.graphql'
generates:
  src/graphql/generated/index.ts:
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
```
Generates fully-typed hooks (`useGetLeadsQuery`, `useAddLeadMutation`) with typed variables/data — eliminates hand-written interfaces drifting from the actual schema, and typed `TData`/`TVariables` flow directly into `cache.modify`/`writeFragment` calls in §2 for compile-time-safe cache surgery. In CI, run codegen as a check (`--check`/diff against committed output) so schema changes that break client types fail the build before merge, not at runtime.

---

## 50 Advanced Interview Questions (Grouped, with Deep Answer Pointers + Cross-Questions)

### A. Caching (1–10)
**1. Walk me through exactly how `InMemoryCache` normalizes a nested query response.**
→ Flattens by `__typename:id` into a flat map; `ROOT_QUERY` holds top-level field pointers as `Reference` objects; nested entities become their own top-level cache entries referenced by pointer, not copied inline — so any query touching `Lead:123` shares the *same* underlying object.
*Cross-Q: "What if the object has no id?"* → Falls back to embedding inline under the parent (no sharing across queries) unless you define composite `keyFields`.

**2. When would you set `keyFields: false` on a type, and why not just leave default normalization?**
→ For value objects with no identity of their own (e.g., `Address`, `Money`) where you want them embedded with the parent to avoid creating meaningless standalone cache entries and to avoid two different `Address` objects with identical shape unintentionally colliding if they happened to share a fabricated key.

**3. `cache.evict()` didn't reduce memory usage — why?**
→ Forgot `cache.gc()`; evict just marks removal, gc actually reclaims unreachable entries. Also possible: another live query still references the entity (not actually unreachable), so gc correctly keeps it.

**4. Explain `cache-and-network` risk if set as a global default.**
→ Every query double-fires (cache paint + forced network refetch) including on every remount, which can flood the network layer; mitigate with per-query overrides and `nextFetchPolicy`.

**5. How do you implement TTL-based cache expiry in Apollo Client, given there's no native TTL?**
→ Store `fetchedAt` per entity, custom `read` function returns `undefined` past threshold to force refetch, or periodic sweep calling `evict`+`gc` on stale keys; contrast with React Query's native `staleTime`.
*Cross-Q: "Why doesn't Apollo have this built in?"* → Because Apollo Client's cache model prioritizes normalized consistency over per-query staleness; TTL is a query-level concept layered on top, not core to the entity graph.

**6. Two queries return the same entity with conflicting field values (e.g., one stale). What does Apollo do?**
→ Last write wins per field at merge time — whichever response resolves last overwrites that field in the normalized entry; if responses race, you can get a "flicker" where the field briefly shows old, then new, then possibly reverts if an in-flight older request resolves after a newer one (network race) — mitigate with `nextFetchPolicy`, aborting stale in-flight requests, or including a version/timestamp field checked in a custom `merge`.

**7. How would you cap unbounded cache growth from an infinite-scroll notification feed?**
→ Custom `merge` that keeps only last N pages/items, explicit `evict` of off-screen entities on scroll-out, periodic `gc()`, and server-side pagination limits — cache isn't free memory, it's a live retained object graph.

**8. Compare Apollo cache invalidation to Redis invalidation strategy.**
→ Apollo: mostly manual/normalization-driven, no native TTL, invalidation via mutation response shape or explicit `evict`/`modify`. Redis: TTL-native (`EXPIRE`), can also do pub/sub-driven invalidation (publish "invalidate key X" to all instances) — pattern often mirrored client-side via subscriptions telling Apollo Client to `refetch`/`evict`.

**9. Client cache shows correct data but server DB was rolled back (failed transaction after optimistic UI applied). How do you handle?**
→ Mutation should return an error; Apollo auto-rolls-back the optimistic layer on error — but if the *service* returns success-looking data that later proves wrong (e.g., async downstream failure after resolver returned), you need a compensating event/subscription to correct the client cache after the fact — pure optimistic UI can't know about async post-commit failures.

**10. `keyArgs` omitted on a paginated field — what breaks?**
→ Each unique arg combination (e.g., each page's `after` cursor) becomes its own separate cache bucket instead of merging into one growing list — "page 2" doesn't append to "page 1," it replaces the field entirely per unique arg signature, so UI shows only the latest page fetched, not the accumulated list.

### B. Mutations (11–18)
**11. Why might a mutation's normalized entity update UI in one query but not another showing the same entity differently?**
→ The second query likely doesn't request the changed field (partial field overlap — normalization only overwrites fields actually present in the mutation response) or the entity is referenced via a non-normalized list `merge` that wasn't updated structurally.

**12. Optimistic response provided the wrong `__typename` — what happens?**
→ Apollo can't normalize/merge it into the graph correctly (or merges into a wrong/unexpected cache slot), typically causing the optimistic UI update to be dropped/ignored or written somewhere unreachable — always mirror exact server shape including `__typename` for every level.

**13. When is `refetchQueries` actively the *wrong* choice at scale?**
→ High-traffic mutation (e.g., "like" button) triggering full list refetch for every click — network/DB cost multiplies with usage; use `update`/`modify` for high-frequency mutations, reserve refetch for low-frequency/high-complexity ones.

**14. How do you prevent duplicate entities after optimistic create + real response reconciliation?**
→ Client-generated real ID (UUID) sent to server so optimistic and real IDs match exactly (no temp/real split), or explicit `update()` evicts the temp entity once real data arrives.

**15. `awaitRefetchQueries` — when do you need it?**
→ When code immediately after the mutation call (e.g., a redirect) depends on the refetched data being in cache already; without it, refetches fire-and-forget in the background and the next screen may read stale/incomplete cache.

**16. A mutation updates a computed/aggregate field on the server (e.g., `unreadCount`) that isn't part of the mutation's own return type — how do you keep it in sync?**
→ Either include it in the mutation response explicitly (add it to the selection set even if "unrelated" to the primary entity) so normalization picks it up, or use `update()`/`cache.modify` to manually decrement/increment it, or fall back to `refetchQueries` targeting just that aggregate query.

**17. What's the danger of putting side effects (analytics, toasts) inside `update()`?**
→ `update()` runs against the optimistic layer too — on mutation failure the cache change rolls back but any imperative side effect already fired and won't be undone, causing "success" toasts/analytics on operations that ultimately failed.

**18. How does Apollo decide whether to merge a mutation response into cache automatically vs needing manual `update`?**
→ Automatic normalization merge happens for any entity in the response matching `__typename:id` already present in the cache — happens for *field value updates on existing entities*; anything structural (added to a list, removed from a list, relationship changes) requires manual `update`/`modify` since Apollo has no way to infer "this new entity should also appear in this other list."

### C. Subscriptions/PubSub (19–26)
**19. In-memory PubSub works in dev but breaks after horizontal scaling in prod — root cause and fix?**
→ Each instance has its own isolated in-memory event bus; publish on instance A never reaches subscribers connected to instance B. Fix: Redis (or Kafka)-backed PubSub as a shared cross-instance broker.

**20. How do you scope a subscription so users only receive their own events, not everyone's?**
→ `withFilter` on the subscribe resolver checking payload ownership against `context.user.id`; also consider partitioning the pubsub channel itself per-user/org (`notifications:${userId}`) so the server doesn't even need to filter irrelevant events client-by-client, reducing fan-out cost.

**21. Client was offline for 10 minutes — how do you ensure it doesn't miss notifications?**
→ Subscriptions are fire-and-forget/no persistence; on reconnect, run a catch-up **query** for anything since last-seen timestamp/cursor — never treat subscriptions as the sole source of truth, always pair with query-based reconciliation.

**22. How do you authenticate a WebSocket subscription connection (no HTTP headers per-message)?**
→ Auth happens once at `connectionParams` during the initial WS handshake (`graphql-ws`'s `context` callback receives `connectionParams`), verified there and attached to the socket's context for the connection's lifetime; reconnect re-authenticates.

**23. When would you choose Kafka over Redis for the pubsub backplane?**
→ Need durability/replay (events must not be lost even if no subscriber was listening), multiple independent consumer groups (WS gateway + analytics pipeline + audit log all consuming the same event stream), or very high sustained throughput with partitioned ordering guarantees — Redis pub/sub is fire-and-forget with no persistence or replay.

**24. How do you gracefully drain WebSocket connections during a rolling deploy?**
→ Stop accepting new connections on the old instance, send a custom "server restarting, please reconnect" message so `graphql-ws` clients (with `retryAttempts`) reconnect to a new instance, then close remaining sockets after a grace period rather than hard-killing them.

**25. A burst of 10,000 notifications (bulk import) floods a single client — how do you protect it?**
→ Server-side batching/debouncing/coalescing into a single "N new items" event rather than N individual pushes; client can also throttle re-renders, but the real fix is reducing event volume server-side.

**26. How does a subscription event, once received, actually update the Apollo Client cache — automatically or do you need `update`?**
→ `useSubscription`'s result data normalizes into cache like any other operation if it returns properly-shaped entities with `__typename`/`id`; for structural changes (adding to a list) you still need an explicit `update`/`onData` handler doing `cache.modify`, same rules as mutations.

### D. API Design/RBAC (27–36)
**27. In a CRM, when would you expose both a GraphQL and REST endpoint for "the same" data?**
→ E.g., `Lead` data via GraphQL for the app UI (flexible aggregation), but a REST `/api/leads/export.csv` endpoint for bulk export/streaming since GraphQL isn't a natural fit for large binary/streamed payloads, or a REST webhook endpoint for third-party CRM integrations that expect standard REST semantics.

**28. Schema-first vs code-first — what breaks down at scale with each?**
→ Schema-first: SDL and resolver implementation can silently drift without codegen discipline (resolver returns a field not in SDL, or vice versa, only caught at runtime/introspection). Code-first: schema is implicit in code structure, harder for cross-team schema review/governance without generating and publishing the SDL separately, and 3rd-party schema tooling (linting, federation composition checks) often expects SDL as the artifact.

**29. Why choose `[Lead!]` over `[Lead!]!` for a list field, and what's the real-world impact?**
→ `[Lead!]!` means if *any single* item's resolver throws, GraphQL nulls the **entire list** (bubbling up because the outer `!` also can't be null); `[Lead!]` allows the list itself to be `null` while the array wrapper survives non-null items — but neither prevents one bad item from still requiring null-bubbling *up to the list itself* if items are `!`. For true per-item partial-failure tolerance without losing the whole list, item type must be nullable (`[Lead]`), sacrificing the "list items are guaranteed non-null" guarantee for resilience.

**30. Explain DataLoader batching timing — why does it only work within "the same tick"?**
→ DataLoader collects all `.load()` calls synchronously queued before the event loop yields (via `process.nextTick`/microtask), then fires one batched request; calls made after an `await` in between are in a *new* tick/batch, so architecting resolvers to call `.load()` as early/parallel as possible (not sequentially awaiting each) is required to actually get batching benefits.

**31. Why must DataLoader instances be created per-request, and what's the failure mode if they aren't?**
→ Per-request scoping prevents cross-user cache/data leakage — a singleton DataLoader would serve User A's cached entity to User B's request if they load the same ID, a real security bug (data leak across tenants/users), plus stale-data risk across requests since DataLoader caches indefinitely per instance lifetime.

**32. How do repository/service/resolver layers make testing easier and enable REST+GraphQL sharing logic?**
→ Services contain business rules independent of transport (GraphQL resolver or REST controller both call the same `leadService.createLead()`), so unit tests target services directly without spinning up a GraphQL server; repositories are mockable for service-layer tests without a real DB.

**33. When do you actually need Apollo Federation vs a single monolithic schema?**
→ When separate teams need independent deploy cadence/ownership over schema slices (organizational boundary, not just "the schema got big") — federation's cost (gateway complexity, cross-service entity resolution latency, `__resolveReference` N+1 risk across services) isn't worth it purely for a large-but-single-team schema; a well-organized modular monolith schema often suffices there.

**34. Federation adds entity resolution across services — how do you avoid N+1 *across services* (not just within one)?**
→ Gateway batches `_entities` requests per subgraph automatically for a single query, but chained resolvers across multiple hops can still fan out; use DataLoader within each subgraph's `__resolveReference`, and design entity keys to minimize cross-service hops per query.

**35. How do input types improve API evolution compared to many scalar arguments?**
→ Adding an optional field to an `input` type is non-breaking for existing clients (they just don't send it); adding a new top-level scalar argument to a field is equally non-breaking too, but grouping related filter/sort/pagination args in one named input keeps signatures stable and self-documenting as the filter surface grows, and allows nested/reusable structures (e.g., reusing `LeadFilterInput` in multiple queries).

**36. RBAC: role check in resolver vs directive vs service layer — where should it *really* live, and why?**
→ Business-rule authorization (e.g., "sales rep can only edit their own leads," ownership-based, not just role-based) belongs in the **service layer** since it needs domain context; simple role-gating ("must be ADMIN to hit this field/mutation at all") is well-suited to **directives** as declarative, schema-visible policy. Directives shouldn't try to encode complex ownership logic — that becomes unreadable in SDL and untestable in isolation.

### E. Security (37–44)
**37. Why does GraphQL return HTTP 200 even when a resolver throws?**
→ Because a single response can be partially successful (some fields resolved, others null+errored) — there's no single HTTP status that represents "half succeeded, half failed" in the way `errors[]` + partial `data` can. Transport-level failures (malformed query, failed parsing/validation) may still use non-200 depending on server/gateway config, but resolver-level errors are, by spec convention, part of a 200 response.

**38. Field-level `@auth` directive: throw or return null when a user lacks permission — which and why?**
→ Return `null` for "not authorized to see this field" (routine, expected per-role hiding) especially if the field is nullable, so the rest of the object still resolves; reserve throwing (`ForbiddenError`) for cases where absence itself is meaningful and expected to surface as an explicit error (e.g., an entire mutation blocked), and make sure sensitive fields are schema-nullable if you intend to null them silently per-role — otherwise nulling bubbles up and destroys the parent object.

**39. How do persisted queries with an allow-list improve security beyond just performance?**
→ Server only executes pre-registered query hashes — arbitrary ad-hoc queries (including deeply nested/expensive introspection-driven attacks) are rejected outright, drastically shrinking the attack surface for a public API; combine with disabling introspection in production for defense-in-depth.

**40. Query depth limiting alone isn't sufficient — why, and what closes the gap?**
→ A shallow query can still be extremely expensive if it requests a huge list (`leads(first: 1000000)`) — depth limiting doesn't see "width"/pagination cost; complexity/cost analysis (assigning per-field cost, multiplied by list size arguments) catches wide-but-shallow abuse that depth limiting misses.

**41. How do you prevent a race condition where a refresh-token flow fires multiple simultaneous refresh calls?**
→ Share a single in-flight refresh Promise across all concurrent 401 responses (via an `errorLink`/interceptor checking "is a refresh already in progress" before firing a new one), queue/retry the original failed requests once that shared promise resolves, rather than each request independently triggering its own refresh call.

**42. Is a GraphQL POST endpoint vulnerable to classic CSRF? What changes that answer?**
→ Not via simple HTML form submission (can't set `application/json` content-type via a bare form), so it's inherently more resistant than typical cookie-authenticated REST endpoints — but if you (a) support GET for persisted queries, (b) allow `text/plain`/`multipart` content types, or (c) rely on cookies rather than header-based Bearer tokens for auth, you reopen CSRF risk and need explicit mitigations (custom-header preflight requirement, SameSite cookies, CSRF tokens).

**43. Where should input validation live if GraphQL's type system already enforces types?**
→ GraphQL validates *shape/type* (`String!`, `Int`) not *business rules* (valid email format, string length limits, cross-field consistency) — that validation belongs in the service layer (or a validation middleware wrapping resolvers) using something like Zod, returning structured `UserInputError` with field-level detail in `extensions`, independent of GraphQL's own type validation.

**44. How do you avoid leaking internal error details (stack traces, DB errors) to clients while still logging them for debugging?**
→ Centralize via `formatError`: pass through known `AppError`/`GraphQLError` instances as-is (they're intentionally safe/user-facing), but for any *unexpected* error, log the full original error server-side (with request/user context) and replace the client-facing message with a generic "Internal server error" + a request/trace ID the client can report, so support/eng can correlate logs without exposing internals.

### F. Concurrency, Availability, Next.js Integration (45–50)
**45. Two users edit the same Lead concurrently — how do you prevent silent data loss?**
→ Optimistic locking with a `version`/`updatedAt` column checked in the `UPDATE ... WHERE id=? AND version=?`; zero rows affected → throw `ConflictError`, client re-fetches latest + merges or prompts the user to resolve manually, rather than blind last-write-wins overwrite.

**46. How do you make a `createLead` mutation safe against client retry-induced duplicates?**
→ Idempotency key (client-generated UUID) sent with the mutation; server stores key→result mapping (Redis, TTL'd) and returns the cached result on retry instead of re-executing the creation logic, so network retries/reconnects don't create duplicate records.

**47. Read replicas serve stale data right after a write — how do you avoid the "I just saved but don't see my own change" bug?**
→ Rely on the mutation's own response (normalization) to update the client cache rather than an immediate replica-read refetch, and/or pin the same request session to primary for a short window post-write, or introduce replica lag monitoring and route read-after-write-critical queries to primary explicitly.

**48. Why can't you run `useSubscription` in a Next.js Server Component?**
→ Subscriptions require a persistent WebSocket connection tied to a specific client — Server Components render once per request on the server and can't maintain a live, long-lived connection back to a specific browser; subscriptions are inherently a client-side, stateful concern and must live in a `'use client'` component.

**49. Why shouldn't you reuse a single module-level Apollo Client singleton across Server Component requests in Next.js, the way you would in the browser?**
→ On the server, a singleton client is shared across *all users'* concurrent requests in the same Node process — its cache and any per-user auth headers baked into the client instance would leak between users; each server-side request must construct its own short-lived client (or at least a fresh cache) scoped to that request's auth context.

**50. How does GraphQL Codegen reduce runtime bugs beyond "just having types"?**
→ Generates types directly from the live/committed schema plus your actual `.graphql` operation documents, so any mismatch between a query's selection set and the real schema (renamed/removed field, changed nullability) is caught as a **compile-time TypeScript error**, not a runtime `undefined` bug in production; running codegen in CI as a schema-diff check also fails builds the moment a backend schema change would break an existing client query, catching breaking changes before merge rather than after deploy.

---

**Wednesday-day tip:** Given your real Eldermark Apollo history, expect the interviewer to probe *why* decisions were made ("why `update()` over `refetchQueries` here"), not just definitions — lean on the trade-off framing above (perf vs correctness vs complexity) rather than reciting facts. The sections most likely to separate "refreshed basics" from "lead" are **§1 (typePolicies/keyArgs internals)**, **§3 (multi-instance PubSub scaling)**, and **§5 (directive-based authz nuance)** — re-read those twice.
