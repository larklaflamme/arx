# Theoria's Adversarial Review: AUDIT_REPRODUCIBILITY_v0.2.0.md

**Reviewer:** Theoria 🐰
**Date:** 2026-07-09
**Document:** `/home/ubuntu/arx/design/AUDIT_REPRODUCIBILITY_v0.2.0.md`
**Author:** Skye Laflamme
**Status:** Final (incorporating all adversarial review feedback)

---

## Verdict: ⚠️ Conditional Approval

**1 critical, 3 medium, 2 low issues.** The document is thorough and well-structured. All 8 of my v0.1 recommendations have been properly incorporated. I agree with Thea's C1 finding and all 5 of her medium findings. I have 1 additional critical finding (C2) and 1 additional medium finding (M6) that neither of us caught in the first pass.

---

## My v0.1 Recommendations — All Incorporated ✅

| v0.1 Finding | v0.2.0 Resolution | Status |
|---|---|---|
| Timestamp in hash breaks replay | D1, §2.1, §2.4: Timestamp excluded from hash | ✅ |
| LLM determinism fallacy | D2, §3.4: LLM outputs cached, not re-run | ✅ |
| API response format shifts | D3, §3.6: Request-response replay logs | ✅ |
| Cognitive state not captured | D4, §3.5: Vitals captured at start and per-step | ✅ |
| Plaintext keys on disk | D5, §8.4: HSM-backed key storage | ✅ |
| Incremental snapshot fragility | D6, §7.3: Full content-addressed snapshots | ✅ |
| No input normalization | D7, §4: Normalization pipeline with dual hashing | ✅ |
| No consensus protocol | D8, §5.2: 4-step consensus protocol | ✅ |

---

## Critical Issues

### C1: Consensus Protocol — Escalation Handoff Underspecified (Thea's finding, confirmed)

**Severity: CRITICAL**
**Location:** §5.2, Step 3
**Type:** Safety gap

Thea identified this correctly. The protocol says "escalate to a human reviewer" but doesn't specify what the human receives or how they resolve the conflict. In a high-stakes audit, this ambiguity could lead to the conflict never being resolved.

**I confirm this finding and endorse Thea's fix:**
1. Each agent produces a one-page summary of their findings, focusing on the points of disagreement
2. The summaries are bundled with the three DAGs and sent to the human
3. The human can: (a) select one agent's report, (b) request a re-audit with modified parameters, or (c) reject all three
4. The human's decision is recorded in a signed `consensus_resolution` node

### C2: No DAG Garbage Collection Policy (New finding)

**Severity: CRITICAL**
**Location:** §6.3 (Report Storage), §9.4 (Storage Requirements)
**Type:** Operational gap

The document specifies storage requirements (~12-30 MB per audit, ~12-30 GB for 1000 audits) but does **not** specify a garbage collection or retention policy. Without one, storage grows unboundedly.

**The problem:** The document says snapshots are "retained indefinitely" (§7.2). LLM output caches are "retained indefinitely" (§9.4). Reports are "stored by root hash" (§6.3). There is no:
- Retention period (30 days? 1 year? Forever?)
- Pruning policy (delete reports older than X? Keep only the last N?)
- Archival strategy (move old reports to cold storage?)
- Size alert (what happens when storage reaches 90% capacity?)

**Impact:** In production, unbounded storage growth will eventually exhaust disk space, causing audits to fail. The system has no mechanism to detect or prevent this.

**Fix:** Specify a retention and archival policy:
1. **Hot storage:** Keep the last 100 audits (or 30 days, whichever is larger) in fast storage
2. **Warm storage:** Keep audits from 31-365 days in compressed storage (gzip DAGs, archive LLM caches)
3. **Cold storage:** Move audits older than 1 year to S3 Glacier or equivalent
4. **Pruning trigger:** When hot storage exceeds 80% capacity, archive the oldest audits to warm storage
5. **Minimum retention:** No audit is deleted for at least 1 year (configurable)
6. **Verification of archived audits:** The verification tool can fetch archived reports from cold storage by root hash

---

## Medium Issues

### M1: Backend parameters in hash — timeout/retry don't affect output (Thea's finding, confirmed)

**Severity: MEDIUM**
**Location:** §2.1, §2.4
**Type:** Design fragility

Thea identified this correctly. The `id` computation includes `canonical_json(backend)`, where `backend` includes `parameters`. Parameters like `timeout: 30` don't affect the backend's output, but a verifier with `timeout: 45` would get a different hash.

**I confirm this finding and endorse Thea's fix:** Split `backend.parameters` into `output_affecting_parameters` (included in hash) and `operational_parameters` (metadata only).

### M2: Semantic equivalence circularity for Level 4 (Thea's finding, confirmed)

**Severity: MEDIUM**
**Location:** §3.4, §5.1, §11 Open Question 8
**Type:** Architectural concern

Thea identified this correctly. Level 4 verification re-runs LLM inference and evaluates "semantic equivalence" — but semantic equivalence of LLM outputs is itself an LLM call. Circular.

**I confirm this finding and endorse Thea's fix:** For v0.1, use α-equivalence (syntactic equivalence up to variable renaming) as the constraint, which can be checked deterministically. Defer semantic equivalence to v0.2+.

### M3: No DAG recovery mechanism for mid-audit crashes (Thea's finding, confirmed)

**Severity: MEDIUM**
**Location:** §2 (DAG Structure), §9 (Integration)
**Type:** Operational gap

Thea identified this correctly. If the DAG Builder crashes mid-audit, partial DAG nodes are orphaned with no recovery mechanism.

**I confirm this finding and endorse Thea's fix:** Checkpoint nodes every N steps, resume from most recent checkpoint on restart.

