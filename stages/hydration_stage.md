# Finalisation Stage — Minority Report System  
**File:** `finalisation_stage.md`  

---

## 1. Purpose  
The finalisation stage merges model outputs and analyst edits into the authoritative **Minority Reports Finalised Log (MRFL)**.  
Implemented in `build_minority_reports_finalised_log_from_edits.py`, it combines the **proposed attribution log (MRPAL)** with the **UI edits log**, producing deterministic, replayable records of finalised outcomes.

---

## 2. Inputs  
- **`minority_reports_proposed_attribution_log` (MRPAL):** proposed causes and confidence scores.  
- **`ui_edits_log`:** human-in-the-loop overrides and annotations.  
- **`demo_run_config`:** provides `run_id`.  

---

## 3. Output  
| Dataset | Purpose |
|----------|----------|
| `minority_reports_finalised_log` (MRFL) | Authoritative, append-only record of finalised reports combining model and human inputs. |

Key fields:  
`report_id`, `store_id`, `sku`, `proposed_cause(_category)`, `final_cause(_category)`, `confidence_score`, `final_confidence`, `annotation_id`, `finalized_at`, `report_status`, `run_id`.

---

## 4. Core Logic  
- **Identity:** derive and normalise `report_id` from UI edits (`minority_reports_new_report_id_13`), using regex `[A-Za-z0-9:_-]`.  
- **Field precedence:**  
  1. If user edit present → take `final_cause`, `final_cause_category`, `final_confidence`.  
  2. Otherwise → fallback to MRPAL fields.  
- **Metadata:** carry `store_id`, `sku`, `window_start`, `window_end` from edit rows.  
- **Status:** tag all rows `report_status='finalised'`; `final_decision` remains null in demo builds.  

---

## 5. Operational Flow  
1. Join MRPAL (model proposals) with `ui_edits_log` (analyst overrides).  
2. Clean identifiers and normalise report IDs.  
3. Write MRFL rows with merged and authoritative values.  
4. Hydrate surfaces MRFL as the source of truth for finalised causes.  
5. Entire process is stateless and replayable from logs.

---

## 6. Example Narrative  
Report `abc123` (Store 327):  
- Proposed cause: *“TikTok Challenge”*, confidence 0.78.  
- Analyst edit (16:05): `final_cause="Competitor Stockout"`, `annotation_id=act-5566`.  
- Output MRFL row:  
  - `proposed_cause=TikTok Challenge`, `confidence_score=0.78`  
  - `final_cause=Competitor Stockout`, `final_confidence=NULL`  
  - `finalized_at=2025-08-20 16:05`  
  - `report_status='finalised'`.

---

## 7. Schema Summary  
| Log | Key Fields | Notes |
|------|-------------|-------|
| MRFL | `report_id`, `proposed_cause(_category)`, `final_cause(_category)`, `confidence_score`, `final_confidence`, `annotation_id`, `finalized_at`, `report_status`, `run_id` | Combines model and analyst inputs into authoritative state. |

---

**Summary:**  
Finalisation unites machine and human judgment into a deterministic, append-only log that serves as the definitive source for downstream reporting and audit.
