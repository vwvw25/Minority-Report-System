# Minority Report System (MRS) Strategy  
**File:** `strategy.md`




---

## Goal  
Detect, cluster, and attribute short-term sales anomalies across retailers so that likely root causes  
(e.g., viral effects, competitor stockouts, data errors, promotions) are identified early and  
downstream decision systems (e.g., MAB, MMM) are protected from distortion.  

---

## Strategy Abstract  

### 1. **Detection**  
- Generates baseline and anomaly metrics using a **mocked baseline generator**.  
- No external baseline feed is ingested in the MVP.  
- Anomalies (“Minority Reports”) are recomputed deterministically on each data arrival,  
  using a stable identifier:  
  `report_id = hash(store_id || first_detected_at)`.  
- The stage is **stateless and idempotent** — re-running detection always yields identical IDs and joins.  

### 2. **Clustering**  
- Groups similar anomalies by comparing feature shapes and timing patterns.  
- In the MVP, clustering is mocked — reports are assigned pre-defined clusters  
  via deterministic lookups from `master_narratives`.  
- In production, a true model (e.g., UMAP + cosine similarity) would replace this logic.  

### 3. **Attribution**  
- Proposes plausible causes (`proposed_cause`, `proposed_cause_category`)  
  and confidence scores for each anomaly.  
- In the MVP, causes are also sourced from `master_narratives`.  
- In production, this would be replaced by a trained attribution model drawing on contextual and campaign data.  

### 4. **Cohorting**  
- Groups temporally overlapping anomalies that share cluster or attribution characteristics  
  into inferred events called **Minority Events** (`report_group_id`).  
- Each cohort represents a single hypothesised underlying cause  
  (e.g., a viral post, competitor stockout, or campaign overlap).  
- Cohorts are written to the **Minority Reports Cohorted Log (MRCH)**  
  and the event registry **Minority Event Log (MELOG)**.  

### 5. **Hydration & Aggregation**  
- Coalesces all authoritative logs (detection, clustering, attribution, cohorting, finalisation)  
  into a single unified dataset via `hydrate_minority_reports.py`.  
- This hydrated dataset underpins the **Minority Report ontology object**,  
  which is viewable and editable in the Workshop UI.  
- Also produces an event-level roll-up (`minority_events`) for analytical use.  

### 6. **Automation / HITL (Illustrative)**  
- Demonstrates how confidence thresholds could drive auto-confirmation  
  or route reports to analysts for manual review.  
- No automation thresholds are implemented in the MVP; logic is illustrative only.  

### 7. **(Future) Protection**  
- Design placeholder showing where validated event status and attribution  
  could feed back into MAB/MMM models to prevent mis-attribution  
  and protect baseline integrity.  

---

## Technical Context  

### Design Principles  

- **Log-driven & stateless:** every stage appends to an authoritative log;  
  no transform ever mutates upstream or hydrated datasets.  
- **Open loops:** closed events can reopen when new data changes understanding;  
  confidence evolves without breaking lineage.  
- **Deterministic & append-only identity:**  
  each report and event ID is derived from stable inputs  
  (`report_id = hash(store_id || first_detected_at)`),  
  ensuring reproducibility across all joins.  

---

### Deterministic & Append-Only Identity  

Report identity is guaranteed by hashing `store_id || first_detected_at`.  
- `first_detected_at` is a deterministic timestamp derived from the anomaly classifier.  
- The same anomaly always produces the same `report_id` on replay —  
  the foundation of idempotence.  
- Downstream joins, clusters, and cohorts depend on this guarantee.  

Foundry’s immutability model aligns with this design: logs never mutate, they only append.  
Each row is a snapshot of anomaly state at its `written_at` timestamp.  
Hydration materialises the latest state while preserving full audit history.  

**Key properties:**  
- **Append-only:** every update adds a new row.  
- **Snapshots:** each row represents a point in the anomaly’s lifecycle.  
- **Hydration resolution:** multiple rows with the same `report_id`  
  are coalesced into the latest version.  
- **Auditability:** full replay of anomaly evolution.  
- **Determinism:** stable IDs + append-only logs = reproducible results.  

#### Example  

**Log (minority_reports_detected_log):**

