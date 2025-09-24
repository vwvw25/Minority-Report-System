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
   - Assign reports to cohorts (`report_group_id`) to represent a Minority Event — our primary unit of analysis.  

5. **Hydrate & aggregate**  
   - Build a wide, one-row-per-report object for UI.  
   - Build a one-row-per-event roll-up with timings, totals, aligned retailer/store fields.  

6. **Automation/HITL**  
   - Apply confidence rules (e.g., weekend mode auto-confirm at >70%).  
   - Surface to analysts when confidence is lower or decisions are consequential.  

7. **Protection**  
   - Feed event status/attribution back to MAB/MMM to prevent false credit and baseline corruption.  

---

## Technical Context

### Design Principles
- **Log-driven & stateless** — every stage appends to an authoritative log; no stage mutates hydrated objects.  
- **Open loops (neuroscience-inspired)** — closed events can reopen; confidence evolves as data accumulates; context can reframe meaning without breaking lineage.  
- **Deterministic identities** — `report_id = hash(store_id || window_start)`; event IDs derived from stable inputs.  
- **UI constraint** — one row per report; multi-candidate attribution encoded as `_2`, `_3` columns (no arrays).  

#### Deterministic Identity
Report identity is guaranteed by hashing `store_id || first_detected_from`.  
- `first_detected_from` is derived from a reproducible rule in the anomaly classifier model.  
- This guarantees the same anomaly produces the same `report_id` on every replay, ensuring idempotence and reliable joins across logs.
- All downstream logs and joins depend on this guarantee; it is non-negotiable.  

#### Primary Key Semantics

All core datasets in the Minority Report System are append-only logs. This behaviour is natural in Foundry: logs never mutate, they only accumulate, and hydrate is the mechanism that materialises state.
Each new row represents a fresh snapshot of an anomaly; existing rows are never updated or deleted.  
This reflects Foundry’s immutability model: datasets grow only by appending rows, and state is materialised later through views such as `hydrate_minority_reports`.  

Within this context, `report_id` functions as the logical primary key.  
Multiple rows with the same `report_id` are expected and represent successive versions of the same anomaly over time, not duplicates.  

- **Append-only** — Every change produces a new row; history is preserved.  
- **Snapshots** — Each row encodes anomaly state at its `written_at` timestamp.  
- **Hydration resolves identity** — Rows with the same `report_id` are coalesced to the latest version.  
- **Auditability** — Because no history is overwritten, the entire lifecycle of an anomaly can be replayed.  
- **Determinism** — Stable report_ids + append-only logs guarantee reproducible results across replays.  

#### Example: Append-only primary key

Append-only logs can contain multiple rows for the same `report_id`.  
Each row is a **snapshot** at a different `written_at`. Hydrate coalesces these into the latest version.

---

**Log (minority_reports_detected_log):**

| report_id                                              | store_id | window_start | window_end | severity | written_at |
|--------------------------------------------------------|----------|--------------|------------|----------|------------|
| MR-hash(STORE_BIR_001|2025-08-07T09:00:00Z)            | STORE_BIR_001 | 09:00        | NULL       | 0.42     | 10:00      |
| MR-hash(STORE_BIR_001|2025-08-07T09:00:00Z)            | STORE_BIR_001 | 09:00        | NULL       | 0.57     | 10:05      |
| MR-hash(STORE_BIR_001|2025-08-07T09:00:00Z)            | STORE_BIR_001 | 09:00        | 10:30      | 0.81     | 10:30      |

---

**Hydrated object (minority_reports):**

| report_id                                              | store_id | window_start | window_end | severity | written_at |
|--------------------------------------------------------|----------|--------------|------------|----------|------------|
| MR-hash(STORE_BIR_001|2025-08-07T09:00:00Z)            | STORE_BIR_001 | 09:00        | 10:30      | 0.81     | 10:30      |

---

**Key points:**  
- Log is **append-only**: each new row adds a snapshot. No overwrites.  
- Multiple rows share the same `report_id` (deterministic from `store_id || first_detected_from`).  
- Hydrate chooses the latest by `written_at` (or `finalized_at` in finalisation logs).  
- This guarantees both **auditability** (all history is retained) and **determinism** (final object is reproducible).  
- Foundry’s append-only design makes this natural: logs never mutate, only accumulate.

---

#### Mocked Components in Demo

Several components are mocked to allow the full pipeline to run without heavy ML infrastructure.  
- **Which are mocked**: baseline model, anomaly classifier, clustering model, attribution model.  
- **How**: deterministic thresholds, static feature vectors or NULLs, lookup-based narratives, fallback rules.  
- **Why**: the purpose of this demo is to demonstrate system design (log-driven, acyclic, idempotent), not to train and serve ML models.  
- **Difference from real**: in production, each mocked model would be a proper ML system (forecasting, anomaly detection, clustering, attribution) with versioning, retraining, and serving infrastructure.  

Strategic mocking means models are not removed or simplified away — they remain in place in the pipeline. Each mocked model emits outputs shaped like real model predictions, so the structure, datasets, and downstream workflows behave exactly as they would in production. The only thing missing is the learning step itself.


---

### Authoritative Logs
- `minority_reports_detected_log` — per-report totals & window_start; contains internally computed baseline and attributed vs baseline sales.  
- `minority_reports_clustered_log` — cluster metadata.  
- `minority_reports_proposed_attribution_log` — proposed causes (+ optional 2/3), confidence, evidence.  
- `minority_event_ended_log` — authoritative window_end.  
- `minority_reports_cohort_log` — per-report cohort assignment (report_group_id, size, confidence).  
- `minority_reports` (hydrated view) — wide, latest-per-log merge for UI.  
- `minority_events_log` — one row per event with: report_ids, aligned `store_id_1…14` and `retailer_1…14` pairs, min/max window, sums/averages, latest cause, confidence, status.  
