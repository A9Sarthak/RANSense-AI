# RANSense-AI — Evaluation Framework

## Overview

This document defines the evaluation methodology, metrics, benchmarks, and target performance for RANSense-AI. All evaluation uses open-source tools on publicly available datasets — no proprietary benchmarks.

---

## Evaluation Framework: RAGAS

We use **RAGAS** (Retrieval-Augmented Generation Assessment) — an open-source framework (Apache-2.0) for automated, reference-free evaluation of RAG pipelines.

| Field | Details |
|---|---|
| Library | RAGAS (Apache-2.0) |
| GitHub | https://github.com/explodinggradients/ragas |
| Integration | CI/CD pipeline — runs on every commit to `src/` |

---

## Metrics

### 1. Context Precision
**What it measures:** Of all the 3GPP/O-RAN chunks retrieved, what fraction are actually relevant to the query?

**Formula:**
```
Context Precision = (Relevant chunks retrieved) / (Total chunks retrieved)
```

**Why it matters:** Low context precision means the LLM is being fed irrelevant spec pages, increasing hallucination risk.

**Our target:** >85%
**Baseline (generic RAG):** ~52%

---

### 2. Answer Faithfulness
**What it measures:** Is every claim in the generated RCA grounded in the retrieved context? Detects hallucination.

**Formula:**
```
Faithfulness = (Claims in answer supported by context) / (Total claims in answer)
```

**Why it matters:** In telecom, a hallucinated MML command can cause a real cell outage. Faithfulness must be very high.

**Our target:** >85%
**Baseline (generic RAG):** ~61%

---

### 3. Answer Relevancy
**What it measures:** Does the generated response actually address what the engineer asked?

**Formula:**
```
Relevancy = cosine_similarity(question_embedding, answer_embedding)
```

**Our target:** >80%

---

### 4. MTTR Reduction (End-to-End Business Metric)
**What it measures:** Time from fault alarm to actionable resolution command.

| Approach | Time |
|---|---|
| Manual (engineer + docs) | ~4 hours |
| RANSense-AI | <15 minutes |
| **Improvement** | **75% reduction** |

---

## Benchmark Dataset

**TeleQnA** (HuggingFace: `ziv/TeleQnA`, CC BY 4.0)

| Field | Details |
|---|---|
| Total QA pairs | 10,000+ |
| Split used | 80% retrieval training / 20% held-out evaluation |
| Evaluation split | 2,000 questions |
| Topics | NR, LTE, 5GC, O-RAN |
| Format | Multiple-choice + open-ended |

**Evaluation procedure:**
1. Feed each TeleQnA question to RANSense-AI as a query
2. RANSense-AI retrieves 3GPP context and generates an answer
3. RAGAS scores Context Precision, Faithfulness, Relevancy automatically
4. Compare against TeleQnA ground truth answers

---

## Baseline Comparisons

| System | Context Precision | Answer Faithfulness | MTTR |
|---|---|---|---|
| Generic LangChain RAG (no telecom tuning) | ~52% | ~61% | N/A |
| TelcoBERT (Ericsson) — spec retrieval only | ~71% | ~69% | ~2 hrs |
| Nokia AVA — rule-based, no generative | N/A | N/A | ~1.5 hrs |
| **RANSense-AI (target)** | **>85%** | **>85%** | **<15 min** |

---

## CI/CD Evaluation Pipeline

Every code commit to `src/` triggers automatic RAGAS evaluation:

```yaml
# .github/workflows/evaluate.yml
on: [push]
jobs:
  ragas-eval:
    steps:
      - name: Run RAGAS on TeleQnA held-out set
        run: python src/evaluate/run_ragas.py --dataset teleqna --split eval
      - name: Assert thresholds
        run: |
          assert context_precision > 0.85
          assert answer_faithfulness > 0.85
      - name: Post results to README badge
```

This ensures every change to the retrieval pipeline or prompt engineering is measured against our targets before merging.

---

## Ablation Study Plan (Phase 2)

We plan to compare these configurations to justify each design decision:

| Ablation | What we remove | Expected drop |
|---|---|---|
| No LlamaParse (use PyPDF2 instead) | Structure-preserving parsing | -15% precision (tables destroyed) |
| No cross-encoder reranker | Reranking step | -10% faithfulness |
| No BM25 (dense only) | Sparse retrieval | -8% precision on keyword queries |
| No Mem0 memory | Historical context | N/A on TeleQnA (no session continuity) |
| No LoRA fine-tuning | Domain adaptation | -12% faithfulness on 3GPP-specific terms |

---

## Known Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| TeleQnA covers 4G+5G mixed | Some questions not RAN-specific | Filter to NR-only subset for evaluation |
| RAN-FaultSim is synthetic | Telemetry evaluation is simulated | Plan to validate against anonymised real traces in Phase 2 |
| Llama-3-8B context window = 8K tokens | Long 3GPP sections may be truncated | Section-level chunking keeps chunks under 512 tokens |

---

*License: Apache-2.0 | GitHub: https://github.com/A9Sarthak/RANSense-AI*
