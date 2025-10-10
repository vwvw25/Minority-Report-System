# Minority Report System (MRS) Strategy

## Goal
Detect, cluster, and attribute short-term sales anomalies across retailers so root causes (e.g., social/viral effects, competitor stockouts, data errors, promos) are identified early and downstream decision systems (MAB, MMM) are protected from distortion.

---

## Categorization
We use an internal anomaly taxonomy with three dimensions:

- **Detection Type** — source category of the anomaly (Data Error, Marketing Misattribution, Competitor Stockout, Social/Viral Event, Operational/Store Issue).  
- **Impact Vector** — which part of the commercial system is distorted (baseline drift, attribution error, spend leakage, operational disruption).  
- **Confidence Tier** — escalation thresholds used by automation/HITL (Low / Medium / High).  

---

## Strategy Abstract
High-level flow:

1. **Baseline & detect (internal)**  
   - Baselines are computed inside detection from historical sales (per store/SKU/window).  
   - No external baseline feed is ingested.  
   - Anomalies (“Minority Reports”) are recomputed on each new data arrival with a deterministic `report_id` (hash of store_id + first_detected_from), keeping the system stateless and idempotent.  

2. **Cluster**  
   - Group similar reports using feature similarity; attach cluster metadata.  

3. **Attribute**  
   - Propose likely causes (category + narrative) with confidence.  
   - Secondary/tertiary candidates stored in fixed `_2`, `_3` columns.  

4. **Cohort**  
   - Assign reports to cohorts (`report_group_id`) to represent a Minority Event — the primary unit of analysis.  

5. **Hydrate & aggregate**  
   - Build a wide, one-row-per-report object for UI.  
   - Build a one-row-per-event roll-up with timings, totals, and aligned retailer/store fields.  

6. **Automation/HITL**  
   - Apply confidence rules (e.g., weekend mode auto-confirm at >70%).  
   - Surface to analysts when confidence is lower or decisions are consequential.  

7. **Protection**  
   - Feed event status/attribution back to MAB/MMM to prevent false credit and baseline corruption.  

---

## Technical Context

### Design Principles
- **Log-driven & stateless** — every stage appends to an authoritative log; no stage mutates hydrated objects.  
- **Open loops** — closed events can reopen; confidence evolves as data accumulates; context can reframe meaning without breaking lineage.  
- **Deterministic & append-only identity** — each report and event ID is derived from stable inputs (`report_id = hash(store_id || window_start)`), ensuring reproducible joins and lineage.   

---

### Deterministic & Append-Only Identity
Report identity is guaranteed by hashing `store_id || first_detected_from`.  
- `first_detected_from` is derived from a reproducible rule in the anomaly classifier.  
- This ensures the same anomaly always produces the same `report_id` on replay — the foundation of idempotence.  
- All downstream joins depend on this guarantee.

Foundry’s immutability model aligns naturally with this: logs never mutate, they only append. Each row is a snapshot of anomaly state at its `written_at` timestamp. Hydrate materialises the latest state while preserving full audit history.

**Key properties:**  
- **Append-only:** every update adds a new row, never overwriting.  
- **Snapshots:** each row represents a moment in the anomaly’s lifecycle.  
- **Hydration resolves identity:** multiple rows with the same `report_id` are coalesced into the latest version.  
- **Auditability:** full replay of anomaly evolution over time.  
- **Determinism:** stable IDs + append-only logs = reproducible results.

#### Example

**Log (minority_reports_detected_log):**

| report_id | store_id | window_start | window_end | severity | written_at |
|------------|-----------|---------------|-------------|-----------|-------------|
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | NULL | 0.42 | 10:00 |
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | NULL | 0.57 | 10:05 |
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | 10:30 | 0.81 | 10:30 |

**Hydrated object (minority_reports):**

| report_id | store_id | window_start | window_end | severity | written_at |
|------------|-----------|---------------|-------------|-----------|-------------|
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | 10:30 | 0.81 | 10:30 |

---

### Mocked Components in Demo
Several components are mocked to allow the full pipeline to run without heavy ML infrastructure.  

- **Mocked components:** baseline model, anomaly classifier, clustering model, attribution model.  
- **Method:** deterministic thresholds, static feature vectors or NULLs, lookup-based narratives, fallback rules.  
- **Purpose:** demonstrate system design (log-driven, acyclic, idempotent), not ML training.  
- **In production:** each mocked model would be a true ML component with versioning, retraining, and serving.  

Strategic mocking preserves architecture integrity — models remain in place and emit schema-correct outputs, so the pipeline behaves exactly as production would. Only the learning step is omitted.

---

### Authoritative Logs
- `minority_reports_detected_log` — per-report totals & window_start; includes computed baseline and attributed vs. baseline sales.  
- `minority_reports_clustered_log` — cluster metadata.  
- `minority_reports_proposed_attribution_log` — proposed causes (+ optional 2/3), confidence, evidence.  
- `minority_event_ended_log` — authoritative window_end.  
- `minority_reports_cohort_log` — per-report cohort assignment (report_group_id, size, confidence).  
- `minority_reports` (hydrated view) — wide, latest-per-log merge for UI.  
- `minority_events_log` — one row per event with report_ids, aligned `store_id_1…14` and `retailer_1…14` pairs, min/max window, sums/averages, latest cause, confidence, status.