| report_id | store_id | window_start | window_end | severity | written_at |
|------------|-----------|---------------|-------------|-----------|-------------|
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | NULL | 0.42 | 10:00 |
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | NULL | 0.57 | 10:05 |
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | 10:30 | 0.81 | 10:30 |

**Hydrated dataset (minority_reports):**

| report_id | store_id | window_start | window_end | severity | written_at |
|------------|-----------|---------------|-------------|-----------|-------------|
| MR-hash(STORE_BIR_001\|2025-08-07T09:00:00Z) | STORE_BIR_001 | 09:00 | 10:30 | 0.81 | 10:30 |

---

### Mocked Components in the MVP  

Several components are mocked so the full pipeline can run  
without external models or heavy ML infrastructure.  

- **Mocked components:** baseline generator, anomaly classifier, clustering model, attribution model.  
- **Data source:** `master_narratives` provides pre-seeded causes, clusters, and timeseries.  
- **Method:** deterministic thresholds, static vectors, lookup-based narratives.  
- **Purpose:** demonstrate full log-driven architecture (acyclic, idempotent, replayable).  
- **In production:** these would become real ML models with versioning and retraining pipelines.  

This approach preserves architecture integrity —  
models remain wired and schema-correct so the pipeline behaves as it would in production,  
only omitting the learning step.  

---

### Authoritative Logs  

| Log | Description |
|-----|--------------|
| **MRDL — minority_reports_detected_log** | Anomaly detections with baseline and uplift metrics. |
| **MRCL — minority_reports_clustered_log** | Cluster assignments and similarity metrics. |
| **MRPAL — minority_reports_proposed_attribution_log** | Proposed causes, categories, and confidence scores. |
| **MRCH — minority_reports_cohorted_log** | Per-report cohort assignments (`report_group_id`, size, confidence). |
| **MELOG — minority_event_log** | One row per inferred event with aggregated fields. |
| **MRFL — minority_reports_finalised_log** | Consolidated state for governance and rereview. |
| **MRRW — minority_reports_rereview_worklist** | Tracks historical reports queued for rereview. |
| **minority_reports** | Hydrated dataset backing the Minority Report ontology object for the Workshop UI. |

---

## Validation Scenarios (MVP)  

**Scenario MRS-001: Competitor Supply Disruption**  
A major competitor experiences a supply chain failure, creating a temporary sales uplift.  
- Detection captures anomaly. 
- Clustering clusters it with competitor stockouts. 
- Attribution tags it as *Competitor Stockout*.  
- Cohorting cohorts the report with other reports that are part of the same Minority Event.
- Subscribed users receive an alert for the Minority Event if it passes threshold metrics.
- Marketing team user view the event via the UI and call retail contact to verify.
- User approves the report.
- MAB reads output dataset, preventing misattribution. 

**Scenario MRS-002: Viral Content Spike**  
Social media virality causes a transient demand surge.  
- Detection captures anomaly. 
- Clustering clusters it with Viral Social Media with a low confidence.
- Attribution tags cause as *Viral Social Media* with a medium confidence. 
- Cohorting cohorts the report with other reports that are part of the same Minority Event.
- Subscribed users receive an alert for the Minority Event if it passes threshold metrics.
- Marketing team user looks at social listening platform and identifies viral video. 
- User adds link to video in supporting_evidence field and approves the Minority Event. 
- MAB reads output dataset, preventing misattribution. 

**Scenario MRS-003: Localised Weather Event**  
Severe weather produces a temporary regional distortion.  
- Detection captures anomaly. 
- Clustering clusters it with Weather Event. 
- Attribution tags cause as *Viral Social Media* with a medium confidence. 
- Cohorting cohorts the report with other reports that are part of the same Minority Event.
- Subscribed users receive an alert for the Minority Event if it passes threshold metrics.
- Marketing team user uses 'request review' option to request team by Data Analysis team.  
- Data Analysis team establishes the cause was a weather event not reflected in context data, edits and approves report.
- Too late to be useful to MAB on this occasion, but the data helps with model retraining. \\\\\\\\\\\\

---

**Summary:**  
The MRS demonstrates a log-driven, stateless, idempotent system that can  
detect, cluster, and attribute short-term anomalies deterministically.  
While its models are mocked, the end-to-end design is faithful to  
production-grade architectural principles — ensuring reproducibility,  
auditability, and seamless future substitution of real machine-learning models.  
