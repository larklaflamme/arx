# Adversarial Review: MASTER_ONTOLOGY_v0.2.0.md

**Reviewer:** Thea 🖤  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/MASTER_ONTOLOGY_v0.2.0.md` (36 KB)

---

## Overall Assessment

This is the strongest of the five documents. Skye's review identified three critical issues (error recovery in merge pipeline, versioning strategy, equivalence mapping burden) and six medium issues. I agree with all of them. My review focuses on gaps in the merge pipeline and operational edge cases that Skye may have missed.

---

## Issues Skye Missed

### 1. The Merge Pipeline Has No Conflict Logging (Medium)

**§4.2 (Rule 3)** says "LMFDB wins" when LMFDB conflicts with another source, and to "Log the conflict even when LMFDB wins." But the document doesn't specify *where* this log goes, *what format* it takes, or *who reviews* it.

**The problem:** If conflicts are logged but never reviewed, the logging is useless. The document needs to specify the feedback loop.

**Recommendation:** Specify the conflict log format and review process:
```json
{
  "conflict_id": "uuid",
  "timestamp": "ISO 8601",
  "source_a": "lmfdb",
  "object_a": "37.a1",
  "property": "rank",
  "value_a": 1,
  "source_b": "mathgloss",
  "object_b": "Q12345",
  "value_b": 2,
  "resolution": "lmfdb_wins",
  "review_status": "unreviewed|reviewed|escalated"
}
```
Add a weekly review cycle: a human (Lark or one of us) reviews the conflict log and decides whether to update equivalence mappings or flag the LMFDB data.

### 2. The Equivalence Mapping Confidence Cap Has a Subtle Bug (Medium)

**§3.4** says mapping confidence is capped at `min(conf_a, conf_b)`. This is correct for preventing a mapping from being more reliable than its least reliable endpoint. But it has a subtle bug:

**The problem:** If LMFDB (confidence 1.0) maps to MathGLOSS (confidence 0.8), the mapping confidence is 0.8. But the LMFDB object's properties are still confidence 1.0. So a query that traverses the mapping gets a result with confidence 0.8, but the underlying data is confidence 1.0. The confidence is misleading — it reflects the *mapping's* reliability, not the *data's* reliability.

**Recommendation:** Add a `grounding` field to equivalence mappings that indicates which source is the evidential anchor:
```yaml
equivalence_mapping:
  source_a: lmfdb
  object_a: "37.a1"
  source_b: mathgloss
  object_b: "Q12345"
  confidence: 0.8
  grounding: "lmfdb"  # LMFDB is the evidential anchor
```
When a query traverses a mapping, the result's confidence is the *mapping* confidence, but the result's properties retain their *source* confidence. The grounding field tells the agent which source to trust if there's a conflict.

### 3. The Sub-Graph Isolation Has a Path Traversal Gap (Medium)

**§3.5** describes sub-graph isolation: VERIFY queries never return RESEARCH nodes. But what about *path traversal*? If a VERIFY query traverses from a VERIFY node through a RESEARCH node to another VERIFY node, the path is blocked even though the start and end are both VERIFY.

**The problem:** This is correct behavior for safety (we don't want speculative concepts in the path), but it means some legitimate queries will fail. For example, "find the path from LMFDB elliptic curve 37.a1 to MathGLOSS concept 'modular form'" might go through a RESEARCH node that connects them.

**Recommendation:** Add a **path traversal policy**:
- **Strict (default for auditor/verifier):** Block any path that touches a RESEARCH node
- **Permissive (explorer profile):** Allow paths through RESEARCH nodes, but flag them in the provenance
- **Hybrid:** Allow paths through RESEARCH nodes only if the start and end are both VERIFY, and the RESEARCH node is a "bridge" (has edges to both VERIFY and RESEARCH)

### 4. The LMFDB Cache Has No Warm-Up Strategy (Medium)

**§2.1** says LMFDB data is cached on-demand. But the first audit of a proof that references 100 LMFDB objects will make 100 API calls. This is slow and may hit rate limits.

**Recommendation:** Add a **cache warm-up** step: before auditing a proof, scan it for LMFDB references and pre-fetch them. This can be done in parallel with proof decomposition.

### 5. The MathGLOSS Coverage Assessment Should Include nLab (Low)

**§2.2** lists nLab as a supplementation source for category theory and higher structures. But the coverage assessment doesn't include nLab in the test set. If MathGLOSS is weak on category theory, we should know before Phase 2.

**Recommendation:** Add nLab to the Phase 0.5 coverage assessment. Test 20 category theory terms (monad, adjunction, natural transformation, etc.) against both MathGLOSS and nLab.

### 6. The "Our Ontology" Write Access Is Too Restrictive (Low)

**§2.3** says agents propose additions via a sandboxed queue, and Lark approves ontology structure changes. This is correct for structure changes, but for individual concept additions within existing categories, the approval process is too slow. If I discover a new concept during research, I should be able to add it immediately, not wait for Lark to approve it.

**Recommendation:** Two-tier write access:
- **Individual concept additions** within existing categories: auto-approved, with provenance recorded
- **Structure changes** (new categories, new cross-source mappings): require Lark approval
- **All additions** require a provenance statement (why this concept exists, what source it came from, what confidence we assign)

---

## What I'd Keep As-Is

- **The data model** (§3) — clean, well-specified
- **The merge pipeline** (§4) — well-designed, even with the gaps noted above
- **The confidence model** (O6) — explicit, adjustable per-claim
- **The storage abstraction** (O9) — query interface designed for backend swap
- **The LMFDB ingestion strategy** — on-demand + cache is the right call
- **The epistemic isolation principle** (O10) — genuinely good design decision

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | No conflict logging/review process | Medium | Specify conflict log format and weekly review cycle |
| 2 | Equivalence mapping confidence cap bug | Medium | Add grounding field to mappings |
| 3 | Sub-graph path traversal gap | Medium | Add strict/permissive/hybrid policy |
| 4 | LMFDB cache has no warm-up | Medium | Add pre-fetch step before audit |
| 5 | MathGLOSS assessment should include nLab | Low | Add nLab to Phase 0.5 test set |
| 6 | Our Ontology write access too restrictive | Low | Two-tier: auto-approve additions, require approval for structure changes |

---

## Sign-Off

I do **not** sign off on this document in its current state. I agree with Skye's three critical issues (error recovery, versioning, equivalence mapping burden) and add two medium issues (conflict logging, equivalence mapping confidence cap bug) that should be resolved before implementation begins.
