# Rereview Stage — Minority Report System  
**File:** `rereview_stage.md`  

---

## Overview  
The rereview stage re-runs **finalised minority reports** through the same clustering and attribution models used in the main flow.  
Its goal is to **re-evaluate lower-confidence historical cases** once models are better trained — without breaking the system’s acyclic, log-driven architecture.  

Implemented as three transforms:  
1. `build_rereview_worklist.py`  
2. `rereview_cluster_reports.py`  
3. `rereview_propose_cause.py`  

All outputs are written to dedicated append-only rereview logs; originals remain untouched. Hydration merges rereview results with precedence logic.

---

## 1. `build_rereview_worklist.py`  
Builds a controlled queue of reports eligible for rereview.  

- **Inputs:** `minority_reports_finalised_log` (MRFL), `demo_run_config`.  
- **Output:** `minority_reports_rereview_worklist`.  
- **Logic:**  
  - Runs only if the main system is operational.  
  - Selects reports older than **6 months** with confidence > **50%**.  
  - Appends one row per qualifying report to the worklist.  

---

## 2. `rereview_cluster_reports.py`  
Re-runs clustering for reports in the worklist.  

- **Inputs:** rereview worklist, `sales_timeseries_data`, `minority_reports_finalised_log`.  
- **Output:** `minority_reports_clustered_rereview_log`.  
- **Logic:**  
  - Slice sales data (`window_start → window_end` from MRFL).  
  - Build feature vectors and pass through the clustering model.  
  - Append new results; **MRCL is never modified.**  

---

## 3. `rereview_propose_cause.py`  
Re-runs attribution for rereviewed reports.  

- **Inputs:** rereview worklist, `sales_timeseries_data`, `minority_reports_finalised_log`.  
- **Output:** `minority_reports_proposed_attribution_rereview_log`.  
- **Logic:**  
  - Gather slices and metadata from MRFL.  
  - Pass through the attribution model to generate new causes and confidence scores.  
  - Append results; **MRPAL remains unchanged.**  

---

## 4. Acyclic Design  
- Rereview outputs (`*_rereview_log`) exist in parallel to core logs.  
- No upstream datasets are overwritten.  
- Hydration merges rereview and original logs under strict precedence.  
- DAG remains acyclic, replayable, and audit-safe.  

---

## 5. Rationale  
Rereview provides a **non-invasive improvement loop**:  
- Targets lower-confidence, historical reports.  
- Leverages updated model weights without risking live instability.  
- Maintains full lineage — both original and rereviewed outputs coexist for comparison.  
- Operates only when the main system is healthy, ensuring reliability and governance compliance.

---

**Summary:**  
Rereview is a downstream-only feedback mechanism that enhances attribution accuracy over time while preserving determinism, acyclicity, and auditability — a design pattern suitable for enterprise-grade production systems.
