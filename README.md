# Minority Report System (MRS)

## Overview

Is today’s sales spike being driven by your latest paid social campaign, or by something else entirely?

The Minority Report System (MRS) is a Human-in-the-Loop pipeline that identifies and classifies sales anomalies before they distort downstream decision systems such as Marketing Attribution and Marketing Mix Modelling.

This project focuses on production system architecture rather than model development. The underlying models are mocked, while the pipelines, datasets, ontology, governance, and user interfaces are implemented as they might be in a real deployment.

![Workspace UI for All Reports](Minority%20Reports%20-%20All%20Reports%20View.png)


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

### Architecture

![Minority Report System](MRS%20Data%20Lineage.png)

![MRS Diagram](MRS%20Diagram.png)

---

### UI

![Individual Report Screen](Minority%20Report%20View.png)

![AIP Assistant](AIP%20Assistant.png)

---

The result is an end-to-end system that enables marketing teams to identify, investigate and classify unexpected sales behaviour.
