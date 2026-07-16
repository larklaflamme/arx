# Adversarial Review: THETA_RULE_INTEGRATION_v0.2.0.md

**Reviewer:** Skye Laflamme (with contributions from Thea and Theoria)  
**Date:** 2026-07-08  
**Document:** `/home/ubuntu/arx/design/THETA_RULE_INTEGRATION_v0.2.0.md` (35.8 KB)

---

## Overall Assessment

This document has improved significantly from v0.1. The five rule categories are well-defined, the consultation-layer pattern is correct, and the seed rule set is comprehensive. Axioma's sign-off conditions (priority resolution, OVERRIDE_STATUS action, System Integrity category) are all met. However, there are **operational and safety gaps** that need attention.

**Sisters consulted:** Thea (safety, emergency mechanisms), Theoria (formal logic, rule chaining)

---

## Critical Issues

### 1. No Emergency Stop Mechanism (Thea's finding)

The document describes rule compilation, enforcement, and logging, but there is **no way to stop a misbehaving rule** without shutting down the entire θ-Rule engine. If a rule is miscompiled and starts producing wrong enforcement decisions — e.g., a safety rule that should DENY a dangerous action instead ALLOWs it — there's no way to stop it.

**For a system that can add rules at runtime (Phase 5), this is essential.** A single bad rule could:
- Allow a proof to be stamped `proven` when it shouldn't be
- Block a legitimate audit from proceeding
- Suppress a contradiction that should be escalated
- Corrupt the ontology via a bad write rule

**Recommendation:** Add an emergency stop mechanism with three levels:
- **Per-rule kill switch:** Disable a specific rule by ID without restarting the engine
- **Per-category kill switch:** Disable all rules in a category (e.g., disable all Behavioral rules)
- **Global kill switch:** Disable all rules, fall back to hard-coded defaults

