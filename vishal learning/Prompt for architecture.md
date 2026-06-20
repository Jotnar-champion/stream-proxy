You are reverse-engineering this local codebase for me.

I am already good at reading and debugging code. I do NOT want a beginner explanation of programming. I want an architecture-first analysis with exact pointers to the important files, functions, and lines that actually matter.

My main goals are:
- understand the end-to-end architecture
- understand the UI to backend integration
- understand the RAG pipeline deeply
- understand ChromaDB / vector DB integration
- understand embeddings, retrieval, ranking, similarity scoring, and any weighting logic
- understand how prompts are built and passed to OpenAI
- understand how OpenAI responses are parsed, streamed, validated, and returned
- understand all DB-related behavior
- understand caching, ACID, concurrency, async behavior, locking, and transactional boundaries
- understand security boundaries and risks
- understand what code lines are “special” or “important” and deserve attention

Do not waste time explaining basic Python or basic JavaScript unless it is needed to understand a specific code path.

Important rules:
- Use the actual code in the repository.
- Reference exact file paths, function names, class names, route names, config keys, and key code blocks.
- When you see something important, call it out explicitly.
- Do not give a generic summary.
- Do not organize primarily by folders. Organize by runtime flow and dependencies.
- Separate clearly:
  1. confirmed from code
  2. likely based on code
  3. unknown or ambiguous
- If there is more than one implementation path, explain the differences.

What I want from you:

1) One end-to-end architecture view
Show the full path from:
UI interaction
→ frontend state / API call
→ FastAPI route
→ service layer
→ RAG orchestration
→ embedding generation
→ vector DB retrieval
→ prompt assembly
→ OpenAI call
→ response parsing/streaming
→ UI render

Give a clear architecture diagram and explain each hop.

2) Critical file map
List the most important files in execution order, not just by folder.
For each file explain:
- why it matters
- what it owns
- what calls it
- what it calls next
- which exact lines or code blocks are the key ones to inspect

3) RAG pipeline deep dive
Explain in detail:
- where ingestion starts
- how documents are loaded
- how chunking is done
- chunk size / overlap / split strategy
- how embeddings are generated
- what embedding model is used
- where vectors are stored
- how ChromaDB is initialized and configured
- collection names and metadata structure
- how retrieval works
- how similarity scores are used
- whether top-k, reranking, filtering, or hybrid retrieval exists
- whether any “weighting” logic exists and where it happens
- how retrieved context is assembled
- how the final prompt is constructed
- how the answer is produced and returned

For every stage give:
file path
function/class
important lines or logic
input
output
why it matters

4) OpenAI integration
Explain:
- where the OpenAI client is created
- what model(s) are used
- whether the response is streamed or buffered
- whether tool/function calling exists
- whether structured outputs exist
- how retries / failures / rate limits are handled
- how tokens/context length are managed
- how the response is transformed before returning to the UI

5) Database and persistence
Explain every storage layer used:
- SQL DB
- NoSQL DB
- ChromaDB / vector DB
- file storage
- any cache layer

For each one explain:
- what data it stores
- read/write paths
- schema/models/collections
- transactional boundaries
- where ACID matters
- how consistency is preserved
- how concurrent access is handled
- whether there is any locking, queuing, or async coordination

6) Caching
Find all caching behavior and explain:
- what is cached
- where it lives
- cache keys
- TTL / invalidation
- fallback behavior
- what happens on cache miss

7) Security
Review and explain:
- authentication
- authorization
- secrets and env vars
- CORS
- prompt injection protections
- SQL injection protections
- unsafe file handling
- rate limiting
- any user-input sanitization
- any AI-specific safety boundaries

8) Frontend integration
Explain:
- page/component structure
- state management
- how the UI triggers backend calls
- how responses come back into UI state
- where loading/error states are handled
- where streaming, partial responses, or citations are rendered if present

