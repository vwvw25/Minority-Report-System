# System Overview — Minority Report System  
**File:** `system_overview.md`  
**Last Updated:** 2025-10-13  

---

## 1. Purpose  
This document provides a high-level overview of the **Minority Report System (MRS)** architecture — how data flows through the pipeline, how each stage interacts, and how the system achieves deterministic, replayable anomaly detection and attribution.

The overview connects concepts defined in:
- `architecture/strategy.md`
- `stages/*_stage.md`
- `architecture/governance_hooks.md`

---

## 2. Overview  

The MRS is part of a broader analytics platform combining an MMM and a MAB to answer the question:  
> “Which combination of TPO and marketing campaigns will produce the highest ROI on spend in June?”

While the MMM sets total budgets for TPO and marketing campaigns at a **monthly** level, the MAB allocates those budgets **daily** across auction-based platforms (e.g. Facebook, TikTok, Programmatic).  

The MRS functions as a **protective layer** between raw sales data and downstream optimisation systems.  
It identifies **sales anomalies** and establishes their causes to prevent them being misattributed to paid campaigns — misattribution that could otherwise cause the MAB to increase spend on the wrong activities.

---

## 3. Architectural Summary  

**System goal:**  
Identify, cluster, and attribute short-term sales anomalies in a reproducible, enterprise-safe way.

**Core characteristics:**  
- **Log-driven:** every transform appends immutable logs.  
- **Stateless:** all state derived from logs at runtime.  
- **Idempotent:** identical inputs yield identical outputs.  
- **Acyclic:** rereview runs downstream only; no circular dependencies.  
- **Governed:** every record traceable via deterministic IDs.  

---

## 4. High-Level Flow

```
unified_sales_data  →  sales_timeseries_data
            │
            ▼
[Detection Stage]
 detect_anomalies.py
 ├── baseline_sales_model_prediction_log
 ├── anomaly_classifier_model_prediction_log
 └── minority_reports_detected_log (MRDL)
            │
            ▼
[Clustering Stage]
 cluster_minority_report.py
 └── minority_reports_clustered_log (MRCL)
            │
            ▼
[Attribution Stage]
 propose_cause_for_minority.py
 └── minority_reports_proposed_attribution_log (MRPAL)
            │
            ▼
[Cohorting Stage]
 build_minority_report_cohorts.py
 ├── minority_reports_cohorted_log (MRCH)
 └── minority_events (MELOG)
            │
            ▼
[User Edit Stages — Workshop UI]
 ├── user_edits_log (report-level)
 └── user_minority_events_edits_log (event-level)
            │
            ▼
[Finalisation Stage]
 build_minority_reports_finalised_log_from_edits.py
 └── minority_reports_finalised_log (MRFL)
            │
            ▼
[Hydration Stage]
 hydrate_minority_reports.py
 └── minority_reports (hydrated dataset)

            │
            ▼
[Rereview Stage — optional]
 ├── build_rereview_worklist.py
 ├── rereview_cluster_reports.py
 └── rereview_propose_cause.py
 ```

The system assumes that sales data is already collated, cleaned, and unified upstream into `unified_sales_data` by the wider platform.  
This dataset is transformed into `sales_timeseries_data`, providing the per-store, per-SKU temporal series from which anomalies are detected.

---

## 5. Data Contracts  

| Dataset | Type | Description |
|---------|------|-------------|
| `MRDL` | Log | Authoritative detection log. |
| `MRCL` | Log | Cluster assignments and confidence. |
| `MRPAL` | Log | Proposed attribution causes and confidence. |
| `MRCH` | Log | Per-tick snapshots of cohort (event) evolution. |
| `MELOG` | Log | One stable row per event (event registry; created by the Cohorting stage). |
| `user_edits_log` | Log | Analyst edits to individual reports via the UI. |
| `user_minority_events_edits_log` | Log | Analyst edits to the **Minority Event** object (cohort-level). |
| `MRFL` | Log | Finalised report rows (merge of model outputs + edits; used for rereview). |
| `minority_reports` | Dataset | Hydrated, one-row-per-report dataset used by the UI. |
| `minority_events` | Dataset | Backing dataset for Minority Events in the UI (created by cohorting). |

Each dataset is append-only, typed, and reproducible.  
Hydration deterministically assembles the **current state** for UI consumption without mutating upstream logs.

---

## 6. Identity and Lineage  

Deterministic identity keys guarantee lineage and reproducibility:

- `report_id = hash(store_id || first_detected_at)` uniquely identifies each anomaly.  
- `report_group_id = hash(report_id || earliest_first_detected_at_in_cohort)` groups anomalies into persistent cohorts (Minority Events).