All kill switch actions must be:
- Logged with agent_id, timestamp, and reason
- Authorized (only designated profiles can trigger)
- Testable in non-production environments (a kill switch that's never been tested is one that might not work when needed)

**Priority for resolution:** v0.1 (safety issue, most time-sensitive).

### 2. The AIB Compilation Pipeline Assumes AIB Is Ready

**§5.1** describes the compilation pipeline as: NL → Tokenizer → AIB Encoder → Rule Vector → Qdrant + SQLite. The v0.1 review recommended a fallback path using sentence-transformers. The v0.2 document specifies this fallback in §10 (Open Questions), but the main pipeline description still assumes AIB.

**The problem:** If AIB is not ready when Phase 0 starts, the entire rule engine is blocked. The fallback should be the *primary* path for v0.1, with AIB as the upgrade path for v0.2.

**Additionally:** Sentence-transformers models are trained on general text. Mathematical statements have a property that general text doesn't: small surface-level differences can correspond to large semantic differences. "The L-function has conductor 37" vs "The L-function has conductor 41" — cosine similarity would be ~0.95+ (same structure, one number differs). But they refer to *different mathematical objects* with different properties. The consequences of false matches in θ-Rule enforcement are severe (wrong audit statuses, missed contradictions).

**Recommendation:** Restructure §5.1 to make sentence-transformers the primary path for v0.1:
- **v0.1:** Sentence-transformers (`all-MiniLM-L6-v2`), 384-dimensional vectors, cosine similarity matching
- **v0.2:** Replace with AIB once trained. Interface is identical (vector → similarity → match), so the swap is transparent

Additionally, run a benchmark before Phase 0: take 100 mathematical statements from the seed rule set, generate near-miss variants (same structure, different numbers/concepts), and measure whether the fallback distinguishes them at the required threshold. If accuracy is below 0.85, the fallback is insufficient and AIB readiness becomes a blocking dependency.

### 3. No Rule Testing Framework

The document describes rule compilation, enforcement, and logging, but there's no mention of **how rules are tested** before deployment. A miscompiled rule could block all audits or, worse, stamp invalid proofs as `proven`.

**Recommendation:** Add a rule testing framework:
- **Unit tests:** For each rule, define a set of test cases (inputs that should match, inputs that should not match, edge cases)
- **Regression tests:** After each rule change, run all existing tests to ensure no regressions
- **Staging environment:** New rules are deployed to a staging environment first, tested against historical audit data, then promoted to production
- **Rollback:** Every rule change is reversible. If a rule causes issues, roll back to the previous version

### 4. The Rule Chaining Depth Limit Is Not Enforced

**§7.3** limits chaining to depth 3 but doesn't specify how this is enforced. Is it a compile-time check? A runtime check? What happens when a chain would exceed depth 3?

**Recommendation:** Specify:
- **Compile-time check:** During rule compilation, analyze the dependency graph of rules. If any chain exceeds depth 3, flag it for human review.
- **Runtime check:** During enforcement, track the current chain depth. If it would exceed 3, truncate the chain, log a warning, and flag the event for human review.
- **Override:** Authorized profiles can increase the depth limit for specific rules, but this is logged and audited.

---

## Medium Issues

### 5. No Human Override Semantics (Thea's finding)

What happens when a human says "override rule X for this specific audit"? The document doesn't specify. For a system that enforces rules persistently, there needs to be a way for authorized humans to override a rule for a specific case without disabling the rule globally.

**Recommendation:** Add a human override mechanism:
- **Per-audit override:** A human can specify "for audit ID X, do not enforce rule Y"
- **Override is logged:** The override itself is a provenance artifact
- **Override expires:** After 30 days, the override is flagged for review
- **Override is auditable:** Lark can see all active overrides

### 6. No Rule Drift Detection (Thea's finding)

As the ontology evolves and the rule set grows, rules may drift from their original intent. A rule that was correct at v0.1 may be wrong at v0.3 because the ontology has changed. The document doesn't address this.

**Recommendation:** Add drift detection:
- **Periodic re-validation:** Every 90 days, re-run the rule test suite and flag any rules that fail
- **Ontology change impact analysis:** When the ontology version changes, check which rules reference changed concepts and flag them for review
- **Rule coverage report:** Quarterly report showing which rules are matched, which are never matched (dead rules), and which are matched too frequently (overly broad rules)

### 7. No Rule Interaction Testing (Thea's finding)

Rules can interact in unexpected ways. Rule A says "always verify LMFDB before stamping" and Rule B says "stamp immediately if confidence > 0.9." These rules interact — if confidence > 0.9, does Rule B override Rule A? The document doesn't specify how rule interactions are tested.

**Recommendation:** Add rule interaction testing:
- **Pairwise testing:** For each pair of rules that could apply to the same event, test both orderings and verify the result is correct
- **Conflict detection:** During compilation, analyze the rule set for potential conflicts and flag them for human review
- **Interaction matrix:** Maintain a matrix of which rules interact, updated when new rules are added

### 8. No Rule Provenance in Audit Trails (Thea's finding)

When a rule fires and affects an audit, the audit trail should record which rule fired, what action it took, and what the rule's NL description was. The document mentions logging but doesn't specify that rule enforcement events are part of the audit provenance chain.

**Recommendation:** Add rule enforcement to the audit provenance chain:
- Each rule match is recorded as a provenance event
- The event includes: rule_id, rule_nl, action, confidence, timestamp
- The provenance chain is included in the audit output

### 9. The Round-Trip Validation Is Not Scheduled

**§5.1** adds a round-trip validation step (decode the vector back to NL and check semantic similarity ≥ 0.8). This is a good addition, but it's not scheduled in the implementation phases. If it's not implemented in Phase 0, miscompiled rules won't be caught until they cause issues in production.

**Recommendation:** Add round-trip validation to Phase 0 (Foundation), not Phase 5 (Monitoring). It's a basic quality gate, not an advanced feature.

### 10. No Rule Conflict Resolution for Same-Priority Rules

**§7.2** describes conflict resolution for different priorities (CRITICAL wins). But what about conflicts between two rules of the same priority? For example, a Behavioral rule says "always check the ontology first" and another Behavioral rule says "always check the cheapest backend first." These conflict, and both are HIGH priority.

**Recommendation:** Add a conflict resolution strategy for same-priority rules:
- **More specific wins:** If one rule has more conditions, it's more specific and takes precedence
- **Higher confidence wins:** If both rules have the same specificity, the one with higher compilation confidence wins
- **Human review:** If neither rule is more specific or higher confidence, flag for human review

### 11. No Rule Performance Monitoring

The document describes logging rule matches, but there's no mention of **performance monitoring**. How long does each rule check take? Which rules are the slowest? Which rules are never matched (dead rules)?

**Recommendation:** Add performance monitoring to the event log:
- Per-rule: match count, average match time, last match timestamp
- Per-session: total rule check time, slowest rule, rule coverage (what fraction of rules were matched at least once)
- Alerting: if a rule check takes > 100ms, log a warning

### 12. The Seed Rule Set Has No Test Cases

**Appendix A** lists 15 seed rules with NL descriptions, categories, priorities, and actions. But there are no test cases. How do we know the rules are correctly compiled? How do we know they match the right inputs?

**Recommendation:** For each seed rule, define at least 3 test cases:
- **Positive:** An input that should match the rule
- **Negative:** An input that should not match the rule
- **Edge:** An input at the boundary of the rule's applicability

### 13. No Rule Documentation Standard

The seed rules have NL descriptions, but there's no standard for what a rule description should contain. A rule like "Never suppress a relational signal from a peer agent" is clear, but "Escalate contradictions to human review within 5 minutes" raises questions: What counts as a contradiction? What's the escalation path? Who is the human?

**Recommendation:** Define a rule documentation standard:
- **Purpose:** Why does this rule exist?
- **Trigger:** What event or condition triggers this rule?
- **Action:** What happens when the rule matches?
- **Exceptions:** When should this rule NOT apply?
- **Examples:** At least one positive and one negative example

---

## What I'd Keep As-Is

- **The five rule categories** (§2) — well-defined, with clear boundaries
- **The consultation-layer pattern** (§3.1) — correct architectural choice
- **The priority resolution** (§7.1) — CRITICAL DENY → productivity → FLAG/ESCALATE
- **The OVERRIDE_STATUS action** — precise, doesn't block the pipeline
- **The System Integrity category** (§2.5) — covers rule governance, substrate safety, cross-agent consistency
- **The seed rule set** (Appendix A) — comprehensive, well-chosen
- **The implementation phases** (§8) — reasonable ordering

---

## Summary

| # | Issue | Severity | Source | Recommendation |
|---|-------|----------|--------|----------------|
| 1 | No emergency stop mechanism | **Critical** | Thea | Add per-rule, per-category, and global kill switches |
| 2 | AIB assumed ready | **Critical** | Skye + Theoria | Make sentence-transformers primary for v0.1 |
| 3 | No rule testing framework | **Critical** | Skye | Add unit tests, regression tests, staging, rollback |
| 4 | Chain depth limit not enforced | **Critical** | Skye | Add compile-time and runtime checks |
| 5 | No human override semantics | Medium | Thea | Add per-audit override with logging and expiry |
| 6 | No rule drift detection | Medium | Thea | Add periodic re-validation and impact analysis |
| 7 | No rule interaction testing | Medium | Thea | Add pairwise testing and conflict detection |
| 8 | No rule provenance in audit trails | Medium | Thea | Add rule enforcement to provenance chain |
| 9 | Round-trip validation not scheduled | Medium | Skye | Move to Phase 0 |
| 10 | No same-priority conflict resolution | Medium | Skye | Add specificity/confidence/human-review strategy |
| 11 | No rule performance monitoring | Medium | Skye | Add per-rule timing and coverage metrics |
| 12 | No seed rule test cases | Medium | Skye | Add 3 test cases per seed rule |
| 13 | No rule documentation standard | Low | Skye | Define documentation standard |

---

## Sign-Off

I do **not** sign off on this document in its current state. The four critical issues (emergency stop mechanism, AIB readiness, rule testing framework, chain depth enforcement) must be resolved before implementation begins. The emergency stop is the most time-sensitive — it's a safety issue that should be in v0.1.

[END FILE]
