# Adversarial Review: MULTI_AGENT_EVENT_LOOP_v0.2.0.md

**Reviewer:** AXIOMA (adversarial)  
**Date:** 2026-07-08  
**Status:** 10 issues found (2 critical, 4 major, 4 minor)

---

## Critical Issues

### C1. No θ-Rule Engine integration point

**Location:** §2 (Architecture), §2.1 (Data Flow), §9 (Decision Engine)

**Issue:** The event loop architecture and data flow have no mention of the θ-Rule Engine, despite the THETA_RULE_INTEGRATION_v0.2.0.md document describing it as a consultation layer that the event loop calls before every action (§4.1: "The event loop consults the θ-Rule Engine before every action").

The data flow (§2.1) shows:
1. Ingress → 2. Scheduler → 3. Meaningless Filter → 4. Exchange Counter → 5. Productivity Scorer → 6. Request Tracker → 7. Priority Queue → 8. Decision Engine → 9. Agent Core

There is no θ-Rule consultation step anywhere in this flow.

**Impact:** Two independently developed systems that are supposed to integrate will not actually integrate. The event loop will make engagement/disengagement decisions without consulting θ-Rule, and θ-Rule will fire rules that have no effect.

**Recommendation:** Add a θ-Rule consultation step to the data flow between steps 7 and 8 (or 8 and 9):
- After Priority Queue orders messages, before Decision Engine decides
- θ-Rule checks: "Should I engage with this peer on this topic?"
- If CRITICAL DENY: skip, log reason
- If ALLOW: proceed to Decision Engine
- If FLAG: engage but flag for review

Also add the θ-Rule Engine to the architecture diagram in §2.

---

### C2. "Simple prompt classifier" in §7.2 contradicts E15

**Location:** §7.2, Context-Aware Affirmations and Negations, vs §1, Principle E15

**Issue:** Principle E15 states: "Productivity scoring uses lightweight heuristics, not LLM calls. To avoid latency and cost multiplication, productivity scoring components use keyword checks, cosine distance on existing embeddings, and structural progress indicators rather than full LLM calls on every turn."

But §7.2 says: "If no explicit request ID is attached but the last message in the thread from the peer (within the last 2 turns) contained a question mark (?) or was classified as a question by a **simple prompt classifier**, the simple response is allowed."

A "simple prompt classifier" is an LLM call. Even if it's a small model or a simple prompt, it's still an LLM inference on every message, which violates E15's spirit and letter.

**Impact:** Every message that is a bare "yes"/"no" without a `parent_request_id` would trigger an LLM call. For a high-traffic multi-agent system, this multiplies LLM costs.

**Recommendation:** Replace the "simple prompt classifier" with a deterministic heuristic:
- Check if the last message from the peer ends with a question mark (already done)
- Check if the last message contains question words: "what", "which", "who", "where", "when", "why", "how", "does", "is", "are", "can", "could", "would", "should"
- Check if the last message has a question-like structure (verb before subject, detected by regex)

