# RANSense-AI — Dataset Documentation

## Overview

This document describes all datasets used and created for the RANSense-AI project as part of Samsung ennovateX AX Hackathon 2026 (Problem 10: RAG based Future-Ready Telecom RAN Assistant).

---

## Datasets Used (External)

### 1. 3GPP Release 17/18 Technical Specifications

| Field | Details |
|---|---|
| Source | https://www.3gpp.org/specifications |
| License | 3GPP Public (free for use with attribution) |
| Format | PDF |
| Volume | ~3,000+ pages across 4 core specifications |

**Documents ingested:**

| Spec ID | Title | Pages (approx) | Pipeline Role |
|---|---|---|---|
| TS 38.300 | NR Overall Description | ~120 | Core NR architecture reference |
| TS 38.401 | NG-RAN Architecture Description | ~180 | gNB/CU/DU split architecture |
| TS 38.411 | NG-RAN Layer 1 General Aspects | ~60 | Physical layer fault context |
| TS 38.711 | Study on NR beyond 52.6 GHz | ~200 | mmWave RAN configuration |

**Ingestion pipeline:**
- Parsed using **LlamaParse** (structure-preserving, handles complex 3GPP tables)
- Section-level chunking with metadata: `{spec_id, section_number, release_version, page_number}`
- Embedded using **BAAI/bge-large-en-v1.5** (MIT license)
- Stored in **ChromaDB** vector store with BM25 sparse index for hybrid retrieval

---

### 2. O-RAN Alliance Architecture Documents

| Field | Details |
|---|---|
| Source | https://www.o-ran.org/specifications |
| License | O-RAN Public (free for use) |
| Format | PDF |
| Coverage | WG1 through WG9 architecture specs |

**Pipeline Role:**
Supplements 3GPP specs with Open RAN interface definitions. Enables cross-reference retrieval between O-RAN fronthaul (F1/E1/Xn interfaces) and 3GPP NR air interface standards. Critical for diagnosing issues in disaggregated vRAN deployments (Samsung Networks target environment).

---

### 3. TeleQnA Benchmark Dataset

| Field | Details |
|---|---|
| HuggingFace | https://huggingface.co/datasets/ziv/TeleQnA |
| License | CC BY 4.0 |
| Size | 10,000+ QA pairs |
| Source | Derived from 3GPP, ETSI, and ITU specifications |

**Contents:**
- Multiple-choice and open-ended questions derived from telecom standards
- Covers NR, LTE, 5GC, and O-RAN topics
- Ground truth answers with specification references

**Pipeline Role:**
Used exclusively for **RAGAS evaluation** — not for training or retrieval. Serves as the held-out benchmark to measure:
- Context Precision: Are retrieved 3GPP chunks relevant?
- Answer Faithfulness: Is the LLM response grounded in retrieved context?
- Answer Relevancy: Does the response address the actual query?

**Target performance:** >85% Answer Faithfulness on held-out TeleQnA subset

---

## Dataset to be Published (Planned — Phase 2)

### 4. RAN-FaultSim: Synthetic 5G Telecom Fault Telemetry Dataset

| Field | Details |
|---|---|
| Status | To be created and published in Phase 2 |
| Publish target | HuggingFace (CC BY 4.0) |
| Format | CSV + JSON |
| Planned size | 5,000+ PM rows, 2,000+ FM alarm records |

**Why we are creating this:**
No publicly available dataset exists that combines structured 5G RAN Performance Management (PM) counter data with Fault Management (FM) alarm records mapped to root-cause labels. This gap makes it impossible to benchmark telemetry-aware RAG systems. RAN-FaultSim fills this gap.

**Planned schema:**

*PM Counter Table (performance_counters.csv):*
```
timestamp | cell_id | gnb_id | prb_utilization | pdcp_throughput_mbps |
rrc_setup_success_rate | ho_success_rate | ul_sinr_db | dl_cqi
```

*FM Alarm Table (fault_alarms.csv):*
```
timestamp | cell_id | alarm_type | alarm_severity | alarm_description |
root_cause_label | mml_resolution_command | resolution_time_mins
```

**Alarm types covered:**
- `RRC_CONNECTION_FAILURE` — Radio resource control setup failure
- `HANDOVER_DROP` — Inter-cell handover failure
- `CELL_OUTAGE` — Complete cell unavailability
- `UL_INTERFERENCE` — Uplink interference from neighbor cells
- `RESOURCE_BLOCK_CONGESTION` — PRB utilization >90%

**Root cause labels (ground truth):**
- `NEIGHBOR_CELL_MISCONFIGURATION`
- `TIMING_ADVANCE_DRIFT`
- `RF_HARDWARE_DEGRADATION`
- `CAPACITY_OVERLOAD`
- `ANTENNA_TILT_MISMATCH`

**Generation methodology:**
Simulated using domain knowledge from 3GPP TS 28.552 (KPI definitions) and real-world 5G network fault patterns documented in O-RAN Alliance WG1 specifications. Random seed + fault injection logic will be open-sourced.

**Pipeline role:**
Feeds the **Telemetry Agent** (Path B in the data flow diagram). Enables live SQL-style KPI queries during fault diagnosis sessions. Cross-correlated with 3GPP spec retrieval (Path A) by the Cross-Encoder Re-Ranker.

---

## Data Governance

| Dataset | License | PII | Proprietary Data |
|---|---|---|---|
| 3GPP TS specs | 3GPP Public | None | No |
| O-RAN docs | O-RAN Public | None | No |
| TeleQnA | CC BY 4.0 | None | No |
| RAN-FaultSim | CC BY 4.0 (planned) | None | No — fully synthetic |

All datasets comply with Samsung ennovateX open-source data rules. No proprietary, licensed, or restricted data is used at any stage of the pipeline.

---

*License: Apache-2.0 | GitHub: https://github.com/A9Sarthak/RANSense-AI*
