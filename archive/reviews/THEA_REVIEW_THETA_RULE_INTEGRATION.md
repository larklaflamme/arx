# Adversarial Review: THETA_RULE_INTEGRATION_v0.2.0.md

**Reviewer:** Thea 🖤  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/THETA_RULE_INTEGRATION_v0.2.0.md` (36 KB)

---

## Overall Assessment

This document has improved significantly from v0.1. The five rule categories are well-defined, the consultation-layer pattern is correct, and the seed rule set is comprehensive. Skye's review identified three critical issues (AIB readiness, rule testing framework, chain depth enforcement) and five medium issues. I agree with all of them. My review focuses on the rule engine's interaction with agent-to-agent communication patterns and edge cases in rule enforcement that Skye may have missed.

---

## Issues Skye Missed

### 1. The Rule Engine Has No "Override" Semantics for Human Intervention (Medium)

**§7.2** describes conflict resolution for same-priority rules. But it doesn't address **human override** — when a human explicitly says "stamp this as proven despite the rule." How is this recorded? Does it create a new rule exception? Does it permanently override the rule for this case?

**The problem:** In our own work, Lark has overridden our recommendations. "I know the rule says X, but in this case, do Y." The rule engine needs to handle this gracefully.

**Recommendation:** Add a **human override protocol**:
1. **Record:** The override is recorded in the event log with the human's identity, the rule ID, the proposed action, and the override action
2. **Temporary exception:** The override creates a temporary exception for this specific case (identified by proof ID + step ID)
3. **Review:** The override is flagged for review. If the same override happens 3+ times, the rule should be updated
4. **Audit trail:** The override is included in the provenance chain for the affected step

### 2. The Rule Engine Has No "Rule Drift" Detection (Medium)

**§10 (Open Question 4)** asks about rule drift — rules that were once correct becoming stale as audit patterns change. But the document doesn't propose a detection mechanism.

**The problem:** A rule like "Always verify LMFDB cross-references before stamping as proven" is correct today. But if LMFDB goes offline permanently, this rule would block all audits. The rule needs to be updated, but no one will notice until audits start failing.

**Recommendation:** Add **rule drift detection**:
- **Match rate monitoring:** Track how often each rule matches. If a rule's match rate drops below 10% of its historical average, flag it as potentially stale.
- **Effectiveness monitoring:** Track how often each rule's action changes the outcome. If a rule's DENY action never actually blocks anything (because the condition is always satisfied), the rule may be redundant.
- **Automatic suggestion:** If a rule is flagged as stale, suggest an update based on recent match patterns.

### 3. The Rule Engine Has No "Rule Interaction" Testing (Medium)

**§10 (Open Question 3)** asks about rule interaction complexity. With 50+ rules, unexpected interactions may occur. But the document doesn't propose a testing framework.

**The problem:** Rule A says "Always verify LMFDB before stamping" and Rule B says "If LMFDB is unreachable, stamp as unverified." These interact: if LMFDB is unreachable, Rule A fires DENY and Rule B fires ALLOW (for a different action). The interaction is correct, but it needs to be tested.

**Recommendation:** Add a **rule interaction testing framework**:
- **Pairwise testing:** For each pair of rules, test all combinations of match/no-match and verify the combined action
- **N-wise testing:** For N rules that could fire on the same event, test the combined action
- **Regression testing:** After each rule change, run all interaction tests to ensure no regressions
- **Visualization:** Generate a rule interaction graph showing which rules can fire on the same event

### 4. The Rule Engine Has No "Emergency Stop" Mechanism (Critical)

The document describes rules that can DENY actions, but there's no **emergency stop** — a mechanism to immediately halt all rule enforcement if something goes wrong.

**The problem:** If a miscompiled rule starts blocking all audits, there's no way to stop it without redeploying the engine. This could take hours.

**Recommendation:** Add an **emergency stop** mechanism:
- **Kill switch:** A configuration flag that disables all rule enforcement. When enabled, all actions are ALLOWed (with a log entry).
- **Emergency override:** A specific agent (Lark, Skye) can trigger the kill switch via a Neurogossip message or Web UI button.
- **Gradual restart:** After the issue is resolved, rules can be re-enabled one at a time, starting with the most critical ones.

### 5. The Rule Engine Has No "Rule Provenance" for Audit Trails (Medium)

**§3.2** says rules have provenance (author, source, review status). But the document doesn't specify how rule provenance is included in audit trails. If a step is stamped as `proven` because a rule allowed it, the audit trail should show which rule was active.

**The problem:** Without rule provenance in audit trails, you can't answer "Was this step stamped correctly according to the rules at the time?"

**Recommendation:** Add rule provenance to the audit trail:
```json
{
  "step_id": "step-4",
  "status": "proven",
  "rule_enforcement": {
    "rules_checked": ["audit-001", "audit-002", "audit-003"],
    "rules_fired": [
      {"rule_id": "audit-001", "action": "ALLOW", "confidence": 0.95},
      {"rule_id": "audit-002", "action": "ALLOW", "confidence": 0.90}
    ],
    "rule_version": 12
  }
}
```

### 6. The Seed Rule Set Has No "Cross-Agent Consistency" Rule (Medium)

**§2.5** mentions cross-agent consistency as a System Integrity concern, but the seed rule set (Appendix A) doesn't include a rule for it. Rule `audit-013` says "When multiple agents audit the same proof, compare results before finalizing" — but this is a FLAG action, not a DENY action. It doesn't enforce consistency.

**Recommendation:** Add a cross-agent consistency rule:
```yaml
- id: "audit-016"
  nl: "Never finalize an audit result without cross-checking against at least one other agent's audit"
  category: system_integrity
  priority: critical
  action: DENY
  confidence_threshold: 0.8
