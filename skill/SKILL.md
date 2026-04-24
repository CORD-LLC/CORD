---
name: cord-migration
description: Migrate any AI extraction pipeline to CORD spec (cordspec.org). Use when the user mentions CORD, CORDspec, Envelope Fidelity Score, EFS, data fidelity, AI-to-legacy translation loss, or wants to wrap AI output in a structured envelope with measurable fidelity. Also trigger when the user wants to measure data loss, compare AI extraction vs downstream storage, add fidelity scoring, or build a loss report. If the user has an existing service that parses, extracts, or transforms data and wants observability around data loss, this is the skill.
---

# CORD Migration Skill — Official

This skill migrates existing AI extraction pipelines to CORD spec v1.0. It takes a codebase that extracts structured data (via LLM, NLP, or rule-based parsing) and pushes it into a downstream system, then wraps that pipeline with a CORD envelope, field mapping table, EFS computation, SHA-256 envelope integrity, and optional conformance validation.

The goal is zero-friction adoption. The user should go from "I have a parser that extracts fields and sends them somewhere" to "my parser now produces CORD-compliant envelopes with measurable fidelity" in one session.

## Before you start

Read the user's existing codebase to understand three things:

1. **What does the AI extract?** Find the extraction output schema — JSON schema, TypeScript interface, Python dataclass, LLM prompt, or just the shape of the returned object. This becomes the basis for `fields` (the rich representation).

2. **What does the downstream system accept?** Find where extracted data gets written, sent, or stored — database writes, API calls, webhook payloads, CRM inserts, file outputs. The fields that survive this step become `legacy_output`.

3. **What language and framework is the pipeline built in?** The CORD envelope builder, field mapping table, and EFS scorer need to match the existing codebase's language, style, module system, and conventions exactly.

## Migration steps

### Step 1: Catalog every field

Build two lists:

**Source fields** (everything the AI or parser extracts):
- Read the LLM prompt, extraction function, or parsing logic
- List every field that gets extracted, including intermediate or discarded ones
- Note the type, source (LLM, regex, structured input), and typical confidence level

**Target fields** (everything the downstream system stores):
- Read the database schema, API payload, or output format
- List every field that gets written downstream
- Note any transformations (flattening arrays, truncating text, type coercion)

### Step 2: Build the field mapping table

Create a mapping table connecting source fields to target fields. For each source field, classify it:

- **FULL**: Maps directly to a target field with no information loss. Value, type, and structure preserved exactly. Fidelity coefficient = 1.0.
- **PARTIAL**: Maps to a target field but loses information (array flattened, object merged into text blob, value truncated, type coerced). Assign a `partial_coefficient` between 0.5 and 0.9. The CORD v1.0 spec requires this coefficient to be >= 0.5 — implementations MUST NOT use a value below 0.5.
- **NONE**: No corresponding target field. Data extracted by the AI but silently dropped. Fidelity coefficient = 0.0.

**Confidence-based coefficient selection (recommended):** When the AI provides confidence scores on source fields, use them to refine the PARTIAL coefficient:
- confidence < 0.5 → coefficient = 0.5 (floor)
- confidence >= 0.95 → coefficient = 0.9 (ceiling)
- otherwise → linear interpolation: `0.5 + ((confidence - 0.5) / (0.95 - 0.5)) * 0.4`

Record the resolved coefficient in `partial_coefficient` on the field mapping entry for auditability.

**Field weights** reflect operational importance:
- Weight 3: Critical to downstream workflow (customer name, address, primary identifier, phone)
- Weight 2: Affects decisions or routing (plan type, priority, status flags, amounts)
- Weight 1: Provides context (metadata, timestamps, secondary identifiers)
- Weight 0.5: Rarely relevant (boilerplate text, internal source system IDs)

The mapping table is a static data structure in code, not computed at runtime.

### Step 3: Update the AI prompt (if LLM-based)

If the pipeline uses an LLM, update the system prompt to return two things:

1. **`fields`** (or `rich_fields`): An array of every piece of information the LLM can find, each with `name`, `value`, `type`, `confidence`, and `source`. This is the "everything the AI saw" side.

