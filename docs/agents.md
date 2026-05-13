# RANSense-AI — Agent Definitions (agents.md)

## Overview

RANSense-AI uses a **LangGraph-orchestrated multi-agent system** with three specialised agents coordinated by a Router Agent. This document defines each agent's role, tools, inputs, outputs, and handoff protocols.

---

## Agent Architecture

```
                    ┌─────────────────┐
     User Query ───▶│  Router Agent   │
                    │  (Classifier)   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     spec_query       hybrid_query    telemetry_query
              │              │              │
              ▼              ▼              ▼
    ┌──────────────┐  ┌──────┴──────┐ ┌───────────────┐
    │ Spec-Search  │  │    Both     │ │  Telemetry    │
    │    Agent     │  │  in parallel│ │    Agent      │
    └──────┬───────┘  └──────┬──────┘ └───────┬───────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             ▼
                  ┌──────────────────────┐
                  │  Cross-Encoder       │
                  │  Re-Ranker           │
                  │  (Result fusion)     │
                  └──────────┬───────────┘
                             ▼
                  ┌──────────────────────┐
                  │  Mem0 Memory Layer   │
                  │  (Context injection) │
                  └──────────┬───────────┘
                             ▼
                  ┌──────────────────────┐
                  │  Llama-3 Generator   │
                  │  RCA + MML output    │
                  └──────────────────────┘
```

---

## Agent 1: Router Agent

**Role:** Entry point. Classifies query intent and dispatches to appropriate downstream agent(s).

| Field | Details |
|---|---|
| Framework | LangGraph StateGraph node |
| Model | Llama-3-8B-Instruct (same base model, lightweight classification prompt) |
| Latency target | <500ms |

**Classification logic:**

| Query Type | Trigger keywords | Dispatch |
|---|---|---|
| `spec_query` | "what is", "how does", "define", "standard", "3GPP", "specification" | Spec-Search Agent only |
| `telemetry_query` | "cell", "alarm", "KPI", "drop rate", "failure", "outage", "latency" | Telemetry Agent only |
| `hybrid_query` | Both spec + telemetry signals, or ambiguous queries | Both agents in parallel |

**Input:** Raw engineer query (string)
**Output:** `{query_type: str, query: str, cell_id: Optional[str]}`
**Handoff:** Dispatches via LangGraph conditional edges to downstream agent nodes

---

## Agent 2: Spec-Search Agent

**Role:** Retrieves relevant 3GPP/O-RAN specification context for the query.

| Field | Details |
|---|---|
| Retrieval | Hybrid dense + sparse (ChromaDB + BM25) |
| Embedding model | BAAI/bge-large-en-v1.5 (MIT) |
| Reranker | BAAI/bge-reranker-large (MIT) |
| Knowledge base | 3GPP TS 38.300/38.401/38.411/38.711 + O-RAN WG1-WG9 docs |
| Top-k retrieval | k=10 candidates → reranked to top 3 |

**Tools available:**
- `vector_search(query, k)` — Dense semantic search in ChromaDB
- `bm25_search(query, k)` — Sparse keyword search
- `rerank(query, candidates)` — Cross-encoder scoring and fusion

**Input:** `{query: str}`
**Output:** `{spec_chunks: List[{text, spec_id, section, score}], top_k: 3}`
**Handoff:** Returns structured spec context to Re-Ranker node

---

## Agent 3: Telemetry Agent

**Role:** Queries live network KPI and fault alarm data for the specified cell site.

| Field | Details |
|---|---|
| Data source | RAN-FaultSim dataset (PM counters + FM alarms) |
| Query method | SQL-style structured queries on tabular data |
| Time window | Configurable (default: last 24 hours) |

**Tools available:**
- `query_pm_counters(cell_id, time_window)` — Fetch KPI metrics (PRB utilization, throughput, RRC rate)
- `query_fm_alarms(cell_id, time_window)` — Fetch active and historical fault alarms
- `detect_anomaly(kpi_series)` — Statistical anomaly detection on KPI time series

**Input:** `{query: str, cell_id: str, time_window: str}`
**Output:** `{telemetry_summary: str, active_alarms: List, kpi_snapshot: Dict}`
**Handoff:** Returns structured telemetry context to Re-Ranker node

---

## Cross-Encoder Re-Ranker

**Role:** Fuses results from Spec-Search Agent and Telemetry Agent. Scores combined context for relevance to original query.

| Field | Details |
|---|---|
| Model | BAAI/bge-reranker-large |
| Input | Query + all candidate chunks from both agents |
| Output | Top-3 most relevant context chunks (mixed spec + telemetry) |
| Purpose | Eliminates hallucination risk by grounding LLM in highest-confidence context only |

---

## Memory Layer (Mem0)

**Role:** Injects historical incident context before LLM generation. Enables the system to "remember" past faults at the same cell site.

| Field | Details |
|---|---|
| Library | Mem0 (Apache-2.0) |
| Storage | Persistent key-value store keyed by `cell_id` |
| Retrieval | Semantic similarity search over memory store |

**Memory operations:**
- `store(cell_id, rca_summary, resolution)` — After each successful diagnosis
- `retrieve(cell_id, query)` — Before LLM generation, inject relevant past incidents
- `update(cell_id, feedback)` — Engineer confirms/corrects diagnosis

**Example memory injection:**
```
[Memory context for Cell 42]
- 2026-03-03: RRC failure due to neighbor cell B interference.
  Resolution: SET UL_INTERFERENCE_THR = 15. Resolved in 8 min.
- 2026-04-11: PDCP throughput drop due to PRB congestion.
  Resolution: ADD CAPACITY_THRESHOLD = 80. Resolved in 12 min.
```

---

## LLM Generator

**Role:** Final synthesis. Takes re-ranked context + memory and generates RCA report + actionable fix.

| Field | Details |
|---|---|
| Model | meta-llama/Meta-Llama-3-8B-Instruct |
| Prompting | Chain-of-Thought (CoT) with telecom domain system prompt |
| Output format | Structured markdown: Root Cause → Evidence → Recommended Action → MML Command |
| Inference | GGUF Q4_K_M quantization, CPU-only (no GPU required) |

**Output schema:**
```json
{
  "root_cause": "UL interference from neighbor cell misconfiguration",
  "evidence": ["TS 38.300 §6.1.2: UL_INTERFERENCE_THR defines...", "Cell 42 SINR = -3dB (threshold: 0dB)"],
  "confidence": 0.91,
  "recommended_action": "Adjust UL interference threshold for Cell 42",
  "mml_command": "SET UL_INTERFERENCE_THR = 15;\nCELL_ID = 42;\nCOMMIT;\nSAVE;",
  "source_citations": ["TS 38.300 §6.1.2", "O-RAN WG4 v4.0 §3.2"],
  "memory_context_used": true
}
```

---

## Handoff Protocol Summary

```
Router → Spec-Search:    {query, query_type}
Router → Telemetry:      {query, query_type, cell_id, time_window}
Spec-Search → ReRanker:  {spec_chunks: List[chunk]}
Telemetry → ReRanker:    {telemetry_summary, active_alarms, kpi_snapshot}
ReRanker → Mem0:         {top_context: List[chunk]}
Mem0 → Generator:        {top_context + memory_context}
Generator → User:        {rca_report, mml_command, citations, confidence}
```

---

*License: Apache-2.0 | GitHub: https://github.com/A9Sarthak/RANSense-AI*
