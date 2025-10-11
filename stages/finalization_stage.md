# Finalisation Stage — Minority Report System  
**File:** `finalisation_stage.md`  

---

## 1. Purpose  
The finalisation stage consolidates model outputs and analyst edits into the authoritative **Minority Reports Finalised Log (MRFL)**.  
Implemented in `build_minority_reports_finalised_log_from_edits.py`, it merges the **proposed attribution log (MRPAL)** with both **user-level** and **event-level** edit logs, producing a replayable record of finalised outcomes.

---

## 2. Inputs  
- **`minority_reports_proposed_attribution_log` (MRPAL):** proposed causes and confidence.  
- **`user_edits_log`:** analyst overrides and annotations at the individual report level.  
- **`user_minority_events_edits_log`:** analyst edits to event-level objects (affecting multiple reports).  
- **`demo_run_config`:** provides `run_id`.  

---

## 3. Output  
| Dataset | Purpose |
|----------|----------|
| `minority_reports_finalised_log` (MRFL) | Authoritative finalised reports combining model and human input. |

Key fields:  
`report_id`, `store_id`, `window_start`, `proposed_cause(_category)`, `final_cause(_category)`,  
`confidence_score`, `final_confidence`, `annotation_id`, `finalized_at`, `report_status`, `run_id`.

---

## 4. Core Logic  
- **Identity:** derive and normalise `report_id` from edit logs (regex clean).  
- **Field precedence:**  
  1. `user_edits_log` overrides individual report fields.  
  2. `user_minority_events_edits_log` overrides at the event level for all associated reports.  
  3. Fallback to latest MRPAL where no edit exists.  
- **Metadata:** carry contextual fields (`store_id`, `sku`, timings) from edit logs.  
- **Status:** all rows tagged `report_status='finalised'`.  
- **Null handling:** `final_decision` reserved (null in demo builds).  

---

## 5. Operational Flow  
1. Merge MRPAL (proposals) with both `user_edits_log` (report-level) and `user_minority_events_edits_log` (event-level).  
2. Cascade precedence to ensure consistent finalisation across linked reports.  
3. Clean and align identifiers.  
4. Write MRFL rows with final fields and timestamps.  
5. Hydration layer surfaces MRFL as the authoritative view.  
6. Entire process is deterministic and replayable from logs — no mutable state.

---

## 6. Example Narrative  
Event `ME-500C24877A1` contains three reports.  
- Proposed cause (MRPAL): *“TV Campaign Launch”*, confidence 0.81.  
- Event-level edit (via `user_minority_events_edits_log`): update cause to *“Retailer Promotion”*.  
- Report-level edit for one SKU (via `user_edits_log`): final_cause = *“Competitor Stockout”*, annotation_id = `act-5566`.  

**Finalised outputs:**  
- Two reports inherit *“Retailer Promotion”* (event-level override).  
- One report uses *“Competitor Stockout”* (individual override).  
- All appear in MRFL as finalised with audit trace back to respective edit log.

---

## 7. Schema Summary  
| Log | Key Fields | Notes |
|------|-------------|-------|
| MRFL | `report_id`, `proposed_cause(_category)`, `final_cause(_category)`, `confidence_score`, `final_confidence`, `annotation_id`, `finalized_at`, `report_status`, `run_id` | Combines model outputs and analyst edits at both report and event level. |

---

### Output Schema — `minority_reports_finalised_log (MRFL)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic identifier for each anomaly (`hash(store_id || first_detected_from)`). |
| `report_group_id` | string | Cohort or event-level identifier derived from MRCH. |
| `event_status` | string | Indicates whether the related event is `active` or `ended`. |
| `version` | integer | Monotonic counter for append-only history. |
| `store_id` | string | Retailer or store identifier. |
| `sku` | string | Product identifier linked to anomaly. |
| `window_start` | timestamp | Start time of the anomaly window. |
| `window_end` | timestamp | End time of the anomaly window. |
| `severity_score` | double | Intensity of the anomaly derived from detection. |
| `impact_estimate` | decimal | Estimated commercial impact (uplift or loss). |
| `run_id` | string | Execution key linking record to demo context. |
| `attributed_sales_total` | integer | Actual total sales during anomaly period. |
| `baseline_sales_total` | integer | Expected baseline sales for comparison. |
| `feature_vector` | array | Feature vector describing anomaly shape. |
| `cluster_id` | string | Cluster identifier inherited from MRCL. |
| `cluster_name` | string | Human-readable label for cluster type (e.g., *TikTok Viral*). |
| `similarity_score` | double | Similarity measure to cluster centroid. |
| `cluster_match_confidence` | double | Confidence of assigned cluster. |
| `tsne_x` | double | 2D projection coordinate (x) for cluster visualization. |
| `tsne_y` | double | 2D projection coordinate (y) for cluster visualization. |
| `proposed_cause` | string | System-proposed primary cause. |
| `proposed_cause_category` | string | Category of system-proposed cause. |
| `confidence_score` | double | Confidence in system-proposed cause. |
| `supporting_evidence` | string | Optional evidence or reference notes. |
| `final_decision` | string | Human or system final decision outcome. |
| `final_cause` | string | Finalized cause after HITL review. |
| `final_cause_category` | string | Category of finalized cause (aligned with taxonomy). |
| `final_confidence` | double | Final confidence level after review or auto-approval. |
| `finalized_at` | timestamp | Timestamp when report was finalized (human or system). |
| `annotation_id` | string | Unique identifier for analyst annotation in UI edits log. |
| `report_status` | string | Lifecycle label (`finalised`, `revised`, `reopened`, etc.). |
| `refresh_seq` | integer | Sequence number for hydration refresh logic. |
| `is_degraded` | boolean | Indicates whether upstream data was missing or incomplete. |
| `degraded_reasons` | string | Encoded list of missing upstream sources (e.g., `MRCL_down`, `MRPAL_down`). |
| `source_used_group_id` | string | Provenance column — identifies which log provided the group ID. |
| `source_used_proposed` | string | Provenance column — identifies which log provided the proposed cause. |
| `source_used_structure` | string | Provenance column — identifies which log provided the event structure. |
| `source_used_window_start` | string | Provenance column — identifies source of window start time. |
| `source_used_window_end` | string | Provenance column — identifies source of window end time. |

---

**Properties**
- Authoritative log for finalized report state (machine or human).  
- Combines detection, clustering, attribution, **and both report-level and event-level human annotations**.  
- Guarantees full auditability via provenance columns.  
- Serves as the canonical “truth layer” for downstream hydration and reporting.

---

**Summary:**  
Finalisation unites machine and human judgment — across both individual reports and grouped events — into a single deterministic log, ensuring auditability, replayability, and a clear source of truth for downstream reporting.
