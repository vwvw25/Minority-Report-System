# Minority Report System (MRS)

## Overview
The **Minority Report System (MRS)** is a production-grade demo pipeline built in Foundry.  
Its purpose is to identify and classify sales anomalies to prevent distortion in downstream decision systems (e.g. MAB, MMM).  

The project focuses on **system design** — log-driven, stateless, and idempotent — rather than on training live machine learning models.  
All core transforms, datasets, governance hooks, and user interfaces are built as they would be in a real deployment.  
Model “intelligence” is mocked so the pipeline runs end-to-end.

➡️ **Watch the demo video [here](INSERT-LINK)** for a walkthrough of the full system and UI.

---

## Key Principles
- **Log-driven:** every stage writes append-only logs; state is materialised by a hydration transform that the Minority Reports object reads from.  
- **Stateless & idempotent:** anomalies are recomputed deterministically from source data on every run.  
- **Open Cognitive Loops:** new logs are created from scratch as fresh data arrives, allowing the system’s understanding of causes to evolve as new evidence emerges.
- **Deterministic IDs:** stable hashing (`store_id || first_detected_from`) guarantees reproducible joins and lineage.  
- **Enterprise-ready:** full ontology, transforms, governance, audit trail, and HITL interfaces are implemented.

---

## Repository Structure

### Architecture
- [`architecture/strategy.md`](./architecture/strategy.md) — High-level system design and philosophy.  
- [`architecture/resilience-and-dependency-strategy.md`](./architecture/resilience-and-dependency-strategy.md) — Resilience and dependency handling patterns.  
- [`architecture/governance_hooks.md`](./architecture/governance_hooks.md) — Audit, override, and auto-approval governance logic.  
- [`architecture/system_overview.md`](./architecture/system_overview.md) — End-to-end data flow, contracts, and ontology.  
- [`architecture/data_lineage.md`](./architecture/data_lineage.md) — Deterministic IDs, append-only logs, and end-to-end lineage/replay procedures.

### Stages
- [`stages/detection_stage.md`](./stages/detection_stage.md)  
- [`stages/clustering_stage.md`](./stages/clustering_stage.md)  
- [`stages/attribution_stage.md`](./stages/attribution_stage.md)  
- [`stages/cohorting_stage.md`](./stages/cohorting_stage.md)  
- [`stages/finalization_stage.md`](./stages/finalization_stage.md)  
- [`stages/hydration_stage.md`](./stages/hydration_stage.md)  
- [`stages/rereview_stage.md`](./stages/rereview_stage.md)  

### UI
- [`ui/ui_workshop_notes.md`](./ui/ui_workshop_notes.md) — Front-end analyst workflows and system interactions.

---

## Mocking Approach
Models are **mocked strategically**:
- Architecture, ontology, and logs are implemented in full.  
- Outputs are schema-correct and plausible.  
- Retraining loops, drift detection, and governance hooks exist in design.  
- The only missing step is *learning itself* — ensuring a frictionless upgrade path when real ML models are introduced.

---

## Success Criteria
This build demonstrates:
- An end-to-end anomaly detection pipeline with append-only logs and deterministic identities.  
- Plausible, schema-correct outputs at every stage.  
- A clear upgrade path to real ML models without changing downstream contracts.  
- Full auditability, replayability, and HITL readiness.  
- **UI impact:** a visually robust and intuitive Workshop interface that makes system value immediately visible.

---

## Status
This is a **demo build**:  
- Intelligence → mocked  
- Architecture → real  
- Governance, lineage, and UI → real  

The system demonstrates how anomaly detection can be operationalised in enterprise environments where **auditability**, **determinism**, and **replayability** are critical.
