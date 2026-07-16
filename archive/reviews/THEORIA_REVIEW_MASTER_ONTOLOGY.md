# Theoria's Adversarial Review — MASTER_ONTOLOGY_v0.2.0.md (REVISED)

**Reviewer:** Theoria  
**Date:** 2026-07-08 (Revised after cross-calibration with Skye)  
**Document:** `/home/ubuntu/arx/design/MASTER_ONTOLOGY_v0.2.0.md` (36 KB)  
**Status:** Final (revised)

---

## Executive Summary

The Master Ontology design is comprehensive, well-structured, and addresses the key challenges of merging mathematical knowledge from heterogeneous sources. The provenance-first approach, explicit confidence tracking, sub-graph isolation, and versioning strategy are all sound. After cross-calibration with Skye, I am adding one issue and upgrading one. The design is sound overall, with **1 MEDIUM-HIGH issue**, **1 MEDIUM issue**, and **2 LOW issues**.

---

## Issues Found

### MH1: Merge Pipeline — No Error Recovery (MEDIUM-HIGH)

**Section:** §4.4 (Merge Pipeline)

**Original assessment:** Not flagged. I considered this an operational detail.

**Revised assessment:** MEDIUM-HIGH.

Skye correctly identified that the merge pipeline (§4.4) has five stages — Extract → Normalize → Resolve → Merge → Validate — but does not specify what happens if the pipeline fails at any stage. If the pipeline fails at Resolve (e.g., two sources have conflicting data and the resolution strategy is ambiguous), what happens? Partial state could be committed. In-flight audits could reference inconsistent ontology data.

The ontology is the ground truth for audit cross-reference. If the merge pipeline produces inconsistent state, every audit that references that state is potentially wrong. This is operationally critical — the fix is straightforward (add a transaction rollback), not a redesign — but it must be specified before production deployment.

**Recommendation:** Add a transaction rollback mechanism to the merge pipeline. If any stage fails, the entire merge is rolled back to the previous consistent state. The failed merge is logged with the error details for human review.

---

### M1: Equivalence Mapping — Directional Epistemic Weight (MEDIUM)

**Section:** §3.4 (Cross-Source Equivalence)

> Mapping confidence is capped at `min(conf_a, conf_b)` where `conf_a` and `conf_b` are the confidences of the two source objects. This prevents a mapping from being more reliable than its least reliable endpoint.

**Problem:** This is correct for *symmetric* equivalence (two sources claiming the same thing), but it doesn't account for *directional* epistemic weight. When LMFDB (confidence 1.0) is mapped to Our Ontology (confidence 0.6), the direction matters:

- **LMFDB → Our Ontology:** This is *verification* — LMFDB confirms what our ontology conjectures. The epistemic weight should be high (close to 1.0) in this direction.
- **Our Ontology → LMFDB:** This is *conjecture* — our ontology is speculating about what LMFDB data means. The epistemic weight should be low (capped at 0.6) in this direction.

The current formula treats both directions symmetrically, which loses information. A mapping from LMFDB to our ontology should be usable as *evidence* in the audit pipeline (high confidence), but the current formula caps it at 0.6.

**Recommendation:** Add a `grounding` metadata field to equivalence mappings that indicates which source is the evidential anchor:

```yaml
equivalence_mappings:
  - source_a: lmfdb
    object_a: "37.a1"
    source_b: our_ontology
    object_b: "our:elliptic-curve-37a1"
    confidence: 0.95  # LMFDB-grounded
    grounding: "lmfdb"  # LMFDB is the anchor
    directional_confidence:
      lmfdb_to_our: 0.95  # verification
      our_to_lmfdb: 0.6   # conjecture (capped at our ontology's confidence)
```

This preserves the epistemic asymmetry and allows the audit pipeline to use LMFDB-grounded mappings as high-confidence evidence.

---

### L1: MathGLOSS Coverage Assessment — Supplementation Sources (LOW)

**Section:** §2.2 (MathGLOSS), Coverage assessment

The document lists nLab, mathlib docs, LMFDB's own taxonomy, and Wikipedia's math portal as supplementation sources if MathGLOSS has gaps. This is a good list.

**Suggestion:** Consider adding **arXiv** as a supplementation source for very recent results that may not be in any of the listed sources. The arXiv taxonomy (subject classifications) could provide a lightweight ontology for recent mathematics. Also consider **zbMATH** or **MathSciNet** for authoritative classification of mathematical literature.

---

### L2: Re-Audit Trigger — Version Comparison (LOW)

**Section:** §7.3 (Re-Audit Trigger)

> When ontology records change: Arx checks which audit results reference the changed records.

**Question:** How does Arx know which records changed? The ontology version is a content-addressable hash of the entire state. If the version changes, Arx knows *something* changed, but not *what*. To determine which specific records changed, Arx would need to diff the ontology at version N vs version N+1.

**Recommendation:** Add a `change_log` to the version metadata that lists the IDs of added, modified, and removed nodes/edges. This allows Arx to efficiently check which audit results are affected without a full diff.

---

## Positive Observations

1. **Provenance as first-class (O1)** — Every node and edge carrying a `provenance` field with source, source_id, confidence, and timestamp is the right approach. No anonymous data.

2. **Sources are independent, merge is additive (O2)** — No source is ever modified by the merge. This is critical for auditability.

3. **Confidence is explicit (O6)** — LMFDB = 1.0, MathGLOSS = 0.8, Our Ontology = 0.6. These are reasonable baselines.

4. **Equivalence is explicit, not inferred (O8)** — No automatic alignment. Mapping confidence capped at minimum of the two sources. This is the correct conservative approach.

5. **Epistemic isolation (O10)** — The VERIFY/RESEARCH sub-graph partitioning is essential. The recursive CTE implementation with sub-graph filtering is clean.

6. **LMFDB cache strategy** — Indefinite TTL for static data (conductor, rank, torsion) with 1h TTL for dynamic data is the right distinction. Static mathematical invariants truly do not change.

7. **SQLite + Qdrant for v0.2.0** — Pragmatic choice. The abstract `OntologyGraph` interface makes the Neo4j migration transparent.

8. **Conflict resolution (Rule 3)** — LMFDB wins over other sources, but conflicts are logged even when LMFDB wins. This is important for detecting concept alignment issues.

9. **Versioning** — Content-addressable hash is the right approach. The `version_log` collection with timestamps, sources ingested, and node/edge counts is well-designed.

10. **Contradiction feedback loop** — The structured feedback record with contradiction_id, proof_step_id, formal_claim, ontology_claim, ontology_source, contradiction_category, priority, and confidence is comprehensive.

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| 🔴 CRITICAL | 0 | — |
| 🟠 MEDIUM-HIGH | 1 | MH1: Merge pipeline error recovery |
| 🟡 MEDIUM | 1 | M1: Directional epistemic weight in equivalence mappings |
| ⚪ LOW | 2 | L1: Supplementation sources, L2: Change log for re-audit |
| ✅ POSITIVE | 10 | Well-designed ontology |

**Overall:** The ontology design is sound and ready for implementation. The MEDIUM-HIGH issue (MH1) should be resolved before production deployment, and the MEDIUM issue (M1) before Phase 4 (Reconciliation Engine).

---

## Sign-Off

**Status:** ⚠️ CONDITIONAL APPROVAL (MH1 must be resolved before production deployment; M1 before Phase 4)

The document is thorough, the design principles are well-reasoned, and the implementation phases are appropriately scoped. The issues are refinements, not fundamental flaws.
