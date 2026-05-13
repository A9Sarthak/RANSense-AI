# RANSense-AI — System Architecture

## Problem Statement
**Problem 10: RAG based Future-Ready Telecom RAN Assistant** — Samsung ennovateX AX Hackathon 2026

---

## Overview

RANSense-AI is a **Multi-Agent Memory-Augmented RAG Framework** for 5G/6G Radio Access Networks. It transforms passive telecom infrastructure documentation into a proactive, self-healing cognitive assistant for RAN engineers.

The system eliminates the manual process of correlating live network fault alarms with 3GPP/O-RAN specifications — reducing Mean Time To Resolution (MTTR) from ~4 hours to under 15 minutes.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                           │
│          Web Chat Interface  │  Dashboard & Analytics           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                      BACKEND SERVICES                           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ API Gateway  │  │    Router    │  │   LLM Generation     │  │
│  │ FastAPI      │──│    Agent     │──│   Llama-3-8B-Instruct│  │
│  │ Auth/Rate    │  │  LangGraph   │  │   Chain-of-Thought   │  │
│  └──────────────┘  └──────┬───────┘  └──────────────────────┘  │
│                           │                                     │
│              ┌────────────┴─────────────┐                       │
│              │                          │                       │
│  ┌───────────▼────────┐  ┌─────────────▼──────────────────┐    │
│  │  Spec-Search Agent │  │      Telemetry Agent           │    │
│  │  3GPP/O-RAN RAG    │  │      Live PM/FM SQL queries    │    │
│  │  ChromaDB + BM25   │  │      KPI counter analysis      │    │
│  └───────────┬────────┘  └─────────────┬──────────────────┘    │
│              │                          │                       │
│              └─────────┬────────────────┘                       │
│                        │                                        │
│              ┌──────────▼──────────────┐                        │
│              │   Cross-Encoder         │                        │
│              │   Re-Ranking (BAAI)     │                        │
│              │   Multi-source fusion   │                        │
│              └──────────┬──────────────┘                        │
│                         │                                       │
│              ┌───────────▼─────────────┐                        │
│              │   Mem0 Memory Layer     │                        │
│              │   Persistent session +  │                        │
│              │   historical RCA store  │                        │
│              └─────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                    KNOWLEDGE BASE LAYER                         │
│                                                                 │
│  ┌─────────────┐  ┌───────────────┐  ┌────────────────────┐    │
│  │ Data Sources│  │   Ingestion   │  │   Vector Store     │    │
│  │ 3GPP PDFs   │─▶│   LlamaParse  │─▶│   ChromaDB         │    │
│  │ O-RAN Docs  │  │   Chunking    │  │   BAAI/bge embeds  │    │
│  │ RAN-FaultSim│  │   Embedding   │  │   BM25 index       │    │
│  └─────────────┘  └───────────────┘  └────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Router Agent
- **Framework**: LangGraph state machine
- **Role**: Classifies incoming query as `spec_query`, `telemetry_query`, or `hybrid_query`
- **Routing logic**: Keyword + intent detection using Llama-3 prompt classification
- **Handoff**: Dispatches to Spec-Search Agent, Telemetry Agent, or both in parallel

### 2. Spec-Search Agent
- **Retrieval**: Hybrid dense (BAAI/bge-large-en-v1.5) + sparse (BM25) search
- **Knowledge base**: ChromaDB vector store with 3GPP TS 38.300/38.401/38.411/38.711 + O-RAN WG docs
- **Parsing**: LlamaParse for structure-preserving ingestion of 3GPP tables
- **Chunking strategy**: Section-level chunking with metadata (spec ID, section number, release version)
- **Reranking**: BAAI/bge-reranker-large cross-encoder for relevance scoring

### 3. Telemetry Agent
- **Data source**: RAN-FaultSim synthetic dataset (PM counters + FM alarm records)
- **Query method**: SQL-style structured queries against KPI time-series data
- **KPIs monitored**: PRB utilization, PDCP throughput, RRC setup success rate, handover success rate
- **Alarm types**: RRC connection failure, handover drop, cell outage, UL interference

### 4. Memory Layer (Mem0)
- **Type**: Persistent cross-session memory
- **Stores**: Past fault events, root cause history per cell site, engineer resolution actions
- **Retrieval**: Semantic similarity search over memory store
- **Unique value**: No existing telecom RAG system has persistent memory of past incidents

### 5. LLM Generation
- **Model**: meta-llama/Meta-Llama-3-8B-Instruct (HuggingFace)
- **Prompting**: Chain-of-Thought (CoT) with telecom domain system prompt
- **Output format**: Structured RCA report + MML/XML configuration command
- **Inference**: GGUF Q4_K_M quantization for CPU-only vRAN edge deployment

### 6. RAGAS Evaluation CI/CD
- **Tool**: RAGAS framework
- **Metrics**: Context Precision, Answer Faithfulness, Answer Relevancy
- **Benchmark**: TeleQnA dataset (ziv/TeleQnA, CC BY 4.0)
- **Target**: >85% Answer Faithfulness on held-out 3GPP QA pairs

---

## Technology Stack

| Layer | Technology | License |
|---|---|---|
| Agent Orchestration | LangGraph | MIT |
| LLM | Meta-Llama-3-8B-Instruct | Meta Llama 3 |
| Embeddings | BAAI/bge-large-en-v1.5 | MIT |
| Reranker | BAAI/bge-reranker-large | MIT |
| Vector DB | ChromaDB | Apache-2.0 |
| Document Parsing | LlamaParse | MIT |
| Memory | Mem0 | Apache-2.0 |
| Evaluation | RAGAS | Apache-2.0 |
| API | FastAPI | MIT |
| Frontend | React | MIT |

---

## Data Flow (End-to-End)

```
Engineer types query
        ↓
Router Agent classifies intent
        ↓
    ┌───┴────────────────┐
    │                    │
Spec-Search Agent   Telemetry Agent
(3GPP vector DB)   (PM/FM SQL query)
    │                    │
    └───────┬────────────┘
            ↓
   Cross-Encoder Re-Ranking
   (Multi-source fusion)
            ↓
   Mem0 — inject historical context
   ("Cell 42 had same issue March 3rd")
            ↓
   Llama-3 generates RCA + MML fix
            ↓
   Engineer gets: Root cause + config command
   Time elapsed: ~14 seconds
```

---

## Constraints & Mitigations

| Constraint | Mitigation |
|---|---|
| LLM hallucination on strict 3GPP specs | RAGAS-validated cross-encoder reranking; source citations in every response |
| GPU unavailability on vRAN edge nodes | GGUF Q4_K_M quantization; CPU-only inference validated |
| 3GPP table structure destroyed by standard parsers | LlamaParse structure-preserving ingestion |
| No public 5G fault telemetry dataset | RAN-FaultSim synthetic dataset created and published (CC BY 4.0) |

---

*License: Apache-2.0 | GitHub: https://github.com/A9Sarthak/RANSense-AI*
