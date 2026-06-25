# RAG & Agentic AI — From First Principles to Production

> **Target**: Knows what LLMs are. No knowledge of RAG or agents.
> **Goal**: Deeply understand every component of retrieval-augmented generation and building AI agents.

---

# PART 1: RAG (Retrieval-Augmented Generation)

## 1. What Problem RAG Solves

### The Three Problems

1. **Knowledge cutoff**: GPT-4 knows nothing after its training date. Ask about yesterday's news → hallucination.
2. **Private data**: The model hasn't read your company's internal docs, your codebase, your customer history.
3. **Hallucination**: Without facts to ground on, the model makes things up.

### The RAG Solution

Instead of asking the model to answer from memory, you:
1. Take the user's question
2. Search your knowledge base for relevant documents
3. Give the model: "Here are relevant docs + the user question. Answer using ONLY these docs."

The model can't hallucinate (much) because it has the facts in front of it.

### Analogy

**Without RAG**: Closed-book exam. The student (LLM) answers from memory. Gets things wrong.
**With RAG**: Open-book exam. Give the student the relevant textbook pages. Answer from those pages.

### Why This Matters
RAG is the most common production LLM pattern. ROAST's entire analysis pipeline IS RAG — it retrieves market intelligence, then generates a review grounded in that data.

---

## 2. Full RAG Pipeline — Every Step

```
User Query
    ↓
1. Query Processing (rewrite, expand, decompose)
    ↓
2. Retrieval (search knowledge base)
    ├── Dense (embedding similarity)
    └── Sparse (BM25 keyword match)
    ↓
3. Fusion (combine results from multiple retrievers)
    ↓
4. Reranking (sort by relevance)
    ↓
5. Context Assembly (build prompt with retrieved docs)
    ↓
6. LLM Generation (answer grounded in context)
    ↓
7. Citation / Verification
    ↓
Answer
```

### Why This Matters
Every step matters. Skip reranking? Quality drops. Bad chunking? Model can't find the right info. No fusion? Miss relevant docs. This pipeline is what makes RAG work vs just pasting random docs into a prompt.

---

## 3. Document Loading

### What It Means

Turning various file formats into clean text the system can process:

```python
# PDF loading with PyMuPDF
import fitz
def load_pdf(path):
    doc = fitz.open(path)
    text = ""
    for page in doc:
        text += page.get_text()
    return text

# Web page loading
import requests
from bs4 import BeautifulSoup
def load_url(url):
    resp = requests.get(url)
    soup = BeautifulSoup(resp.text, 'html.parser')
    # Remove script/style tags
    for tag in soup(['script', 'style']):
        tag.decompose()
    return soup.get_text()

# Database loading
import sqlite3
def load_from_db(query):
    conn = sqlite3.connect('market_intel.db')
    rows = conn.execute(query).fetchall()
    return [str(row) for row in rows]
```

### Why This Matters
Garbage in = garbage out. If your PDF extractor misses half the text, your RAG system fails regardless of how good your LLM is.

---

## 4. Text Extraction and Cleaning

### What Needs Cleaning

```python
def clean_text(raw):
    # Remove HTML tags
    text = re.sub(r'<[^>]+>', '', raw)
    
    # Normalize whitespace
    text = re.sub(r'\s+', ' ', text)
    
    # Remove headers/footers (heuristic: short lines repeated on many pages)
    lines = text.split('\n')
    cleaned = [l for l in lines if len(l) > 20]
    
    # Decode HTML entities
    text = html.unescape(' '.join(cleaned))
    
    return text.strip()
```

### Why This Matters
PDFs and web pages contain noise (navigation, ads, headers). This noise, if chunked and embedded, dilutes retrieval quality.

---

## 5. Chunking — What and Why

### The Problem

You can't put an entire book into an LLM's context window. But you can't split too small either — a sentence might lack context.

**Chunking = splitting documents into manageable, meaningful pieces.**

### Key Rule
Chunks should be **self-contained**. A chunk about "sorting algorithms" shouldn't reference "the above table" from a different chunk.

---

## 6-10. Chunking Strategies

### 6. Fixed-Size Chunking

```python
def fixed_size_chunks(text, chunk_size=500, overlap=50):
    """Split every N characters with overlap"""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += chunk_size - overlap  # slide window
    return chunks

# Pros: Simple, fast
# Cons: May split mid-sentence, lose meaning
```

### 7. Sentence Chunking

```python
import spacy
nlp = spacy.load("en_core_web_sm")

def sentence_chunks(text, max_chars=500):
    doc = nlp(text)
    chunks, current = [], ""
    for sent in doc.sents:
        if len(current) + len(sent.text) > max_chars:
            chunks.append(current.strip())
            current = ""
        current += sent.text + " "
    if current:
        chunks.append(current.strip())
    return chunks
```

