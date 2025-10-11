# Cohorting Stage — Minority Report System  
**File:** `cohorting_stage.md`  

---

## 1. Purpose  
The cohorting stage groups short-term anomalies (**minority reports**) into cohesive **events** for each store.  
Implemented in `cohort_reports.py`, it deterministically identifies continuous anomaly windows, updates metrics over time, and writes outputs to the **Minority Reports Cohorted Log (MRCH)** and **Minority Event Log (MELOG)**.

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
| `minority_event_log` (MELOG) | One stable row per event (registry of active and ended events). |

Both logs are **append-only** and keyed by deterministic IDs (`report_id = hash(store_id || first_detected_from)`).

Additionally, each cohort is assigned a stable group identifier:  
`report_group_id = hash(store_id || earliest_first_detected_at_in_cohort)`  
which guarantees deterministic grouping and clean lineage for every cohort.

---

## 4. Core Logic  
1. **Event grouping:** for each `store_id`, combine temporally adjacent reports within a gap threshold (e.g., 30 min).  
2. **Event identity:** `first_detected_from` anchors the group; `report_id` derived deterministically.  
3. **Metric updates:** refresh severity, impact, and sales totals each tick.  
4. **Closure:** if no new reports within threshold → mark event ended; write final MRCH and MELOG rows.  
5. **Stateless replay:** each run recomputes events from logs; no in-memory state required.

---

## 5. Operational Flow  
1. MRDL provides anomaly rows.  
2. Cohorting groups them by store/time gap.  
3. MRCH appends snapshots (`event_status='active'` or `'ended'`).  
4. MELOG maintains the single authoritative event registry.  
5. Downstream hydration merges these with detection, clustering, and attribution logs.
6. Event-level metadata may be updated post-cohorting via user_minority_events_edits_log, maintaining full audit trace.

---

## 6. Example Narrative  
Store 327 emits anomalies at 11:55, 12:00, 12:05 → grouped as one event:  
- `report_id = hash('327' || 11:55)`  
- `window_start=11:55`, `window_end=12:05`, `severity=0.72`, `attributed_sales_total=8,200`.  
- MRCH rows written per tick; MELOG created once at 11:55.  
- At 13:05, event ends → final MRCH + MELOG rows with `event_status='ended'`.

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
