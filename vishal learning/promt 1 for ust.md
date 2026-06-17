You are an expert technical interview coach and AI/full-stack engineering mentor.

## CANDIDATE PROFILE
- 8 years full-stack experience
- Skills: React, Next.js, Node.js, AWS (Lambda, API Gateway, S3, SQS, SNS, DynamoDB, CloudWatch), Microservices, Serverless Architecture, RDBMS (PostgreSQL/MySQL), NoSQL (DynamoDB, Elasticsearch)
- Interviewing for: AI-inclined full-stack engineering role at UST Global
- Key Project: Full-stack AI-powered intelligence platform for NSE (National Stock Exchange) market
  - Stack: Next.js (frontend), FastAPI (backend)
  - Uses structured datasets and prompt-driven analysis for contextual market analysis
  - Intelligent design support and retrieval-based insights
  - Built scalable data ingestion and analytics pipeline consuming NSE and SEBI public datasets
  - AI layer built using Claude (Anthropic) as the primary LLM
  - Techniques used: RAG (Retrieval Augmented Generation), Embeddings, VectorDB, Prompt Engineering
  - Likely stack for AI layer: Claude API (claude-3-opus / claude-3-5-sonnet), LangChain or LlamaIndex, pgvector or Pinecone as VectorDB, sentence-transformers or Cohere embed for embeddings, FastAPI as AI orchestration layer

## YOUR TASK
For EACH topic listed below, do the following:

1. **Search the internet** — cover Reddit (r/MachineLearning, r/LocalLLaMA, r/node, r/aws), Stack Overflow, Medium blogs, Towards Data Science, Google AI Blog, Microsoft Research Blog, Anthropic Blog, OpenAI Blog, AWS Blog, MDN Web Docs, Hugging Face docs, LangChain docs, arXiv AI research papers, dev.to, and any other authoritative resources.

2. **Generate 20-30 most commonly asked interview questions** for that topic (covering beginner to advanced).

3. **Generate 20 deep-dive, real-world edge case questions** — think scenarios like:
   - "How do you handle a Lambda function that times out due to a slow downstream dependency?"
   - "How do you prevent race conditions when 2 users simultaneously update the same DB record in Node.js?"
   - "How would your RAG pipeline handle a query for which no relevant chunks exist in the VectorDB?"

4. **For EVERY question**, provide:
   - A **detailed, expert-level answer**
   - **Code snippets** wherever applicable (Node.js, Python, TypeScript, AWS CDK/CloudFormation, SQL, etc.)
   - Real-world context tied to the candidate's project (NSE market AI platform) where relevant

5. **At the end of each topic**, provide **2-3 must-read study resources** with URLs — prefer official docs, research papers, and high-quality blogs.

---

## TOPICS TO COVER

### TOPIC 1: Idempotency
- REST API idempotency (GET, PUT, DELETE, POST)
- Idempotency keys in payment APIs and distributed systems
- Handling idempotency in AWS Lambda (SQS triggers, exactly-once processing)
- DynamoDB conditional writes for idempotent operations
- Database-level idempotency patterns
- Real-world: idempotent data ingestion pipeline for NSE stock data

### TOPIC 2: Knowledge Base (in AI/LLM context)
- What is a knowledge base in the context of LLMs
- RAG vs fine-tuning vs knowledge base
- Amazon Bedrock Knowledge Base
- Building a custom knowledge base with embeddings
- Chunking strategies (fixed-size, semantic, recursive)
- Knowledge base for NSE/SEBI structured financial data

### TOPIC 3: Transformer Model
- Transformer architecture (attention, self-attention, multi-head attention, positional encoding)
- Encoder-only vs decoder-only vs encoder-decoder models
- BERT vs GPT architecture differences
- Tokenization (BPE, WordPiece, SentencePiece)
- Context window and token limits
- How transformers power LLMs like Claude and GPT

### TOPIC 4: RAG (Retrieval Augmented Generation)
- RAG pipeline architecture end-to-end
- Chunking, embedding, storing, retrieving, re-ranking
- Naive RAG vs Advanced RAG vs Modular RAG
- Hybrid search (vector + keyword BM25)
- Re-ranking with cross-encoders
- Handling hallucinations in RAG
- Evaluating RAG (RAGAS metrics: faithfulness, answer relevancy, context precision)
- RAG for structured financial data (NSE stock prices, SEBI filings)
- LangChain vs LlamaIndex for RAG
- Real-world: RAG pipeline over NSE dataset using FastAPI + pgvector + Claude

### TOPIC 5: Neural Networks
- Perceptron, layers, activation functions (ReLU, sigmoid, softmax)
- Backpropagation and gradient descent
- CNNs, RNNs, LSTMs and their use cases
- Overfitting, dropout, regularization
- Transfer learning
- How neural networks relate to LLMs

### TOPIC 6: Cursor AI
- What is Cursor AI and how it differs from GitHub Copilot
- Cursor's agent mode, chat mode, inline edit
- Using Cursor with custom codebases (.cursorrules)
- Cursor AI vs Copilot vs Windsurf for enterprise use
- Practical use in building the NSE AI platform

### TOPIC 7: Claude (Anthropic)
- Claude model family (Haiku, Sonnet, Opus, claude-3-5-sonnet-20241022)
- Claude's Constitutional AI and RLHF approach
- Claude API usage (messages API, system prompts, tool use / function calling)
- Claude vs GPT-4 vs Gemini — when to use which
- Claude for long-context document analysis (200K context window)
- Prompt caching with Claude for cost optimization
- Using Claude in the NSE market analysis platform

