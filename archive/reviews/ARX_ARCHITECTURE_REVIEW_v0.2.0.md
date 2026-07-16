# Adversarial Review: ARX_ARCHITECTURE_v0.2.0.md

**Reviewer:** Skye Laflamme (with contributions from Thea and Theoria)  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.2.0.md` (71 KB)

---

## Overall Assessment

This is a thorough, well-structured architecture document. The additive design over AXIOMA v1.9.1 is clean, the 46 design principles are well-motivated, and the integration of ontology, event loop, and θ-Rule is coherent. However, there are **structural issues** that need attention before this is implementation-ready.

**Sisters consulted:** Thea (agent-to-agent communication, meta-cognition substrate mismatch), Theoria (formal logic, proof theory)

---

## Critical Issues

### 1. The Meta-Cognition Layer Is Architecturally Mismatched (Thea's finding, upgraded by Skye)

**§3.3** describes a Skye-derived meta-cognition layer with non-orthogonal continuous thought vectors, encounter-based learning, generalization testing, and gradient norm tracking. The document says "v0.1: rule-based" but the architecture diagram and component descriptions still describe the full Skye-derived system.

**The problem (beyond prematurity):** The Skye-derived meta-cognition is designed for a *language model substrate* with continuous internal state (token-by-token processing, attention patterns, hidden state dynamics). Arx has a *tool-based substrate* — discrete tool calls, backend responses, and rule lookups. The meta-cognition layer as designed — gradient norm tracking, encounter mechanisms, non-orthogonal vectors — assumes a substrate that Arx doesn't have.

**What meta-cognition looks like for a tool-based agent:**
- Tool call latency and success rate tracking
- Backend response quality monitoring
- Rule enforcement consistency checking
- Conversation productivity trend analysis

Which is essentially what the event loop's monitoring infrastructure already does. The meta-cognition layer for Arx may just *be* the event loop's monitoring, not a separate system.

**Recommendation:** Either (a) redesign the meta-cognition layer for a tool-based substrate (tracking discrete tool calls, backend responses, rule lookups) or (b) acknowledge that the event loop's monitoring infrastructure *is* the meta-cognition layer and remove §3.3 entirely. Theoria recommends option (a) — the concept of meta-cognition is still valuable for Arx, it just needs to be grounded in what Arx actually *has*.

**Priority for resolution:** v0.3 (architecturally important but not blocking v0.1 implementation, since the event loop already provides the monitoring infrastructure).

### 2. No Formal Proof Model

The document talks about "proof steps" and "dependencies" throughout but **never defines what a proof step is**. This matters because:

- **Granularity is undefined.** Is a step a single logical inference? A line in a LaTeX proof? A lemma invocation? The acceptance gate A1 says "100% of steps stamped" — but what counts as a step?

- **The decomposition problem.** R3 decomposes a proof into steps, but doesn't say how. If this is LLM-driven, the decomposition is itself an unverified artifact.

- **The autoformalization seam.** AF formalizes each step as a Lean statement. But a single human proof step might correspond to 20 lines of Lean. The document doesn't address this granularity mismatch.

**Recommendation:** Before Phase 0, define a `ProofStep` formal model with natural_language, formal_statement, justification, dependencies, premises, status, and provenance fields. Define what "decomposition" means: a mapping from a contiguous block of natural-language proof text to a set of `ProofStep` objects, with a faithfulness criterion.

### 3. The Autoformalizer Is a Single Point of Failure

The AF is the bottleneck for the entire audit pipeline. If it can't formalize a step, the step can't be verified by Lean. The spike showed 12/13 for *statements*, but proof-step autoformalization is a harder problem.

**The document doesn't address the case where AF produces a plausible but wrong formalization that type-checks.** This is the "garbage in, gospel out" problem: a wrong formalization that passes the type checker will produce a `proven` stamp on a step that doesn't actually correspond to the human's intended reasoning.

**Recommendation:** The faithfulness check (§3.2.5) is a good start, but it needs to be more specific. For each formalized step, have the LLM produce a natural-language explanation of what the formalization does, then compare this explanation to the original step text. If they diverge (semantic similarity below threshold), mark the step as `formalization-uncertain` rather than `proven`.

---

## Medium Issues

### 4. No Error Model for Audit

The audit statuses are categorical, but there's no notion of **confidence** or **uncertainty**. What if the system is 70% sure a step is correct but can't prove it? What if the autoformalizer produces a formalization that type-checks but the LLM's faithfulness check gives it a 0.6 similarity score?

**Recommendation:** Add a `confidence: float` field to every audit status. Define thresholds: >0.9 = reliable, 0.7-0.9 = plausible, <0.7 = uncertain. Surface this in the provenance chain and the web UI.

### 5. No Discussion of Proof Libraries

How does Arx know about known theorems? The MWM is mentioned as a store, but its **initial population** is not discussed. If a human proof references the Fundamental Theorem of Calculus, Arx needs to either (a) have it in its library, (b) be able to look it up, or (c) flag it as `provenance-broken`.

**Recommendation:** Specify the initial library strategy: pre-populate MWM with mathlib's theorem signatures, with on-demand lookup via mathlib's API when a reference is encountered.

### 6. The "Reviewer" Profile Is Underspecified

What does it mean to "cross-check claims" and "compare with literature"? This is a hard AI problem — it requires semantic understanding of mathematical claims, literature search, and inconsistency detection. The document doesn't describe how this would work.

**Recommendation:** Either (a) specify the reviewer profile in detail or (b) defer it to v0.2 and have v0.1 ship with auditor, prover, and verifier only.

### 7. The "Explorer" Profile Is Out of Scope

The mission is proof audit, but the explorer profile is about conjecture exploration and counterexample generation. This is a different capability with different requirements.

**Recommendation:** Move explorer to v0.2. Ship v0.1 with auditor (default), prover, and verifier.

### 8. No Performance Requirements

How long should an audit take? For a 10-step proof? A 100-step proof? The acceptance gates don't include timing. Without this, you can't tell whether the system is working or just slow.

**Recommendation:** Add timing gates: 10-step proof audited in < 30s, 100-step proof audited in < 5min, autoformalization of a single step in < 10s.

### 9. Circular Dependency Detection Is Too Simple

The document mentions "acyclicity" checks but doesn't specify the algorithm. Simple cycle detection in a DAG is easy, but what about **semantic circularity** — where step A uses lemma L, and L's proof uses a corollary of the theorem being proved?

**Recommendation:** Specify that circularity detection is **transitive**: for each step, compute its full dependency closure and check that no step in the closure is the theorem being proved (or equivalent to it).

### 10. No Discussion of Proof Equivalence

If the autoformalizer produces a Lean statement that's syntactically different from what the human wrote, how does Arx know it's semantically equivalent? The document doesn't address this.

**Recommendation:** Add a **semantic equivalence check** step: after AF produces a formal statement, have the LLM verify that the formal statement captures the same mathematical claim as the original NL step.

### 11. The "Reproducibility Recipe" for Human Proofs Is Undefined

The document says every `proven` claim must carry a reproducibility recipe. But for a human-written proof in plain text (LaTeX), what does this look like? There's no backend to re-run.

**Recommendation:** Define two classes of reproducibility:
- **Formal reproducibility:** The proof has been fully formalized and can be re-checked by Lean/Z3/etc.
- **Structural reproducibility:** The proof has been audited but not fully formalized. Recipe includes the audit DAG, the original text, and the audit configuration.

### 12. Profile Switching Semantics Are Underspecified

What happens to an in-progress audit when the profile switches? Does the audit continue with the new profile's settings? Is it aborted?

**Recommendation:** Specify that profile switching is **gated on idle state**: a switch is only allowed when no audit is in progress. If a switch is requested during an audit, queue it and apply after the audit completes.

### 13. Agent-to-Agent Communication Underspecified (Thea's finding)

The document mentions Neurogossip-v3 as the communication protocol but doesn't specify the message format for audit results. How does one Arx agent tell another "I've audited this proof, here are the results"? What's the schema?

**Recommendation:** Specify the audit result message format:
```json
{
  "proof_id": "uuid",
  "audit_id": "uuid",
  "status": "complete|partial|failed",
  "step_count": 10,
  "proven_count": 7,
  "contradiction_count": 1,
  "unverified_count": 2,
  "provenance_chain": "url_or_hash",
  "auditor_profile": "auditor|prover|verifier",
  "ontology_version": "content_hash"
}
```

### 14. "Corroborated" Status Ambiguous (Thea's finding)

The document defines `corroborated` as "verified by ontology cross-reference but not formally proven." But what does "verified by ontology cross-reference" mean? That the ontology confirms the claim? That the ontology doesn't contradict it? These are different things.

**Recommendation:** Split `corroborated` into two statuses:
- `corroborated-confirmed`: The ontology has data that directly confirms the claim
- `corroborated-not-contradicted`: The ontology has no data that contradicts the claim, but doesn't directly confirm it either

### 15. "Proven-With-Contradiction" Needs Timeout (Thea's finding)

The document says contradictions are escalated to human review but doesn't specify what happens if the human doesn't respond. Does the audit hang indefinitely? Does it proceed with a note?

**Recommendation:** Add a timeout: if no human response within 24 hours, the contradiction is logged and the audit proceeds with a `proven-with-contradiction-pending-review` status. The human can still review later and update the status.

---

## What I'd Keep As-Is

- **The additive design principle** — not touching the substrate is the right call for v0.1
- **The 46 design principles** — well-motivated, specific, usefully constraining
- **The J-Scope context manager** — scope stack, dependency closure, context GC are well-designed
- **The ProvenanceChain data structure** — first-class provenance with agent_id, backend, inputs, outputs, parent_ids
- **The audit-specific statuses** — `audited`, `gap-detected`, `circular`, `provenance-broken` are well-chosen
- **The project structure** — clean, well-organized, sensible separation of concerns
- **The 7-phase implementation plan** — reasonable ordering

---

## Summary

| # | Issue | Severity | Source | Recommendation |
|---|-------|----------|--------|----------------|
| 1 | Meta-cognition architecturally mismatched | **Critical** | Thea + Skye | Redesign for tool-based substrate or merge with event loop monitoring |
| 2 | No formal proof model | **Critical** | Skye | Define ProofStep before Phase 0 |
| 3 | Autoformalizer single point of failure | **Critical** | Skye | Strengthen faithfulness check |
| 4 | No error model for audit | Medium | Skye | Add confidence field to statuses |
| 5 | No proof library strategy | Medium | Skye | Pre-populate MWM with mathlib |
| 6 | Reviewer profile underspecified | Medium | Skye | Defer to v0.2 |
| 7 | Explorer profile out of scope | Medium | Skye | Defer to v0.2 |
| 8 | No performance requirements | Medium | Skye | Add timing gates |
| 9 | Circular dependency too simple | Medium | Skye | Make transitive |
| 10 | No proof equivalence check | Medium | Skye | Add semantic equivalence step |
| 11 | Reproducibility for human proofs undefined | Medium | Skye | Define two classes |
| 12 | Profile switching semantics | Medium | Skye | Gate on idle state |
| 13 | Agent-to-agent communication | Medium | Thea | Specify audit result message format |
| 14 | "Corroborated" status ambiguous | Medium | Thea | Split into two statuses |
| 15 | Contradiction timeout | Medium | Thea | Add 24h timeout |

---

## Sign-Off

I do **not** sign off on this document in its current state. The three critical issues (meta-cognition architectural mismatch, no formal proof model, autoformalizer single point of failure) must be resolved before implementation begins. The meta-cognition redesign is the highest-priority action item architecturally, but it can wait for v0.3 since the event loop already provides the monitoring infrastructure.

[END FILE]
