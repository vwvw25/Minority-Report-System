# System Overview — Minority Report System  
**File:** `system_overview.md`  
**Last Updated:** 2025-10-13  

---

## 1. Purpose  
This document provides a high-level overview of the **Minority Report System (MRS)** architecture — how data flows through the pipeline, how each stage interacts, and how the system achieves deterministic, replayable anomaly detection and attribution.

The overview connects concepts defined in:
- `architecture/strategy.md`
- `stages/*_stage.md`
- `architecture/governance_hooks.md`

---

## 2. Overview  

The MRS is part of a broader analytics platform combining an MMM and a MAB to answer the question:  
> “Which combination of TPO and marketing campaigns will produce the highest ROI on spend in June?”

While the MMM sets total budgets for TPO and marketing campaigns at a **monthly** level, the MAB allocates those budgets **daily** across auction-based platforms (e.g. Facebook, TikTok, Programmatic).  

Accurate attribution of sales to campaigns is critical to ensuring the MAB allocates budget efficiently.  
The MRS specifically identifies **sales anomalies** and establishes their causes to prevent them being misattributed to paid campaigns — misattribution that could otherwise cause the MAB to increase spend on the wrong activities.

---

## 3. Architectural Summary  

**System goal:**  
Identify, cluster, and attribute short-term sales anomalies in a reproducible, enterprise-safe way.

**Core characteristics:**  
- **Log-driven:** every transform appends immutable logs.  
- **Stateless:** all state derived from logs at runtime.  
- **Idempotent:** identical inputs yield identical outputs.  
- **Acyclic:** rereview runs downstream only; no circular dependencies.  
- **Governed:** every record traceable via deterministic IDs.  

---

## 4. High-Level Flow

unified_sales_data  →  sales_timeseries_data
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
 cluster_minority_report.py
 └── minority_reports_clustered_log (MRCL)
            │
            ▼
[Attribution Stage]
 propose_cause_for_minority.py
 └── minority_reports_proposed_attribution_log (MRPAL)
            │
            ▼
[Cohorting Stage]
 build_minority_report_cohorts.py
 ├── minority_reports_cohorted_log (MRCH)
 └── minority_event_log (MELOG)
            │
            ▼
[User Edit Stages — Workshop UI]
 ├── user_edits_log (report-level)
 └── user_minority_events_edits_log (event-level)
            │
            ▼
[Finalisation Stage]
 build_minority_reports_finalised_log_from_edits.py
 └── minority_reports_finalised_log (MRFL)
            │
            ▼
[Hydration Stage]
 hydrate_minority_reports.py
 ├── minority_reports (hydrated dataset)
 └── minority_events_log (hydrated dataset)
            │
            ▼
[Rereview Stage — optional]
 ├── build_rereview_worklist.py
 ├── rereview_cluster_reports.py
 └── rereview_propose_cause.py
 
---

## 5. Data Contracts  

| Dataset | Type | Description |
|---------|------|-------------|
| `MRDL` | Log | Authoritative detection log. |
| `MRCL` | Log | Cluster assignments and similarity metrics. |
| `MRPAL` | Log | Proposed attribution causes and confidence. |
| `MRCH` | Log | Per-tick snapshots of cohort (event) evolution. |
| `MELOG` | Log | One stable row per event (event registry). |
| `user_edits_log` | Log | Analyst edits to individual reports via the UI. |
| `user_minority_events_edits_log` | Log | Analyst edits to the **Minority Event** object (cohort-level). |
| `MRFL` | Log | Finalised report rows (merge of model outputs + edits; used for rereview). |
| `minority_reports` | Dataset | Hydrated, one-row-per-report object used by the UI. |
| `minority_events_log` | Dataset | Hydrated, one-row-per-event object used by the UI. |

Each dataset is append-only, typed, and reproducible.  
Hydration deterministically assembles the **current state** for UI consumption without mutating upstream logs.

---

## 6. Identity and Lineage  

Deterministic identity keys guarantee lineage and reproducibility:

- `report_id = hash(store_id || first_detected_at)` uniquely identifies each anomaly.  
- `report_group_id = hash(report_id || earliest_first_detected_at_in_cohort)` groups anomalies into persistent cohorts (Minority Events).

This ensures idempotent recomputation, consistent joins, and complete auditability across all pipeline stages.

---

## 7. Governance Hooks  
Governance fields are consistent across all logs:  
`report_id`, `run_id`, `written_at`, `report_status`, `is_degraded`.  
They ensure auditability, replayability, and regulatory traceability.  
See `architecture/governance_hooks.md` for detail.

---

## 8. Operational Model  

- **Failure isolation:** Each stage reads only defined upstream logs; failures do not cascade.  
- **Graceful degradation:** Missing inputs yield NULLs in their columns; the UI remains functional.  
- **Rebuildability:** Any point-in-time state can be reconstructed from logs.  
- **Observability:** Health metrics (freshness, coverage, error rate) are computed continuously.

### 8.1 Alerting and Notification (Implemented)
When a new Minority Event exceeds a configured severity threshold (e.g., ≥ 0.9) or matches high-impact categories (e.g., viral, competitor stockout), the system triggers an alert.

- **Source:** Alerts are emitted when report/event criteria are met.  
- **Recipients:** Subscribed users and managers for the relevant scope (retailer/region/category).  
- **Delivery:** Email and in-platform notification (link opens the corresponding Minority Event in the UI).  
- **Audit:** Every alert is logged to `ui_notifications_log` with `event_id`, `trigger_condition`, `delivered_to`, and `written_at`.

---

## 9. User Interaction Path  
1. Analyst views hydrated **Minority Reports** and **Minority Events** in the Workshop UI.  
2. Filters by confidence, cause, or cluster to prioritise review.  
3. Makes edits:  
   - **Report-level** → `user_edits_log` (e.g., cause correction, approval).  
   - **Event-level** → `user_minority_events_edits_log` (e.g., cohort cause, evidence).  
4. Finalisation (`build_minority_reports_finalised_log_from_edits.py`) merges edits and model outputs into **MRFL** (authoritative for rereview).  
5. Optional rereview loop reprocesses selected historical cases with updated models.

---

## 10. System Guarantees  

| Property | Description |
|---------|-------------|
| **Determinism** | Re-running the pipeline produces identical outputs. |
| **Immutability** | Logs never mutate; all history is preserved. |
| **Traceability** | Each field is traceable via `report_id`/`report_group_id` lineage. |
| **Audit readiness** | Every change is stamped with author and timestamp. |
| **Extensibility** | Mocked models can be swapped for real ones without schema change. |

---

## 11. Demo Reset Design  

The MRS uses a **Run-ID architecture** that allows clean demo resets without losing historical data:

- Every dataset includes a `run_id` column.  
- The active `run_id` is stored in `demo_run_config`.  
- Read paths filter by that value, isolating the current run.

**Benefits**  
- Instant reset while preserving previous sessions.  
- Enables before/after comparisons for reviewers.  
- Guarantees reproducible lineage across multiple demos.

Example query:
