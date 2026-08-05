# System Overview — Minority Report System

## Purpose

The Minority Report System (MRS) identifies unusual short-term sales behaviour, proposes likely causes, and maintains the organisation's current operational understanding of those events.

It sits between raw sales data and downstream optimisation systems such as Marketing Mix Modelling (MMM), Trade Promotion Optimisation (TPO), and Multi-Armed Bandits (MAB), helping prevent external events—such as viral social media, competitor stockouts, weather, or data errors—from being incorrectly attributed to paid marketing activity.

---

## High-Level Architecture

```text
unified_sales_data → sales_timeseries_data
                           │
                           ▼

Detection → Clustering → Attribution → Cohorting
    │            │             │            │
    └────────────┴─────────────┴────────────┘
                         │
                         ▼
                 Append-only logs
                         │
            continuously read by
                         │
                         ▼
                  Hydration layer
                         │
                         ▼
         Minority Report / Minority Event objects
                  │                   ▲
                  ▼                   │
             AIP Assistant            ▼
                                 Workshop application/Human review
                                    
                   
```

---

## Processing Pipeline

The processing pipeline consists of four analytical stages.

### Detection

Detection identifies statistically significant deviations from expected sales behaviour and creates a **Minority Report** for each anomaly.

### Clustering

Clustering compares the behavioural signature of each anomaly against historical anomalies to identify similar sales patterns.

Behavioural similarity provides evidence about the nature of an anomaly without determining its cause.

### Attribution

Attribution combines behavioural evidence with contextual information—including campaign calendars, promotions, weather, operational incidents and competitor intelligence—to propose likely causes and confidence scores.

### Cohorting

Cohorting groups related Minority Reports into higher-level **Minority Events**, representing a single underlying business phenomenon affecting multiple reports.

---

## Hydration

Each analytical stage writes only to its own authoritative append-only log.

The Hydration layer continuously reconstructs the latest state of every Minority Report and Minority Event by combining information from those logs using deterministic precedence rules.

Rather than interacting directly with pipeline datasets, both the Workshop application and the AIP Assistant operate on these hydrated ontology objects.

This means every stage progressively enriches the same operational objects as new evidence becomes available.

---

## Human Review

Analysts review Minority Reports and Minority Events through the Workshop application.

They can:

- approve or reject proposed causes;
- edit classifications;
- attach supporting evidence;
- request further investigation.

These actions contribute additional evidence to the system and become incorporated into the hydrated ontology objects while preserving a complete audit history.

---

## Ontology Objects

### Minority Report

Represents a single detected sales anomaly.

A Minority Report evolves throughout the pipeline as additional behavioural, contextual and human evidence becomes available.

### Minority Event

Represents the underlying phenomenon believed to connect one or more Minority Reports.

Examples include:

- Viral social media events
- Competitor stockouts
- Marketing campaigns
- Weather events
- Retail data errors

---

## Architectural Principles

The system is designed around several core principles:

- **Log-driven** — every processing stage writes only to its own authoritative log.
- **Append-only** — historical records are never modified or deleted.
- **Stateless** — the current system state is reconstructed through hydration rather than stored directly.
- **Deterministic** — identical inputs produce identical identities and outputs.
- **Replayable** — historical system state can be reconstructed at any point in time.
- **Human-governed** — analyst judgement augments machine outputs without replacing historical evidence.

---

## MVP

The current implementation demonstrates the complete architecture using deterministic mocked models and synthetic sales scenarios.

Although the machine learning components are mocked, the surrounding architecture—including ontology objects, hydration, governance, human review and lineage—matches the intended production design.
