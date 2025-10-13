# Rereview Stage — Minority Report System  
**File:** `rereview_stage.md`  

---

## Overview  
The rereview stage re-runs **finalised Minority Reports** through the same clustering and attribution transforms used in the main pipeline.  
Its goal is to **re-evaluate older or low-confidence historical reports** once new data, updated model logic, or revised thresholds become available —  
without breaking the system’s acyclic, log-driven architecture.  

Implemented as three transforms:  
1. `build_rereview_worklist.py`  
2. `rereview_cluster_reports.py`  
3. `rereview_propose_cause.py`  

All outputs are written to dedicated append-only rereview logs; originals remain untouched.  
During hydration, rereview results take precedence when a newer `rereview_round` exists.  

---

## 1. `build_rereview_worklist.py`  
Constructs a controlled queue of reports eligible for rereview.  

- **Inputs:**  
  - `minority_reports_finalised_log` (MRFL)  
  - Optionally the hydrated `minority_reports` (for latest state checks)  
  - `demo_run_config`  

- **Output:**  
  - `minority_reports_rereview_worklist` (MRRW)  

- **Logic:**  
  - Runs only when the main system is healthy.  
  - Selects reports that meet one or more of the following conditions:  
    - Older than **6 months**,  
    - **Confidence < 0.5**, or  
    - Affected by **model version change** or **schema drift**.  
  - Appends one row per qualifying report to the worklist with the trigger reason.  

---

## 2. `rereview_cluster_reports.py`  
Re-runs clustering for reports in the rereview worklist.  

- **Inputs:**  
  - `minority_reports_rereview_worklist`  
  - `sales_timeseries_data`  
  - `minority_reports_finalised_log`  

- **Output:**  
  - `minority_reports_clustered_rereview_log`  

- **Logic:**  
  - Extracts sales slices using `window_start → window_end` from MRFL.  
  - Rebuilds feature vectors for each report and passes them through the clustering model.  
  - Appends new results tagged with rereview metadata.  
  - **MRCL (main clustering log) is never modified.**  

> **MVP Note:** In the current demo build, the clustering model is mocked.  
> The rereview stage reuses the same lookup logic and seeded clusters from `master_narratives`,  
> preserving architectural fidelity without executing a new model run.  

---

## 3. `rereview_propose_cause.py`  
Re-runs attribution for rereviewed reports.  

- **Inputs:**  
  - `minority_reports_rereview_worklist`  
  - `sales_timeseries_data`  
  - `minority_reports_finalised_log`  

- **Output:**  
  - `minority_reports_proposed_attribution_rereview_log`  

- **Logic:**  
  - Gathers sales slices and metadata from MRFL.  
  - Passes them through the attribution model to produce refreshed causes and confidence scores.  
  - Appends new results; **MRPAL (main attribution log) remains unchanged.**  

> **MVP Note:** As with clustering, this stage does not retrain or re-infer a model.  
> It simulates the rereview process using deterministic lookups and static mock data.  

---

## 4. Acyclic Design  
- Rereview outputs (`*_rereview_log`) exist **in parallel** to their main counterparts.  
- No upstream logs are ever overwritten.  
- Hydration merges rereview results with strict precedence:  
  - if `rereview_round > 0`, rereview data supersedes the original for that report.  
- The DAG remains **acyclic, replayable, and audit-safe**.  

---

## 5. Rationale  
Rereview provides a **non-invasive improvement loop** that allows the system to learn and self-correct over time without instability.  

- Targets **low-confidence or outdated** reports.  
- Leverages updated thresholds, schemas, or model logic safely.  
- Maintains complete lineage — original and rereviewed outputs coexist for comparison.  
- Operates only when the core pipeline is healthy.  
- Fully compatible with append-only and idempotent system design.  

---

### Output Schema — `minority_reports_rereview_worklist (MRRW)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic ID linking back to the affected Minority Report (`hash(store_id || first_detected_at)`). |
| `round` | integer | Rereview iteration number (e.g., 1 = first rereview cycle). |
| `trigger_reason` | string | Reason rereview was initiated (e.g., *low confidence*, *model update*, *schema drift*, *analyst override*). |
| `trigger_at` | timestamp | Timestamp when rereview trigger was generated. |
| `outputs_generated_at` | timestamp | Timestamp when rereview outputs were produced (not re-finalised). |

**Properties**
- Tracks **post-finalisation quality assurance cycles** for Minority Reports.  
- Automatically populated when new data or models necessitate re-evaluation.  
- Enables selective re-submission to the HITL interface or automated processing.  
- Ensures full reproducibility and versioned lineage via `round` and `trigger_reason`.  
- Allows direct comparison between original and rereviewed outputs.  

---

**Summary:**  
The rereview stage acts as a downstream feedback mechanism — enhancing model accuracy,  
data completeness, and confidence calibration over time.  
By isolating rereview into its own append-only logs and merge layer,  
the system maintains full determinism, auditability, and architectural stability while allowing continuous refinement.