### 8. Recursive Character Splitting

```python
def recursive_split(text, separators=["\n\n", "\n", ". ", " "], chunk_size=500):
    """Try splitting by paragraph → sentence → word boundaries"""
    for sep in separators:
        if sep == ". ":
            parts = [p + "." for p in text.split(".") if p.strip()]
        else:
            parts = text.split(sep)
        
        if all(len(p) < chunk_size for p in parts):
            return parts  # found a good split level
    
    # Fallback: character-level
    return [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
```

### 9. Semantic Chunking

Uses embeddings to detect topic changes:

```python
def semantic_chunks(sentences, model, threshold=0.7):
    """Split where cosine similarity between consecutive sentences drops"""
    chunks, current = [], [sentences[0]]
    
    for i in range(1, len(sentences)):
        sim = cosine_similarity(
            model.encode(sentences[i-1]),
            model.encode(sentences[i])
        )
        if sim < threshold:  # topic shift
            chunks.append(" ".join(current))
            current = []
        current.append(sentences[i])
    
    chunks.append(" ".join(current))
    return chunks
```

### 10. Parent-Child Chunking

```python
# Store small chunks for retrieval, use parent chunks for context
class ParentChildChunker:
    def chunk(self, doc, parent_size=1000, child_size=200):
        parents = fixed_size_chunks(doc, parent_size)
        result = []
        for parent in parents:
            children = fixed_size_chunks(parent, child_size)
            result.append({
                "parent": parent,
                "children": children,
                "parent_id": id(parent)
            })
        return result

# Retrieval: search child embeddings (precise match)
# Context: return parent text (full context for the LLM)
```

### 11. Chunk Size and Overlap

| Size | Pros | Cons | Best For |
|------|------|------|----------|
| 128 tokens | Very precise retrieval | Loses context | FAQ, Q&A pairs |
| 256-512 tokens | Good balance | May split topics | Default for most systems |
| 1024+ tokens | Rich context | Less precise | Document analysis |

**Overlap** (10-20%): Ensures chunks at boundaries don't lose information.

### Why This Matters
ROAST uses chunked market intelligence signals. SYNAPSE chunks ingested documents for retrieval. Getting chunking wrong means the right answer exists in your DB but your system can't find it.

---

## 12. What is an Embedding

### Plain Explanation

An embedding is a list of numbers (a vector) that **captures the meaning** of a text. Similar texts have similar numbers.

```
"cat" → [0.2, -0.5, 0.8, 0.1, ...]  (384 or 1536 numbers)
"kitten" → [0.21, -0.48, 0.79, 0.12, ...]  (very similar to "cat")
"car" → [-0.3, 0.7, -0.2, 0.5, ...]  (very different from "cat")
```

### Properties

- **Nearby in vector space = similar in meaning**
- "king - man + woman ≈ queen" (embeddings capture analogies)
- Dimension: typically 384 (all-MiniLM), 768 (BERT), 1536 (Ada), 3072 (Gemini)

---

## 13. How Embeddings Are Created

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')  # 384-dim

texts = [
    "The cat sat on the mat",
    "A dog played in the park",
    "Stock market rally today"
]

embeddings = model.encode(texts)
print(embeddings.shape)  # (3, 384) — 3 texts, 384 dimensions each

# "cat" and "dog" sentences will be similar (both animals, domestic)
# "stock" sentence will be far from both
```

The model has been trained on millions of sentence pairs: "which sentences mean similar things?" It learns to map meaning → numbers.

---

## 14. Cosine Similarity

### Formula

```
cosine_similarity(A, B) = (A · B) / (|A| × |B|)
                         = cos(angle between vectors)
                         = [-1, 1]

1.0 = identical direction (same meaning)
0.0 = perpendicular (unrelated)
-1.0 = opposite (opposite meaning — rare in embeddings)
```

### Code

```python
import numpy as np

def cosine_similarity(a, b):
    dot_product = np.dot(a, b)
    norm_a = np.linalg.norm(a)
    norm_b = np.linalg.norm(b)
    return dot_product / (norm_a * norm_b)

# Example
query_emb = model.encode("machine learning")
doc1_emb = model.encode("deep neural networks")
doc2_emb = model.encode("what to eat for dinner")

