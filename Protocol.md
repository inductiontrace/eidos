# Eidos Protocol — **Draft**

**Status:** draft-0.1 (subject to change)
**Scope:** Contract for the Eidos MCP tool. Defines message types and callable methods. Focuses on wire-level behavior; implementation details are out of scope.

---

# 1) Transport & Versioning

* **Transport:** Model Context Protocol (MCP). Each item below is an MCP method with JSON request/response bodies.
* **Content type:** `application/json` unless noted.
* **Versioning:** `X-Eidos-Version` header and `version` fields in responses. Semantic versioning `MAJOR.MINOR.PATCH`. Clients **MUST** check compatibility (`major` changes are breaking).
* **Time:** RFC 3339 UTC timestamps.
* **IDs:** Strings, globally unique (ULID recommended).
* **Units:** Pixels for image coordinates, meters for SLAM/world coordinates.
* **Coordinate frame (images):** origin (0,0) at top-left; x right, y down.
* **Bounding boxes:** `[x, y, w, h]` in pixels (float).
* **Masks:** RLE per COCO (`counts`, `size`) or compressed PNG; implementation **MUST** advertise supported encodings.
* **Confidences:** 0.0–1.0; calibrated if `provenance.calibrated=true`.

---

# 2) Epistemic Boundaries (normative)

* **Shades** **MUST NOT** emit semantic labels. It outputs **Appearances** (pixels + non-semantic geometry).
* **Bearers** **MUST** express semantic outputs as **Signs** (hypotheses with provenance) and **MUST NOT** mark them as truth.
* **Forms** is the **only** source of accepted facts (**Forms**) and derived **Events**.
* Vendor/camera analytics **MUST** be wrapped as **ExternalSigns** and processed like any other Sign.

---

# 3) Data Models

## 3.1 Appearance

Non-semantic view of a frame and optional geometry.

```json
{
  "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R",
  "stream_id": "garage_north",
  "ts_utc": "2025-08-08T16:43:12.174Z",
  "pts_ms": 1234567,
  "media": {
    "format": "shm",                   // shm|uri|bytes
    "ref": "shm://eidos/buf/8742219",  // or file://..., http(s)://..., data:...
    "width": 1920,
    "height": 1080,
    "codec": "h264"                    // if compressed
  },
  "geometry": {
    "segments": [
      { "bbox": [412.0,220.0,180.0,160.0], "mask": { "rle": {"counts":"...","size":[1080,1920]} }, "score": 0.91 }
    ],
    "depth_ref": "shm://eidos/depth/8742219",  // optional
    "flow_ref":  "shm://eidos/flow/8742219"    // optional
  },
  "pose": {                                  // optional SLAM camera/world pose
    "frame": "world",
    "translation_m": [1.23, 0.04, 0.77],
    "rotation_qwxyz": [0.98, 0.01, 0.02, 0.19],
    "covariance": null
  },
  "version": "0.1.0"
}
```

## 3.2 ExternalSigns

Vendor/NVR analytics passed through unmodified (as claims).

```json
{
  "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R",
  "source": "vendor:hikvision",
  "detections": [
    { "bbox": [400,210,190,170], "label": "person", "confidence": 0.74 }
  ],
  "version": "0.1.0"
}
```

## 3.3 Sign

A semantic hypothesis produced by Bearers (or external escalation). Not truth.

```json
{
  "sign_id": "01J2W6T3PF4D3F7JY0Z2H3C2RK",
  "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R",
  "track_id": "trk-3f9a",
  "roi": { "bbox": [412,220,180,160] },
  "label": "raspberry_pi",
  "confidence": 0.78,
  "features": { "embedding": "base64...", "clip_model": "ViT-L/14" },
  "provenance": {
    "models": [
      {"name": "yolo-n", "rev": "e1d0s", "p": 0.71},
      {"name": "clip-retrieval", "rev": "b9", "p": 0.82}
    ],
    "fusion": "wbf+weighted_vote",
    "calibrated": true,
    "created_utc": "2025-08-08T16:43:12.190Z",
    "origin": "bearers"
  },
  "version": "0.1.0"
}
```

## 3.4 Form

An accepted fact about an entity or relation.

