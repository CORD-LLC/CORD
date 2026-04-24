# CORD Specification v1.0

**Canonical Object for Relational Data**
Status: Release
Version: 1.0.0
Published: September 2025
License: Apache 2.0

---

## Document Hierarchy

This document — `CORD-v1.0.md` — is the **normative reference specification** for CORD v1.0. It is the authoritative source for all conformance requirements, schema definitions, and protocol semantics.

The accompanying RFC PDF is an abstract summary intended for initial evaluation. Where the RFC and this document conflict, this document prevails.

---

## Abstract

CORD (Canonical Object for Relational Data) is a protocol specification for preserving AI-generated structured data when it is translated into legacy system formats. It defines a canonical interaction envelope, a versioning model, a dual-format output structure, a fidelity scoring metric (EFS), an envelope integrity mechanism, and a conformance model.

CORD operates at the data layer — above transport protocols, below application business logic.

## 1. Terminology

- **Envelope**: A CORD Interaction Envelope — the canonical container for a single interaction event.
- **EFS**: Envelope Fidelity Score — a normalized metric [0.0, 1.0] quantifying translation fidelity.
- **Snapshot**: A full-state envelope that serves as the root of a version chain.
- **Delta**: An incremental update envelope referencing a parent by UUID.
- **FULL**: Mapping status indicating complete preservation in the target system.
- **PARTIAL**: Mapping status indicating partial preservation with information loss.
- **NONE**: Mapping status indicating no preservation — field has no target representation.

## 2. Envelope Structure

### 2.1 Required Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| cord_version | string | Yes | Spec version: `"1.0"` |
| envelope_id | string (UUID v4) | Yes | Globally unique identifier |
| envelope_type | enum | Yes | `"snapshot"` or `"delta"` |
| version | integer (≥1) | Yes | Monotonic counter within version chain |
| parent_envelope_id | string \| null | Yes | Parent UUID; `null` for snapshots |
| created_at | string (ISO 8601) | Yes | UTC creation timestamp |
| domain | string | Yes | Classification domain |
| source_system | string | Yes | Originating AI system identifier |
| target_system | string | Yes | Target legacy system identifier |
| fields | array | Yes | Structured field objects |
| legacy_output | object | Yes | Flattened legacy-format representation |
| loss_report | object | Yes | EFS score and per-field mapping status |
| event_log | array | Optional | Ordered interaction events |
| x_cord_digest | string | Recommended | SHA-256 envelope integrity digest |

### 2.2 Field Object

| Property | Type | Description |
|---|---|---|
| name | string | Machine-readable identifier |
| value | any | String, number, array, or object |
| type | string | Semantic type |
| confidence | float [0.0–1.0] | AI confidence in this value |
| source | string | Value origin |
| changed | boolean | True if changed from parent (delta only) |

Supported types: `text`, `integer`, `float`, `number`, `boolean`, `date`, `phone`, `email`, `currency`, `code`, `code_array`, `text_array`, `structured`.

Supported sources: `nlp`, `user_input`, `classifier`, `rule`, `external`, `structured`, `inferred`.

### 2.3 Loss Report

| Property | Type | Description |
|---|---|---|
| efs | float [0.0–1.0] | Envelope Fidelity Score |
| field_mappings | array | Per-field mapping entries |

### 2.4 Field Mapping Entry

| Property | Type | Description |
|---|---|---|
| field | string | Source field name |
| status | enum | `FULL`, `PARTIAL`, or `NONE` |
| target | string \| null | Target field name |
| partial_coefficient | float \| null | Resolved coefficient in [0.5, 0.9] when PARTIAL |
| note | string | Human-readable explanation |

### 2.5 Event Log Entry

| Property | Type | Description |
|---|---|---|
| event_type | string | `utterance`, `classification`, `input`, `signal`, `handoff`, `error` |
| timestamp | string | UTC ISO 8601 |
| actor | string | Entity that produced the event |
| content | string \| null | Event content |
| field | string \| null | Affected field |
| data | object \| null | Additional event data |

## 3. Versioning

### 3.1 Snapshots

A snapshot envelope is the root of a new version chain. It contains complete interaction state.

- `version` MUST be 1
- `parent_envelope_id` MUST be null
- `envelope_type` MUST be `"snapshot"`

### 3.2 Deltas

A delta envelope is an incremental update to an existing version chain.

- `version` MUST be `parent_version + 1`
- `parent_envelope_id` MUST reference the parent envelope's UUID
- `envelope_type` MUST be `"delta"`
- Only changed fields are included; each MUST have `changed: true`

### 3.3 Event-Log-Only Deltas

A delta envelope with an empty `fields` array is valid when the purpose is to record event log entries without modifying interaction field state, provided the envelope has a non-null `parent_envelope_id` and a version counter incremented from the parent.

### 3.4 Version Chain Materialization

Consumers MAY materialize full interaction state at any point by replaying the snapshot and applying deltas forward in version order.

## 4. Envelope Fidelity Score (EFS)

### 4.1 Computation

```
EFS = Σ(weight_i × coefficient_i) / Σ(weight_i)
```

Fidelity coefficients:
- FULL → 1.0
- PARTIAL → coefficient in [0.5, 0.9], baseline 0.5
- NONE → 0.0

Field weighting is implementation-defined.

### 4.2 PARTIAL Coefficient Bounds

The PARTIAL fidelity coefficient MUST be in the range [0.5, 0.9]. Implementations MUST NOT use a value below 0.5. The baseline default is 0.5.

