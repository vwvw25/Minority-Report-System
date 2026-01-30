# System Resilience & Dependency Strategy  

This document defines the resilience, dependency, and fault-tolerance patterns used in the Minority Report System (MRS).  
The goal is to ensure anomaly detection, clustering, attribution, and hydration continue producing useful, auditable outputs even when upstream inputs are delayed, missing, or degraded.  

---

## 1. Architectural Patterns  

- **Append-only logs + late hydration**  
  All transforms write only to `*_log` datasets. The hydrate step is the only “current state” read.  
  If one log is stale or missing, hydrate still builds rows from what exists (coalescing nulls) so the UI does not fail.  

- **Stage isolation**  
  Each stage reads only its defined inputs and never depends on downstream outputs. Failures are contained within a single dataset.  

- **Run-scoped views**  
  UIs read views filtered by `demo_run_config.run_id`. A run can be “reset” simply by advancing the run_id, even if prior logs are unhealthy.  

- **Graceful degradation**  
  - If clustering is down → detection still runs.  
  - If attribution is down → hydrate still serves detection/cluster context without causes.  

---

## 2. Data Contracts & Null-Tolerant Stitching  

- **Null-safe hydration** — coalesce across sources; missing inputs yield NULLs only in their columns.  
- **Schema evolution guards** — ingest transforms add absent columns as typed NULLs before joins to tolerate partial deploys.  
- **Idempotent keys** — `(report_id, written_at, version, store_id, sku)` ensure replay or re-emission without duplication.  

---

## 3. Failure Containment & Recovery  

- **Retry + backoff** — configure transform retries with exponential backoff for transient sources.  
- **Dead-letter datasets** — irrecoverable rows (e.g., JSON parsing errors) are routed to `_errors` datasets with `report_id`, raw payload, and error.  
- **Circuit breakers** — if a non-critical input (e.g., contextual_events) is unavailable, touch it for lineage but fall back to NULL columns.  
- **Time-boxed joins** — bound time-series joins to prevent runaway shuffles from bad upstream ranges.  

---

## 4. Lateness, Replay, and Exactly-Once  

- **Watermarks by design** — all logs may arrive late; hydrate always picks the latest by `written_at`.  
- **Safe replays** — append-only outputs + stable keys mean detection or attribution can rerun safely.  
- **Dedup on write** — each transform ends with `.dropDuplicates([key columns])` to enforce idempotence.  

---

## 5. Degradation Policy (by Stage)  

- **Detection (critical)**  
  If baseline/classifier unavailable, fall back to a deviation heuristic.  
  Mark as degraded detection and continue.  

- **Clustering (important, non-blocking)**  
  If clustering fails, emit row with `cluster_id = NULL`. Hydrate/UI show “unclustered.”  

- **Attribution (important, non-blocking)**  
  If inputs fail, emit with `proposed_cause = 'unknown'`, `confidence_score = NULL`. HITL can still proceed.  

- **Finalisation (human)**  
  If edits stream is down, pipeline continues. Hydrate shows latest machine proposal.  
  If either `user_edits_log` or `user_minority_events_edits_log` is temporarily unavailable, pipeline continues with available inputs and marks MRFL as degraded.  

---

## 6. Coding Patterns (Keep)  

- **Optional input touch** without dependency:  

      _ = contextual.dataframe().select(F.lit(1)).limit(0).count()

- **Typed null scaffolding:**  

      df = df.withColumn("on_promotion", F.lit(None).cast("boolean"))

- **Ambiguity-proof joins** — alias inputs before join, select with explicit aliases only.  

- **Tolerant timestamp parsing:**  

      c = F.regexp_replace(F.trim(col), r"Z$", "+00:00")
      F.coalesce(
          F.to_timestamp(c, "yyyy-MM-dd'T'HH:mm:ss.SSSXXX"),
          F.to_timestamp(c, "yyyy-MM-dd'T'HH:mm:ssXXX"),
          F.to_timestamp(c, "yyyy-MM-dd HH:mm:ss")
      )

---

## 7. Finalisation & Recovery  

### Data Consolidation Resilience  

Both the **Finalisation** and **Hydration** stages consolidate multiple sources into a unified, authoritative record.  
While **Finalisation** produces a static audit log for rereview, and **Hydration** produces the live dataset surfaced to analysts, both rely on the same resilience framework.  

**1. Explicit Precedence Chains**  
Field groups are merged deterministically using `coalesce` in a fixed order  
(e.g., detection → clustering → attribution → cohort → finalisation → UI edits).  
If no source provides a value, it remains `NULL` — never guessed or silently imputed.  

**2. Source Provenance**  
For every field group, a `source_used_*` column records which log provided the value  
(`detected`, `clustered`, `attribution`, `cohort`, `finalised`, or `ui`).  
This ensures end-to-end auditability of every field.  

**3. Degraded Flags**  
If a preferred upstream source is missing (e.g., no attribution log or cohort record), the consolidated row is tagged:  

`is_degraded = true`  
`report_status = <stage>_degraded_<reason>`  

Short encoded reasons indicate missing inputs:  
- `MRCL_down` → clustering or cohort missing  
- `MRPAL_down` → attribution missing  
- `MEEL_down` → ended log missing  

**4. Observability**  
Provenance and degradation fields (`source_used_*`, `degraded_reasons`) enable fast diagnosis of reduced confidence — whether caused by clustering, attribution, cohorting, or ingestion gaps.  

