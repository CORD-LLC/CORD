# CORD — Canonical Object for Relational Data

**An open protocol standard for AI-generated interaction data.**

CORD defines how conversational AI agents structure, version, and transmit interaction data — and how to measure what gets lost when that data is translated into legacy systems.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v1.0.0-green.svg)](docs/spec.md)
[![Media Type](https://img.shields.io/badge/media%20type-application%2Fcord%2Bjson-orange.svg)](docs/spec.md#media-type)

---

## The Problem

Every day, AI agents collect rich, structured interaction data — typed fields, arrays, confidence scores, intent classifications, conversation histories. Then they hand that data off to legacy systems: CRMs, EHRs, ATS platforms, case management tools.

Those legacy systems were built before AI. They don't understand typed enumerations, arrays, or confidence scores. They have a text field. Maybe two.

**The data gets compressed. Information is destroyed. Nobody measures it.**

CORD fixes that.

---

## What CORD Does

CORD is a protocol standard that defines:

1. **A canonical envelope format** — a structured, versioned container for AI-generated interaction data, serialized as `application/cord+json`
2. **Snapshot and delta versioning** — every interaction state is preserved, reconstructable, and auditable
3. **Dual-format output** — full-fidelity CORD output alongside legacy-compatible output, simultaneously
4. **The Envelope Fidelity Score (EFS)** — a normalized metric (0.0–1.0) quantifying information loss during format conversion, comparable across systems and industries
5. **Conformance tiers** — a certification framework so receiving systems can declare their CORD compliance level

---

## Who CORD Is For

CORD is domain-agnostic. It applies anywhere AI agents collect structured data for transmission to legacy-format receiving systems:

| Industry | AI Agent | Legacy Receiving System |
|---|---|---|
| Healthcare | Telehealth intake agent | HL7 v2 ADT / fax referral |
| Real Estate | Buyer qualification agent | MLS lead form |
| Insurance | Claims intake agent | ACORD form |
| Legal | Client intake agent | Case management system |
| Financial Services | Wealth management agent | CRM contact record |
| HR / Talent | Candidate screening agent | ATS import record |
| Government | Benefits intake agent | Legacy case management |
| Automotive | Sales AI agent | CRM / DMS lead record |

---

## Quick Start

### A minimal CORD envelope

```json
{
  "envelope_id": "env_01J9X2K4M8N3P5Q7R0S6T",
  "envelope_type": "snapshot",
  "schema_version": "1.0.0",
  "created_at": "2026-01-15T14:32:00Z",
  "tenant_id": "tenant_abc123",
  "channel": "web_chat",
  "subject": {
    "fields": [
      {
        "field_name": "full_name",
        "field_value": "Jordan Rivera",
        "field_type": "string",
        "confidence": 1.0,
        "extraction_source": "rule_based"
      },
      {
        "field_name": "intent_classification",
        "field_value": "purchase_inquiry",
        "field_type": "enum",
        "confidence": 0.91,
        "extraction_source": "ml_inference"
      }
    ]
  },
  "lifecycle_stage": "QUALIFIED",
  "events": [],
  "efs": {
    "score": null,
    "computed_at": null,
    "note": "EFS computed at transmission time against target legacy format"
  }
}
```

See [`examples/`](examples/) for full envelope examples across multiple industries.

---

## Repository Structure

```
cord-spec/
├── README.md                  ← You are here
├── LICENSE                    ← Apache 2.0
├── CHANGELOG.md               ← Version history
├── docs/
│   ├── spec.md                ← Full CORD protocol specification
│   └── efs.md                 ← Envelope Fidelity Score reference
├── schema/
│   └── cord-envelope.schema.json   ← JSON Schema for envelope validation
├── conformance/
│   └── conformance-tiers.md   ← CORD conformance tier definitions
└── examples/
    ├── healthcare-snapshot.json
    ├── real-estate-snapshot.json
    ├── hr-screening-snapshot.json
    └── delta-example.json
```

---

## The Envelope Fidelity Score

The EFS is a normalized score between **0.0** (total information loss) and **1.0** (perfect fidelity) that quantifies what is destroyed when a CORD envelope is converted to a legacy format.

EFS values are **cross-system comparable**. An EFS of 0.19 means the same thing whether the legacy system is an HL7 v2 message, a Salesforce lead record, or an ACORD form.

Example EFS benchmarks observed across industries:

| Legacy Format | Typical EFS | Information Lost |
|---|---|---|
| HL7 v2 ADT (telehealth) | ~0.19 | ~81% |
| MLS lead form (real estate) | ~0.26 | ~74% |
| ACORD form (insurance claims) | ~0.15 | ~85% |
| ATS import record (HR) | ~0.23 | ~77% |
| CRM contact record (financial) | ~0.23 | ~77% |

> The EFS computation method is defined in the CORD certified implementation specification. See [`docs/efs.md`](docs/efs.md).

---

## Conformance

CORD defines three conformance tiers for receiving systems and implementations. See [`conformance/conformance-tiers.md`](conformance/conformance-tiers.md).

---

## License

CORD is published under the [Apache License 2.0](LICENSE). The protocol specification, schema, and examples are free to implement and build upon.

The `application/cord+json` media type is submitted for IANA registration.

---

## Contributing

CORD is maintained by CORD Protocol Holdings LLC. Contributions, issue reports, and implementation feedback are welcome via GitHub Issues and Pull Requests.

**Website:** [cordspec.org](https://cordspec.org)
**Contact:** hello@cordspec.org
