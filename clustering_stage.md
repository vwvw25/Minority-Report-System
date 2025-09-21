# Clustering Stage — Minority Report System  
**File:** `clustering_stage.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
The clustering stage assigns each minority report to a cluster of similar anomalies.  
It is implemented as a **transform** — `feature_vector_cluster_match.py` — which:  
1. Slices the relevant sales timeseries for each report.  
2. Builds a feature vector that describes the anomaly’s shape.  
3. Applies the clustering model (real or mock).  
4. Writes results into the **minority reports clustered log (MRCL)**.  

---

## 2. Inputs  
- **`minority_reports_detected_log` (MRDL)**  
  - Supplies `report_id`, `store_id`, `sku`, `window_start`, `written_at`.  

- **`sales_timeseries_data`**  
  - Provides actuals to build sales slices `[window_start → written_at]`.  

- **`demo_run_config`**  
  - Provides `run_id`.  

---

## 3. Outputs  

### `minority_reports_clustered_log` (MRCL)  
One row per minority report per tick, containing cluster assignment.  
- `report_id`, `store_id`, `sku`  
- `feature_vector` (raw embedding or NULL in mock builds)  
- `cluster_id`, `cluster_name`  
- `similarity_score`, `cluster_match_confidence`  
- `tsne_x`, `tsne_y` (for UI projection)  
- `written_at`, `run_id`  
- `report_status='clustered'`  

---

## 4. Logic  

### 4.1 Slice construction  
- For each MRDL row:  
  - Take `[window_start → written_at]`.  
  - Pull actual sales observations from `sales_timeseries_data`.  

### 4.2 Feature vector  
- Compute descriptive features of anomaly behaviour:  
  - Cumulative uplift vs. baseline.  
  - Growth rate / slope.  
  - Volatility.  
  - Time-of-day distribution.  
- In demo builds, this may be mocked (feature_vector left NULL).  

### 4.3 Clustering model  
- Apply **clustering model** (e.g. k-means, DBSCAN, rule-based).  
- Assign:  
  - `cluster_id` (deterministic).  
  - `cluster_name` (human-readable label).  
- Compute similarity metrics where available:  
  - `similarity_score`  
  - `cluster_match_confidence`.  

### 4.4 Append to log  
- Write row to `minority_reports_clustered_log`.  
- Cluster assignment evolves each tick as more sales data is included in the slice.  

---

## 5. Timing Fields  
- **`window_start`** → carried from MRDL.  
- **`written_at`** → tick timestamp; defines upper bound of slice.  
- Cluster identity is recalculated each tick, but hydrate will coalesce latest values.  

---

## 6. Flow Summary  
1. **MRDL** defines anomaly window.  
2. **Sales slice** pulled from `sales_timeseries_data`.  
3. **Feature vector** built.  
4. **Clustering model** assigns cluster_id + metrics.  
5. **Transform writes MRCL row**.  
6. **Downstream attribution** consumes MRCL.  

---

## 7. Example Narrative  

- MRDL: Store 327 anomaly, window_start=15:30, written_at=17:00.  
- Transform slices sales [15:30 → 17:00].  
- Feature vector built (shows steady uplift).  
- Clustering model matches cluster “Social Media Spike”.  
- MRCL row written:  
  - `report_id=abc123`  
  - `cluster_id=7`  
  - `cluster_name='TikTok Viral'`  
  - `similarity_score=0.92`  
  - `cluster_match_confidence=0.87`  
  - `written_at=17:00`.  

---

## 8. Schemas  

### `minority_reports_clustered_log` (MRCL)  
- `report_id`  
- `store_id`  
- `sku`  
- `feature_vector`  
- `cluster_id`  
- `cluster_name`  
- `similarity_score`  
- `cluster_match_confidence`  
- `tsne_x`  
- `tsne_y`  
- `written_at`  
- `run_id`  
- `report_status`  
