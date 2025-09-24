# Minority Report System (MRS)

## Overview
The Minority Report System (MRS) is a mocked but production-grade demo pipeline built in Foundry.  
Its goal is to **detect, cluster, attribute, and cohort short-term sales anomalies** so that commercial teams can understand root causes and prevent distortion in downstream decision systems (e.g., MAB, MMM).  

The project focuses on **system design** — log-driven, stateless, and idempotent — rather than training live machine learning models.  
All core transforms, datasets, governance hooks, and user interfaces are built as they would be in a deployment; model “intelligence” is mocked so the pipeline runs end-to-end without external ML infrastructure.  

➡️ **To best understand the system, watch the demo video [here](INSERT-LINK).**

---

## Key Principles
- **Log-driven:** every stage writes append-only logs; state is materialised via hydrate views.  
- **Stateless & idempotent:** anomalies are recomputed deterministically from source data on every run.  
- **Open loops:** events can be reopened, reframed, or revised as new evidence arrives.  
- **Deterministic IDs:** stable hashing (`store_id || first_detected_from`) guarantees reproducible joins and lineage.  
- **Enterprise-ready:** complete ontology, transforms, audit trail, and HITL interfaces are present.  

---

## Repository Structure
- [`architecture/Strategy.md`](./architecture/Strategy.md) — High-level system strategy and design philosophy.  
- [`architecture/Resilience-and-dependency-strategy.md`](./architecture/Resilience-and-dependency-strategy.md) — Resilience and dependency handling patterns.  
- [`stages/`](./stages) — Stage-level docs:  
  - [Detection](./stages/detection_stage.md)  
  - [Clustering](./stages/clustering_stage.md)  
  - [Cohorting](./stages/cohorting_stage.md)  
  - [Attribution](./stages/attribution_stage.md)  
  - [Finalization](./stages/finalization_stage.md)  
  - [Hydration](./stages/hydration_stage.md)  
  - [Rereview](./stages/rereview_stage.md)  

---

## Mocking Approach
Models are **mocked strategically**:
- Architecture, ontology, and logs are built in full.  
- Outputs are schema-correct and plausible.  
- Retraining loops, drift detection, and governance hooks are present in design.  
- The only missing piece is the learning step itself.  

This means the upgrade path is clear: real ML models can replace the mocks without altering downstream contracts.

---

## Success Criteria
- End-to-end pipeline runs in Foundry with append-only logs and deterministic IDs.  
- Outputs are realistic and schema-aligned at every stage.  
- Analysts can review anomalies in Workshop via HITL.  
- Architecture is complete, auditable, and replayable.  
- Upgrade to live ML models requires no redesign.  

---

## Status
This is a **demo build**:  
- Intelligence = mocked.  
- Architecture = real.  
- Governance, lineage, and UI = real.  

The system demonstrates how anomaly detection can be operationalised in enterprise settings where **auditability, determinism, and replayability** are critical.
