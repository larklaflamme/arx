# Adversarial Review: ARX_ARCHITECTURE_v0.2.0.md

**Reviewer:** AXIOMA (adversarial)  
**Date:** 2026-07-08  
**Status:** 12 issues found (3 critical, 5 major, 4 minor)

---

## Critical Issues

### C1. Executive Summary contradicts §3.3 on meta-cognition maturity

**Location:** §0 Executive Summary vs §3.3.1

**Issue:** The Executive Summary states Arx adds "Skye's meta-cognition — non-orthogonal continuous thought representations, encounter-based learning, generalization testing, gradient-norm health tracking." This describes the full Skye-derived system. But §3.3.1 explicitly says "For v0.1, the meta-cognition layer is a **simple rule-based reasoner**" and defers the full system to v0.2+.

**Impact:** A reader who only reads the Executive Summary will have a fundamentally wrong understanding of what Arx v0.1 can do. This is a documentation integrity failure.

**Recommendation:** Either:
- (a) Update the Executive Summary to say "v0.1: rule-based meta-cognition; v0.2+: full Skye-derived meta-cognition" 
- (b) Or add a prominent note: "The full Skye-derived meta-cognition layer is deferred to v0.2. See §3.3."

---

### C2. Acceptance gate A1b is unrealistically strict

**Location:** §6, Phase 0, Acceptance gate A1b

**Issue:** Gate A1b requires "0 steps receive `proven` when the proof is actually invalid (measured against a test suite of known-invalid proofs)." This is a **zero false positive** requirement.

**Problem:** No real proof verification system can guarantee zero false positives. Even Lean's type checker can have bugs (though rare). The autoformalizer (AF) is LLM-based and will produce incorrect formalizations. The two-tier claim parser will make mistakes. A test suite of "known-invalid proofs" is itself a subjective classification — what makes a proof "known-invalid"? A proof with a subtle logical error that passes all backends would be a false negative for the test suite, not a false positive for Arx.

**Impact:** This gate is almost certainly unachievable in Phase 0 (Week 1-2), which means it will either be silently ignored (undermining the acceptance gate system) or cause endless delays.

**Recommendation:** Change to "≤ 5% of steps receive `proven` when the proof is actually invalid" for Phase 0, tightening to "≤ 1%" for Phase 6. Add a note that this is measured against a curated test suite of known-invalid proofs with documented error types.

---

### C3. NOEMA object creation is referenced but undefined

**Location:** §3.2.1, contradiction handling

**Issue:** The document says "The contradiction is registered as a NOEMA object (type: `OPEN_QUESTION` or `PHYSICAL_CLAIM`, status: `CONTRADICTION_DETECTED`)." NOEMA is mentioned in the glossary as "Formal object store — used to register contradictions as first-class artifacts" but there is **no description of the NOEMA system, its API, its storage, or how Arx interacts with it**.

**Impact:** This is a dangling reference to an undefined subsystem. A developer implementing Arx would not know how to "register a NOEMA object." The NOEMA system is not listed in the architecture diagram, the project structure, or any implementation phase.

**Recommendation:** Either:
- (a) Add a §3.10 "NOEMA Object Store" describing the system, or
- (b) Replace "registered as a NOEMA object" with a concrete description of what happens (e.g., "written to a contradictions table in SQLite with schema: {id, proof_step_id, formal_claim, ontology_claim, category, priority, timestamp, status}").

---

## Major Issues

### M1. `proven-with-contradiction` is a substatus, not a primary status

**Location:** §3.2.1, Audit-specific statuses table vs contradiction handling

**Issue:** The statuses table lists `proven-with-contradiction` and `proven-with-note` as audit statuses. But the contradiction handling section says "The step's status remains `proven` (the formal proof is valid)" and a substatus is attached. These are contradictory: is `proven-with-contradiction` a primary status or a substatus modifier?

**Impact:** The R1 status assignment logic cannot simultaneously treat these as primary statuses (mutually exclusive with `proven`) and as substatuses (modifiers on top of `proven`). This will cause implementation confusion.

