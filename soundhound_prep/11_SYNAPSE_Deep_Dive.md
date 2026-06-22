# SYNAPSE — AI Knowledge Graph & Reasoning Engine (Deep Dive)

> **For**: Complete refresher. Understand everything you built.
> **Goal**: Answer any interview question about SYNAPSE's architecture, LangGraph pipeline, ingestion, or budget system.

---

## What is SYNAPSE?

A self-updating AI knowledge graph that ingests from 15+ sources (arXiv, HuggingFace, GitHub, Semantic Scholar) and answers complex reasoning queries via an 8-node LangGraph pipeline.

**User flow**: Ask a question → system searches its knowledge graph + web → runs 8-step reasoning → returns synthesized answer with citations + RAGAS quality scores.

---

## High-Level Architecture

```mermaid
flowchart TB
    User["User"] --> Frontend["React SPA<br/>14 routes"]
    Frontend --> API["FastAPI Backend<br/>21 endpoints"]
    
    API --> REASON["Reasoning Pipeline<br/>8-node LangGraph"]
    API --> KG["Neo4j Knowledge Graph<br/>19 node types, 23 relationships"]
    API --> VS["pgvector<br/>384-dim embeddings<br/>HNSW index"]
    API --> BUDGET["BudgetOracle<br/>Per-model token tracking"]
    
    subgraph Ingest["Daily Ingestion (15 sources)"]
        SF["Source Factory"] --> FETCH["Parallel Fetch"]
        FETCH --> EXTRACT["Fast-Path Extraction"]
        EXTRACT --> REL["Relationship Rules"]
        REL --> NEO4J["Neo4j Batch MERGE"]
        NEO4J --> EMB["Embedding Generation"]
        EMB --> SIM["Semantic Similarity Edges"]
    end
    
    KG --> Ingest
    VS --> Ingest
    
    BUDGET --> GG["Groq<br/>6 models"]
    BUDGET --> GF["Gemini<br/>Shared TPM pool"]
    BUDGET --> LOCAL["Local BM25<br/>Zero cost"]
```

---

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Backend | FastAPI | Async, WebSocket, auto docs |
| Frontend | React 19, Vite 6, Tailwind v4, Sigma.js | Graph visualization, fast builds |
| Knowledge Graph | Neo4j | Graph DB for entity-relationship queries |
| Vector Store | pgvector (Neon Postgres) | halfvec(384), HNSW index on cosine |
| Embeddings | all-MiniLM-L6-v2 (384-dim, PyTorch) | Free, runs on CPU |
| Embeddings (fast) | gte-small via ONNX Runtime | 2-3x faster, opt-in via env var |
| LLM Primary | Groq (multi-key, 6 models) | Fast inference, good free tier |
| Orchestration | LangGraph (StateGraph) | 8-node state machine with conditional edges |
| Storage | Firestore (checkpoints), DynamoDB (budget), SQS (jobs) |
| MCP | Memory, Sequential Thinking, Filesystem servers via stdio |
| CI/CD | GitHub Actions (daily ingest, 2-hourly scrape, weekly eval) |
| Deployment | Cloud Run + Firebase Hosting |

---

## Project Structure

