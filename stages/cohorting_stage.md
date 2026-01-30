# Cohorting Stage — Minority Report System  
**File:** `cohorting_stage.md`  

---

## 1. Purpose  
The cohorting stage groups related anomalies (**Minority Reports**) into higher-order **Minority Events** — sets of anomalies that are likely driven by the same underlying cause.  
Implemented in `build_minority_report_cohorts.py`, it analyses temporal overlap, store proximity, and cluster similarity to infer shared causality.  
The transform deterministically assigns each anomaly to an event group (`report_group_id`), updates aggregate metrics over time, and writes outputs to the **Minority Reports Cohorted Log (MRCH)** and **Minority Event Log (MELOG)**.

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL):** anomaly IDs, timing, and totals.  
- **`anomaly_classifier_model_prediction_log`** and **`baseline_sales_model_prediction_log`:** supply per-tick metrics.  
- *(Optional)* `contextual_data`, `all_campaign_data` for enrichment.  
- **`demo_run_config`:** provides `run_id`.  

---

## 3. Outputs  
| Dataset | Purpose |
|----------|----------|
| `minority_reports_cohorted_log` (MRCH) | Per-tick snapshots of evolving events. |
| `minority_events` (MELOG) | One stable row per event (registry of active and ended events). |

Both logs are **append-only** and keyed by deterministic identifiers.  
Each anomaly retains its `report_id = hash(store_id || first_detected_from)`.  
Cohorts are assigned a `report_group_id`, computed as a hash of the `report_id` and the `first_detected_from` of the earliest anomaly within that cohort.  
This guarantees reproducible groupings and stable lineage across replays.

---

## 4. Core Logic  
1. **Causal grouping:** group anomalies (`minority_reports`) that appear to share a common underlying cause,  
   using temporal overlap and similarity in their **cluster** and **attribution** outputs.  
   The transform does not infer new causality — it consolidates existing model signals to identify shared events.  
2. **Event identity:** assign a deterministic  
   `report_group_id = hash(report_id || earliest_first_detected_at_in_group)`.  
   Each anomaly retains its own `report_id` but inherits the same group ID as others in the inferred event.  
3. **Metric aggregation:** compute event-level metrics (e.g., cumulative severity, uplift, duration, report count).  
4. **Closure:** when all reports within a group have ended, mark the event as `ended` and append final MRCH and MELOG rows.  
5. **Stateless replay:** each run recomputes cohort membership deterministically from the input logs;  
   no intermediate state or persistence is required.

---

## 5. Operational Flow  
1. **Inputs:**  
   - `minority_reports_clustered_log` (MRCL)  
   - `minority_reports_proposed_attribution_log` (MRPAL)  
   - `minority_reports_detected_log` (MRDL)  
   - `demo_run_config`  

2. **Group:** combine anomalies that share time overlap and similar cluster or attribution results.  
3. **Assign:** generate deterministic `report_group_id`s based on earliest detection within each cohort.  
4. **Write:** append cohort snapshots to `minority_reports_cohorted_log` (MRCH).  
5. **Register:** create or update one authoritative row per event in `minority_events` (MELOG).  
6. **Post-cohort edits:** analysts may later update event-level data via `user_minority_events_edits_log`,  
   maintaining a complete audit trail.

---

## 6. Example Narrative  
Store 327 shows an anomaly from 11:55–12:05 linked to cluster *TikTok Viral* and proposed cause *Influencer Mention*.  
Store 412 shows a similar spike from 11:57–12:06 with the same cluster and cause.  
Cohorting identifies overlap in timing and attribution and groups both reports into a single Minority Event:

- `report_group_id = hash('ME-e87a413ed08d' || 11:55)`  
- `window_start=11:55`, `window_end=12:06`, `severity=0.72`, `attributed_sales_total=8,200`.  
- MRCH records snapshots for each tick; MELOG creates one event record at 11:55.  
- When both anomalies close, the event is marked `ended` and final MRCH + MELOG rows are appended.  
- Subsequent analyst updates (e.g., tagging as “confirmed viral effect”) are logged in `user_minority_events_edits_log`.
- 
---

## 7. Schema Summary  
| Log | Key Fields | Notes |
|------|-------------|-------|
| MRCH | `report_id`, `store_id`, `window_start`, `window_end`, `event_status`, `written_at` | Per-tick snapshots. |
| MELOG | `event_id`, `store_id`, `window_start`, `window_end`, `event_status`, `created_at`, `finalized_at` | Event registry. |

---

### Output Schema — `minority_reports_cohort_log (MRCH)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic identifier for each anomaly (`hash(store_id || first_detected_from)`). |
| `report_group_id` | string | Deterministic event or cohort identifier (`hash(store_id || earliest_first_detected_at_in_cohort)`). |
| `event_status` | string | Indicates whether the cohort is `active` or `ended`. |
| `report_status` | string | Lifecycle status of individual report within the cohort. |
| `version` | integer | Monotonic counter for append-only versioning. |
| `store_id` | string | Retailer or store identifier. |
| `sku` | string | Product identifier associated with the anomaly. |
| `window_start` | timestamp | Beginning of anomaly window. |
| `window_end` | timestamp | End of anomaly window (nullable if still active). |
| `severity_score` | double | Intensity of anomaly derived from detection stage. |
| `impact_estimate` | decimal | Estimated commercial impact for this cohorted report. |
| `run_id` | string | Execution key linking this run to demo context. |
| `attributed_sales_total` | decimal | Actual total sales within the anomaly window. |
| `baseline_sales_total` | decimal | Expected baseline sales for the same period. |
| `feature_vector` | array | Encoded representation of anomaly shape (carried forward from clustering). |
| `cluster_id` | string | Assigned cluster ID inherited from clustering stage. |
| `cluster_name` | string | Label for the identified cluster type (e.g., *TikTok Viral*). |
| `similarity_score` | double | Distance or similarity measure to the cluster centroid. |
| `cluster_match_confidence` | double | Model confidence in cluster assignment. |
| `tsne_x` | double | 2D x-coordinate for UMAP/t-SNE visualisation. |
| `tsne_y` | double | 2D y-coordinate for UMAP/t-SNE visualisation. |
| `proposed_cause` | string | Primary proposed cause for the cohort. |
| `proposed_cause_category` | string | Category of the primary cause (e.g., *Marketing*, *Operational*). |
| `confidence_score` | double | Confidence level in the proposed cause. |
| `supporting_evidence` | string | Optional notes or references supporting the attribution. |
| `cohort_id` | string | Secondary key for the event or cohort group. |
| `cohort_name` | string | Human-readable label for the cohort/event. |
| `cohort_confidence` | double | Confidence in the cohort-level cause attribution. |
| `cohort_reason` | string | Rationale for grouping (e.g., *temporal adjacency*, *shared cause*). |
| `cohort_size` | integer | Number of reports grouped into this cohort. |

**Properties**
- Append-only and fully replayable.  
- Bridges report-level and event-level perspectives (`report_id` ↔ `report_group_id`).  
- Schema unifies anomaly, clustering, and attribution data into a cohesive event view.  
- Downstream logs (MELOG, MRFL) derive from this authoritative cohort layer.

---

**Summary:**  
Cohorting unifies related anomalies into store-level events using deterministic, replayable logic.  
It behaves like a stateful tracker while remaining stateless and reproducible—preserving full auditability and continuity for downstream stages.