### 4.3 Confidence-Based Coefficient Selection

When AI confidence scores are available on source fields, implementations SHOULD propagate the confidence score into the PARTIAL coefficient selection:

- confidence < low_threshold → coefficient = 0.5 (floor)
- confidence ≥ high_threshold → coefficient = 0.9 (ceiling)
- otherwise → linear interpolation in [0.5, 0.9]

The low_threshold and high_threshold are implementation-defined. Recommended defaults: low = 0.5, high = 0.95.

### 4.4 Auditable Coefficients

When a field's mapping status is PARTIAL, the resolved coefficient SHOULD be recorded in the `partial_coefficient` property of the field mapping entry. This enables full auditability of EFS computation.

### 4.5 Interpretation

| Range | Tier | Description |
|---|---|---|
| 0.90–1.00 | High fidelity | Nearly all content preserved |
| 0.50–0.89 | Moderate fidelity | Meaningful content preserved but significant loss |
| 0.00–0.49 | Low fidelity | Most content lost; schema modernization recommended |

### 4.6 EFS as Translation Metric

EFS measures translation fidelity — the proportion of source information preserved in the target format — not the accuracy or completeness of the source data. A score of 1.0 indicates faithful translation, not that the source data is correct.

## 5. Dual-Format Output

Every envelope carries both representations:

- **fields**: Full-fidelity CORD-native representation with confidence and source attribution
- **legacy_output**: Flattened legacy-format representation matching the target system's schema

This enables direct comparison and downstream routing.

## 6. Media Type

`application/cord+json`

Implementations MUST produce valid JSON. Use in `Content-Type` and `Accept` headers for HTTP transport.

## 7. Mapping Status Taxonomy

### 7.1 FULL

Field value completely represented in legacy output. No semantic content discarded. The value, type, and structure are preserved exactly (minor type coercion, e.g. integer → string, is acceptable).

### 7.2 PARTIAL

Field value partially represented. Some semantic content preserved; meaningful information was discarded or truncated. Examples:

- Array flattened to comma-separated string
- Structured object merged into text blob
- Value truncated to fit character limit
- Date losing timezone information
- Multiple values reduced to primary value only

### 7.3 NONE

Field could not be represented in legacy output. No information from this field exists in the legacy record. The data was extracted by the AI but the target system has no field to receive it.

## 8. Event Log

The event log is an ordered, typed, timestamped record of discrete events that produced the envelope's field state. It provides a complete audit trail.

Event log is optional per CORD v1.0. Implementations that include event logs are eligible for CORD-Compliant conformance.

## 9. Applicability

### 9.1 CORD is required when

- AI-generated structured data passes through legacy or third-party systems
- Data transformations occur across system boundaries
- Fidelity, traceability, or auditability are critical

### 9.2 CORD is not required when

- Systems are natively interoperable with no schema mismatch
- No transformation or interpretation occurs during transit
- Data fidelity is not a concern for the use case

## 10. Conformance

See [CONFORMANCE.md](../CONFORMANCE.md) for the full conformance model.

### 10.1 Summary

Three validation categories:
1. Envelope Structure
2. EFS Reporting
3. Mapping Completeness

Four tiers: CORD-Compliant, CORD-Partial, CORD-Compatible, Non-Compliant.

Self-assessment is sufficient under v1.0.

## 11. Security Considerations

### 11.1 Sensitive Data in Fields

CORD envelopes may contain sensitive personal, medical, or financial data. Implementations MUST apply appropriate encryption in transit (TLS 1.2 minimum) and at rest. CORD does not define encryption mechanisms — these are the responsibility of the implementing system.

### 11.2 Envelope Integrity

Implementations SHOULD ensure envelope integrity via hashing or signing. Two approaches are supported:

**Option A — Hash digest (RECOMMENDED for v1.0)**

Implementations SHOULD include an `x_cord_digest` extension field containing a SHA-256 hash of the canonical JSON serialization of the envelope (excluding the `x_cord_digest` field itself):

```json
{
  "cord_version": "1.0",
  "envelope_id": "...",
  "x_cord_digest": "sha256:a3f9c201b7d1e443..."
}
```

Canonical serialization: sorted keys, no whitespace separators, ensure ASCII encoding.

**Option B — Asymmetric signature**

Implementations requiring non-repudiation SHOULD apply an asymmetric signature (e.g. RS256 or ES256) to the serialized envelope and transmit the signature out-of-band or as an extension field. CORD does not define a signing scheme in v1.0 — this is implementation-defined.

### 11.3 Confidence Score Handling

`confidence` values are AI-generated estimates and MUST NOT be treated as guarantees. Downstream systems that make decisions based on field values SHOULD consider the associated confidence score.

### 11.4 Replay Protection

Implementations that require replay protection SHOULD validate `envelope_id` uniqueness at ingestion. The `created_at` timestamp SHOULD be used to reject envelopes older than an implementation-defined staleness threshold.

### 11.5 Conformance Validation

CORD-compliant systems MUST pass three validation categories: (1) envelope structure — all required fields present and correctly typed; (2) EFS reporting — loss_report complete with valid status values and correctly computed score; (3) mapping completeness — every source field has a corresponding loss_report entry with no silent omissions.

Self-assessment against these three categories is sufficient to declare CORD-compliant status under v1.0. A formal certification program with automated test suites and a public conformance registry is under development for a future revision.
