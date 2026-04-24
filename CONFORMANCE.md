# CORD Conformance Model

CORD defines four conformance tiers assessed against three validation categories. Conformance is evaluated at the level of a complete translation pipeline — the originating AI system, the translation process, and the target legacy system.

## Validation Categories

CORD-compliant systems MUST pass all three validation categories.

### Category 1 — Envelope Structure

All required top-level fields present and correctly typed:

- `cord_version` = `"1.0"`
- `envelope_id` is a valid UUID
- `envelope_type` is `"snapshot"` or `"delta"`
- `version` is an integer >= 1
- `parent_envelope_id` is null for snapshots, non-null for deltas
- `created_at` is a valid ISO 8601 UTC timestamp
- `domain`, `source_system`, `target_system` are non-empty strings
- `fields` is an array of valid field objects (name, value, type, confidence, source)
- `legacy_output` is an object
- `loss_report` is an object

### Category 2 — EFS Reporting

- `loss_report.efs` is a float in [0.0, 1.0]
- `loss_report.field_mappings` is an array
- Every field mapping entry has `field` (string) and `status` (FULL, PARTIAL, or NONE)
- PARTIAL entries SHOULD include `partial_coefficient` in [0.5, 0.9]
- Implementations MUST NOT use a PARTIAL coefficient below 0.5

### Category 3 — Mapping Completeness

- Every field in the `fields` array has a corresponding entry in `loss_report.field_mappings`
- No silent omissions — if the AI extracted it, the loss report accounts for it
- A delta envelope with an empty `fields` array is valid when the purpose is to record event log entries without modifying field state

## Conformance Tiers

| Tier | Requirements |
|---|---|
| **CORD-Compliant** | Passes all three categories. Event log present. Per-field notes on mapping entries. |
| **CORD-Partial** | Passes Category 2 and Category 3. May omit event log, per-field notes, or version chaining. |
| **CORD-Compatible** | Produces or consumes core CORD envelope fields without full EFS reporting. Appropriate for systems in active migration. |
| **Non-Compliant** | Does not produce CORD-structured envelopes. Does not participate in the CORD ecosystem. Non-compliant indicates the system does not participate — not that it is defective. |

## Self-Assessment

Self-assessment against the three validation categories is sufficient to declare conformance under CORD v1.0. A formal certification program with automated test suites and a public conformance registry is under development.

## Declaring Conformance

Standard language for product documentation:

> "This system produces CORD-compliant envelopes (v1.0)."

Include your conformance tier in API responses:

```json
{
  "cord_version": "1.0",
  "conformance": "CORD-Compliant"
}
```
