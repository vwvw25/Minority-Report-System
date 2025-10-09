# System Overview — Minority Report System  
**File:** `system_overview.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
This document provides a high-level overview of the **Minority Report System (MRS)** architecture — how data flows through the pipeline, how each stage interacts, and how the system achieves deterministic, replayable anomaly detection and attribution.

The overview connects concepts defined in:
- `architecture/Strategy.md`
- `stages/*_stage.md`
- `architecture/governance_hooks.md`

---

## 2. Architectural Summary  

**System goal:**  
Identify, cluster, and attribute short-term sales anomalies in a reproducible, enterprise-safe way.

**Core characteristics:**  
- **Log-driven:** every transform appends immutable logs.  
- **Stateless:** all state derived from logs at runtime.  
- **Idempotent:** identical inputs yield identical outputs.  
- **Acyclic:** rereview runs downstream only; no circular dependencies.  
- **Governed:** every record traceable via deterministic IDs.  

---

## 3. High-Level Flow  

    sales_timeseries_data
            │
            ▼
    [Detection Stage]
     detect_anomalies.py
     ├── baseline_sales_model_prediction_log
     ├── anomaly_classifier_model_prediction_log
     └── minority_reports_detected_log (MRDL)
            │
            ▼
    [Clustering Stage]
     feature_vector_cluster_match.py
     └── minority_reports_clustered_log (MRCL)
            │
            ▼
    [Attribution Stage]
     propose_cause_for_minority.py
     └── minority_reports_proposed_attribution_log (MRPAL)
            │
            ▼
    [Cohorting Stage]
     cohort_reports.py
     ├── minority_reports_cohorted_log (MRCH)
     └── minority_event_log (MELOG)
            │
            ▼
    [Finalisation Stage]
     build_minority_reports_finalised_log_from_edits.py
     └── minority_reports_finalised_log (MRFL)
            │
            ▼
    [Hydration Stage]
     hydrate_minority_reports.py
     ├── minority_reports (hydrated)
     └── minority_events_log (wide event view)
            │
            ▼
    [Rereview Stage]
     (optional downstream loop)
     ├── build_rereview_worklist.py
     ├── rereview_cluster_reports.py
     └── rereview_propose_cause.py

---

## 4. Data Contracts  

| Dataset | Type | Description |
|----------|------|-------------|
| `MRDL` | Log | Authoritative detection log. |
| `MRCL` | Log | Cluster assignments and metrics. |
| `MRPAL` | Log | Proposed attribution causes. |
| `MRCH` | Log | Event snapshots. |
| `MRFL` | Log | Analyst-finalised causes. |
| `minority_reports` | View | Hydrated, one-row-per-report object for UI. |
| `minority_events_log` | View | One-row-per-event aggregation for cohort view. |

Each dataset is append-only, typed, and reproducible.  
Hydration merges them deterministically to materialise current state.

---

## 5. Governance Hooks  
Governance fields are consistent across all logs:  
`report_id`, `run_id`, `written_at`, `report_status`, `is_degraded`.  
They ensure auditability, replayability, and regulatory traceability.  
See `architecture/governance_hooks.md` for detail.

---

## 6. Operational Model  
- **Failure isolation:** Each stage reads only upstream logs. A failure never cascades.  
- **Graceful degradation:** Missing inputs yield NULLs; UI never breaks.  
- **Rebuildability:** Any point-in-time state can be reproduced from logs.  
- **Observability:** Health metrics (freshness, coverage, error rate) computed continuously.  

---

## 7. User Interaction Path  
1. Analyst views hydrated minority reports in the Workshop UI.  
2. Filters by confidence, cause, or cluster to prioritise review.  
3. Edits or confirms causes → writes to `ui_edits_log`.  
4. Finalisation merge produces `MRFL`.  
5. Optional rereview loop reprocesses low-confidence historical cases.  

---

## 8. System Guarantees  
| Property | Description |
|-----------|-------------|
| **Determinism** | Re-running pipeline produces identical outputs. |
| **Immutability** | Logs never mutate; all history preserved. |
| **Traceability** | Each field traceable via `report_id` lineage. |
| **Audit readiness** | Every change stamped with author + timestamp. |
| **Extensibility** | Real ML models can replace mocks without schema change. |

---

**Summary:**  
The Minority Report System is a self-contained, log-driven pipeline demonstrating how anomaly detection, attribution, and human review can be operationalised in an enterprise-grade, auditable architecture.  
Every dataset, model, and decision is reproducible and traceable from raw sales data to analyst action.
