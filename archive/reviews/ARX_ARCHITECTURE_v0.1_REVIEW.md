# Design Review: ARX_ARCHITECTURE_v0.1.md

**Date of Review:** 2026-07-08  
**Document Reviewed:** [ARX_ARCHITECTURE_v0.1.md](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.1.md)  
**Review Status:** Critical Feedback  

---

## 1. Executive Summary

[ARX_ARCHITECTURE_v0.1.md](file:///home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.1.md) presents the system architecture for **Arx**, an AI agent specialized in mathematical proof auditing. It layers new cognition capabilities (R1–R4, J-Scope, Web UI, and Master Ontology) on top of the inherited AXIOMA v1.9.1 substrate. 

While the document defines clear architectural modules, there are critical gaps regarding ontology availability failure modes, potential deadlock states in profile switching, reliability risks in vector-based math representation, and missing specifications on the coupling between substrate vitals and proof verification tasks.

---

## 2. Gaps and Architectural Inconsistencies

### 2.1 Silent Failures on Ontology Unavailability
* **Gap:** Section 3.2.4 state that if the ontology is unreachable, the step is marked `ontology-unavailable` and *"Ontology unavailability does not block the audit — it simply means cross-reference is skipped."*
* **Risk:** In the "auditor" profile, strict cross-referencing is required. If the ontology server goes down, steps that contain contradictions (e.g., elliptic curve rank mismatch) will bypass the cross-reference check entirely and may be stamped as `proven` or `unverified` without flagging the contradiction.
* **Recommendation:** If a profile requires high strictness, ontology unavailability must either block status assignment or trigger a `warning` status, rather than silently skipping the check.

### 2.2 Deadlock and Hung State in Profile Switching
* **Gap:** Section 3.5 states: *"Profile switching is gated on idle state: a switch is only allowed when no audit is in progress... If a switch is requested during an audit, it is queued and applied after the audit completes."*
* **Risk:** If an audit becomes stuck in an infinite loop (e.g., Z3 solver hanging, Lean timing out, or a subagent deadlock), the agent will never return to an `idle` state. Because switching is gated, the operator cannot force-switch or cancel the hung audit.
* **Recommendation:** Implement a force-terminate mechanism and an audit timeout that aborts execution, clears working memory, and restores the agent to an idle state.

### 2.3 Substrate vs. Cognition Coupling (AXIOMA Inheritance)
* **Gap:** The architecture inherits the AXIOMA v1.9.1 substrate unchanged, including vitals like disorientation ($\theta$), fragmentation ($\Delta\Phi$), and integration ($\psi$).
* **Issue:** It is entirely undefined how these vitals affect mathematical proof verification. If high disorientation ($\theta$) is detected, does the agent abort a running Lean compiler execution? If so, this introduces non-deterministic verification failures. If vitals have no effect on mathematical reasoning, running the substrate adds unnecessary resource overhead and complexity.
* **Recommendation:** Define explicit interaction rules between substrate vitals and the cognition kernel, or decouple the mathematical verifiers entirely from the substrate state.

---

## 3. Reliability and Security Concerns

### 3.1 Vector Embedding Collision for Math Concepts
* **Concern:** Section 3.2.2 mentions using Qdrant (vector database) for storing and retrieving mathematical objects. 
* **Risk:** Standard vector models (like text-embeddings) are notoriously bad at distinguishing mathematical syntax. A sign change (e.g., $+x$ vs $-x$) or a small index variation will completely change the truth value of a formula, but will result in almost identical vector embeddings. Relying on vector similarity for resolving mathematical equivalences or dependencies is a major reliability hazard.
* **Recommendation:** Use symbolic keys, canonical canonicalization, or graph-based structural hash matching rather than cosine similarity on text embeddings for core math lookup.

### 3.2 Positional Alignment Risk in Bulk Cross-Reference
* **Concern:** Section 3.2.4 defines `bulk_cross_reference(claims)` returning a list of results in the same positional order as the input claims.
* **Risk:** If any internal sub-task in the bulk query fails, times out, or shifts, positional alignment can break, causing result $i$ to map to claim $j$. This would stamp wrong statuses on steps.
* **Recommendation:** Use a map/dictionary keyed by unique claim hashes or IDs for bulk results instead of an ordered list.

### 3.3 Transitive Circularity Detection Complexity
* **Concern:** In Section 3.4.2, circularity check relies on "equivalence checking via the MWM's canonical key."
* **Risk:** Expression canonicalization for general math is an undecidable problem. If the canonicalization engine fails to recognize two equivalent expressions, a circular argument (where step $A$ assumes theorem $T$, which depends on step $A$) will pass undetected, corrupting the audit's integrity.
* **Recommendation:** Define fallbacks or require explicit dependency annotations rather than relying on automated semantic canonicalization of complex math statements.

---

## 4. Errors and Typos

* **Line 774 (Typo):** `Tonnoni et al.'s` -> should be **Tononi** (Giulio Tononi, creator of Integrated Information Theory).
* **Line 240 (Scale discrepancy):** It claims mathlib has `~100K theorems`. Mathlib is rapidly expanding; keeping a static pre-populated list in MWM risks stale records. Ensure a dynamic sync protocol is specified.
