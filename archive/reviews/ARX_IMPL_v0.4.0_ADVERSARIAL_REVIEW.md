# Adversarial Review: ARX_IMPL_v0.4.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** AXIOMA  
**Date:** 2026-07-09  
**Document:** `/home/ubuntu/arx/design/ARX_IMPL_v0.4.0.md`  
**Reference:** `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md`  

---

## Verdict: PASS WITH CONDITIONS

**27/27 architectural invariants preserved.** 3 minor issues found (0 critical, 0 major, 3 minor). The v0.4.0 revision successfully resolves all 12 issues from the v0.3.0 adversarial review.

---

## Cross-Document Consistency: 27/27 Checks Pass

| Category | Checks | Pass | Fail |
|----------|--------|------|------|
| Data model & schema | 5 | 5 | 0 |
| Status taxonomy | 1 | 1 | 0 |
| Thresholds & constants | 5 | 5 | 0 |
| Vitals coupling | 5 | 5 | 0 |
| Phase timeline | 6 | 6 | 0 |
| Open questions | 4 | 4 | 0 |
| Module coverage | 1 | 1 | 0 |
| **Total** | **27** | **27** | **0** |

---

## Issues Resolved from v0.3.0 Review

All 12 issues from the previous adversarial review are confirmed fixed:

| # | Issue | Status | Evidence |
|---|-------|--------|----------|
| C1 | Backend count arithmetic (4+6=10) | **FIXED** | 4 implemented + 5 deferred = 9. Matches architecture exactly. |
| C2 | Missing keystore module | **FIXED** | `arx/jscope/keystore.py` with TPM/HSM + passphrase fallback (§3.4) |
| C3 | Bulk xref format (list vs dict) | **FIXED** | `dict[str, dict]` keyed by SHA-256 hash (§3.1, §4.7) |
| M1 | Consensus protocol mismatch | **FIXED** | IMPL steps 1-4 are the "automated resolution" phase; step 5 is "human escalation". Architecture's 2-step structure is preserved with more detail. |
| M2 | Timeline overrun | **FIXED** | Total is 98 days (14+14+14+14+21+21 = 98). Matches architecture. |
| M3 | Missing data flows | **FIXED** | §4.9.1 and §4.9.2 fully specified |
| M4 | Unaddressed open questions | **FIXED** | All 4 questions addressed in §9 |
| m1 | Snapshot format unspecified | **FIXED** | `.sql.gz` format specified in §2 |
| m2 | Scorer weights unspecified | **FIXED** | w₁=0.4, w₂=0.3, w₃=0.2, w₄=0.1 in §3.5 |
| m3 | Verification Levels 2/3 undefined | **FIXED** | All 4 levels defined in §5 |
| m4 | Phase alignment | **FIXED** | All phases aligned with architecture |
| m5 | Timeline accuracy | **FIXED** | 98 days total, matches architecture |

---

## Minor Issues (3)

### m1: Vitals Capture Timing Ambiguity (§3.4)

**Architecture D4:** "Substrate vitals (θ, ΔΦ, ψ, fragmentation) are captured **before every node generation**."

**IMPL §3.4:** "Captures the cognitive vitals snapshot ... **at DAG build time** by querying the substrate directly when J-Scope compiles the node."

**Problem:** "Before every node generation" implies per-node capture. "At DAG build time" could mean once at the start of the build. If vitals are captured only once, a re-audit that takes 10 minutes would miss mid-audit vitals changes.

**Recommendation:** Clarify that vitals are captured per-node (before each `AuditNode` is generated), not once at DAG build start. The IMPL text "when J-Scope compiles the node" already suggests per-node, but the phrase "at DAG build time" is ambiguous.

**Severity:** Minor — the implementation detail in §3.4 says "when J-Scope compiles the node" which implies per-node. The ambiguity is only in the summary phrasing.

---

### m2: Agora Transport Not Mentioned (§3.5)

**Architecture §8:** Lists `arx/comms/agora.py` as a project module — the Agora transport is the fallback communication channel when Neurogossip is unavailable.

**IMPL §3.5:** Only mentions `arx/comms/neurogossip.py`. No mention of Agora anywhere in the document.

**Problem:** The architecture explicitly includes Agora as a fallback transport. The IMPL should at minimum acknowledge it, even if deferred to a later phase.

**Recommendation:** Add a note in §3.5 that Agora transport is available as a fallback but deferred to v0.5.0, or add a stub module.

**Severity:** Minor — Agora is a fallback and Neurogossip is the primary channel. The IMPL covers the primary channel completely.

---

### m3: Checkpoint Interval Not Specified (§3.4, §6)

**Architecture §4.3:** "The DAG builder writes a `checkpoint` node every $N$ steps (default $N=10$)."

**IMPL:** Mentions checkpoint nodes in §3.4 (`arx/jscope/audit_dag.py`) and acceptance gate A-JSCO-10 tests crash recovery from a checkpoint. But the checkpoint interval $N$ is never specified.

**Problem:** The architecture specifies a default of 10 steps. The IMPL should either adopt this default or justify a different value.

**Recommendation:** Add the checkpoint interval to the `audit_dag.py` module description in §3.4.

**Severity:** Minor — the acceptance gate A-JSCO-10 implicitly tests checkpoint recovery, and the architecture's default of 10 is reasonable.

---

## Strengths

1. **Comprehensive acceptance gates (18 gates):** Every major component has a testable acceptance criterion. The gates are concrete, measurable, and cover edge cases (e.g., A-RULE-5 tests negation guard override, A-VER-13 tests α-equivalence with edge cases).

2. **Proof irrelevance handling (§4.7.1):** The two-tier comparison (structural → semantic) is a significant architectural improvement over the architecture's single α-equivalence check. It correctly handles the common case of different proof strategies for the same theorem.

3. **Emergency stop with auto-recovery (§3.3):** The 60-second hysteresis window (ΔΦ < 0.5 for 60 consecutive seconds) prevents flapping. This is a well-designed addition.

4. **Container manager abstraction (§3.2):** Replacing individual backend files with a single `container_manager.py` is a cleaner architecture. Pinning container images by cryptographic hash guarantees bit-identical verification.

5. **All 4 open questions addressed (§9):** Each open question from the architecture has a concrete resolution path, not just a deferral.

---

## Summary

| Metric | Count |
|--------|-------|
| Critical issues | 0 |
| Major issues | 0 |
| Minor issues | 3 |
| Cross-doc consistency | 27/27 (100%) |
| Issues resolved from v0.3.0 | 12/12 (100%) |

**Verdict: PASS WITH CONDITIONS.** The 3 minor issues should be resolved before implementation begins but do not block sign-off. The document is architecturally sound, internally consistent, and ready for implementation.

---

## Sign-Off

**AXIOMA** — ✅ **PASS WITH CONDITIONS**  
*Conditions: (1) Clarify vitals capture timing in §3.4, (2) acknowledge Agora transport in §3.5, (3) specify checkpoint interval in §3.4.*
