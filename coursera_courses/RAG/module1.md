# RAG Overview

**RAG = Retrieval-Augmented Generation — letting LLMs use retrieved data to generate better, grounded answers.**

## What is RAG?
- **RAG** = Retrieval-Augmented Generation
- Lets LLMs use **external data** (not in training set) to answer questions
- Like ChatGPT/Claude "searching the web" before answering

## Why RAG?
| Problem | RAG Solution |
|---------|-------------|
| LLM only knows training data | RAG adds private/current documents |
| Hallucinations | Grounds answers in retrieved facts |
| Can't access internal docs | RAG retrieves from company KB |

## Course Covers
**Foundations:**
- How search systems work
- How LLMs reason
- Combining both effectively

**Practical:**
- Data prep + chunking
- Prompting for grounded answers
- Evaluating real traffic
- Tuning hyperparameters
- Building production RAG pipelines

## RAG Evolution
| Trend | Impact |
|-------|--------|
| Better LLMs | Hallucinate less, reason better over context |
| Larger context windows | Less aggressive chunking needed |
| Multi-modal RAG | RAG over PDFs, images, slides |
| **Agentic RAG** | AI agents decide what to search, query multiple DBs, re-retrieve on failure |

## Agentic RAG (Key Highlight)
Instead of humans deciding retrieval — **agents** do it:
- Choose what to search
- Pick which database
- Multiple rounds of retrieval
- Self-correct mistakes

## RAG in Agentic Workflows
- RAG is often **one step** in a multi-step agent pipeline (e.g. step 5 of 7)
- It feeds the agent the info it needs to continue reasoning

---

## Course Roadmap (Module 1 Overview)

### What You'll Learn This Module
- Foundational RAG concepts
- RAG system components (retriever + LLM)
- Build a **simple functional RAG system** (first programming assignment)

### RAG Architecture (Mental Model)
```
User Query
    │
    ▼
┌──────────────┐
│   Retriever   │  ← Searches knowledge base for relevant info
│  (finds docs) │
└──────┬───────┘
       │ Retrieved context
       ▼
┌──────────────┐
│     LLM      │  ← Generates answer grounded in retrieved context
│ (generates)  │
└──────┬───────┘
       │
       ▼
   Grounded Answer
```

### What This Course Covers (Full Roadmap)
| Module | Topic |
|--------|-------|
| 1 | Foundations + Build simple RAG |
| 2 | Robust retriever |
| 3 | Vector database |
| 4 | Sophisticated LLM usage |
| 5+ | Monitoring & evaluation techniques |

### Key Insight
RAG = **Retriever** + **LLM** paired together. The retriever finds relevant information from a trusted knowledge base, then the LLM generates a response grounded in that information.

---

## Why LLMs Need RAG

### What Are LLMs?
- **Mathematical models** trained on massive datasets from the open internet
- Training: model learns information contained in training data
- Capabilities: answer questions, summarize, rewrite, give feedback, generate code
- Think of them like a person who read huge chunks of the internet — broad general knowledge

### The Problem
- When you prompt an LLM, it relies on **what it learned during training**
- You're hoping the info it needs was in the training data (ideally many times)
- But lots of info **won't be included**:
  - Private company databases
  - Hidden or hard-to-access information
  - Breaking news (published after training cut-off)
- Unreasonable to expect LLMs to be experts on everything
- They give **much better responses with access to better information**

### The RAG Solution
- RAG improves LLM performance by giving them access to info they **don't know from training**
- Same reason humans look things up before answering

### Three Levels of Questions

| Level | Example | Info Needed |
|-------|---------|-------------|
| **General knowledge** | "Why are hotels expensive on weekends?" | None — LLM already knows |
| **Recent/specific** | "Why are Vancouver hotels expensive THIS weekend?" | Need to search (Taylor Swift concert) |
| **Deep research** | "Why doesn't Vancouver have more downtown hotel capacity?" | Need specialized documents |

### The Core Idea
**Put the information in the prompt.**

A RAG system modifies the prompt before sending it to the LLM:

```
User question + Retrieved context → Augmented prompt → LLM → Grounded answer
```

### How It Works
| Step | What Happens |
|------|-------------|
| 1. User asks question | "Why are hotels expensive this weekend?" |
| 2. **Retrieval** | Retriever searches knowledge base for relevant info |
| 3. **Augmentation** | Retrieved info + original question = new prompt |
| 4. **Generation** | LLM responds using the augmented prompt |

### Retriever (Key Component)
- Manages a **knowledge base** of trusted, relevant, private information
- When a query comes in, it finds the most relevant documents
- Passes those documents to the LLM as context

### Why "Retrieval-Augmented Generation"?
```
Retrieval:  Find relevant info from knowledge base
Augmented:  Add that info to the prompt
Generation: LLM generates answer using the augmented prompt
```
