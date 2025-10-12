# Hydration Stage — Minority Report System  
**File:** `hydration_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The hydration stage materialises the **latest state** of each Minority Report by combining data from all authoritative logs.  
Implemented in the transform `hydrate_minority_reports.py`, it coalesces the most recent information for every report —  
across detection, clustering, attribution, cohorting, and finalisation — into a single, denormalised dataset.  

This hydrated dataset underpins the **Minority Report ontology object**, providing a continuously up-to-date,  
queryable representation of each report’s state at any point in time.  
The transform never mutates upstream data; it reads append-only logs and deterministically assembles  
the current operational and analytical view of every report and event.

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
5. Output hydrated dataset (`minority_reports`).  

---

## 7. Example Narrative  
Report `abc123` appears in multiple logs:  
- Detected at 15:30, clustered as *“Social Media Viral”*, attributed as *“TikTok Challenge”*, both with low confidence.  
- User, who has spoken to retail chain, later finalises as *“Competitor Stockout”*.  
Hydration coalesces these into a single row:  
`final_cause="Competitor Stockout"`, `confidence_score=0.78`, `cluster_name="Social Spike"`, `report_status="finalised"`.  

---

## 8. Resilience  
- **Null-tolerant joins** ensure reports still appear even if later logs are missing.  
- **Fixed precedence chain** prevents circular dependencies.  
- **Append-only rebuild** means hydration can always be replayed safely.

---

### Output Schema — `minority_reports` (Hydrated View)

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic identifier for each anomaly (`hash(store_id || first_detected_from)`). |
| `report_group_id` | string | Cohort or event-level identifier linking report to an event. |
| `event_status` | string | Current lifecycle state of the event (`active` / `ended`). |
| `version` | integer | Version number for replayable updates. |
| `store_id` | string | Retailer or store identifier. |
| `sku` | string | Product identifier linked to the anomaly. |
| `window_start` | timestamp | Start time of the anomaly window. |
| `window_end` | timestamp | End time of the anomaly window (nullable if ongoing). |
| `severity_score` | double | Intensity of anomaly as derived from detection. |
| `impact_estimate` | decimal | Estimated commercial impact (uplift or loss). |
| `run_id` | string | Execution key linking record to demo session. |
| `attributed_sales_total` | integer | Actual sales recorded during anomaly window. |
| `baseline_sales_total` | integer | Expected baseline sales for comparison. |
| `feature_vector` | array | Feature vector describing anomaly shape (from clustering). |
| `cluster_id` | string | Cluster identifier inherited from MRCL. |
| `cluster_name` | string | Label for cluster type (e.g., *TikTok Viral*, *Stockout*). |
| `similarity_score` | double | Similarity of anomaly to its cluster centroid. |
| `cluster_match_confidence` | double | Confidence in cluster match. |
| `tsne_x` | double | 2D embedding coordinate (x) for cluster visualization. |
| `tsne_y` | double | 2D embedding coordinate (y) for cluster visualization. |
| `proposed_cause` | string | Primary proposed cause generated by attribution model. |
| `proposed_cause_category` | string | Category of the primary proposed cause. |
| `confidence_score` | double | Confidence score for the primary cause. |
| `proposed_cause_2` | string | Secondary proposed cause (alternative hypothesis). |
| `proposed_cause_category_2` | string | Category of the secondary cause. |
| `confidence_score_2` | double | Confidence score for the secondary cause. |
| `proposed_cause_3` | string | Tertiary proposed cause (fallback hypothesis). |
| `proposed_cause_category_3` | string | Category of the tertiary cause. |
| `confidence_score_3` | double | Confidence score for the tertiary cause. |
| `supporting_evidence` | string | Optional free-text notes or external references. |
| `cohort_id` | string | Cohort identifier inherited from MRCH. |
| `cohort_name` | string | Human-readable label for the cohort. |
| `cohort_confidence` | double | Confidence in the cohort-level cause. |
| `cohort_reason` | string | Reason the cohort was formed (e.g., *temporal proximity*, *shared cluster*). |
| `cohort_size` | integer | Number of reports grouped into the cohort. |
| `final_decision` | string | Final outcome (approved / rejected / pending). |
| `final_cause` | string | Final confirmed cause (after analyst or system validation). |
| `final_confidence` | double | Final confidence level post-validation. |
| `finalized_at` | timestamp | Timestamp when the report was finalized. |
| `annotation_id` | string | Unique ID for analyst annotation (links to `ui_edits_log`). |
| `report_status` | string | Lifecycle state of the report (`proposed`, `finalised`, `reopened`). |
| `refresh_seq` | long | Sequence number for hydration refresh tracking. |

**Properties**
- Unified, **hydrated view** merging detection, clustering, attribution, cohorting, and finalisation layers.  
- One row per `report_id` representing the most recent state of each anomaly.  
- Fully replayable and lineage-safe via deterministic IDs and source provenance.  
- Primary dataset powering the Workshop UI and dashboards.

---

**Summary:**  
Hydration is the reconciliation layer of the Minority Report System.  
It deterministically fuses every log into a coherent, analyst-facing view — idempotent, replayable, and enterprise-ready.
