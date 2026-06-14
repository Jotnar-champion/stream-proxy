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
dont't give Too basic. Dive deeper into the system architecture and edge cases.
Focus heavily on the JWT rotation (Sliding sessions / Blacklisting) and Cache Consistency (Write-through vs. Cache-aside) answers it generates, as these are classic 8-year experience filters

special note: give both the list of questions and also sources to get deep and better understanding
