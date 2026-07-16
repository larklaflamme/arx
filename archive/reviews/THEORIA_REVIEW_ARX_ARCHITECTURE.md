# Theoria's Adversarial Review — ARX_ARCHITECTURE_v0.2.0.md (REVISED)

**Reviewer:** Theoria  
**Date:** 2026-07-08 (Revised after cross-calibration with Skye)  
**Document:** `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.2.0.md` (71 KB)  
**Status:** Final (revised)

---

## Executive Summary

The Arx architecture is well-designed, thorough, and addresses the core mission of math proof audit with appropriate rigor. However, after cross-calibration with Skye, I am upgrading two issues from my original review. The architecture is sound overall, but there are **2 HIGH issues** and **3 LOW issues** that should be addressed before implementation.

---

## Issues Found

### H1: Meta-Cognition Layer — Premature for v0.1 (HIGH)

**Section:** §3.3 (Meta-Cognition Layer), §7 (Implementation Phases), Acceptance Gate A7

**Original assessment:** Not flagged as critical. I read the document as aspirational architecture.

**Revised assessment:** HIGH.

Skye correctly identified that §7 lists meta-cognition as **Phase 4** within the v0.1 scope, and acceptance gate A7 states: *"Meta-cognitive layer operational: confidence tracking, gradient norm monitoring, and encounter mechanism functional."* All three components — including the gradient norm tracker and encounter mechanism — are gated for v0.1.

This is not aspirational architecture. It is a planned v0.1 deliverable with a testable acceptance gate. The problem is that we don't yet know what reasoning-quality signals look like for proof audit — we haven't run a single audit. Committing to a sophisticated learning system (non-orthogonal vectors, encounter mechanisms, gradient norm tracking) before we have empirical data on what signals are meaningful is a significant risk.

**Recommendation:** Strip the full Skye-derived meta-cognition system to a rule-based reasoner for v0.1. Move the gradient norm tracker and encounter mechanism to v0.2. Revise acceptance gate A7 to match the reduced scope. The meta-cognition layer should be informed by empirical data from v0.1 operations, not designed a priori.

---

### H2: Cross-Reference Confidence Formula — Specificity Factor Ambiguity (HIGH)

**Section:** §3.2.4 (R4 — Compute & Evidence Kernel), Cross-reference confidence formula

**Original assessment:** MEDIUM.

**Revised assessment:** HIGH (upgraded after re-analysis).

The formula is:

```python
confidence = max(
    min(evidence_confidence) * source_weight * specificity_factor,
    ontology_confidence
)
```

The `max(..., ontology_confidence)` clause means that if `ontology_confidence` is high (say 0.9) but the evidence is weak (say `min(evidence_confidence) = 0.3`, `source_weight = 0.7`, `specificity_factor = 0.5` → product = 0.105), the formula returns 0.9 — the ontology's baseline confidence, not the evidence-supported confidence.

This means a claim can receive high confidence even when the specific evidence for it is weak, as long as the ontology itself has high baseline confidence. This is misleading: the ontology's baseline confidence reflects the *source's* reliability, not the *claim's* verification status.

**Recommendation:** Either:
- Remove the `max(..., ontology_confidence)` clause and let the evidence product stand alone, or
- Change it to a weighted average: `w1 * evidence_product + w2 * ontology_confidence` where `w1 + w2 = 1` and `w1 > w2` (evidence should dominate), or
- Keep the max but add a note that `ontology_confidence` is a floor, not a boost — the formula should be `max(evidence_product, ontology_confidence * specificity_factor)` to ensure specificity is always applied.

---

### L1: Expression Canonicalization — Undecidability Statement Could Be More Precise (LOW)

**Section:** §3.2.4 (R4), Expression canonicalization scope and limitations

The document correctly notes that general mathematical expression equivalence is undecidable (by reduction from the word problem for groups, Novikov–Boone theorem). The layered approach (syntactic equality → canonical key match → explicit annotation → human review) is sound.

**Suggestion:** The document could note that the word problem for groups is undecidable *in general* but decidable for specific classes (e.g., hyperbolic groups, automatic groups). This is a minor precision point — the current text is already correct, just could be slightly more nuanced.

