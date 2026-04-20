# CORD Protocol Specification

**Version:** 1.0.0
**Status:** Published
**Media Type:** `application/cord+json`
**Maintained by:** CORD Protocol Holdings LLC
**Website:** cordspec.org

---

## 1. Introduction

CORD (Canonical Object for Relational Data) is an open protocol standard for structuring, versioning, and transmitting AI-generated conversational interaction data.

CORD addresses a fundamental interoperability problem: AI agents operating across modern channels (text, voice, web chat, form submission) collect richly structured data that legacy receiving systems — built before the era of AI — cannot represent. When that data is translated into legacy interchange formats, information is destroyed. CORD makes that loss measurable, comparable, and actionable.

### 1.1 Design Principles

- **Domain-agnostic.** CORD imposes no constraints on field vocabulary. The same protocol serves healthcare, real estate, insurance, legal, financial services, HR, government, and any other domain.
- **Fidelity-first.** The full-fidelity envelope is always the source of truth. Legacy output is a derived representation.
- **Auditable.** Every state change is preserved as an append-only event. No data is ever overwritten.
- **Measurable.** Information loss during format conversion is quantified via the Envelope Fidelity Score (EFS), a normalized metric comparable across systems and industries.

---

## 2. Envelope Structure

A CORD envelope is a JSON document serialized as `application/cord+json`. Every envelope must conform to this specification and to the JSON Schema defined in `schema/cord-envelope.schema.json`.

### 2.1 Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `envelope_id` | string | yes | Globally unique identifier for this envelope version |
| `envelope_type` | enum | yes | `"snapshot"` or `"delta"` |
| `parent_envelope_id` | string | conditional | Required when `envelope_type` is `"delta"`. References the immediately preceding envelope version. |
| `schema_version` | string | yes | CORD schema version. Must be `"1.0.0"` for this specification. |
| `created_at` | string (ISO 8601) | yes | Timestamp of envelope creation |
| `tenant_id` | string | yes | Identifier of the receiving entity tenant |
| `channel` | enum | yes | Input channel. See §2.2. |
| `subject` | object | yes | Subject interaction data. See §2.3. |
| `lifecycle_stage` | enum | yes | Current interaction lifecycle stage. See §2.4. |
| `prospect_score` | object | no | Scoring data. See §2.5. |
| `events` | array | yes | Append-only event log. See §2.6. |
| `efs` | object | no | Envelope Fidelity Score record. See §3. |
| `extensions` | object | no | Domain-specific extension fields. |

### 2.2 Channel Values

| Value | Description |
|---|---|
| `text_messaging` | SMS or messaging platform |
| `voice` | Voice call with speech-to-text transcription |
| `web_chat` | Browser-based chat interface |
| `web_form` | Web form submission |
| `email` | Email-based interaction |
| `api` | Direct API submission |

### 2.3 Subject Object

The `subject` object contains the structured field data extracted from the AI-subject interaction.

```json
{
  "subject_id": "subj_01J9X2K4M8N3P5Q7R0S6T",
  "fields": [
    {
      "field_name": "full_name",
      "field_value": "Jordan Rivera",
      "field_type": "string",
      "confidence": 1.0,
      "extraction_source": "rule_based",
      "extracted_at": "2026-01-15T14:31:45Z"
    },
    {
      "field_name": "intent_classification",
      "field_value": "purchase_inquiry",
      "field_type": "enum",
      "confidence": 0.91,
      "extraction_source": "ml_inference",
      "extracted_at": "2026-01-15T14:32:00Z"
    }
  ],
  "contact": {
    "phone": "+1-555-0192",
    "email": "jordan.rivera@example.com"
  }
}
```

#### 2.3.1 Field Object

| Field | Type | Required | Description |
|---|---|---|---|
| `field_name` | string | yes | Field identifier. Domain-specific vocabulary. |
| `field_value` | any | yes | Extracted field value. May be string, number, boolean, array, or object. |
| `field_type` | enum | yes | `"string"`, `"number"`, `"boolean"`, `"enum"`, `"array"`, `"object"`, `"range"` |
| `confidence` | number | yes | Extraction confidence score, 0.0–1.0 |
| `extraction_source` | enum | yes | `"rule_based"`, `"ml_inference"`, `"hybrid"`, `"subject_provided"` |
| `extracted_at` | string (ISO 8601) | yes | Timestamp of extraction |

**Fields with `confidence` below the tenant-configured threshold (default: 0.70) should be flagged for human review before transmission.**

### 2.4 Lifecycle Stage Values

CORD defines five standard lifecycle stages. Tenants may define additional stages via the `extensions` field.

| Value | Description |
|---|---|
| `NEW` | Initial contact. No qualifying information collected. |
| `INQUIRY` | Subject has expressed interest and provided basic contact information. |
| `QUALIFIED` | Subject has expressed specific intent or engagement. |
| `NEGOTIATION` | Subject is actively discussing terms or parameters. |
| `CLOSED` | Interaction complete (committed or withdrawn). |

### 2.5 Prospect Score Object

```json
{
  "score": 74,
  "tier": "HOT",
  "computed_at": "2026-01-15T14:32:00Z",
  "factor_count": 6
}
```

Score tiers: `COLD` (0–24), `INTERESTED` (25–49), `WARM` (50–69), `HOT` (70–100).

The full factor-level attribution breakdown is available in the CORD certified implementation.

### 2.6 Event Log

The `events` array is an **append-only** log. Events are never deleted or modified. Each event records a state change, score update, escalation, or channel interaction.

```json
{
  "event_id": "evt_01J9X2K4M8N3",
  "event_type": "lifecycle_transition",
  "occurred_at": "2026-01-15T14:32:00Z",
  "data": {
    "from_stage": "INQUIRY",
    "to_stage": "QUALIFIED",
    "triggering_intent": "schedule_appointment"
  }
}
```

