# simple-proxy

Simple reverse proxy to bypass CORS, used by [movie-web](https://movie-web.app).
Read the docs at https://docs.movie-web.app/proxy/introduction

---

### features:
 - Deployable on many platforms - thanks to nitro
 - header rewrites - read and write protected headers
 - bypass CORS - always allows browser to send requests through it
 - secure it with turnstile - prevent bots from using your proxy

> [!WARNING]
> Turnstile integration only works properly with cloudflare workers as platform

### supported platforms:
 - cloudflare workers
 - AWS lambda
 - nodejs
 - netlify edge functions

 - https://docs.google.com/document/d/1BVvmzGaDvYws0paWDawOKVCsM078jobMzNnmqS9uHhc/edit?usp=sharing



--------------------------------------------------------------------------------------------------------

Act as an elite Staff Engineer and Technical Interview Coach. I have a critical 1st-round interview in 2 days for a Senior/Lead Full-Stack role (Next.js, Node.js, AWS, PostgreSQL/SQL Server). I am an 8-year enterprise developer, so DO NOT explain basic concepts like closures, React hooks, the event loop, Promises, async/await, IAM basics, or basic SQL. 
Go online right now and crawl Reddit (e.g., r/cscareerquestions, r/javascript, r/aws), Glassdoor, LeetCode discuss forums, and tech blogs to find senior-level interview questions, real-world architectural traps, and highly focused resources for Marlabs or similar enterprise mid-to-senior rounds. 
Provide me with a highly concentrated, advanced cheat sheet broken into these specific sections:
1. THE "TRAP" QUESTIONS (Deep Technical Curveballs)
Give me 5-6 advanced questions designed to trap a senior developer on things they think they know. 
- Example: Edge cases in JS memory leaks despite using async/await; hidden bottlenecks in Next.js App Router streaming; race conditions in Node.js event-driven architectures. Provide the question and the exact, deep technical answer/solution.
2. REAL-WORLD SCENARIO DILEMMAS & ARCHITECTURE (Problem + In-depth Solution)
Provide deep, production-grade solutions for these exact scenarios:
- JWT Security: What is the absolute best enterprise-grade pattern if an access token is compromised/stolen or expired? How do we safeguard against replay attacks without destroying performance?
- Distributed Caching Strategy: Client-side vs. Server-side (Redis) caching. What are the major pitfalls (cache avalanche, stampede, consistency issues)? How do we handle high-throughput read/write optimization without stale data?
- Technology Trade-offs: Give me a sharp, definitive architectural comparison and "Why" justification for:
  * AWS API Gateway vs. Nginx (When to use which at enterprise scale?)
  * Next.js App Router vs. Standard Client-Side React (Architectural and SEO/performance trade-offs).
3. CRASH COURSE STUDY SOURCES (No long videos)
I have 2 days. Do not link 4-5 hour tutorials. Provide links/names of high-signal, high-density resources: senior-targeted cheat sheets, GitHub repositories with production-grade Next.js/Node/AWS boilerplates, or deep-dive technical blogs/articles that I can read in 15 minutes to upgrade my mental models.
4. 20 MUST-DO JS & UI/UX CODING ROUND REFRESHERS
List 20 specific front-end/UI coding challenges or LeetCode-style JS questions that test deep JS concepts (e.g., building a custom debounce/throttle with trailing/leading options, custom Promise.all/race, virtualized lists, recursive DOM traversal, form builders with state validation). Just list the problems and the core JS/UI trap to watch out for.
5. DSA PATTERN QUICK-REFRESH
Give me 1-2 high-yield, senior-level questions for each of these patterns to practice live coding resilience:
- Sliding Window
- Two Pointer
- Hashmap / Frequency Counter
- Kadane's Algorithm
For each, state the optimal time/space complexity and the core "trick" to solving it under pressure.



-----------------------------------------------------------------------------------------------------------------------

You are an expert Principal Database Architect and Technical Interview Coach. Your task is to create an exhaustive, highly structured interview preparation guide tailored for a Senior Full-Stack Developer with 8 years of experience. 

The tone should be professional, clear, and direct—balancing deep technical precision with actionable interview strategies. Avoid high-level summaries; provide concrete, production-grade explanations and deep architecture insights.

---

### SECTION 1: The SQL Mastery Matrix
Provide a detailed breakdown of the following core SQL concepts. For each concept, you must include:
1. **The Core Mechanism:** A simple but technically precise explanation of how it works under the hood.
2. **When to Use It vs. Alternative Approaches:** Clear engineering trade-offs (e.g., Subquery vs. Join, HAVING vs. WHERE, UNION vs. UNION ALL).
3. **Decision Tree/Mental Model:** How a senior engineer instantly determines this is the right tool for the job.

**Concepts to Cover:**
* **Joins:** INNER, LEFT, RIGHT, FULL OUTER, SELF, and CROSS JOIN (Include visual data-flow alignment and performance implications like nested loops vs. hash joins).
* **Subqueries:** Correlated vs. Non-correlated (and when a CTE or JOIN is preferred).
* **GROUP BY & HAVING:** Deep dive on data aggregation phases (filtering before vs. after aggregation).
* **UNION vs. UNION ALL:** Memory allocation and duplicate elimination trade-offs.
* **Date Intervals & Time-Series Arithmetic:** Handling dynamic lookbacks (e.g., `INTERVAL 30 DAY`, `DATEDIFF`).

---

### SECTION 2: Production-Level Interview Problems (L3/Senior Level)
Provide exactly 10 highly realistic SQL query problems frequently asked in senior full-stack interviews. 
* **Schema Requirements:** Provide explicit `CREATE TABLE` DDL statements with appropriate data types, primary keys, and foreign keys.
* **Problem Variety:** Each of the 10 problems must explicitly highlight and test one or more concepts from Section 1.
* **For EACH of the 10 problems, you must provide:**
    1.  **The Problem Statement:** Complex business requirements (similar to the sample provided below).
    2.  **The Optimized SQL Query:** Clean, production-ready SQL using modern syntax.
    3.  **The Approach & Execution Order Strategy:** Explain *how* to talk through the solution during a live coding interview (e.g., "First, we filter the date range to minimize the dataset, then we aggregate...").
    4.  **Expected Interviewer Cross-Questions & Answers:** 2-3 aggressive follow-up questions an interviewer might ask to test edge cases (e.g., "What happens if two users have the exact same spend?", "How does this query scale if the table has 500 million rows? What indexes would you add?").

* *Sample Problem Style to Emulate:*
    * **Tables:** `users(user_id INT PRIMARY KEY, name VARCHAR(100))` and `orders(order_id INT PRIMARY KEY, user_id INT, order_date DATE, amount DECIMAL(10,2))`.
    * **Task:** Write a SQL query to return the top 10 customers by total spend in the last 30 days. Only include customers who placed at least 2 orders in the last 30 days. Return: `user_id`, `name`, `total_orders`, `total_spend`. Sort by `total_spend` DESC.

---

### SECTION 3: System Architecture, Distributed Systems & Database Scaling
Transition into advanced system design and database internals. Provide absolute clarity on the following architectural topics:

#### 1. CAP Theorem & Deep Business Usecases
* Explain the trade-offs between **Consistency, Availability, and Partition Tolerance**.
* Analyze how CAP applies to specific business application types:
    * **E-commerce (Cart & Inventory):** Highly available vs. strongly consistent trade-offs (e.g., checkout vs. browsing).
    * **Banking & Financial Ledgers:** Why CP is non-negotiable and how latency is handled.

#### 2. Sharding vs. Partitioning
* Define Horizontal Partitioning (Table-level) vs. Sharding (Database/Node-level).
* Explain the exact mechanics of how Sharding fits into the CAP theorem (e.g., how network partitions affect cross-shard queries).
* Detail standard sharding strategies (Hash-based, Range-based, Directory-based) and their respective pitfalls (hotspots, resharding complexity).

#### 3. ACID Properties & Transactional Isolation
* Provide a concrete breakdown of Atomicity, Consistency, Isolation, and Durability.
* Explain the 4 standard Transaction Isolation Levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable) and the specific phenomena they prevent (Dirty Reads, Non-repeatable Reads, Phantom Reads).

#### 4. Database Scaling & The SQL vs. NoSQL Dichotomy
* **Scaling Mechanics:** Differentiate between Vertical Scaling (Scale-Up) and Horizontal Scaling (Scale-Out). Explain the physical and architectural bottlenecks of both.
* **The Scaling Ease Matrix:** Explain why SQL databases are inherently easier to scale vertically but notoriously complex to scale horizontally (due to distributed Joins and ACID compliance), whereas NoSQL databases are designed for effortless horizontal scaling (due to denormalized data models and eventual consistency).
* **Architectural Decision Framework:** Provide a definitive guide on how to choose between SQL and NoSQL based on:
    * *Business Requirements:* Compliance, auditability, speed-to-market.
    * *Functional Requirements:* Complex relational reporting vs. simple key-value lookups.
    * *Technical Requirements:* Write-heavy vs. Read-heavy workloads, strict schema vs. dynamic JSON documents, data volume projections.
dont't give Too basic. Dive deeper into the system architecture and edge cases.
Focus heavily on the JWT rotation (Sliding sessions / Blacklisting) and Cache Consistency (Write-through vs. Cache-aside) answers it generates, as these are classic 8-year experience filters

special note: give both the list of questions and also sources to get deep and better understanding
