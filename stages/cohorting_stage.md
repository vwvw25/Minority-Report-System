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

**Summary:**  
Cohorting unifies related anomalies into store-level events using deterministic, replayable logic.  
It behaves like a stateful tracker while remaining stateless and reproducible—preserving full auditability and continuity for downstream stages.
