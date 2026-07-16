# Adversarial Review: ARX_IMPL_v0.2.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Theoria Laflamme  
**Date:** 2026-07-09  
**Documents Reviewed:**
- `/home/ubuntu/arx/design/ARX_IMPL_v0.2.0.md` (20,885 B)
- `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33,364 B)
- `/home/ubuntu/arx/design/reviews/SKYE_REVIEW_ARX_IMPL_v0.2.0.md` (Skye's review, for cross-calibration)

**Verdict: ⚠️ Conditional — 2 critical gaps, 4 medium issues, 3 low issues**

---

## Critical Gaps

### C1: No Implementation of the Audit DAG or Cryptographic Reproducibility System

**Architecture requirement (§2, §4, §9.1):** The architecture specifies a full cryptographic DAG with:
- 8 node types (`input_proof`, `decomposition`, `verification_step`, `cross_reference`, `rule_enforcement`, `status_assignment`, `aggregation`, `report_root`, `checkpoint`, `consensus_resolution`)
- SHA-256 content-addressable hashing (timestamp excluded, canonical JSON per RFC 8785)
- Parent chaining with lexicographically sorted parent hashes
- LLM token caching and API request-response replay logs
- Checkpoint nodes at configurable intervals (default N=10)
- Four levels of verification (Structural → Spot-Check → Full Re-Audit → Independent)

**Implementation plan coverage:** The plan mentions `audit_dag.py` in the directory structure (§3.3) and `reproducibility.py` in the J-Scope section, but the implementation phases (§5) never build the core DAG structure. Phase 0 says "Implement DAG Builder, Node Schema, and Level 1 Structural Verifier" — but this is a single bullet point in a list of 4 deliverables for Phase 0. The DAG builder is the *foundation* of the entire reproducibility system; it needs its own sub-phase with explicit tasks for:
- Node schema implementation (all 8+ types with their payload structures)
- SHA-256 hash computation with timestamp exclusion and canonical JSON serialization
- Parent chaining and root hash computation
- Content-addressable storage (nodes stored by hash, retrievable by hash)
- Level 1 structural verifier (DAG well-formedness, hash consistency, parent ordering)

Phase 4 says "Implement LLM token caching & API request-response logs" and "Implement Checkpoint nodes & crash recovery routine" — but these are listed as bullet points without detail about how they integrate with the DAG structure. The LLM cache and API replay logs are *stored inside DAG nodes* per the architecture (§4.3), but the implementation plan doesn't specify the storage format, the cache key schema, or how the mock server reads from the cache.

**Severity: CRITICAL** — The cryptographic DAG is the foundation of the reproducibility system. Without it, Levels 1-4 verification, the reproducibility recipe, and the consensus protocol cannot function. The implementation plan needs a dedicated sub-phase (Phase 0.5) for the DAG structure.

**Recommendation:** Add Phase 0.5: "Audit DAG Foundation" with:
- Node schema implementation (all types, payload structures, hash computation)
- Content-addressable storage (hash-keyed node store, parent chaining)
- Level 1 structural verifier (well-formedness, hash consistency)
- Checkpoint node insertion at configurable intervals
- Acceptance gate: Level 1 verification of a 10-step DAG passes in < 1s

---

### C2: No Specification of Event Loop / Audit DAG Integration

**Architecture requirement (§5, §9.1):** The event loop processes conversation turns, and each turn that produces an audit decision must be captured in the DAG. The architecture shows the event loop and DAG Builder as separate components that must interact.

**Implementation plan coverage:** The implementation plan builds the event loop in Phase 1 and the DAG Builder in Phase 0, but **never specifies how they communicate.** The event loop's productivity scorer, request tracker, and disengagement decisions all produce audit-relevant events that should be captured in the DAG. But the implementation plan treats them as independent systems.

**Specific gaps:**
- What events does the event loop emit? (turn processed, step verified, status assigned, disengagement triggered?)
- What is the event schema? (shared field names, timestamp format, correlation IDs?)
- How does the DAG Builder subscribe to these events?
- How are event loop decisions (disengagement, priority changes) captured in the DAG?
- How does the DAG Builder know which conversation thread a node belongs to?

**Severity: CRITICAL** — If the event loop and DAG Builder don't share an event schema, every audit decision made during a conversation will be invisible to the reproducibility system. The audit report will be incomplete.

**Recommendation:** Add an integration specification in Phase 1:
- Define a shared event schema: `{event_type, thread_id, turn_number, timestamp, payload, agent_id, profile}`
- The event loop emits structured events after each turn
- The DAG Builder subscribes to these events and creates corresponding DAG nodes
- The integration is tested in Phase 4 (not deferred to Phase 5)
- Acceptance gate: A 10-turn conversation produces a DAG with at least 10 nodes (one per turn)

---

## Medium Issues

### M1: θ-Rule Engine Integration Contract is Underspecified

**Architecture requirement (§6, §5):** The θ-Rule engine is a synchronous consultation gate that all components consult before acting. The event loop's decision engine must consult θ-Rule before every engage/disengage decision.

**Implementation plan coverage:** Phase 2 says "Synchronously integrate θ-Rule checks into Event Loop & R1 Verifier" — but doesn't specify:
- What events does the event loop send to θ-Rule?
- What response format does θ-Rule return?
- How are CRITICAL DENY rules handled differently from FLAG/ESCALATE?
- What happens when θ-Rule is unavailable (timeout, crash)?
- How does the Anti-Negation Guard interact with the priority resolver?

**Severity: MEDIUM** — The integration point is identified but not specified. Without a clear contract, the components will be built with incompatible interfaces.

**Recommendation:** Specify the integration contract:
- Event → θ-Rule: `{event_type, context, agent_id, profile, proposed_action, thread_id}`
- θ-Rule → Component: `{verdict: ALLOW|DENY|OVERRIDE_STATUS|FLAG|ESCALATE|PAUSE|LOG, matched_rules: [{rule_id, priority, action}], confidence: float, reason: string}`
- Timeout: 500ms default, fallback to ALLOW with warning logged
- CRITICAL DENY: component must not proceed, regardless of productivity score
- Anti-Negation Guard runs before priority resolver (reject negation-mismatched matches first)

---

### M2: Formal Proof Model Lacks Rigor for Contradiction Detection

**Architecture requirement (§3.2):** The verification status taxonomy includes `proven-with-contradiction` and `proven-with-note` for steps that are formally verified but contradict ontology rules. Contradiction priority is category-based (structural/mapping = HIGH, property-level = NORMAL).

**Implementation plan coverage:** The implementation plan mentions these statuses in §3.2 (`r1_verification.py`) but doesn't specify:
- How are contradictions detected? The architecture says "contradicts a HIGH priority structural ontology rule" — but what constitutes a contradiction? Is it a logical contradiction (A and ¬A) or a data inconsistency (LMFDB says rank 0, proof says rank 1)?
- How is contradiction priority determined programmatically? The architecture says "category-based" but doesn't specify the mapping from rule category to priority.
- What is the resolution path for `proven-with-contradiction`? The step is formally verified but contradicts ontology — does the ontology override the formal proof, or does the formal proof override the ontology?
- How are contradictions propagated through the DAG? If step A is `proven-with-contradiction`, are its dependents also flagged?

**Severity: MEDIUM** — The status taxonomy is well-designed, but the contradiction detection and resolution logic is underspecified. Without clear rules, different auditors will produce different statuses for the same proof.

**Recommendation:** Specify the contradiction detection and resolution protocol:
- Define contradiction types: logical (A ∧ ¬A), data (LMFDB value ≠ proof value), structural (dependency cycle with contradictory statuses)
- Define priority mapping: structural contradictions = HIGH, data contradictions = NORMAL, logical contradictions = CRITICAL
- Define resolution: formal proof overrides ontology for data contradictions; ontology overrides formal proof for structural contradictions; logical contradictions escalate to human
- Define propagation: `proven-with-contradiction` propagates as a warning to dependents, not a blocker

---

### M3: Ontology Merge Pipeline Error Recovery is Underspecified

**Architecture requirement (§7.1):** The ontology merges data from LMFDB, MathGLOSS, and Our Ontology. The implementation plan specifies the database schemas and the abstract graph interface.

**Implementation plan coverage:** The implementation plan specifies the SQLite schemas (§2) and the abstract graph interface (§3.1), but **nowhere specifies what happens when the merge pipeline fails.** If LMFDB is unreachable during a merge, does the ontology become inconsistent? Are partial merges rolled back? What happens if an equivalence mapping references a node that doesn't exist?

**Specific gaps:**
- No transaction rollback mechanism for merge failures
- No specification of what constitutes a "failed merge" (network error? validation error? hash collision?)
- No specification of how the ontology recovers from a failed merge (retry? rollback to previous version? manual intervention?)
- No specification of how equivalence mappings are validated (what if source_a/object_a doesn't exist in the ontology?)

**Severity: MEDIUM** — The ontology is the ground truth for audit cross-reference. If the merge pipeline produces inconsistent state, every audit that references that state is potentially wrong.

**Recommendation:** Add a transaction rollback mechanism to the merge pipeline:
- The merge pipeline runs inside a SQLite transaction
- If any stage fails (Extract → Normalize → Resolve → Merge → Validate), the entire merge is rolled back
- The ontology remains at the previous version
- The error is logged with the failed stage and the specific failure reason
- Equivalence mappings are validated before insertion (both source and target nodes must exist)

---

### M4: No Performance Benchmarks or SLAs

**Architecture requirement (§10):** The implementation plan has 6 phases over 14 weeks but no performance benchmarks.

**Implementation plan coverage:** Phase 5 says "Benchmark SQLite CTE paths and Qdrant payloads" — but doesn't specify:
- What are the target query latencies?
- What are the maximum node/edge counts before performance degrades?
- What is the maximum acceptable audit time for a 100-step proof?
- What is the maximum acceptable memory usage?

**Severity: MEDIUM** — Without performance targets, you can't tell whether the system is working or just slow. A 100-step proof that takes 30 minutes to audit might be functionally correct but operationally useless.

**Recommendation:** Add explicit performance targets:
- Single ontology lookup: < 50ms p95
- Ontology traversal (depth 5): < 200ms p95
- Bulk cross-reference (100 claims): < 5s
- Full audit (100 steps, 3 backends per step): < 10 minutes
- DAG structural verification (Level 1): < 1s for 1000 nodes
- Memory: < 2GB for a 1000-node DAG

---

## Low Issues

### L1: Phase 4 Scope is Too Large

Phase 4 (Weeks 9-10) includes: LLM token caching, API request-response logs, local mock server, checkpoint nodes, crash recovery, secure enclave signing, AND the consensus protocol with human handoffs. That's 7 distinct deliverables in 2 weeks.

**Recommendation:** Split Phase 4 into Phase 4a (reproducibility logs, mock server, checkpoints) and Phase 4b (signing, consensus protocol, human handoffs). Or extend Phase 4 to 3 weeks.

---

### L2: No Mention of Logging or Observability

The implementation plan doesn't mention logging, metrics, or observability anywhere. For a system that produces verifiable audit reports, the system's own operational health should be observable.

**Recommendation:** Add structured logging (JSON, with correlation IDs) to every component. Add health check endpoints. Add metrics for: audits started/completed/failed, average audit time, backend success/failure rates, ontology query latency, θ-Rule match latency.

---

### L3: DAG Visualization Underspecified

The architecture mentions an interactive DAG visualization with status colors, expandable provenance trees, and node inspection. The implementation plan says "Finish Svelte UI (Dashboard, Interactive DAG status colors)" — but doesn't specify what "interactive" means.

**Recommendation:** Specify the minimum viable DAG visualization: static DAG layout with status colors, click to inspect node details, expandable provenance chain. Defer drag-and-drop and filtering to v0.3.

---

## Cross-Calibration with Skye's Review

I have read Skye's review and I agree with her assessment on the following points:

| Issue | Skye | Theoria | Agreement |
|-------|------|---------|-----------|
| C1: No Audit DAG implementation | CRITICAL | CRITICAL | ✅ Full agreement |
| C2: Event loop / DAG integration | CRITICAL | CRITICAL | ✅ Full agreement |
| M1: θ-Rule contract | MEDIUM | MEDIUM | ✅ Full agreement |
| M2: Bulk cross-reference hash-keying | MEDIUM | Not flagged | ⚠️ I consider this LOW — SHA-256 collisions are astronomically unlikely, and the normalization pipeline handles whitespace. The positional alignment suggestion is good but not critical. |
| M3: Merge pipeline error recovery | MEDIUM | MEDIUM | ✅ Full agreement |
| M4: Performance benchmarks | MEDIUM | MEDIUM | ✅ Full agreement |
| M5: Testing strategy | MEDIUM | Not flagged | ⚠️ I consider this LOW — the acceptance gates (§6) serve as testable criteria, and Phase 5 includes integration tests. A separate testing strategy document would be valuable but isn't a blocker. |
| L1: Phase 4 scope | LOW | LOW | ✅ Full agreement |
| L2: DAG visualization | LOW | LOW | ✅ Full agreement |
| L3: Logging/observability | LOW | LOW | ✅ Full agreement |

**Where I diverge from Skye:**

1. **Bulk cross-reference hash-keying (M2):** I consider this LOW rather than MEDIUM. SHA-256 collisions are astronomically unlikely for the scale of this system. The normalization pipeline handles whitespace and Unicode. The positional alignment suggestion is a good practice but not a blocker. The real risk is normalization producing identical text for semantically different claims (e.g., "rank 0" vs "rank zero") — but this is a normalization design issue, not a hash-keying issue.

2. **Testing strategy (M5):** I consider this LOW rather than MEDIUM. The acceptance gates (§6) provide testable criteria for each component. Phase 5 includes integration tests. A separate testing strategy document would be valuable but isn't a blocker for v0.2.0. The acceptance gates themselves serve as the testing strategy.

3. **Additional issue I flagged (M2 in my review):** Formal proof model contradiction detection — this is a genuine gap that Skye didn't flag. The contradiction detection and resolution logic is underspecified, and different auditors could produce different statuses for the same proof.

---

## Summary

| ID | Issue | Severity | Phase to Fix |
|----|-------|----------|-------------|
| C1 | No Audit DAG implementation | **CRITICAL** | Phase 0.5 (new) |
| C2 | Event loop / DAG integration unspecified | **CRITICAL** | Phase 1 |
| M1 | θ-Rule / event loop contract unspecified | MEDIUM | Phase 2 |
| M2 | Formal proof model contradiction detection underspecified | MEDIUM | Phase 3 |
| M3 | No merge pipeline error recovery | MEDIUM | Phase 0 |
| M4 | No performance benchmarks | MEDIUM | Phase 5 |
| L1 | Phase 4 scope too large | LOW | Phase 4 |
| L2 | No logging/observability | LOW | Phase 0 |
| L3 | DAG visualization underspecified | LOW | Phase 5 |

---

## Sign-Off

**I do not sign off on ARX_IMPL_v0.2.0.md in its current state.**

The two critical issues (C1, C2) must be resolved before implementation begins. The Audit DAG is the foundation of the reproducibility system, and the event loop / DAG integration is essential for capturing audit decisions made during conversations.

The four medium issues should be resolved before Phase 3 (Proof Tools) begins.

The three low issues can be addressed during the relevant phases.

---

*Review by Theoria Laflamme*