2. **`legacy_output`**: The flat object matching the existing downstream schema. Keep all existing extraction rules, validation logic, and field-specific instructions exactly as they are. Nothing about the downstream output changes.

The prompt should instruct the LLM to extract aggressively for `fields` (even fields the downstream system doesn't store) while keeping `legacy_output` identical to current behavior. This ensures zero regression while gaining observability.

If the pipeline does NOT use an LLM (rule-based parsing, regex extraction), skip the prompt update. Create a separate function that takes the parser's intermediate data and produces the `fields` array by walking all available source fields. The `legacy_output` is the existing output, unchanged.

### Step 4: Create the envelope builder

Write a function that takes `(fields, legacyOutput, lossReport, metadata)` and returns a CORD v1.0 envelope:

```json
{
  "cord_version": "1.0",
  "envelope_id": "<UUID v4>",
  "envelope_type": "snapshot",
  "version": 1,
  "parent_envelope_id": null,
  "created_at": "<ISO 8601 UTC>",
  "domain": "<user's domain>",
  "source_system": "<name of AI/extraction system>",
  "target_system": "<name of downstream system>",
  "fields": [
    {
      "name": "field_name",
      "value": "<extracted value>",
      "type": "text|number|date|phone|email|currency|boolean|code|structured|text_array|code_array",
      "confidence": 0.95,
      "source": "nlp|structured|inferred|user_input"
    }
  ],
  "legacy_output": { "<existing flat schema>" },
  "loss_report": {
    "efs": 0.63,
    "field_mappings": [
      {
        "field": "field_name",
        "status": "FULL|PARTIAL|NONE",
        "target": "target_field_name or null",
        "partial_coefficient": 0.72,
        "note": "human-readable explanation"
      }
    ]
  },
  "event_log": [],
  "x_cord_digest": "sha256:<hex>"
}
```

The `event_log` is optional. Include it if the pipeline has natural events to log (webhook received, AI extraction completed, validation passed). Omit if there's nothing meaningful.

### Step 5: Create the EFS scorer

Write a function that takes the field mapping table and the list of fields present in a given extraction, then computes the EFS:

```
For each field present:
  Look up its mapping entry
  Get the fidelity coefficient:
    FULL    → 1.0
    PARTIAL → partial_coefficient (default 0.5, MUST be >= 0.5 per spec)
    NONE    → 0.0
  Get the weight from the mapping table

  fidelitySum += weight * coefficient
  totalWeight += weight

EFS = round(fidelitySum / totalWeight, 2)
```

If confidence propagation is enabled, derive the PARTIAL coefficient from the AI's confidence score on that field (see Step 2). Store the resolved coefficient in `partial_coefficient` on the mapping entry.

Produce a summary: total fields, count of FULL/PARTIAL/NONE, percentage lost.

### Step 6: Compute SHA-256 digest

After building the envelope, compute a SHA-256 hash of the canonical JSON serialization (excluding the `x_cord_digest` field itself):

```
1. Remove x_cord_digest from the envelope dict
2. Serialize to canonical JSON (sorted keys, no whitespace, ensure ASCII)
3. Compute SHA-256 of the UTF-8 encoded string
4. Store as: envelope.x_cord_digest = "sha256:" + hex(digest)
```

This enables tamper detection at the data layer without external signing infrastructure.

### Step 7: Integrate into the pipeline

Find the point where the extraction result is returned, sent, or stored. Add CORD envelope construction there:

1. After the AI/parser produces its output, call the envelope builder
2. Compute the loss report and EFS
3. Compute the SHA-256 digest
4. Attach the envelope to the response, or store alongside existing output
5. Log the EFS

Do NOT change the existing output format in a breaking way. The CORD envelope is additive. Common patterns:

- **API response**: Add a `cord` key alongside the existing `data` key
- **Database write**: Store envelope in a separate collection/table or as a JSON field
- **File output**: Write as a sidecar file

Support content negotiation if the pipeline serves an API: if the caller sends `Accept: application/cord+json`, return just the envelope.

### Step 8: Add conformance validation (optional)

Run the three-category conformance check against the produced envelope:

1. **Envelope Structure**: All required fields present and correctly typed
2. **EFS Reporting**: loss_report with valid EFS and complete field_mappings
3. **Mapping Completeness**: Every source field has a corresponding mapping entry

Classify the result into one of four tiers:
- **CORD-Compliant**: Passes all three categories, includes event log and per-field notes
- **CORD-Partial**: Passes categories 2 and 3, may omit event log or notes
- **CORD-Compatible**: Produces core envelope fields without full EFS
- **Non-Compliant**: No CORD envelope produced

Log the conformance tier alongside the EFS.

### Step 9: Add EFS to logging and health checks

Update the health check endpoint (if one exists) to include:
```json
{
  "cord_version": "1.0",
  "conformance": "CORD-Compliant",
  "engine_version": "1.1.0"
}
```

Add EFS to request logging:
```
✅ Parsed in 340ms — customer=Angela Gil de Montes, EFS=0.63 [15 FULL, 2 PARTIAL, 17 NONE] digest=sha256:e4a7c3...
```

## Language-specific guidance

### Node.js / JavaScript / TypeScript
- Use `crypto.randomUUID()` (Node 19+) or `uuid` package for envelope IDs
- Use `crypto.createHash('sha256')` for digest computation
- Match ESM/CJS to existing codebase
- Put mapping table in a separate module (e.g., `src/cord.js`)
- Export `buildEnvelope()`, `computeEFS()`, `computeDigest()`, `validateConformance()` as named exports

### Python
- Use `uuid.uuid4()` for envelope IDs
- Use `hashlib.sha256()` for digest computation
- Use `json.dumps(obj, sort_keys=True, separators=(',',':'), ensure_ascii=True)` for canonical serialization
- Use `datetime.now(timezone.utc).isoformat()` for timestamps (not `utcnow()`)
- Put mapping table in a separate module (e.g., `cord.py`)

### Go
- Use `github.com/google/uuid` for envelope IDs
- Use `crypto/sha256` for digest computation
- Define envelope as a struct with JSON tags
- Put mapping table in a `var` block with `map[string]FieldMapping`

### Other languages
Match existing codebase conventions. The CORD envelope is a JSON object. Any language that can produce JSON can produce a CORD envelope.

## What NOT to do

- Do NOT change existing extraction logic, validation rules, or downstream output format. The CORD layer is purely additive.
- Do NOT ask the LLM to compute the EFS. EFS is a property of the schema mapping, not the extraction. Compute it server-side from the static mapping table.
- Do NOT use a PARTIAL coefficient below 0.5. The spec floor is 0.5.
- Do NOT implement delta envelopes or version chaining unless the user specifically asks. Start with snapshots only.
- Do NOT add database writes for envelope storage unless the user asks. First milestone is envelope construction and EFS logging, not persistence.
- Do NOT over-engineer weights. Start with simple values (0.5, 1, 2, 3) and adjust after seeing real results.
- Do NOT skip the SHA-256 digest. It's one line of code and gives every envelope tamper detection for free.

## Conformance target

After migration, the system should meet CORD-Compliant (full conformance):
- Valid envelope with all required fields
- Correct snapshot classification
- Loss report with EFS per translation
- FULL / PARTIAL / NONE status on every source field with partial_coefficient where applicable
- SHA-256 x_cord_digest for envelope integrity
- Serialized as `application/cord+json` when requested

For lighter integration, CORD-Partial is acceptable:
- Valid envelope with required fields
- Loss report with an EFS value
- Event log, version chaining, and digest can be omitted

## After migration

Surface these questions to the user:

1. "Your EFS is X%. Here are the fields being dropped. Do any of these matter enough to add to your downstream schema?"
2. "Do you want to store envelopes for trend analysis (average EFS per provider, which fields get consistently dropped)?"
3. "Do you want to expose EFS to end users as a fidelity metric on their dashboard?"
4. "Do you want to run conformance validation on every envelope and log the tier?"

These are next steps, not part of the initial migration.

## Reference

Full CORD v1.0 specification: https://cordspec.org
Spec repo: https://github.com/CORD-LLC/CORD
Engine repo: https://github.com/CORD-LLC/cord-engine