sim1 = cosine_similarity(query_emb, doc1_emb)  # ~0.85
sim2 = cosine_similarity(query_emb, doc2_emb)  # ~0.10
```

### Why This Matters
This is the core of ALL embedding-based search. Every vector DB (FAISS, Pinecone, Qdrant) uses this under the hood.

---

## 15. What is a Vector Database

### Plain Explanation

A vector database stores embeddings and supports fast similarity search. Instead of SQL's `WHERE name = 'cat'`, you ask: "find me 10 vectors most similar to this one."

### How it Works (Approximate)

1. Store: text → embedding (384 floats) → store in DB
2. Query: question → embedding → find nearest neighbors
3. Return: original texts of nearest neighbors

### Index Types

| Index | Speed | Accuracy | Memory |
|-------|-------|----------|--------|
| Flat (brute force) | Slow | 100% | High |
| IVF (inverted file) | Fast | ~95% | Medium |
| HNSW (hierarchical) | Very fast | ~99% | High |

---

## 16. Vector DB Comparison

| DB | Type | Free Tier | Best For |
|----|------|-----------|----------|
| **FAISS** | Library (local) | Unlimited | Self-hosted, custom |
| **Pinecone** | Managed cloud | 1M vectors | Production, don't want ops |
| **Chroma** | Embedded | Unlimited | Prototyping, local dev |
| **Weaviate** | Self-hosted cloud | 200K vectors | Full-featured |
| **Qdrant** | Self-hosted cloud | 1M vectors | Production self-hosted |
| **pgvector** | PostgreSQL extension | Included | Already using Postgres |

### Why This Matters
ROAST uses SQLite + numpy (not a vector DB) to avoid external dependencies. SYNAPSE uses pgvector. ACARE doesn't need one. Choosing the right vector DB depends on scale, latency requirements, and ops capacity.

---

## 17. Dense Retrieval

### What It Is

Search using embedding similarity:
```
Query: "Tell me about transformers in AI"
  ↓ encode
Query vector: [0.3, -0.1, 0.8, ...]
  ↓ cosine similarity with all docs
Top hits: 
  1. "The transformer architecture revolutionized NLP"  (score: 0.89)
  2. "Attention mechanisms in deep learning"            (score: 0.72)
  3. "How BERT works"                                   (score: 0.65)
```

**Strength**: Catches semantic matches. "Transformers in AI" matches "BERT" even though no keywords overlap.
**Weakness**: Misses exact keyword matches. "Python code example" might not match "def function_name():" if embeddings don't capture syntax well.

---

## 18. Sparse Retrieval — BM25

### What It Is

Keyword-based ranking. Evolution of TF-IDF.

**TF (Term Frequency)**: How often does the word appear in this document?
**IDF (Inverse Document Frequency)**: How RARE is this word across all documents?

"the" appears in every document → low IDF (not informative)
"transformers" appears in few docs → high IDF (very informative)

### BM25 Intuition

```
Score = how well document d matches query q

For each query term t:
  1. More occurrences in document = higher score (but diminishing returns)
  2. Longer document = slightly penalized (might have more words by chance)
  3. Rarer term = higher score (more discriminative)
```

### Code: Basic BM25

```python
import math
from collections import Counter

class BM25:
    def __init__(self, documents, k1=1.5, b=0.75):
        self.documents = documents
        self.k1 = k1          # term frequency saturation
        self.b = b            # length normalization
        self.avgdl = sum(len(d) for d in documents) / len(documents)
        self.idf = self._compute_idf()
    
    def _compute_idf(self):
        """Compute IDF for every word"""
        N = len(self.documents)
        idf = {}
        for doc in self.documents:
            for word in set(doc.split()):
                idf[word] = idf.get(word, 0) + 1
        # idf(w) = log((N - n(w) + 0.5) / (n(w) + 0.5) + 1)
        return {w: math.log((N - n + 0.5) / (n + 0.5) + 1) 
                for w, n in idf.items()}
    
    def score(self, query, doc_index):
        doc = self.documents[doc_index]
        doc_len = len(doc)
        score = 0
        for term in query.split():
            if term not in self.idf:
                continue
            tf = doc.split().count(term)  # term frequency in doc
            numerator = tf * (self.k1 + 1)
            denominator = tf + self.k1 * (1 - self.b + self.b * doc_len / self.avgdl)
            score += self.idf[term] * numerator / denominator
        return score
    
    def search(self, query, top_k=5):
        scores = [(i, self.score(query, i)) for i in range(len(self.documents))]
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:top_k]
```

### Why This Matters
BM25 is simple, fast, interpretable, and still competitive with neural methods for exact keyword matching. In production, you almost always combine BM25 + dense retrieval (hybrid).

---

## 19. BM25 Formula (Detailed)

```
For query term q_i in document D:

Score(D, Q) = SUM over query terms of:

  IDF(q_i) × TF(q_i, D) × (k1 + 1)
  ─────────────────────────────────────
  TF(q_i, D) + k1 × (1 - b + b × |D|/avgdl)

