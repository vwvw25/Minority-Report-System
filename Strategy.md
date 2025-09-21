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
   - Anomalies (“Minority Reports”) are recomputed on each new data arrival with a deterministic `report_id` (hash of store_id + window_start), keeping the system stateless and idempotent.  

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

### Authoritative Logs
- `minority_reports_detected_log` — per-report totals & window_start; contains internally computed baseline and attributed vs baseline sales.  
- `minority_reports_clustered_log` — cluster metadata.  
- `minority_reports_proposed_attribution_log` — proposed causes (+ optional 2/3), confidence, evidence.  
- `minority_event_ended_log` — authoritative window_end.  
- `minority_reports_cohort_log` — per-report cohort assignment (report_group_id, size, confidence).  
- `minority_reports` (hydrated view) — wide, latest-per-log merge for UI.  
- `minority_events_log` — one row per event with: report_ids, aligned `store_id_1…14` and `retailer_1…14` pairs, min/max window, sums/averages, latest cause, confidence, status.  

### Field Precedence (per report)
- Keys/meta: prefer Clustered; `window_start` from Detected; `window_end` from Ended.  
- Attribution: Proposed Attribution (latest by `a_written_at`).  
- Analyst overrides: edit/finalisation logs supersede machine attribution.  
- Unified `written_at` = greatest of contributing sources.  

### Retailer Alignment
- Retailer is per store. Event roll-ups emit `store_id_n` and its corresponding `retailer_n` from the same position (not a unique set of retailers).  

---

## Orchestration & Scheduling
- Tasks run as append-only transforms; failures are contained per stage.  
- Retry + backoff policies are applied (see [Resilience Spec](./resilience.md)).  
- Replays are safe: stable keys (`report_id`, `report_group_id`) guarantee idempotence.  
- Late arrivals simply replace earlier snapshots on the next run.  

---

## System Resilience & Dependency Strategy
- Detailed patterns (degraded modes, retries, _errors datasets, coalesce precedence chains) are in [Resilience Spec](./resilience.md).  
- Summary here:  
  - Attribution failure → emit `unknown` cause with NULL confidence.  
  - Clustering failure → emit unclustered rows but keep detection output alive.  
  - Hydrate is null-tolerant: missing sources yield NULL columns rather than pipeline failure.  
  - Finaliser adds `is_degraded` and suffixed `report_status` to make degradation explicit.  

---

## Observability & Monitoring
- **SLIs:**  
  - Freshness: `max(now() - max(written_at))` per log.  
  - Coverage: % of detected reports with attribution/cluster within SLA.  
  - Error rate: % rows in `_errors`.  
- **SLOs:**  
  - Detection freshness < 30 min (p95).  
  - Attribution freshness < 4 hrs (p95).  
  - <1% rows into `_errors`.  
- Dashboards: small views per log for freshness, row counts, status distribution.  
- Alerts: trigger on freshness breaches or coverage drops.  

---

## Versioning & Change Management
- Schema evolution: add absent columns as typed NULLs before joins to avoid crashes.  
- Backward compatibility: downstream always coalesces across sources, so old fields degrade to NULL.  
- Transform runs end with `.dropDuplicates(keys)` to ensure idempotence.  
- Explicit governance: new fields require documentation and validation before promotion.  

---

## Scaling & Performance Optimization
- Partition by `run_id` and store/SKU to keep joins bounded.  
- Cap per-run worklists; avoid giant shuffles.  
- Use time-boxed joins to prevent runaway queries.  
- Dry-run mode (`limit(1000)`) supports quick iteration and chaos testing.  

---

## Audit & Governance

### Audit Principles
- **Deterministic lineage** — all objects use append-only logs with deterministic IDs (`report_id`, `report_group_id`) and `written_at` timestamps. This guarantees replayability and makes historical reconstruction trivial.  
- **Separation of machine vs human input** — machine attribution and clustering live in their own logs (`*_proposed_*`), while analyst overrides flow through `edit_logs` and `minority_reports_finalisation_log`. Provenance is never lost.  
- **Immutable history** — overrides do not overwrite machine outputs; they supplement them. Both views remain queryable for audit.  

