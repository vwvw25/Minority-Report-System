Here’s the rewritten MRS strategy doc in Palantir style, correcting the baseline point (they’re computed internally, not ingested):

⸻

Goal

Detect, cluster, and attribute short-term sales anomalies across retailers so root causes (e.g., social/viral effects, competitor stockouts, data errors, promos) are identified early and downstream decision systems (MAB, MMM) are protected from distortion.

Categorization

We use an internal anomaly taxonomy with three dimensions:
	•	Detection Type — source category of the anomaly (Data Error, Marketing Misattribution, Competitor Stockout, Social/Viral Event, Operational/Store Issue).
	•	Impact Vector — which part of the commercial system is distorted (baseline drift, attribution error, spend leakage, operational disruption).
	•	Confidence Tier — escalation thresholds used by automation/HITL (Low / Medium / High).

Strategy Abstract

High-level flow:
	•	Baseline & detect (internal): baselines are computed inside detection from historical sales (per store/SKU/window). No external baseline feed is ingested. Anomalies (“Minority Reports”) are recomputed on each new data arrival with a deterministic report_id (hash of store_id + window_start), keeping the system stateless and idempotent.
	•	Cluster: group similar reports using feature similarity; attach cluster metadata.
	•	Attribute: propose likely causes (category + narrative) with confidence, passing through secondary/tertiary candidates when available.
	•	Cohort: assign reports to cohorts (report_group_id) to represent a Minority Event — our primary unit of analysis.
	•	Hydrate & aggregate: build a wide, one-row-per-report object for UI, and a one-row-per-event roll-up with timings, totals, and aligned retailer/store fields.
	•	Automation/HITL: apply confidence rules (e.g., weekend mode auto-confirm at >70%) and surface to analysts when confidence is lower or decisions are consequential.
	•	Protection: feed event status/attribution back to MAB/MMM to prevent false credit and baseline corruption.

Technical Context

Design principles
	•	Log-driven & stateless: every stage appends to an authoritative log; no stage mutates a hydrated object. Re-running with new data re-emits the same IDs for the same windows.
	•	Open loops (neuroscience-inspired): closed events can reopen on new evidence; confidence evolves as data accumulates; context can reframe meaning without breaking lineage.
	•	Deterministic identities: report_id = hash(store_id || window_start); cohort/event IDs are derived from stable inputs (e.g., cluster + cause category).
	•	UI constraint: one row per report; multi-candidate attribution encoded as fixed “_2”, “_3” columns (no arrays).

Authoritative logs / objects
	•	minority_reports_detected_log — source of truth for per-report totals & window_start; contains internally computed baseline and attributed vs baseline sales.
	•	minority_reports_clustered_log — cluster metadata (ids, similarity, optional embeddings).
	•	minority_reports_proposed_attribution_log — proposed causes (+ optional 2/3), confidence, evidence.
	•	minority_event_ended_log — authoritative window_end.
	•	minority_reports_cohort_log — per-report cohort assignment (report_group_id, size, confidence).
	•	minority_reports (hydrated view) — wide, latest-per-log merge for UI.
	•	minority_events_log — one row per event (report_group_id) with: report_ids, aligned store_id_1…14 and retailer_1…14 pairs, min/max window, sums/averages, latest cause, confidence, status.

Field precedence (per report)
	•	Keys/meta: prefer Clustered where present; window_start from Detected; window_end from Ended.
	•	Attribution from Proposed Attribution (latest by that log’s timestamp).
	•	Final HITL fields override when present (from UI/Finalised log).
	•	Unified written_at = greatest of contributing sources.

Retailer alignment
	•	Retailer is per store. Event roll-ups emit store_id_n and its corresponding retailer_n from the same position (not a unique set of retailers). This preserves one-to-one mapping needed by marketers.

Blind Spots and Assumptions

Assumptions
	•	Historical data is sufficient to compute stable baselines internally.
	•	Attribution features (campaigns, competitor, ops) are available and timely.
	•	Store→retailer mapping is complete and correct.
	•	Timestamps are reliable for “latest” selection across logs.

Blind spots
	•	Sub-threshold anomalies that don’t exceed detection limits.
	•	Cross-retailer events where signals dilute attribution.
	•	Novel causes not yet modelled.
	•	Upstream latency or outages that delay evidence and confidence evolution.

False Positives

Potential sources
	•	Planned maintenance/test activity misread as Data Error.
	•	Uncatalogued promos/campaigns causing lifts without metadata.
	•	Local exogenous shocks (weather, roadworks) not in enrichment feeds.

Minimisation
	•	Maintain suppression lists and maintenance calendars.
	•	Add backend filters for high-frequency benign patterns.
	•	Use analyst feedback to tune attribution thresholds and narratives.

Validation

True-positive validation (repeatable tests)
	•	Inject synthetic viral signal → expect detection, clustering, and attribution = Social/Viral; event roll-up created; MAB/MMM guarded.
	•	Simulate store-feed corruption → expect attribution = Data Error; auto-confirm if confidence >70% in weekend mode; downstream protection flagged.
	•	Replay competitor stockout series → expect Competitor attribution and spend-shielding where relevant.

Priority

Priority combines cause and confidence:
	•	High — Data Error, System Failure, Competitor Stockout with confidence ≥70% or high estimated impact.
	•	Medium — Marketing Misattribution, Social/Viral with moderate confidence/impact.
	•	Low — Confidence <30% or negligible commercial impact.

Response

When an event surfaces:
	1.	Operate at the event level (report_group_id) — don’t chase stores unless asked.
	2.	Corroborate attribution with ops/marketing/competitor intel.
	3.	Protect models — exclude/reweight affected windows in MAB/MMM.
	4.	Escalate by cause
	•	Data Error → notify feeds/ops; backfill if baselines were impacted.
	•	Misattribution → adjust campaign credit and reallocate spend.
	•	Competitor → flag to commercial; monitor substitution/elasticities.

Additional Resources
	•	Minority Report System build overview (internal)
	•	Attribution model & narratives specification (internal)
	•	MAB/MMM protection playbooks (internal)

⸻