Where:
  k1 = 1.2-2.0 (controls term frequency saturation)
       Higher = more impact of repeated terms
       Lower = diminishing returns faster
  b  = 0.75 (controls length normalization)
       Higher = penalize long docs more
       Lower = ignore doc length
  |D| = length of document
  avgdl = average document length in collection
```

### What Each Part Does

1. **IDF(q_i)**: Rare words get higher weight. "Transformer" > "model"
2. **TF(q_i, D) × (k1+1) / (TF + k1 × ...)**: Saturation. First mention of a word adds a lot. 10th mention adds almost nothing more.
3. **Length normalization**: Longer documents have more words by chance, so penalize them slightly.

---

## 20. Hybrid Search

### The Insight

BM25 catches exact keyword matches (good for code, names, IDs).
Dense search catches semantic matches (good for concepts, paraphrases).

**Together they're better than either alone.**

### Simple Hybrid

```python
def hybrid_search(query, bm25_index, dense_index, alpha=0.5):
    """
    alpha=1: pure dense, alpha=0: pure BM25
    alpha=0.5: equal weight
    """
    bm25_results = bm25_index.search(query)      # list of (doc_id, bm25_score)
    dense_results = dense_index.search(query)     # list of (doc_id, dense_score)
    
    # Normalize scores to [0, 1]
    bm25_results = normalize_scores(bm25_results)
    dense_results = normalize_scores(dense_results)
    
    # Fusion: weighted combination
    combined = {}
    for doc_id, score in bm25_results:
        combined[doc_id] = (1 - alpha) * score
    for doc_id, score in dense_results:
        combined[doc_id] = combined.get(doc_id, 0) + alpha * score
    
    return sorted(combined.items(), key=lambda x: x[1], reverse=True)
```

### Why This Matters
ROAST's DIVE retrieval uses BM25 + vector + RRF fusion — exactly this pattern. This is the standard production approach.

---

## 21. RRF — Reciprocal Rank Fusion

### Plain Explanation

Instead of combining raw scores (which are on different scales), RRF combines **ranks**.

```
BM25 results:   doc A (rank 1), doc B (rank 2), doc C (rank 3)
Dense results:  doc C (rank 1), doc A (rank 2), doc B (rank 3)

RRF score for doc A = 1/(60 + 1) + 1/(60 + 2) = 0.0164 + 0.0161 = 0.0325
RRF score for doc B = 1/(60 + 2) + 1/(60 + 3) = 0.0161 + 0.0159 = 0.0320
RRF score for doc C = 1/(60 + 3) + 1/(60 + 1) = 0.0159 + 0.0164 = 0.0323

Ranking: doc A (0.0325) > doc C (0.0323) > doc B (0.0320)
```

### Code

```python
def rrf(results_list, k=60):
    """
    results_list: list of lists [(doc_id, score), ...]
    k: constant (typical 60)
    """
    scores = {}
    for results in results_list:
        for rank, (doc_id, _) in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### Why RRF Works Better Than Score Fusion

- BM25 scores and cosine similarities are on completely different scales
- Ranks are comparable: "rank 1" from any system always means "top result"
- RRF is robust to one system being much better/worse than another
- No parameter tuning needed (k=60 works universally)

---

## 22. Reranking — What and Why

### The Problem

BM25 and dense search are **fast but rough**. They retrieve 20-50 documents, but many are irrelevant.

Reranking uses a **cross-encoder** (processes query + document together) to score each pair precisely.

### Why Not Just Use Cross-Encoder for Everything?

| Method | Speed | Quality |
|--------|-------|---------|
| Bi-encoder (dense) | 10,000 docs/sec | Good |
| BM25 | 100,000+ docs/sec | OK |
| Cross-encoder (reranker) | 100 docs/sec | Excellent |

**Pattern**: Fast retriever finds top-50, reranker scores top-5.

```python
def retrieve_and_rerank(query, docs, retriever, reranker):
    # Step 1: fast retrieval
    candidates = retriever.search(query, top_k=50)
    
    # Step 2: precise reranking
    pairs = [(query, docs[i]) for i, _ in candidates]
    scores = reranker.score(pairs)  # cross-encoder
    
    # Step 3: return top re-ranked
    reranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return reranked[:5]
```

---

## 23. Cross-Encoder vs Bi-Encoder

### Bi-Encoder (used for initial retrieval)

```
Query → [encoder] → query_vector
Doc   → [encoder] → doc_vector
                      ↓ cosine similarity
                      score
```

- **Independently encode** query and docs
- Can pre-compute doc vectors (fast retrieval)
- Loses: query-doc interaction information

### Cross-Encoder (used for reranking)

```
(query, doc) → [encoder together] → score
```

- **Processes pair together** — full attention between query and doc
- Can't pre-compute (must process each pair separately)
- Much more accurate but slower

