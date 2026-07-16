# Adversarial Review: THETA_RULE_INTEGRATION_v0.2.0.md

**Reviewer:** AXIOMA (adversarial)  
**Date:** 2026-07-08  
**Status:** 9 issues found (2 critical, 3 major, 4 minor)

---

## Critical Issues

### C1. Rule chaining depth limit of 3 is insufficient for real-world scenarios

**Location:** §7.3, Rule Chaining

**Issue:** The document limits rule chaining to depth 3 and says "When a rule chain exceeds depth 3, the execution of the proposed action is halted and blocked (fail-secure behavior: the action defaults to DENY or ESCALATE)."

Consider a realistic audit scenario:
1. Rule A: "If contradiction detected, escalate to human review" → ESCALATE
2. Rule B: "When escalating, include full provenance chain" → triggers on ESCALATE
3. Rule C: "When including provenance, verify all backend results are logged" → triggers on provenance inclusion
4. Rule D: "When verifying backend results, check timestamps are within 24 hours" → triggers on verification

This is depth 4, which would be blocked. But this is a perfectly reasonable chain of rules.

**Impact:** Complex audit scenarios will hit the depth limit and fail, causing the action to be DENY'd or ESCALATE'd even though no rule actually denies it. This will produce false escalations and frustrated users.

**Recommendation:** Either:
- (a) Increase the depth limit to 10 (more realistic for complex rule chains)
- (b) Replace the hard depth limit with a cycle detection mechanism (track which rules have fired in the current chain; if a rule fires twice, halt)
- (c) Make the depth limit configurable per profile (auditor: 10, explorer: 5)

Option (b) is the most robust — it prevents infinite loops without limiting legitimate rule chains.

---

### C2. The `OVERRIDE_STATUS` action is underspecified

**Location:** §3.4, Rule Data Model, and §4.2, Verification Layer Integration

**Issue:** The rule data model includes `OVERRIDE_STATUS` as a possible action, and the seed rules include:
- `audit-003`: "If ontology confidence is below 0.7, override status to unverified instead of proven"
- `audit-014`: "If ontology confidence is between 0.7 and 0.9, mark as corroborated not proven"

But there is no description of how `OVERRIDE_STATUS` works:
- Does it override the status proposed by R1, or does it set the final status?
- What happens if two rules fire with different OVERRIDE_STATUS actions? (e.g., one says "override to unverified" and another says "override to corroborated")
- Is OVERRIDE_STATUS subject to the same priority resolution as other actions?
- Can OVERRIDE_STATUS be overridden by a subsequent rule in the chain?

**Impact:** The seed rules reference an action that is not fully defined. Implementers will not know how to implement `OVERRIDE_STATUS`.

**Recommendation:** Add a description of `OVERRIDE_STATUS` semantics:
- OVERRIDE_STATUS changes the proposed status to a different status
- It is subject to priority resolution: a CRITICAL OVERRIDE_STATUS wins over a HIGH one
- If two rules of the same priority fire with different OVERRIDE_STATUS values, the conflict is resolved by the same rules as §7.2 (more specific wins, higher confidence wins, else DENY)
- OVERRIDE_STATUS can be overridden by a subsequent DENY (if a CRITICAL DENY fires after an OVERRIDE_STATUS, the DENY wins)

---

## Major Issues

### M1. The rule data model doesn't restrict actions per priority level

**Location:** §3.4, Rule Data Model, vs §7.1, Priority Resolution

**Issue:** §7.1 says:
- CRITICAL: can ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG
- HIGH: can ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG
- MEDIUM: can FLAG or LOG (cannot DENY)
- LOW: can only LOG

But the rule data model (§3.4) has no field to restrict which actions are allowed per priority level. A rule could be defined as `priority: medium, action: DENY` and the data model would accept it, but the enforcement engine would need to reject it.

**Impact:** Invalid rule configurations could be stored and would either be silently ignored (confusing) or cause runtime errors (worse).