```
synapse/
├── api/                   # FastAPI application
│   ├── main.py           # 114 lines. App creation, lifespan events, routes
│   ├── middleware.py      # 98 lines. CORS, rate limiting (30/min/IP), security headers
│   ├── v1/
│   │   ├── router.py     # 869 lines. 14 data endpoints
│   │   ├── reasoning.py  # 393 lines. 7 reasoning pipeline endpoints
│   │   └── groq_status.py # 72 lines. Admin: key health, model status
│   ├── groq_manager.py   # 476 lines. Multi-key Groq management
│   └── semantic_cache.py # 124 lines. pgvector query cache (0.95 threshold)
├── schema/               # Data models + config
│   ├── config.py         # 140 lines. Pydantic Settings (40+ env vars)
│   ├── models.py         # 88 lines. GraphNode, GraphEdge, FactTier, etc.
│   ├── domain_loader.py  # 29 lines. Load domain packs from domains/<name>/
│   └── setup.py          # 37 lines. Create Neo4j schema from domain pack
├── domains/
│   └── ai/               # AI domain pack
│       ├── schema.yaml   # 19 node types, 23 relationship types
│       ├── sources.yaml  # 15 free-tier data sources
│       └── aliases.jsonl # 213 technique/model name aliases
├── config/thresholds.yaml # 31 lines. Runtime thresholds (confidence, retries, budget)
├── reasoning/            # LangGraph pipeline
│   ├── graph/
│   │   ├── builder.py    # 201 lines. Load YAML → build StateGraph
│   │   ├── state.py      # 55 lines. ReasoningState TypedDict
│   │   ├── checkpoint.py # 84 lines. Firestore checkpointing
│   │   └── definitions/default.yaml # 96 lines. Graph topology
│   ├── nodes/            # 8 node functions
│   │   ├── entry.py           # Budget check, session init
│   │   ├── decomposition.py   # GPT-OSS 20B query breakdown
│   │   ├── retrieval.py       # 4-tier hybrid retrieval
│   │   ├── analysis_crew.py   # Extract + Analyze + Contradict
│   │   ├── synthesis.py       # Llama 3.3 70B answer gen
│   │   ├── critic.py          # GPT-OSS 20B quality eval
│   │   └── output.py          # Final assembly + RAGAS
│   └── subagents/
│       ├── web_research.py    # 254 lines. DDG + Tavily + Crawl4AI
│       └── manager.py         # 71 lines. Subagent spawning
├── ingestion/            # Data pipeline
│   ├── generic_source.py # Universal fetcher (JSON, XML, RSS)
│   ├── source_factory.py # Load YAML config, create fetchers
│   ├── circuit_breaker.py # Per-source, persisted to JSON
│   ├── neo4j/
│   │   ├── client.py     # Neo4j driver singleton
│   │   └── writer.py     # Batch MERGE (200/doc, dedup keys)
│   └── pipeline/
│       └── run.py        # 9-stage orchestrator + relationships + embeddings
├── embedding/
│   ├── generator.py      # all-MiniLM-L6-v2, 384-dim
│   ├── onnx_generator.py # gte-small via ONNX (faster, opt-in)
│   └── qdrant_client.py  # pgvector async client (misnamed, kept for compat)
├── retrieval/
│   ├── query_engines.py  # vector, BM25, graph, hybrid queries
│   └── session_index.py  # Per-session in-memory index
├── budget/
│   ├── oracle.py         # BudgetOracle: gate before every LLM call
│   ├── register.py       # Per-model RPM/TPM/RPD budgets
│   ├── scheduler.py      # LeakyBucketScheduler convoy control
│   ├── fallback_chains.yaml # Per-task fallback model chain
│   ├── dynamodb.py       # Budget state persistence
│   └── sqs_queue.py      # Job queue for async reasoning
├── providers/
│   ├── protocol.py       # InferenceProvider ABC
│   ├── groq_provider.py  # Groq implementation
│   └── local_provider.py # BM25 extractive fallback
├── prompt/
│   ├── assembler.py      # 5-layer prompt builder, budget trimming
│   └── roles/            # 6 role prompts (decomposition, analyzer, critic, etc.)
├── frontend/             # React SPA
│   └── src/
│       ├── pages/        # 14 routes
│       └── components/   # Layout + Reveal animation
├── sync/                 # 2-hourly background scraper (9 sources)
├── eval/                 # RAGAS monitoring
├── webhook/              # Event notification system
├── deploy/               # Cloud Run deployment
└── tests/                # 18 test files, 7 integration guards
```

---

## 8-Node LangGraph Pipeline

```mermaid
flowchart TB
    ENTRY["Node 1: Entry<br/>Budget gate + session init"] --> DECOMP["Node 2: Decomposition<br/>GPT-OSS 20B<br/>Break query into sub-questions"]
    DECOMP --> RET["Node 3: Retrieval<br/>4-tier hybrid search"]
    DECOMP --> WEB["Node 4: Web Research<br/>DDG + Tavily + Crawl4AI"]
    RET --> ANALYSIS["Node 5: Analysis Crew<br/>Extractor 8B → Analyzer Scout → Contradiction Detector"]
    WEB --> ANALYSIS
    ANALYSIS --> SYNTH["Node 6: Synthesis<br/>Llama 3.3 70B<br/>Markdown answer"]
    SYNTH --> CRITIC["Node 7: Critic<br/>GPT-OSS 20B<br/>Grounding + Completeness + Logic"]
    CRITIC -->|"pass ≥ 0.8"| OUTPUT["Node 8: Output<br/>Final assembly + RAGAS eval"]
    CRITIC -->|"fail + retry < 2"| ANALYSIS
    CRITIC -->|"fail + retry ≥ 2"| OUTPUT
```

