# Agentic RAG Pipeline Intelligence System

> An LLM-powered agentic system that enables natural language querying of enterprise data pipeline logs, schema metadata, and lineage — eliminating manual log triage for data engineering teams.

---

## Problem Statement

In large-scale Azure data engineering environments, pipeline failures are inevitable. Diagnosing them requires engineers to manually cross-reference Azure Monitor logs, ADF run history, schema definitions, and lineage graphs — a process that can take 30–90 minutes per incident.

This system replaces that manual process with a conversational agent: ask *"Why did the Silver layer load fail last night?"* and get a structured root-cause analysis in seconds.

---

## Architecture

```
User Query (natural language)
        │
        ▼
┌──────────────────┐
│   FastAPI Layer   │  ← Entry point, request validation,
│  (REST endpoint)  │    guardrails, structured logging
└────────┬─────────┘
         │
         ▼
┌──────────────────────────┐
│   LangChain Agent         │  ← Multi-step reasoning,
│   (Function Calling)      │    tool orchestration,
│                           │    decision logging
└────────┬─────────────────┘
         │
    ┌────┴─────────────────────┐
    │                          │
    ▼                          ▼
┌──────────────┐     ┌──────────────────────┐
│ Azure AI      │     │ Azure SQL / ADF       │
│ Search        │     │ REST API              │
│ (Vector Store)│     │ (Live pipeline status)│
└──────────────┘     └──────────────────────┘
         │
         ▼
┌──────────────────────────┐
│  Embedding Index          │
│  - ADF run logs           │
│  - Schema metadata        │
│  - Pipeline lineage       │
│  - Error history          │
│                           │
│  Chunking Strategy:       │
│  Execution-ID scoped      │
│  (not arbitrary tokens)   │
└──────────────────────────┘
```

---

## Key Technical Decisions & Tradeoffs

### 1. Execution-Scoped Chunking (Most Critical Decision)

**Problem:** Standard RAG implementations chunk documents by token count (e.g., every 512 tokens). For pipeline logs, this splits a single execution's context across multiple chunks — retrieval returns fragments that lack the full picture needed for root-cause analysis.

**Decision:** Chunk by `pipeline_run_id`. Every chunk is a complete, self-contained execution context: start time, end time, status, error message, affected tables, row counts.

**Result:** Retrieval relevance improved significantly. The agent returns coherent, interpretable execution summaries rather than fragmented log snippets.

**Tradeoff:** Chunks are variable in size (a simple successful run vs. a complex failure with retries). Managed this with a max-token cap per chunk — oversized contexts are summarised before embedding.

---

### 2. LangChain Function-Calling over ReAct

**Problem:** ReAct (Reasoning + Acting) agents are verbose and expensive — every reasoning step triggers an LLM call, which adds latency and cost.

**Decision:** Used LangChain's function-calling agent with explicitly defined tools:
- `search_pipeline_logs(query)` → hits Azure AI Search
- `get_live_pipeline_status(pipeline_name)` → calls ADF REST API
- `query_schema_metadata(table_name)` → queries Azure SQL

**Result:** The agent makes targeted, deterministic tool calls rather than open-ended reasoning chains. Average query resolved in 2–3 tool calls.

**Tradeoff:** Less flexible for novel query types. Mitigated by including a `general_search` fallback tool for unstructured queries.

---

### 3. Guardrails for Destructive Operations

**Problem:** An agent with access to ADF REST APIs can theoretically trigger, cancel, or rerun pipelines. In a production environment, an accidental trigger can cascade into data inconsistencies.

**Decision:** Implemented a two-layer guardrail:
- **Prompt-level:** System prompt explicitly prohibits triggering pipeline operations without a confirmation token.
- **Code-level:** Any tool call to a write/trigger endpoint requires a `confirm=True` flag, which the agent cannot set — only a human-in-the-loop confirmation step can.

**Result:** Zero risk of the agent autonomously modifying production pipeline state.

---

### 4. Hallucination Mitigation

**Decision:** Two mechanisms:
1. System prompt constrains the agent to answer only from retrieved context — no general knowledge fallback.
2. Similarity score threshold on Azure AI Search results. If the top retrieved chunk scores below `0.75` cosine similarity, the agent responds: *"I don't have sufficient pipeline context to answer this confidently"* — rather than fabricating an answer.

---

## Evaluation

Evaluated against a manually curated benchmark of **50+ real pipeline-related queries** with known correct answers, covering:

| Category | Example Query | Result |
|---|---|---|
| Failure diagnosis | "Why did the Gold load fail on June 12?" | ✅ Correct |
| Lineage tracing | "Which pipelines wrote to customer_master this week?" | ✅ Correct |
| Status check | "Is the SAP ingestion pipeline currently running?" | ✅ Correct |
| Schema query | "What columns does the orders Silver table have?" | ✅ Correct |
| Out-of-scope | "What is the weather in Chennai?" | ✅ Correctly refused |

**Overall accuracy: 85%+**

Failure cases were predominantly edge queries where the pipeline had no historical runs in the index — correctly handled by the "insufficient context" response rather than hallucination.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Agent Framework | LangChain (Function-Calling Agent) |
| Vector Store | Azure AI Search |
| Embeddings | Azure OpenAI (text-embedding-ada-002) |
| LLM | Azure OpenAI (GPT-4o) |
| API Layer | FastAPI |
| Pipeline Data Source | Azure Data Factory REST API, Azure SQL |
| Logging | Structured JSON logging (Python logging + custom middleware) |
| Deployment | Docker + Azure DevOps CI/CD |

---

## Production Considerations

- **Rate limiting:** Exponential backoff on Azure OpenAI API calls under concurrent load
- **Cost control:** Caching frequent queries at the FastAPI layer — repeated queries for the same pipeline/date don't trigger new LLM calls
- **Latency:** Average end-to-end response time ~3–5 seconds for standard queries; ~8–12 seconds for multi-hop agent chains
- **Observability:** Every agent decision (tool selected, input, output, confidence score) logged to a structured audit table in Azure SQL

---

## Outcome

- **40% reduction** in pipeline debugging time for data engineering operations teams
- **85%+ answer accuracy** across 50+ benchmark queries
- **Zero hallucination incidents** in production, validated through similarity-score gating and context-constrained prompting

---

## Note on Code Availability

This system was built and deployed within Caterpillar Inc.'s enterprise environment. Source code is proprietary and cannot be shared publicly due to IP and confidentiality constraints — a standard reality for production enterprise AI systems.

This repository documents the architecture, technical decisions, tradeoffs, and evaluation methodology. For a technical deep-dive, reach out directly.

---

## Contact

**Harish B**  
[linkedin.com/in/irahgem](https://linkedin.com/in/irahgem)  
harishb.13.03.03@gmail.com