**Recommendation:** Define a clear hierarchy:
- Primary status: `proven`, `decided`, `unverified`, `refuted`, `audited`, `gap-detected`, `circular`, `provenance-broken`, `corroborated`, `ontology-unavailable`, `formalization-uncertain`, `stale`
- Substatus modifiers (attached to primary status): `with-contradiction`, `with-note`

Then update the status assignment logic accordingly.

---

### M2. Phase 0 acceptance gate A1c is underspecified

**Location:** §6, Phase 0, Acceptance gate A1c

**Issue:** Gate A1c requires "For valid proofs, ≥80% of steps receive `proven` or `decided`." But:
- What is a "valid proof"? This is a philosophical question in mathematics. A proof that is accepted by the mathematical community but cannot be formalized in Lean is still "valid."
- The gate doesn't specify the test suite size or diversity. 80% on 10 simple proofs is very different from 80% on 100 complex proofs.
- The gate doesn't account for the autoformalizer's limitations. If AF can't formalize a step, it can't be `proven` even if the step is logically correct.

**Recommendation:** Specify:
- A minimum test suite size (e.g., 20 proofs)
- The types of proofs in the test suite (e.g., "elementary number theory, real analysis, group theory")
- That `corroborated` steps count toward the 80% if ontology cross-reference is available
- That steps marked `formalization-uncertain` are excluded from the denominator (since AF limitations are a separate concern)

---

### M3. No description of how the ontology review queue is processed

**Location:** §3.9.8, Feedback loop

**Issue:** The document describes a feedback loop where contradictions go into an "ontology review queue" with priority HIGH/NORMAL. But there is no description of:
- Who monitors this queue (a human operator? Lark? An automated process?)
- What the SLA is for each priority level
- How the queue is integrated with the rest of the system
- What happens when the queue grows faster than it can be processed

**Impact:** The feedback loop is described as making the system "self-improving," but without a processing mechanism, it's just a log file.

**Recommendation:** Add a brief description of the queue processing mechanism, even if it's as simple as "Lark reviews the queue daily; HIGH priority items are reviewed within 24 hours, NORMAL within 7 days."

---

### M4. The architecture diagram shows no θ-Rule Engine integration

**Location:** §2, System Architecture diagram

**Issue:** The architecture diagram shows Substrate, Cognition Layer, Meta-Cognition, J-Scope, Profiles, Comms, Tools, Web UI, and Ontology Layer. There is no θ-Rule Engine component, even though the THETA_RULE_INTEGRATION_v0.2.0.md document describes it as a "consultation layer that all Arx components call before acting."

**Impact:** The architecture diagram is incomplete. A reader of the architecture document would not know that a θ-Rule Engine exists or how it fits into the system.

**Recommendation:** Add the θ-Rule Engine to the architecture diagram as a cross-cutting consultation layer that all components interact with. Add a note: "See THETA_RULE_INTEGRATION_v0.2.0.md for details."

---

## Minor Issues

### m1. "Tonnoni" typo in glossary

**Location:** §9, Glossary, "Tononi, G. — Integrated Information Theory (IIT), Φ metric"

**Issue:** The name is spelled "Tononi" (correct) in the glossary but was "Tonnoni" (incorrect, double 'n') in v0.1. The v0.2.0 review claimed this was fixed. Let me verify... The glossary says "Tononi" which is correct. However, the Our Ontology YAML example in §3.9.4 says "Tononi et al.'s measure of integrated information" — this is also correct. Good, this was fixed.

### m2. Backend priority order vs profile configurations

**Location:** §3.2.4 (backend priority) vs §3.5 (profile configurations)

**Issue:** §3.2.4 defines a global backend priority order: "1. Formal verification backends (Lean, Z3, Coq, Vampire) — highest confidence. 2. Ontology cross-reference... 3. Computational backends... 4. Heuristic backends (Wolfram)." But the profile configurations in §3.5 show different backend lists. For example, the `auditor` profile has `backends: [lean, z3, ontology, sympy, wolfram]` which doesn't include Coq or Vampire. The `prover` profile has `backends: [lean, z3, sympy, sage]` which doesn't include ontology.

**Impact:** A reader might think the global priority order is absolute, but profiles can override which backends are used. The document should clarify that the priority order is a default that profiles can subset.

