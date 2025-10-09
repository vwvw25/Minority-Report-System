# Hydration Stage — Minority Report System  
**File:** `hydration_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The hydration stage materialises the **current state** of every minority report by merging data from all authoritative logs.  
It consolidates detection, clustering, attribution, cohorting, finalisation, and rereview outputs into a **single, wide, analyst-ready object**.

Implemented in the transform `hydrate_minority_reports.py`, hydration is the only stage that reads across logs.  
It never mutates upstream datasets — it simply coalesces them to produce deterministic, queryable views.

---

## 2. Inputs  
| Dataset | Purpose |
|----------|----------|
| `minority_reports_detected_log` (MRDL) | Core anomaly metadata and totals. |
| `minority_reports_clustered_log` (MRCL) | Cluster assignments and similarity metrics. |
| `minority_reports_proposed_attribution_log` (MRPAL) | Model-proposed causes and confidence. |
| `minority_reports_cohorted_log` (MRCH) | Event-level snapshots. |
| `minority_reports_finalised_log` (MRFL) | Analyst-finalised causes. |
| `*_rereview_log` datasets | Optional rereviewed clustering/attribution outputs. |
| `demo_run_config` | Provides `run_id` and filtering context. |

---

## 3. Outputs  
| Dataset | Description |
|----------|--------------|
| `minority_reports` | One row per report_id — latest state across all logs. |
| `minority_events_log` | One row per event_id — aggregated cohort view (mirrors MRCH + MELOG). |

Both are **read-only materialisations**; they can be rebuilt deterministically from logs.

---

## 4. Core Logic  
1. **Union rereview logs** with originals where present (higher precedence).  
2. **Coalesce fields** by fixed precedence:  
   - Finalisation → Attribution → Clustering → Detection.  
3. **Typed null-safety:** missing sources contribute NULLs, never block hydration.  
4. **Version selection:** pick latest `written_at` per `report_id`.  
5. **Status propagation:**  
   - If degradation detected (missing upstream logs), flag `is_degraded=true`.  
   - Compose `report_status` from latest stage (`finalised`, `proposed`, `clustered`, etc.).  

Hydration is designed to be **idempotent and stateless** — a full rebuild produces identical results given the same logs.

---

## 5. Schema Highlights — `minority_reports`  
| Field | Source | Description |
|--------|---------|-------------|
| `report_id`, `store_id`, `sku` | MRDL | Primary identifiers. |
| `window_start`, `window_end` | MRDL / MEEL | Temporal bounds. |
| `cluster_name`, `similarity_score` | MRCL | Latest clustering info. |
| `proposed_cause(_category)`, `confidence_score` | MRPAL / rereview | Model proposals. |
| `final_cause(_category)`, `final_confidence` | MRFL | Analyst-finalised values. |
| `event_status`, `cohort_id` | MRCH | Event context. |
| `is_degraded` | Hydration | True if any upstream source missing. |
| `written_at`, `run_id` | Hydration | Materialisation metadata. |

---

## 6. Operational Flow  
1. Collect all source logs (including rereview).  
2. Align schemas and append typed nulls for any missing columns.  
3. Join by deterministic IDs (`report_id`, `run_id`).  
4. Coalesce fields by precedence and latest timestamp.  
5. Output hydrated datasets (`minority_reports`, `minority_events_log`).  

---

## 7. Example Narrative  
Report `abc123` appears in multiple logs:  
- Detected at 15:30, clustered as *“Social Spike”*, attributed as *“TikTok Challenge”*.  
- Analyst later finalises as *“Competitor Stockout”*.  
Hydration coalesces these into a single row:  
`final_cause="Competitor Stockout"`, `confidence_score=0.78`, `cluster_name="Social Spike"`, `report_status="finalised"`.  

---

## 8. Resilience  
- **Null-tolerant joins** ensure reports still appear even if later logs are missing.  
- **Fixed precedence chain** prevents circular dependencies.  
- **Append-only rebuild** means hydration can always be replayed safely.  

---

**Summary:**  
Hydration is the reconciliation layer of the Minority Report System.  
It deterministically fuses every log into a coherent, analyst-facing view — idempotent, replayable, and enterprise-ready.
