# Detection Stage — Minority Report System  
**File:** `detection_stage.md`  

---

## 1. Purpose  
The detection stage is the **entry point** of the Minority Report System.  
It combines a **Baseline Sales Model** and an **Anomaly Classifier Model**, orchestrated by the transform `detect_anomalies.py`, to identify and formalise sales anomalies as **minority reports**.  
All downstream clustering, attribution, and hydration depend on these logs.

---

## 2. Inputs  
- **`sales_timeseries_data`** — observed sales by store × time.  
- **`demo_run_config`** — provides `run_id`.  

The detection stage assumes pre-integrated, clean sales data is available as `unified_sales_data`.  
This upstream dataset is produced by other platform components responsible for collating, deduplicating, and standardising retailer sales from brick-and-mortar stores.  
The Minority Report System does not perform ETL or data cleaning — it consumes `unified_sales_data` to construct time-series datasets (`sales_timeseries_data`) for anomaly detection.

---

## 3. Outputs  
| Dataset | Purpose |
|----------|----------|
| `baseline_sales_model_prediction_log` | Expected sales per store/SKU/timestamp. |
| `anomaly_classifier_model_prediction_log` | Flags potential anomalies with timing and severity. |
| `minority_reports_detected_log` (MRDL) | Consolidated anomaly view (one row per tick per anomaly). |
| `minority_event_ended_log` (MEEL) | Marks anomaly closure. |

Each log is **append-only**, keyed by stable IDs (`report_id = hash(store_id || first_reported_at)`).

### Output Schema — `minority_reports_detected_log (MRDL)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic identifier for each anomaly (`hash(store_id || first_detected_from)`). |
| `version` | integer | Monotonic counter for row versioning (append-only history). |
| `store_id` | string | Retailer or location identifier. |
| `sku` | string | Product identifier associated with the anomaly. |
| `first_detected_from` | timestamp | Earliest tick where anomaly was detected. |
| `window_start` | timestamp | Beginning of anomaly time window. |
| `window_end` | timestamp | End of anomaly time window (nullable if active). |
| `severity_score` | double | Model-derived measure of anomaly intensity. |
| `impact_estimate` | decimal | Estimated commercial impact (e.g., uplift or loss). |
| `report_status` | string | Lifecycle label (`detected`, `clustered`, `proposed`, etc.). |
| `written_at` | timestamp | Time of record emission (append timestamp). |
| `run_id` | string | Execution key linking this run to demo context. |
| `attributed_sales_total` | integer | Total sales during anomaly window. |
| `baseline_sales_total` | integer | Baseline (expected) sales for same window. |

**Properties**
- Append-only, no updates or deletes.  
- Deterministic identifiers for replayability.  
- Compatible with downstream clustering and hydration joins.  
- Serves as the authoritative source of anomaly detection events.

---

## 4. Logic Overview  
1. **Baseline Model** predicts expected sales.  
2. **Anomaly Classifier** compares actuals vs baseline to detect deviations.  
3. **Detection Transform** merges outputs:  
   - Assigns deterministic `report_id`.  
   - Aggregates totals (actual vs baseline) from anomaly onset to current tick.  
   - Writes MRDL and, when the event closes, MEEL rows.

---

## 5. Operational Flow  
1. Sales data ingested.  
2. Baseline model generates expected trajectory.  
3. Classifier flags deviation.  
4. Detection transform writes MRDL row (`report_status='detected'`).  
5. When anomaly ends, MEEL row finalises `window_end`.  
6. Downstream stages (clustering, attribution, HITL) consume these logs.

---

## 6. Example Narrative  
- **15:30:** anomaly begins at Store 327.  
- **16:00:** detection transform runs —  
  baseline predicts normal sales; classifier flags anomaly.  
  MRDL row written (`first_reported_at=16:00`, totals for [15:30→16:00]).  
- **17:00:** next tick updates totals with same `report_id`.  
- **21:00:** anomaly ends → MEEL row written (`window_end=21:00`).  

---

## 7. Key Schema Summary  
| Log | Key Fields | Notes |
|-----|-------------|-------|
| `baseline_sales_model_prediction_log` | `store_id`, `sku`, `timestamp`, `expected_units` | Baseline reference. |
| `anomaly_classifier_model_prediction_log` | `store_id`, `sku`, `window_start`, `window_end`, `severity_score` | Anomaly signal. |
| `minority_reports_detected_log` | `report_id`, `store_id`, `sku`, `window_start`, `baseline_sales_total`, `attributed_sales_total`, `report_status` | Authoritative anomaly record. |
| `minority_event_ended_log` | `report_id`, `window_end`, `event_status` | Closure marker. |

---

**Summary:**  
Detection transforms raw sales into structured, deterministic anomaly logs.  
It guarantees reproducibility, append-only history, and safe downstream consumption — forming the backbone of the Minority Report System.
