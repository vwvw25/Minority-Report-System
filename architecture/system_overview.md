# System Overview — Minority Report System  
**File:** `system_overview.md`  
**Last Updated:** 2025-10-11  

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
    [User Edit Stages]
     ui_edits_log.py
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
| `user_edits_log` | Log | Analyst edits made to individual reports via the Workshop UI. |
| `user_minority_events_edits_log` | Log | Analyst edits made to the **Minority Event** object, affecting all linked reports in a cohort. |
| `MRFL` | Log | Analyst-finalised causes, combining model outputs and all edits. |
| `minority_reports` | View | Hydrated, one-row-per-report object for UI. |
| `minority_events_log` | View | One-row-per-event aggregation for cohort view. |

Each dataset is append-only, typed, and reproducible.  
Hydration merges them deterministically to materialise current state.

---

## 4.1 Edit & Annotation Layer  

Both report-level and event-level edits are captured in dedicated logs for full traceability and replayability.  

| Log | Scope | Description |
|------|--------|-------------|
| `user_edits_log` | Report-level | Captures analyst changes to individual reports (e.g., confirming or correcting proposed causes). |
| `user_minority_events_edits_log` | Event-level | Captures analyst changes to the parent event object, applying to all related reports within a cohort. |

Edits from both sources are merged into the **Finalised Log (MRFL)** to create an auditable record of all human and machine interventions.

---

## 5. Identity and Lineage  

Each anomaly and cohort has deterministic identity keys to guarantee lineage and reproducibility.

- `report_id = hash(store_id || first_detected_at)` uniquely identifies each anomaly.  
- `report_group_id = hash(store_id || earliest_first_detected_at_in_cohort)` groups related anomalies into persistent cohorts.

This ensures idempotent recomputation, consistent joins across logs, and complete auditability across all pipeline stages.

---

## 6. Governance Hooks  
Governance fields are consistent across all logs:  
`report_id`, `run_id`, `written_at`, `report_status`, `is_degraded`.  
They ensure auditability, replayability, and regulatory traceability.  
See `architecture/governance_hooks.md` for detail.

---

## 7. Operational Model  
- **Failure isolation:** Each stage reads only upstream logs. A failure never cascades.  
- **Graceful degradation:** Missing inputs yield NULLs; UI never breaks.  
- **Rebuildability:** Any point-in-time state can be reproduced from logs.  
- **Observability:** Health metrics (freshness, coverage, error rate) computed continuously.

### Alerting and Notification

When a new minority event exceeds a defined severity threshold (typically ≥ 0.9) or matches high-impact categories (e.g., viral, competitor stockout), the system triggers an alert.  

- **Source:** Alerts are generated where reports or event meet set criteria. 
- **Recipients:** Users and managers subscribed to relevant categories.  
- **Delivery:** Email and in-platform notification (UI entry point opens directly to the corresponding Minority Event screen).  
- **Governance:** Every alert is logged to `ui_notifications_log` with `event_id`, `trigger_condition`, and `delivered_to`.  

This ensures operational awareness of critical anomalies without requiring constant manual monitoring of dashboards.

---

## 8. User Interaction Path  
1. Analyst views hydrated minority reports in the Workshop UI.  
2. Filters by confidence, cause, or cluster to prioritise review.  
3. Makes one or more edits:  
   - **Report-level:** updates specific fields (e.g., cause, confidence) → logged to `user_edits_log`.  
   - **Event-level:** updates shared cohort metadata (e.g., cause, category, confidence) → logged to `user_minority_events_edits_log`.  
4. Finalisation merge (`build_minority_reports_finalised_log_from_edits.py`) consolidates both edit layers with model outputs to produce `MRFL`.  
5. Optional rereview loop reprocesses low-confidence historical cases.  

---

## 9. System Guarantees  
| Property | Description |
|-----------|-------------|
| **Determinism** | Re-running pipeline produces identical outputs. |
| **Immutability** | Logs never mutate; all history preserved. |
| **Traceability** | Each field traceable via `report_id` lineage. |
| **Audit readiness** | Every change stamped with author + timestamp. |
| **Extensibility** | Real ML models can replace mocks without schema change. |

---

## 10. Demo Reset Design  

The Minority Report System uses a **Run-ID architecture** that allows clean demo resets without losing historical data.  

- Every dataset includes a `run_id` column.  
- The active `run_id` is stored in `demo_run_config`.  
- All read-only views automatically filter to that value, isolating the current run.  

**Benefits**  
- Instant reset while preserving previous sessions.  
- Enables before-and-after comparisons for reviewers.  
- Guarantees reproducible lineage across multiple demos.  

```sql
SELECT * 
FROM proposals_union p
JOIN demo_run_config c ON p.run_id = c.run_id
WHERE p.status = '3_attribution_proposed';
