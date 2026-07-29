# Minority Report System (MRS)

## Overview

Is today’s sales spike being driven by your latest paid social campaign, or by something else entirely?

The Minority Report System (MRS) is a Human-in-the-Loop pipeline that identifies and classifies sales anomalies before they distort downstream decision systems such as Marketing Attribution and Marketing Mix Modelling.

This project focuses on production system architecture rather than model development. The underlying models are mocked, while the pipelines, datasets, ontology, governance, and user interfaces are implemented as they might be in a real deployment.


➡️ **Watch the demo video [here](https://www.youtube.com/watch?v=1CyLjJoZHFw)** for a walkthrough of the full system and UI.

➡️ **Read my portfolio piece** about the MRS [here.](https://app.notion.com/p/Designing-the-Minority-Report-System-Building-Trustworthy-Enterprise-Knowledge-for-Marketing-AI-3ac1d53cfded8051bd59ca5d3cd7492e)

---

## Key Principles
- **Log-driven:** every stage writes append-only logs; state is materialised by a hydration transform that the Minority Reports object reads from.  
- **Stateless & idempotent:** anomalies are recomputed deterministically from source data on every run.  
- **Open Cognitive Loops:** new logs are created from scratch as fresh data arrives, allowing the system’s understanding of causes to evolve as new evidence emerges.
- **Deterministic IDs:** stable hashing (`store_id || first_detected_from`) guarantees reproducible joins and lineage.  
- **Enterprise-ready:** full ontology, transforms, governance, audit trail, and HITL interfaces are implemented.

---

## Repository Structure

The repository is organised around a set of Design Documents. Together they capture the reasoning behind the system’s design.

### Architecture

![Minority Report System](MRS%20Data%20Lineage.png)

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

The result is an end-to-end system that enables marketing teams to identify, investigate and classify unexpected sales behaviour.
