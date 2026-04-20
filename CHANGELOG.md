# CORD Protocol Changelog

All notable changes to the CORD specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
CORD uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] — 2026-04-15

### Added
- Initial published specification
- `application/cord+json` media type definition
- Canonical envelope structure with `envelope_id`, `envelope_type`, `parent_envelope_id`, `schema_version`, `created_at`, `tenant_id`, `channel`, `subject`, `lifecycle_stage`, `events`, `efs`, and `extensions`
- Snapshot and delta versioning with parent reference chain
- Five standard lifecycle stages: `NEW`, `INQUIRY`, `QUALIFIED`, `NEGOTIATION`, `CLOSED`
- Six channel types: `text_messaging`, `voice`, `web_chat`, `web_form`, `email`, `api`
- Field object with `field_name`, `field_value`, `field_type`, `confidence`, `extraction_source`, `extracted_at`
- Seven field types: `string`, `number`, `boolean`, `enum`, `array`, `object`, `range`
- Four extraction sources: `rule_based`, `ml_inference`, `hybrid`, `subject_provided`
- Prospect score object with tier classification (`COLD`, `INTERESTED`, `WARM`, `HOT`)
- Append-only event log with seven event types
- EFS record structure with `score`, `target_format`, field mapping counts, and severity summary
- Three field mapping statuses: `FULL`, `PARTIAL`, `NONE`
- Three severity classifications: `CRITICAL`, `WARNING`, `INFORMATIONAL`
- Three conformance tiers: `CORD-Compliant`, `CORD-Partial`, `CORD-Aware`
- JSON Schema (`cord-envelope.schema.json`)
- Industry examples: healthcare, real estate, HR/talent acquisition, delta versioning
- EFS benchmark table across seven industries

---

## Upcoming

### [1.1.0] — Planned
- Public conformance registry
- EFS benchmark publication standard
- Extended channel types (in-app, kiosk)
- Formal IANA media type registration completion
- Additional industry examples: insurance, legal, financial services, government

---

*Maintained by CORD Protocol Holdings LLC — cordspec.org*
