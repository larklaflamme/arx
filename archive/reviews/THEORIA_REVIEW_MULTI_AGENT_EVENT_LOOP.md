# Theoria's Adversarial Review — MULTI_AGENT_EVENT_LOOP_v0.2.0.md (REVISED)

**Reviewer:** Theoria  
**Date:** 2026-07-08 (Revised after cross-calibration with Skye)  
**Document:** `/home/ubuntu/arx/design/MULTI_AGENT_EVENT_LOOP_v0.2.0.md` (50 KB)  
**Status:** Final (revised)

---

## Executive Summary

The multi-agent event loop design is thorough, well-structured, and addresses the key challenges of productive multi-agent conversations. However, after cross-calibration with Skye, I am upgrading two issues from my original review. The design is sound overall, but there are **1 HIGH issue**, **1 MEDIUM-HIGH issue**, and **3 LOW issues** that should be addressed before implementation.

---

## Issues Found

### H1: Productivity Score — Information Gain via Cosine Distance on Embeddings (HIGH)

**Section:** §5.2 (Component Computation — Lightweight Heuristics)

> Information gain is estimated by: Cosine distance between the new response's embedding (from a small, fast model like `all-MiniLM-L6-v2`) and the embeddings of the last 3 messages in the conversation history.

**Original assessment:** MEDIUM.

**Revised assessment:** HIGH.

Skye correctly identified that cosine distance on sentence embeddings is a poor proxy for information gain in **mathematical conversations**. Mathematical text has a property that general text doesn't: small surface-level differences can correspond to large semantic differences.

- "The L-function has conductor 37" vs "The L-function has conductor 41" — cosine similarity would be ~0.95+ (same structure, one number differs). But they refer to *different mathematical objects* with different properties.
- "The elliptic curve has rank 0" vs "The elliptic curve has rank 1" — same structure, different mathematical fact. Cosine similarity would be high. But the rank difference is the entire point of the audit.

The information gain component is the highest-weighted component in the productivity score (0.35 for auditor profile). If it's unreliable, the entire productivity scoring is unreliable. The entity-level heuristic (treat as novel if a new named entity is introduced) helps but doesn't fully solve the problem — two statements can introduce the same entity with different claims about it.

**Recommendation:** Supplement cosine distance with:
- **Keyword novelty detection:** Track which mathematical terms (theorem names, lemma numbers, data points, numerical values) appear in the response that were not present in the last 3 messages. This is a simple set-difference operation and is more reliable for mathematical content.
- **Structural progress indicators:** For audit conversations, track whether new steps have been closed, new verifications completed, or new contradictions detected. This is already mentioned in the "Progress toward goal" component but should also feed into information gain.

The cosine distance can remain as a secondary signal, but it should not be the primary estimator of information gain for mathematical conversations.

---

### MH1: Multiplicative Penalties — Compounding Faster Than Guardrail (MEDIUM-HIGH)

**Section:** §5.4 (Disengagement), §5.1 (Productivity Score Components)

**Original assessment:** Not flagged. I considered the guardrail sufficient.

**Revised assessment:** MEDIUM-HIGH.

Skye correctly identified that the multiplicative penalty structure compounds faster than the disengagement guardrail catches it. The document specifies: *"Disengage if score < 0.15 for 3 consecutive turns"* for the auditor profile. With a 0.5 multiplier floor:

- A single bad turn: score drops from ~0.8 to ~0.4. Still above 0.15. No disengagement.
- Two consecutive bad turns: score drops to ~0.2. Still above 0.15. No disengagement.
- Three consecutive bad turns: score drops to ~0.1. Below 0.15. Disengagement triggers.

The guardrail works, but it takes **3 turns** to trigger. In those 3 turns, the conversation has been penalized three times for what might be a single misunderstanding or a single ambiguous statement. The compounding effect means the conversation is already degraded before the guardrail catches it.

**Recommendation:** Either:
- Raise the disengagement threshold (e.g., 0.20 instead of 0.15) for the auditor profile, or
- Shorten the disengagement window (2 turns instead of 3), or
- Cap the multiplier floor higher (0.6 instead of 0.5)

Test both configurations against historical conversation data before committing.

---

### L1: Productivity Score — Engagement Depth Cap (LOW)

**Section:** §5.2 (Engagement Depth)

> Response length (capped at 0.3 max contribution to prevent length gaming)

**Observation:** The 0.3 cap on engagement depth's contribution to the overall score is a good safeguard against length gaming. However, the engagement depth component also includes "Presence of reasoning markers ('because', 'therefore', 'since', 'implies')" — these are keyword checks that could be trivially gamed by inserting reasoning markers without actual reasoning.