#### 2.6.1 Event Types

| Event Type | Description |
|---|---|
| `field_extracted` | A new field was added to the envelope |
| `lifecycle_transition` | Lifecycle stage changed |
| `score_changed` | Prospect score updated |
| `escalation_triggered` | Conversation escalated to human agent |
| `identity_merged` | Envelope merged with another envelope after identity resolution |
| `legacy_transmission` | Envelope transmitted to a legacy receiving system |
| `canonical_import` | Envelope created from a legacy format ingestion |

---

## 3. Envelope Fidelity Score (EFS)

The Envelope Fidelity Score is a normalized metric, 0.0–1.0, that quantifies the information loss when a CORD envelope is converted to a legacy interchange format.

- **1.0** = perfect fidelity (no information lost)
- **0.0** = total loss (all information destroyed)

EFS values are **cross-system and cross-industry comparable**. An EFS of 0.23 means the same thing whether the target system is an HL7 ADT message, a Salesforce lead, or an ACORD form.

### 3.1 EFS Record

When a CORD envelope is transmitted to a legacy system, an EFS record is attached:

```json
{
  "efs": {
    "score": 0.23,
    "computed_at": "2026-01-15T14:35:00Z",
    "target_format": "salesforce_lead_v3",
    "field_count_source": 18,
    "field_count_full_mapping": 3,
    "field_count_partial_mapping": 4,
    "field_count_no_mapping": 11,
    "severity_summary": {
      "critical": 5,
      "warning": 4,
      "informational": 9
    }
  }
}
```

### 3.2 Field Mapping Status

For each field in the source envelope, the EFS engine assigns one of three mapping statuses:

| Status | Description |
|---|---|
| `FULL` | Field has a direct mapping in the legacy format. No information loss. |
| `PARTIAL` | Field is partially representable in the legacy format. Some information lost. |
| `NONE` | Field has no mapping in the legacy format. All information lost. |

### 3.3 Severity Classification

Each field loss is classified by severity:

| Severity | Description |
|---|---|
| `CRITICAL` | High-importance field with significant loss |
| `WARNING` | Moderate loss or moderate-importance field |
| `INFORMATIONAL` | Low-impact loss |

### 3.4 EFS Computation

The EFS computation method — including the weighting model, loss coefficient formulas, and semantic compression detection algorithms — is defined in the CORD certified implementation specification.

Implementors seeking to produce EFS-compliant scores should refer to the conformance documentation. See [`conformance/conformance-tiers.md`](../conformance/conformance-tiers.md).

---

## 4. Versioning: Snapshot and Delta

### 4.1 Snapshot Envelopes

A snapshot envelope contains the complete current state of an interaction. Snapshot envelopes are generated for:
- First interaction with a new subject
- Periodic state compaction
- Legacy format ingestion (`canonical_import`)

`"envelope_type": "snapshot"` — no `parent_envelope_id` required.

### 4.2 Delta Envelopes

A delta envelope contains only the fields that changed since the previous version. Delta envelopes reference their parent via `parent_envelope_id`.

`"envelope_type": "delta"` — `parent_envelope_id` required.

```json
{
  "envelope_id": "env_02J9X3L5N9P4Q8R1S7T",
  "envelope_type": "delta",
  "parent_envelope_id": "env_01J9X2K4M8N3P5Q7R0S6T",
  "schema_version": "1.0.0",
  "created_at": "2026-01-15T15:10:00Z",
  "subject": {
    "fields": [
      {
        "field_name": "lifecycle_stage",
        "field_value": "NEGOTIATION",
        "field_type": "enum",
        "confidence": 1.0,
        "extraction_source": "rule_based",
        "extracted_at": "2026-01-15T15:09:55Z"
      }
    ]
  }
}
```

### 4.3 State Reconstruction

The current state of any interaction is reconstructed by loading the initial snapshot and sequentially applying all subsequent delta envelopes in version order.

---

## 5. Cross-Channel Identity Resolution

When the same subject interacts across multiple channels, CORD supports envelope merging via the identity resolution mechanism.

Identity resolution uses tiered match keys:
1. Exact phone number match (highest confidence)
2. Exact email address match
3. Fuzzy name + postal code match (lowest confidence)

A configurable confidence threshold (default: 0.85) determines whether a match is accepted. Confirmed matches generate an `identity_merged` event and produce a unified envelope with a single `subject_id` and merged event history.

---

## 6. Legacy Format Canonicalization

CORD supports ingesting data from legacy formats via the canonicalization pipeline. Incoming legacy data is:

1. Detected by format type (content-type headers, namespace detection, structural analysis)
2. Mapped to canonical CORD fields via a format-specific adapter
3. Validated against the CORD schema
4. Wrapped in a snapshot envelope with `envelope_type: "canonical_import"`

The original source payload is preserved in the `extensions` field.

---

## 7. Media Type

CORD envelopes use the media type:

```
application/cord+json
```

This media type is submitted for registration with the Internet Assigned Numbers Authority (IANA).

---

## 8. Schema

The normative JSON Schema for CORD envelope validation is defined in:

```
schema/cord-envelope.schema.json
```

All CORD envelopes must validate against this schema.

---

## 9. Conformance

See [`conformance/conformance-tiers.md`](../conformance/conformance-tiers.md) for the full conformance tier definitions.

---

## 10. Versioning Policy

| Version | Status |
|---|---|
| 1.0.0 | Current |

Minor versions (1.x.0) may add optional fields. Major versions (2.0.0) may introduce breaking changes. All changes are recorded in [`CHANGELOG.md`](../CHANGELOG.md).

---

## License

Apache License 2.0. See [`LICENSE`](../LICENSE).
