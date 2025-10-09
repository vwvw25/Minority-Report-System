# Attribution Stage — Minority Report System  
**File:** `attribution_stage.md`  

---

## 1. Purpose  
The attribution stage proposes **causes** for each minority report via the transform `propose_cause_for_minority.py`.  
It joins contextual and campaign data to infer the most plausible explanation for each anomaly and writes authoritative outputs to the **Minority Reports Proposed Attribution Log (MRPAL)**.

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL):** anomaly IDs, store/SKU, timings, totals.  
- **`sales_timeseries_data`:** backing time series for checks.  
- **`contextual_data`:** weather, holidays, competitor events.  
- **`all_campaign_data`:** marketing and promo metadata.  
- **`demo_run_config`:** provides run_id.  

---

## 3. Model Logic  
1. **Primary cause:** overlap of campaign or contextual event with anomaly window.  
2. **Secondary/tertiary:** other overlapping signals, stored in `_2`, `_3` fields.  
3. **Fallback:** if no match, emit `proposed_cause=NULL`, `confidence=NULL`.  
4. **Carry-through:** baseline and attributed sales totals from MRDL.  

**Example**  
Store 327 anomaly during “2-for-1 Chocolate Promo” + bank holiday →  
- Primary: *Campaign*, confidence 0.87  
- Secondary: *Bank Holiday Uplift*, confidence 0.55  

---

## 4. Output: `minority_reports_proposed_attribution_log` (MRPAL)
One row per minority report per tick.

| Field | Description |
|-------|--------------|
| `report_id`, `store_id`, `sku` | Identifiers |
| `window_start`, `window_end` | Anomaly window |
| `proposed_cause(_2,_3)` / `proposed_cause_category(_2,_3)` | Ranked causes |
| `confidence_score(_2,_3)` | Confidence values |
| `supporting_evidence` | Campaign ID / context flag |
| `baseline_sales_total`, `attributed_sales_total` | From MRDL |
| `report_status='proposed'`, `written_at`, `run_id` | Metadata |

---

## 5. Governance & Validation  
- **Field precedence:** Analyst edits > Attribution > Cluster metadata.  
- **Confidence tiers:** High > 70% (auto-confirm), Medium 30–70% (review), Low < 30% (open loop).  
- **Synthetic tests:** inject known causes (e.g., TikTok spike) to verify attribution.  
- **Audit:** All rows append-only with deterministic IDs and timestamps for replayability.  

---

## 6. Resilience & Monitoring  
- Writes only to MRPAL (append-only, idempotent).  
- If inputs fail → emit `proposed_cause_category='unknown'`, `confidence=NULL`.  
- **SLIs:** freshness < 4 h (p95); ≥80% coverage within SLA; <1% rows to `_errors`.  

---

**Summary:**  
Attribution enriches minority reports with plausible causes and confidence scores, ensuring downstream systems remain interpretable even under partial data.  
The transform is deterministic, replayable, and resilient—demonstrating end-to-end system design maturity without relying on live ML training.
