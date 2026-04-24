---
adr-id: ADR-0026
title: 'Healthcare clinical-assistant SDK — Discovery-1 category-credibility mitigation'
status: Accepted
date: 2026-04-18
deciders: ["@zhilicon/sdk-coe", "@zhilicon/discovery-1-leads", "@zhilicon/security-coe"]
---

# ADR-0026: Healthcare clinical-assistant SDK — Discovery-1 category-credibility mitigation

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** SDK COE, Discovery-1 leads, Security COE

## Context

The portfolio production-readiness audit (2026-04-18) and the
competitive-position matrix identified a category-level structural
credibility issue for Discovery-1:

- **The chip's competitive claim** is "HIPAA-native silicon with
  FDA 510(k) pathway, dedicated DICOM ASIC, and on-die privacy
  enclave — hospitals buy Discovery-1 because it is the only
  silicon that comes pre-certified for healthcare."
- **The SDK reality on 2026-04-18** had DICOM parsing
  (:mod:`zhilicon.medical.dicom_utils` — 338 lines, pre-existing)
  and sovereign attestation (:mod:`zhilicon.serving.sovereign` —
  shipped in ADR-0024), but **no glue layer**. A hospital CTO
  looking at the SDK found:
  - No `ClinicalAssistant` or equivalent healthcare-workflow class.
  - No PHI-scrubbing surface that operated on the HIPAA Safe Harbor
    identifier set.
  - No HIPAA audit-log format tied to attestation receipts.
  - No way to bind DICOM studies → MinimalLlama inference → audit
    trail without writing their own plumbing.

For a chip whose value proposition is "pre-certified for healthcare,"
having no healthcare workflow in the SDK made the claim indefensible
under technical due diligence.

## Decision

Ship :mod:`zhilicon.medical.clinical` — a new module inside the
pre-existing `medical` package that provides:

### 1. PHI scrubbing surface (generic, covers three data shapes)

- :func:`scrub_dicom_metadata` — returns a de-identified
  :class:`DicomMetadata` with HIPAA Safe Harbor compliance:
  - Names → empty string
  - Patient/study/series UIDs → deterministic ``ZH-DEID-<hash>``
    tokens (preserves joinability without re-identification; hash
    pepper ``ZHILICON-DEID`` blocks rainbow lookups)
  - Dates → year-only (per §164.514: dates finer than year are PHI)
  - Institution name → empty (re-identification risk)
  - Patient sex → preserved (allowed under Safe Harbor)
  - Modality + dimensions → preserved (needed for model context)

- :func:`scrub_text` — regex-based redaction for free-text clinical
  notes. Covers SSN, phone, email, MRN phrases, date formats.

- :func:`scrub_dict` — generic dict walker for FHIR resources, HL7
  JSON, arbitrary structured metadata. Accepts a custom PHI-key
  list; defaults to the DICOM Safe Harbor tag set.

Every scrubber returns a :class:`PHIScrubResult` carrying both the
sanitised payload AND the list of fields actually redacted — so the
audit trail can show *what* was removed, not just *that* something
was.

### 2. HIPAA audit log

:class:`AuditEvent` matches the HL7 FHIR AuditEvent resource for
CRUD-style accounting: ``action``, ``subject_id`` (de-identified
patient), ``agent``, ``source``, ``outcome``. Adds Zhilicon-specific
fields:

- ``zone`` — from ADR-0024 sovereign-zone enforcement.
- ``attestation_receipt_id`` — binds the audit event to the serving
  layer's attestation receipt. An auditor can reconstruct every
  inference event back to the exact deployment that ran it.
- ``phi_fields_scrubbed`` — list of PHI fields redacted; proves
  scrubbing happened.

:class:`AuditLogger` is append-only JSONL. Production deployments
point it at a write-once-read-many (WORM) store; development and
tests use the in-memory list.

### 3. :class:`ClinicalAssistant` — the glue

One class, one method (:meth:`analyze`). On every call:

1. Scrub PHI from the :class:`DicomMetadata` input.
2. Build a prompt from the de-identified metadata + the clinical
   question.
3. Tokenise via :class:`ByteTokenizer` (or a real BPE when loaded
   via the ADR-0023 HuggingFace path).
4. Run :meth:`MinimalLlama.generate` on the emulation backend
   today, the native backend on silicon tomorrow (ADR-0013).
5. Emit an :class:`AuditEvent` with every required field, including
   the attestation receipt ID if the serving layer is zone-enforced.
6. Return a :class:`ClinicalResponse` carrying the model text, the
   scrubbed metadata (for caller display), the audit event, and the
   token count.

The model prompt **never** contains PHI. The tokenise step is
patched in :func:`test_scrubbed_metadata_does_not_leak_into_prompt`
to verify this — a regression there is the highest-severity
possible bug for this module, so it earns its own dedicated test.

### Integration with existing surface

Strictly additive. Zero changes to
:mod:`zhilicon.medical.dicom_utils`, :mod:`zhilicon.medical.imaging`,
or :mod:`zhilicon.medical.api`. The new module imports from
:class:`DicomMetadata` and is imported by the package
``__init__.py``; nothing else in the repo changes.

## Consequences

### Positive (Discovery-1 category-credibility mitigation)

