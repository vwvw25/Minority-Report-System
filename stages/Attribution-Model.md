
# Attribution Specification

## Goal
Define how proposed causes are generated, ranked, and stored for each Minority Report, ensuring consistency, explainability, and downstream protection of MAB/MMM pipelines.

---

## Scope
The attribution spec applies only to the attribution model and its outputs.  
It covers:

- `minority_reports_proposed_attribution_log` — authoritative per-report attribution log.  
- `minority_reports` — hydrated view where the latest attribution fields are surfaced.  
- `minority_events_log` — event-level roll-up with latest cause and confidence.  

It does **not** cover detection, clustering, or cohorting — those are handled in their respective specs.

---

## Attribution Logic

### Input Features
The attribution model consumes:

- Sales series data (baselines & anomalies from detection).  
- Campaign metadata (paid media spend, channel mix, targeting).  
- Contextual signals (search/social mentions, weather, competitor prices/stock, ops logs).  
- System health & feeds (data pipeline checksums, missing records).  

### Process
1. **Candidate generation**  
   - Multiple possible causes are considered for each anomaly.  
   - Categories include: Data Error, Marketing Misattribution, Competitor Stockout, Social/Viral, Operational/Store Issue.  

2. **Scoring & ranking**  
   - Each candidate receives a confidence score ∈ [0,1].  
   - Scores are model-driven but calibrated by feedback loops (HITL adjustments, event validation).  

3. **Selection**  
   - Highest-confidence cause becomes `proposed_cause_category` + `proposed_cause`.  
   - Secondary/tertiary candidates are flattened into fixed columns (`proposed_cause_2_*`, `proposed_cause_3_*`).  
   - Arrays are avoided to preserve one-row-per-report schema compatibility.  

---

## Field Definitions

### In `minority_reports_proposed_attribution_log`
- `report_id` — deterministic ID (store_id + window_start hash).  
- `proposed_cause_category` — high-level bucket (Data Error, Competitor Stockout, etc.).  
- `proposed_cause` — narrative (e.g., “Store Feed Data Error”, “Competitor Mars SKU out of stock”).  
- `confidence_score` — float [0,1].  
- `proposed_cause_2_*`, `proposed_cause_3_*` — optional candidates.  
- `a_written_at` — attribution log timestamp.  

### In `minority_reports`
- Attribution fields are selected from the latest available attribution log (`a_written_at`), unless superseded by analyst overrides in the **edit logs** / **finalization log**.  
- Final fields: `proposed_cause_category`, `proposed_cause`, `confidence_score`, `proposed_cause_2_*`, `proposed_cause_3_*`, `attribution_written_at`.  

### In `minority_events_log`
- Event-level roll-up surfaces the latest attribution across all reports in the event.  
- Confidence is aggregated (avg, max) and stored as `confidence_score`.  
- Latest attribution narrative is preserved for context.  

---

## Field Precedence
- **Analyst overrides** (edit/finalization logs) > **Proposed Attribution log** > **Cluster metadata**.  
- If multiple candidates exist, ordering is by confidence score.  
- If scores tie, deterministic tie-break via lexical sort on category.  

---

## Blind Spots & Assumptions
- Cause categories are finite; novel causes require retraining or taxonomy extension.  
- Attribution relies on external feeds (campaign, competitor); latency or incompleteness reduces confidence.  
- Confidence is model-relative, not absolute probability — calibration is ongoing.  

---

## Validation
- **Synthetic data tests:** inject controlled anomalies with known causes (e.g., TikTok spike) and confirm attribution.  
- **Backtests:** run over historical anomalies with ground truth.  
- **HITL cross-check:** analyst feedback loop comparing proposed vs. confirmed causes.  

---

## Response
- **High confidence (>70%)** — automation may act (auto-confirm, feed downstream systems).  
- **Medium confidence (30–70%)** — analyst review recommended.  
- **Low confidence (<30%)** — attribution flagged as tentative; event remains “open loop” for reframing.  

---

## Audit & Governance
- All attribution decisions are logged with deterministic IDs and timestamps for replayability.  
- Edit/finalization logs preserve analyst overrides separately for lineage.  
- Periodic audits ensure override rates remain within tolerance and model drift is tracked.  

---

## Model Retraining
- Retraining cadence: quarterly or when analyst override rate > X%.  
- Training data: curated from validated anomalies with confirmed causes.  
- Governance: retrained models must pass backtests and synthetic injection validation before promotion.  

---

## System Dependencies & Resilience

Attribution depends directly on:
- **Detection outputs** (baselines, anomalies)  
- **Contextual feeds** (campaign metadata, competitor signals, ops logs)  

### Built-in Resilience Patterns
- **Append-only logs:** Attribution writes only to `minority_reports_proposed_attribution_log`. No destructive updates; late or replayed runs simply append with a new `a_written_at`.  
- **Null-tolerant hydration:** If attribution rows are missing, the hydrate/finalize step coalesces from fallbacks (e.g., cluster or UI edits) so downstream objects still render.  
- **Degraded status flags:** The finalizer attaches `is_degraded=true` and suffixes `report_status` (e.g., `finalised_MRPAL_down`) when attribution is absent but a fallback filled the field.  
- **Graceful degradation:** If attribution fails entirely:
  - `proposed_cause_category = "unknown"`  
  - `proposed_cause = NULL`  
  - `confidence_score = NULL`  
  This allows HITL to proceed without blocking detection/cluster context.  

### Recovery & Replay
- Because attribution logs are keyed by `(report_id, a_written_at)`, replaying attribution after feed recovery is safe and idempotent.  
- Downstream systems (MAB/MMM) consume attribution with confidence-weighting. If confidence is null or degraded, they down-weight or exclude the event until reprocessed.  

### Monitoring & Guardrails
- **Freshness SLI:** max(now() - max(a_written_at)) must remain < 4 hours (p95).  
- **Coverage SLI:** ≥80% of detected reports should have attribution within SLA.  
- **Error dataset:** irrecoverable parsing failures (e.g., malformed campaign JSON) are routed to `_errors` with `report_id`, payload, and error string.  

This ensures attribution is **non-blocking**, observable, and recoverable: the pipeline continues producing usable reports even if attribution feeds lag or fail, and downstream systems adjust automatically until backfill occurs.
