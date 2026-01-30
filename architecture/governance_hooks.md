# Governance Hooks — Minority Report System  
**File:** `governance_hooks.md`  
**Last Updated:** 2025-09-21  

---

## 1. Purpose  
Governance in the Minority Report System (MRS) is built **into the architecture**, not bolted on.  
Every transform produces immutable, append-only logs with deterministic identifiers, enabling full lineage, audit, and replay across the entire anomaly detection lifecycle.  

This document summarises the key **governance hooks** that make the system transparent, compliant, and trustworthy in enterprise environments.

---

## 2. Design Principles  
- **Append-only history:** no row is ever updated or deleted — all changes are additive.  
- **Deterministic lineage:** identical inputs always produce identical outputs.  
- **Reproducibility:** the entire system can be replayed from raw logs to reproduce any past state.  
- **Human accountability:** every user edit and model proposal is logged separately, never overwriting prior entries.  
- **Traceability:** every record links upstream and downstream via stable identifiers.  

---

## 3. Governance Fields  

| Field | Description | Example Source |
|--------|--------------|----------------|
| `report_id` | Deterministic hash of store_id & first_detected_from. Stable key across all logs. | All stages |
| `event_id` | Derived from `report_id` or cohort group. Used for aggregating reports into events. | Cohorting |
| `run_id` | Execution key from `demo_run_config`. Ensures reproducibility per pipeline run. | All transforms |
| `written_at` | Timestamp when row was appended. Used for freshness and version tracking. | All transforms |
| `finalized_at` | Timestamp of analyst action. Differentiates machine vs. human state. | Finalisation |
| `annotation_id` | Unique ID for analyst edits (UUID or `actionRid`). | Finalisation |
| `source_used_*` | Hydration provenance columns identifying which log supplied each field. | Hydration |
| `is_degraded` | Boolean flag if any upstream source missing or delayed. | Hydration |
| `report_status` | Categorical lifecycle label (`detected`, `clustered`, `proposed`, `finalised`, `ended`). | All stages |

These fields collectively provide full **temporal and procedural lineage** — allowing any value in the UI to be traced back to its source row and transform.

---

## 4. Audit Trail Coverage  

| Stage | Audit Focus | Mechanism |
|--------|--------------|-----------|
| Detection | Who/when anomalies were first raised | Append-only MRDL + deterministic `report_id` |
| Clustering | Model-derived grouping lineage | MRCL with versioned `written_at` and `cluster_id` |
| Attribution | Cause proposals and confidence scores | MRPAL with model version metadata |
| Finalisation | Human overrides and annotation trail | MRFL + `annotation_id`, `finalized_at` |
| Hydration | Source provenance and precedence | `source_used_*` + `is_degraded` flags |
| Rereview | Improvement cycles | `*_rereview_log` parallel histories |

Every value seen in the UI or exported dataset can be traced through these logs without ambiguity.

---

## 5. Governance Enforcement  

- **Data immutability:** enforced by Foundry dataset lineage (no destructive writes).  
- **Schema versioning:** controlled via transform contracts — new fields only appended, never removed.  
- **Access control:** governed at dataset level (RBAC) — write access restricted per stage.  
- **Human-in-the-loop governance:**  
  - Analyst actions logged in `ui_edits_log` and 'edit_minority_event_object_log'.  
  - `annotation_id` binds user actions to data provenance.  
- **Operational validation:** health views and SLIs monitor coverage, freshness, and error rates to detect governance drift.  

---

## 5.1 Automated Decision Controls  

Auto-approval is treated as a **governed automation feature**, not a default behaviour.  
It can be enabled once model reliability and governance thresholds are met.  

**Design principles:**  
- Initially disabled until sufficient validation evidence exists.  
- Activation requires explicit approval from governance owners.  
- Applies only to high-confidence reports (typically ≥ 70%) in pre-defined conditions such as weekend operations.  
- Every auto-approval is written to `ui_edits_log` or 'edit_minority_event_object_log' with `user_role='system'`, maintaining the same audit trail as human actions.  
- Auto-approvals appear in the UI with a governance badge and can be manually reversed.  

See [`ui_workshop_notes.md`](../ui_workshop_notes.md) for the front-end workflow illustrating this behaviour.

---

## 6. Replay & Audit Scenarios  

**Scenario A — Rebuild a report from history**  
1. Filter all logs by `report_id='abc123'`.  
2. Sort by `written_at`.  
3. Replay sequentially (MRDL → MRCL → MRPAL → MRFL → Hydration).  
→ Produces identical final state, confirming system determinism.

**Scenario B — Audit analyst edit**

**Per-report audit**
1. Query `annotation_id` from `minority_reports_finalised_log` (MRFL).  
2. Join to `ui_edits_log` for analyst identity, timestamp, and edit context.  
3. Compare proposed vs. final causes to trace the decision change.  
→ Provides full lineage of per-report human intervention.

**Per-event audit**
1. Query `event_id` from `minority_events`.  
2. Join to `edit_minority_event_object_log` for event-level modifications (e.g., metadata, evidence, or classification updates).  
3. Compare event metadata before and after edit to validate reasoning and governance traceability.  
→ Provides a parallel audit trail for higher-level event object changes.

---

## 7. Compliance Alignment  
The design satisfies key enterprise and regulatory principles:

| Principle | Implementation |
|------------|----------------|
| **Auditability** | Every mutation logged with timestamp and author. |
| **Data lineage** | Deterministic IDs and append-only architecture. |
| **Reproducibility** | Re-running transforms yields identical outputs. |
| **Data minimisation** | Only essential operational fields stored; no PII. |
| **Accountability** | All HITL interactions explicitly logged with user identifiers. |

---

## 8. Reopen & Audit Protocol  

When new evidence emerges after closure, the MRS allows structured reopening of reports while maintaining full auditability and regulatory compliance.

### Required Reopen Record

    {
      "minority_report_id": "uuid",
      "reopened_by": "user_id",
      "reopened_timestamp": "timestamp",
      "reopened_reason": "enum[new_evidence, attribution_error, regulatory_request]",
      "reopened_notes": "string (≥50 chars)"
    }

### Processing Paths
- **Manual Update Path** — Human directly edits or overrides attribution; system appends new record to `minority_reports_proposed_attribution_log` and flags the edit.  
- **System Reprocessing Path** — Triggers full attribution re-run using current data and model context; new proposal logged and routed to HITL for approval.  

### Audit Trail Integration
Each reopen action generates:  
- New `hitl_annotations_log` entry (`reopened = true`)  
- New `minority_reports_proposed_attribution_log` record  
- Flags for traceability:  
  - `manual_post_review_edit = true`  
  - `attribution_override_post_review = true`  
  - `system_reprocessing_post_closure = true`  

### Governance Guarantees  
- Every post-closure modification is fully justified and timestamped.  
- Original state is preserved in immutable logs.  
- All reopen actions are traceable by `annotation_id` and `report_id`.  
- Enables compliant evolution of understanding without compromising historical accuracy.  

---

## 9. Summary  
Governance in the MRS is intrinsic — achieved through **immutable logs, deterministic IDs, and explicit human annotation capture**.  
The result is a fully replayable, audit-ready system where every anomaly, cluster, and decision is traceable end-to-end, meeting the standards expected of enterprise data governance.