### Access Control & Security
- Analyst overrides are restricted to authorised users; all edits are logged with user, timestamp, and reason.  
- Sensitive feeds (campaign data, competitor intel) require least-privilege access controls.  
- Governance enforces that schema changes, overrides, and model retrains are reviewed before promotion.  

### Audit Surfaces
- **Per-report audit trail** — any `report_id` can be traced across detection, clustering, attribution, edits, and finalisation.  
- **Per-event audit trail** — any `report_group_id` can be reconstructed from constituent reports, with a clear timeline of proposals, overrides, and final status.  
- **Override register** — percentage and frequency of analyst overrides are logged and trended, allowing governance thresholds (e.g., “<15% override rate”) to be enforced.  

### Governance Processes
- **Periodic audit reviews** — dashboards surface override rates, stale cohorts, and attribution drift. Reviews ensure analyst intervention is aligned with governance policy.  
- **Model drift checks** — attribution models are retrained or recalibrated if overrides consistently exceed thresholds. Retraining cadence and acceptance criteria are documented in the attribution spec.  
- **Change management** — schema evolution and log changes follow versioned deploys with backward-compatible scaffolding, ensuring no audit gaps.  

### Compliance & Assurance
- **Replayability** — given any date/time, the system can be re-run or queried to reproduce the state of attribution and cohorts at that moment.  
- **Audit resilience** — even in degraded modes (e.g., attribution feeds down), NULLs and degraded status flags are emitted explicitly, ensuring there are no silent drops.  
- **External review** — logs are structured to support external audit/compliance queries without custom rebuilds.  

---

## Blind Spots & Assumptions
- Historical data sufficient for baselines.  
- Attribution features timely and complete.  
- Store→retailer mapping accurate.  
- Novel causes require taxonomy updates and model retraining.  

---

## False Positives
- Maintenance/test activity misread as Data Error.  
- Uncatalogued promos or local exogenous shocks.  
- Mitigated via suppression lists, backend filters, analyst feedback loops.  

---

## Validation
- Inject synthetic viral signal → expect detection & attribution = Social/Viral.  
- Simulate store-feed corruption → expect Data Error attribution, auto-confirm if >70% confidence.  
- Replay competitor stockout → expect Competitor attribution.  

---

## Priority
- **High** — Data Error, System Failure, Competitor Stockout with confidence ≥70% or high impact.  
- **Medium** — Misattribution, Social/Viral with moderate confidence/impact.  
- **Low** — <30% confidence or negligible impact.  

---

## Response
When an event surfaces:  
1. Operate at event level (`report_group_id`).  
2. Corroborate attribution with ops/marketing/competitor intel.  
3. Protect models — exclude/reweight affected windows in MAB/MMM.  
4. Escalate by cause:  
   - **Data Error** → notify feeds/ops; backfill baselines.  
   - **Misattribution** → adjust campaign credit and reallocate spend.  
   - **Competitor** → notify commercial; monitor substitution/elasticities.

---

## Case Studies & Applied Design Patterns

To illustrate how the Minority Report System (MRS) behaves in practice, this section walks through representative narratives used in the demo (12 August 2025). Each case shows how detection, clustering, attribution, and HITL combine to surface, reframe, and resolve anomalies.  

---

### 1. TikTok Viral Challenge (“Jupyter Challenge”)  
- **Event:** ME-a3df75279922 (report_group_id: MR-UK-001)  
- **Scope:** 10 London stores; SKU = Jupyter Bar Duo  
- **Pattern:** Initial misattribution → corrected by refresh + HITL  
- **Trajectory:**  
  - Initial attribution: Campaign Spend (confidence 0.55)  
  - Final attribution: TikTok Viral (refreshed, confidence 0.72)  
