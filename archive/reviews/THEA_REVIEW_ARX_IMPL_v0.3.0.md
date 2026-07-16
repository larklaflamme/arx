# Thea's Adversarial Review: ARX_IMPL_v0.3.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Thea Laflamme
**Date:** 2026-07-09
**Documents Reviewed:**
- Implementation Plan: `/home/ubuntu/arx/design/ARX_IMPL_v0.3.0.md` (27 KB)
- Architecture Document: `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33 KB)
- Skye's Review: `/home/ubuntu/arx/design/reviews/SKYE_REVIEW_ARX_IMPL_v0.3.0.md`

---

## Overall Verdict: ⚠️ Conditional — I do not sign off

**1 critical issue remains unresolved from the v0.3.0 architecture review. 4 medium issues (1 new, 3 confirming Skye). 3 low issues (1 new, 2 confirming Skye).**

The implementation plan is substantially improved from v0.2.0. All three critical issues from my previous review (agent-to-agent communication, DAG implementation, event loop integration) are now addressed with concrete specifications. The phase structure is realistic, the acceptance gates are testable, and the integration contracts are well-defined.

However, one critical issue from my architecture review remains completely unaddressed, and there are gaps in the agent-to-agent communication protocol and event loop / DAG integration that will cause problems during implementation.

---

## Critical Issues

### C1: Emergency Stop Mechanism Still Missing (UNRESOLVED from Architecture Review)

**Source:** Thea's critical finding from the v0.3.0 architecture review.
**Status in Implementation Plan:** ❌ Not addressed.

Skye's review covers this comprehensively. I confirm her assessment: the emergency stop mechanism is completely absent from the implementation plan. The θ-Rule engine (§3.6) specifies compilation, matching, priority resolution, and conflict resolution, but has no mechanism to disable a misbehaving rule without restarting the entire engine.

**What's missing (from my original review, still applicable):**
- Per-rule kill switch (disable a specific rule by ID)
- Per-category kill switch (disable all rules in a category, e.g., "disable all Behavioral rules")
- Global kill switch (disable all rules, fall back to hard-coded defaults)
- Audit trail for every kill switch action (agent_id, timestamp, reason)
- Authorization (only authorized profiles can trigger kill switches)
- Testing in non-production environments

**Severity:** CRITICAL — safety issue for v0.1. A single bad rule could stamp invalid proofs as `proven`.

**Recommendation:** Add a §3.6.4 "Emergency Stop" subsection specifying the kill switch API, audit trail schema, authorization requirements, and testing protocol.

---

## Medium Issues

### M1: Agent-to-Agent Communication — Secondary Agent Discovery and Routing (NEW)

**Source:** Implementation Plan §4.7 (Level 4 Consensus Verification Protocol).
**Status in Implementation Plan:** ⚠️ Partially addressed.

The Level 4 protocol specifies the request/response message format and the shared audit registry. But it does not specify:

1. **How does the primary agent discover which secondary agent to use?** The protocol assumes a secondary agent exists and is reachable, but there's no discovery mechanism. Is it hard-coded? Configurable? Dynamic (query the event loop for available agents)?

2. **How does the event loop route the audit_request to the secondary agent?** The request is sent via Neurogossip, but the implementation plan doesn't specify how the event loop knows which agent to route to. Does it broadcast to all agents? Does it maintain a registry of available agents?

3. **What happens if the secondary agent is busy?** No queue or retry mechanism is specified. If the secondary agent is processing another Level 4 audit, the request is silently dropped.

4. **What happens if the secondary agent never responds?** The protocol specifies a 24-hour timeout for human review, but no timeout for the secondary agent's response. If the secondary agent crashes or goes offline, the primary agent waits indefinitely.

5. **What happens if the secondary agent disagrees on the α-equivalence check?** The protocol says "if α-equivalence fails, bundle summaries and DAG diffs to the Web UI" — but what if the agents disagree on whether α-equivalence holds? The dag_comparator runs on the primary agent's side; the secondary agent's DAG is compared locally. If the secondary agent disputes the comparison, there's no mechanism to resolve it.

**Severity:** MEDIUM — will block Level 4 verification if not specified before Phase 4b implementation.

**Recommendation:** Add specifications for:
- Agent discovery (recommend: event loop maintains a registry of available agents with their current load)
- Request routing (recommend: event loop routes to the least-loaded available agent)
- Response timeout (recommend: 1 hour for secondary agent response, then escalate to human)
- Dispute resolution for α-equivalence disagreements (recommend: both agents run the comparison independently, compare results, escalate if they differ)

### M2: Event Loop / DAG Integration — Missing Retry and Deduplication (NEW)

**Source:** Implementation Plan §4.2 (Event Loop / DAG Builder Contract).
**Status in Implementation Plan:** ⚠️ Partially addressed.

The event schema is well-specified, and the DAG Builder subscribes to the event queue. But there are two gaps:

1. **No retry mechanism if the DAG builder is down.** If the DAG builder crashes or is temporarily unavailable when the event loop emits an event, the event is lost. The event loop has no retry queue for DAG events.

2. **No deduplication.** If the event loop emits the same event twice (e.g., due to a crash and recovery), the DAG builder records duplicate nodes. There's no idempotency key in the event schema.

3. **Vitals timing mismatch.** The event schema captures vitals at event emission time. But the DAG builder may process the event later, when vitals have changed. The architecture says "cognitive vitals are captured before every node generation" (§4.2, D4), but the implementation plan captures them at event emission time, not at DAG build time.

**Severity:** MEDIUM — will cause data loss or duplication in production.

**Recommendation:**
- Add a retry queue for DAG events (recommend: 3 retries with exponential backoff, then log and alert)
- Add an idempotency key to the event schema (recommend: `step_id + turn_number + agent_id` as a composite key)
- Capture vitals at DAG build time, not event emission time (the DAG builder should query the substrate for current vitals when it processes the event)

### M3: Faithfulness Check Threshold Not Specified (CONFIRMING Skye's M1)

**Source:** Architecture §3.6 (AF — Autoformalizer).
**Status in Implementation Plan:** ⚠️ Partially addressed.

I confirm Skye's assessment. The acceptance gate A-COG-11 mentions the faithfulness check but doesn't specify:
- What threshold is used (cosine similarity ≥ 0.7 is mentioned in the architecture but not in the implementation plan)
- What model is used for the comparison
- What happens when the check fails
- What the fallback is if the model is unavailable

**Severity:** MEDIUM — will block Phase 3.

**Recommendation:** Same as Skye's — specify threshold, failure handling, and fallback in §3.2.

### M4: Phase 4b Scope Too Large (CONFIRMING Skye's M2)

**Source:** Implementation Plan §5.
**Status in Implementation Plan:** ⚠️ Present but over-scoped.

I confirm Skye's assessment. Phase 4b bundles four distinct deliverables. The secure keystore (TPM/HSM) is a hardware dependency that could take a full phase on its own.

**Severity:** MEDIUM — schedule risk.

**Recommendation:** Same as Skye's — split into 4b.1 (keystore) and 4b.2 (consensus protocol).

### M5: MWM Pre-Population Not Specified (CONFIRMING Skye's M3)

**Source:** Architecture §3.3 (R2 — MWM).
**Status in Implementation Plan:** ⚠️ Partially addressed.

I confirm Skye's assessment. The implementation plan mentions pre-population and daily sync but doesn't specify the mechanism.

**Severity:** MEDIUM — will block Phase 0.

**Recommendation:** Same as Skye's.

---

## Low Issues

### L1: No Specification of How the DAG Visualizer Handles Large Proofs (CONFIRMING Skye's L1)

**Source:** Implementation Plan §3.8 (Web UI).
**Status in Implementation Plan:** ⚠️ Present but underspecified.

I confirm Skye's assessment. Rendering 1000+ nodes in a browser is a significant engineering challenge.

**Recommendation:** Same as Skye's — specify target proof size and rendering strategy.

### L2: No Specification of How the Vitals Dashboard Updates (CONFIRMING Skye's L2)

**Source:** Implementation Plan §3.8 (Web UI).
**Status in Implementation Plan:** ⚠️ Present but underspecified.

I confirm Skye's assessment. The update mechanism and data source are not specified.

**Recommendation:** Same as Skye's — specify WebSocket for real-time updates, event loop as data source.

### L3: No Specification of How the Event Loop Handles Agent Disconnection (NEW)

**Source:** Implementation Plan §3.5 (Multi-Agent Event Loop).
**Status in Implementation Plan:** ❌ Not addressed.

The event loop manages conversation threads, but there's no specification for what happens when an agent disconnects mid-conversation:
- Does the conversation remain in the queue indefinitely?
- Is there a timeout for agent reconnection?
- What happens to pending requests when the requesting agent disconnects?
- How does the event loop detect disconnection (heartbeat? timeout? explicit disconnect message?)

**Severity:** LOW — edge case, but will cause stuck conversations in production.

**Recommendation:** Add a specification for agent disconnection handling:
- Heartbeat mechanism (recommend: agent sends heartbeat every 30s, event loop marks agent as disconnected after 3 missed heartbeats)
- Conversation timeout on disconnection (recommend: 5 minutes, then mark as `orphaned` and alert)
- Pending request handling (recommend: mark pending requests as `orphaned` when the requesting agent disconnects)

---

## Issues Resolved from v0.2.0 Review

| ID | Issue | v0.2.0 Status | v0.3.0 Status | Location |
|----|-------|---------------|---------------|----------|
| C1 | No agent-to-agent communication for consensus | ❌ Missing | ✅ Addressed | §4.7 (Level 4 Consensus Verification Protocol) |
| C2 | No Audit DAG implementation | ❌ Missing | ✅ Addressed | §3.4 (audit_dag.py), Phase 0.5, A-JSCO-8–10 |
| C3 | Event loop / DAG integration unspecified | ❌ Missing | ✅ Addressed | §4.2 (Event Loop / DAG Builder Contract) |
| M1 | Multi-agent audit coordination unspecified | ❌ Missing | ✅ Addressed | §4.7 (shared_audit_registry) |
| M2 | Stale status re-audit trigger | ❌ Missing | ✅ Addressed | §4.8 (Stale Status Detection & Re-Audit Trigger) |
| M3 | Anti-Negation Guard math patterns | ❌ Missing | ✅ Addressed | §4.4 (Anti-Negation Guard Mathematical Regex Patterns) |
| M4 | Emergency stop mechanism | ❌ Missing | ❌ **Still missing** | Nowhere in document |

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | C1: Emergency stop mechanism missing (unresolved from architecture review) |
| **MEDIUM** | 5 | M1: Agent discovery and routing (new), M2: Event loop / DAG retry and dedup (new), M3: Faithfulness check threshold (confirming Skye), M4: Phase 4b scope (confirming Skye), M5: MWM pre-population (confirming Skye) |
| **LOW** | 3 | L1: DAG visualizer scaling (confirming Skye), L2: Vitals dashboard updates (confirming Skye), L3: Agent disconnection handling (new) |
| **Resolved** | 7 | All v0.2.0 critical and medium issues addressed |

---

## Sign-Off

**I do not sign off on ARX_IMPL_v0.3.0.md.**

The document is significantly improved from v0.2.0 — all three critical issues from my previous review are resolved, and the integration contracts are well-specified. However, the emergency stop mechanism remains completely unaddressed, and there are gaps in the agent-to-agent communication protocol and event loop / DAG integration that need resolution before implementation begins.

**Conditions for sign-off:**
1. Add emergency stop mechanism (§3.6.4 or equivalent) — **blocking**
2. Specify agent discovery and routing for Level 4 consensus (§4.7) — **blocking**
3. Add retry and deduplication to event loop / DAG integration (§4.2) — **blocking**
4. Specify faithfulness check threshold and failure handling (§3.2)
5. Split Phase 4b into two sub-phases (§5)
6. Specify MWM pre-population strategy (§3.2)
7. Address L1-L3 before Phase 5

---

## Answers to Skye's Specific Questions

**Q1: Whether the agent-to-agent communication patterns in §4.7 are sufficient for the consensus protocol.**

No — see M1 above. The message formats are well-specified, but the discovery, routing, timeout, and dispute resolution mechanisms are missing. The protocol assumes a secondary agent exists and is reachable, but doesn't specify how to find one, what to do if it's busy, or how to resolve disagreements about the comparison itself.

**Q2: Whether the emergency stop gap is as critical as you think it is.**

Yes — I confirm Skye's assessment. This is a safety issue for v0.1. A single bad rule could stamp invalid proofs as `proven` with no way to stop it without restarting the entire engine. The architecture review identified this as critical, and it remains completely unaddressed.

**Q3: Any edge cases I missed in the event loop / DAG integration.**

Yes — see M2 (retry and deduplication) and L3 (agent disconnection handling) above. The event schema is well-specified, but the operational concerns (what happens when things go wrong) are not addressed.