**5. Recovery Procedures**  
- **External feed down** → pipeline continues; attribution degrades; rerun attribution after recovery.  
- **Cluster model error** → route to mock/bypass clustering; hydrated data still available.  
- **Schema drift** → missing columns filled as typed NULLs; pipeline remains non-failing.  
- **Edits log unavailable** → derive `report_id` from existing identifiers; emit minimal valid record.  

**Guarantee**  
Both stages always emit a valid record, even when upstream data is incomplete.  
This ensures uninterrupted operation and transparent confidence signalling across the entire Minority Report System.  

---

## 8. Bulkheads & Concurrency  

- Isolate compute pools (detection vs refresh).  
- Cap per-run worklists; partition processing by `run_id`.  

---

## 9. Chaos & Validation  

- **Dry-run mode** — transforms accept flag to limit to `limit(1000)`.  
- **Fault injection** — periodically drop a non-critical input in staging; confirm degrade path and SLIs.  
- **Backfill rehearsal** — replay old `run_id` end-to-end; confirm idempotency and coverage.  

---

## 10. Future Enhancement: Event-Driven Recall  

The MVP currently reprocesses reports on a fixed cadence via `review_reports_scheduler.py`.  
A future enhancement replaces this batch pattern with **event-driven recall triggers**, creating truly cognitive “open loops”:

**Recall Triggers**  
1. New pattern discovered → recall similar past anomalies.  
2. Confidence drift detected → recall prior reports in same cluster.  
3. Model updates or new data sources → recall affected historical records.  
4. Contradictory evidence → re-evaluate past attributions automatically.  

**Example**  
> When a “TikTok Viral” cluster splits into two new sub-clusters, all historical reports in that cluster are re-evaluated.

**Benefits**  
- Continuous self-correction of understanding.  
- Resource-efficient: only relevant history is reprocessed.  
- Mimics associative recall in human cognition.  

---

## 11. Monitoring, SLOs, and Guardrails  

### SLIs  
- Freshness per log: `max(now() - max(written_at))`.  
- Coverage: % of detected reports that have cluster/attribution within SLA.  
- Error rate: % of rows into `_errors` datasets.  

### SLOs (examples)  
- Detection freshness < 30 min (p95).  
- Cluster + Attribution freshness < 4 hrs (p95).  
- < 1% rows into `_errors`.  

### Health Views  
Dashboards with last `written_at`, row counts by status, and error counts per log.  

### Alerting  
Trigger alerts when freshness breaches or coverage drops (e.g., attribution coverage < 80% for active reports).  

---

## 12. Operational Example — Data Correction Event (Retail Feed)  

This example illustrates how the Minority Report System (MRS) handles data corrections from an upstream retail source — demonstrating its resilience, replayability, and governance guarantees.  

### 12.1 Scenario  
Over a weekend, sales reported from Waitrose stores show an unexpected uplift.  
The **Detection Stage** flags these anomalies as a Minority Event, attributing them to a likely **Data Error** with low-to-medium confidence.  

Lewis, an analyst, verifies the anomalies in the Workshop UI and shares the findings internally.  
Waitrose later confirms that the uplift was caused by a till malfunction that duplicated several days of sales data.  
The retailer reissues corrected data for the affected period.  

### 12.2 Upstream Behaviour  
Upon receiving corrected transactional data, Waitrose resends the feed to the ingestion system (`unified_sales_data`).  
This triggers automatic regeneration of `sales_timeseries_data` and initiates a full downstream recomputation through the MRS pipeline.  

No manual resets are required — the system detects that new data has arrived and automatically replays detection, clustering, attribution, and cohorting stages for the affected stores and time windows.  

### 12.3 System Response  
1. **Detection Stage** re-runs using corrected series.  
   - The previously inflated sales are now consistent with baseline expectations.  
   - The anomaly classifier no longer emits those spikes as valid Minority Reports.  
2. **Cohorting and Attribution** auto-adjust.  
   - The former event (`report_group_id`) remains in logs for audit purposes.  
   - A finalised entry is written marking the event `event_status='ended'` and `resolution_reason='data correction'`.  
3. **Hydration** deterministically surfaces the corrected state.  
   - The old anomaly rows remain in the historical logs (`MRDL`, `MRCH`, `MELOG`).  
   - The active hydrated dataset (`minority_reports`) now reflects corrected totals and closed events.  
   - Downstream systems (MMM/MAB) read only the latest valid data, ensuring no false attribution.  

### 12.4 Governance and Traceability  
- **Audit trail:** The original anomaly, its resolution, and all intermediate states remain in immutable logs.  
- **Status propagation:** The affected reports carry `report_status='superseded'` and `is_degraded=true` until reprocessed.  
- **Transparency:** The system automatically annotates the resolution in `minority_events`, preserving full lineage.  
- **Replayability:** Any historical state — before or after correction — can be reconstructed by rehydrating logs at a given `written_at` timestamp.  

### 12.5 Business Outcome  
- The Data Error Minority Event is automatically closed without manual intervention.  
- All downstream analytical models consume the corrected dataset.  
- Historical accuracy and lineage are preserved — enabling audit, root-cause analysis, and confidence in autonomous operation.  

**Summary:**  
This scenario demonstrates the MRS’s capacity for **self-healing and deterministic correction**.  
Through stateless recomputation, append-only logs, and governed hydration, it ensures that retail data corrections propagate seamlessly, preserving both analytical integrity and full audit traceability.  

---

## Summary  

The system is designed for graceful degradation, auditability, and safe replays.  
Failures in one stage do not cascade, and the UI always remains responsive with the best available context.  
The **finalise log** enforces explicit precedence and provenance, ensuring outputs remain trustworthy while clearly flagging degraded conditions.  