- **Evidence:** System picked up a hashtag via social listening; analyst added TikTok link.  
- **Outcome:** Finalised after HITL approval; model misattribution corrected before it could distort MAB.  

---

### 2. False Positive — Till Fault  
- **Event:** MR-UK-002  
- **Scope:** 15 Waitrose SE stores  
- **Pattern:** Spurious spike correctly rejected  
- **Trajectory:**  
  - System attribution confidence <0.3 → weak signal.  
  - Ops email (HITL input) confirmed till fault.  
- **Outcome:** HITL closed event as Data Error false positive.  

---

### 3. Competitor Stock-out  
- **Event:** MR-UK-003  
- **Scope:** 10 Birmingham stores  
- **Pattern:** Sustained lift from competitor absence  
- **Trajectory:**  
  - Attribution: Competitor Stock-Out (confidence ~0.81)  
  - Refresh across 3 days reinforced stability.  
- **Outcome:** Approved as competitor-driven; protected attribution model and spend-shielding logic.  

---

### 4. National TV Recipe Feature  
- **Event:** MR-UK-004  
- **Scope:** 8 UK cities  
- **Pattern:** Novel cause not recognised by model; surfaced via HITL  
- **Trajectory:**  
  - System: low-confidence, unclustered (TikTok alt at 0.28)  
  - Analyst added TV clip; attribution updated.  
- **Outcome:** Finalised with HITL correction; demonstrates “open loop” principle.  

---

### 5. Major Podcast Mention  
- **Event:** MR-UK-005  
- **Scope:** 7 UK cities  
- **Pattern:** System miss, recovered by HITL  
- **Trajectory:**  
  - System attribution confidence <0.3; no cause proposed.  
  - Analyst logged podcast transcript snippet.  
- **Outcome:** Approved with human cause; validated HITL safety net.  

---

### 6. London Waterloo — Train Disruption  
- **Event:** MR-UK-006  
- **Scope:** 4 commuter stores around Waterloo  
- **Pattern:** Operational exogenous shock  
- **Trajectory:**  
  - Attribution: Travel Disruption (confidence 0.74)  
  - Supported by Transport API delay alert.  
- **Outcome:** Finalised with minimal HITL input.  

---

### 7. Deliveroo Free Bar Campaign  
- **Event:** ME-58d24bee24c6 (report_group_id: MR-UK-007)  
- **Scope:** UK-wide  
- **Pattern:** Strong system alignment with external metadata  
- **Trajectory:**  
  - Attribution: Deliveroo Promo (confidence 0.92)  
  - Evidence: Campaign log matched uplift.  
- **Outcome:** High-confidence, auto-approved; textbook example of automation working cleanly.  

---

### 8. Instagram Reels Spike  
- **Event:** MR-UK-008  
- **Scope:** Norwich stores  
- **Pattern:** Misattribution corrected by HITL  
- **Trajectory:**  
  - Initial attribution: Facebook campaign spend (confidence 0.58).  
  - Analyst input corrected to Reels viral.  
- **Outcome:** Event finalised after correction; highlights need for human-in-loop in boundary cases.  

---

## Lessons Across Case Studies
- **Model + HITL complementarity:** Automation handles clear cases (Deliveroo, Competitor Stock-out), while HITL corrects ambiguous ones (TikTok, Reels, TV, Podcast).  
- **Degraded modes visible:** False positives and missed cases are surfaced with low confidence, preventing silent corruption of models.  
- **Contextual enrichment matters:** External feeds (social, transport, campaign logs) raise confidence and reduce reliance on HITL.  
- **Replayability:** Each narrative can be reconstructed via deterministic report_group_id and logs, ensuring auditability.  
---

## Additional Resources
- [Resilience Spec](./resilience.md)  
- Attribution model & narratives specification (internal)  
- MAB/MMM protection playbooks (internal)  