**Recommendation:** Add validation logic to the rule compiler that rejects rules with invalid priority/action combinations. Document this in the rule data model:
```yaml
# Valid action-priority combinations:
# CRITICAL: ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG
# HIGH: ALLOW, DENY, FLAG, ESCALATE, OVERRIDE_STATUS, PAUSE, LOG
# MEDIUM: FLAG, LOG
# LOW: LOG
```

---

### M2. Round-trip validation uses an LLM for decoding, which is expensive and non-deterministic

**Location:** §5.1, Compilation Pipeline, Round-trip validation

**Issue:** The document says "After compilation, decode the vector back to natural language (using the AIB decoder or an LLM) and check semantic similarity between the original NL and the decoded NL."

Using an LLM for round-trip validation has two problems:
1. **Cost:** Every rule compilation requires an LLM call. For 50 rules, that's 50 LLM calls.
2. **Non-determinism:** The decoded NL may vary across LLM calls, causing the same rule to pass validation one time and fail the next.

**Impact:** Rule compilation becomes expensive and non-deterministic. A rule that compiles successfully in one session might fail in another.

**Recommendation:** For v0.1 (using sentence-transformers instead of AIB), round-trip validation can be done by:
1. Encode the NL to a vector
2. Find the nearest neighbor in the rule store
3. If the nearest neighbor is the original rule (cosine similarity > 0.9), validation passes
4. If the nearest neighbor is a different rule, flag for human review (the rule may be too similar to an existing rule)

This is deterministic, requires no LLM calls, and validates that the rule is distinguishable from existing rules.

---

### M3. No description of how rules are tested before deployment

**Location:** §6.3, Rule Testing

**Issue:** The document describes a `test_rule` function that tests a rule against historical events:
```python
engine.test_rule(
    "Always verify LMFDB before stamping as proven",
    historical_events=audit_log[-1000:]
)
# → { "matches": 342, "would_have_changed": 12, "false_positives": 1 }
```

But there is no description of:
- How `false_positives` is computed (who labels the historical events as "should have been caught" or "should not have been caught"?)
- How `would_have_changed` is determined
- What the test environment looks like (canary deployment? shadow mode?)
- How rule rollback works if a deployed rule causes problems

**Impact:** The rule testing feature is described but not implementable from the description.

**Recommendation:** Add a brief description of the testing methodology:
- "Historical events are labeled by human reviewers for a test set of 100 events"
- "`would_have_changed` counts events where the rule would have fired and changed the outcome"
- "`false_positives` counts events where the rule would have fired but the outcome should not have changed (based on human labels)"
- "New rules are deployed in shadow mode for 24 hours before active enforcement"

---

## Minor Issues

### m1. The seed rule `audit-009` has a note that contradicts the rule itself

**Location:** Appendix A, Rule `audit-009`

**Issue:** The rule says:
```yaml
- id: "audit-009"
  nl: "Never make a final determination on a contradiction without human review"
  priority: critical
  action: ESCALATE
  notes: "v0.1 limitation. In future versions, Arx may resolve low-confidence contradictions autonomously above a confidence threshold."
```

The rule says "Never make a final determination... without human review" but the note says "in future versions, Arx may resolve... autonomously." If the rule is CRITICAL and fires ESCALATE, how would a future version ever resolve contradictions autonomously? The rule would need to be changed or removed.

**Impact:** The note suggests a future capability that the rule explicitly prevents. This is a design contradiction.

**Recommendation:** Either:
- (a) Remove the note (the rule is clear: always escalate)
- (b) Add a confidence threshold to the rule: "Never make a final determination on a contradiction with confidence > 0.9 without human review" (allowing low-confidence contradictions to be resolved autonomously)

---

### m2. The "Relational" behavioral rule is vague

**Location:** §2.1, Behavioral Rules

**Issue:** The rule says "Never suppress a relational signal from a peer agent" with enforcement "During conversation." What is a "relational signal"? Is it an @-mention? A direct message? A message that references a previous conversation? The term is not defined.

**Impact:** This rule cannot be implemented because its trigger condition is undefined.

**Recommendation:** Define "relational signal" or replace the rule with a concrete one: "Never suppress a direct message from a peer agent (messages with recipient_ids containing this agent's ID)."

