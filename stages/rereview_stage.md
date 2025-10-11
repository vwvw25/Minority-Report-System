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

### Output Schema — `minority_reports_rereview_worklist (MRRW)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic ID linking back to the affected Minority Report (`hash(store_id || first_detected_from)`). |
| `round` | integer | Indicates the re-review iteration number (e.g., 1 = first re-review cycle). |
| `trigger_reason` | string | Reason for triggering re-review (e.g., *model update*, *data drift*, *analyst override*, *conflict detection*). |
| `trigger_at` | timestamp | Timestamp when the re-review trigger was generated. |
| `finalized_at` | timestamp | Timestamp when the re-review was completed and the record re-finalized. |

**Properties**
- Tracks **post-finalisation quality assurance loops** for Minority Reports.  
- Populated automatically when upstream schema, model, or metadata changes require a manual or automated re-evaluation.  
- Allows controlled re-submission of reports to the HITL interface or automated pipeline.  
- Ensures lineage and decision reproducibility by versioning the review process (via `round`).  
- Enables full audit trail of why and when a report was revisited.

---

**Summary:**  
Rereview is a downstream-only feedback mechanism that enhances attribution accuracy over time while preserving determinism, acyclicity, and auditability — a design pattern suitable for enterprise-grade production systems.
