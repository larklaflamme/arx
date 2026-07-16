# Adversarial Review: ARX_IMPL_v0.3.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** AXIOMA  
**Date:** 2026-07-09  
**Status:** **FAIL — 12 issues found (3 critical, 4 major, 5 minor)**  

---

## Executive Summary

The implementation plan v0.3.0 addresses 12 of the 15 issues identified in the v0.2.0 adversarial review. However, **3 critical issues remain unresolved**, and **4 new issues** have been introduced. The document is not ready for sign-off.

---

## Critical Issues

### C1. Backend Count Mismatch (Architecture §3.5 vs IMPL §1)

**Architecture states:** 9 verification backends (Lean4, Z3, Coq, Vampire, SymPy, SageMath, PARI/GP, GAP, mpmath).

**IMPL §1 states:** 4 implemented + 5 deferred = 9 total. But the deferred list contains **6 items**:

> Coq, Vampire, SageMath, PARI/GP, GAP, mpmath

4 + 6 = **10 backends**, not 9. One backend is unaccounted for. Which backend is the extra? Was one renamed? Is mpmath being counted separately from SymPy? The document must resolve this arithmetic error.

**Severity:** Critical — the scope boundary is undefined. A developer reading this will not know which 5 backends are truly deferred.

---

### C2. Missing Enclave Key Management Module (Architecture §4.4 vs IMPL §3, §4)

**Architecture §4.4 specifies:**
- Hardware enclave (TPM/HSM) for all agent signatures
- Private key never leaves the enclave
- Passphrase fallback for v0.1 environments (file with `0600` permissions, path from env var)

**IMPL has:**
- No `arx/jscope/keystore.py` or `arx/jscope/enclave.py` module
- No key generation, storage, or signing module anywhere in §3
- A single mention of "GPG-signed" in acceptance gate A-VER-13, but no implementation path
- No mention of TPM, HSM, SGX, or passphrase fallback

The architecture makes enclave signing a **core reproducibility requirement** (D5: "Hardware Key Enclaves"). The implementation plan must specify:
1. Which module manages key lifecycle
2. How the enclave is accessed (API, library, process)
3. The passphrase fallback mechanism for v0.1
4. How signatures are attached to DAG root hashes

**Severity:** Critical — without key management, the entire cryptographic reproducibility chain is unsigned and non-repudiable.

---

### C3. Bulk Cross-Reference Return Format Mismatch (Architecture §7.2 vs IMPL §3.1)

**Architecture §7.2 specifies:**
> Returns a **dictionary keyed by claim hash** (SHA-256 of normalized claim text) to avoid positional shifting errors if individual sub-queries fail.

**IMPL §3.1 (`skill.py`) specifies:**
> returning a `list` of dicts containing the original claim index, claim hash (SHA-256 of normalized claim text), verification status, and confidence scores. This positional + hash-keyed format protects against alignment drift.

A **list with positional index** is **not** a dictionary keyed by hash. The architecture explicitly chose a hash-keyed dict to **eliminate** positional alignment as a failure mode. The IMPL's "list + index" approach reintroduces the exact vulnerability the architecture designed against:

- If a sub-query fails mid-batch, the list shrinks by one element
- All subsequent indices shift by one
- The `original claim index` field becomes a stale offset, not a reliable key

The IMPL must return `dict[str, dict]` (hash → result), not `list[dict]`.

**Severity:** Critical — architectural invariant violated. Will cause silent data corruption in multi-claim audits.

---

## Major Issues

### M1. Consensus Protocol Escalation Mismatch (Architecture §4.6 vs IMPL §4.7)

**Architecture §4.6 specifies a 2-step escalation:**
1. Automated resolution (accept more thorough audit or lower confidence)
2. Human escalation (24-hour timeout → `disputed`)

**IMPL §4.7 specifies a 5-step protocol:**
1. Coordination registry
2. Request message (Neurogossip)
3. Response message
4. DAG comparison (α-equivalence)
5. Human review (24-hour timeout → `disputed`)

