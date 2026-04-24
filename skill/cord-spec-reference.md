# CORD Specification Reference — v1.0

**Canonical Object for Relational Data**
Full specification: https://cordspec.org

## Envelope Structure

Every CORD envelope MUST contain these top-level fields:

| Field | Type | Required | Description |
|---|---|---|---|
| cord_version | string | Yes | `"1.0"` |
| envelope_id | string (UUID v4) | Yes | Globally unique identifier |
| envelope_type | enum | Yes | `"snapshot"` or `"delta"` |
| version | integer (≥1) | Yes | Monotonic counter within version chain |
| parent_envelope_id | string \| null | Yes | Parent UUID; `null` for snapshots |
| created_at | string (ISO 8601) | Yes | UTC creation timestamp |
| domain | string | Yes | Classification domain (e.g. `"healthcare"`) |
| source_system | string | Yes | Originating AI system identifier |
| target_system | string | Yes | Target legacy system identifier |
| fields | array | Yes | Structured field objects |
| legacy_output | object | Yes | Flattened legacy-format representation |
| loss_report | object | Yes | EFS score and per-field mapping status |
| event_log | array | Optional | Ordered interaction events |
| x_cord_digest | string | Recommended | SHA-256 digest: `"sha256:<hex>"` |

## Field Object

Each entry in the `fields` array:

| Property | Type | Description |
|---|---|---|
| name | string | Machine-readable identifier (snake_case) |
| value | any | String, number, array, or object |
| type | string | Semantic type: `text`, `number`, `integer`, `float`, `date`, `phone`, `email`, `currency`, `boolean`, `code`, `code_array`, `text_array`, `structured` |
| confidence | float [0.0–1.0] | AI confidence in this value |
| source | string | Origin: `nlp`, `user_input`, `classifier`, `rule`, `external`, `structured`, `inferred` |
| changed | boolean | True if changed from parent (delta only) |

## Loss Report

| Property | Type | Description |
|---|---|---|
| efs | float [0.0–1.0] | Envelope Fidelity Score |
| field_mappings | array | Per-field mapping entries |

### Field Mapping Entry

| Property | Type | Description |
|---|---|---|
| field | string | Source field name |
| status | enum | `FULL`, `PARTIAL`, or `NONE` |
| target | string \| null | Target field name, or null if NONE |
| partial_coefficient | float \| null | Resolved coefficient in [0.5, 0.9] when PARTIAL |
| note | string | Human-readable explanation |

## EFS Computation

```
EFS = Σ(weight_i × coefficient_i) / Σ(weight_i)

Coefficients:
  FULL    → 1.0
  PARTIAL → partial_coefficient (default 0.5, range [0.5, 0.9])
  NONE    → 0.0

PARTIAL coefficient MUST be >= 0.5. Implementations MUST NOT use a value below 0.5.
```

### Confidence Propagation (recommended)

When AI confidence scores are available:
```
if confidence < 0.5:   coefficient = 0.5   (floor)
if confidence >= 0.95: coefficient = 0.9   (ceiling)
else:                  coefficient = 0.5 + ((confidence - 0.5) / 0.45) * 0.4
```

### EFS Interpretation

| Range | Tier | Meaning |
|---|---|---|
| 0.90–1.00 | High | Nearly all content preserved |
| 0.50–0.89 | Moderate | Meaningful content preserved but significant loss |
| 0.00–0.49 | Low | Most content lost; schema modernization recommended |

## Envelope Integrity

### SHA-256 Digest

1. Serialize envelope to canonical JSON (sorted keys, no whitespace, ensure ASCII)
2. Exclude the `x_cord_digest` field before hashing
3. Compute SHA-256 of UTF-8 encoded canonical string
4. Store as `x_cord_digest: "sha256:<hex_digest>"`

### Replay Protection

- Validate `envelope_id` uniqueness at ingestion
- Reject envelopes with `created_at` older than staleness threshold

## Versioning

- **Snapshot**: Full-state envelope. `version = 1`, `parent_envelope_id = null`.
- **Delta**: Incremental update. `version = parent + 1`, `parent_envelope_id` = parent's UUID. Only changed fields included, each with `changed: true`.
- A delta with empty `fields` array is valid for event-log-only updates.

## Conformance

### Three Validation Categories

1. **Envelope Structure**: Required fields present, types correct, snapshot/delta rules satisfied
2. **EFS Reporting**: loss_report present with valid EFS, field_mappings with FULL/PARTIAL/NONE
3. **Mapping Completeness**: Every source field has a mapping entry

### Four Tiers

| Tier | Requirements |
|---|---|
| CORD-Compliant | All three categories pass; event log and notes present |
| CORD-Partial | Categories 2+3 pass; may omit event log, notes, or version chaining |
| CORD-Compatible | Core envelope fields without full EFS reporting |
| Non-Compliant | No CORD envelope produced |

## Media Type

`application/cord+json`

Use in `Content-Type` and `Accept` headers. Implementations MUST produce valid JSON.

## Event Log Entry

| Property | Type | Description |
|---|---|---|
| event_type | string | `utterance`, `classification`, `input`, `signal`, `handoff`, `error` |
| timestamp | string | UTC ISO 8601 |
| actor | string | Entity that produced the event |
| content | string \| null | Event content (utterance events) |
| field | string \| null | Affected field (classification events) |
| data | object \| null | Additional event data |