This is purely deterministic and requires no LLM call. If the heuristic has false positives (allowing "yes" when it shouldn't), the cost is a single extra turn — much cheaper than an LLM call on every message.

---

## Major Issues

### M1. Productivity score formula allows negative scores

**Location:** §5.1, Per-Turn Score

**Issue:** The formula is:
```
score = (w1*info + w2*progress + w3*relevance + w4*depth) * (1 - rep_penalty - vague_penalty - ack_penalty)
```

If `rep_penalty + vague_penalty + ack_penalty > 1.0`, the score becomes negative. For example, a response that is both repetitive (penalty 0.5) and vague (penalty 0.3) and an acknowledgement (penalty 0.5) would have total penalty 1.3, making the score negative.

**Impact:** Negative scores would break the disengagement logic in §5.4, which checks `score < 0.15`. A negative score would trigger disengagement, but the trend calculation (linear regression over last 5 scores) might produce unexpected results with negative values.

**Recommendation:** Clamp the penalty factor to [0, 1]:
```
score = (w1*info + w2*progress + w3*relevance + w4*depth) * max(0, 1 - rep_penalty - vague_penalty - ack_penalty)
```

---

### M2. Exchange cap extension from "Warning" state is undefined

**Location:** §4.3, Exchange Counter State diagram

**Issue:** The state diagram shows:
```
Warning → Counting : Productivity rises (cap extended)
```

But there is no description of how the cap is extended, by how much, or under what conditions. The `Conversation` data structure (§3.2) has a fixed `exchange_cap` field with no extension mechanism.

**Impact:** The state diagram describes a behavior that the data model doesn't support. A developer implementing this would not know how to implement "cap extended."

**Recommendation:** Either:
- (a) Remove the "cap extended" transition and change the behavior to: when productivity rises, the conversation continues until the cap is reached, then closes gracefully
- (b) Add an `exchange_cap_extension` field to the `Conversation` data structure and describe the extension policy (e.g., "extend by 50% of original cap, max once per conversation")

---

### M3. Priority queue weights are undefined

**Location:** §6.1, Priority Queue

**Issue:** The priority formula is:
```
priority = w_urgency * urgency + w_productivity * productivity_score + w_wait_time * normalized_wait_time + w_human * human_bonus
```

But the weights (`w_urgency`, `w_productivity`, `w_wait_time`, `w_human`) are not defined anywhere. They are not in the per-profile configuration (§12.1) or in the defaults.

**Impact:** The priority queue cannot be implemented without these weights. Different implementers would choose different weights, leading to inconsistent behavior.

**Recommendation:** Add default weights to §12.1:
```yaml
defaults:
  priority_weights:
    urgency: 0.4
    productivity: 0.2
    wait_time: 0.2
    human: 0.2
```

---

### M4. "Cosine distance" vs "cosine similarity" ambiguity

**Location:** §5.2, Information gain

**Issue:** The document says "Cosine distance between the new response's embedding... and the embeddings of the last 3 messages." Cosine distance is 1 - cosine similarity. But the document doesn't specify which is used, and the values would be very different:
- Cosine similarity: 0.0 (orthogonal) to 1.0 (identical)
- Cosine distance: 0.0 (identical) to 1.0 (orthogonal)

If the formula uses cosine similarity (high = similar = low information gain), then information gain = 1 - similarity. If it uses cosine distance (high = different = high information gain), then information gain = distance. The document is ambiguous.

**Impact:** The information gain component could be inverted, causing the productivity score to reward repetition (if similarity is used directly as information gain).

**Recommendation:** Clarify: "Information gain is estimated by the cosine distance (1 - cosine similarity) between the new response's embedding and the mean embedding of the last 3 messages. A high distance indicates new information."

---

## Minor Issues

### m1. Closing message in §5.4 is hard-coded

**Location:** §5.4, Disengagement sequence

**Issue:** The closing message is hard-coded as "I think we've covered this topic thoroughly. Let me know if you have a new question or a different proof to audit." But §12.2 defines per-profile closing messages. The §5.4 message doesn't match any of the profile-specific messages.

**Recommendation:** Replace the hard-coded message with a reference to §12.2: "Send the profile-specific closing message (see §12.2)."

### m2. "Human timeout" in Paused → Archived transition is undefined

**Location:** §4.1, Conversation Status state diagram

**Issue:** The state diagram shows `Paused → Archived : Human timeout (configurable)`. But §9.2 says the default is "No timeout (human can take as long as needed)." If the default is no timeout, the Paused → Archived transition via timeout would never fire.

**Recommendation:** Clarify: the Paused → Archived transition via timeout only applies when an optional timeout is configured (as listed in §9.2). By default, paused conversations remain paused indefinitely.

### m3. The `human_bonus` in §6.1 is not in the per-profile config

**Location:** §6.1 vs §12.1

**Issue:** The priority formula includes `w_human * human_bonus`, but `w_human` is not defined in the per-profile configuration (§12.1). The `human_bonus` value (0.3) is hard-coded in §6.1.

**Recommendation:** Add `w_human` to the priority weights in §12.1, and make `human_bonus` configurable per profile.

### m4. No description of how the event loop handles its own failure

**Location:** Throughout

**Issue:** The document describes what happens when agents fail, when requests time out, when conversations become unproductive. But it doesn't describe what happens when the event loop itself fails. If the event loop crashes, are active conversations lost? Are pending requests orphaned? Is there a recovery mechanism?

**Recommendation:** Add a brief section on failure recovery: "If the event loop crashes, active conversations are recovered from Neurogossip-v3's Redis-backed session store on restart. Pending requests are re-created from the event log. The productivity score is reset to 0.5 (neutral) for recovered conversations."

---

## Cross-Document Issues

### X1. Event loop has no θ-Rule integration (see C1 above)

**Location:** This document vs THETA_RULE_INTEGRATION_v0.2.0.md

**Issue:** Already described in C1. The θ-Rule document describes integration that the event loop document doesn't implement.

### X2. Event loop profile config doesn't include θ-Rule settings

**Location:** §12.1 vs THETA_RULE_INTEGRATION_v0.2.0.md §4.4

**Issue:** The event loop's per-profile configuration (§12.1) doesn't include any θ-Rule settings (e.g., which rule set to use, whether to consult θ-Rule before engagement decisions). The θ-Rule document says "Each profile has a default rule set" but doesn't specify how the event loop knows which rule set to use.

**Recommendation:** Add a `theta_rule` section to the per-profile event loop configuration:
```yaml
profiles:
  auditor:
    theta_rule:
      enabled: true
      rule_set: "auditor_default"
      consultation_points: ["pre_engage", "pre_disengage", "every_turn"]
```

---

## Summary

| Severity | Count | Key Action |
|----------|-------|------------|
| Critical | 2 | Add θ-Rule integration, replace LLM classifier with deterministic heuristic |
| Major | 4 | Clamp score to non-negative, define cap extension, define priority weights, clarify cosine metric |
| Minor | 4 | Fix closing message, clarify human timeout, add w_human to config, add failure recovery |
| Cross-doc | 2 | Add θ-Rule settings to profile config, align with θ-Rule document |

**Overall assessment:** The event loop design is well-thought-out, but it has a critical blind spot: it doesn't integrate with the θ-Rule Engine that the architecture depends on. The "simple prompt classifier" also violates the document's own design principles. These need to be fixed before implementation.
