# Clustering Stage — Minority Report System  
**File:** `clustering_stage.md`  

---

## 1. Purpose  
The clustering stage groups similar anomalies so analysts can see emerging patterns (e.g., viral spikes vs. data errors).  
Implemented in `cluster_minority_report.py`, it:  
1. Builds feature vectors from recent sales slices.  
2. Applies a clustering model (real or mocked).  
3. Writes results to the **Minority Reports Clustered Log (MRCL)** for downstream attribution.

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL):** anomaly metadata and timing.  
- **`sales_timeseries_data`:** source sales data to build anomaly slices.  
- **`demo_run_config`:** provides run_id.  

---

## 3. Output  
| Dataset | Purpose |
|----------|----------|
| `minority_reports_clustered_log` (MRCL) | Cluster assignment and similarity metrics per minority report. |

Key fields:  
`report_id`, `store_id`, `sku`, `cluster_id`, `cluster_name`, `similarity_score`, `cluster_match_confidence`, `feature_vector`, `written_at`, `run_id`.

---

## 4. Core Logic  
- **Slice Construction:** extract actual sales from window_start up to the latest detection timestamp (written_at) for each anomaly.
- **Feature Vector:** describe anomaly shape (uplift, slope, volatility, timing).  
  - In demo builds, vector may be mocked (NULL).  
- **Clustering Model:** assign deterministic `cluster_id` and `cluster_name`, compute similarity/confidence metrics.  
- **Append:** write one MRCL row per tick (append-only, replayable).

---

## 5. Operational Flow  
1. MRDL defines anomaly window.  
2. Sales slices built from `sales_timeseries_data`.  
3. Feature vectors generated.  
4. Clustering assigns group and metrics.  
5. MRCL row appended for each tick.  
6. Downstream attribution consumes MRCL.

---

## 6. Example Narrative  
Store 327 anomaly (15:30 → 17:00):  
- Feature vector shows sharp uplift.  
- Clustering model assigns `cluster_name="TikTok Viral"`, `similarity_score=0.92`, `cluster_match_confidence=0.87`.  
- MRCL row written (`report_id=abc123`, `cluster_id=7`, `written_at=17:00`).  

---

## 7. Implementation Realism (Refined)

In the MVP, the clustering stage is **structurally complete but functionally mocked**.  
The transform `cluster_minority_report.py` preserves the full data flow and schema expected in production but does not perform real vectorisation or similarity scoring.

**Mock Behaviour**
- The transform retrieves predefined cluster assignments and metadata directly from `master_narratives`.  
- All analytical fields (`feature_vector`, `similarity_score`, `cluster_match_confidence`, `tsne_x`, `tsne_y`) are populated deterministically or with placeholder values.  
- This ensures downstream transforms (attribution, hydration, finalisation) function exactly as they would in a production pipeline.

**Intended Production Behaviour**
- Compute feature vectors from anomaly time series (capturing shape, volatility, asymmetry, and temporal signatures).  
- Compare vectors against trained centroids in a “cluster library” using cosine or Euclidean distance.  
- Return best-fit cluster and confidence score per anomaly.  
- Generate **UMAP coordinates** for visualization and proximity analysis, replacing t-SNE for scalability and determinism.

**Design Principle:** *Mock the intelligence, not the structure.*  
Even though the model logic is static in the MVP, the transform and dataset behave exactly as they would in production, maintaining full determinism, lineage, and interface compatibility.

---

### Output Schema — `minority_reports_clustered_log (MRCL)`

| Field | Type | Description |
|--------|------|-------------|
| `report_id` | string | Deterministic identifier for the anomaly (`hash(store_id || first_detected_from)`). |
| `version` | integer | Monotonic counter for versioned append history. |
| `store_id` | string | Retailer or store identifier. |
| `sku` | string | Product identifier for the anomaly. |
| `window_start` | timestamp | Start time of the anomaly window. |
| `window_end` | timestamp | End time of the anomaly window (nullable if active). |
| `severity_score` | double | Model-derived measure of anomaly intensity. |
| `impact_estimate` | decimal | Estimated commercial uplift or loss during anomaly period. |
| `feature_vector` | array | Numeric feature vector describing anomaly shape. |
| `cluster_id` | string | Assigned cluster identifier (from cluster library). |
| `cluster_name` | string | Human-readable label for the cluster pattern (e.g., *TikTok Viral*, *Data Error*). |
| `similarity_score` | double | Distance-based similarity measure to cluster centroid. |
| `cluster_match_confidence` | double | Confidence of the assigned cluster match. |
| `tsne_x` | double | 2D x-coordinate for UMAP/t-SNE visualisation. |
| `tsne_y` | double | 2D y-coordinate for UMAP/t-SNE visualisation. |
| `report_status` | string | Lifecycle label (`clustered`, `attributed`, etc.). |
| `written_at` | timestamp | Time of record emission (append timestamp). |
| `run_id` | string | Execution key linking this run to demo context. |
| `attributed_sales_total` | integer | Actual sales during anomaly window. |
| `baseline_sales_total` | integer | Baseline (expected) sales for same period. |

**Properties**
- Append-only and deterministic — replays yield identical outputs.  
- Clustering metadata enriches anomalies for downstream attribution.  
- Schema aligns with MRDL to ensure consistent joins and hydration.

---

**Summary:**  
Clustering links individual anomalies into meaningful patterns, producing deterministic, append-only cluster logs that enrich attribution and cohort analysis while maintaining full replayability.