```json
{
  "form_id": "form-01J2W6V9KQ5R",
  "entity_id": "ent-91c2",
  "class": "raspberry_pi",
  "attributes": {
    "location": "desk",
    "present": true,
    "pose": { "tilt_deg": 22.0 }
  },
  "evidence": [{ "sign_id": "01J2W6T3PF4...", "weight": 0.6 }, { "sign_id": "sig-oracle", "weight": 0.4 }],
  "policy": { "accept_threshold": 0.75 },
  "version": "0.1.0",
  "ts_accepted": "2025-08-08T16:43:12.240Z"
}
```

## 3.5 Event

Narrative-ready change derived from Forms.

```json
{
  "event_id": "evt-201",
  "kind": "placement",     // enum: appearance|placement|movement|orientation_change|departure|interaction
  "ts_utc": "2025-08-08T16:43:13.002Z",
  "actor": "ent-user-42",  // optional
  "object": "ent-91c2",
  "place": "desk",
  "details": { "pose_change": { "tilt_deg": 22.0 } },
  "evidence_forms": ["form-01J2W6V9KQ5R"],
  "version": "0.1.0"
}
```

## 3.6 EscalationRequest / EscalationResponse

Used when Bearers seeks external help (e.g., cloud LMM or user).

**Request**

```json
{
  "escalation_id": "esc-01J2W7A1A3TQ",
  "reason": "low_confidence|disagreement|novelty",
  "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R",
  "track_id": "trk-3f9a",
  "roi": { "bbox": [412,220,180,160] },
  "context": {
    "best_hypotheses": [{"label":"bag","p":0.42},{"label":"cat","p":0.33}],
    "image_ref": "uri://eidos/crops/esc-01J2W7A1A3TQ.jpg",
    "metadata": { "stream_id": "garage_north" }
  },
  "privacy": { "blur_faces": true, "policy_ref": "policy/default" },
  "version": "0.1.0"
}
```

**Response** (ingested as an additional Sign; `origin: oracle`)

```json
{
  "escalation_id": "esc-01J2W7A1A3TQ",
  "sign": {
    "sign_id": "sig-oracle",
    "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R",
    "track_id": "trk-3f9a",
    "roi": { "bbox": [412,220,180,160] },
    "label": "raspberry_pi",
    "confidence": 0.83,
    "provenance": { "origin": "oracle", "name": "openai-gpt-v", "rev": "2025-08-08", "calibrated": false }
  },
  "version": "0.1.0"
}
```

---

# 4) Methods (MCP)

> **Note:** Parameter and return schemas are normative. Omitted fields are optional unless marked **required**.

## 4.1 Streams

### `list_streams`

Request:

```json
{ "cursor": null, "limit": 100 }
```

Response:

```json
{
  "streams": [
    {"id":"garage_north","uri":"rtsp://...","state":"connected","codec":"h264","width":1920,"height":1080,"fps":15.0}
  ],
  "next_cursor": null,
  "version": "0.1.0"
}
```

### `add_stream`

```json
{
  "id": "garage_north",
  "uri": "rtsp://user:pass@host:554/Streaming/Channels/101",
  "transport": "tcp",
  "latency_ms": 120,
  "decode": false
}
```

Response: `{ "ok": true }`

### `remove_stream`

```json
{ "id": "garage_north" }
```

Response: `{ "ok": true }`

### `snapshot`

```json
{ "stream_id": "garage_north", "format": "jpeg", "quality": 85 }
```

Response:

```json
{ "image_bytes": "base64...", "ts_utc": "2025-08-08T16:43:12.174Z" }
```

## 4.2 Appearances

### `get_appearance`

```json
{ "appearance_id": "01J2W6Q3Y0X9F0C3C1VQKJ0V8R" }
```

Response: **Appearance**

### `subscribe_appearances`

Server streams **Appearance** objects.
Request:

```json
{ "stream_id": "garage_north", "since": "2025-08-08T16:43:00Z", "mode": "live" }
```

## 4.3 Signs

### `get_signs`

```json
{
  "stream_id": "garage_north",
  "since": "2025-08-08T16:43:00Z",
  "filters": { "label_in": ["person","raspberry_pi"], "min_conf": 0.4 }
}
```

Response:

```json
{ "signs": [ /* Sign */ ], "version": "0.1.0" }
```

### `subscribe_signs`

Request:

```json
{ "stream_id": "garage_north", "since": null }
```