### Node 1: Entry
**What**: Budget gate check + session initialization.
- Calls BudgetOracle.can_afford() for estimated tokens
- Generates UUID session_id
- Estimates complexity (low/medium/high from query length)
- If budget insufficient → status = FAILED immediately

### Node 2: Decomposition (GPT-OSS 20B)
**What**: Breaks complex query into simpler sub-questions.

```
Query: "Compare LoRA vs QLoRA for fine-tuning LLMs"
→ Sub-questions:
  1. "How does LoRA work for fine-tuning?"
  2. "How does QLoRA differ from LoRA?"
  3. "What are the memory requirements for each?"
  4. "Which performs better on downstream tasks?"
→ 6 search queries with type + priority
→ retrieval_strategy: hybrid
→ merge_strategy: comparative
```

### Node 3: Retrieval (4-tier)
**What**: Searches across multiple sources, fastest first.

```
Tier 1: Neo4j KG query (0 tokens, O(1) graph traversal)
  → "Find all Techniques related to 'fine-tuning'"
  → Returns: LoRA node, QLoRA node, related papers

Tier 2: Cross-session pgvector cache (1 embedding, 0.85 threshold)
  → "Has someone asked about LoRA vs QLoRA before?"
  → Returns: cached answer if similarity ≥ 0.85

Tier 3: Vector + BM25 hybrid (1 embedding + BM25 scoring)
  → Vector weight 0.6, BM25 weight 0.4
  → Top 30 results, fused by score

Tier 4: Session index (in-memory, zero cost)
  → Documents uploaded in current session
  → Simple substring match
```

### Node 4: Web Research (only if retrieval confidence < 0.65)
**What**: Fetches live data from web. Runs in parallel with retrieval.

**Sources tried in order**:
1. **DuckDuckGo** (AsyncDDGS, free, no API key)
2. **Tavily** (API key, priority-ranked, better quality)
3. **Crawl4AI** (Playwright-based, full JS rendering)
4. **ZenRows** (anti-bot bypass fallback)
5. **aiohttp** (final fallback, simple requests)

**Dedup**: Tavily results preferred. URL dedup. Content stored in SessionIndex.

### Node 5: Analysis Crew (3 sub-phases)

**Phase 1 — Extractor** (llama-3.1-8b):
- Takes retrieved content (text, web pages, KG results)
- Extracts factual claims as JSON list
- Each claim: {claim_text, source, confidence, date}

**Phase 2 — Analyzer** (Llama 4 Scout):
- Maps claims to evidence
- Resolves source conflicts by FactTier (T1=system, T2=cross-verified, T3=single source, T4=inferred)
- Produces structured claim-evidence map

**Phase 3 — Contradiction Detector** (CrewAI agent or fallback):
```
Input: "Source A says LoRA needs 8GB. Source B says 16GB."
Output: {
  "contradictions": [{
    "claim_a": "LoRA requires 8GB VRAM",
    "claim_b": "LoRA requires 16GB VRAM",
    "verdict": "CONTRADICT",
    "conflict_severity": "medium",
    "explanation": "Difference may be due to model size (7B vs 13B)"
  }]
}
```

**Subagent spawning**: For "high" complexity sub-questions, spawns isolated subgraphs (max 3, 45s timeout) using SubagentManager.

### Node 6: Synthesis (Llama 3.3 70B)
**What**: Produces structured Markdown answer.

**Output structure**:
```
## Summary
[1-2 paragraph overview]

## Key Findings
| Finding | Evidence | Confidence |
|---------|----------|------------|
| LoRA uses rank decomposition | Source A, B, C | High |
| QLoRA adds 4-bit quantization | Source D, E | High |

## Contradictions Detected
[if any]

## Knowledge Gaps
[what wasn't found]

## Sources
[deduplicated, top 10]
```

### Node 7: Critic (GPT-OSS 20B)
**What**: Evaluates synthesis quality on 3 axes.

```
Grounding (≥0.8): Are claims traceable to sources?
Completeness (≥0.8): All sub-questions addressed?
Logic (≥0.8): Does reasoning hold together?

Overall: PASS / FAIL
If FAIL: specific feedback for retry
```

**Auto-pass**: If provider fails, auto-pass at minimal score (prevents infinite loop).

### Node 8: Output
**What**: Final assembly.
- Trims sources to top 10
- Drops duplicates
- Runs RAGAS evaluation (faithfulness, relevancy, precision, recall)
- Sets status = COMPLETE
- Returns final answer to user

