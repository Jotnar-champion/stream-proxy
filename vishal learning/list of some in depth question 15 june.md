



Fetched 3 resources

Fetched 2 resources





Fetched 3 resources

Fetched 3 resources

# Adobe CS2 Interview War Room — 2-Day Prep Plan

I've crawled Glassdoor (282 Adobe SWE interviews), Blind, LeetCode discuss, and cross-referenced with senior full-stack interview patterns. Here's your complete sparring guide. **I'll ask questions in each section — answer them out loud or in text and I'll drill you like a real interviewer.**

---

## What Adobe Actually Tests at CS2 Level (from real interview data)

Adobe CS2 interviews confirmed to include:
- **Coding round** (LC Medium/Hard — trees, graphs, arrays)
- **System Design** (1-2 rounds — scalability, real products like Experience Cloud)
- **Hiring Manager behavioral** (leadership principles, ownership, conflict)
- **Skills/Domain test** on your declared stack

---

## ROUND 1 — Next.js Deep Dive

> **Q1 — The Trap Question:**
> "Explain the difference between `getServerSideProps`, `getStaticProps`, and the App Router's `fetch` with `cache: 'no-store'` vs `revalidate`. Under what real business scenario would you pick each — and what breaks if you pick wrong?"

> **Q2 — Edge Case:**
> "Your Next.js app serves a dashboard that loads user-specific data. You used `getStaticProps` with ISR. A user says their data is stale for 10 minutes after they update their profile. What went wrong architecturally, and how do you fix it without sacrificing performance for the 99% of users who just read data?"

> **Q3 — Hydration Trap:**
> "Your senior dev pushed code that causes a React hydration mismatch in production only — works fine in dev. Walk me through how you'd debug this. What are the top 3 causes at an enterprise scale?"

---

## ROUND 2 — Node.js Internals & Production Traps

> **Q4 — Event Loop:**
> "You have a Node.js API that handles file uploads and simultaneously serves read requests. Under load, reads start timing out. You haven't hit CPU limits. What's happening? Walk me through the event loop, libuv thread pool, and how you'd diagnose and fix this."

> **Q5 — Memory Leak:**
> "Your Node service's heap keeps growing every 24 hours until it crashes. You restart it and it's fine. You have no obvious circular references. Name 5 non-obvious causes of memory leaks in Node.js production apps and how you'd identify each with specific tools."

> **Q6 — Clustering vs Worker Threads:**
> "When would you use Node.js `cluster` vs `worker_threads` vs just spinning up more pods in Kubernetes? A junior dev says 'just use cluster, it's the same thing.' What's your response?"

---

## ROUND 3 — Security (JWT / Auth — This Is a Guaranteed Topic)

> **Q7 — The Stolen JWT Scenario:**
> "A user's JWT access token gets stolen via XSS. Your tokens are valid for 15 minutes. What is your complete incident response and architectural remediation? Be specific — I want headers, storage strategy, rotation mechanism, and what you'd add to the system going forward."

> **Q8 — Refresh Token Rotation:**
> "Explain refresh token rotation. Now tell me: what happens if a legitimate user's refresh token request fails due to a network error right after the server has already rotated and invalidated the old token? How do you handle this without logging them out?"

> **Q9 — CORS Trap:**
> "Your API has `Access-Control-Allow-Origin: *`. A security auditor flags it critical. You tell the team to restrict it to your domain. Three days later, your mobile app team says their requests are failing. What's happening and what's the correct fix for a multi-client API?"

---

## ROUND 4 — Caching (Multi-Layer — They Love This)

> **Q10 — Cache Strategy Design:**
> "Design a caching strategy for a product page on an e-commerce platform. The page has: product description (changes rarely), stock count (changes every second), personalized recommendations (user-specific), and price (changes with promotions). Walk me through every cache layer you'd use and why."

> **Q11 — Cache Invalidation:**
> "You have Redis caching your DB queries with a 5-minute TTL. A bug in your invalidation logic causes stale prices to show. The fix is a cache flush. Your Redis has 2M keys. `FLUSHALL` would take down other services sharing the same Redis. How do you selectively invalidate without FLUSHALL?"

> **Q12 — CDN Edge Cache:**
> "A user in Europe says they see outdated content for 20 minutes after a publish. Your origin server shows the correct data immediately. Trace the full request path and explain every point where the stale data could be sitting. How do you purge programmatically?"

---

## ROUND 5 — AWS Architecture

> **Q13 — Real Architecture Trap:**
> "Your Next.js app on EC2 behind an ALB handles 10K req/s normally but spikes to 100K during a product launch. Your RDS PostgreSQL becomes the bottleneck. Walk me through your scaling strategy. I want specific AWS services, not buzzwords."

> **Q14 — S3 + CloudFront Security:**
> "You're serving user-uploaded documents via S3. Some documents are private (only visible to the owner). A junior dev says 'just make the S3 bucket public and check permissions in the app.' What are the 3 things wrong with this and what's the correct AWS-native solution?"

> **Q15 — Lambda Cold Start:**
> "You migrated an API route to Lambda. P99 latency went from 120ms to 800ms. The client is furious. Diagnose and fix. What's the difference between provisioned concurrency and reserved concurrency and when do you use each?"

---

## ROUND 6 — PostgreSQL / SQL Server Performance

> **Q16 — Slow Query:**
> "This query takes 12 seconds on a 50M row table:
> ```sql
> SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending' ORDER BY created_at DESC LIMIT 20;
> ```
> Walk me through your optimization process step by step. What does `EXPLAIN ANALYZE` tell you that `EXPLAIN` alone doesn't?"

