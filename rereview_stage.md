# Rereview Stage — Minority Report System
**File:** `rereview_stage.md`  
**Last Updated:** 2025-09-21  

---

## Overview
The rereview stage is designed to re-run already finalised minority reports back through the **same clustering and attribution models** used in the main classification flow.  
The purpose is to improve attribution on lower-confidence reports once models have been more highly trained, without breaking the system’s acyclic, log-driven architecture.  

Rereview is implemented as three transforms:
1. `build_rereview_worklist.py`  
2. `rereview_cluster_reports.py`  
3. `rereview_propose_cause.py`  

All rereview outputs are written to **separate append-only logs**, never overwriting the originals. Hydration reads rereview logs alongside the originals, applying precedence where applicable.  

---

## 1. Transform: `build_rereview_worklist.py`

### Purpose
Build a controlled queue of reports eligible for rereview.  

### Inputs
- `minority_reports_finalised_log` (MRFL)  
- `demo_run_config`  

### Outputs
- `minority_reports_rereview_worklist`  

### Logic
- Checks that the **main system is operational**. If not, the transform does not write logs (ensuring rereview never occurs during partial outages).  
- Selects reports that meet the **conditional logic**:  
  - Report is older than **6 months**  
  - Confidence score > **50%**  
- Appends one row per qualifying report to the rereview worklist.  

---

## 2. Transform: `rereview_cluster_reports.py`

### Purpose
Re-run clustering for rereviewed reports using the **same clustering model** as in the main system.  

### Inputs
- `minority_reports_rereview_worklist`  
- `sales_timeseries_data`  
- `minority_reports_finalised_log` (MRFL)  

### Outputs
- `minority_reports_clustered_rereview_log`  

### Logic
- For each report in the worklist:  
  - Slice sales data using `window_start → window_end` from MRFL.  
  - Build a feature vector over that slice.  
  - Pass the feature vector to the clustering model.  
- Write results to the rereview clustering log (MRCL is never modified).  

---

## 3. Transform: `rereview_propose_cause.py`

### Purpose
Re-run attribution for rereviewed reports using the **same attribution model** as in the main system.  

### Inputs
- `minority_reports_rereview_worklist`  
- `sales_timeseries_data`  
- `minority_reports_finalised_log` (MRFL)  

### Outputs
- `minority_reports_proposed_attribution_rereview_log`  

### Logic
- For each report in the worklist:  
  - Gather time-series slices and metadata from MRFL.  
  - Pass inputs through the attribution model.  
  - Generate proposed cause(s) and updated confidence scores.  
- Write results to the rereview attribution log (MRPAL is never modified).  

---

## 4. Acyclic Design

- **No rereview transform overwrites MRCL, MRPAL, or MRFL.**  
- All rereview outputs are written to dedicated `*_rereview_log` datasets.  
- Hydration merges rereview logs with originals under strict precedence rules.  
- Because rereview logs are downstream-only, the DAG remains acyclic and fully replayable.  

---

## 5. Rationale

- Reports selected are **lower-confidence historical cases**.  
- By re-running them through better-trained clustering and attribution models, the system improves the overall quality of attributions.  
- Audit trail is preserved: both original and rereviewed proposals are available in logs.  
- Rereview is **conditional, safe, and non-invasive**:  
  - Only runs if the main system is operational  
  - Never rewrites existing logs  
  - Isolates outputs in rereview logs  