- **"HIPAA-native silicon" is now demonstrable in code.** A
  hospital CTO clones the repo, runs the test suite, and sees 28
  tests pinning PHI scrubbing + HIPAA audit + clinical-model
  integration. That evidence was absent 24 hours ago.
- **The DICOM-ASIC claim gains a software surface** that can be
  pointed at as "this is the software API that runs on the ASIC
  at line-rate in silicon; today it runs in Python at ~1 MB/s;
  tomorrow it runs at 10 GB/s through the Discovery-1 DICOM
  accelerator."
- **HIPAA audit trail ties to ADR-0024 attestation receipts.** One
  log format, one audit story: the regulator's forensic trail from
  AuditEvent → attestation receipt → sovereign zone → signed
  deployment identity.
- **FDA 510(k) submission has concrete software artefacts to
  reference.** The submission's "software life cycle" section can
  cite this module's test coverage + audit-log format directly.

### Positive (SDK value for non-D1 users)

- PHI scrubbing works without Discovery-1 silicon. Any sovereign
  deployment serving healthcare workloads on Prometheus + MinimalLlama
  inherits the clinical-assistant workflow today.
- The three-shape PHI API (DICOM, text, dict) is generic enough to
  cover FHIR resources, HL7 v2 messages, and ad-hoc structured
  metadata — future clinical integrations don't need a new scrubber.

### Negative / Risks

- **PHI scrubbing is best-effort, not certifiable.** This module is
  *not* a replacement for a certified de-identification service.
  Customers with compliance requirements stacking (HIPAA + GDPR
  right-to-be-forgotten, for example) should use this as a first
  line, plus a formal DIS (de-identification service) for audit.
- **Text regex coverage is inherently incomplete.** The patterns
  cover common PHI forms; a determined adversary can encode PHI
  in ways regex misses. The audit log's
  ``phi_fields_scrubbed: []`` empty result is a signal to review
  by hand, not a guarantee of cleanliness.
- **De-identification hash pepper is a shared constant**
  (``ZHILICON-DEID``). A production deployment wanting stronger
  unlinkability across sites should supply a per-site pepper — a
  future release adds the knob.
- **ClinicalAssistant integration with the ADR-0024 sovereign
  zone is advisory.** The ``zone_id`` and ``attestation_receipt_id``
  are recorded in the audit event but not enforced at the
  :class:`ClinicalAssistant` level. A future batch binds the
  assistant to a live attestation receipt so a response cannot be
  generated without a valid receipt.

### Neutral

- **This module is inside the existing ``medical`` package, not a
  new ``healthcare`` package.** Avoided creating a parallel name —
  the pre-existing ``medical`` is the right home for this
  capability.

## Alternatives considered

### Option A: Build a parallel ``zhilicon.healthcare`` package

Rejected. The pre-existing ``zhilicon.medical`` package already
covers DICOM loading + preprocessing + anonymisation. Creating a
second package splits the namespace and forces every integrator to
import from two places.

### Option B: Use ``presidio`` for PHI scrubbing

Presidio is Microsoft's PHI/PII detection library. Excellent
coverage on text, but heavy (spaCy + transformer models for entity
recognition; 500 MB install). For an SDK module that must be
importable in air-gapped sovereign clusters without network, the
dep weight is disqualifying. Our regex covers the 80% case; a
future batch adds presidio as an optional accelerator for
high-assurance deployments.

### Option C: Require production deployments to use a third-party
de-identification service

Rejected as the *only* option. Third-party DIS is the right
solution for certification-grade de-identification — but requiring
it before any Zhilicon demo means we cannot show a hospital CTO
PHI-safe behaviour today. Ship the best-effort scrubber now,
document the cert-grade DIS integration path for customers who
need it.

### Option D: Embed PHI scrubbing inside ``ClinicalAssistant`` and
hide the scrubbers from the public API

Rejected. A hospital data-engineer who has a FHIR bundle, not a
DICOM study, still wants the PHI scrubber. Exposing the three
scrubbers individually lets the component be reused without the
full ClinicalAssistant wrapper.

## References

- [ADR-0015: Minimal LLaMA reference model](ADR-0015-minimal-llama-reference-model.md)
- [ADR-0023: HuggingFace SafeTensors weight loader](ADR-0023-huggingface-weight-loader.md)
- [ADR-0024: Sovereign-zone enforcement + attestation receipts](ADR-0024-sovereign-zone-enforcement.md)
- [`src/sdk/python/zhilicon/medical/clinical.py`](../../src/sdk/python/zhilicon/medical/clinical.py)
- [`src/tests/sdk/python/test_medical_clinical.py`](../../src/tests/sdk/python/test_medical_clinical.py)
- [`docs/strategy/EXECUTION_TO_A_PLUS_ROADMAP.md`](../strategy/EXECUTION_TO_A_PLUS_ROADMAP.md) — Discovery-1 gates updated to reflect healthcare-SDK mitigation
- HIPAA Safe Harbor de-identification: 45 CFR §164.514(b)(2)
- HL7 FHIR AuditEvent resource: <https://www.hl7.org/fhir/auditevent.html>
- Portfolio audit 2026-04-18 (this session): Discovery-1 §HIPAA and §DICOM-ASIC gap line items
- Batch-40 session history.
