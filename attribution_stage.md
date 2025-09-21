# Attribution Stage — Minority Report System  
**File:** `attribution_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The attribution stage proposes **causes** for each minority report.  
It is implemented as a **transform** — `propose_cause_for_minority.py` — which runs the **Attribution Model**.  

This stage:  
1. Consumes minority reports and supporting sales slices.  
2. Joins in contextual and campaign datasets.  
3. Applies attribution model logic.  
4. Writes authoritative results to the **minority reports proposed attribution log (MRPAL)**.  

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL)**  
  - Supplies report_id, store_id, sku, anomaly timing, and sales totals.  

- **`sales_timeseries_data`**  
  - Provides backing time series for further checks.  

- **`contextual_data`**  
  - Store- and environment-level context (e.g., weather, holiday flags, competitor events).  

- **`all_campaign_data`**  
  - Marketing and promotional campaign metadata (timings, SKUs, geographies).  

- **`demo_run_config`**  
  - Provides run_id.  

---

## 3. Attribution Model  

### 3.1 Purpose  
- Infer the most likely cause(s) of each minority report by matching report timing and SKU to contextual and campaign data.  

### 3.2 Logic  
- **Primary cause selection:**  
  - If campaign in `all_campaign_data` overlaps with the anomaly (store, SKU, time) → assign campaign cause, confidence scaled by overlap strength.  
  - Else if contextual_data explains anomaly (holiday uplift, stockout, competitor disruption) → assign cause accordingly.  
  - Else → fallback (demo build: confidence=0.7; current build: confidence=NULL).  

- **Secondary/tertiary candidates:**  
  - Additional overlapping signals (e.g., campaign + holiday) are passed through as `*_2`, `*_3`.  

- **Sales totals:**  
  - Attribution stage carries forward `baseline_sales_total` and `attributed_sales_total` from MRDL.  

---

## 4. Outputs  

### 4.1 `minority_reports_proposed_attribution_log` (MRPAL)  
The authoritative attribution model log.  
- One row per minority report per tick.  
- Fields include:  
  - `report_id`, `store_id`, `sku`  
  - `window_start`, `window_end`  
  - `severity_score`, `impact_estimate`  
  - `proposed_cause`, `proposed_cause_category`, `confidence_score`  
  - `proposed_cause_2`, `proposed_cause_category_2`, `confidence_score_2`  
  - `proposed_cause_3`, `proposed_cause_category_3`, `confidence_score_3`  
  - `supporting_evidence` (campaign ID, contextual flag, etc.)  
  - `baseline_sales_total`, `attributed_sales_total`  
  - `report_status='proposed'`  
  - `written_at`, `run_id`  

---

## 5. Timing Fields  
- **`window_start`** → carried from MRDL.  
- **`window_end`** → filled when MEEL marks anomaly ended.  
- **`written_at`** → tick timestamp of attribution.  

---

## 6. Flow Summary  
1. **MRDL** provides minority reports with timings and totals.  
2. **Sales_timeseries_data** backs further checks if needed.  
3. **Contextual_data** and **all_campaign_data** joined in.  
4. **Attribution model** proposes cause(s) and confidence score(s).  
5. **Transform writes MRPAL row**.  
6. **Downstream** cohorting and HITL consume MRPAL.  

---

## 7. Example Narrative  
- MRDL: Store 327 anomaly, window_start=15:30, written_at=17:00.  
- all_campaign_data: “2-for-1 Chocolate Promo” running at Store 327, SKU match.  
- contextual_data: also bank holiday flagged for that date.  
- Attribution model outputs:  
  - Primary cause = “Campaign: 2-for-1 Chocolate Promo”, confidence=0.87.  
  - Secondary cause = “Bank Holiday uplift”, confidence=0.55.  
- MRPAL row written with both causes, carried sales totals, and supporting_evidence fields.  

---

## 8. Schemas  

### 8.1 `minority_reports_proposed_attribution_log` (MRPAL)  
- `report_id`  
- `store_id`  
- `sku`  
- `window_start`  
- `window_end`  
- `severity_score`  
- `impact_estimate`  
- `proposed_cause`  
- `proposed_cause_category`  
- `confidence_score`  
- `proposed_cause_2`  
- `proposed_cause_category_2`  
- `confidence_score_2`  
- `proposed_cause_3`  
- `proposed_cause_category_3`  
- `confidence_score_3`  
- `supporting_evidence`  
- `baseline_sales_total`  
- `attributed_sales_total`  
- `report_status`  
- `written_at`  
- `run_id`  