Stream: **Sign**

### `post_external_signs`

```json
{ "appearance_id":"...", "source":"vendor:...", "detections":[...] }
```

Response: `{ "ok": true }`

### `post_escalation_request`

Body: **EscalationRequest**
Response: `{ "accepted": true, "escalation_id": "esc-..." }`

### `post_escalation_response`

Body: **EscalationResponse**
Response: `{ "ok": true }`

## 4.4 Forms & Events

### `get_forms`

```json
{
  "entity_id": null,
  "class_in": ["person","raspberry_pi"],
  "since": "2025-08-08T00:00:00Z"
}
```

Response:

```json
{ "forms": [ /* Form */ ], "version": "0.1.0" }
```

### `get_events`

```json
{
  "stream_id": "garage_north",
  "kinds": ["placement","orientation_change"],
  "since": "2025-08-08T00:00:00Z"
}
```

Response:

```json
{ "events": [ /* Event */ ], "version": "0.1.0" }
```

### `subscribe_events`

Request: `{ "since": null, "kinds": null }`
Stream: **Event**

## 4.5 Models & Policy

### `list_models`

Response:

```json
{
  "models": [
    { "name":"yolo-n", "tier":"tiny|base|heavy", "rev":"e1d0s", "capabilities":["detect"] },
    { "name":"clip-retrieval", "tier":"base", "capabilities":["embed","retrieve"] }
  ],
  "version":"0.1.0"
}
```

### `set_model_tier`

```json
{ "name": "yolo-n", "tier": "base" }
```

Response: `{ "ok": true }`

### `get_policy`

Response (selected fields):

```json
{
  "thresholds": { "accept": 0.75, "escalate": 0.45 },
  "consensus": { "box_fusion": "wbf", "vote": "weighted" },
  "privacy": { "redact_faces": true, "retain_days": 30 },
  "version": "0.1.0"
}
```

### `set_policy`

```json
{
  "thresholds": { "accept": 0.8, "escalate": 0.5 },
  "privacy": { "redact_faces": true }
}
```

Response: `{ "ok": true }`

## 4.6 Health & Metrics

### `get_health`

Response:

```json
{
  "uptime_s": 123456,
  "streams": [{ "id":"garage_north","state":"connected","loss_pct":0.2 }],
  "version":"0.1.0"
}
```

### `get_metrics`

Response: Prometheus text or JSON snapshot (implementation-defined; content-type advertised via MCP capability).

---

# 5) Errors

Uniform error object returned with non-OK results:

```json
{
  "error": {
    "code": "EIDOS.BAD_REQUEST",  // examples below
    "message": "human readable",
    "details": { "field": "uri", "reason": "invalid" }
  }
}
```

**Codes**

* `EIDOS.BAD_REQUEST`
* `EIDOS.NOT_FOUND`
* `EIDOS.CONFLICT`
* `EIDOS.UNAUTHORIZED`
* `EIDOS.UNAVAILABLE`   (resource temporarily unavailable)
* `EIDOS.UNSUPPORTED`   (capability not enabled)
* `EIDOS.INTERNAL`

---

# 6) Capability Advertisement

Clients may query optional features:

```json
{
  "caps": {
    "media_refs": ["shm","uri","bytes"],
    "mask_encodings": ["rle","png"],
    "pose": true,
    "depth": true,
    "flow": false,
    "escalation": true
  },
  "version": "0.1.0"
}
```

---

# 7) Security & Privacy (protocol level)

* **Redaction hints** may be supplied in `EscalationRequest.privacy`. Eidos **MUST** honor policy before emitting any external request.
* **Provenance** is required on all **Signs**; clients **MUST** not treat unlabeled provenance as trusted.
* **Retention** and **access** are governed by policy (`get_policy` / `set_policy`), not by ad-hoc method behavior.

---

# 8) Conformance Summary

An implementation claiming conformance to **Eidos Protocol draft-0.1**:

1. Implements all required methods in §4 (Streams, Appearances, Signs, Forms, Events, Models/Policy, Health).
2. Emits **Appearances** without semantic labels.
3. Emits **Signs** with provenance, never as truth.
4. Emits **Forms/Events** only after policy evaluation over Signs.
5. Supports RFC 3339 timestamps, specified coordinate systems, and advertised media/mask capabilities.