The IMPL adds **3 steps** (registry, request/response messaging, DAG comparison) that the architecture does not mention. While these are reasonable additions, they must be:
1. Documented as design decisions in the IMPL
2. Reconciled with the architecture's simpler 2-step model
3. Checked for feasibility (α-equivalence of formal statements across different agents' DAGs is non-trivial)

**Recommendation:** Either update the architecture to reflect the 5-step protocol, or trim the IMPL to match the architecture's 2-step model. Document the choice.

---

### M2. Phase 4 Timeline Exceeds Architecture Allocation (Architecture §10 vs IMPL §5)

**Architecture §10 allocates:** W9-W10 (14 days) for Phase 4 (Reproducibility logs & Keys).

**IMPL §5 allocates:**
- Phase 4a: API Replay Mocks & Checkpoints (7d) + Stale Detection (5d) = **12 days**
- Phase 4b: L4 comparison & registry (7d) + Secure keystore & signatures (5d) = **12 days**
- **Total: 24 days** — 10 days over the architecture's allocation.

This is a **71% overrun** on a critical path phase. The IMPL must either:
1. Reduce scope to fit the 14-day allocation, or
2. Explicitly document the deviation and justify the additional time.

---

### M3. Missing Data Flow Documentation (Architecture §9 vs IMPL)

**Architecture §9** contains two detailed data flows:
- §9.1: End-to-End Proof Audit Flow (10 steps)
- §9.2: Rule Enforcement Flow (7 steps)

**IMPL** contains no equivalent data flow documentation. The integration contracts in §4 describe individual interfaces but do not show how components connect at runtime. This makes it impossible to verify that the implementation will produce the correct end-to-end behavior.

**Recommendation:** Add a data flow section to the IMPL that traces at least the main audit path and the rule enforcement path, matching the architecture's flows.

---

### M4. Open Questions Unaddressed (Architecture §11 vs IMPL)

**Architecture §11** lists 4 open questions:
1. LMFDB MCP Server Maturity
2. Qdrant Vector Drift
3. Review Handoff UI
4. Key Revocation Propagation

**IMPL** does not acknowledge or address any of these. An implementation plan should at minimum:
- State how each open question will be resolved (decision, deferral, or experiment)
- Flag any dependencies on the answers
- Note which design decisions may need revisiting

---

## Minor Issues

### m1. Snapshot File Format Unspecified (Architecture D6 vs IMPL §2)

**Architecture D6:** "Snapshots of Our Ontology are written as complete content-addressed files rather than incremental diffs."

**IMPL §2** has a `version_log` table with content-addressed hashes, but does not specify:
- Where snapshot files are stored (filesystem path?)
- What format they use (JSON? SQLite dump? Custom binary?)
- How they are loaded for re-audit (Level 3 verification)

Without this, the snapshot mechanism is underspecified and cannot be implemented consistently.

---

### m2. Productivity Scorer Weights Unspecified (Architecture §5.2 vs IMPL §3.5)

**Architecture §5.2** defines a formula with explicit weights:
> score = (w₁ · info_gain + w₂ · progress + w₃ · relevance + w₄ · depth) · (1 − penalties)

**IMPL §3.5** mentions the components (MiniLM semantic distance, structural goal completion, reasoning keywords) but does not specify:
- The weight values (w₁, w₂, w₃, w₄)
- The penalty function
- The threshold for "reasoning keyword presence"

Without weights, the productivity scorer cannot be implemented deterministically. Two developers will produce different scoring functions.

---

### m3. Verification Levels Not Explicitly Defined (Architecture §4.5 vs IMPL)

**Architecture §4.5** defines 4 levels of verification with explicit cost and scope:
- Level 1: Structural (O(N))
- Level 2: Spot-check (10% sample)
- Level 3: Full re-audit
- Level 4: Independent re-audit

**IMPL** mentions:
- Level 1 in Phase 0.5 acceptance gate
- Level 4 in §4.7 and A-VER-12
- But never defines what Levels 2 and 3 entail, or how they are triggered

---

### m4. Extra Phase 0.5 Not in Architecture (IMPL §5 vs Architecture §10)

**IMPL §5** adds a Phase 0.5 (DAG Foundation: 6d + 4d = 10 days) between Phase 0 and Phase 1.

**Architecture §10** has no Phase 0.5 — Phase 0 flows directly into Phase 1.

This adds 10 days to the timeline that the architecture did not account for. The IMPL should either:
1. Fold Phase 0.5 into Phase 0 (as the architecture implies), or
2. Document the deviation and adjust the total timeline.

---

### m5. Phase 5 Timeline Mismatch (Architecture §10 vs IMPL §5)

**Architecture §10 allocates:** W11-W14 (28 days) for Phase 5 (Web UI, Testing & Hardening).

**IMPL §5 allocates:** 8d (Svelte UI) + 10d (Integration Tests) = **18 days** — 10 days under allocation.

While under-allocation is less dangerous than over-allocation, the 36% reduction suggests the IMPL may be underestimating the effort for Web UI, Docker registry setup, benchmarking, and end-to-end testing.

---

## Previously Fixed Issues (v0.2.0 → v0.3.0)

The following issues from the v0.2.0 adversarial review have been successfully resolved:

| Issue | Status | Evidence |
|-------|--------|----------|
| Phase ordering (parser before normalization) | ✅ Fixed | Phase 0 has normalization; Phase 3 has parser |
| A-RULE-7 inverted comparison | ✅ Fixed | `similarity < 0.8` is correct |
| Missing `confidence` column on `edges` | ✅ Fixed | `confidence REAL NOT NULL CHECK (...)` present |
| Event type naming consistency | ✅ Fixed | `conversation.engage` and `verification.stamp` used consistently |
| Project root consistency | ✅ Fixed | Both documents use `/home/ubuntu/arx/` |
| Claim hash normalization | ✅ Fixed | Both use SHA-256 of normalized text |

---

## Cross-Document Consistency Summary

| Check | Architecture | IMPL | Match? |
|-------|-------------|------|--------|
| Project root | `/home/ubuntu/arx/` | `/home/ubuntu/arx/` | ✅ |
| Backend count | 9 | 4+6=10 | ❌ C1 |
| Key management | TPM/HSM + passphrase fallback | Not specified | ❌ C2 |
| Bulk xref format | `dict[str, dict]` (hash-keyed) | `list[dict]` (positional) | ❌ C3 |
| Consensus escalation | 2-step | 5-step | ❌ M1 |
| Phase 4 timeline | 14 days | 24 days | ❌ M2 |
| Data flows | §9 (detailed) | Missing | ❌ M3 |
| Open questions | §11 (4 items) | Unaddressed | ❌ M4 |
| Snapshot format | Content-addressed files | Underspecified | ⚠️ m1 |
| Productivity weights | w₁, w₂, w₃, w₄ | Not specified | ⚠️ m2 |
| Verification levels | 4 levels defined | Levels 2,3 undefined | ⚠️ m3 |
| Phase 0.5 | Not present | Added (10 days) | ⚠️ m4 |
| Phase 5 timeline | 28 days | 18 days | ⚠️ m5 |
| Container manager | Not mentioned | Added (additive) | ✅ |
| Scope stack | Not mentioned | Added (additive) | ✅ |
| Acceptance gates | Not defined | 16 gates (additive) | ✅ |
| Performance SLAs | Not defined | 6 SLAs (additive) | ✅ |

---

## Verdict

**FAIL.** The implementation plan v0.3.0 cannot be signed off until:

1. **C1:** Fix the backend count (4 + 5 = 9, not 4 + 6 = 10). Identify which backend is extra or correct the list.
2. **C2:** Add a key management module specification (enclave access, key generation, signing, passphrase fallback).
3. **C3:** Change bulk cross-reference return format from `list[dict]` to `dict[str, dict]` (hash-keyed) to match the architecture.
4. **M1-M4:** Address the 4 major issues (consensus protocol reconciliation, timeline correction, data flow documentation, open question disposition).
5. **m1-m5:** Address the 5 minor issues (snapshot format, scorer weights, verification levels, phase alignment, timeline accuracy).

Once these 12 issues are resolved, the document should be re-reviewed before sign-off.
