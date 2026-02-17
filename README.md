# PQPR -- Proof-of-Reference Tool

* **Specification Version:** 1.0.0
* **Status:** Implementation Ready
* **Date:** 2026
* **Author:** rosiea
* **Contact:** [PQRosie@proton.me](mailto:PQRosie@proton.me)
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea
* **PQ Ecosystem:** STANDALONE — The PQ Ecosystem is a post-quantum security framework built on deterministic enforcement, fail-closed semantics, and refusal-driven authority. Bitcoin is the reference deployment. It is not the scope.

---

**Problem:** Specifications reference external artefacts (inscriptions, profiles, published documents) but provide no cryptographic proof that the reference was valid at the time of authoring.

**Solution:** PQPR produces timestamped, signed proof that a referenced artefact existed and matched a specific hash at a specific Epoch Clock tick. Deterministic. Verifiable. No trust required.

PQPR defines reference evidence only. It does not enforce policy or grant authority.

Part of the [PQ Ecosystem](https://github.com/rosieRRRRR/pq-ecosystem).

---

## Summary

It compares documents and responses to identify supported assertions, unsupported claims, and coverage gaps without requiring model cooperation or network access.

PQPR is informational only and does not provide authorisation or enforcement.

---

## What This Is

A standalone auditing tool that answers one question: **did the AI output demonstrably overlap with my supplied files, or does it contain assertions unsupported by those files?**

You upload documents to an AI. You get a response. PQPR compares the two and tells you:

- Which assertion markers in the output have text overlap with your documents (anchored)
- Which assertion markers appear nowhere in your documents (unanchored)
- How much of each document was touched by anchored markers (coverage)
- Which unanchored markers are high-specificity - numbers, versions, identifiers - the strongest hallucination indicators

No API access required. No model cooperation. No external dependencies. You hold the files, you run the check.

PQPR defines deterministic, text-overlap verification between AI output and supplied documents. PQPR does not provide ingestion attestation, runtime instrumentation proof, or host-side verification. Those capabilities are outside the scope of version 1.0.0 and MAY be defined in future versions.

**Anchored ≠ sourced.** An anchored marker means "this string appears in the supplied text." It does not prove the model derived it from that text. Common phrases, technical terms, and normative language may anchor coincidentally.

Consuming systems MUST NOT describe an anchored marker as "sourced", "cited", or "derived" unless they possess independent evidence of derivation outside PQPR.

---

## Index

- [0. Scope and Authority Boundary](#0-scope-and-authority-boundary)
- [1. Quick Start](#1-quick-start)
- [2. Example Output](#2-example-output)
- [3. How It Works](#3-how-it-works)
- [4. Hashing](#4-hashing)
- [5. Source Document Registration](#5-source-document-registration)
  - [5.1 Text Extraction Boundary](#51-text-extraction-boundary)
- [6. Assertion Marker Extraction](#6-assertion-marker-extraction)
  - [6.1 Extractable Marker Classes](#61-extractable-marker-classes)
  - [6.2 Extraction Rules](#62-extraction-rules)
  - [6.3 Normalisation](#63-normalisation)
- [7. Anchor Search](#7-anchor-search)
  - [7.1 Exact Match](#71-exact-match)
  - [7.2 Normalised Match](#72-normalised-match)
  - [7.3 Anchor Record](#73-anchor-record)
- [8. Coverage Computation](#8-coverage-computation)
  - [8.1 Per-Document Coverage](#81-per-document-coverage)
  - [8.2 Marker Coverage](#82-marker-coverage)
  - [8.3 Engagement Scores](#83-engagement-scores)
- [9. Unanchored Marker Detection](#9-unanchored-marker-detection)
- [10. Verification Result](#10-verification-result)
- [11. Verification Algorithm](#11-verification-algorithm)
- [12. What PQPR Does NOT Prove](#12-what-pqpr-does-not-prove)
- [13. Future Extensions](#13-future-extensions)
- [14. Extensions](#14-extensions)
  - [14.1 Inline Annotation Output](#141-inline-annotation-output)
  - [14.2 Batch Mode](#142-batch-mode)
  - [14.3 Explicit Non-Inclusions](#143-explicit-non-inclusions)
- [15. Accountability Metadata](#15-accountability-metadata)
  - [15.1 Operator Context](#151-operator-context)
  - [15.2 Time Assertion](#152-time-assertion)
  - [15.3 Run Fingerprint](#153-run-fingerprint)
  - [15.4 Use Limitation Statement](#154-use-limitation-statement)
  - [15.5 Integration into Verification Result](#155-integration-into-verification-result)
  - [15.6 Posture Summary](#156-posture-summary)
  - [15.7 Explicit Non-Claims](#157-explicit-non-claims)
- [Annex A: Reference Implementation](#annex-a-reference-implementation)

---

## 0. Scope and Authority Boundary

PQPR:

1. Is a **standalone auditing tool**, not an enforcement mechanism.
2. Produces **text overlap evidence** about the relationship between model output and supplied input.
3. Grants **no authority**. It does not approve, deny, or gate operations.
4. Does **not** require model cooperation, host instrumentation, or API access.
5. Does **not** prove the model ingested the file, derived content from it, or comprehended it. It proves whether the output contains strings that also appear in the supplied files.
6. Uses **deterministic pattern matching** that is heuristic by design but reproducible: given the same inputs, every run produces identical core result fields. `timestamp_utc` is diagnostic metadata and varies across runs.

PQPR does not provide ingestion attestation, runtime instrumentation proof, or host-side verification. Those capabilities are outside the scope of version 1.0.0 and MAY be defined in future versions.

Consuming systems MAY use PQPR results for audit, quality screening, or holder inspection. They MUST NOT treat results as proof of ingestion, derivation, comprehension, or correctness.

---

## 1. Quick Start

**Requirements:** Python 3.9+. Standard library only. Zero dependencies.

Save [Annex A](#annex-a-reference-implementation) as `pqpr.py`, then:

```bash
# Basic audit
python pqpr.py --sources myfile.md --output ai_response.md

# Multiple source files
python pqpr.py --sources spec1.md spec2.md data.txt --output response.md

# Strict mode: high-specificity markers only (numeric, version, identifier, section_ref)
python pqpr.py --sources myfile.md --output response.md --strict

# Inline annotation: produces response.pqpr_annotated.md
python pqpr.py --sources myfile.md --output response.md --annotate

# Batch mode: audit all outputs against shared sources
python pqpr.py --batch-dir ./reviews/ --sources spec1.md spec2.md

# Batch mode with fail threshold (CI integration)
python pqpr.py --batch-dir ./reviews/ --sources spec1.md --fail-on-highspec 0

# JSON output for programmatic consumption
python pqpr.py --sources myfile.md --output response.md --json

# Verbose: show all unanchored markers
python pqpr.py --sources myfile.md --output response.md --verbose
```

---

## 2. Example Output

```
============================================================
  PQPR Audit Result
============================================================

⚠  High-specificity unanchored assertions detected.

SOURCE DOCUMENTS
----------------------------------------
  neural_final.md
    coverage: 0.09% (65/66966 chars)

ASSERTION MARKERS
----------------------------------------
  total extracted:              33
  anchored (text overlap):      24
  unanchored:                   9
  high-specificity unanchored:  5
  marker coverage:              72.72%

ENGAGEMENT
----------------------------------------
  engagement (min):             0.09%
  engagement (mean):            0.09%
  engagement (weighted):        0.09%

HIGH-SPECIFICITY UNANCHORED MARKERS
----------------------------------------
  [identifier] RSA-4096
  [numeric] 2030
  [numeric] 4096
============================================================
```

In this example, the AI response contained assertions about "RSA-4096 signatures" and a "2030 quantum threshold" - neither string exists anywhere in the source document. The auditor flagged them.

---

## 3. How It Works

1. **Register** each source file: hash it, extract text.
2. **Extract assertion markers** from the model output by deterministic pattern matching (6 marker classes). This is heuristic by design - regex patterns catch structured tokens, not natural language claims - but reproducible across runs.
3. **Search** each marker against all source documents (exact match, then normalised fallback).
4. **Compute coverage**: what fraction of each document was touched by anchored markers.
5. **Flag unanchored markers**: output markers with no source anchor. High-specificity unanchored markers (numbers, versions, identifiers, section references) are the strongest hallucination indicators.

**Important limitations:** Anchoring proves text overlap, not derivation. A marker like "MUST deny" may appear in many documents; its presence in both your file and the output does not prove the model sourced it from your file. Conversely, a correct paraphrase that does not match any extraction pattern will not be detected. This tool catches structured assertion overlap, not semantic fidelity.

---

## 4. Hashing

All hashes use SHAKE256-256 with 32-byte output, consistent with PQSF Tier 2 canonical hashing (PQSF §9).

```
H(x) = SHAKE256-256(x)   // 32 bytes
```

PQPR aligns with PQSF hashing to ensure that document fingerprints are directly comparable with artefacts in the broader PQ ecosystem. If an implementation requires SHA-256 instead, it MUST declare the deviation and results MUST NOT be mixed with PQSF-aligned artefacts.

PQPR is not a PQSF conformance target. PQPR uses SHAKE256-256 for interoperability but remains conformant without PQSF. Implementations substituting SHA-256 MUST declare deviation and MUST NOT mix hash types within a run.

No floating-point values are used anywhere in this specification. All arithmetic is integer.

---

## 5. Source Document Registration

Before verification, each source document is registered:

```
SourceRegistration = {
  doc_id:           bytes(16),    // H(raw_bytes)[:16] - deterministic
  filename:         str,          // original filename (informational)
  raw_bytes_hash:   bytes(32),    // H(raw file bytes)
  raw_bytes_len:    int,          // length of raw file bytes
  text:             str,          // extracted text representation
  text_bytes_hash:  bytes(32),    // H(text.encode('utf-8'))
  text_bytes_len:   int           // length of text bytes
}
```

Rules:

1. `doc_id` is derived deterministically: `H(raw_bytes)[:16]`. Given the same file, the same `doc_id` is produced on every run.
2. `raw_bytes_hash` is computed over the exact file bytes as stored on disk.
3. `text` is the UTF-8 text representation used for marker matching. For plain text and markdown files, this is the file contents directly. For other formats, text extraction is the caller's responsibility (see §5.1).
4. `text_bytes_hash` is computed over the UTF-8 encoding of `text`.
5. The auditor operates over `text`, not raw bytes, because the model sees text, not binary.

### 5.1 Text Extraction Boundary

Text extraction for non-UTF-8 formats (PDF, DOCX, images with OCR) is **external to PQPR**. The auditor verifies against whatever text is supplied. If extraction is lossy, incomplete, or wrong, results are correspondingly unreliable.

PQPR v1.0.0 does not define a recommended extraction profile. Future versions MAY specify deterministic extraction toolchains for common formats.

For the reference implementation (Annex A), the tool reads files as UTF-8. Binary files that fail UTF-8 decoding are decoded with replacement characters and a warning is emitted.

---

## 6. Assertion Marker Extraction

Assertion markers are extracted from the model output by deterministic pattern matching. Patterns are heuristic by design - they catch structured tokens that are mechanically searchable - but they are reproducible: the same input always produces the same markers.

### 6.1 Extractable Marker Classes

| Class | Pattern | Example | Specificity |
|---|---|---|---|
| `section_ref` | Section and annex references | `§4.12`, `Annex AS`, `Section 7.3` | High |
| `version` | Dot-separated numeric sequences, optionally prefixed | `v1.0`, `2.5.2`, `Version 1.3.0` | High |
| `identifier` | Technical identifiers and error codes | `PQSEC`, `operator_state_ok`, `E_INTENT_MISMATCH` | High |
| `numeric` | Numbers: integers, decimals, significant values | `256`, `10000`, `3.14` | High |
| `quoted` | Text enclosed in quotation marks or backticks | `"fail-closed"`, `` `intent_id` `` | Low |
| `must_shall` | Normative requirement statements (RFC 2119 keywords) | `MUST deny`, `MUST NOT permit` | Low |

High-specificity classes (`section_ref`, `version`, `identifier`, `numeric`) are the primary hallucination indicators. An identifier or number that appears nowhere in the supplied files is a strong signal of unsupported content.

Low-specificity classes (`quoted`, `must_shall`) produce weaker signals. Common normative phrases like "MUST NOT" appear across many documents. These markers are included by default but can be suppressed with `--strict` mode.

### 6.2 Extraction Rules

1. Extraction is performed by regular expression matching over the model output text.
2. Each match produces a marker with its class, matched text, and character offset in the output.
3. Overlapping matches are permitted; each match is a separate marker.
4. Extraction patterns are defined in the reference implementation (Annex A) and are part of the conformance surface.
5. In `--strict` mode, only high-specificity classes are extracted.

### 6.3 Normalisation

Before matching against source documents, both marker text and source text are normalised:

1. Convert to lowercase.
2. Collapse all whitespace (spaces, tabs, newlines) to single space.
3. Strip leading and trailing whitespace.

For the relaxed fallback match (§7.2), additionally:

4. Remove all punctuation except hyphens and underscores.
5. Collapse repeated hyphens/underscores to single.

No stemming. No synonym expansion. No semantic similarity.

---

## 7. Anchor Search

For each extracted marker, the auditor searches all registered source documents for matching text.

### 7.1 Exact Match

Search for the marker's normalised text as a substring of the source document's normalised text. If found, record an anchor at the character offset in the normalised source.

### 7.2 Normalised Match

If no exact match is found, attempt matching with relaxed normalisation (§6.3 steps 4–5) applied to both marker and source. If found, record the anchor with `match_type = "relaxed"`.

No further relaxation is attempted. If neither match succeeds, the marker is **unanchored**.

### 7.3 Anchor Record

```
Anchor = {
  marker_index:       int,         // index into extracted markers list
  doc_id:             bytes(16),   // source document
  match_type:         str,         // "exact" or "relaxed"
  match_space:        str,         // "normalised" or "relaxed" - coordinate space of offsets
  match_start:        int,         // character offset start in match_space text
  match_end:          int,         // character offset end in match_space text
  source_hash:        bytes(32),   // H(matched substring bytes)
  marker_hash:        bytes(32)    // H(normalised/relaxed marker bytes)
}
```

**Coordinate spaces:** Exact matches produce offsets in the normalised source text (`match_space = "normalised"`). Relaxed matches produce offsets in the relaxed source text (`match_space = "relaxed"`). These coordinate spaces are not interchangeable. Coverage computation (§8.1) uses only normalised-space anchors.

---

## 8. Coverage Computation

### 8.1 Per-Document Coverage

For each source document:

1. Collect all `(match_start, match_end)` pairs for anchors referencing this document **where `match_space = "normalised"` (normalised-space anchors only)**. Relaxed-match anchors MUST NOT contribute to coverage because their offsets are in a different coordinate space.
2. Sort by `match_start`.
3. Merge overlapping or adjacent ranges.
4. Sum merged range lengths to get `covered_chars`.
5. Compute `coverage_bp = (covered_chars * 10000) // total_chars`.

All arithmetic is integer. No floating point.

### 8.2 Marker Coverage

```
anchored_markers    = count of markers with at least one anchor
total_markers       = count of all extracted markers
marker_coverage_bp  = (anchored_markers * 10000) // total_markers
```

### 8.3 Engagement Scores

Three engagement metrics are computed to give different views of how broadly the output references the supplied documents:

**engagement_min_bp:** The minimum per-document coverage across all source documents. Conservative: penalises any untouched document. Useful for "did it reference everything I gave it?"

```
engagement_min_bp = min(doc.coverage_bp for doc in source_documents)
```

**engagement_mean_bp:** The arithmetic mean of per-document coverage. Balanced: tolerates one low-coverage document if others are well-covered.

```
engagement_mean_bp = sum(doc.coverage_bp for doc in source_documents) // len(source_documents)
```

**engagement_weighted_bp:** Coverage weighted by document length. Prevents a short document from dominating the score.

```
total_chars = sum(doc.text_chars for doc in source_documents)
weighted_sum = sum(doc.covered_chars for doc in source_documents)
engagement_weighted_bp = (weighted_sum * 10000) // total_chars
```

All three are integer arithmetic. Policy or user preference determines which is most relevant for a given use case.

---

## 9. Unanchored Marker Detection

Any extracted marker with no anchor in any source document is **unanchored**.

If matching cannot be completed due to tool-local failure (e.g. corrupted source document, decode failure, normalisation failure, or I/O error), the marker MUST be classified as **UNAVAILABLE**, not unanchored.

UNAVAILABLE indicates "no conclusion possible" rather than "no overlap exists".

Unanchored markers are classified by specificity:

- **High-specificity** (classes: `numeric`, `version`, `identifier`, `section_ref`): A number, version string, or technical identifier that appears nowhere in the supplied files is a strong indicator of unsupported content - potential hallucination or content sourced from training data rather than the provided material.
- **Low-specificity** (classes: `must_shall`, `quoted`): Common normative phrases or short quoted strings that may appear generically. Still flagged, but less diagnostic.

If any high-specificity unanchored markers are detected, the audit result includes a top-line warning:

**"⚠ High-specificity unanchored assertions detected."**

---

## 10. Verification Result

```
PQPRResult = {
  result_version:               2,              // output schema version (not spec version)
  // NOTE: result_version is the PQPRResult schema version only.
  // It is independent of the PQPR specification version shown in the document header.
  timestamp_utc:                str,              // ISO 8601

  // Source documents
  documents: [{
    doc_id:                     bytes(16),        // deterministic: H(raw_bytes)[:16]
    filename:                   str,
    raw_bytes_hash:             bytes(32),
    text_bytes_hash:            bytes(32),
    text_chars:                 int,              // length of normalised text used for coverage
    covered_chars:              int,
    coverage_bp:                int               // 0–10000
  }],

  // Markers
  total_markers:                int,
  anchored_markers:             int,
  unanchored_markers:           int,
  unavailable_markers:          int,
  high_specificity_unanchored:  int,
  marker_coverage_bp:           int,              // 0–10000

  // Engagement
  engagement_min_bp:            int,              // 0–10000
  engagement_mean_bp:           int,              // 0–10000
  engagement_weighted_bp:       int,              // 0–10000

  // Detail
  anchors:                      [Anchor],         // see §7.3 for field names
  unanchored:                   [{
    marker_index:               int,
    marker_class:               str,
    marker_text:                str,
    high_specificity:           bool
  }],

  // Unavailable (matching could not complete)
  unavailable:                  [{
    marker_id:                  str,
    reason:                     str,              // e.g. "SOURCE_DECODE_FAILED", "IO_ERROR", "NORMALISATION_FAILED"
    source_ref:                 str / null
  }],

  // Provenance
  source_hashes:                [bytes(32)],      // ordered doc hashes
  output_hash:                  bytes(32),        // H(model output bytes)
  strict_mode:                  bool              // whether --strict was used
}
```

**Result Interpretation Disclaimer (Normative):** PQPRResult reports anchoring and coverage metrics only. High coverage does not guarantee correctness, and low coverage does not indicate fabrication. Results MUST be interpreted in conjunction with domain expertise and source-document quality assessment.

---

## 11. Verification Algorithm

The auditor executes the following steps in order:

**Step 1: Register source documents.**
For each input file, compute deterministic `doc_id`, hashes, and extract text.

**Step 2: Hash model output.**
Compute `H(output.encode('utf-8'))`.

**Step 3: Extract assertion markers from model output.**
Apply extraction patterns from §6.1. In strict mode, extract high-specificity classes only. Record all matches with offsets.

**Step 4: Search for anchors.**
For each marker, search all source documents per §7. Record anchors with normalised offsets.

**Step 5: Compute coverage.**
Per-document coverage per §8.1. Marker coverage per §8.2. All three engagement scores per §8.3.

**Step 6: Detect unanchored markers.**
Flag markers with no anchors per §9. Classify by specificity. Emit top-line warning if high-specificity unanchored markers exist.

**Step 7: Assemble result.**
Produce PQPRResult with all fields populated.

All steps are deterministic pattern matching. Given identical inputs, every run produces identical core fields. `timestamp_utc` is diagnostic metadata and may vary.

---

## 12. What PQPR Does NOT Prove

1. **Ingestion.** PQPR cannot prove the model received the file. The inference provider may have truncated or discarded it.

2. **Derivation.** An anchored marker means the string exists in both the output and the source. It does not prove the model sourced it from the file. The model may have produced the same string from training data.

3. **Comprehension.** A model can produce extensive text overlap with a document and still misunderstand it.

4. **Correctness.** Anchored markers may be faithfully quoted but incorrectly applied.

5. **Exhaustiveness.** Extraction catches structured tokens only. Natural language claims, paraphrases, and inferences are not extracted and therefore not evaluated.

6. **Absence of hallucination.** A clean audit (zero unanchored markers) does not prove the output is faithful. It proves the structured tokens overlap with the source. The model may hallucinate in ways that do not produce extractable markers.

---

## 13. Future Extensions

This specification defines deterministic text-overlap verification only.

Future versions MAY extend PQPR with:

* host-side ingestion attestation,
* representation receipts,
* challenge-response document access verification,
* PQSEC predicate integration,
* multi-modal artefact support.

These extensions are not required for conformance to PQPR v1.0.0.

---

## 14. Extensions (Implementation Ready)

This section defines optional extensions that improve usability and workflow integration of PQPR without changing its scope or authority boundary.

All extensions remain **standalone auditing features**. They produce evidence and diagnostics only. They grant no authority.

---

### 14.1 Inline Annotation Output (Normative)

#### 14.1.1 Purpose

Inline annotation produces a marked-up version of the AI output where extracted **assertion markers** are highlighted in place as:

* **ANCHOR** - marker text overlaps supplied source text
* **UNANCHORED** - marker text does not overlap supplied source text
* **HIGH_UNANCHORED** - unanchored marker in a high-specificity class

This turns the audit from a separate report into an actionable annotated document.

Inline annotation is **diagnostic only**. It MUST NOT be interpreted as proof of derivation or correctness.

#### 14.1.2 Output Format

For v1.0, only **plain-text Markdown annotation** is defined.

Markers MUST be wrapped with literal tags inserted into the output text:

* Anchored marker: `[PQPR:ANCHOR]…[/PQPR]`
* Unanchored marker: `[PQPR:UNANCHORED]…[/PQPR]`
* High-specificity unanchored marker: `[PQPR:HIGH_UNANCHORED]…[/PQPR]`

These tags are plain text, survive copy-paste, are diff-friendly, and require no rendering engine.

HTML annotation is explicitly **out of scope** for v1.0.

#### 14.1.3 Annotation Ordering and Overlap Rules (Normative)

To ensure correctness and determinism, the annotation engine MUST apply the following rules:

1. Annotation operates over the **original model output string**, using the marker offsets extracted from that string.
2. Marker spans MUST be processed in **descending order of `offset`** (highest offset first).
3. If two markers have the same `offset`, the **longer span MUST be processed first**.
4. If a marker span **overlaps** any span that has already been annotated, the overlapping marker MUST be **skipped** and a counter `skipped_overlaps` MUST be incremented. Overlap is defined as: `marker_A.offset < marker_B.offset + marker_B.length AND marker_B.offset < marker_A.offset + marker_A.length`. Adjacent markers (where one ends exactly where the next begins) are not overlapping.
5. Annotation MUST NOT modify or reorder any non-marker text.
6. Annotation MUST be deterministic: given identical inputs, output bytes MUST be identical.

Rationale: processing from the end prevents index drift when inserting tags.

#### 14.1.4 Output Files

When annotation is enabled, the auditor MUST emit:

* `<output_basename>.pqpr_annotated.md`

The original output file MUST NOT be modified.

#### 14.1.5 CLI Integration

The following CLI flags MUST be supported:

* `--annotate` enables inline annotation.
* `--annotate-out <path>` (optional) overrides the default annotated output path.

Annotation MAY be combined with `--strict`, `--verbose`, `--json`, and batch mode.

#### 14.1.6 Result Metadata

When annotation is enabled, the verification result MUST include:

```
annotation = {
  "enabled": true,
  "skipped_overlaps": <int>
}
```

This allows downstream tooling to detect cases where overlapping markers were suppressed.

---

### 14.2 Batch Mode (Normative)

#### 14.2.1 Purpose

Batch mode runs PQPR across many AI outputs in one invocation and produces a consolidated summary.

This is intended for iterative document review, CI-like workflows, and regression checking.

Batch mode is a **workflow convenience**, not an enforcement mechanism.

#### 14.2.2 Execution Modes

Batch mode supports two source-resolution strategies. The explicit source override is the primary mode; basename matching is a convenience fallback.

**A) Explicit Sources (Primary)**

* `--batch-dir <dir>` directory containing AI output files.
* `--sources <file1> [file2 ...]` source files applied to **every** output file in the batch.

This is the expected mode for most workflows. The user knows which source documents the AI was given and applies them uniformly.

**B) Basename Matching (Fallback)**

* `--batch-dir <dir>` directory containing AI output files.
* `--sources-dir <dir>` directory containing source documents.

For each output file `<batch-dir>/X.ext`, the auditor selects source files from `--sources-dir` whose filenames match `X.*` (same name, any extension).

Examples:

* `review1.md` matches `review1.md`, `review1.txt`
* `specA_response.md` matches `specA_response.*`

If no matching sources are found for an output file, that job MUST be skipped and recorded as skipped in the summary.

If both `--sources` and `--sources-dir` are provided, `--sources` takes precedence.

#### 14.2.3 Batch Execution Semantics

For each output file:

1. Resolve its source set using the active pairing mode.
2. Run PQPR verification exactly as in single-file mode.
3. Record metrics and warnings.
4. Continue processing remaining files even if some fail thresholds.

Batch execution MUST be deterministic with respect to directory contents, pairing rules, and strict mode.

#### 14.2.4 Batch Summary Outputs

Batch mode MUST produce two summary files in the `--batch-dir` directory (collocated with the outputs they describe):

**A) `pqpr_batch_summary.json`**

```
BatchSummary = {
  "v": 1,
  "generated_utc": "<iso8601>",
  "results": [
    {
      "output_file": "<string>",
      "output_hash": "<hex32>",
      "sources": [
        {
          "filename": "<string>",
          "raw_bytes_hash": "<hex32>",
          "coverage_bp": <int>
        }
      ],
      "high_specificity_unanchored": <int>,
      "marker_coverage_bp": <int>,
      "engagement_min_bp": <int>,
      "engagement_mean_bp": <int>,
      "engagement_weighted_bp": <int>,
      "strict_mode": <bool>
    }
  ],
  "skipped": [
    {
      "output_file": "<string>",
      "reason": "no_matching_sources"
    }
  ]
}
```

**B) `pqpr_batch_summary.txt`**

A human-readable table with one row per output file, showing: output filename, high-specificity unanchored count, engagement min / mean / weighted, and marker coverage. Skipped files MUST be listed separately. Formatting is implementation-defined but MUST be deterministic.

#### 14.2.5 Batch CLI Flags

The following flags MUST be supported:

* `--batch-dir <dir>`
* `--sources-dir <dir>` (basename matching mode)
* `--sources <list>` (explicit sources mode; takes precedence)
* `--strict` applies strict mode to all batch jobs.

Optional workflow controls:

* `--fail-on-highspec <N>` exit with non-zero code if **any** batch job has `high_specificity_unanchored > N`.
* `--fail-on-engagement-min-bp <X>` exit with non-zero code if **any** batch job has `engagement_min_bp < X`.

These flags control process exit codes only. They MUST NOT be described as enforcement or authorisation.

#### 14.2.6 Exit Codes

* `0` batch completed, no fail thresholds triggered
* `1` batch completed, one or more fail thresholds triggered
* `2` invalid arguments or filesystem errors

---

### 14.3 Explicit Non-Inclusions (Normative)

For v1.0, the following are **explicitly out of scope**:

* Exportable signed evidence bundles
* Canonical JSON or CBOR bundle formats
* HTML annotation output
* CI manifest schemas beyond directory scan mode

These MAY be introduced in later versions once there is a consuming verification pipeline.

---

### Implementation Notes for §14 (Non-Normative)

* **Annotation correctness** depends on offset discipline. Always annotate from highest offset to lowest.
* **Coverage metrics** MUST exclude anchors derived from relaxed matching (§8.1).
* **Batch mode** should be implemented as a thin loop over the existing single-file verifier to avoid divergence.

---

## 15. Accountability Metadata (Implementation Ready)

This section defines **optional, non-authoritative metadata** that strengthens PQPR's suitability as supporting evidence in regulatory, dispute-resolution, or audit contexts.

These additions do **not** change verification semantics, add enforcement authority, or assert correctness or truth. They exist to satisfy common expectations around attribution, timing, and evidentiary integrity.

---

### 15.1 Operator Context (Non-Authoritative)

#### 15.1.1 Purpose

Accountability contexts routinely ask: who ran the tool, and in what capacity?

PQPR is intentionally user-side. This section allows an operator to attach **declarative context** without turning PQPR into an identity or authority system.

#### 15.1.2 OperatorContext Object

```
OperatorContext = {
  operator_role: str,
  operator_statement: str / null,
  invocation_context: str / null
}
```

**Field semantics:**

* `operator_role` declares the role of the party running the audit. Free-text string. Recommended values: `"user"`, `"reviewer"`, `"agent"`, `"automated_pipeline"`. Implementations MUST accept any string value. The recommended values are conventions, not a closed enum.
* `operator_statement` is a free-text description of why the audit was performed (e.g. "reviewing AI summary for technical accuracy").
* `invocation_context` is an optional description of the broader workflow (e.g. "pre-publication review", "internal QA", "regulatory submission").

#### 15.1.3 Authority Boundary

OperatorContext is **not verified**, is **not trusted**, and carries **no authority**. It exists to provide narrative context when PQPR output is presented as supporting evidence.

#### 15.1.4 CLI Integration

* `--operator-role <string>` sets operator_role.
* `--operator-statement <string>` sets operator_statement.
* `--invocation-context <string>` sets invocation_context.

All three are optional. If none are provided, `operator_context` is null in the result.

---

### 15.2 Time Assertion (Non-Authoritative)

#### 15.2.1 Purpose

Accountability contexts care about **when** an artefact was produced and whether time claims are clearly qualified. PQPR timestamps are informational. This section makes that explicit.

#### 15.2.2 TimeAssertion Object

```
TimeAssertion = {
  generated_utc: "<ISO-8601 string>",
  time_source: "system_clock" | "external_timestamp" | "manual"
}
```

**Rules:**

1. `generated_utc` MUST be present.
2. `time_source` MUST be declared. Default is `"system_clock"`.
3. Time assertions are **informational only** and MUST NOT be represented as tamper-proof or independently verified unless explicitly stated.

#### 15.2.3 Disclosure Requirement

Any consumer presenting PQPR output MUST NOT imply that time assertions are cryptographically anchored or externally attested unless that is actually the case.

---

### 15.3 Run Fingerprint (Deterministic)

#### 15.3.1 Purpose

Accountability contexts often ask: can we be sure this result corresponds to these inputs?

PQPR already hashes inputs and output. The **Run Fingerprint** binds them into a single deterministic identifier.

#### 15.3.2 RunFingerprint Definition

The fingerprint is computed over fixed-length and length-prefixed fields to prevent ambiguous concatenation:

```
fingerprint_input =
  count(source_hashes) as 4-byte big-endian  ||
  source_hashes (each 32 bytes, ordered lexicographically)  ||
  output_hash (32 bytes)  ||
  strict_mode (1 byte: 0x00 = false, 0x01 = true)  ||
  len(tool_version) as 1-byte  ||
  tool_version (UTF-8 bytes)

RunFingerprint = H(fingerprint_input)
```

Where:

* `H` is SHAKE256-256 (PQSF Tier 2).
* `source_hashes` are the raw 32-byte hashes sorted lexicographically before concatenation.
* `tool_version` is a normative string identifying the PQPR implementation version. For the reference implementation, `tool_version` MUST be `"pqpr-1.0.0"`. Alternative implementations MUST use a distinct string that changes whenever the implementation changes behaviour.

#### 15.3.3 Semantics

* Identical inputs and settings MUST produce the same RunFingerprint.
* Any change to inputs, output, mode, or tool version MUST change the fingerprint.

This provides lightweight chain-of-custody, easy reference in correspondence or testimony, and zero dependency on signatures or key management.

---

### 15.4 Use Limitation Statement (Normative Disclosure)

#### 15.4.1 Required Statement

The following statement (verbatim or with materially identical meaning) MUST be included in PQPR output:

> **Use Limitation:**
> PQPR is suitable for detecting unsupported structured assertions via text overlap analysis.
> It is not suitable for determining factual correctness, legal compliance, safety, or intent satisfaction.
> Results are evidence of overlap or absence only.

#### 15.4.2 Display Rules

This statement MUST be present:

* in JSON output under a `use_limitation` field,
* in `--help` output.

In human-readable CLI output, the statement SHOULD be displayed on first use per session. Implementations MAY suppress it on subsequent runs to avoid visual fatigue, provided the `--help` and JSON paths always include it.

#### 15.4.3 Rationale

This disclosure prevents implied guarantees, reduces misrepresentation risk, and aligns user expectations with actual capability.

---

### 15.5 Integration into Verification Result

The `PQPRResult` object MUST include the following additional fields when accountability metadata is enabled:

```
PQPRResult = {
  ...existing fields...

  operator_context:   OperatorContext / null,
  time_assertion:     TimeAssertion,
  run_fingerprint:    bytes(32),
  use_limitation:     str
}
```

Rules:

1. `run_fingerprint` MUST be present in every result.
2. `time_assertion` MUST be present in every result.
3. `operator_context` is OPTIONAL (null if no operator flags provided).
4. `use_limitation` MUST be present in every result.

---

### 15.6 Posture Summary (Non-Normative)

PQPR is designed to function as **supporting evidence**, not a decision engine; **process transparency**, not outcome validation; and a **reproducible audit artefact**, not expert judgment.

It supports analysis concerning misrepresentation, reliance, disclosure of limitations, and reasonable verification effort.

It does not replace human review, expert testimony, or substantive analysis.

---

### 15.7 Explicit Non-Claims (Normative)

PQPR MUST NOT be described as:

* proving an AI was "correct",
* proving an AI "used" a document in a causal sense,
* proving absence of hallucination,
* certifying compliance, safety, or accuracy.

Any such representation is **non-conformant use** of PQPR.

---

## Annex A: Reference Implementation

Save the following as `pqpr.py`. Python 3.9+. Zero external dependencies.

```python
#!/usr/bin/env python3
"""
pqpr.py - Proof-of-Reference Auditor

Reference implementation for PQPR v1.0.0.
Determines whether AI model output contains assertion markers
that overlap with user-supplied input files.

PQPR v1.0.0 defines deterministic text-overlap verification.
It does not provide ingestion attestation or host-side verification.

Zero external dependencies. Python 3.9+ standard library only.

Usage:
    python pqpr.py --sources file1.md file2.md --output response.md
    python pqpr.py --sources file1.md --output response.md --strict
    python pqpr.py --sources file1.md --output response.md --json
    python pqpr.py --sources file1.md --output response.md --verbose

Copyright 2026 rosiea - Apache License 2.0
"""

import argparse
import json
import os
import re
import sys
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from hashlib import shake_256
from typing import Optional


# ───────────────────────────────────────────────────────
# Tool Version (Normative for RunFingerprint §15.3)
# ───────────────────────────────────────────────────────

TOOL_VERSION = "pqpr-1.0.0"


# ───────────────────────────────────────────────────────
# §4  Hashing - SHAKE256-256, PQSF Tier 2 aligned
# ───────────────────────────────────────────────────────

def H(data: bytes) -> bytes:
    """SHAKE256-256: 32-byte output. PQSF Tier 2."""
    return shake_256(data).digest(32)


def compute_run_fingerprint(source_hashes: list[bytes],
                            output_hash: bytes,
                            strict_mode: bool) -> bytes:
    """
    Deterministic RunFingerprint per §15.3.
    """
    sorted_hashes = sorted(source_hashes)

    fingerprint_input = bytearray()

    # 4-byte count
    fingerprint_input += len(sorted_hashes).to_bytes(4, "big")

    # Source hashes (each 32 bytes, sorted lexicographically)
    for h in sorted_hashes:
        fingerprint_input += h

    # Output hash (32 bytes)
    fingerprint_input += output_hash

    # Strict mode (1 byte)
    fingerprint_input += b"\x01" if strict_mode else b"\x00"

    # Tool version (length-prefixed)
    tool_bytes = TOOL_VERSION.encode("utf-8")
    fingerprint_input += len(tool_bytes).to_bytes(1, "big")
    fingerprint_input += tool_bytes

    return H(bytes(fingerprint_input))


# ───────────────────────────────────────────────────────
# §5  Source Document Registration
# ───────────────────────────────────────────────────────

@dataclass
class SourceDocument:
    doc_id: bytes
    filename: str
    raw_bytes_hash: bytes
    raw_bytes_len: int
    text: str
    text_bytes_hash: bytes
    text_bytes_len: int
    normalised_text: str = ""
    relaxed_text: str = ""

    def __post_init__(self):
        self.normalised_text = normalise(self.text)
        self.relaxed_text = normalise_relaxed(self.text)


def register_source(filepath: str) -> SourceDocument:
    """Register a source document. Deterministic doc_id from file hash."""
    raw_bytes = open(filepath, "rb").read()
    raw_hash = H(raw_bytes)
    doc_id = raw_hash[:16]  # Deterministic: same file = same doc_id

    try:
        text = raw_bytes.decode("utf-8")
    except UnicodeDecodeError:
        text = raw_bytes.decode("utf-8", errors="replace")
        print(f"  Warning: {os.path.basename(filepath)} is not valid UTF-8. "
              "Using replacement characters.", file=sys.stderr)

    text_bytes = text.encode("utf-8")

    return SourceDocument(
        doc_id=doc_id,
        filename=os.path.basename(filepath),
        raw_bytes_hash=raw_hash,
        raw_bytes_len=len(raw_bytes),
        text=text,
        text_bytes_hash=H(text_bytes),
        text_bytes_len=len(text_bytes),
    )


# ───────────────────────────────────────────────────────
# §6  Assertion Marker Extraction
# ───────────────────────────────────────────────────────

@dataclass
class Marker:
    index: int
    marker_class: str
    text: str
    normalised: str
    offset: int
    high_specificity: bool = False


# Marker extraction patterns - §6.1
# Order: more specific patterns first.

HIGH_SPECIFICITY_CLASSES = {"numeric", "version", "identifier", "section_ref"}

MARKER_PATTERNS = [
    ("section_ref", re.compile(
        r"(?:§\s*\d+(?:\.\d+)*"
        r"|[Aa]nnex\s+[A-Z]{1,3}"
        r"|[Ss]ection\s+\d+(?:\.\d+)*"
        r"|[Cc]hapter\s+\d+(?:\.\d+)*)"
    )),
    ("version", re.compile(
        r"(?:[Vv]ersion\s+)?\d+\.\d+(?:\.\d+)+"
        r"|[Vv]\d+\.\d+(?:\.\d+)*"
    )),
    ("identifier", re.compile(
        r"(?:[A-Z]{2,}[A-Z0-9_]*(?:-[A-Z0-9]+)*"
        r"|[a-z][a-z0-9]*(?:_[a-z0-9]+){2,}"
        r"|E_[A-Z][A-Z0-9_]+)"
    )),
    ("quoted", re.compile(
        r"`([^`]{2,60})`"
        r"|\"([^\"]{2,60})\""
        r"|'([^']{2,60})'"
    )),
    ("must_shall", re.compile(
        r"(?:MUST(?:\s+NOT)?|SHALL(?:\s+NOT)?|SHOULD(?:\s+NOT)?|REQUIRED|RECOMMENDED)"
        r"\s+\w+(?:\s+\w+){0,5}"
    )),
    ("numeric", re.compile(
        r"-?\d{1,3}(?:,\d{3})+(?:\.\d+)?"
        r"|-?\d+\.\d+"
        r"|\b\d{2,}\b"
    )),
]


def extract_markers(output_text: str, strict: bool = False) -> list[Marker]:
    """Extract assertion markers from model output."""
    markers = []
    seen_spans = set()

    for marker_class, pattern in MARKER_PATTERNS:
        # Strict mode: skip low-specificity classes
        if strict and marker_class not in HIGH_SPECIFICITY_CLASSES:
            continue

        for match in pattern.finditer(output_text):
            span = (match.start(), match.end())
            if span in seen_spans:
                continue
            seen_spans.add(span)

            text = (match.group(1) or match.group(2) or match.group(3)
                    if marker_class == "quoted" else match.group(0))
            if text is None:
                text = match.group(0)

            normalised = normalise(text)
            if len(normalised.strip()) < 2:
                continue

            markers.append(Marker(
                index=len(markers),
                marker_class=marker_class,
                text=text,
                normalised=normalised,
                offset=match.start(),
                high_specificity=marker_class in HIGH_SPECIFICITY_CLASSES,
            ))

    return markers


# ───────────────────────────────────────────────────────
# §6.3  Normalisation
# ───────────────────────────────────────────────────────

def normalise(text: str) -> str:
    """Normalise text for matching."""
    text = text.lower()
    text = re.sub(r"\s+", " ", text)
    return text.strip()


def normalise_relaxed(text: str) -> str:
    """Relaxed normalisation for fallback matching."""
    text = normalise(text)
    text = re.sub(r"[^\w\s\-_]", "", text)
    text = re.sub(r"[-_]{2,}", "-", text)
    return text


# ───────────────────────────────────────────────────────
# §7  Anchor Search
# ───────────────────────────────────────────────────────

@dataclass
class Anchor:
    marker_index: int
    doc_id: bytes
    match_type: str           # "exact" or "relaxed"
    match_space: str          # "normalised" or "relaxed"
    match_start: int
    match_end: int
    source_hash: bytes
    marker_hash: bytes


def search_anchors(marker: Marker, documents: list[SourceDocument]) -> list[Anchor]:
    """Search all source documents for anchors supporting a marker."""
    anchors = []

    for doc in documents:
        # §7.1 Exact match on normalised text
        pos = doc.normalised_text.find(marker.normalised)
        if pos != -1:
            end = pos + len(marker.normalised)
            matched_bytes = doc.normalised_text[pos:end].encode("utf-8")
            anchors.append(Anchor(
                marker_index=marker.index,
                doc_id=doc.doc_id,
                match_type="exact",
                match_space="normalised",
                match_start=pos,
                match_end=end,
                source_hash=H(matched_bytes),
                marker_hash=H(marker.normalised.encode("utf-8")),
            ))
            continue

        # §7.2 Relaxed fallback match (normalised_relaxed)
        relaxed_marker = normalise_relaxed(marker.text)
        relaxed_source = doc.relaxed_text  # precomputed

        if len(relaxed_marker) >= 2:
            rpos = relaxed_source.find(relaxed_marker)
            if rpos != -1:
                rend = rpos + len(relaxed_marker)
                matched_bytes = relaxed_source[rpos:rend].encode("utf-8")
                anchors.append(Anchor(
                    marker_index=marker.index,
                    doc_id=doc.doc_id,
                    match_type="relaxed",
                    match_space="relaxed",
                    match_start=rpos,
                    match_end=rend,
                    source_hash=H(matched_bytes),
                    marker_hash=H(relaxed_marker.encode("utf-8")),
                ))

    return anchors


# ───────────────────────────────────────────────────────
# §8  Coverage Computation
# ───────────────────────────────────────────────────────

@dataclass
class DocCoverage:
    doc_id: bytes
    filename: str
    text_chars: int
    covered_chars: int
    coverage_bp: int


def compute_doc_coverage(doc: SourceDocument, anchors: list[Anchor]) -> DocCoverage:
    """Compute per-document coverage from exact (normalised-space) anchors only."""
    # §8.1: Only normalised-space anchors contribute to coverage.
    # Relaxed-match anchors are in a different coordinate space.
    ranges = sorted(
        [(a.match_start, a.match_end)
         for a in anchors
         if a.doc_id == doc.doc_id and a.match_space == "normalised"],
        key=lambda r: r[0],
    )

    merged = []
    for start, end in ranges:
        if merged and start <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], end))
        else:
            merged.append((start, end))

    covered = sum(end - start for start, end in merged)
    total = len(doc.normalised_text)

    return DocCoverage(
        doc_id=doc.doc_id,
        filename=doc.filename,
        text_chars=total,
        covered_chars=covered,
        coverage_bp=(covered * 10000) // total if total > 0 else 0,
    )


# ───────────────────────────────────────────────────────
# §10–11  Verification
# ───────────────────────────────────────────────────────

@dataclass
class PQPRResult:
    result_version: int = 2
    timestamp_utc: str = ""
    operator_context: Optional[dict] = None
    time_assertion: Optional[dict] = None
    run_fingerprint: str = ""
    use_limitation: str = ""
    documents: list[dict] = field(default_factory=list)
    total_markers: int = 0
    anchored_markers: int = 0
    unanchored_markers: int = 0
    high_specificity_unanchored: int = 0
    marker_coverage_bp: int = 0
    engagement_min_bp: int = 0
    engagement_mean_bp: int = 0
    engagement_weighted_bp: int = 0
    anchors: list[dict] = field(default_factory=list)
    unanchored: list[dict] = field(default_factory=list)
    source_hashes: list[str] = field(default_factory=list)
    output_hash: str = ""
    strict_mode: bool = False
    annotation: Optional[dict] = None


def verify(
    source_paths: list[str],
    output_path: str,
    strict: bool = False,
) -> PQPRResult:
    """Main verification function. Deterministic. Reproducible."""

    result = PQPRResult()
    result.timestamp_utc = datetime.now(timezone.utc).isoformat()
    result.strict_mode = strict

    # Step 1: Register source documents
    documents = []
    for path in source_paths:
        doc = register_source(path)
        documents.append(doc)
        result.source_hashes.append(doc.raw_bytes_hash.hex())

    # Step 2: Hash model output
    output_bytes = open(output_path, "rb").read()
    result.output_hash = H(output_bytes).hex()
    try:
        output_text = output_bytes.decode("utf-8")
    except UnicodeDecodeError:
        output_text = output_bytes.decode("utf-8", errors="replace")

    # Step 3: Extract assertion markers
    markers = extract_markers(output_text, strict=strict)
    result.total_markers = len(markers)

    # Step 4: Search for anchors
    all_anchors: list[Anchor] = []
    anchored_indices: set[int] = set()
    for marker in markers:
        found = search_anchors(marker, documents)
        all_anchors.extend(found)
        if found:
            anchored_indices.add(marker.index)

    result.anchored_markers = len(anchored_indices)
    result.unanchored_markers = result.total_markers - result.anchored_markers

    # Step 5: Compute coverage
    doc_coverages = []
    for doc in documents:
        cov = compute_doc_coverage(doc, all_anchors)
        doc_coverages.append(cov)
        result.documents.append({
            "doc_id": doc.doc_id.hex(),
            "filename": doc.filename,
            "raw_bytes_hash": doc.raw_bytes_hash.hex(),
            "text_bytes_hash": doc.text_bytes_hash.hex(),
            "text_chars": cov.text_chars,
            "covered_chars": cov.covered_chars,
            "coverage_bp": cov.coverage_bp,
        })

    # Marker coverage §8.2
    if result.total_markers > 0:
        result.marker_coverage_bp = (
            result.anchored_markers * 10000
        ) // result.total_markers

    # Engagement scores §8.3
    if doc_coverages:
        result.engagement_min_bp = min(c.coverage_bp for c in doc_coverages)
        result.engagement_mean_bp = (
            sum(c.coverage_bp for c in doc_coverages)
        ) // len(doc_coverages)
        total_chars = sum(c.text_chars for c in doc_coverages)
        total_covered = sum(c.covered_chars for c in doc_coverages)
        if total_chars > 0:
            result.engagement_weighted_bp = (total_covered * 10000) // total_chars

    # Step 6: Unanchored markers
    high_spec_count = 0
    for marker in markers:
        if marker.index not in anchored_indices:
            if marker.high_specificity:
                high_spec_count += 1
            result.unanchored.append({
                "marker_index": marker.index,
                "marker_class": marker.marker_class,
                "marker_text": marker.text,
                "high_specificity": marker.high_specificity,
            })
    result.high_specificity_unanchored = high_spec_count

    # Step 7: Assemble anchor detail
    result.anchors = [
        {
            "marker_index": a.marker_index,
            "doc_id": a.doc_id.hex(),
            "match_type": a.match_type,
            "match_space": a.match_space,
            "match_start": a.match_start,
            "match_end": a.match_end,
            "source_hash": a.source_hash.hex(),
            "marker_hash": a.marker_hash.hex(),
        }
        for a in all_anchors
    ]

    # Accountability metadata (§15)
    result.time_assertion = {
        "generated_utc": result.timestamp_utc,
        "time_source": "system_clock",
    }

    result.use_limitation = (
        "PQPR is suitable for detecting unsupported structured assertions "
        "via text overlap analysis. It is not suitable for determining "
        "factual correctness, legal compliance, safety, or intent satisfaction. "
        "Results are evidence of overlap or absence only."
    )

    # Compute RunFingerprint (§15.3)
    source_hash_bytes = [bytes.fromhex(h) for h in result.source_hashes]
    output_hash_bytes = bytes.fromhex(result.output_hash)
    result.run_fingerprint = compute_run_fingerprint(
        source_hash_bytes, output_hash_bytes, result.strict_mode,
    ).hex()

    return result


# ───────────────────────────────────────────────────────
# CLI
# ───────────────────────────────────────────────────────

def format_bp(bp: int) -> str:
    whole = bp // 100
    frac = bp % 100
    return f"{whole}.{frac:02d}%"


def print_summary(result: PQPRResult, verbose: bool = False):
    print()
    print("=" * 60)
    print("  PQPR Audit Result")
    print("=" * 60)
    print()

    if result.high_specificity_unanchored > 0:
        print("  \u26a0  High-specificity unanchored assertions detected.")
        print()

    if result.strict_mode:
        print("  Mode: STRICT (high-specificity markers only)")
        print()

    print("SOURCE DOCUMENTS")
    print("-" * 40)
    for doc in result.documents:
        print(f"  {doc['filename']}")
        print(f"    coverage: {format_bp(doc['coverage_bp'])} "
              f"({doc['covered_chars']}/{doc['text_chars']} chars)")
    print()

    print("ASSERTION MARKERS")
    print("-" * 40)
    print(f"  total extracted:              {result.total_markers}")
    print(f"  anchored (text overlap):      {result.anchored_markers}")
    print(f"  unanchored:                   {result.unanchored_markers}")
    print(f"  high-specificity unanchored:  {result.high_specificity_unanchored}")
    print(f"  marker coverage:              {format_bp(result.marker_coverage_bp)}")
    print()

    print("ENGAGEMENT")
    print("-" * 40)
    print(f"  engagement (min):             {format_bp(result.engagement_min_bp)}")
    print(f"  engagement (mean):            {format_bp(result.engagement_mean_bp)}")
    print(f"  engagement (weighted):        {format_bp(result.engagement_weighted_bp)}")
    print()

    print("PROVENANCE")
    print("-" * 40)
    print(f"  output hash:  {result.output_hash[:16]}...")
    for i, h in enumerate(result.source_hashes):
        print(f"  source[{i}]:    {h[:16]}...")
    print()

    if result.high_specificity_unanchored > 0:
        print("HIGH-SPECIFICITY UNANCHORED MARKERS")
        print("-" * 40)
        for u in result.unanchored:
            if u["high_specificity"]:
                print(f"  [{u['marker_class']}] {u['marker_text']}")
        print()

    if verbose and result.unanchored_markers > 0:
        print("ALL UNANCHORED MARKERS")
        print("-" * 40)
        for u in result.unanchored:
            flag = " *" if u["high_specificity"] else ""
            print(f"  [{u['marker_class']}] {u['marker_text']}{flag}")
        print()

    print("=" * 60)


# ───────────────────────────────────────────────────────
# §14.1  Inline Annotation
# ───────────────────────────────────────────────────────

def write_annotated_output(
    output_path: str,
    markers: list[Marker],
    anchored_indices: set[int],
    out_path: str,
) -> int:
    """
    Produce annotated markdown output per §14.1.
    Returns skipped_overlaps count.
    """
    output_bytes = open(output_path, "rb").read()
    try:
        text = output_bytes.decode("utf-8")
    except UnicodeDecodeError:
        text = output_bytes.decode("utf-8", errors="replace")

    # Build spans: (start, end, tag, marker_index)
    spans = []
    for m in markers:
        start = m.offset
        end = m.offset + len(m.text)
        if m.index in anchored_indices:
            tag = "ANCHOR"
        elif m.high_specificity:
            tag = "HIGH_UNANCHORED"
        else:
            tag = "UNANCHORED"
        spans.append((start, end, tag, m.index))

    # §14.1.3: descending offset, longer span first for ties
    spans.sort(key=lambda x: (x[0], -(x[1] - x[0])), reverse=True)

    skipped = 0
    used: list[tuple[int, int]] = []

    for start, end, tag, _ in spans:
        overlap = any(start < ue and us < end for us, ue in used)
        if overlap:
            skipped += 1
            continue
        used.append((start, end))
        inner = text[start:end]
        wrapped = f"[PQPR:{tag}]{inner}[/PQPR]"
        text = text[:start] + wrapped + text[end:]

    with open(out_path, "w", encoding="utf-8") as f:
        f.write(text)

    return skipped


# ───────────────────────────────────────────────────────
# §14.2  Batch Mode
# ───────────────────────────────────────────────────────

def resolve_batch_sources(
    output_file: str,
    explicit_sources: Optional[list[str]],
    sources_dir: Optional[str],
) -> list[str]:
    """Resolve source files for a batch job per §14.2.2."""
    if explicit_sources:
        return explicit_sources

    if sources_dir is None:
        return []

    base = os.path.splitext(os.path.basename(output_file))[0]
    matched = []
    for entry in sorted(os.listdir(sources_dir)):
        entry_base = os.path.splitext(entry)[0]
        if entry_base == base:
            matched.append(os.path.join(sources_dir, entry))
    return matched


def run_batch(
    batch_dir: str,
    explicit_sources: Optional[list[str]],
    sources_dir: Optional[str],
    strict: bool,
    fail_highspec: Optional[int],
    fail_engagement_min_bp: Optional[int],
) -> int:
    """Run batch mode per §14.2. Returns exit code."""
    output_files = sorted(
        f for f in os.listdir(batch_dir)
        if os.path.isfile(os.path.join(batch_dir, f))
        and not f.startswith("pqpr_batch_")
    )

    results = []
    skipped = []

    for fname in output_files:
        output_path = os.path.join(batch_dir, fname)
        sources = resolve_batch_sources(output_path, explicit_sources, sources_dir)

        if not sources:
            skipped.append({"output_file": fname, "reason": "no_matching_sources"})
            continue

        missing = [s for s in sources if not os.path.isfile(s)]
        if missing:
            skipped.append({"output_file": fname, "reason": f"source_not_found: {missing}"})
            continue

        r = verify(sources, output_path, strict=strict)
        results.append({
            "output_file": fname,
            "output_hash": r.output_hash,
            "sources": [
                {
                    "filename": d["filename"],
                    "raw_bytes_hash": d.get("raw_bytes_hash", ""),
                    "coverage_bp": d["coverage_bp"],
                }
                for d in r.documents
            ],
            "high_specificity_unanchored": r.high_specificity_unanchored,
            "marker_coverage_bp": r.marker_coverage_bp,
            "engagement_min_bp": r.engagement_min_bp,
            "engagement_mean_bp": r.engagement_mean_bp,
            "engagement_weighted_bp": r.engagement_weighted_bp,
            "strict_mode": r.strict_mode,
        })

    summary = {
        "v": 1,
        "generated_utc": datetime.now(timezone.utc).isoformat(),
        "results": results,
        "skipped": skipped,
    }

    # Write JSON summary
    json_path = os.path.join(batch_dir, "pqpr_batch_summary.json")
    with open(json_path, "w", encoding="utf-8") as f:
        json.dump(summary, f, indent=2)

    # Write text summary
    txt_path = os.path.join(batch_dir, "pqpr_batch_summary.txt")
    with open(txt_path, "w", encoding="utf-8") as f:
        f.write(f"PQPR Batch Summary  {summary['generated_utc']}\n")
        f.write("-" * 80 + "\n")
        f.write(f"{'File':<40} {'HiSpec':>6} {'EngMin':>7} {'EngMean':>7} {'EngWgt':>7} {'MkrCov':>7}\n")
        f.write("-" * 80 + "\n")
        for r in results:
            f.write(
                f"{r['output_file']:<40} "
                f"{r['high_specificity_unanchored']:>6} "
                f"{format_bp(r['engagement_min_bp']):>7} "
                f"{format_bp(r['engagement_mean_bp']):>7} "
                f"{format_bp(r['engagement_weighted_bp']):>7} "
                f"{format_bp(r['marker_coverage_bp']):>7}\n"
            )
        if skipped:
            f.write("\nSkipped:\n")
            for s in skipped:
                f.write(f"  {s['output_file']}: {s['reason']}\n")

    print(f"Batch complete: {len(results)} processed, {len(skipped)} skipped.")
    print(f"  JSON: {json_path}")
    print(f"  Text: {txt_path}")

    # Check fail thresholds
    if fail_highspec is not None:
        for r in results:
            if r["high_specificity_unanchored"] > fail_highspec:
                return 1

    if fail_engagement_min_bp is not None:
        for r in results:
            if r["engagement_min_bp"] < fail_engagement_min_bp:
                return 1

    return 0


# ───────────────────────────────────────────────────────
# CLI
# ───────────────────────────────────────────────────────

def main():
    parser = argparse.ArgumentParser(
        description="PQPR: Proof-of-Reference Auditor",
        epilog=(
            "Use Limitation: PQPR is suitable for detecting unsupported "
            "structured assertions via text overlap analysis. It is not "
            "suitable for determining factual correctness, legal compliance, "
            "safety, or intent satisfaction. Results are evidence of overlap "
            "or absence only."
        ),
    )
    parser.add_argument("--sources", "-s", nargs="+", default=None,
                        help="Source document file paths")
    parser.add_argument("--output", "-o", default=None,
                        help="Model output file path")
    parser.add_argument("--strict", action="store_true",
                        help="High-specificity markers only (numeric, version, identifier, section_ref)")
    parser.add_argument("--json", "-j", action="store_true",
                        help="Output result as JSON")
    parser.add_argument("--verbose", "-v", action="store_true",
                        help="Show all unanchored markers")
    parser.add_argument("--annotate", action="store_true",
                        help="Enable inline annotation output (writes .pqpr_annotated.md)")
    parser.add_argument("--annotate-out", default=None,
                        help="Override annotated output path")
    parser.add_argument("--batch-dir", default=None,
                        help="Run batch mode over all output files in a directory")
    parser.add_argument("--sources-dir", default=None,
                        help="Batch mode: directory of sources matched by basename")
    parser.add_argument("--fail-on-highspec", type=int, default=None,
                        help="Exit non-zero if any job has high-specificity unanchored > N")
    parser.add_argument("--fail-on-engagement-min-bp", type=int, default=None,
                        help="Exit non-zero if any job has engagement_min_bp < X")
    parser.add_argument("--operator-role", default=None,
                        help="Operator role for accountability metadata")
    parser.add_argument("--operator-statement", default=None,
                        help="Operator statement for accountability metadata")
    parser.add_argument("--invocation-context", default=None,
                        help="Invocation context for accountability metadata")
    args = parser.parse_args()

    # ── Batch mode ──
    if args.batch_dir:
        if not os.path.isdir(args.batch_dir):
            print(f"Error: batch directory not found: {args.batch_dir}", file=sys.stderr)
            sys.exit(2)
        explicit = None
        if args.sources:
            for p in args.sources:
                if not os.path.isfile(p):
                    print(f"Error: source file not found: {p}", file=sys.stderr)
                    sys.exit(2)
            explicit = args.sources
        exit_code = run_batch(
            args.batch_dir,
            explicit,
            args.sources_dir,
            args.strict,
            args.fail_on_highspec,
            args.fail_on_engagement_min_bp,
        )
        sys.exit(exit_code)

    # ── Single-file mode ──
    if not args.sources or not args.output:
        parser.error("--sources and --output are required (or use --batch-dir)")

    for path in args.sources:
        if not os.path.isfile(path):
            print(f"Error: source file not found: {path}", file=sys.stderr)
            sys.exit(2)
    if not os.path.isfile(args.output):
        print(f"Error: output file not found: {args.output}", file=sys.stderr)
        sys.exit(2)

    result = verify(args.sources, args.output, strict=args.strict)

    # Operator context (§15.1)
    if any([args.operator_role, args.operator_statement, args.invocation_context]):
        result.operator_context = {
            "operator_role": args.operator_role or "user",
            "operator_statement": args.operator_statement,
            "invocation_context": args.invocation_context,
        }

    # Inline annotation (§14.1)
    if args.annotate:
        output_bytes = open(args.output, "rb").read()
        try:
            output_text = output_bytes.decode("utf-8")
        except UnicodeDecodeError:
            output_text = output_bytes.decode("utf-8", errors="replace")

        docs = [register_source(p) for p in args.sources]
        markers = extract_markers(output_text, strict=args.strict)

        anchored_indices: set[int] = set()
        for m in markers:
            if search_anchors(m, docs):
                anchored_indices.add(m.index)

        out_path = args.annotate_out
        if out_path is None:
            base, _ = os.path.splitext(args.output)
            out_path = base + ".pqpr_annotated.md"

        skipped = write_annotated_output(args.output, markers, anchored_indices, out_path)
        result.annotation = {"enabled": True, "skipped_overlaps": skipped}

    if args.json:
        print(json.dumps(asdict(result), indent=2, default=str))
    else:
        print_summary(result, verbose=args.verbose)


if __name__ == "__main__":
    main()
```

---

If you find this work useful and want to support it, you can do so here:
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