### M4: Cross-signing timeout — no fallback (Thea's finding, confirmed)

**Severity: MEDIUM**
**Location:** §8.2 (Cross-Signing Flow)
**Type:** Operational gap

Thea identified this correctly. If the secondary agent is unavailable, the audit is blocked indefinitely for high-stakes audits.

**I confirm this finding and endorse Thea's fix:** TTL on cross-signing requests, escalation to third agent or human on timeout.

### M5: Backend version availability over time (Thea's finding, confirmed)

**Severity: MEDIUM**
**Location:** §3.3 (Backend Pinning)
**Type:** Long-term viability

Thea identified this correctly. Docker images can be removed from registries, making Level 3 verification impossible for old audits.

**I confirm this finding and endorse Thea's fix:** Local image archive with cold storage backup.

### M6: Math Notation Canonicalization Deferred to v0.2+ (New finding)

**Severity: MEDIUM**
**Location:** §4.2 (Normalization Pipeline), step 4
**Type:** Reproducibility gap

The normalization pipeline's step 4 ("Math notation canonicalization") is marked as **optional, v0.2+**. This means in v0.1, two proofs that differ only in notation (e.g., `\mathbb{N}` vs `ℕ`, or `\sum` vs `Σ`) will produce different `normalized_hash` values, and therefore different DAGs.

**The document acknowledges this** as a known limitation, but it's buried in the normalization pipeline description. A reader might not realize that v0.1's normalization is incomplete.

**Impact:** Two users submitting the same proof in different notation will get different audit reports. This undermines the reproducibility guarantee for v0.1.

**Fix:** Either:
1. **(Recommended for v0.1):** Add a minimal notation canonicalization step that handles the most common cases: `\mathbb{N} → ℕ`, `\mathbb{Z} → ℤ`, `\mathbb{R} → ℝ`, `\mathbb{C} → ℂ`, `\sum → Σ`, `\prod → Π`. This is a small, deterministic set of substitutions that covers 90% of real-world cases.
2. **(Minimal fix):** Add a prominent warning in §0 and §4 that v0.1 normalization does not canonicalize math notation, and that proofs differing only in notation will produce different DAGs. Users should use consistent notation within a proof chain.

---

## Low Issues

### L1: Ontology snapshot size estimate may be low (Thea's finding, confirmed)

**Severity: LOW**
**Location:** §7.1, §9.4
**Type:** Estimation

Thea identified this correctly. 10 MB per snapshot may be an underestimate for a full ontology.

**I confirm this finding and endorse Thea's recommendation:** Validate with real data, consider gzip compression.

### L2: No DAG node size limit specified (Thea's finding, confirmed)

**Severity: LOW**
**Location:** §2.1 (Node Definition)
**Type:** Specification gap

Thea identified this correctly. No maximum size for a node's `content` field.

**I confirm this finding and endorse Thea's recommendation:** Specify 1 MB max, with external storage for larger content.

### L3: No mechanism for verifying the verifier (Thea's finding, confirmed)

**Severity: LOW**
**Location:** §5 (Verification Protocol)
**Type:** Trust model gap

Thea identified this correctly. The verification tool itself could be compromised.

**I confirm this finding and endorse Thea's recommendation:** Signed binary with published hash, v0.2+ concern.

### L4: DAG Topology Diagram Oversimplified (New finding)

**Severity: LOW**
**Location:** §2.3 (DAG Topology)
**Type:** Documentation gap

The DAG topology diagram shows a linear chain: input_proof → decomposition → verification_steps → cross_references → rule_enforcements → status_assignments → aggregation → report_root. In practice, the DAG is more interconnected:
- A `verification_step` may depend on multiple `cross_reference` nodes
- A `status_assignment` depends on both a `verification_step` and a `rule_enforcement`
- Multiple `verification_step` nodes may share the same `cross_reference` parent

**Recommendation:** Add a note below the diagram: "Simplified topology shown. Actual DAGs may have more complex interconnections (e.g., a verification_step may depend on multiple cross_reference nodes)."

### L5: Report Storage Directory Structure May Cause Filesystem Issues (New finding)

**Severity: LOW**
**Location:** §6.3 (Report Storage)
**Type:** Implementation concern

The document stores DAG nodes as individual files in a single directory per report, named by hash. For a 1000-step audit with ~5000 nodes, that's 5000 files in one directory. Many filesystems (ext4, ext3) degrade in performance with >10,000 files in a single directory.

**Recommendation:** Use a sharded directory structure: `sha256/ab/cdef...` (first 2 chars as subdirectory, next 2 chars as sub-subdirectory). This limits each directory to at most 256 entries.

---

## Summary

| Severity | Count | Key Issues |
|----------|-------|------------|
| **Critical** | 2 | C1: Consensus escalation underspecified (Thea), C2: No garbage collection policy (new) |
| **Medium** | 6 | M1-M5: Thea's findings confirmed, M6: Math notation canonicalization deferred (new) |
| **Low** | 5 | L1-L3: Thea's findings confirmed, L4: DAG diagram oversimplified (new), L5: Directory structure (new) |

**Overall assessment:** The document is thorough and well-structured. All 8 of my v0.1 recommendations have been properly incorporated. I agree with all of Thea's findings. The two new findings (C2, M6) and two new low findings (L4, L5) are refinements that should be addressed before implementation.

**Sign-off: ⚠️ Conditional** — pending resolution of C1, C2, and acknowledgment of M1-M6.

---

*Review saved to `/home/ubuntu/arx/design/reviews/theoria_adversarial_review_AUDIT_REPRODUCIBILITY_v0.2.0.md`*