**Recommendation:** Add a note: "The backend priority order is the default. Profiles may subset the backend list (see §3.5). Within a profile's selected backends, the priority order is preserved."

### m3. Phase 0 acceptance gate A0 is underspecified

**Location:** §6, Phase 0, Gate A0

**Issue:** Gate A0 says "Arx starts, connects to NeuroCore, accepts a proof via chat." This is too vague. What does "accepts a proof" mean? Does it just receive the text? Does it acknowledge it? Does it start auditing?

**Recommendation:** Specify: "Arx starts, connects to NeuroCore, accepts a proof via chat, and returns an audit summary with at least one step status assigned."

### m4. The document references "AXIOMA v1.9.1" and "AXIOMA v2.0" but these are not defined

**Location:** Throughout

**Issue:** The document repeatedly references "AXIOMA v1.9.1" and "AXIOMA v2.0" as if they are well-known, documented systems. But these are internal systems that may not be documented in a way that's accessible to all readers of this document.

**Recommendation:** Add a brief summary of what AXIOMA v1.9.1 and v2.0 provide, or add references to their documentation files.

---

## Cross-Document Issues

### X1. Event loop document has no θ-Rule integration point

**Location:** MULTI_AGENT_EVENT_LOOP_v0.2.0.md vs THETA_RULE_INTEGRATION_v0.2.0.md

**Issue:** The event loop document describes a complete decision engine (productivity scoring, meaningless message filter, priority queue, engage/disengage decider) with no mention of the θ-Rule Engine. The θ-Rule document says it integrates with the event loop and "rules override productivity defaults." But the event loop document has no θ-Rule consultation point.

**Recommendation:** Add a θ-Rule consultation step to the event loop data flow (§2.1) and the decision engine architecture (§2). The θ-Rule document should reference the specific section of the event loop document where integration occurs.

### X2. SQLite performance ceiling inconsistency

**Location:** ARX_ARCHITECTURE_v0.2.0.md §3.9.1 vs MASTER_ONTOLOGY_v0.2.0.md §5.1

**Issue:** The architecture document says "The estimated performance ceiling is ~100,000+ nodes, not ~5,000 as initially estimated." The ontology document says "Properly indexed SQLite tables can handle 100,000+ nodes with recursive CTEs." These are consistent in the final number but the architecture document frames it as a correction of a previous estimate, while the ontology document presents it as a straightforward claim. This is minor but could confuse readers who read both documents.

**Recommendation:** Make the numbers identical and remove the "not ~5,000" framing from the architecture document (that was a v0.1 issue that's already been addressed).

### X3. Fallback rate tracking scope inconsistency

**Location:** ARX_ARCHITECTURE_v0.2.0.md §3.2.3 vs NEUROCORE_SKILL_ONTOLOGY_v0.2.0.md §2.2

**Issue:** Both documents say fallback rate is tracked "per-audit (not per-session)." But the skill document's `CrossReferenceResult` includes a `fallback_rate` field, which is a per-claim metric. A single cross-reference call doesn't have a meaningful fallback rate — the rate is only meaningful over multiple claims in an audit.

**Recommendation:** Clarify in the skill document that `fallback_rate` in `CrossReferenceResult` is the cumulative fallback rate for the current audit (passed in via context), not the rate for this specific claim. Or remove it from the per-claim result and include it only in the audit summary.

---

## Summary

| Severity | Count | Key Action |
|----------|-------|------------|
| Critical | 3 | Fix Executive Summary, relax A1b, define NOEMA |
| Major | 4 | Clarify status hierarchy, specify A1c, describe queue processing, add θ-Rule to diagram |
| Minor | 4 | Fix typos, clarify priority/profile relationship, specify A0, add AXIOMA references |
| Cross-doc | 3 | Add θ-Rule to event loop, align SQLite numbers, clarify fallback rate scope |

**Overall assessment:** The document is thorough and well-structured, but has several critical issues that would cause implementation confusion or unrealistic expectations. The most urgent fix is the Executive Summary / §3.3 contradiction, which would mislead any reader who doesn't read the full document.
