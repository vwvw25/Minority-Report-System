# Model Lifecycle — Minority Report System  
**File:** `architecture/model_lifecycle.md`  
**Last Updated:** 2025-10-12  

---

## 1. Purpose  

This document defines how models are structured, trained, and integrated within the **Minority Report System (MRS)** — including mocked behaviour in the MVP build and production-ready architecture for future machine-learning deployment.  

MRS separates *model intelligence* from *pipeline mechanics*: each model writes to its own log and is versioned for full auditability.  
Although the current system uses mocked models, all interfaces and schemas are production-accurate, ensuring real ML components can be dropped in without breaking lineage or idempotence.  

---

## 2. Model Inventory  

| Model | Role | MVP Implementation | Future Implementation |
|--------|------|--------------------|------------------------|
| **Baseline Model** | Estimates expected sales for each store/SKU/time window. | Deterministic calculation of a “plausible” baseline directly from `sales_timeseries_data`, producing visually realistic but fabricated values. | Trained regression model (e.g. LightGBM) using historical sales, seasonality, promotions, and calendar features. |
| **Anomaly Classifier** | Determines whether deviations from baseline qualify as anomalies. | Wired transform executes but performs no learning — anomalies are predefined in `master_narratives` and surfaced as detections. | Supervised binary classifier trained on labelled anomalies. |
| **Clustering Model** | Groups anomalies by shape similarity. | Wired transform (`feature_vector_cluster_match.py`) performs lookups into `master_narratives`, assigning deterministic cluster names and similarity scores. | Unsupervised clustering (e.g. t-SNE + HDBSCAN) trained on feature vectors. |
| **Attribution Model** | Proposes likely causes and confidence scores. | Wired transform (`propose_cause_for_minority.py`) performs lookups into `master_narratives`, returning fixed narrative text and static confidence levels. | Multi-modal model combining campaign, social, and contextual features. |

---

## 3. Mock Data Generation  

The MVP simulation produces **believable but synthetic data** for a fixed window  
(7–12 August).  

- All anomalies correspond to fictional events described in `master_narratives`.  
- Only stores referenced in those narratives exhibit anomalies; others remain flat.  
- Baselines and anomalies are generated in the correct structure for downstream compatibility.  
- This allows the full detection → clustering → attribution → finalisation flow to operate as if real models were running.  

---

## 4. Model Build and Training (Production)  

When replaced with real models, training would occur entirely outside the runtime DAG.  

**Data Sources**  
- `unified_sales_data` — collated and cleaned multi-retailer sales feeds.  
- Campaign, contextual, and operational metadata.  
- Historical anomaly logs for supervised training.  

**Feature Engineering**  
- Rolling aggregates, lags, trend coefficients.  
- Campaign and promotion overlaps.  
- Contextual enrichments (weather, region, store type).  

**Training Process**  
1. Extract training data from authoritative logs.  
2. Split into train / validation / hold-out sets.  
3. Train models (regression, classifier, clustering, or attribution).  
4. Evaluate and record metrics.  
5. Register artefacts in `model_registry_log` with version, date, and metrics.  

---

## 5. Deployment and Versioning  

Each inference transform (detection, clustering, attribution) logs the version of the model used.  

**Metadata recorded per inference:**  
- `model_version`  
- `trained_on` date  
- `model_registry_source`  
- `model_confidence_metric`  

Version changes never alter schema.  
Rollbacks simply update the referenced `model_version` in configuration.  

---

## 6. Drift Detection and Retraining  

Model monitoring will compare **live distributions** against training references.  

**Drift Signals**  
- Input feature drift (distributional shift).  
- Residual drift (rising prediction error).  
- Confidence decay (widening uncertainty).  

**Retraining Triggers**  
Retraining is always offline and may occur due to:  
1. **Reactive retraining** — triggered by detected drift or reduced accuracy.  
2. **Proactive retraining** — scheduled to include new data and improve coverage as the dataset grows.  

Because MRS models are stateless, new data entering the system does not self-train models. Retraining produces a new `model_version` for subsequent runs.  

---

## 7. Retraining Workflow  

1. Detect drift or reach scheduled retrain interval.  
2. Extract updated training data from authoritative logs (MRDL, MRCL, MRPAL).  
3. Train and validate candidate model.  
4. Register new version in `model_registry_log`.  
5. Update transform configuration to reference new version.  
6. Optionally trigger **Rereview Stage** to reprocess historical reports using the new model.  

---

## 8. Governance and Auditability  

- **Model Registry:** every trained model is appended with metadata (version, parameters, metrics).  
- **Immutable lineage:** inference logs always reference a specific version.  
- **Approval control:** new versions require governance approval before activation.  
- **Drift and retrain events:** logged to `model_monitoring_log` and `model_retrain_event_log`.  
- **Reproducibility:** any past output can be traced to the exact model build and dataset.  

---

## 9. MVP Limitations  

- All model logic is mocked and deterministic.  
- No live drift monitoring or retraining loops exist.  
- Confidence values are static and not statistically derived.  
- Model versions are hard-coded for reproducibility.  
- The architecture is designed so each model can later be replaced by a real counterpart with no schema changes.  

---

## 10. Summary  

The Model Lifecycle defines the operational boundaries between model intelligence and the log-driven pipeline.  
Even in its mocked form, the system behaves like production ML infrastructure: deterministic outputs, versioned inference, and full auditability.  
When real models are integrated, they will inherit the same guarantees — **idempotent, reproducible, and fully traceable**.  

---