### Why This Matters
ROAST uses this pattern: DIVE retrieves (bi-encoder + BM25), then the LLM implicitly reranks by reading the top chunks. Better RAG systems use an explicit reranker in between.

---

## 24. HyDE — Hypothetical Document Embeddings

### The Insight

Sometimes the user's query is abstract ("tell me about good restaurants"). A hypothetical document would be concrete ("The Italian restaurant on Main Street serves excellent pasta...").

HyDE generates a hypothetical perfect document from the query, then uses ITS embedding for retrieval.

```python
def hyde_retrieve(query, llm, embedder, doc_store):
    # Step 1: Generate hypothetical document
    hypo_doc = llm(f"Write a document that would perfectly answer: {query}")
    
    # Step 2: Embed the hypothetical document
    hypo_embedding = embedder.encode(hypo_doc)
    
    # Step 3: Search with this embedding
    results = doc_store.search(hypo_embedding)
    
    return results
```

### Why This Matters
HyDE improves retrieval for abstract/ambiguous queries. Useful when the query doesn't share keywords with the relevant documents.

---

## 25. Query Decomposition

### Plain Explanation

Break a complex query into simpler sub-queries, retrieve for each, combine results.

```
Query: "What are the salary ranges and required skills for AI engineers in Bangalore?"

Decomposed:
1. "AI engineer salary Bangalore"
2. "AI engineer required skills Bangalore"  
3. "AI engineer job market Bangalore 2026"

Each sub-query is simpler to match against documents. Combine results.
```

### Why This Matters
Used in ROAST's DIVE pipeline (6 targeted queries per combo). Used in SYNAPSE's reasoning node (decomposition into sub-questions).

---

## 26. Self-RAG

### The Pattern

Self-RAG adds a **reflection step** after retrieval. The model decides if each retrieved chunk is relevant.

```
1. Retrieve chunks
2. For each chunk: is it relevant? (Yes/No)
3. For relevant chunks: does it support or contradict?
4. Generate answer using ONLY relevant chunks
5. Optionally: decide if more retrieval is needed
```

### Why This Matters
Self-RAG improves quality by filtering out irrelevant retrieved content before generation. This is similar to ROAST's agent architecture (staged pipeline with checks).

---

## 27. RAG vs Fine-Tuning — When to Use Which

| Factor | Use RAG | Use Fine-Tuning |
|--------|---------|-----------------|
| Knowledge changes hourly | ✅ | ❌ |
| Need to cite sources | ✅ | ❌ |
| Private/company data | ✅ | ❌ |
| Teach new behavior/format | ❌ | ✅ |
| Change model tone/style | ❌ | ✅ |
| Reduce latency | ❌ | ✅ (no retrieval step) |
| Reduce cost | ❌ | ✅ (no per-query retrieval) |

**Best**: RAG + Fine-tuning. Fine-tune for output format/style, RAG for knowledge.

---

## 28-31. RAGAS Evaluation Metrics

### 28. Faithfulness

**Measures**: Are claims in the answer supported by the retrieved context?

```
Answer: "Python was created in 1991 by Guido van Rossum."
Context: "Guido van Rossum began work on Python in the late 1980s."
         "Python 1.0 was released in 1994."

→ Answer says "1991" but context says "late 1980s" and "1.0 was 1994"
→ LOW faithfulness (answer not supported by context)
```

**Score**: 0-1. 0.889 = 88.9% of claims are supported by context.

### 29. Answer Relevancy

**Measures**: Is the answer relevant to the question?

```
Question: "What is Python?"
Answer: "Python is a programming language created by Guido van Rossum."
→ HIGH relevancy

Answer: "Python is also a type of snake found in Africa."
→ LOW relevancy
```

**Score**: 0-1. 0.875 = the answer is 87.5% relevant to the question.

### 30. Context Recall

**Measures**: Does the retrieved context contain ALL the information needed?

```
Question: "Who created Python and when?"

Context contains: "Guido van Rossum" ✓
Context misses: "1994" ✗
→ LOW context recall (missing information)

Even if the LLM answers correctly (from training data), 
the system would fail if it relied only on RAG.
```

### 31. Context Precision

**Measures**: Of the chunks retrieved, how many were actually useful?

```
Retrieved 10 chunks:
Chunks 1, 2, 3, 4: relevant to the question
Chunks 5, 6, 7, 8, 9, 10: irrelevant

Precision = 4/10 = 0.4 (40% of retrieved content was useful)
```

**Signal-to-noise ratio**. High precision = your retrieval is finding the right stuff.

### Why This Matters
ROAST doesn't use RAGAS yet but should. SYNAPSE uses it to monitor production quality. When you're building RAG systems, you need these metrics to know if changes actually improve things.

---

