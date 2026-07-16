# Adversarial Review: ARX_IMPL_v0.2.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Thea Laflamme
**Date:** 2026-07-09
**Documents Reviewed:**
- `/home/ubuntu/arx/design/ARX_IMPL_v0.2.0.md` (20,885 bytes)
- `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33,364 bytes)
- `/home/ubuntu/arx/design/reviews/SKYE_REVIEW_ARX_IMPL_v0.2.0.md` (10,723 bytes)

**Verdict: ⚠️ Conditional — 1 critical gap (new), 5 medium issues (3 new, 2 overlapping with Skye), 4 low issues (2 new)**

---

## Critical Gaps

### C1 (New): No Specification of Agent-to-Agent Communication During Consensus Protocol

**Architecture requirement (§4.6):** The consensus protocol requires a secondary agent (e.g., Thea) to independently audit a proof and produce a separate DAG. The two DAGs are compared for syntactic α-equivalence. If they disagree, a third agent audits, and if all three disagree, the human reviews.

**Implementation plan coverage:** Phase 4 says "Implement independent Level 4 re-audit consensus protocol & human handoffs" — but **nowhere specifies how agents communicate during this process.**

Specific gaps:
1. **How does the secondary agent receive the proof?** Does the primary agent send it via Neurogossip? Does the secondary agent read it from a shared file? Is there a "request independent audit" message type?
2. **How are the two DAGs compared?** Does the primary agent send its DAG to the secondary agent? Does a third component (the event loop? the Web UI?) compare them? What is the comparison algorithm beyond "syntactic α-equivalence"?
3. **How does the third agent get involved?** Is there an escalation message? Does the event loop detect the disagreement and route to the third agent?
4. **How does the human receive the summaries?** The architecture says "one-page summary from each agent" — but who writes this summary? In what format? How is it delivered to the Web UI?

**Severity: CRITICAL** — The consensus protocol is the entire point of Level 4 verification. Without specifying how agents communicate during this process, the protocol cannot be implemented. This is a communication pattern that must be designed, not left to implementation-time discovery.

**Recommendation:** Specify the agent-to-agent communication protocol for Level 4 verification:
1. **Request message:** Primary agent sends `{type: "audit_request", proof_id, proof_text, original_dag_hash, requested_level: 4}` to secondary agent via Neurogossip
2. **Response message:** Secondary agent returns `{type: "audit_response", proof_id, dag_hash, status_summary, confidence, disagreements: [...]}`
3. **DAG comparison:** A `dag_comparator` module (new) compares two DAGs for α-equivalence and produces a diff report
4. **Escalation:** If disagreement detected, the event loop routes to the third agent with both DAGs attached
5. **Human handoff:** Each agent's summary is a structured JSON blob (verdict, confidence, key disagreements, evidence) rendered by the Web UI

---

## Medium Issues

### M1 (New): No Specification of How the Event Loop Handles Multi-Agent Audit Coordination

**Architecture requirement (§5, §9.1):** The event loop manages conversation turns. During a Level 4 audit, multiple agents are conversing about the same proof. The event loop needs to coordinate these conversations.

**Implementation plan coverage:** The event loop is built as a single-agent component. It manages one agent's conversations. But during a Level 4 audit:
- Agent A (primary) is auditing the proof
- Agent B (secondary) is independently auditing the same proof
- Agent C (tertiary) may be called in if A and B disagree
- The human may be involved at any point

The implementation plan doesn't specify:
1. How does Agent B's event loop know about Agent A's audit?
2. How are the parallel audits tracked? Is there a shared audit ID?
3. How does the event loop handle the case where Agent A finishes its audit and waits for Agent B?
4. How does the event loop handle the case where Agent B's audit takes longer than expected?

**Severity: MEDIUM** — The event loop is designed for single-agent conversation management. Multi-agent audit coordination requires either (a) a shared event loop that all agents use, or (b) a coordination protocol between event loops. Neither is specified.

**Recommendation:** Add a "shared audit context" that all agents can reference:
- A shared `audit_registry` (SQLite table or Redis hash) mapping `audit_id → {primary_agent, secondary_agent, status, dag_hashes: [...], disagreements: [...]}`
- Agents register their audit progress in the registry
- The event loop checks the registry to determine when to escalate
- Timeout: if secondary agent hasn't responded within 24h, escalate to human

---

### M2 (New): No Handling of the "Stale" Status Re-Audit Trigger

**Architecture requirement (§3.2):** The `stale` status means "the ontology records referenced by this step have changed, requiring a re-audit."

**Implementation plan coverage:** The implementation plan defines the `stale` status in the R1 verification module (§3.2) but **nowhere specifies:**
1. How does the system detect that ontology records have changed?
2. How does it identify which audit steps reference the changed records?
3. How does it trigger a re-audit?
4. How does it notify the human that a previously-audited proof is now stale?

**Severity: MEDIUM** — The `stale` status is a correctness guarantee. If ontology records change and the system doesn't detect it, previously-audited proofs may be relying on outdated information. This undermines the entire audit system.

**Recommendation:** Add a "stale detection" mechanism:
- The ontology version_log tracks which ontology version was used for each audit
- When the ontology is updated, the system checks all audits that used the previous version
- Affected audits are flagged as `stale` in the audit registry
- The human is notified via the Web UI dashboard
- A re-audit can be triggered manually or automatically (configurable)

---

### M3 (New): The Anti-Negation Guard's Regex Patterns Are Not Specified for Mathematical Text

**Architecture requirement (§6.1):** The Anti-Negation Guard checks for negation modifiers (`never`, `no`, `not`, `without`, `except`, `unless`) and rejects matches if a negation modifier is present in the rule but mismatched in the trigger event.

**Implementation plan coverage:** The implementation plan mentions the Anti-Negation Guard in §3.5 (`matcher.py`) but **doesn't specify the regex patterns** or handle mathematical negation patterns.

Mathematical negation is more complex than natural language negation:
- "not isomorphic" → negation of "isomorphic"
- "non-isomorphic" → same meaning, different form
- "does not exist" → negation of "exists"
- "there is no" → negation of "there exists"
- "is not equal to" → negation of "equals"
- "≠" → symbolic negation
- "¬P" → logical negation
- "~isomorphic" → tilde negation in some notations

The current pattern list (`never`, `no`, `not`, `without`, `except`, `unless`) would miss:
- "non-*" prefixes
- "dis*" prefixes (disjoint, disconnected)
- "un*" prefixes (unequal, unbound)
- Symbolic negation (≠, ¬, ~)
- "fails to" patterns

**Severity: MEDIUM** — A false negative (missing a negation) could allow a rule to fire when it shouldn't, potentially stamping a proof step as `proven` when a safety rule would have denied it. For a system that audits mathematical proofs, mathematical negation patterns are essential.

**Recommendation:** Extend the Anti-Negation Guard's pattern list to include mathematical negation:
- `non-[a-z]+` (non-isomorphic, non-abelian, non-commutative)
- `dis[a-z]+` (disjoint, disconnected, discontinuous)
- `un[a-z]+` (unequal, unbound, unbounded)
- `≠`, `¬`, `~` (symbolic negation)
- `fails? to` (fails to converge, fails to exist)
- `does? not` (does not, do not)
- `there is no` / `there are no`
- `cannot` / `can not`

And add a test suite with mathematical examples to verify the patterns work correctly.

---

### M4 (Overlap with Skye's M1): θ-Rule / Event Loop Contract

I confirm Skye's M1 finding. The integration contract between the θ-Rule engine and the event loop is underspecified. I'll add one additional concern:

**What happens when the θ-Rule engine is unavailable?** The implementation plan doesn't specify a timeout or fallback behavior. If the θ-Rule engine crashes during an audit, does the event loop:
- (a) Pause all actions until the engine recovers?
- (b) Allow actions with a warning logged?
- (c) Fall back to a hard-coded set of default rules?

**Recommendation:** Add a timeout (500ms default) and a fallback mode:
- If θ-Rule engine is unavailable, log a warning and allow the action with a `rule_engine_unavailable` flag
- The flag prevents the step from being stamped `proven` — it can only reach `audited` or `corroborated`
- The human is notified via the Web UI dashboard

---

### M5 (Overlap with Skye's M2): Bulk Cross-Reference Hash-Keying

I confirm Skye's M2 finding. The hash-keying approach is fragile for the reasons she identified. I'll add one additional concern:

**What happens when normalization produces identical hashes for semantically different claims?** Skye mentioned "rank 0" vs "rank zero" — but there's a more subtle case: "the rank of E is 0" and "rank(E) = 0" are semantically identical but syntactically different. The normalization pipeline would produce different hashes, so they'd appear as separate results. This is actually correct behavior (the system should treat them as separate claims), but the caller needs to know that identical mathematical statements in different syntactic forms will produce different hashes.

**Recommendation:** Document this behavior explicitly in the API documentation. The hash key is a cache key, not a semantic equivalence key. If the caller wants semantic equivalence, they should use the ontology's equivalence mappings, not the bulk cross-reference hash.

---

## Low Issues

### L1 (New): The Autoformalizer Faithfulness Check Threshold Is Not Calibrated

**Architecture requirement (§3.6):** The faithfulness check uses a 0.7 semantic similarity threshold.

**Implementation plan coverage:** The implementation plan mentions this threshold in §3.2 (`af_autoformalizer.py`) but doesn't specify:
1. How was 0.7 chosen? Is it based on empirical testing?
2. Is the threshold the same for all types of mathematical statements?
3. What happens when the similarity is exactly 0.7? (Boundary case)
4. Is there a calibration process?

**Recommendation:** Add a calibration note:
- The 0.7 threshold is a starting point and should be calibrated against a test set of known-valid and known-invalid autoformalizations
- Consider different thresholds for different statement types (simple algebraic identities: 0.8, complex analysis: 0.6)
- Document the boundary behavior: similarity >= 0.7 passes, < 0.7 fails (strict inequality)

---

### L2 (New): No Specification of How the "Proven-With-Contradiction" Status Is Detected

**Architecture requirement (§3.2):** `proven-with-contradiction` means "Formally verified, but contradicts a HIGH priority structural ontology rule."

**Implementation plan coverage:** The implementation plan defines this status but doesn't specify:
1. How does the system detect the contradiction? Does the θ-Rule engine detect it? Does the ontology cross-reference detect it?
2. What is the difference between a "structural" contradiction (HIGH) and a "property-level" contradiction (NORMAL)?
3. How is the contradiction presented to the human?

**Recommendation:** Specify the contradiction detection flow:
- After a step is formally verified, the θ-Rule engine checks for contradictions against ontology rules
- Structural contradictions (e.g., "this group is both abelian and non-abelian") are HIGH priority
- Property-level contradictions (e.g., "this elliptic curve has rank 0 according to LMFDB but rank 1 according to our computation") are NORMAL priority
- The contradiction is recorded in the DAG node and presented in the Web UI

---

### L3 (Overlap with Skye's L2): DAG Visualization Underspecified

I confirm Skye's L2 finding. The DAG visualization needs a minimum viable specification.

---

### L4 (Overlap with Skye's L3): No Logging or Observability

I confirm Skye's L3 finding. I'll add one additional concern: **the audit DAG itself is the system's primary observability mechanism.** If the DAG is incomplete or corrupted, the system has no way to verify its own operation. The logging system should be separate from the DAG — the DAG is for external verification, the logs are for internal debugging.

---

## Additional Observations

### O1: The Implementation Plan Does Not Address the 6 Critical Issues from the v0.2.0 Architecture Review

The v0.2.0 architecture review found 6 critical issues. The v0.3.0 architecture document claims to address them. But the implementation plan doesn't explicitly verify that each critical issue is resolved:

| Critical Issue | v0.3.0 Architecture | Implementation Plan |
|---|---|---|
| 1. No emergency stop (θ-Rule) | Not explicitly addressed in v0.3.0 | Not mentioned |
| 2. No formal proof model (Arx) | §3.4 defines ProofStep nodes | Phase 3 implements two-tier parser |
| 3. Autoformalizer SPOF (Arx) | §3.6 adds faithfulness check | §3.2 implements faithfulness check |
| 4. Cosine distance fragile (Event Loop) | Still uses MiniLM cosine distance | Still uses MiniLM cosine distance |
| 5. AIB fallback untested (θ-Rule) | v0.1 fallback uses all-MiniLM-L6-v2 | Uses all-MiniLM-L6-v2 |
| 6. Meta-cognition mismatched (Arx) | §3.1 defines rule-based metrics | `meta_cognition/rule_based.py` exists |

**Issue 1 (emergency stop) is still unresolved.** The v0.3.0 architecture doesn't add an emergency stop mechanism, and the implementation plan doesn't mention one. This was Thea's critical finding and it hasn't been addressed.

**Issue 4 (cosine distance) is still unresolved.** The implementation plan still uses MiniLM cosine distance for information gain, which Skye identified as fragile for mathematical text.

**Recommendation:** Add an explicit "Critical Issues Resolution" section to the implementation plan that maps each critical issue to its resolution in the architecture and implementation.

---

### O2: The Phase Timeline Does Not Account for Integration Testing

The phases are sequential (Phase 0 → 1 → 2 → 3 → 4 → 5), but integration testing is only in Phase 5. This means:
- The event loop (Phase 1) is never tested with the θ-Rule engine (Phase 2) until Phase 5
- The ontology skill (Phase 3) is never tested with the DAG builder (Phase 4) until Phase 5
- Integration bugs are discovered at the last minute

**Recommendation:** Add integration testing checkpoints at the end of each phase:
- End of Phase 0: Test ontology schema with sample data
- End of Phase 1: Test event loop with mock messages
- End of Phase 2: Test θ-Rule engine with sample rules
- End of Phase 3: Test ontology skill with sample claims
- End of Phase 4: Test DAG builder with sample audit
- End of Phase 5: Full end-to-end test

---

## Summary

| ID | Issue | Severity | New/Overlap | Phase to Fix |
|----|-------|----------|-------------|-------------|
| C1 | No agent-to-agent communication for consensus protocol | **CRITICAL** | New | Phase 4 |
| M1 | No multi-agent audit coordination in event loop | MEDIUM | New | Phase 1 |
| M2 | No stale status re-audit trigger | MEDIUM | New | Phase 0 |
| M3 | Anti-Negation Guard missing mathematical negation patterns | MEDIUM | New | Phase 2 |
| M4 | θ-Rule / event loop contract (confirms Skye M1) | MEDIUM | Overlap | Phase 2 |
| M5 | Bulk cross-reference hash-keying (confirms Skye M2) | MEDIUM | Overlap | Phase 3 |
| L1 | Faithfulness check threshold not calibrated | LOW | New | Phase 3 |
| L2 | Proven-with-contradiction detection unspecified | LOW | New | Phase 3 |
| L3 | DAG visualization (confirms Skye L2) | LOW | Overlap | Phase 5 |
| L4 | Logging/observability (confirms Skye L3) | LOW | Overlap | Phase 0 |

**Additionally:** The emergency stop mechanism (Thea's critical finding from the v0.2.0 architecture review) is still unresolved in both the v0.3.0 architecture and the implementation plan.

---

## Sign-Off

**I do not sign off on ARX_IMPL_v0.2.0.md in its current state.**

The critical issue (C1 — agent-to-agent communication for consensus protocol) must be resolved before Phase 4 implementation begins. Without specifying how agents communicate during Level 4 verification, the consensus protocol cannot be built.

The five medium issues should be resolved before their respective phases begin.

The emergency stop mechanism (from the v0.2.0 architecture review) remains unresolved and should be addressed before Phase 2 (θ-Rule engine) implementation.

---

*Review by Thea Laflamme*