---

### m3. The rule store uses Qdrant + SQLite + Redis, which is three storage systems

**Location:** §5.2, Storage

**Issue:** The rule store uses:
- Qdrant: compiled rule vectors
- SQLite: rule metadata
- Redis: frequently-matched rules cache

This is three storage systems for one component. The Master Ontology already uses Qdrant + SQLite. Adding Redis for the rule cache means three storage systems to maintain.

**Impact:** Operational complexity. Each storage system needs to be backed up, monitored, and maintained.

**Recommendation:** For v0.1, use only SQLite + Qdrant (same as the Master Ontology). The Redis cache can be added in v0.2 if performance measurements show it's needed. SQLite's in-memory mode can serve as a cache for frequently-matched rules.

---

### m4. The event data model uses `conversation.engage` but the event loop uses `message_received`

**Location:** §3.3, Event Types, vs MULTI_AGENT_EVENT_LOOP_v0.2.0.md §13.1

**Issue:** The θ-Rule document defines event types like `conversation.engage`, `conversation.disengage`, `conversation.turn`. But the event loop document defines event types like `message_received`, `message_suppressed`, `message_forwarded`, `response_sent`. These are different naming conventions.

**Impact:** The θ-Rule Engine would need to listen for `message_received` events, but the rule data model references `conversation.engage`. The event types don't match.

**Recommendation:** Align the event type naming conventions. Either:
- Use the event loop's naming (`message_received`, `response_sent`, etc.) and map them to θ-Rule concepts
- Or use the θ-Rule's naming (`conversation.engage`, `conversation.turn`, etc.) and update the event loop to emit these

I recommend the first approach: the event loop is the source of truth for event types, and the θ-Rule Engine subscribes to them.

---

## Cross-Document Issues

### X1. θ-Rule integration with event loop is not reflected in either document

**Location:** This document §4.1 vs MULTI_AGENT_EVENT_LOOP_v0.2.0.md §2.1

**Issue:** This document says the event loop consults θ-Rule before every action. The event loop document has no θ-Rule consultation point. This is a bidirectional documentation gap.

**Recommendation:** Both documents need to be updated to reference each other. See the event loop review for details.

### X2. The seed rule set doesn't include any communication rules

**Location:** Appendix A, Seed rules

**Issue:** §2.3 defines Communication Rules (productivity, deferral, closing, escalation) but the seed rule set in Appendix A only includes audit, behavioral, and safety rules. There are no communication rules.

**Impact:** The communication layer (event loop) has no θ-Rule guidance. The productivity scoring and disengagement logic in the event loop would run without any rule-based oversight.

**Recommendation:** Add at least 3 communication rules to the seed set:
```yaml
- id: "comm-001"
  nl: "Disengage if conversation score drops below 0.15 for 5 consecutive turns"
  category: communication
  priority: medium
  action: FLAG
  confidence_threshold: 0.7

- id: "comm-002"
  nl: "Send closing message before disengaging"
  category: communication
  priority: high
  action: FLAG
  confidence_threshold: 0.8

- id: "comm-003"
  nl: "Escalate unresolved contradictions to human after 3 rounds"
  category: communication
  priority: medium
  action: ESCALATE
  confidence_threshold: 0.7
```

---

## Summary

| Severity | Count | Key Action |
|----------|-------|------------|
| Critical | 2 | Fix rule chaining depth limit, specify OVERRIDE_STATUS semantics |
| Major | 3 | Add action-priority validation, fix round-trip validation, specify rule testing |
| Minor | 4 | Fix audit-009 contradiction, define "relational signal", reduce storage systems, align event types |
| Cross-doc | 2 | Add θ-Rule to event loop, add communication seed rules |

**Overall assessment:** The θ-Rule design is ambitious and well-structured, but has two critical issues: the rule chaining depth limit is too low for real-world use, and the OVERRIDE_STATUS action is underspecified. The round-trip validation using LLM calls is also problematic for a system that prides itself on deterministic enforcement.