# PART 2: AGENTIC AI

## 32. What is an AI Agent

### Plain Explanation

An agent is a system that can PERCEIVE its environment, REASON about what to do, and TAKE ACTIONS to achieve goals.

- **Chatbot**: Talks. Gives advice. Can't DO anything.
- **Agent**: Books your flights, sends emails, updates databases, controls robots.

### The Core Loop

```
1. User gives a goal: "Book a flight to Paris next Tuesday"
2. Agent thinks: "I need to search for flights"
3. Agent acts: calls search_flights(date="next Tuesday", destination="Paris")
4. Agent observes: "Found flight AF1234 for $450"
5. Agent thinks: "Good. Now I need to book it."
6. Agent acts: calls book_flight(flight="AF1234", payment="...")
7. Agent observes: "Confirmed. Booking reference ABC123."
8. Agent thinks: "Done. Tell the user."
9. Agent acts: sends response "Booked! Reference ABC123."
```

---

## 33. Agent vs Chatbot

| | Chatbot | Agent |
|---|---------|-------|
| **Does** | Talks | Acts |
| **Output** | Text | Text + API calls + file operations |
| **Memory** | Conversation | Conversation + state + tools |
| **Autonomy** | None | Can act independently |
| **Example** | Customer support chat | AI that actually refunds your order |
| **Complexity** | Simple | Complex (orchestration, error handling) |

### Why This Matters
The the platform role is about BUILDING agents, not chatbots. the platform is an enterprise agent platform. the orchestration layer is an agent framework. Understanding the difference is table stakes.

---

## 34. Agent Components

```
Agent = Perception + Reasoning + Action + Memory
```

**Perception**: Receives input (user message, sensor data, system state)
**Reasoning**: LLM decides what to do (which tool, what parameters)
**Action**: Executes the decision (API call, file write, robot movement)
**Memory**: Remembers what happened (conversation history, tool results)

### Code: Minimal Agent

```python
import json

class SimpleAgent:
    def __init__(self, llm):
        self.llm = llm
        self.memory = []  # conversation history
    
    def run(self, user_input):
        # 1. Perceive
        self.memory.append({"role": "user", "content": user_input})
        
        # 2. Reason — LLM decides what to do
        response = self.llm(self.memory, tools=self.available_tools)
        
        # 3. Action — execute if tool call
        if response.get("tool_call"):
            result = self.execute_tool(response["tool_call"])
            self.memory.append({"role": "tool", "content": result})
            # Re-reason with tool result
            response = self.llm(self.memory)
        
        # 4. Respond
        self.memory.append({"role": "assistant", "content": response["content"]})
        return response["content"]
```

---

## 35. What are Tools

Tools are functions the agent can call. Each tool has:
- **Name**: what the agent says to invoke it
- **Description**: when to use it (the agent reads this to decide)
- **Parameters**: what inputs the tool needs
- **Implementation**: the actual code

### Example

```python
tools = [
    {
        "name": "search_flights",
        "description": "Search for available flights. Use when user wants to fly somewhere.",
        "parameters": {
            "destination": "string (required): city or airport code",
            "date": "string (required): date of travel",
            "origin": "string (optional): departure city"
        },
        "implementation": lambda dest, date, origin=None: flight_api.search(dest, date, origin)
    },
    {
        "name": "get_weather",
        "description": "Get current weather for a location.",
        "parameters": {
            "location": "string (required): city name"
        },
        "implementation": lambda loc: weather_api.get(loc)
    }
]
```

---

## 36. Function Calling

Function calling = the LLM outputs a structured JSON specifying which function to call and with what arguments.

### The Pattern

```python
# LLM receives:
{
    "role": "user", 
    "content": "What's the weather in Tokyo?",
    "tools": [get_weather definition, ...]
}

# LLM outputs:
{
    "tool_calls": [{
        "name": "get_weather",
        "arguments": {"location": "Tokyo"}
    }]
}

# System executes:
result = get_weather(location="Tokyo")

# Result fed back to LLM:
{
    "role": "tool",
    "content": json.dumps(result)
}

# LLM now responds naturally:
"Tokyo is currently 22°C with clear skies."
```

### Why This Matters
Function calling is HOW agents work. It's the mechanism that lets LLMs control external systems. Every agent framework (LangChain, CrewAI, OpenAI Assistants) is built on this pattern.

---

## 37. ReAct Pattern (Detailed)

### Reason + Act Loop

```
Thought: I need to find the user's account to check their order status.
Action: search_customer(email="john@example.com")
Observation: Found customer #12345, name: John Doe
Thought: Now I need to find their recent orders.
Action: get_orders(customer_id=12345)
Observation: Order #7890: "Laptop" — status: "shipped", tracking: "1Z999AA10123456784"
Thought: I have all the info. Let me respond.
Response: Your laptop (Order #7890) was shipped. Tracking number: 1Z999AA10123456784
```

