# Clustering Stage — Minority Report System  
**File:** `clustering_stage.md`  

---

## 1. Purpose  
The clustering stage groups similar anomalies so analysts can see emerging patterns (e.g., viral spikes vs. data errors).  
Implemented in `feature_vector_cluster_match.py`, it:  
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
- **Slice Construction:** extract actual sales `[window_start → written_at]` per anomaly.  
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

The mocked clustering model preserves full production fidelity while remaining lightweight for the demo.

**Feature Vector Generation (Real)**  
- Extracts fixed-length anomaly windows.  
- Normalizes series shape (scale-independent).  
- Computes slope, volatility, symmetry, kurtosis.  

**Cluster Matching (Real)**  
- Compares generated vectors to seeded centroids in a “cluster library.”  
- Calculates similarity via cosine or Euclidean distance.  
- Returns closest cluster + similarity score.  

**t-SNE Visualization (Mocked)**  
- Pre-computed offline once; coordinates stored for UI display.  

**Design Principle:** *Mock the intelligence, not the structure.*  
All datasets, transforms, and lineage behave exactly as they would in production.

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
