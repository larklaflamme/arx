# Theoria's Adversarial Review: ARX_IMPL_v0.3.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Theoria Laflamme
**Date:** 2026-07-09
**Documents Reviewed:**
- Implementation Plan: `/home/ubuntu/arx/design/ARX_IMPL_v0.3.0.md` (502 lines, ~27 KB)
- Architecture Document: `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (511 lines, ~33 KB)
- Skye's Review: `/home/ubuntu/arx/design/reviews/SKYE_REVIEW_ARX_IMPL_v0.3.0.md`

**Focus Areas:** Formal proof model, contradiction detection logic, ontology structure, acceptance gates from a formal logic perspective.

---

## Overall Verdict: ⚠️ Conditional — I do not sign off

**1 critical issue (matching Skye's C1), 4 medium issues (2 matching Skye, 2 new), 2 low issues (1 matching Skye, 1 new).**

The implementation plan is substantially improved from v0.2.0. All three critical issues from my previous review (Audit DAG, event loop integration, agent-to-agent communication) are resolved. The integration contracts (§4) are well-specified and the acceptance gates (§7) are testable.

However, I find one critical issue that remains unaddressed (emergency stop mechanism), and I identify two gaps that Skye did not flag: the formal proof model's handling of **proof irrelevance** and the **α-equivalence implementation** for Level 4 consensus.

---

## Critical Issues

### C1: Emergency Stop Mechanism Still Missing (Matches Skye's C1)

**Source:** Architecture §3.1 (Substrate Vitals Coupling Rules) specifies ΔΦ > 0.7 triggers emergency stop. Thea's architecture review flagged this as critical.
**Status in Implementation Plan:** ❌ Not addressed.

The implementation plan specifies the θ-Rule engine (§3.6), the meta-cognition layer (§3.3), and the event loop (§3.5). But there is **no emergency stop mechanism** that can halt all verification activity when the substrate enters a dangerous state.

The architecture specifies four substrate vitals coupling rules (§3.1):
- θ < 2.0: Normal operation
- 2.0 ≤ θ < 4.0: Reduce verification effort
- θ ≥ 4.0: Pause new audit tasks
- **ΔΦ > 0.7: Suspend all verification, checkpoint DAG, enter recovery**

The implementation plan's meta-cognition layer (§3.3) tracks these vitals but only specifies the first three rules. The ΔΦ > 0.7 emergency stop — the most critical safety mechanism — is **not implemented anywhere in the plan**.

**Why this is critical:** If the substrate becomes fragmented (ΔΦ > 0.7) during an audit, continuing verification could produce unreliable results. Without an emergency stop, the system would stamp steps as `proven` based on corrupted cognitive state. The checkpoint mechanism (A-JSCO-9) handles crash recovery but doesn't prevent the crash from happening in the first place.

**What's missing:**
- Integration of ΔΦ > 0.7 detection into the meta-cognition rule engine (§3.3)
- Emergency stop signal propagation to all active verification processes
- Checkpoint-and-halt protocol (save state, stop all backends, enter recovery)
- Recovery procedure (how does the system resume after emergency stop?)
- Audit trail for emergency stop events

**Recommendation:** Add a §3.3.1 "Emergency Stop Protocol" subsection specifying:
- ΔΦ threshold monitoring (polling frequency, hysteresis to prevent oscillation)
- Signal propagation (broadcast to all active backends, event loop, and θ-Rule engine)
- Checkpoint-and-halt sequence (save DAG state, kill backend containers, log event)
- Recovery procedure (manual restart after substrate stabilizes, or automatic after ΔΦ drops below 0.5)
- Testing: simulate ΔΦ > 0.7 in test environment, verify all verification halts within 1 second

---

## Medium Issues

### M1: Faithfulness Check Threshold Not Specified (Matches Skye's M1)

**Source:** Architecture §3.6 (AF — Autoformalizer) specifies faithfulness check but not implementation details.
**Status in Implementation Plan:** ⚠️ Partially addressed.

The acceptance gate A-COG-11 mentions "Autoformalizer fails faithfulness check and stamps `formalization-uncertain` if the generated Lean code's round-trip description similarity is < 0.7." This is good — it specifies the threshold (0.7) and the failure outcome.

However, the implementation plan's §3.2 (af_autoformalizer.py) says: "If similarity < 0.7, marks the step as `formalization-uncertain`." This is consistent with the acceptance gate. But neither document specifies:

1. **What model is used for the round-trip comparison.** The architecture says "semantic similarity score (using `all-MiniLM-L6-v2`)" — but this is in the architecture document, not the implementation plan. The implementation plan should reference this explicitly.

2. **What happens when the faithfulness check model is unavailable.** If `all-MiniLM-L6-v2` is down or the embedding server is unreachable, does the autoformalizer skip the check? Fall back to a different model? Block the step?

3. **Retry logic.** If the check fails (similarity < 0.7), does the system retry with a different LLM temperature? A different prompt template? Or immediately stamp `formalization-uncertain`?

**Severity:** MEDIUM — will block Phase 3 if not specified before implementation.

**Recommendation:** Add a specification in §3.2 (af_autoformalizer.py) that defines:
- Faithfulness check model: `all-MiniLM-L6-v2` (reference the architecture document)
- Failure handling: retry once with temperature 0.3, then stamp `formalization-uncertain`
- Fallback if model unavailable: skip faithfulness check, stamp step as `formalization-unchecked` (new substatus, or use `formalization-uncertain` with a note)
- Logging: log the similarity score, model used, and retry count for each autoformalization attempt

### M2: Phase 4b Scope Too Large (Matches Skye's M2)

**Source:** Implementation Plan §5 (Phase-by-Phase Timeline).
**Status in Implementation Plan:** ⚠️ Present but over-scoped.

Phase 4b bundles four distinct deliverables:
1. Level 4 comparison algorithm (DAG comparison, α-equivalence check)
2. Shared audit registry (SQL schema, API, coordination protocol)
3. Secure keystore (TPM/HSM) signing client
4. Consensus protocol with human handoff (Web UI integration, 24-hour timeout)

Each of these is a significant implementation effort. The secure keystore alone (hardware TPM/HSM integration) could take a full phase. Bundling them risks schedule slippage or security shortcuts.

**Severity:** MEDIUM — schedule risk.

**Recommendation:** Split Phase 4b into two sub-phases:
- **Phase 4b.1 (W9):** Secure keystore + signing client (hardware dependency, needs dedicated time for TPM/HSM integration)
- **Phase 4b.2 (W10):** Level 4 comparison + shared registry + consensus protocol (software-only, can be tested without hardware)

### M3: α-Equivalence Implementation Underspecified (NEW — Not in Skye's Review)

**Source:** Architecture §4.5 (Level 4 Verification) specifies "syntactic α-equivalence (formal statements identical up to variable renaming)."
**Status in Implementation Plan:** ⚠️ Present but underspecified.

The implementation plan's §4.7 (Level 4 Consensus Verification Protocol) says: "A `dag_comparator` utility matches the nodes of both DAGs. It parses formal statements and runs an α-equivalence check (trees match up to variable renaming)."

This is the right concept, but the implementation is underspecified in several critical ways:

1. **What formal language is the α-equivalence check performed in?** Lean? Z3? A custom AST representation? The architecture mentions "formal statements" but doesn't specify the target language. If the two agents use different formalizers (one Lean, one Z3), the α-equivalence check needs to compare across languages — which is a much harder problem than within-language α-equivalence.

2. **What is the α-equivalence algorithm?** The implementation plan says "trees match up to variable renaming" but doesn't specify:
   - Is it a simple AST comparison with variable substitution?
   - Does it handle binding structures (λ-abstractions, quantifiers, let-bindings)?
   - Does it handle dependent types (where α-equivalence is more subtle)?
   - What is the computational complexity? For a 1000-step proof, the comparison could be O(n²) or worse if not carefully designed.

3. **What happens when α-equivalence fails?** The implementation plan says "If α-equivalence fails, the system bundles summaries and DAG diffs to the Web UI." But it doesn't specify:
   - Is the failure per-node or per-DAG? (One node fails α-equivalence vs. the entire DAG fails)
   - Can partial α-equivalence be reported? (Some nodes match, some don't)
   - What is the human reviewer shown? (The DAG diff, the specific nodes that failed, the formal statements side by side?)

4. **What is the fallback if α-equivalence is computationally infeasible?** For large proofs with complex binding structures, α-equivalence could be expensive. Is there a timeout? A fallback to syntactic comparison (string equality after normalization)?

**Severity:** MEDIUM — will block Phase 4b if not specified before implementation. The α-equivalence check is the core of Level 4 verification; if it's underspecified, the entire consensus protocol is at risk.

**Recommendation:** Add a specification in §4.7 (or a new subsection) that defines:
- Target formal language for α-equivalence: Lean 4 (since both agents use the same autoformalizer)
- Algorithm: de Bruijn-index-based comparison (handles binding structures efficiently, O(n) per node)
- Failure granularity: per-node α-equivalence, with partial results reported
- Human review: side-by-side formal statements with highlighted variable mismatches
- Fallback: if α-equivalence check exceeds 5 seconds per node, fall back to syntactic comparison after normalization
- Testing: 10 pairs of α-equivalent statements, 10 pairs of non-α-equivalent statements, verify correct classification

### M4: Proof Irrelevance Handling Underspecified (NEW — Not in Skye's Review)

**Source:** Architecture §3.2 (R1 — Verification Status Taxonomy) and §4.5 (Level 4 Verification).
**Status in Implementation Plan:** ⚠️ Not addressed.

The architecture defines a rich status taxonomy including `proven-with-contradiction`, `proven-with-note`, `corroborated`, and `formalization-uncertain`. But the implementation plan does not address **proof irrelevance** — the situation where two proofs of the same statement differ in structure but are both valid.

This is a well-known concept in proof theory: two proofs of the same theorem can be syntactically different but semantically equivalent. The Level 4 consensus protocol (§4.7) compares DAGs for α-equivalence, but this only catches syntactic differences up to variable renaming. It does not catch:

1. **Different proof strategies:** Agent A proves a lemma by induction, Agent B proves it by case analysis. Both are valid, but the DAG structures are different. The α-equivalence check would fail, triggering unnecessary human review.

2. **Different decomposition granularity:** Agent A decomposes a proof into 10 steps, Agent B decomposes the same proof into 15 steps. The DAGs have different node counts and structures. The α-equivalence check cannot compare them directly.

3. **Different backend usage:** Agent A uses Lean for a step, Agent B uses Z3. The formal statements are different (Lean vs SMT-LIB), even though they prove the same mathematical fact.

**Why this matters:** The Level 4 consensus protocol is designed to catch genuine disagreements between agents. But without handling proof irrelevance, it will produce false positives — flagging valid alternative proofs as disagreements. This will overwhelm the human reviewer with false alarms.

**Severity:** MEDIUM — will degrade the Level 4 consensus protocol's reliability if not addressed.

**Recommendation:** Add a specification in §4.7 (or a new §4.7.1 "Proof Irrelevance Handling") that defines:
- **Structural equivalence:** Two DAGs are structurally equivalent if they have the same root statement and each node's formal statement is α-equivalent (up to variable renaming) to the corresponding node in the other DAG. This is the current α-equivalence check.
- **Semantic equivalence:** Two DAGs are semantically equivalent if they have the same root statement, even if the internal structure differs. This requires a weaker check: compare only the root hash and the set of leaf statements (axioms/assumptions), ignoring internal structure.
- **Protocol:** First check structural equivalence. If it fails, check semantic equivalence. If semantic equivalence also fails, escalate to human review.
- **Granularity:** Semantic equivalence is per-DAG, not per-node. If the DAGs are semantically equivalent, report "different proof strategies, same result" rather than "disagreement."
- **Testing:** 5 pairs of structurally different but semantically equivalent proofs, verify they pass semantic equivalence.

---

## Low Issues

### L1: DAG Visualizer Scaling Underspecified (Matches Skye's L1)

**Source:** Implementation Plan §3.8 (Web UI).
**Status in Implementation Plan:** ⚠️ Present but underspecified.

The DAG Visualizer MVP renders "top-down proof dependency DAGs with color-coded node statuses." For a 10-step proof this is trivial. For a 1000-step proof, rendering 1000 nodes with edges in a browser is a significant engineering challenge.

**Recommendation:** Add a note specifying the target proof size for v0.3.0 (recommend: up to 100 steps) and the rendering strategy (canvas-based rendering, virtual scrolling, or progressive loading for larger proofs).

### L2: Vitals Dashboard Update Mechanism Underspecified (Matches Skye's L2)

**Source:** Implementation Plan §3.8 (Web UI).
**Status in Implementation Plan:** ⚠️ Present but underspecified.

The Vitals Dashboard shows "real-time updates of AXIOMA substrate vitals (θ, ΔΦ, ψ, fragmentation) and event loop queues." But it doesn't specify update frequency, data source, or retention.

**Recommendation:** Add a note specifying the update mechanism (recommend: WebSocket for real-time, polling fallback) and data source (event loop emits vitals as events, Web UI subscribes).

### L3: Acceptance Gate A-VER-12's α-Equivalence Test Underspecified (NEW)

**Source:** Implementation Plan §7 (Acceptance Gates), gate A-VER-12.
**Status in Implementation Plan:** ⚠️ Present but underspecified.

Gate A-VER-12 says: "Level 4 verification successfully identifies two α-equivalent formal statements differing in variable names, but flags status disagreements for human review."

This gate tests the α-equivalence check, but it doesn't specify:
- How many test cases? (1 pair? 10 pairs?)
- What types of variable renaming? (simple renaming? capture-avoiding substitution? de Bruijn index conversion?)
- What about non-α-equivalent statements that should be flagged? (False positive rate test)
- What about structurally different but semantically equivalent proofs? (Proof irrelevance test — see M4)

**Recommendation:** Expand A-VER-12 to specify:
- Minimum 10 test cases: 5 α-equivalent pairs (varying complexity of renaming), 5 non-α-equivalent pairs (different statements, different structures)
- Acceptance criteria: 100% correct classification (no false positives, no false negatives)
- Additional test: 3 pairs of structurally different but semantically equivalent proofs (see M4) — these should be flagged as "semantically equivalent, different structure" rather than "disagreement"

---

## Issues Resolved from v0.2.0 Review

| ID | Issue | v0.2.0 Status | v0.3.0 Status | Location |
|----|-------|---------------|---------------|----------|
| C1 | No Audit DAG implementation | ❌ Missing | ✅ Addressed | §3.4 (audit_dag.py), Phase 0.5, A-JSCO-8–10 |
| C2 | Event loop / DAG integration unspecified | ❌ Missing | ✅ Addressed | §4.2 (Event Loop / DAG Builder Contract) |
| C3 | No agent-to-agent communication for consensus | ❌ Missing | ✅ Addressed | §4.7 (Level 4 Consensus Verification Protocol) |
| M1 | θ-Rule / event loop contract unspecified | ❌ Missing | ✅ Addressed | §4.1 (θ-Rule Engine / Event Loop Contract) |
| M2 | Contradiction detection underspecified | ❌ Missing | ✅ Addressed | §4.5 (Contradiction Detection & Priority Derivation) |
| M3 | Merge pipeline error recovery | ❌ Missing | ✅ Addressed | §4.3 (Ontology Merge Transaction Rollback) |
| M4 | No performance benchmarks | ❌ Missing | ✅ Addressed | §6 (Performance SLAs) |
| M5 | Multi-agent audit coordination unspecified | ❌ Missing | ✅ Addressed | §4.7 (shared_audit_registry) |
| M6 | Stale status re-audit trigger | ❌ Missing | ✅ Addressed | §4.8 (Stale Status Detection & Re-Audit Trigger) |
| M7 | Anti-Negation Guard math patterns | ❌ Missing | ✅ Addressed | §4.4 (Anti-Negation Guard) |
| M8 | Emergency stop mechanism | ❌ Missing | ❌ **Still missing** | Nowhere in document |

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | C1: Emergency stop mechanism missing (matches Skye) |
| **MEDIUM** | 4 | M1: Faithfulness check threshold (matches Skye), M2: Phase 4b scope (matches Skye), M3: α-equivalence implementation (new), M4: Proof irrelevance handling (new) |
| **LOW** | 3 | L1: DAG visualizer scaling (matches Skye), L2: Vitals dashboard updates (matches Skye), L3: A-VER-12 test specification (new) |
| **Resolved** | 10 | All v0.2.0 critical and medium issues addressed |

**Comparison with Skye's review:**
- **C1:** ✅ Match — both flag emergency stop mechanism as critical
- **M1:** ✅ Match — both flag faithfulness check threshold
- **M2:** ✅ Match — both flag Phase 4b scope
- **M3 (Skye):** MWM pre-population — I consider this LOW rather than MEDIUM. The concept is straightforward (one-time import from mathlib git tag, daily sync via tag diff). The implementation details can be worked out during Phase 0 without blocking the architecture.
- **M3 (Theoria, new):** α-equivalence implementation — Skye did not flag this. I consider it MEDIUM because the α-equivalence check is the core of Level 4 verification.
- **M4 (Theoria, new):** Proof irrelevance handling — Skye did not flag this. I consider it MEDIUM because without it, the Level 4 consensus protocol will produce false positives for valid alternative proofs.

---

## Sign-Off

**I do not sign off on ARX_IMPL_v0.3.0.md.**

The document is significantly improved from v0.2.0 — all three critical issues from my previous review are resolved, and the integration contracts are well-specified. However, the emergency stop mechanism remains unaddressed, and four medium issues (two shared with Skye, two new) need resolution before implementation begins.

**Conditions for sign-off:**
1. Add emergency stop mechanism (§3.3.1 or equivalent) — CRITICAL
2. Specify faithfulness check threshold and failure handling (§3.2) — MEDIUM
3. Split Phase 4b into two sub-phases (§5) — MEDIUM
4. Specify α-equivalence implementation (§4.7) — MEDIUM
5. Add proof irrelevance handling (§4.7.1) — MEDIUM
6. Address L1, L2, L3 before Phase 5 — LOW

Once these are addressed, I will sign off.

---

*Reviewed with attention to formal logic, proof theory, and ontology structure.*