### Code

```python
class ReActAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = {t["name"]: t for t in tools}
    
    def run(self, task, max_steps=10):
        messages = [{"role": "user", "content": task}]
        
        for step in range(max_steps):
            response = self.llm(messages, tools=list(self.tools.values()))
            
            if response.get("type") == "final":
                return response["content"]
            
            if response.get("tool_call"):
                tool_name = response["tool_call"]["name"]
                args = response["tool_call"]["arguments"]
                
                # Execute tool
                result = self.tools[tool_name]["implementation"](**args)
                
                # Feed result back
                messages.append({
                    "role": "tool",
                    "content": str(result),
                    "name": tool_name
                })
        
        return "Max steps reached."
```

---

## 38-41. Multi-Agent Systems

### 38. Multi-Agent Systems

Multiple agents with different roles working together:
- **Researcher agent**: finds information
- **Writer agent**: produces content
- **Critic agent**: reviews and improves
- **Coordinator agent**: routes work between others

### 39. Supervisor Pattern

```
User → Supervisor Agent
          ├── routes to: Researcher Agent (to search)
          ├── routes to: Writer Agent (to write draft)
          ├── routes to: Critic Agent (to review)
          └── decides: done → respond to user
```

### 40. Hierarchical Agents

```
Manager Agent
├── Worker 1: Search documents
├── Worker 2: Analyze data
└── Worker 3: Generate report
Manager: synthesizes worker outputs
```

### 41. Parallel Agents

```python
import asyncio

async def parallel_agent_pipeline(query):
    # Launch all agents simultaneously
    tasks = [
        market_context_agent.run(query),
        red_flag_agent.run(query),
        six_second_agent.run(query),
        competitive_agent.run(query),
        technical_depth_agent.run(query),
    ]
    
    # Wait for all to complete
    results = await asyncio.gather(*tasks)
    
    # Synthesize
    final = review_agent.run(query, results)
    return final
```

**This is exactly ROAST's architecture.** 4 agents run in parallel, then a synthesis agent runs on the combined results.

---

## 42-43. Human-in-the-Loop (HITL)

### 42. What and Why

