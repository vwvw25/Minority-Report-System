# Data Lineage — Minority Report System  

---

## 1. Purpose  
This document describes how data moves through the Minority Report System (MRS) — from raw inputs to hydrated ontology objects used by the human-in-the-loop interface.
It captures the end-to-end lineage between transforms, datasets, and user interfaces, ensuring full transparency and reproducibility.

---

## 2. Logical Flow  

[sales_timeseries_data]  
   │  
   ▼  
**[Detection Stage]**  
→ `minority_reports_detected_log` (MRDL)  
→ `minority_event_ended_log` (MEEL) *(closure events)*  
   │  
   ▼  
**[Clustering Stage]**  
→ `minority_reports_clustered_log` (MRCL)  
   │  
   ▼  
**[Attribution Stage]**  
→ `minority_reports_proposed_attribution_log` (MRPAL)  
   │  
   ▼  
**[Cohorting Stage]**  
→ `minority_reports_cohorted_log` (MRCH)  
→ `minority_events` (MELOG)  
   │  
   ▼  
**[User Interaction]**  
→ `user_edits_log` *(per-report edits)*  
→ `user_minority_events_edits_log` *(event-level edits)*  
   │  
   ▼  
**[Finalisation Stage]**  
→ `minority_reports_finalised_log` (MRFL)  
   │  
   ▼  
**[Hydration Stage]**  
→ `minority_reports` *(wide view, one row per report)*  
→ `minority_events` *(wide view, one row per event)*  
   │  
   ▼  
**[Rereview Stage]** *(downstream-only)*  
→ `minority_reports_rereview_worklist` (MRRW)  
→ `minority_reports_clustered_rereview_log`  
→ `minority_reports_proposed_attribution_rereview_log`  

---

## 3. Dataset Index  

| Code | Dataset Name | Produced By | Consumed By | Purpose |
|------|---------------|-------------|--------------|----------|
| **MRDL** | `minority_reports_detected_log` | `detect_anomalies.py` | MRCL, Cohorting, Hydration | Authoritative detection log capturing anomalies, timings, and sales context. |
| **MEEL** | `minority_event_ended_log` | `detect_anomalies.py` | Attribution, Hydration | Authoritative event-closure log, used to mark anomaly windows as ended. |
| **MRCL** | `minority_reports_clustered_log` | `feature_vector_cluster_match.py` | Attribution, Hydration | Cluster assignments, similarity scores, and confidence metrics. |
| **MRPAL** | `minority_reports_proposed_attribution_log` | `propose_cause_for_minority.py` | HITL, Finalisation | Proposed causes and confidence levels for each anomaly. |
| **MRCH** | `minority_reports_cohorted_log` | `cohort_reports.py` | Hydration | Per-tick snapshots of active and ended anomaly groups. |
| **MELOG** | `minority_events` | `cohort_reports.py` | Hydration | Registry of all active and ended events (one row per event). |
| **user_edits_log** | `user_edits_log` | UI (analyst actions) | Finalisation | Human-in-the-loop per-report edits and annotations. |
| **user_minority_events_edits_log** | `user_minority_events_edits_log` | UI (analyst actions) | Finalisation, Hydration | Analyst edits to minority event objects (metadata, evidence, etc.). |
| **MRFL** | `minority_reports_finalised_log` | `build_minority_reports_finalised_log_from_edits.py` | Hydration | Final authoritative merge of machine proposals and analyst decisions. |
| **minority_reports** | `minority_reports` *(view)* | `hydrate_minority_reports.py` | UI, API | Hydrated, latest-per-report object for user interface. |
| **minority_events_log** | `minority_events` *(view)* | `hydrate_minority_reports.py` | UI, API | Wide event-level aggregation for event dashboards. |
| **MRRW** | `minority_reports_rereview_worklist` | `build_rereview_worklist.py` | rereview_cluster_reports.py | Queue of reports eligible for rereview. |
| **MRCRL** | `minority_reports_clustered_rereview_log` | `rereview_cluster_reports.py` | rereview_propose_cause.py, Hydration | Rerun clustering results for historical low-confidence reports. |
| **MRPARL** | `minority_reports_proposed_attribution_rereview_log` | `rereview_propose_cause.py` | Hydration | Updated attribution results after rereview pass. |

---

## 4. Why Both `minority_reports_cohorted_log` (MRCH) and `minority_events` (MELOG) Exist

Although both datasets relate to the same underlying anomaly-event lifecycle, they serve distinct and complementary purposes:

| Log | Purpose | Characteristics |
|------|----------|----------------|
| **MRCH — minority_reports_cohorted_log** | **Time-series snapshots** of each active event. Captures how the system’s understanding of an event evolves over time (e.g., expanding window, changing severity, cumulative impact). | Multiple rows per event; append-only; used for temporal analysis, trend visualisation, and audit of intermediate states. |
| **MELOG — minority_event_log** | **Stable registry** of each event. One row per event summarising final or latest state (start, end, totals, cause, status). | Single authoritative record per event; used for attribution, reporting, and integration with MMM/MAB. |

**Design Principle:**  
MRCH shows *how understanding changed over time*, while MELOG provides *what the system ultimately concluded*.  
Both are required for full lineage:

- **Temporal auditability** → MRCH preserves every historical snapshot.  
- **Lineage stability** → MELOG provides deterministic, joinable event identities.  
- **Resilience and performance** → MELOG offers a lightweight registry when MRCH grows large.  

Together, they guarantee complete replayability and audit-readiness of event-level intelligence.

## 5. Lineage Summary  

1. **MRDL → MRCL → MRPAL → MRCH → MRFL → Hydration**  
   - Primary lineage of a Minority Report from detection to finalisation.  
   - Each stage appends its own authoritative log; no overwrites.  

2. **MEEL + MELOG → Hydration**  
   - Define event lifecycle (`active`, `ended`).  
   - Feed into the event-wide views used in the UI.  

3. **User Logs (UI)**  
   - `user_edits_log` and `user_minority_events_edits_log` supply human overrides.  
   - These feed into Finalisation and are auditable via annotation IDs.  

4. **Rereview Path**  
   - Independent, downstream-only loop.  
   - Rereview logs (`*_rereview_log`) exist alongside originals.  
   - Hydration merges rereview results with precedence, never overwriting MRCL/MRPAL/MRFL.  

---

## 6. Design Properties  

- **Append-only:** every dataset records immutable history.  
- **Deterministic:** stable IDs (`report_id`, `report_group_id`) guarantee consistent lineage.  
- **Replayable:** any past state can be reconstructed by replaying transforms.  
- **Governed:** every dataset includes `run_id`, `written_at`, and provenance columns for full auditability.  
- **Acyclic:** rereview runs downstream only — no circular dependencies.  

---

**Summary:**  
This lineage demonstrates how the Minority Report System maintains a complete, deterministic data trail — from anomaly detection through human interaction, finalisation, and optional rereview — all without mutating historical data.  
Every dataset in the flow is traceable, auditable, and reproducible, forming the backbone of the MRS governance model.
