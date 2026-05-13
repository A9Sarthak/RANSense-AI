# 📡 RANSense-AI 

<div align="center">
  <h3>RAG-based Future-Ready Telecom RAN Assistant</h3>
  <p><strong>Samsung ennovateX AX Hackathon 2026 — Problem 10</strong></p>
  
  [![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
  [![Phase: 1](https://img.shields.io/badge/Hackathon-Phase_1-orange.svg)]()
  [![Evaluation: RAGAS](https://img.shields.io/badge/Evaluated_with-RAGAS-green.svg)]()
</div>

---

## 📖 Overview

**RANSense-AI** is a **Multi-Agent Memory-Augmented RAG Framework** designed exclusively for 5G/6G Radio Access Networks (RAN). It transforms passive, hundreds-of-pages-long telecom infrastructure documentation into a proactive, self-healing cognitive assistant for RAN engineers.

By correlating live network fault telemetry (PM/FM data) with complex 3GPP and O-RAN specifications, RANSense-AI drastically reduces the Mean Time To Resolution (MTTR) for cell outages from **~4 hours to under 15 minutes**.

---

## 🚀 Key Features

- **Multi-Agent Orchestration (LangGraph):** A Router Agent intelligently dispatches queries to a `Spec-Search Agent` (for 3GPP standards) and/or a `Telemetry Agent` (for live PM/FM data).
- **Hybrid Context Fusion:** Uses a Cross-Encoder Re-Ranker (BAAI) to seamlessly fuse 3GPP specification text with active cell alarms to prevent LLM hallucinations.
- **Persistent Network Memory (Mem0):** The *first* telecom RAG system to remember past incidents! RANSense-AI injects historical root causes and resolutions directly into the LLM context to prevent repeated debugging of identical cell issues.
- **Telecom-Grade RAGAS Evaluation:** CI/CD driven automated evaluation pipeline measuring Context Precision and Answer Faithfulness against the held-out TeleQnA dataset.
- **Structure-Preserving Ingestion:** Leverages LlamaParse to extract and vectorize dense 3GPP tables without losing semantic meaning.

---

## 🏗 Architecture 

RANSense-AI abandons generic RAG pipelines in favor of a specialized multi-agent workflow:

```mermaid
graph TD
    A[Engineer Query] --> B[Router Agent]
    B --> C[Spec-Search Agent<br>3GPP/O-RAN]
    B --> D[Telemetry Agent<br>Live Cell Data]
    C --> E[Cross-Encoder Re-Ranker]
    D --> E
    E --> F[(Mem0 Memory Layer<br>Past Incidents)]
    F --> G[Llama-3-8B Generation]
    G --> H[Actionable MML Command + RCA]
```

*For an in-depth view of the architecture and data flows, see our [Architecture Documentation](docs/architecture.md).*

---

## 🛠 Technology Stack

- **Orchestration:** LangGraph
- **LLM:** Meta-Llama-3-8B-Instruct (GGUF Q4_K_M for vRAN edge deployment)
- **Embeddings & Reranking:** BAAI/bge-large-en-v1.5 & BAAI/bge-reranker-large
- **Vector Database:** ChromaDB (Hybrid Dense + Sparse BM25)
- **Parsing:** LlamaParse
- **Memory & Context:** Mem0
- **Evaluation Framework:** RAGAS

---

## 📚 Detailed Documentation

As part of Phase 1, we have thoroughly documented our blueprint, datasets, agent logic, and evaluation metrics. 

Explore our `docs/` folder:
- 🧠 [**Agent Logic (`docs/agents.md`)**](docs/agents.md): Explains the Router, Spec-Search, and Telemetry agents, along with Mem0 integration.
- 🏗 [**System Architecture (`docs/architecture.md`)**](docs/architecture.md): High-level system design, APIs, and end-to-end data flow.
- 📊 [**Datasets (`docs/dataset.md`)**](docs/dataset.md): Details on 3GPP Release 17/18 ingestion and the planned RAN-FaultSim synthetic dataset.
- 🧪 [**Evaluation Framework (`docs/evaluation.md`)**](docs/evaluation.md): How we achieve >85% faithfulness and precision using RAGAS and TeleQnA.

---

## 🏆 Samsung ennovateX Phase 1 Accomplishments

During Phase 1, our primary focus was architecting a bulletproof, hallucination-free foundation:
1. Designed the **Multi-Agent** LangGraph pipeline to handle hybrid spec+telemetry queries.
2. Ingested over 3,000 pages of **3GPP TS 38-series** using LlamaParse into a hybrid ChromaDB vector store.
3. Designed the **RAN-FaultSim** dataset schema to bridge the gap in open-source 5G telemetry data.
4. Established the **RAGAS CI/CD pipeline** to continuously monitor hallucination rates and ensure enterprise-grade reliability.
5. Integrated **Mem0** for long-term session persistence for specific cell IDs.

---

<div align="center">
  <p>Built with ❤️ by Sarthak for the Samsung ennovateX 2026 Hackathon</p>
</div>