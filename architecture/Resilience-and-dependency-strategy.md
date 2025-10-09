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

- **Null-safe hydration** — coalesce across sources; missing inputs yield NULLs in only their columns.  
- **Schema evolution guards** — ingest transforms add absent columns as typed NULLs before joins to tolerate partial deploys.  
- **Idempotent keys** — `(report_id, written_at, version, store, sku)` ensure replay or re-emission without duplication.  

---

## 3. Failure Containment & Recovery

- **Retry + backoff** — configure transform retries with exponential backoff for transient sources.  
- **Dead-letter datasets** — irrecoverable rows (e.g., JSON parsing errors) are routed to a small `_errors` dataset with `report_id`, raw payload, and error.  
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

### Finalise Log Resilience

The **finalise log** consolidates all sources into the authoritative “current state” row per report.  
Resilience here is critical since this version is exposed to the UI and analysts.  

**Key patterns:**  

- **Explicit precedence chains**  
  Each field group is filled with `coalesce` in a fixed order (e.g., proposed cause: cohort → attribution → UI). If no source provides the field, it remains NULL.  

- **Source provenance**  
  For every field group, a `source_used_*` column records which log actually provided the value (`cohort`, `attribution`, `clustered`, `detected`, `ui`, or `none`).  

- **Degraded flag**  
  If a preferred source was missing (e.g., no cohort row, attribution log down), the row is marked:  
  - `is_degraded = true`  
  - `report_status = finalised_<reason>` (e.g., `finalised_MRCL_down_MEEL_down`).  

- **Reasons**  
  Reasons are concatenated short labels per missing primary, e.g.:  
  - `MRCL_down` → cohort missing  
  - `MRPAL_down` → attribution missing  
  - `MEEL_down` → ended log missing  

- **Observability**  
  These provenance fields (`source_used_*`, `degraded_reasons`) may be kept internal but allow ops to quickly diagnose whether degradation was due to cohorts, attribution, clustering, etc.  

This guarantees that **finalised rows always exist** while clearly signalling when confidence is reduced.

### Recovery Procedures

- **External feed down** → pipeline continues, attribution degrades; backfill and rerun attribution when feed recovers.  
- **Cluster model error** → route to mock/bypass clustering; hydrate still shows detection rows.  
- **Hydrate schema check fails** → upstream adds missing columns as typed NULL; hydrate tolerates.  
- **Edits log schema drift** → compute `report_id` from available fields; emit empty but non-failing output.  

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

## 10. Monitoring, SLOs, and Guardrails

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

## Summary

The system is designed for graceful degradation, auditability, and safe replays.  
Failures in one stage do not cascade, and the UI always remains responsive with the best available context.  
The **finalise log** enforces explicit precedence and provenance, ensuring outputs remain trustworthy while clearly flagging degraded conditions.