---

## Budget System

```mermaid
flowchart LR
    subgraph Before["Before Every LLM Call"]
        O["BudgetOracle.can_afford()"] -->|"Yes"| S["LeakyBucketScheduler.acquire()"]
        O -->|"No"| F["Try fallback model"]
        F -->|"No fallback left"| L["Local BM25<br/>zero cost, always works"]
    end
    
    subgraph After["After LLM Call"]
        U["BudgetOracle.record_usage()"]
        U --> D["DynamoDB persist"]
        D --> SQ["SQS fallback"]
    end
```

**BudgetOracle** — singleton, consulted before every LLM call:
- `can_afford(task_type, estimated_tokens)` → walks fallback chain, finds first model with remaining budget
- `resolve_model(task_type)` → returns cheapest available model
- `record_usage(model_id, tokens_used)` → deduct from remaining
- **Prompt caching**: GPT-OSS models get 1/4 token cost on cached prefixes
- **Persists to DynamoDB**: Restores budget state on restart

**Fallback chains per task** (from `fallback_chains.yaml`):
```
decomposition: gpt-oss-20b → qwen3-32b → local
synthesis: llama-3.3-70b → gpt-oss-20b → local
critic: gpt-oss-20b → llama-3.1-8b → local
extraction: llama-3.1-8b → local
analysis: llama-4-scout → llama-3.1-8b → local
```

**Local provider**: BM25 extractive summarization. Tokenizes context, scores against task, returns top 3 passages. Zero cost, zero API calls, always works. Quality is lower but system never dead-ends.

---

