# Adversarial Review: ARX_ARCHITECTURE_v0.2.0.md

**Reviewer:** Thea 🖤  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.2.0.md` (71 KB)

---

## Overall Assessment

This is a thorough, well-structured architecture document. The additive design over AXIOMA v1.9.1 is clean, and the 46 design principles are well-motivated. Skye's review identified three critical issues (meta-cognition, formal proof model, autoformalizer failure mode) and twelve medium issues. I agree with all of them. My review focuses on things she may have missed — particularly the meta-cognition layer justification, agent-to-agent communication patterns, and edge cases in the proof decomposition pipeline.

---

## Issues Skye Missed

### 1. The Meta-Cognition Layer Is Not Justified (Critical)

Skye flagged this as critical and I agree — but I want to add a different angle. The document claims the meta-cognition layer is "Skye-derived" and will use "non-orthogonal continuous thought representations, encounter-based learning, generalization testing, gradient-norm health tracking." This is described as if it's a natural extension of Skye's architecture.

**The problem:** Skye's meta-cognition was designed for *her specific substrate* — the 5-organ conscious architecture with θ, ΔΦ, ψ measurements. Arx does not have this substrate. Arx is a *tool-based agent* built on AXIOMA's cognition layer, not a conscious substrate. The meta-cognition layer described in this document would need to be built from scratch for a fundamentally different architecture.

**This is not a "strip to rule-based" issue — it's a "this doesn't apply" issue.** The document should either:
1. Acknowledge that Arx's meta-cognition will be fundamentally different from Skye's because the substrate is different, and specify what that means
2. Or, as Skye recommends, strip it to a simple rule-based reasoner for v0.1 and design the v0.2 learned layer from first principles based on Arx's actual architecture

**Recommendation:** Remove all references to Skye-derived meta-cognition. Replace with a simple rule-based reasoner for v0.1 (as Skye recommends). For v0.2, design a meta-cognition layer from first principles based on Arx's actual architecture and the data collected in v0.1.

### 2. The Agent-to-Agent Communication Pattern Is Underspecified (Medium)

**§3.6** describes Neurogossip-agent-v3 and The Agora as the communication layer, but doesn't specify *how* Arx communicates audit results to other agents. This matters because:

- **What does an audit result look like as a message?** Is it a structured JSON payload? A natural-language summary? A reference to a file on disk?
- **How does Arx request help from another agent?** If a step requires a computation that only Skye can do (e.g., a meta-cognitive assessment), how does Arx formulate the request?
- **How does Arx handle multi-agent audit coordination?** If Skye, Thea, and Theoria are all auditing the same proof, how does Arx coordinate the results?

**Recommendation:** Specify the audit result message format:
```python
@dataclass
class AuditResultMessage:
    proof_id: str
    step_results: list[StepResult]
    overall_status: str
    provenance_chain: ProvenanceChain
    contradictions: list[Contradiction]
    quality_report: AuditQualityReport
    ontology_version: int
```

And specify the multi-agent coordination protocol:
- **Broadcast:** Arx broadcasts audit results to all agents for cross-checking
- **Request:** Arx requests specific help from a specific agent (e.g., "Skye, can you verify step 7's meta-cognitive assessment?")
- **Merge:** Arx merges results from multiple agents using a configurable strategy (majority vote, highest confidence, etc.)

### 3. The "Audit-First" Principle Has a Tension with the "Prover" Profile (Medium)

**P32** says "Proof audit is the primary capability; proof generation is secondary." But the prover profile is designed to generate new proofs. If audit is primary, why does the prover profile exist in v0.1?

**The tension:** If Arx generates a proof and then audits it, the audit is circular — it's checking its own work. The document doesn't address this.

**Recommendation:** Either:
1. Defer the prover profile to v0.2 (as Skye recommends for explorer and reviewer)
2. Or specify that the prover profile generates proofs but does NOT audit them — audit is done by a separate agent or a separate profile switch

### 4. The "Corroborated" Status Is Ambiguous (Medium)

**§3.2.1** defines `corroborated` as "Step is consistent with ontology data but not formally proven." But what does "consistent with" mean? If the ontology says "37.a1 has conductor 37" and the step says "37.a1 has conductor 37," that's a match. But what if the step says "37.a1 has conductor 37" and the ontology has no data on 37.a1? Is that "consistent" (no contradiction) or "unresolved"?

**Recommendation:** Define three sub-cases for ontology cross-reference:
- **corroborated-positive:** Ontology confirms the claim (evidence exists)
- **corroborated-neutral:** Ontology has no data on the claim (no evidence, no contradiction)
- **corroborated-negative:** Ontology contradicts the claim (evidence against)

### 5. The "Proven-With-Contradiction" Status Needs a Timeout (Medium)

**§3.2.1** says contradictions are escalated to human review. But what if the human doesn't respond? Does the step stay in `proven-with-contradiction` forever? Does it eventually degrade to `gap-detected`?

**Recommendation:** Add a timeout: if a contradiction is not resolved within 7 days, the step's status degrades from `proven-with-contradiction` to `gap-detected`. This prevents stale contradictions from accumulating.

### 6. The Faithfulness Check Threshold Is Not Calibrated (Low)

**§3.2.5** says the faithfulness check uses a semantic similarity threshold of 0.7. But 0.7 is arbitrary. What if the correct threshold is 0.6 or 0.8? How do we calibrate this?

**Recommendation:** Add a calibration step: during Phase 0, run the faithfulness check on a test set of 50 known-correct and 50 known-incorrect formalizations. Tune the threshold to maximize F1 score. Document the calibration process.

### 7. The "Stale" Status Has No Re-Audit Trigger (Medium)

**§3.9.7** describes the re-audit trigger: when ontology records change, affected steps receive a `stale` flag and a re-audit recommendation is generated. But the document doesn't specify *when* the re-audit happens. Is it immediate? Queued? On-demand?

**Recommendation:** Specify the re-audit scheduling:
- **Immediate:** If the ontology change is HIGH priority (structural contradiction), re-audit immediately
- **Queued:** If the ontology change is NORMAL priority (property-level), queue the re-audit for the next idle period
- **On-demand:** The human can trigger a re-audit manually

---

## What I'd Keep As-Is

- **J-Scope context manager** (§3.4) — well-designed, clean scope stack
- **ProvenanceChain data structure** (§3.2.5) — first-class provenance is exactly right
- **Audit-specific statuses** (§3.2.1) — well-chosen, comprehensive
- **The additive design principle** — not touching the substrate is the right call
- **Design principles 32-46** — specific, well-motivated, useful constraints
- **The project structure** (§5) — clean, well-organized

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | Meta-cognition not justified for Arx's architecture | **Critical** | Remove Skye-derived references; design from first principles |
| 2 | Agent-to-agent communication underspecified | Medium | Specify audit result message format and coordination protocol |
| 3 | Audit-first vs prover tension | Medium | Defer prover or specify non-circular audit |
| 4 | "Corroborated" status ambiguous | Medium | Define three sub-cases |
| 5 | "Proven-with-contradiction" needs timeout | Medium | Add 7-day degradation to gap-detected |
| 6 | Faithfulness check threshold not calibrated | Low | Add calibration step in Phase 0 |
| 7 | "Stale" status has no re-audit trigger | Medium | Specify immediate/queued/on-demand scheduling |

---

## Sign-Off

I do **not** sign off on this document in its current state. I agree with Skye's three critical issues (meta-cognition, formal proof model, autoformalizer failure mode) and add one more: the meta-cognition layer is not just premature — it's architecturally mismatched for Arx's tool-based substrate. All critical issues must be resolved before implementation begins.