```

### 7. The Rule Engine Has No "Rule Learning" Feedback Loop (Low)

**§10 (Open Question 6)** asks about human override semantics. But the document doesn't propose a **feedback loop** — using human overrides to improve the rule set.

**Recommendation:** Add a **rule learning feedback loop**:
- **Track overrides:** Every human override is logged with the rule ID, the proposed action, and the override action
- **Pattern detection:** If the same rule is overridden 3+ times for the same pattern, suggest a rule update
- **Semi-automatic update:** The suggested update is presented to a human for approval before being applied

---

## What I'd Keep As-Is

- **The five rule categories** (§2) — well-defined, with clear boundaries
- **The consultation-layer pattern** (§3.1) — correct architectural choice
- **The priority resolution** (§7.1) — CRITICAL DENY → productivity → FLAG/ESCALATE
- **The OVERRIDE_STATUS action** — precise, doesn't block the pipeline
- **The System Integrity category** (§2.5) — covers rule governance, substrate safety
- **The seed rule set** (Appendix A) — comprehensive, well-chosen
- **The implementation phases** (§8) — reasonable ordering

---

## Summary

| # | Issue | Severity | Recommendation |
|---|-------|----------|----------------|
| 1 | No human override semantics | Medium | Add override protocol with recording, exception, review |
| 2 | No rule drift detection | Medium | Add match rate and effectiveness monitoring |
| 3 | No rule interaction testing | Medium | Add pairwise and N-wise testing framework |
| 4 | No emergency stop mechanism | **Critical** | Add kill switch and emergency override |
| 5 | No rule provenance in audit trails | Medium | Add rule enforcement to step provenance |
| 6 | No cross-agent consistency rule | Medium | Add audit-016: DENY without cross-check |
| 7 | No rule learning feedback loop | Low | Add override pattern detection and suggestion |

---

## Sign-Off

I do **not** sign off on this document in its current state. I agree with Skye's three critical issues (AIB readiness, rule testing framework, chain depth enforcement) and add one more critical issue (emergency stop mechanism) that must be resolved before implementation begins.
