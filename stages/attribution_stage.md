# Attribution Stage — Minority Report System  
**File:** `attribution_stage.md`  

---

## 1. Purpose  
The attribution stage proposes **causes** for each minority report via the transform `propose_cause_for_minority.py`,  
which applies the **attribution model** (mocked in the MVP) to contextual and campaign data to infer the most plausible explanation for each anomaly.  
It writes authoritative outputs to the **Minority Reports Proposed Attribution Log (MRPAL)**.

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL):** anomaly IDs, store/SKU, timings, totals.  
- **`sales_timeseries_data`:** backing time series for checks.  
- **`contextual_data`:** weather, holidays, competitor events.  
- **`all_campaign_data`:** marketing and promo metadata.  
- **`demo_run_config`:** provides run_id.  

---

## 3. Model Logic  
1. **Primary cause:** inferred by the attribution model through probabilistic evaluation of campaign overlaps, contextual signals, and cluster proximity, weighted by historical priors.
2. **Secondary/tertiary:** other overlapping signals, stored in `_2`, `_3` fields.  
3. **Fallback:** if no viable candidate cause is identified, assign `proposed_cause='unknown'` with a nominal `confidence_score=0.05` rather than emitting NULLs.

**Example**  
Store 327 anomaly during “2-for-1 Chocolate Promo” + bank holiday →  
- Primary: proposed_cause_category: TPO, proposed_cause: 2-for-1 Chocolate Promo, confidence_score 0.87  
- Secondary: proposed_cause_category: Social Media Viral, proposed_cause: TikTok, confidence 0.55  

### Model Behaviour (Production)

In the mock model, the attribution model is mocked via deterministic lookups to `master_narratives`.  
In production, this stage would invoke a **model inference transform** — a component that applies a pre-trained attribution model to new anomaly data.  
The model would evaluate temporal, geographic, and contextual correlations between sales anomalies and candidate causes, returning a ranked list of plausible attributions with confidence scores.

Model training would occur separately, outside the operational pipeline, using historical anomalies and campaign data.  
This separation ensures the pipeline remains stateless and deterministic while still benefiting from periodically retrained models.

### Inputs (Feature Families)
- Temporal alignment between campaign or event and anomaly window.  
- Lead/lag correlations between campaign activity and sales patterns.  
- Geographic overlap between campaign coverage and affected stores.  
- Cluster similarity to historically known patterns.  
- External signals such as weather, competitor price/stock changes, and social mentions.  
- Structural priors from campaign metadata (spend, format, channel, historical uplift).

### Outputs
The model emits a ranked list of candidate causes with associated probabilities or confidence scores.  
Example:

{
  "report_id": "MR-982d45",
  "proposed_cause": "2-for-1 Chocolate Promo",
  "proposed_cause_category": "Marketing",
  "confidence_score": 0.87,
  "proposed_cause_2": "Hot Weather",
  "confidence_score_2": 0.42,
  "proposed_cause_3": "Competitor Stockout",
  "confidence_score_3": 0.33
}

---

## 4. Transform Logic  
Most fields in the output are *carried through* from upstream logs to maintain schema continuity and enable deterministic joins.  
Only the cause and confidence columns are model-generated.

**Steps**
1. Join MRDL (detection), MRCL (clustering), and contextual datasets.  
2. Pass relevant features to the attribution model.  
3. Carry through upstream fields (`report_id`, `window_start`, `baseline_sales_total`, etc.) unchanged.  
4. Append model outputs (`proposed_cause`, `confidence_score`, etc.).  
5. Write to `minority_reports_proposed_attribution_log` (MRPAL) with `report_status='proposed'`.  

---

## 5. Output: `minority_reports_proposed_attribution_log` (MRPAL)
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

## 6. Governance & Validation  
- **Field precedence:** Analyst edits > Attribution > Cluster metadata.  
- **Confidence tiers:** Process of working with client to establish threshold tiers for alerts + autoconfirm (when/where it becomes viable). 
- **Synthetic tests:** inject known causes (e.g., TikTok spike) to verify attribution.  
- **Audit:** All rows append-only with deterministic IDs and timestamps for replayability.  

---

## 7. Resilience & Monitoring  
- Writes only to MRPAL (append-only, idempotent).  
- If inputs fail → emit `proposed_cause_category='unknown'`, `confidence=NULL`.  
- **SLIs:** freshness < 4 h (p95); ≥80% coverage within SLA; <1% rows to `_errors`.  

---

### Output Schema — `minority_reports_proposed_attribution_log (MRPAL)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic anomaly identifier (`hash(store_id || first_detected_from)`). |
| `version` | integer | Monotonic counter for row versioning (append-only history). |
| `store_id` | string | Retailer or store identifier. |
| `sku` | string | Product identifier linked to anomaly. |
| `window_start` | timestamp | Start time of anomaly window. |
| `window_end` | timestamp | End time of anomaly window (nullable if active). |
| `severity_score` | double | Intensity of the anomaly derived from detection stage. |
| `impact_estimate` | decimal | Estimated commercial impact (uplift or loss). |
| `proposed_cause` | string | Primary proposed cause of the anomaly. |
| `proposed_cause_category` | string | Category classification of the primary cause (e.g., *Marketing*, *Operational*, *Data Error*). |
| `confidence_score` | double | Model-derived confidence in the primary cause. |
| `proposed_cause_2` | string | Secondary proposed cause (alternative hypothesis). |
| `proposed_cause_category_2` | string | Category of the secondary cause. |
| `confidence_score_2` | double | Confidence in secondary cause. |
| `proposed_cause_3` | string | Tertiary proposed cause (fallback hypothesis). |
| `proposed_cause_category_3` | string | Category of the tertiary cause. |
| `confidence_score_3` | double | Confidence in tertiary cause. |
| `supporting_evidence` | string | Optional notes or reference to supporting data (e.g., campaign ID, social trigger). |
| `report_status` | string | Lifecycle label (`proposed`, `finalised`, etc.). |
| `written_at` | timestamp | Time the record was appended. |
| `run_id` | string | Execution key linking this record to demo context. |
| `attributed_sales_total` | integer | Actual sales observed during the anomaly window. |
| `baseline_sales_total` | integer | Baseline expected sales for comparison. |

**Properties**
- Append-only log capturing machine-proposed attributions.  
- Deterministic schema matching MRDL and MRCL for reliable joins.  
- Serves as authoritative record for all model-based attribution proposals.  
- Enables side-by-side comparison of confidence tiers and model evolution.

---

**Summary:**  
Attribution enriches minority reports with plausible causes and confidence scores, ensuring downstream systems remain interpretable even under partial data.  
The transform is deterministic, replayable, and resilient—demonstrating end-to-end system design maturity without relying on live ML training.
