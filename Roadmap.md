# Eidos Roadmap

## Scope & Principles

* **Eidos is an MCP tool** (agent-neutral), repo stands alone.
* **Three layers** inside Eidos:
  **Shades** (appearances & geometry, no names) → **Bearers** (signs, consensus, escalation) → **Forms** (truth, identity, events, memory).
* **Epistemic firewalls:** Shades never labels; Bearers never asserts truth; Forms adjudicates and remembers.
* **Compatibility-first:** ingest common streams (RTSP/MJPEG) without vendor lock-in.
* **Provenance & auditability** at every step; privacy policies centrally enforced.

---

## Milestones

### M0 — Project Bootstrap (Week 0–1)

**Deliverables**

* Repo scaffolding, `mcp.json`, CI, basic docs (README, this roadmap).
* JSON Schemas stubs: `appearance`, `sign`, `form`, `event`.
* Local dev stack: Docker Compose with MediaMTX (or test RTSP source).

**Exit Criteria**

* `list_streams`, `add_stream`, `remove_stream` MCP methods return stubbed data.
* End-to-end smoke test runs in CI.

---

### M1 — Shades MVP (Week 1–3)

**Deliverables**

* RTSP/MJPEG ingest with jitter buffer, UTC/PTS stamping.
* Optional geometry outputs that are **non-semantic**: motion mask (bg subtractor), optical flow (optional), pose from SLAM if enabled.
* Snapshot endpoint.
* Metrics: input fps/bitrate, queue depth, drops.

**Exit Criteria**

* `get_appearance(stream_id)` yields frames + timestamps (+ optional geometry).
* Can ingest at least **4 × 1080p\@15fps** concurrently on a mid GPU host, decode off if Bearers consumes packets.

---

### M2 — Bearers v1 (Week 3–6)

**Deliverables**

* Model registry with tiers (`yolo-tiny`, `yolo-base` via ONNX Runtime/TensorRT where available).
* Tracking (ByteTrack/OC-SORT).
* Consensus v1: IoU matching + Weighted Box Fusion + calibrated voting; temporal smoothing.
* Criteria Engine: thresholds, margin tests, novelty check via CLIP embedding distance.
* Escalation hook: produce `escalation_request` (ROI + context), no external call yet.

**Exit Criteria**

* `get_signs(stream_id, since=…)` returns fused detections w/ provenance.
* New-object events detectable via track birth.
* Sustains **1080p\@15fps** per stream with `yolo-tiny` on CPU/GPU mix; graceful degradation under load.

---

### M3 — Forms v1 (Week 6–9)

**Deliverables**

* Entity graph with stable IDs; acceptance policy over signs.
* Events: `appearance`, `placement`, `movement`, `orientation_change`, `departure`.
* Reference DB: exemplars (crops+masks), embeddings (CLIP), provenance.
* Dataset export: COCO/YOLO (images, labels, splits).
* Privacy/policy layer (redaction flags, retention, escalation budgets).

**Exit Criteria**

* `get_forms(query…)` returns accepted facts; `get_events(…)` streams derived events.
* Repeated sightings of the same object resolve to the same entity (>90% on test set).

---

### M4 — Escalation & Feedback Loop (Week 9–11)

**Deliverables**

* Escalation adapters: post `escalation_request` to an orchestrator (e.g., Dubito) and accept `escalation_response` back as additional signs.
* User labeling pathway (manual confirm → Forms accept).
* Policy-driven rate limiting, redaction (blur faces/plates) before escalation.

**Exit Criteria**

* Round-trip escalations enrich Forms and reduce future uncertainty on similar exemplars.

---

### M5 — Robustness & Observability (Week 11–13)

**Deliverables**

* Health reporting per stream; automatic backoff/reconnect; clock drift detection.
* Prometheus metrics; structured logs; OpenTelemetry traces across layers.
* Fault injection tests (packet loss, PTS jumps).

**Exit Criteria**

* Chaos tests pass with graceful recovery; SLOs defined (see below).

---

### M6 — Alpha Release (Week 13–14)

**Deliverables**

* Hardening, config docs, MCP endpoint docs.
* Example integrations: CLI, minimal web preview.

**Exit Criteria**

* Tag `v0.1.0-alpha`; runbook for deployment.

---

## Post-Alpha Tracks

### Compatibility

* ONVIF Profile S (discovery/events) adapter → feeds **external signs** to Bearers.
* Recording/playback for offline runs; VOD reprocessing.

### Identity & Recognition

* Re-ID embeddings store; person/object instance matching over time.
* Optional face pipeline (pluggable; policy-gated).

### Performance & Scale

* Hardware decode path (NVDEC/V4L2 M2M) when Bearers needs frames.
* Batch inference and ROI tiling; stream sharding across workers.

### Data & Training

* Active learning loop: Forms exports hard examples; fine-tuning via simple recipe (no bespoke research).
* Model calibration refresh tooling.

---

## MCP Surface (stable through v0.1)

*Query/command names only; spec lives in protocol docs.*

* **Streams:** `list_streams`, `add_stream`, `remove_stream`, `snapshot`
* **Appearances:** `get_appearance`, `subscribe_appearances`
* **Signs:** `get_signs`, `subscribe_signs`
* **Forms/Events:** `get_forms`, `get_events`, `subscribe_events`
* **Policy/Models:** `get_policy`, `set_policy`, `list_models`, `set_model_tier`
* **Escalation:** `post_escalation_request`, `post_escalation_response`

---

## SLO Targets (initial)

* **Latency (appearance → signs):** p95 ≤ 250 ms at 1080p\@15fps, `yolo-tiny`.
* **Event freshness (signs → form/event):** p95 ≤ 300 ms.
* **Uptime:** 99.5% over 30 days (excluding upstream camera outages).
* **Data retention:** configurable, default 72h for raw frames, 30d for crops/embeddings.

---

## Risks & Mitigations

* **Vendor stream quirks / RTSP instability:** Strict retries, keepalive, protocol fallbacks; fuzz tests.
* **Model drift / poor calibration:** Regular calibration set; monitoring of confidence vs. acceptance.
* **Privacy concerns:** Redaction defaults, policy gating of escalation, audit trails.
* **Resource contention:** Backpressure and admission control; model tier auto-downgrade under load.

---

## Test Plan (high level)

* **Unit:** adapters, fusion, policy decisions.
* **Integration:** multi-cam pipelines, lost packets, clock skew, reconnection.
* **Scenario:** “new object enters,” “ambiguous object,” “escalation accepted/rejected,” “identity persistence across sessions.”
* **Performance:** throughput/latency under mixed workloads; GPU/CPU permutations.

---

## Tech Notes (pragmatic defaults)

* **Shades:** GStreamer/FFmpeg for ingest; Python service first, option to migrate hot paths to Rust/Go later.
* **Bearers:** ONNX Runtime/TensorRT; PyTorch for calibration/offline tasks.
* **Forms:** SQLite + Parquet for artifacts; FAISS for embeddings; Arrow IPC between services.
* **Telemetry:** Prometheus, OpenTelemetry.

---

This roadmap gets Eidos from **reliable ingest** to **durable scene understanding** with clear contracts, measurable SLOs, and room to grow—without tying it to any one orchestrator.