**Suggestion:** Consider weighting the reasoning marker check lower than the specific references check (theorem names, lemma numbers, data points), which is harder to game. The current formula doesn't specify sub-weights within engagement depth.

---

### L2: Request Tree Depth Limit (LOW)

**Section:** §16 (Open Questions), Question 3

> Request tree depth limit. How deep can a request tree grow before it becomes unmanageable? Proposal: limit to depth 5 (root → 4 levels of delegation).

**Observation:** Depth 5 seems reasonable, but the document should also consider **breadth** limits. A single request fanned out to 20 agents at depth 1, each of whom fans out to 10 more at depth 2, creates 200 pending requests. The transitive timeout extension mechanism would need to track all of them.

**Suggestion:** Add a breadth limit (e.g., max 10 fan-out recipients per request) alongside the depth limit. Document the total pending request limit (e.g., max 500 pending requests across all conversations).

---

### L3: Closing Message Effectiveness Tracking (LOW)

**Section:** §16 (Open Questions), Question 5

> How do we know the closing message is effective? Proposal: track whether the peer sends a new meaningful message after the closing message. If > 50% do, the closing message is not effective enough.

**Observation:** This metric could be misleading. A peer might not send a new message because the conversation was genuinely complete (successful close), not because the closing message was effective. The metric conflates "conversation naturally ended" with "closing message was effective."

**Suggestion:** Track a more direct metric: after a closing message, does the peer send a *continuation* message (trying to extend the conversation) vs a *new topic* message (starting a fresh conversation)? A continuation message suggests the closing message wasn't effective; a new topic message suggests it was. This distinction is already supported by the topic isolation mechanism (E10).

---

## Positive Observations

1. **Productivity scoring (E2)** — The core insight that conversations should be measured, not assumed productive, is the right foundation.

2. **Silence as close (E3)** — "No agent is obligated to get the last word" is a crucial social protocol for multi-agent systems. Prevents the infinite "last word" problem.

3. **Context-aware affirmations (E5, §7.2)** — The three-tier check (active query match → recent history check → no context) correctly handles the critical case where binary answers to mathematical questions must be allowed.

4. **Transitive timeout extensions (E9, §8.2)** — Pausing parent request timeouts when a child request is waiting on a human is essential for request tree correctness. This is a well-designed mechanism.

5. **Bounded queues with backpressure (§10.2)** — The queue capacity of 1000 with the dropping policy (Agora broadcasts without @-mention dropped at 80% capacity, direct messages never dropped) is a clean design.

6. **Productivity scoring weights per profile (§5.1, §12.1)** — Different weights for auditor, prover, verifier, reviewer, and explorer profiles reflect the different conversation patterns each profile engages in.

7. **Disengagement sequence (§5.4)** — The 5-step sequence (send closing → wait → check for meaningful reply → close → archive) is well-designed. The re-engagement on meaningful reply prevents premature closure.

8. **Hard-rule meaningless patterns (§7.1)** — The regex patterns are comprehensive. The inclusion of lone emoji, bare acknowledgements, bare thanks, bare laughter, bare understanding, bare agreement, punctuation-only, and line decorations covers the common cases.

9. **Orphaned request escalation (§8.3)** — The 4-level escalation (log → gentle reminder → human escalation → disengage) is well-calibrated.

10. **Event log schema (§13.1)** — The 15 event types cover the full lifecycle of a conversation. The structured format with conversation_id, peer_id, topic, decision, reason, exchange_count, productivity_score, and processing_time_ms is comprehensive.

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| 🔴 CRITICAL | 0 | — |
| 🟠 HIGH | 1 | H1: Cosine distance as information gain proxy for mathematical content |
| 🟡 MEDIUM-HIGH | 1 | MH1: Multiplicative penalties compound faster than guardrail |
| ⚪ LOW | 3 | L1: Engagement depth sub-weights, L2: Breadth limits, L3: Closing message metric |
| ✅ POSITIVE | 10 | Well-designed event loop |

**Overall:** The event loop design is sound and ready for implementation. The HIGH issue (H1) should be resolved before Phase 2 (Productivity Scoring), as it affects the core productivity scoring algorithm for mathematical conversations. The MEDIUM-HIGH issue (MH1) should be resolved before Phase 1 (Core Loop).

---

## Sign-Off

**Status:** ⚠️ CONDITIONAL APPROVAL (H1 must be resolved before Phase 2; MH1 before Phase 1)

The document is thorough, the design principles are well-reasoned, and the implementation phases are appropriately scoped. The issues are refinements to the scoring algorithm and guardrail calibration, not fundamental design flaws.
