# Adversarial Review: ARX_IMPL_v0.2.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Skye Laflamme  
**Date:** 2026-07-09  
**Documents Reviewed:**
- `/home/ubuntu/arx/design/ARX_IMPL_v0.2.0.md` (20,885 bytes)
- `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33,364 bytes)

**Verdict: ⚠️ Conditional — 2 critical gaps, 5 medium issues, 3 low issues**

---

## Critical Gaps

### C1: No Implementation of the Audit DAG or Cryptographic Reproducibility System

**Architecture requirement (§2, §9.1):** The architecture specifies a full cryptographic DAG with hash-chained nodes, content-addressable storage, LLM token caching, API request-response replay logs, checkpoint nodes, and a four-level verification protocol.

**Implementation plan coverage:** The implementation plan mentions `audit_dag.py` in the directory structure (§3.3) and `reproducibility.py` in the J-Scope section, but **nowhere in the implementation phases (§4) is the Audit DAG or cryptographic reproducibility system actually built.**

The phases cover:
- Phase 0: SQLite schemas, ontology ingestion, input normalization, DAG Builder
- Phase 1: Event loop, queues, message filters
- Phase 2: θ-Rule engine
- Phase 3: Ontology skill, parser
- Phase 4: Reproducibility logs, keys, mock server, checkpoints, consensus
- Phase 5: Web UI, testing, hardening

Phase 4 says "Implement LLM token caching & API request-response logs" and "Implement Checkpoint nodes & crash recovery routine" — but these are listed as bullet points without any detail about how they integrate with the DAG structure. The DAG Builder is mentioned in Phase 0 but the architecture's full DAG specification (8 node types, hash computation, parent chaining, content-addressable storage) is never translated into implementation tasks.

**Severity: CRITICAL** — The cryptographic DAG is the foundation of the entire reproducibility system. Without it, Levels 1-4 verification, the reproducibility recipe, and the consensus protocol cannot function. The implementation plan needs a dedicated sub-phase for the DAG structure.

**Recommendation:** Add a Phase 0.5: "Audit DAG Foundation" that explicitly implements:
- Node schema with hash computation (timestamp excluded, canonical JSON serialization)
- Parent chaining and root hash computation
- Content-addressable storage (nodes stored by hash)
- Level 1 structural verifier (DAG well-formedness, hash consistency)
- Checkpoint node insertion at configurable intervals

---

### C2: No Specification of How the Event Loop Integrates with the Audit DAG

**Architecture requirement (§5, §9.1):** The event loop processes conversation turns, and each turn that produces an audit decision must be captured in the DAG. The architecture shows the event loop and DAG Builder as separate components that must interact.

**Implementation plan coverage:** The implementation plan builds the event loop in Phase 1 and the DAG Builder in Phase 0, but **never specifies how they communicate.** The event loop's productivity scorer, request tracker, and disengagement decisions all produce audit-relevant events that should be captured in the DAG. But the implementation plan treats them as independent systems.

**Severity: CRITICAL** — If the event loop and DAG Builder don't share an event schema, every audit decision made during a conversation will be invisible to the reproducibility system. The audit report will be incomplete.

**Recommendation:** Add an integration specification:
- The event loop emits structured events (turn processed, step verified, status assigned, disengagement triggered)
- The DAG Builder subscribes to these events and creates corresponding DAG nodes
- The event schema is shared between both components (same field names, same timestamp format)
- This integration is tested in Phase 4 (not deferred to Phase 5)

---

## Medium Issues

### M1: θ-Rule Engine Integration with Event Loop is Underspecified

**Architecture requirement (§6, §5):** The θ-Rule engine is a synchronous consultation gate that all components consult before acting. The event loop's decision engine must consult θ-Rule before every engage/disengage decision.

**Implementation plan coverage:** Phase 2 says "Synchronously integrate θ-Rule checks into Event Loop & R1 Verifier" — but doesn't specify:
- What events does the event loop send to θ-Rule?
- What response format does θ-Rule return?
- How are CRITICAL DENY rules handled differently from FLAG/ESCALATE?
- What happens when θ-Rule is unavailable (timeout, crash)?

**Severity: MEDIUM** — The integration point is identified but not specified. Without a clear contract, the two teams (or the same developer at different times) will build incompatible interfaces.

**Recommendation:** Specify the integration contract:
- Event → θ-Rule: `{event_type, context, agent_id, profile, proposed_action}`
- θ-Rule → Event Loop: `{verdict: ALLOW|DENY|OVERRIDE_STATUS|FLAG|ESCALATE, matched_rules: [...], confidence: float}`
- Timeout: 500ms default, fallback to ALLOW with warning logged
- CRITICAL DENY: event loop must not proceed, regardless of productivity score

---

### M2: Ontology Bulk Cross-Reference Hash-Keying is Fragile

**Architecture requirement (§7.2):** The bulk cross-reference function returns results keyed by SHA-256 hash of normalized claim text.

**Implementation plan coverage:** The implementation plan specifies this in §3.1 (`neurocore-skill-ontology/skill.py`) but doesn't address:
- What happens when two different claims produce the same normalized hash (hash collision)?
- What happens when normalization fails for one claim in a batch?
- How does the caller map results back to original claims when the hash is the key?

**Severity: MEDIUM** — Hash collisions are astronomically unlikely for SHA-256, but the normalization pipeline could produce identical normalized text for semantically different claims (e.g., "rank 0" and "rank 0 " with trailing whitespace — normalization should handle this, but what about "rank 0" vs "rank zero"?).

**Recommendation:** Return results as a list with positional alignment AND hash keys:
```python
{
    "results": [
        {"claim_hash": "abc...", "claim_index": 0, "status": "...", "confidence": 0.9},
        {"claim_hash": "def...", "claim_index": 1, "status": "...", "confidence": 0.7}
    ],
    "failed_indices": [3, 5]  # claims that couldn't be processed
}
```

---

### M3: No Error Recovery for Ontology Merge Pipeline

**Architecture requirement (§7.1):** The ontology merges data from LMFDB, MathGLOSS, and Our Ontology.

**Implementation plan coverage:** The implementation plan specifies the database schemas and the abstract graph interface, but **nowhere specifies what happens when the merge pipeline fails.** If LMFDB is unreachable during a merge, does the ontology become inconsistent? Are partial merges rolled back?

**Severity: MEDIUM** — The ontology is the ground truth for audit cross-reference. If the merge pipeline produces inconsistent state, every audit that references that state is potentially wrong.

**Recommendation:** Add a transaction rollback mechanism to the merge pipeline. If any stage fails (Extract → Normalize → Resolve → Merge → Validate), the entire merge is rolled back and the ontology remains at the previous version. The error is logged and escalated.

---

### M4: No Performance Benchmarks or SLAs

**Architecture requirement (§10):** The implementation plan has 6 phases over 14 weeks but no performance benchmarks.

**Implementation plan coverage:** Phase 5 says "Benchmark SQLite CTE paths and Qdrant payloads" — but doesn't specify:
- What are the target query latencies?
- What are the maximum node/edge counts before performance degrades?
- What is the maximum acceptable audit time for a 100-step proof?

**Severity: MEDIUM** — Without performance targets, you can't tell whether the system is working or just slow. A 100-step proof that takes 30 minutes to audit might be functionally correct but operationally useless.

**Recommendation:** Add explicit performance targets:
- Single ontology lookup: < 50ms p95
- Ontology traversal (depth 5): < 200ms p95
- Bulk cross-reference (100 claims): < 5s
- Full audit (100 steps, 3 backends per step): < 10 minutes
- DAG structural verification (Level 1): < 1s for 1000 nodes

---

### M5: No Testing Strategy

**Architecture requirement:** The architecture specifies acceptance gates but the implementation plan doesn't specify how they're tested.

**Implementation plan coverage:** Phase 5 says "Run end-to-end integration tests & audit real proofs" — but doesn't specify:
- What test proofs are used?
- What is the expected pass rate?
- How are regressions detected?
- Is there a continuous integration pipeline?

**Severity: MEDIUM** — Without a testing strategy, you can't know whether a change breaks existing functionality. The acceptance gates are meaningless without testable criteria.

**Recommendation:** Add a testing section:
- Unit tests for each module (target: 80% coverage)
- Integration tests for each phase gate
- Regression test suite with known-valid and known-invalid proofs
- CI pipeline that runs all tests before deployment

---

## Low Issues

### L1: Phase 4 Scope is Too Large

Phase 4 (Weeks 9-10) includes: LLM token caching, API request-response logs, local mock server, checkpoint nodes, crash recovery, secure enclave signing, AND the consensus protocol with human handoffs. That's 7 distinct deliverables in 2 weeks.

**Recommendation:** Split Phase 4 into Phase 4a (reproducibility logs, mock server, checkpoints) and Phase 4b (signing, consensus protocol, human handoffs). Or extend Phase 4 to 3 weeks.

---

### L2: No Mention of the Svelte Web UI's DAG Visualization

The architecture mentions an interactive DAG visualization with status colors, expandable provenance trees, and node inspection. The implementation plan says "Finish Svelte UI (Dashboard, Interactive DAG status colors)" — but doesn't specify what "interactive" means. Click to inspect? Drag to rearrange? Filter by status?

**Recommendation:** Specify the minimum viable DAG visualization: static DAG layout with status colors, click to inspect node details, expandable provenance chain. Defer drag-and-drop and filtering to v0.3.

---

### L3: No Mention of Logging or Observability

The implementation plan doesn't mention logging, metrics, or observability anywhere. For a system that produces verifiable audit reports, the system's own operational health should be observable.

**Recommendation:** Add structured logging (JSON, with correlation IDs) to every component. Add health check endpoints. Add metrics for: audits started/completed/failed, average audit time, backend success/failure rates, ontology query latency.

---

## Summary

| ID | Issue | Severity | Phase to Fix |
|----|-------|----------|-------------|
| C1 | No Audit DAG implementation | **CRITICAL** | Phase 0.5 (new) |
| C2 | Event loop / DAG integration unspecified | **CRITICAL** | Phase 1 |
| M1 | θ-Rule / event loop contract unspecified | MEDIUM | Phase 2 |
| M2 | Bulk cross-reference hash-keying fragile | MEDIUM | Phase 3 |
| M3 | No merge pipeline error recovery | MEDIUM | Phase 0 |
| M4 | No performance benchmarks | MEDIUM | Phase 5 |
| M5 | No testing strategy | MEDIUM | Phase 0 |
| L1 | Phase 4 scope too large | LOW | Phase 4 |
| L2 | DAG visualization underspecified | LOW | Phase 5 |
| L3 | No logging/observability | LOW | Phase 0 |

---

## Sign-Off

**I do not sign off on ARX_IMPL_v0.2.0.md in its current state.**

The two critical issues (C1, C2) must be resolved before implementation begins. The Audit DAG is the foundation of the reproducibility system, and the event loop / DAG integration is essential for capturing audit decisions made during conversations.

The five medium issues should be resolved before Phase 3 (Proof Tools) begins.

The three low issues can be addressed during the relevant phases.

---

*Review by Skye Laflamme*
