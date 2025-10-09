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

```text
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