### TOPIC 8: LLM (Large Language Models) — General
- What are LLMs, how are they trained (pretraining, RLHF, instruction tuning)
- LLM inference (temperature, top-p, top-k, max_tokens)
- Prompt engineering patterns (zero-shot, few-shot, chain-of-thought, ReAct)
- LLM orchestration (LangChain, LlamaIndex, LangGraph)
- Fine-tuning vs RAG vs prompt engineering — when to use what
- LLM cost optimization strategies
- Open-source LLMs (Llama 3, Mistral, Phi-3) vs proprietary
- LLM security: prompt injection, jailbreaking, data leakage prevention

### TOPIC 9: AI Chatbot
- Architecture of a production AI chatbot
- Conversation memory (buffer memory, summary memory, vector memory)
- Multi-turn conversation design
- Streaming responses (Server-Sent Events, WebSocket)
- Building a chatbot for NSE market insights using Next.js + FastAPI + Claude
- Handling ambiguous or out-of-scope queries
- Chatbot evaluation metrics

### TOPIC 10: System Design for AI Applications
- Design a RAG-based document Q&A system (like for NSE/SEBI filings)
- Design a real-time stock market analysis AI platform (your exact project)
- Design a scalable LLM API gateway with rate limiting, caching, fallback
- Design an AI data ingestion pipeline (NSE public dataset → VectorDB)
- Async processing: SQS + Lambda for embedding generation at scale
- Caching LLM responses (semantic caching with Redis/Momento)
- Multi-tenancy in AI SaaS applications
- Observability for AI systems (LangSmith, Helicone, custom logging)

### TOPIC 11: VectorDB
- What is a VectorDB and why it's needed for AI
- Vector similarity search (cosine similarity, dot product, euclidean)
- ANN algorithms (HNSW, IVF, PQ)
- pgvector vs Pinecone vs Weaviate vs Qdrant vs ChromaDB comparison
- Metadata filtering in VectorDB
- Hybrid search (dense + sparse vectors)
- VectorDB performance tuning at scale
- Using pgvector with PostgreSQL for NSE data embeddings

### TOPIC 12: Hallucination in LLMs
- What is hallucination and why it happens
- Types: factual, reasoning, source hallucination
- Mitigation strategies: RAG, grounding, citation enforcement, output validation
- Hallucination detection tools (Guardrails AI, RAGAS, TruLens)
- Prompt engineering to reduce hallucination
- Real-world: preventing hallucination in NSE financial analysis responses

### TOPIC 13: Gemini (Google)
- Gemini model family (Nano, Flash, Pro, Ultra, 1.5 Pro)
- Gemini vs Claude vs GPT-4 benchmarks and use cases
- Gemini multimodal capabilities
- Google Vertex AI integration
- When would you choose Gemini over Claude for a project?

### TOPIC 14: GPT (OpenAI)
- GPT-4 vs GPT-4o vs GPT-4-turbo differences
- OpenAI API (chat completions, assistants API, function calling, vision)
- OpenAI Embeddings (text-embedding-3-small, text-embedding-3-large)
- Fine-tuning GPT models
- OpenAI vs Anthropic Claude — practical comparison

### TOPIC 15: Embedding Models & Text Embedding
- What are embeddings and how they work
- Sentence transformers (all-MiniLM, all-mpnet, BGE, E5)
- OpenAI text-embedding-3 vs Cohere embed v3 vs Voyage AI
- Embedding dimensions and trade-offs
- Chunking strategies impact on embedding quality
- Embedding financial text (NSE stock data, SEBI circulars)
- Bi-encoder vs cross-encoder for retrieval and re-ranking
- Evaluating embedding quality (MTEB benchmark)

---

## CORE FULL-STACK QUESTIONS (from candidate's background)

Also generate 20 deep-dive questions with answers for EACH of these:

### Node.js / Express
Starting example question (answer this too):
"Write an Express.js REST API with:
1. JWT authentication middleware
2. Global error handling middleware  
3. Request validation (using Zod or Joi)
4. Rate limiting
5. Async error propagation"
Then 20 more edge case real-world Node.js questions.

### AWS Serverless
- Lambda cold starts, provisioned concurrency, SnapStart
- Lambda + SQS: exactly-once processing, DLQ, visibility timeout
- API Gateway: throttling, caching, Lambda proxy integration
- Step Functions for AI orchestration workflows
- EventBridge for event-driven microservices

### DynamoDB
- Single-table design patterns
- GSI vs LSI trade-offs
- DynamoDB Streams + Lambda for event sourcing
- Conditional writes for optimistic locking
- DynamoDB for storing LLM conversation history

### Elasticsearch
- Inverted index, relevance scoring (BM25)
- Elasticsearch for hybrid search in RAG pipelines
- Index mapping, analyzers, tokenizers
- Elasticsearch vs OpenSearch

### React / Next.js
- Server Components vs Client Components
- Next.js App Router architecture
- Streaming UI with React Suspense + LLM streaming responses
- SWR vs React Query for data fetching
- Building real-time stock dashboard with Next.js

### FastAPI (for AI project context)
- FastAPI async endpoints
- Background tasks for async embedding generation
- FastAPI + LangChain integration
- Dependency injection in FastAPI

---

## OUTPUT FORMAT
For each topic, structure output as:

### [TOPIC NAME]

#### Common Interview Questions (20-30)
Q1: [Question]
**Answer:** [Detailed answer with code if applicable]

...

#### Deep-Dive Real-World Edge Case Questions (20)
Q1: [Scenario-based question]
**Answer:** [Detailed answer with code]

...

#### Must-Read Study Resources
1. [Title] — [URL] — [Why it's useful]
2. ...
3. ...

---

Make sure ALL answers are production-grade, include TypeScript/Python code where relevant, and reference the candidate's NSE AI platform project where it naturally fits. Search the internet for the latest information, especially for model comparisons and benchmarks (as of 2025-2026).
