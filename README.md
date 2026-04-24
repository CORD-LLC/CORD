# CORD — Canonical Object for Relational Data

**The open protocol for preserving AI-generated data across systems.**

CORD sits between AI systems and legacy software, ensuring structured data is preserved, translated, and measurable across every interaction.

[![Spec](https://img.shields.io/badge/spec-v1.0-2563eb)](https://cordspec.org)
[![License](https://img.shields.io/badge/license-Apache%202.0-green)](LICENSE)
[![Try It](https://img.shields.io/badge/tester-live-16a34a)](https://cordspec.org/tester)

## What CORD does

AI systems extract rich, structured data. Legacy systems — CRMs, EHRs, DMSs, ATSs — compress it into flat schemas. Information is silently lost. CORD makes that loss visible, measurable, and auditable.

Every CORD envelope contains:

- **Full-fidelity fields** — everything the AI extracted, with confidence scores and source attribution
- **Legacy output** — the flat representation that actually gets written to the downstream system
- **Loss report** — per-field mapping status (FULL / PARTIAL / NONE) and an Envelope Fidelity Score (EFS)
- **Integrity** — SHA-256 digest for tamper detection without external signing infrastructure

## Envelope Fidelity Score (EFS)

EFS is a normalized metric on [0.0, 1.0] that quantifies how much information survives translation.

```
EFS = Σ(weight × coefficient) / Σ(weight)

FULL    → 1.0
PARTIAL → [0.5, 0.9] based on confidence propagation
NONE    → 0.0
```

An EFS of 0.63 means 37% of the structured data the AI extracted was lost in translation.

## Quick example

```json
{
  "cord_version": "1.0",
  "envelope_id": "a3f9c201-7d42-4b1a-9e0f-2c88d3f5ab01",
  "envelope_type": "snapshot",
  "version": 1,
  "parent_envelope_id": null,
  "created_at": "2025-09-14T10:22:00Z",
  "domain": "healthcare",
  "source_system": "intake-ai-v2",
  "target_system": "epic-ehr-adapter",
  "fields": [
    { "name": "chief_complaint", "value": "chest pain, onset 3h, radiating to left arm", "type": "text", "confidence": 0.97, "source": "nlp" },
    { "name": "icd10_codes", "value": ["R07.9", "R07.4", "I20.9"], "type": "code_array", "confidence": 0.84, "source": "classifier" },
    { "name": "pain_score", "value": 7, "type": "integer", "confidence": 0.92, "source": "nlp" }
  ],
  "legacy_output": {
    "chief_complaint": "chest pain",
    "icd10_primary": "R07.9",
    "pain_score": "7"
  },
  "loss_report": {
    "efs": 0.61,
    "field_mappings": [
      { "field": "chief_complaint", "status": "PARTIAL", "partial_coefficient": 0.5, "note": "Onset and radiation detail truncated" },
      { "field": "icd10_codes", "status": "PARTIAL", "partial_coefficient": 0.62, "note": "Primary code only; 2 of 3 lost" },
      { "field": "pain_score", "status": "FULL", "note": "Value preserved; type coerced to string" }
    ]
  },
  "x_cord_digest": "sha256:e4a7c3f901b2d844a1e09f3bc20d1847..."
}
```

## Conformance

CORD defines four conformance tiers based on three validation categories:

| Category | Validates |
|---|---|
| Envelope Structure | Required fields, types, snapshot/delta rules |
| EFS Reporting | loss_report with valid EFS and field_mappings |
| Mapping Completeness | Every source field has a mapping entry |

| Tier | Requirement |
|---|---|
| **CORD-Compliant** | All three categories pass; event log and notes present |
| **CORD-Partial** | Categories 2+3 pass; may omit event log or notes |
| **CORD-Compatible** | Core envelope fields without full EFS |
| **Non-Compliant** | No CORD envelope produced |

## Resources

- **Specification**: [cordspec.org](https://cordspec.org)
- **Live Tester**: [cordspec.org/tester](https://cordspec.org/tester)
- **Reference Engine**: [cord-engine](https://github.com/CORD-LLC/cord-engine)
- **Migration Skill**: See [`/skill`](skill/) for Claude Code integration

## License

Apache 2.0 — see [LICENSE](LICENSE).
