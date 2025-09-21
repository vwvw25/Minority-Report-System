# Rereview Stage — Minority Report System  
**File:** `rereview_stage.md`  
**Last Updated:** 2025-09-21  

---

## Overview  
The rereview stage provides a clean way to re-run clustering and attribution for already-finalised minority reports, without overwriting original logs.  
It is implemented as **three transforms**:  

1. `build_rereview_worklist.py` → queue of reports flagged for rereview.  
2. `rereview_cluster_reports.py` → re-run clustering for those reports.  
3. `rereview_propose_cause.py` → re-run attribution for those reports.  

All three write to **separate append-only logs**. Hydration merges rereview outputs with originals under strict precedence rules, keeping the system **stateless and acyclic**.  

---

## 1. Transform: `build_rereview_worklist.py`  

### Purpose  
Selects minority reports requiring rereview and builds a dedicated worklist.  

### Inputs  
- `minority_reports_finalised_log` (MRFL) — authoritative source of HITL outcomes.  
- `demo_run_config`.  

### Outputs  
- `minority_reports_rereview_worklist`  
  - Fields: `report_id`, `store_id`, `sku`, `reason_code`, `requested_at`, `run_id`.  

### Logic  
- Filter MRFL for rows flagged as `needs_rereview` (by user or system rule).  
- Append one row per flagged report to the worklist log.  

---

## 2. Transform: `rereview_cluster_reports.py`  

### Purpose  
Re-runs clustering on reports from the rereview worklist.  

### Inputs  
- `minority_reports_rereview_worklist`.  
- `sales_timeseries_data`.  
- Optional: `baseline_sales_model_prediction_log`.  
- `demo_run_config`.  

### Outputs  
- `minority_reports_clustered_rereview_log`  
  - Mirrors schema of MRCL.  
  - Fields: `report_id`, `feature_vector`, `cluster_id`, `cluster_name`,  
    `similarity_score`, `cluster_match_confidence`, `tsne_x`, `tsne_y`,  
    `written_at`, `run_id`.  

### Logic  
- For each report_id in the worklist:  
  - Slice sales data using `window_start → window_end` (from MRFL).  
  - Build feature vector.  
  - Run clustering model.  
- Write results as new log rows; originals in MRCL remain untouched.  

---

## 3. Transform: `rereview_propose_cause.py`  

### Purpose  
Re-runs attribution for rereviewed reports, using contextual and campaign data.  

### Inputs  
- `minority_reports_clustered_rereview_log`.  
- `sales_timeseries_data`.  
- `contextual_data`, `all_campaign_data`.  
- `demo_run_config`.  

### Outputs  
- `minority_reports_proposed_attribution_rereview_log`  
  - Mirrors schema of MRPAL.  
  - Fields: `report_id`, `proposed_cause`, `proposed_cause_category`,  
    `confidence_score`, `supporting_evidence`,  
    `baseline_sales_total`, `attributed_sales_total`,  
    `proposed_cause_2/_3`, `confidence_score_2/_3`,  
    `report_status`, `written_at`, `run_id`.  

### Logic  
- For each rereviewed cluster:  
  - Evaluate campaign overlaps and contextual signals.  
  - Propose cause(s) + confidence scores.  
- Write results as new log rows; originals in MRPAL remain untouched.  

---

## 4. Acyclic Design  

- **No rereview transform writes back to MRCL, MRPAL, or MRFL.**  
- All rereview outputs are written to `*_rereview_log`.  
- Hydration unions rereview logs with originals:  
  - If rereview exists → it takes precedence in the **hydrated view only**.  
  - Originals remain immutable for audit.  

This ensures:  
- Auditability (all proposals preserved).  
- Stateless replays (hydrated object always reproducible from logs).  
- DAG remains acyclic (no feedback into upstream transforms).  

---

## 5. Flow Summary  

1. **Finalised reports** (MRFL) → `build_rereview_worklist.py` → `minority_reports_rereview_worklist`.  
2. **Worklist** → `rereview_cluster_reports.py` → `minority_reports_clustered_rereview_log`.  
3. **Cluster rereview log** → `rereview_propose_cause.py` → `minority_reports_proposed_attribution_rereview_log`.  
4. **Hydration** merges rereview logs with originals under precedence rules.  

---