## 21 API Endpoints

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/` | Service info + docs link |
| GET | `/health` | Neo4j node/edge counts |
| GET | `/api/v1/whats-new` | Entities from last N days |
| GET | `/api/v1/search` | Full-text search with type filter, cursor |
| GET | `/api/v1/similar` | Top-k pgvector similarity |
| GET | `/api/v1/export` | Subgraph export (JSON-LD, CSV, GraphML) |
| GET | `/api/v1/diff` | Temporal comparison |
| GET | `/api/v1/leaderboard` | Top tools/papers/models |
| GET | `/api/v1/technique/{name}/ecosystem` | 2-hop technique graph |
| GET | `/api/v1/org/{name}/releases` | Organization entities |
| GET | `/api/v1/model/{hf_id}/lineage` | Base model + fine-tunes |
| POST | `/api/v1/query` | NL-to-Cypher translation |
| GET | `/api/v1/query/suggestions` | Query auto-suggest |
| POST | `/api/v1/reason` | Submit reasoning query (async) |
| GET | `/api/v1/reason/{id}` | Poll reasoning result |
| GET | `/api/v1/reason/{id}/stream` | SSE stream of reasoning |
| POST | `/api/v1/ingest` | Upload document for session indexing |
| GET | `/api/v1/budget` | Per-model budget status |
| POST | `/api/v1/webhook/subscribe` | Subscribe to events |
| GET | `/api/v1/eval` | RAGAS evaluation summary |
| GET/POST | `/api/v1/groq/*` | Admin: key status, rotate, reset |

---

## Ingestion Pipeline (9 Stages)

```mermaid
flowchart LR
    S1["1. Source Factory<br/>Load 15 sources"] --> S2["2. Parallel Fetch<br/>asyncio.gather"]
    S2 --> S3["3. Fast-Path<br/>SourceDoc → GraphNode"]
    S3 --> S4["4. Relationships<br/>Rule-based edges<br/>NO LLM"]
    S4 --> S5["5. Neo4j Nodes<br/>Batch MERGE (200)"]
    S5 --> S6["6. Neo4j Edges<br/>Batch MERGE"]
    S6 --> S7["7. Embeddings<br/>all-MiniLM-L6-v2"]
    S7 --> S8["8. Semantic Similarity<br/>cosine ≥ 0.85"]
    S8 --> S9["9. Webhook Dispatch<br/>Notify subscribers"]
```

**15 sources**: arxiv, huggingface_daily_papers, huggingface_trending_models, papers_with_code, github_trending, semantic_scholar, dair_ai_ml_papers, towards_data_science, openai_blog, huggingface_blog, marktechpost, openalex, and more.

**Relationship extraction (NO LLM)**: Rule-based from 80-entry topic → technique map:
```
Paper about "transformer attention" → IMPLEMENTS → "Attention Mechanism"
Tool with github_repo containing "llm" → DEPENDS_ON → relevant techniques
Model named "Llama-3-70B" → FINE_TUNED_FROM → "Llama 3"
```

**Circuit breaker per source**: 3 failures → OPEN for 30 mins. Persisted to JSON file with fcntl.flock atomic writes.

---

## Frontend (14 Routes)

| Route | Page | What It Shows |
|-------|------|---------------|
| `/` | Dashboard | Live stats (nodes, edges, embeddings), system status, source ticker |
| `/search` | Search | BM25 + vector search, type filter, entity cards |
| `/ask` | Ask | NL query → shows generated Cypher → typed result cards |
| `/reason` | Reason | 7-stage pipeline viz, SSE streaming, rendered Markdown, RAGAS scores |
| `/graph` | Graph | Sigma.js WebGL interactive graph with node detail panel |
| `/diff` | Diff | Date picker → shows added/removed entities |
| `/leaderboard` | Leaderboard | 4 tabs: Tools/Papers/Techniques/LMArena |
| `/quality` | Quality | RAGAS metrics bars, last 10 evaluations |
| `/export` | Export | Cypher query → format selector → file download |
| `/budget` | Budget | Per-model RPM/RPD/TPM bars (green/amber/red) |
| `/ingest` | Ingest | Drag-and-drop file upload for session indexing |
| `/docs` | Docs | Swagger link + API reference table |
| `/about` | About | Feature highlights + attribution |

---

## Embedding System (Dual Model)

```mermaid
flowchart LR
    subgraph PyTorch["PyTorch Path (default)"]
        A["all-MiniLM-L6-v2<br/>384-dim<br/>~50ms per embedding"]
    end
    subgraph ONNX["ONNX Path (opt-in)"]
        B["gte-small<br/>384-dim<br/>~20ms per embedding<br/>(2-3x faster)"]
    end
    
    ENV["SYNAPSE_USE_ONNX=1"] -->|"set"| ONNX
    ENV -->|"unset"| PyTorch
    
    PyTorch --> C["PGVectorStore<br/>asyncpg + halfvec(384)<br/>HNSW on cosine<br/>connection pool (min=1, max=5)"]
    ONNX --> C
```

**Three embedding types**:
1. Paper: title + abstract (first 512 chars)
2. Entity: name + description (first 256 chars)
3. Query: raw query text

**PGVectorStore** (in `embedding/qdrant_client.py` — legacy name, but it's pgvector not Qdrant):
- `synapse_vectors` table: id TEXT, label TEXT, name TEXT, domain TEXT, embedding halfvec(384)
- HNSW index on cosine distance
- Connection pool: min=1, max=5
- Async (asyncpg) + sync wrappers for compatibility

---

## NL-to-Cypher Translation (`query/nl_to_cypher.py`)

Translates English questions into read-only Cypher queries.

```
User: "What models were released in 2025?"
→ Cypher: "MATCH (m:Model) WHERE m.release_date CONTAINS '2025' RETURN m.name, m.release_date LIMIT 50"
```

**Security**: Blocks write keywords (CREATE, DELETE, MERGE, DROP, SET, REMOVE) with word-boundary regex. Requires MATCH/WITH/CALL + RETURN. Auto-appends LIMIT 50.

**Schema caching**: Loads Neo4j schema (labels, properties, relationship types) with exponential backoff (3 retries). Formats into prompt with curated property lists per label.

**Cache**: 256 most recent query results. Evicts oldest half when full.

---

## Key Architecture Decisions

1. **Neo4j + pgvector dual storage**: Graph DB for relationships (who published what, what implements which technique). Vector DB for similarity search. Different tools for different jobs.

2. **LangGraph over LangChain chains**: Chains are linear. Graphs support cycles (critic → retry), branches (retrieval + web research in parallel), and conditional edges (critic pass/fail routing).

3. **Budget oracle before every LLM call**: Prevents surprise bills. Every call is gated by budget check + model resolution + concurrency enforcement. Combined with fallback chains, the system never overspends and never dead-ends.

4. **Rule-based relationships (NO LLM for ingestion)**: LLMs are expensive and unreliable for structured data. Ingestion relationships are determined by rules (keyword matching, ontology definitions, alias resolution). This is faster and more reliable.

5. **RAGAS evaluation baked into pipeline**: Every answer gets a quality score. This enables monitoring, alerting, and continuous improvement. Without eval, you can't know if changes make things better or worse.
