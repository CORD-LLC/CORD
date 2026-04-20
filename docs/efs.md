# Envelope Fidelity Score (EFS) Reference

**CORD Protocol v1.0.0**

---

## What Is the EFS?

The Envelope Fidelity Score (EFS) is a normalized metric that quantifies the information loss that occurs when a CORD envelope is converted to a legacy interchange format.

**Range:** 0.0 (total information loss) to 1.0 (perfect fidelity)

The EFS answers a specific question:

> *"Of all the structured information an AI agent collected during this interaction, how much survived the translation into the receiving system?"*

---

## Why EFS Matters

Before EFS, there was no standard way to measure AI-to-legacy data degradation. Organizations had no visibility into how much information was being silently destroyed every time an AI agent handed off to a CRM, EHR, ATS, or case management system.

EFS changes that. It gives organizations:

- **Visibility** — a quantified measurement of information loss per transmission
- **Comparability** — EFS values are consistent and comparable across vendors, industries, and legacy formats
- **Accountability** — receiving systems can publish their EFS benchmarks, making data loss a visible, measurable property of their integration

---

## EFS Is Cross-System Comparable

This is the defining property of EFS. Because it uses a standardized computation methodology across all CORD-compliant implementations:

- An EFS of 0.23 from a healthcare AI agent means the same thing as an EFS of 0.23 from an HR screening agent
- An EFS of 0.19 for an HL7 v2 target is directly comparable to an EFS of 0.31 for a Salesforce lead target
- Organizations can benchmark their legacy systems against industry EFS averages

---

## EFS Benchmarks by Industry

The following representative EFS values have been observed across industries:

| Domain | AI Agent | Legacy Target | Observed EFS | Loss |
|---|---|---|---|---|
| Healthcare | Telehealth intake | HL7 v2 ADT | ~0.19 | ~81% |
| Real Estate | Buyer qualification | MLS lead form | ~0.26 | ~74% |
| Insurance | Claims intake | ACORD form | ~0.15 | ~85% |
| Legal | Client intake | Case management | ~0.24 | ~76% |
| Financial Services | Wealth management | CRM contact | ~0.23 | ~77% |
| HR / Talent | Candidate screening | ATS import | ~0.23 | ~77% |
| Government | Benefits intake | Legacy case mgmt | ~0.17 | ~83% |

These values reflect the structural limitations of legacy formats, not the quality of any particular implementation.

---

## What Drives EFS?

EFS is determined by the structural gap between the CORD envelope (AI-native, richly typed) and the legacy target format (limited fields, untyped text).

Three categories of information loss are recognized in CORD:

### Type Compression
A typed or enumerated field (with an associated confidence score and type constraint) is serialized into an untyped free-text field. The structure, type safety, and associated metadata are lost — even if the raw value is partially preserved.

*Example: An intent classification of `"purchase_inquiry"` with confidence 0.91 becomes the string "interested in buying" in a notes field.*

### Array-to-Scalar Compression
An array of structured elements is collapsed to a single scalar value. All but one element, and all structural relationships between elements, are lost.

*Example: A 12-element skills inventory becomes "Python, JavaScript, and more" in a resume notes field.*

### Relationship Compression
A field that participates in a structured relationship (e.g., a score linked to a detailed factor breakdown) is transmitted without its related fields. The score arrives at the receiving system, but the explanation does not.

*Example: A prospect score of 74 arrives in the CRM without the 6-factor attribution breakdown that explains it.*

---

## Field-Level Loss Report

Each EFS computation produces a field-level loss report attached to the envelope. The report includes:

- Total field count in the source envelope
- Count of fields with full, partial, and no mapping
- The aggregate EFS score
- A per-field detail array with:
  - Field name
  - Mapping status (`FULL`, `PARTIAL`, `NONE`)
  - Severity classification (`CRITICAL`, `WARNING`, `INFORMATIONAL`)
  - Description of data lost (if any)

---

## EFS in the CORD Envelope

The EFS record is attached to a CORD envelope at transmission time:

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

---

## Publishing Your EFS Benchmark

Receiving systems are encouraged to publish their EFS benchmark — the typical EFS observed when CORD envelopes are transmitted to their format. This gives upstream AI systems and operators a standardized measure of how well your system preserves AI-collected data.

To publish an EFS benchmark:
1. Run a CORD-compliant EFS computation against a representative sample of envelopes targeting your format
2. Record the median and range of EFS values observed
3. Publish as part of your integration documentation

---

## EFS Computation Specification

The full EFS computation methodology — including the weighting model, loss coefficient formulas for each compression category, and the aggregate score formula — is defined in the CORD certified implementation specification.

Implementations that produce EFS scores must conform to the certified computation method to ensure cross-system comparability. See [`conformance/conformance-tiers.md`](../conformance/conformance-tiers.md) for conformance requirements.

**Contact:** hello@cordspec.org for certified implementation access.
