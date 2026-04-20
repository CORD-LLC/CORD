# CORD Conformance Tiers

**CORD Protocol v1.0.0**

---

## Overview

CORD defines three conformance tiers for data interchange systems. Conformance tiers allow receiving systems, AI platforms, and integration vendors to declare their level of CORD support in a standardized, verifiable way.

---

## Conformance Tiers

### Tier 1 — CORD-Compliant

A CORD-Compliant system fully implements the CORD protocol, including:

- Accepting and parsing `application/cord+json` envelopes at full field fidelity
- Preserving all typed fields, arrays, confidence scores, and AI-derived metadata
- Producing or consuming snapshot and delta envelope chains with correct parent references
- Computing and attaching EFS records using the certified computation method
- Issuing field-level loss reports with per-field mapping status and severity classification
- Passing all CORD conformance test suite scenarios

**Declared as:** `"cord_conformance": "compliant"`

---

### Tier 2 — CORD-Partial

A CORD-Partial system implements a subset of the CORD protocol. Specifically:

- Accepts `application/cord+json` envelopes
- Preserves a defined subset of CORD fields (documented in the system's CORD integration profile)
- May not support snapshot/delta versioning
- May not produce full EFS records, but must expose a declared field mapping profile
- Passes a defined subset of CORD conformance test suite scenarios

**Declared as:** `"cord_conformance": "partial"` with an accompanying field mapping profile.

---

### Tier 3 — CORD-Aware

A CORD-Aware system does not natively process `application/cord+json` but:

- Accepts data sourced from CORD envelopes via a declared field mapping adapter
- Documents which CORD fields map to its native fields
- Allows the EFS to be computed externally and attached to outbound transmissions

**Declared as:** `"cord_conformance": "aware"` with a published field mapping adapter spec.

---

## Conformance Declaration

Systems declare their conformance tier in their integration documentation and optionally in API response headers:

```
X-CORD-Conformance: compliant
X-CORD-Schema-Version: 1.0.0
```

---

## Conformance Testing

The CORD conformance test suite defines scenarios that verify correct envelope construction, versioning, EFS computation, and field-level loss reporting.

The full test suite and certified test runner are available to registered implementors. Contact hello@cordspec.org to register as a CORD conformance tester.

---

## Why Conformance Tiers Matter

Without a conformance framework, "CORD support" is undefined. A system could claim compatibility while discarding 90% of the data.

The conformance tiers give:

- **AI platforms** a clear signal about how much fidelity a receiving system will preserve
- **Enterprises** a benchmark for evaluating integration vendors
- **Receiving systems** a structured path to improving their CORD support over time

---

## Roadmap

Future CORD versions will introduce:
- A public conformance registry where certified systems can list their tier and field mapping profile
- Automated conformance badge generation for GitHub and documentation sites
- EFS benchmark publication standards for Tier 1 and Tier 2 systems

---

*For questions about conformance or to begin the certification process, contact hello@cordspec.org.*
