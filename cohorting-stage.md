# Cohorting Stage — Minority Report System  
**File:** `cohorting_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The cohorting stage groups individual short-term anomalies (**minority reports**) into cohesive **events** at the same store.  
It is implemented as a **transform** — `cohort_reports.py` — which:  
1. Groups temporally adjacent reports for a store_id.  
2. Retains the `first_detected_from` of the earliest constituent report.  
3. Updates severity, impact, and totals as new reports arrive.  
4. Writes results into both a **temporal log** (`minority_reports_cohorted_log`) and an **event registry** (`minority_event_log`).  

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL)**  
  - Provides report_id, store_id, sku, anomaly timing, sales totals.  

- **`anomaly_classifier_model_prediction_log`**  
  - Supplies per-tick classifier snapshots.  

- **`baseline_sales_model_prediction_log`**  
  - Supplies baseline totals.  

- **Optional:**  
  - **`contextual_data`**  
  - **`all_campaign_data`**  
  - Used for enriching metadata, not for grouping.  

- **`demo_run_config`**  
  - Provides run_id.  

---

## 3. Outputs  

### 3.1 `minority_reports_cohorted_log` (MRCH)  
- One row per `(store_id, report_id, written_at)`.  
- Represents the evolving state of an event over time.  
- Fields include:  
  - `report_id` (hash(store_id || first_detected_from))  
  - `store_id`  
  - `first_detected_from`  
  - `window_start`, `window_end`  
  - `severity_score`, `impact_estimate`  
  - `baseline_sales_total`, `attributed_sales_total`  
  - `event_status` (active | ended)  
  - `written_at`, `run_id`  

### 3.2 `minority_event_log` (MELOG)  
- One row per **minority event**.  
- Stable registry of events, used as the dataset for the **minority_event object**.  
- Created once per event (at `first_detected_from`), updated with final `window_end` + status on closure.  
- Fields include:  
  - `event_id` (same as report_id)  
  - `store_id`  
  - `first_detected_from`  
  - `window_start`, `window_end`  
  - `severity_score`, `impact_estimate`  
  - `baseline_sales_total`, `attributed_sales_total`  
  - `event_status` (active | ended)  
  - `created_at`, `finalized_at`  
  - `run_id`  

---

## 4. Logic  

### 4.1 Define event identity  
- Group reports by `store_id`.  
- Use time-gap threshold (e.g. 30 minutes) to split events.  
- For each group:  
  - `report_id = hash(store_id || first_detected_from)`  
  - `first_detected_from = earliest anomaly in the group`.  

### 4.2 Aggregate metrics  
- For each active event, update with latest:  
  - `window_end`  
  - `severity_score`  
  - `impact_estimate`  
  - `baseline_sales_total`  
  - `attributed_sales_total`  
- Optionally merge contextual metadata (promo flags, campaign IDs).  

### 4.3 Handle event closure  
- If no new report for store_id within threshold → event is **ended**.  
- On closure:  
  - Finalize `window_end`.  
  - Seal event in both MRCH (last snapshot) and MELOG (final row, `event_status='ended'`).  

---

## 5. Statefulness without State  
Although cohorting behaves like a stateful process, it is implemented by **log replay**:  
- Input = full MRDL (and supporting logs).  
- Transform deterministically recomputes cohorts on each run.  
- Output = consistent MRCH (snapshots) and MELOG (event registry).  
- Fully stateless execution, reproducible from logs alone.  

---

## 6. Flow Summary  
1. **MRDL** provides anomaly reports.  
2. **Cohorting transform** groups them by store_id with a time-gap threshold.  
3. **MRCH** logs evolving event snapshots (per written_at).  
4. **MELOG** registers one row per event for the minority_event object.  
5. **Downstream hydrate** consumes both MRCH and MELOG, alongside MRDL, MRPAL, MRFL.  

---

## 7. Example Narrative  
- Store 327 emits MRDL rows at 11:55, 12:00, 12:05.  
- Cohorting transform groups them (all within threshold).  
- Cohort event created:  
  - `report_id = hash('327' || 11:55)`  
  - `first_detected_from=11:55`, `window_start=11:55`, `window_end=12:05`.  
  - Severity updated to 0.72, attributed_sales_total=8,200.  
- MRCH: writes a row at each tick (`written_at=11:55`, `12:00`, `12:05`).  
- MELOG: writes a single row at `11:55` (event created).  
- No new reports after 12:35. At 13:05, event marked ended.  
  - MRCH row at 13:05 → `event_status='ended'`.  
  - MELOG row updated with final `window_end=12:05`, `finalized_at=13:05`, `event_status='ended'`.  

---

## 8. Schemas  

### 8.1 `minority_reports_cohorted_log` (MRCH)  
- `report_id`  
- `store_id`  
- `first_detected_from`  
- `window_start`  
- `window_end`  
- `severity_score`  
- `impact_estimate`  
- `baseline_sales_total`  
- `attributed_sales_total`  
- `event_status`  
- `written_at`  
- `run_id`  

### 8.2 `minority_event_log` (MELOG)  
- `event_id`  
- `store_id`  
- `first_detected_from`  
- `window_start`  
- `window_end`  
- `severity_score`  
- `impact_estimate`  
- `baseline_sales_total`  
- `attributed_sales_total`  
- `event_status`  
- `created_at`  
- `finalized_at`  
- `run_id`  
