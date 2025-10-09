# Detection Stage — Minority Report System  
**File:** `detection_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The detection stage is the **entry point** of the Minority Report System.  
It runs:  
1. A **Baseline Sales Model** — predicts expected sales.  
2. An **Anomaly Classifier** — compares actual vs. baseline to detect unusual behaviour.  
3. A **Detection Transform (`detect_anomalies.py`)** — orchestrates the models, generates logs, attaches stable IDs, computes sales totals, and formalises **minority reports**.  

All downstream clustering/attribution relies on these logs.  

---

## 2. Inputs  
- **`sales_timeseries_data`**  
  - Actual observed sales, store_id × sku × 5-minute buckets.  

- **`demo_run_config`**  
  - Provides `run_id`.  

---

## 3. Outputs  

### 3.1 `baseline_sales_model_prediction_log`  
- Expected sales trajectory per store/sku/timestamp.  
- Generated inside `detect_anomalies.py`.  
- Used for both anomaly comparison and sales totals.  

### 3.2 `anomaly_classifier_model_prediction_log`  
- One row per tick if anomaly suspected.  
- Fields:  
  - `window_start` = estimated onset (may backfill earlier than detection).  
  - `window_end` = null until anomaly ends.  
  - `first_detected_from` = tick when anomaly first noticed.  
  - `severity_score`, `impact_estimate`.  
  - `written_at` (tick timestamp).  

### 3.3 `minority_reports_detected_log` (MRDL)  
- One row per tick per minority report.  
- Fields include:  
  - `report_id` = hash(store_id || first_reported_at).  
  - `store_id`, `sku`.  
  - `first_reported_at` = first anomaly tick in X-hour window (stable anchor).  
  - `first_detected_from` = classifier’s detection timestamp (may shift).  
  - `window_start`, `window_end`.  
  - `severity_score`, `impact_estimate`.  
  - `baseline_sales_total` = ∑ baseline predictions between window_start → written_at.  
  - `attributed_sales_total` = ∑ actual sales between window_start → written_at.  
  - `report_status='detected'`.  
  - `written_at`, `run_id`.  

### 3.4 `minority_event_ended_log` (MEEL)  
- Written when classifier stops reporting an anomaly.  
- Fields: `report_id`, `store_id`, `sku`, final `window_end`.  
- `event_status='ended'`.  

---

## 4. Logic  

### 4.1 Baseline Sales Model (inside detection transform)  
- Runs against `sales_timeseries_data`.  
- Produces expected values per store/sku/5-minute bucket.  
- Appends results to `baseline_sales_model_prediction_log`.  

### 4.2 Anomaly Classifier (inside detection transform)  
- Compares actual vs. baseline for the same slice.  
- If anomaly detected:  
  - Emits row to `anomaly_classifier_model_prediction_log`.  
  - Calculates `window_start` (best guess of true onset, may precede detection).  
  - Records `first_detected_from` (when anomaly first noticed).  
  - Leaves `window_end` null until closure.  

### 4.3 Minority Report Generation  
- Detection transform unifies sales, baseline, and classifier outputs into **minority reports**.  
- Rules:  
  - **Stable ID**:  
    - If anomaly at store/sku within X hours → carry earliest tick’s `written_at` forward as `first_reported_at`.  
    - Else set `first_reported_at = current written_at`.  
    - `report_id = hash(store_id || first_reported_at)`.  
  - **Totals**:  
    - Actuals = ∑ sales from `sales_timeseries_data` between window_start → written_at.  
    - Baseline = ∑ predictions over same slice.  
  - Write MRDL row.  
- When anomaly ceases, MEEL row is written.  

---

## 5. Timing Fields  

- **`first_reported_at`** → the first time the system raised an anomaly at this store/sku. Stable; anchors report_id.  
- **`first_detected_from`** → when the anomaly was first detectable by classifier. May backfill.  
- **`window_start`** → classifier’s best estimate of true onset.  
- **`window_end`** → recorded in MEEL when anomaly ends.  

---

## 6. Flow Summary  

1. **Actuals** → from `sales_timeseries_data`.  
2. **Baseline model** runs → writes to `baseline_sales_model_prediction_log`.  
3. **Anomaly classifier** runs → writes to `anomaly_classifier_model_prediction_log`.  
4. **Detection transform** packages all signals into minority reports →  
   - Writes `minority_reports_detected_log`.  
   - Writes `minority_event_ended_log` when anomaly ends.  
5. **Downstream** → clustering, attribution, HITL.  

---

## 7. Example Narrative  

- 15:30: anomaly begins at Store 327.  
- 16:00: detection transform runs:  
  - Baseline model predicts normal sales.  
  - Anomaly classifier detects deviation.  
  - Logs:  
    - baseline prediction row  
    - anomaly classifier row (`window_start=15:30`, `first_detected_from=16:00`, `written_at=16:00`)  
    - MRDL row with `first_reported_at=16:00`, totals for [15:30 → 16:00].  
- 17:00: another tick. Same `report_id`. Totals updated [15:30 → 17:00].  
- 21:00: anomaly no longer detected. Detection transform writes MEEL row with `window_end=21:00`.  

---

## 8. Schemas  

### 8.1 `baseline_sales_model_prediction_log`  
- `store_id`  
- `sku`  
- `timestamp`  
- `expected_units`  
- `written_at`  
- `run_id`  

### 8.2 `anomaly_classifier_model_prediction_log`  
- `store_id`  
- `sku`  
- `window_start`  
- `window_end`  
- `first_detected_from`  
- `severity_score`  
- `impact_estimate`  
- `written_at`  
- `run_id`  

### 8.3 `minority_reports_detected_log` (MRDL)  
- `report_id`  
- `store_id`  
- `sku`  
- `first_reported_at`  
- `first_detected_from`  
- `window_start`  
- `window_end`  
- `severity_score`  
- `impact_estimate`  
- `baseline_sales_total`  
- `attributed_sales_total`  
- `report_status`  
- `written_at`  
- `run_id`  

### 8.4 `minority_event_ended_log` (MEEL)  
- `report_id`  
- `store_id`  
- `sku`  
- `window_end`  
- `event_status`  
- `written_at`  
- `run_id`  
