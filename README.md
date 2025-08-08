# Eidos

*Eidos is a vision-awareness service for an agentic LLM, packaged as an MCP tool. It ingests camera streams, normalizes appearances and geometry, runs multi-model inference to propose signs (labels), curates durable entities and events, and offers a coherent, queryable picture of the scene.*

<img width="50%" alt="94a7fb97-28e8-46e7-a306-5dd636842d26" src="https://github.com/user-attachments/assets/5b5f1b23-1a1f-4e36-ae34-47c8cf41e65c" />

## Overview

**Eidos** turns raw video into structured, durable understanding while staying **Agent-agnostic** and reusable by any MCP-capable agent. It is organized around a strict interpretation of Plato’s cave to keep responsibilities clean and composable.

### Philosophy → Architecture

* **Shades (Appearances & Geometry)**
  The “shadows.” Centralized access/ingest for cameras and NVRs. Produces time-aligned appearances (pixels) and non-semantic geometry (e.g., segmentation proposals, depth/flow, SLAM poses). **No names.**

* **Bearers (Signs & Consensus)**
  The “name-givers.” Runs detectors/trackers/OCR/pose and fuses multiple models. Emits **signs** (competing labeled hypotheses with provenance), applies consensus and uncertainty handling, and orchestrates escalation when needed (e.g., to higher-tier models or external helpers). **Claims, not truth.**

* **Forms (Truth, Identity & Memory)**
  The “accepted forms.” Curates stable entities and relations over time, resolves identity, records provenance, keeps exemplars and embeddings, and derives **events** (placement, movement, orientation) that summarize what actually happened.

### What Eidos Does

* **Normalize** diverse video sources into consistent, time-stamped appearances and geometry.
* **Interpret** scenes via multi-model inference and tracking, preserving every model’s claim with provenance.
* **Consolidate** evidence into durable entities and events, maintaining a searchable memory over time.
* **Escalate** difficult cases through policy-driven routes (stronger local models, external assistants, or user input).
* **Integrate** cleanly as an **MCP tool**, so DECS (or any other orchestrator) can query understanding without Eidos depending on DECS specifics.

### Data Flow (High Level)

```
Shades (appearances, geometry)
    → Bearers (signs, consensus, escalation)
        → Forms (entities, relations, events)
            → Orchestrators (e.g., your LLM agent) for narrative & action
```

### Design Principles

* **Epistemic firewalls:**
  Shades never names; Bearers never asserts truth; Forms decides and remembers.
* **Provenance first:**
  Every claim keeps source, confidence, and context for auditability.
* **Agent-neutral:**
  No agent-specific dependencies; exposed as a portable MCP tool.
* **Composable intelligence:**
  Prefer fusing existing models and lookups over building bespoke ones.
* **Privacy & control:**
  Policies govern escalation, retention, and redaction without leaking semantics across layers.

### Core Concepts (Terminology)

* **Appearance:** Time-aligned pixels + non-semantic geometry.
* **Sign:** A labeled hypothesis about an appearance, with confidence and provenance.
* **Form:** An accepted fact about an entity or relation after evaluating signs over time.
* **Event:** A narrative-ready change derived from forms (e.g., “object placed on desk”).

---

Eidos aims to provide a clean, principled bridge from video **appearance** to machine-usable **understanding**, while remaining modular, auditable, and easy to orchestrate in your agent. Looking for an Agent and hungry for more philosophical references? Check out [DECS-stack](https://github.com/inductiontrace/decs-stack)! A three‑voice architecture inspired by “Dubito, ergo cogito, ergo sum.”
