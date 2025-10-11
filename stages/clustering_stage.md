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

**Summary:**  
Clustering links individual anomalies into meaningful patterns, producing deterministic, append-only cluster logs that enrich attribution and cohort analysis while maintaining full replayability.

**Summary:**  
Clustering links individual anomalies into meaningful patterns, producing deterministic, append-only cluster logs that enrich attribution and cohort analysis while maintaining full replayability.