9) Concurrency and async behavior
Explain:
- async routes
- background tasks
- thread/process boundaries
- any queueing
- any parallel retrieval / parallel API calls
- any race conditions to watch out for
- what parts are safe vs fragile under concurrent requests

10) Important code lines to watch
At the end, give me a “watch list” of the most important code blocks or lines that deserve attention.
For each one say:
- file path
- line/function
- why it is critical
- what could break if it changes

11) Reading path
Give me the best reading order for understanding this codebase quickly.
Start with the most important execution path, not with folders.

12) Python guidance
When you explain Python files, only explain the Python details needed to understand architecture:
- imports
- modules
- packages
- __init__.py
- dependency injection patterns
- classes
- async functions

Do not teach Python basics beyond that.

I want the final output to be practical and concise enough to use as a working map of the codebase, not a tutorial.




==========================================================================================================================================================================



Now zoom in only on the RAG pipeline and ChromaDB flow. Show the exact files, functions, and key lines where retrieval scoring, weighting, and prompt assembly happen.







=============================================================================================================================================================




You are now acting as a local setup and debugging engineer for this repository.

I need to run this app on a new laptop that has:
- Node installed
- basic Python 3.x installed
- but I may not have everything else installed yet

I want exact, practical setup instructions based on this repository, not generic advice.

Important rules:
- Use the repo’s actual files: package.json, requirements.txt, pyproject.toml, uvicorn config, .env files, Docker files, migrations, seed scripts, and startup scripts.
- If there are multiple ways to run it, explain the preferred way first.
- If something is missing, say so directly.
- Do not assume Docker, pip packages, PostgreSQL, Redis, or any other DB is already available.
- Tell me exactly what to install, in what order, and what commands to run.
- Keep it practical and command-focused.

What I want:

1) Dependency inventory
Identify all runtime dependencies:
- frontend runtime
- backend runtime
- Python dependencies
- Node dependencies
- database dependencies
- vector DB dependencies
- cache dependencies
- any external services like OpenAI

2) Startup path
Find the real startup path for both frontend and backend:
- main entry files
- app initialization
- env loading
- router registration
- DB initialization
- ChromaDB initialization
- OpenAI client initialization
- any migrations or bootstrapping
- how the frontend talks to the backend

3) Exact local setup steps
Assume a clean machine and give me the full installation order:
- Node version / package manager needs
- Python virtualenv setup
- pip or poetry or uv install steps
- any OS-level dependencies
- how to install or start databases
- how to start ChromaDB or its persistence layer
- how to create the .env file
- how to set required environment variables
- how to run migrations
- how to seed data if needed
- how to start backend
- how to start frontend
- how to verify the app is working

4) DB and vector DB setup
Explain exactly how to set up:
- SQL DB if present
- ChromaDB / vector store
- any cache store like Redis if present

For each one, tell me:
- whether it is local, containerized, or managed
- where the config lives
- how persistence works
- what data needs to exist before the app can run

5) Failure points
Tell me the most likely setup failures and where to look first:
- missing env vars
- wrong Python version
- missing npm packages
- wrong DB URL
- ChromaDB connection issues
- OpenAI key issues
- CORS or frontend-backend mismatches
- migration failures
- vector store empty / ingestion missing

6) Debug checklist
Give me a direct checklist for these cases:
- backend starts but AI responses fail
- frontend loads but API calls fail
- vector DB returns nothing
- prompts are malformed
- DB transactions fail
- caching behaves unexpectedly

7) Exact commands
Give me the actual commands I should run, grouped in order.

8) Reading path for setup
Tell me which files to read first if I want to understand how setup works:
- package files
- environment files
- startup files
- DB config
- vector DB config
- ingestion scripts
- route registration
- frontend API client

9) Important notes
Call out any hidden assumptions in the repo:
- hardcoded ports
- local paths
- required services
- default model names
- default collection names
- persistence directories
- platform-specific issues

Keep the answer direct and runnable. I do not want theory unless it explains a real setup or debugging decision.