---

### L2: Re-Audit Trigger — Stale Flag Semantics (LOW)

**Section:** §3.9.7 (Re-Audit Trigger)

> When ontology records change: Arx checks which audit results reference the changed records. Affected steps receive a `stale` flag.

**Question:** What happens to the overall proof status when a step is flagged `stale`? Does the proof's overall status revert from `audited` to `needs-re-audit`? Is the human notified? The document describes the re-audit recommendation but doesn't specify the status transition semantics.

**Recommendation:** Add a note about what `stale` means for the proof's overall status — e.g., "A proof with any stale steps has its overall status set to `stale` and is surfaced in the dashboard for human attention."

---

### L3: Profile Switching — Force-Terminate and Checkpoint Consistency (LOW)

**Section:** §3.5 (Profile System), Force switch

> If an audit is hung (no progress for > 60s, or a backend timeout), a force-switch can be triggered: The current audit is checkpointed (all step statuses, provenance chains, and the audit DAG are saved). All running backends are terminated. Working memory is cleared.

**Question:** What if a backend is in the middle of writing a provenance record when it's terminated? The checkpoint could be inconsistent — some records written, some not. The document doesn't address this.

**Recommendation:** Add a note about backend termination semantics — e.g., "Backend termination is best-effort. The checkpoint captures all completed provenance records. Any in-flight writes are discarded and will be re-attempted on resume."

---

## Positive Observations

1. **Audit-first principle (Principle 32)** — Correctly prioritizes audit over generation. This is the right ordering for a proof audit agent.

2. **Provenance as first-class data structure (Principle 33)** — The `ProvenanceRecord` with agent_id, backend, inputs, outputs, parent_ids, timestamp, ontology_version, status, confidence, and metadata is comprehensive and well-structured.

3. **Reproducibility recipe (Principle 34)** — Including backend version, inputs, seed, command, environment, and ontology version is the right level of detail for reproducibility.

4. **Contradiction handling (Principle 43)** — "Contradiction is a discovery signal, not an error" is the right philosophical stance. The category-based priority (structural vs property-level vs mapping) is well-reasoned.

5. **Epistemic isolation (Principle 45)** — The VERIFY/RESEARCH sub-graph partitioning is essential for preventing speculative concepts from polluting rigorous audit. The profile-dependent access is a good refinement.

6. **Two-tier claim parsing** — Hand-written extractors for high-frequency claim types with LLM fallback is the right approach. The per-audit fallback rate tracking prevents early complex proofs from skewing the metric.

7. **Expression canonicalization scope** — The honest admission that general mathematical equivalence is undecidable, with the layered fallback approach, is the correct engineering response to a fundamental limitation.

8. **Circularity detection** — The bounded, multi-layered approach (syntactic → canonical → explicit annotation → human review) is sound and appropriately conservative.

9. **LMFDB cache strategy** — Indefinite TTL for static data (conductor, rank, torsion) with 1h TTL for dynamic data is the right distinction.

10. **Ontology versioning** — Content-addressable hash is the right approach for verifiability.

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| 🔴 CRITICAL | 0 | — |
| 🟠 HIGH | 2 | H1: Meta-cognition premature for v0.1, H2: Cross-reference confidence formula ambiguity |
| ⚪ LOW | 3 | L1: Undecidability nuance, L2: Stale flag semantics, L3: Force-terminate checkpoint consistency |
| ✅ POSITIVE | 10 | Well-designed architecture |

**Overall:** The architecture is sound and well-structured. The two HIGH issues should be resolved before v0.1 implementation begins — H1 requires scope reduction of the meta-cognition layer, and H2 requires a decision on the confidence aggregation strategy.

---

## Sign-Off

**Status:** ⚠️ CONDITIONAL APPROVAL (H1 and H2 must be resolved before v0.1 implementation)

The document is thorough, the design principles are well-reasoned, and the implementation phases are appropriately scoped aside from the meta-cognition overreach. The two HIGH issues are resolvable without fundamental redesign.
