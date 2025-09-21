# Finalisation Stage — Minority Report System  
**File:** `finalisation_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The finalisation stage consolidates model outputs and human edits into the authoritative **minority reports finalised log (MRFL)**.  
It is implemented as a **transform** — `build_minority_reports_finalised_log_from_edits.py`.  

This stage:  
1. Consumes proposed attribution outputs (MRPAL).  
2. Consumes user edits from the UI (`user_edits_log`).  
3. Normalises and cleans identifiers.  
4. Writes authoritative finalised rows into **MRFL**.  

---

## 2. Inputs  
- **`minority_reports_proposed_attribution_log` (MRPAL)**  
  - Provides proposed_cause, confidence_score, and attribution metadata.  

- **`user_edits_log`**  
  - Source of human-in-the-loop edits (proposed cause/category overrides, annotation, timestamps).  

- **`demo_run_config`**  
  - Provides run_id.  

---

## 3. Outputs  

### 3.1 `minority_reports_finalised_log` (MRFL)  
- Authoritative record of finalised reports after HITL edits.  
- One row per report_id, per edit version.  
- Fields include:  
  - `report_id` (cleaned from edit logs, regex `[A-Za-z0-9:_-]`)  
  - `store_id`, `sku`  
  - `window_start`, `window_end`  
  - `severity_score`, `impact_estimate`  
  - `baseline_sales_total`, `attributed_sales_total`  
  - `proposed_cause`, `proposed_cause_category`, `confidence_score` (from MRPAL)  
  - `final_cause`, `final_cause_category`, `final_confidence` (from edits)  
  - `final_decision` (currently null)  
  - `finalized_at` (from `actiontimestamp`)  
  - `annotation_id` (from actionRid or uuid())  
  - `report_status='finalised'`  
  - `written_at`, `run_id`  

---

## 4. Logic  

### 4.1 Report identity  
- `report_id` is not present as a clean field in edit logs.  
- Derived from `minority_reports_new_report_id_13` and normalised via regex.  
- All joins use this cleaned report_id.  

### 4.2 Field precedence  
- If user edit present → final_cause, final_cause_category, final_confidence set from edit.  
- Else → fallback to latest MRPAL values.  
- `finalized_at` strictly from `actiontimestamp`.  
- Other contextual fields (store_id, sku, version, etc.) carried forward from edit row.  

### 4.3 Status handling  
- All finalised rows tagged `report_status='finalised'`.  
- `final_decision` currently left null (reserved for later use).  

---

## 5. Statefulness  
- Deterministic, stateless transform: replays logs on each run.  
- MRFL content always reproducible from:  
  - MRPAL (proposed fields)  
  - user_edits_log (overrides)  
  - config  

---

## 6. Flow Summary  
1. **MRPAL** supplies proposed attribution.  
2. **user_edits_log** supplies user overrides.  
3. **Transform** merges and cleans identifiers.  
4. **MRFL row** written with authoritative final values.  
5. **Downstream hydrate** consumes MRFL to overwrite proposed fields with finalised ones.  

---

## 7. Example Narrative  
- Report abc123 (Store 327) proposed cause = “TikTok Challenge”, confidence=0.78.  
- User edit submitted at `2025-08-20 16:05`:  
  - final_cause = “Competitor Stockout”  
  - annotation_id = `act-5566`  
- Transform produces MRFL row:  
  - `report_id=abc123`  
  - `proposed_cause=TikTok Challenge`, `confidence_score=0.78`  
  - `final_cause=Competitor Stockout`, `final_confidence=NULL`  
  - `finalized_at=2025-08-20 16:05`  
  - `annotation_id=act-5566`  
  - `report_status='finalised'`  

---

## 8. Schemas  

### 8.1 `minority_reports_finalised_log` (MRFL)  
- `report_id`  
- `store_id`  
- `sku`  
- `window_start`  
- `window_end`  
- `severity_score`  
- `impact_estimate`  
- `baseline_sales_total`  
- `attributed_sales_total`  
- `proposed_cause`  
- `proposed_cause_category`  
- `confidence_score`  
- `supporting_evidence`  
- `final_decision`  
- `final_cause`  
- `final_cause_category`  
- `final_confidence`  
- `finalized_at`  
- `annotation_id`  
- `report_status`  
- `written_at`  
- `run_id`  