> **Q17 — N+1 in Production:**
> "Your Node.js API fetches 100 posts and for each post fetches the author. You see 101 queries in your APM tool. How do you fix this in raw SQL, in an ORM like Prisma/TypeORM, and architecturally? What's the tradeoff of each approach?"

> **Q18 — Transaction Isolation Trap:**
> "Two concurrent requests both try to deduct from a user's account balance. Your code reads the balance, calculates the new balance, then writes it. Under what isolation level does this cause a race condition, and what's the correct fix without using pessimistic locking everywhere?"

---

## ROUND 7 — Full-Stack Performance Optimization (Adobe's Core Business)

> **Q19 — Core Web Vitals:**
> "Your LCP is 4.8s on mobile. Your CLS is 0.25. Your INP is 380ms. For each metric, name the top 2 causes specific to a Next.js app and the exact fix. Adobe cares deeply about this — they build tools that measure it."

> **Q20 — The Bundle Size Trap:**
> "Your Next.js bundle analyzer shows `moment.js` adding 68KB gzipped. You remove it. Two weeks later a different library pulled it back in as a transitive dependency. How do you prevent this at the CI/CD level permanently?"

---

## ROUND 8 — System Design (Expect This at CS2)

> **Q21 — Design Adobe Analytics Event Pipeline:**
> "Design a system that ingests 1 million page-view events per second from websites worldwide, processes them in near-real-time, and makes the data queryable within 5 seconds. What does your tech stack look like on AWS? Where does PostgreSQL fit vs a columnar store?"

---

## ROUND 9 — Behavioral (Adobe's HM Round)

Adobe specifically tests **ownership** and **cross-functional influence**:

> **Q22:** "Tell me about a time you disagreed with a technical decision made by a more senior engineer. What did you do and what was the outcome?"

> **Q23:** "Describe the most complex full-stack system you've owned end-to-end. What would you do differently?"

> **Q24:** "A PM wants a feature shipped in 2 weeks. Your estimate is 6 weeks due to security requirements. How do you handle this conversation?"

---

## Your 2-Day Battle Plan

| Day | Morning | Afternoon | Evening |
|-----|---------|-----------|---------|
| **Day 1** | Answer Q1-Q9 above (I'll drill you) | 2x LC Medium (array/string, tree) | System design: design a URL shortener with Next.js + Node + AWS |
| **Day 2** | Answer Q10-Q21 (I'll drill you) | Review JWT/OWASP Top 10 | Mock behavioral round, sleep early |

---

## Start Now

**Pick any question above and answer it.** I'll respond as your interviewer — pushing on edge cases, poking holes in your answer, and asking "what if the load is 10x more?" exactly like a real Adobe CS2 panel would.

Which topic do you want to get grilled on first?> SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending' ORDER BY created_at DESC LIMIT 20;
> ```
> Walk me through your optimization process step by step. What does `EXPLAIN ANALYZE` tell you that `EXPLAIN` alone doesn't?"

> **Q17 — N+1 in Production:**
> "Your Node.js API fetches 100 posts and for each post fetches the author. You see 101 queries in your APM tool. How do you fix this in raw SQL, in an ORM like Prisma/TypeORM, and architecturally? What's the tradeoff of each approach?"

> **Q18 — Transaction Isolation Trap:**
> "Two concurrent requests both try to deduct from a user's account balance. Your code reads the balance, calculates the new balance, then writes it. Under what isolation level does this cause a race condition, and what's the correct fix without using pessimistic locking everywhere?"

---

## ROUND 7 — Full-Stack Performance Optimization (Adobe's Core Business)

> **Q19 — Core Web Vitals:**
> "Your LCP is 4.8s on mobile. Your CLS is 0.25. Your INP is 380ms. For each metric, name the top 2 causes specific to a Next.js app and the exact fix. Adobe cares deeply about this — they build tools that measure it."

> **Q20 — The Bundle Size Trap:**
> "Your Next.js bundle analyzer shows `moment.js` adding 68KB gzipped. You remove it. Two weeks later a different library pulled it back in as a transitive dependency. How do you prevent this at the CI/CD level permanently?"

---

## ROUND 8 — System Design (Expect This at CS2)

> **Q21 — Design Adobe Analytics Event Pipeline:**
> "Design a system that ingests 1 million page-view events per second from websites worldwide, processes them in near-real-time, and makes the data queryable within 5 seconds. What does your tech stack look like on AWS? Where does PostgreSQL fit vs a columnar store?"

---

## ROUND 9 — Behavioral (Adobe's HM Round)

Adobe specifically tests **ownership** and **cross-functional influence**:

> **Q22:** "Tell me about a time you disagreed with a technical decision made by a more senior engineer. What did you do and what was the outcome?"

> **Q23:** "Describe the most complex full-stack system you've owned end-to-end. What would you do differently?"

> **Q24:** "A PM wants a feature shipped in 2 weeks. Your estimate is 6 weeks due to security requirements. How do you handle this conversation?"

---

## Your 2-Day Battle Plan

| Day | Morning | Afternoon | Evening |
|-----|---------|-----------|---------|
| **Day 1** | Answer Q1-Q9 above (I'll drill you) | 2x LC Medium (array/string, tree) | System design: design a URL shortener with Next.js + Node + AWS |
| **Day 2** | Answer Q10-Q21 (I'll drill you) | Review JWT/OWASP Top 10 | Mock behavioral round, sleep early |

---

## Start Now

**Pick any question above and answer it.** I'll respond as your interviewer — pushing on edge cases, poking holes in your answer, and asking "what if the load is 10x more?" exactly like a real Adobe CS2 panel would.

Which topic do you want to get grilled on first?