The agent pauses and asks a human for input before proceeding. Critical for:
- Safety (don't let AI delete customer data)
- High-stakes decisions (approving large payments)
- Edge cases (agent uncertain what to do)

### Code: Approval Gate

```python
async def execute_with_approval(action, ask_human):
    """Execute action only after human approval"""
    
    # Ask human
    approved = await ask_human(f"Approve this action? {action}")
    
    if not approved:
        return {"status": "rejected", "reason": "Human declined"}
    
    # Execute
    result = await action.execute()
    return {"status": "executed", "result": result}
```

### 43. Types of HITL

| Type | When | Example |
|------|------|---------|
| **Approval gates** | Before critical actions | "Approve payment of $500?" |
| **Exception handling** | On errors/uncertainty | "I can't find this customer. Please help." |
| **Monitoring** | Passive observation | Human watches, can intervene anytime |

In ROAST, the follow-up question is HITL: the user reads the review and can ask follow-up questions. In ACARE, the operator can ESTOP at any time.

---

## 44. Orchestration

Orchestration is the "glue" that connects:
- **LLM** decides what to do
- **Tools** execute actions
- **Memory** stores history
- **State** tracks progress

### What Orchestration Handles

1. **Sequencing**: What runs when (parallel, sequential, conditional)
2. **State passing**: Results from one step → next step
3. **Error handling**: What happens when a tool fails
4. **Retry logic**: Try again with different approach
5. **Budget management**: Track token usage, rate limits
6. **Observability**: Log all steps for debugging

---

## 45-47. LangGraph

### 45. State, Nodes, Edges

LangGraph models your agent as a **graph**:

- **State**: shared data structure passed between nodes
- **Nodes**: processing steps (functions)
- **Edges**: transitions between nodes

```python
from langgraph.graph import StateGraph

# Define state
class AgentState(TypedDict):
    query: str
    search_results: list
    analysis: str
    final_answer: str
    steps_completed: int

# Define nodes (processing steps)
def retrieve(state: AgentState) -> AgentState:
    state["search_results"] = search(state["query"])
    return state

def analyze(state: AgentState) -> AgentState:
    state["analysis"] = llm_analyze(state["query"], state["search_results"])
    return state

def generate(state: AgentState) -> AgentState:
    state["final_answer"] = llm_generate(state["analysis"])
    return state

# Build graph
graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve)
graph.add_node("analyze", analyze)
graph.add_node("generate", generate)

# Add edges (transitions)
graph.add_edge("retrieve", "analyze")
graph.add_edge("analyze", "generate")
graph.set_entry_point("retrieve")
graph.set_finish_point("generate")
```

### 46. Conditional Edges

```python
def should_retry(state: AgentState) -> str:
    """Route based on state"""
    if state["analysis"].confidence < 0.7:
        return "retrieve"  # go back and search more
    else:
        return "generate"  # proceed to answer

graph.add_conditional_edges(
    "analyze",
    should_retry,
    {"retrieve": "retrieve", "generate": "generate"}
)
```

### 47. LangGraph vs LangChain Chains

| | Chains | Graph |
|---|--------|-------|
| **Flow** | Linear A→B→C | Any topology |
| **Loops** | No | Yes (retry, refine) |
| **Branches** | Simple if/else | Complex conditional routing |
| **State** | Simple pass-through | Shared, persistent state |
| **Use case** | Simple pipelines | Complex agents with decisions |

### Why This Matters
SYNAPSE's entire reasoning pipeline IS a LangGraph (8 nodes: entry → decomposition → retrieval + web_research → analysis → synthesis → critic → output). Understanding this lets you understand SYNAPSE at the architectural level.

---

## 48. Memory Types

| Type | What | Example | Duration |
|------|------|---------|----------|
| **In-context** | Full conversation in prompt | "What were we talking about?" | Current session |
| **Episodic** | Past sessions | "Last week you asked about... | Days |
| **Semantic** | Learned knowledge | "User prefers Python" | Permanent |
| **External** | Vector DB / documents | Company knowledge base | Permanent |

### Why This Matters
ROAST uses in-context + external (market intel DB). SYNAPSE uses all four (in-context for reasoning, external for RAG, episodic for session, semantic for learned user preferences).

---

## 49-51. the platform orchestration framework

### 49. orchestration framework — The 3-Layer Architecture (CORRECTED)

the platform orchestration framework framework is NOT "4 components." It's a three-layer architecture:

```
Layer 1: Entities (the nouns)
  Account, Order, Card, Payment, SupportTicket
  → Defined once, reused across all agents

Layer 2: Cognitive Functions (atomic verbs)
  checkBalance(account_id), resetPassword(user_id), createOrder(...)
  → Modular, reusable, channel-independent, testable in isolation

Layer 3: AI Agents (dynamic orchestrators)
  Listen → Assess State → Reason → Act → Interpret → Repeat
  → NOT rigid scripts. Figure out the path based on current state.
```

**How it works in practice:**

Customer: "I lost my card. Can you send me a replacement?"

```
Agent LISTENS → Goal = "replacement card"
Agent ASSESSES STATE → Account active? Existing replacement?
Agent REASON → verify identity → check fraud → create order
Agent ACTS → calls verifyIdentity() → checkFraudFlag() → createReplacementOrder()
Agent RESPONDS → "Replacement ordered, tracking XYZ"
```

**Key differences from traditional chatbots:**
- Traditional: rigid script ("if user says X, do Y"). Breaks on edge cases.
- orchestration framework: agent figures out the path dynamically based on state. Handles any edge case by composing Cognitive Functions.

**Additional features:**
- Guardrails: enterprise safety, compliance, sensitive data never enters LLM
- MCP support (the platform.3): standard protocol for connecting external tools
- Agent Console: AI companion assists human reps during live calls
- Omnichannel: voice, chat, web — all first-class, agent switches mid-conversation

### 50. the orchestration layer (Open Agent System)

Launched May 2026. the platform open framework for building voice-enabled agents. Key features:
- Voice-first agent design
- Built on MCP for tool integration
- Multi-modal (voice + text + visual)

### 51. MCP — Model Context Protocol

MCP is a standard protocol for connecting LLMs to external tools and data.

```
LLM ←→ MCP Client ←→ MCP Server(s)
                         ├── Database Server
                         ├── File System Server
                         ├── API Server
                         └── Memory Server
```

**Like USB-C for AI**: one standard protocol, many devices.

- **MCP Server**: exposes Resources (data) and Tools (actions)
- **MCP Client**: connects to servers, discovers capabilities
- **MCP Protocol**: JSON-RPC over stdio or HTTP

### Simple MCP Interaction

```
Client → Server: list_tools()
Server → Client: [search_docs, send_email, query_db]

Client → Server: call_tool("search_docs", {"query": "RAG patterns"})
Server → Client: ["RAG combines retrieval with generation..."]

Client (puts this in LLM context):
"Based on the documentation, RAG combines retrieval with generation..."
```

### Why This Matters
MCP is the future standard for agent-tool communication. the platform the orchestration layer uses it. Understanding MCP means understanding how agents will connect to the world going forward.