This ensures idempotent recomputation, consistent joins, and complete auditability across all pipeline stages.

---

## 7. Supporting Ontology Objects

| Ontology | Purpose |
|-----------|----------|
| **Store** | Provides retailer-level and geographic metadata for joining sales and anomaly data. |
| **Campaign** | Supplies contextual and marketing campaign information used during attribution. |
| **Minority Report** | Represents a single detected anomaly, hydrated from detection, clustering, attribution, and finalisation logs. |
| **Minority Event** | Represents a cohort of related anomalies inferred to share a common cause, created by the Cohorting stage. |

---

## 8. Governance Hooks  
Governance fields are consistent across all logs:  
`report_id`, `run_id`, `written_at`, `report_status`, `is_degraded`, `degraded_reasons`, `source_used_*`.  
They ensure auditability, replayability, and regulatory traceability.  
See [Governance Hooks](../architecture/governance_hooks.md) for detail.

---

## 9. Operational Model  

- **Failure containment:** Each stage reads from defined upstream logs with fixed fallback precedence; upstream issues degrade outputs but do not halt execution.  
- **Graceful degradation:** When inputs are missing, fields are populated via fallback sources and the record is explicitly flagged (`is_degraded=true`, `degraded_reasons`, `report_status` annotated).  
  This preserves continuity while making degradation transparent and auditable.  
- **Rebuildability:** Any point-in-time state can be deterministically reconstructed from append-only logs.  
- **Observability:** Health metrics (freshness, coverage, error rate, degradation rate) are continuously monitored and exposed in dashboards.

### 9.1 Alerting and Notification (Design-Complete; Implementation Pending)
When a new Minority Event exceeds configured thresholds (e.g., severity, total attributed sales), the system triggers an alert. 

- **Source:** Alerts are emitted when report or event criteria are met.  
- **Recipients:** Subscribed users and managers for the relevant scope (retailer/region/category).  
- **Delivery:** Email and in-platform notification (link opens the corresponding Minority Event in the UI).  
- **Audit:** Every alert is logged to `ui_notifications_log` with `event_id`, `trigger_condition`, `delivered_to`, and `written_at`.

---

## 10. User Interaction Path  

Analysts, managers, and data scientists interact with the system through the **Workshop UI**, which provides both monitoring and human-in-the-loop correction.

Below is a representative flow, showing how a user and the system interact end-to-end.

### Scenario MRS-002: Viral Content Spike

A viral social media post drives a transient surge in sales.

1. **Detection** captures the anomaly from `sales_timeseries_data`.  
2. **Clustering** assigns it to the *Viral Social Media* cluster with low confidence.  
3. **Attribution** proposes *Viral Social Media* as the cause with medium confidence.  
4. **Cohorting** groups the report with others showing similar patterns, forming a single Minority Event.  
5. **Alerting** triggers a notification when the event’s severity passes defined thresholds.  
6. **Analyst review:** a marketing analyst opens the event in the UI, checks social listening tools, and confirms a viral video is trending.  
7. **Edit:** the analyst adds the video link in the `supporting_evidence` field and approves the Minority Event.  
8. **Finalisation:** the system records the decision in `user_minority_events_edits_log` and merges it into `MRFL`.  
9. **Protection:** the MAB reads the output dataset, preventing misattribution of the uplift to paid marketing.

This flow demonstrates how human judgment, model outputs, and governance logs combine to protect downstream optimisation systems while maintaining transparency and control.

---

## 11. System Guarantees  

| Property | Description |
|---------|-------------|
| **Determinism** | Re-running the pipeline produces identical outputs. |
| **Immutability** | Logs never mutate; all history is preserved. |
| **Traceability** | Each field is traceable via `report_id`/`report_group_id` lineage. |
| **Audit readiness** | Every change is stamped with author and timestamp. |
| **Extensibility** | Mocked models can be replaced with real ones without schema change. |

---

## 12. Demo Reset Design  

The MRS uses a **Run-ID architecture** that allows clean demo resets without losing historical data:

- Every dataset includes a `run_id` column.  
- The active `run_id` is stored in `demo_run_config`.  
- Read paths filter by that value, isolating the current run.

**Benefits**  
- Instant reset while preserving previous sessions.  
- Enables before/after comparisons for reviewers.  
- Guarantees reproducible lineage across multiple demos.

**Example query:**
```sql
SELECT * 
FROM minority_reports
WHERE run_id = (SELECT run_id FROM demo_run_config)
AND report_status = 'proposed';
