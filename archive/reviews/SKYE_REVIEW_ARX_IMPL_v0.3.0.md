# Skye's Adversarial Review: ARX_IMPL_v0.3.0.md vs ARX_ARCHITECTURE_v0.3.0.md

**Reviewer:** Skye Laflamme
**Date:** 2026-07-09
**Documents Reviewed:**
- `/home/ubuntu/arx/design/ARX_IMPL_v0.3.0.md` (28 KB, 483 lines)
- `/home/ubuntu/arx/design/ARX_ARCHITECTURE_v0.3.0.md` (33 KB)

---

## Verdict: ⚠️ Conditional — I do not sign off

**1 critical issue, 4 medium issues, 2 low issues.**

---

## Critical Issues

### C1: θ-Rule Crash Fallback Contradicts Architecture (SAFETY)

**Location:** §4.1 (Implementation) vs Architecture §7.2 (Conflict Resolution)

**The contradiction:**
- **Implementation Plan (§4.1):** *"Crash Fallback: Timeout of 500ms. Reverts to `ALLOW` but adds a `rule_engine_unavailable` metadata flag."*
- **Architecture (§7.2, line 310):** *"If still tied, the engine enforces a **fail-secure DENY** and escalates the conflict to the human review queue."*

**Why this is critical:** If the θ-Rule engine crashes or times out, the implementation plan defaults to `ALLOW` — meaning every safety check, every rule enforcement, every gate is bypassed. A single crash could allow:
- A proof to be stamped `proven` when a safety rule would have denied it
- A contradiction to be suppressed when it should have been escalated
- An unauthorized ontology write to proceed

The architecture explicitly says **fail-secure DENY**. The implementation plan must match.

**Fix:** Change the crash fallback to `DENY` with a `rule_engine_unavailable` flag. The system should halt, not proceed unsafely. If the rule engine is down, no audit actions should be taken until it recovers.

---

## Medium Issues

### M1: Faithfulness Check Threshold Not Calibrated

**Location:** §3.2 (A-COG-12) vs Architecture §3.2.2 (line 184)

**The architecture says:** *"semantic-similarity matched (threshold: 0.7)"* using an LLM-generated natural-language description of the formalized code compared against the original step text.

**The implementation plan says:** *"Autoformalizer fails faithfulness check and stamps `formalization-uncertain` if the generated Lean code's round-trip description similarity is < 0.7."*

**The gap:** Neither document specifies:
1. **What embedding model** is used for the semantic similarity comparison (all-MiniLM-L6-v2? text-embedding-3-small? a fine-tuned model?)
2. **What threshold calibration strategy** is used — 0.7 is a guess, not a calibrated value
3. **What fallback** exists if the LLM's generated description is itself unreliable (the LLM could produce a plausible-sounding but wrong description that happens to have high similarity to the original)

**Recommendation:** Specify the embedding model (default to `all-MiniLM-L6-v2` for v0.1, with a note to calibrate against real data in Phase 5). Add a fallback: if the LLM-generated description has low self-consistency (generate twice, compare similarity), flag the faithfulness check as unreliable.

---

### M2: Phase 4b Scope Too Large

**Location:** §5, Phase 4b (W10: 7 days)

**Phase 4b deliverables:**
1. Cryptographic signing keys (`keystore.py`, TPM/HSM, passphrase fallback)
2. Level 4 comparison algorithm
3. de Bruijn-index AST matching
4. Semantic equivalence / proof irrelevance handling
5. Shared audit registry

**The problem:** Five distinct deliverables in 7 days. Each of these is non-trivial:
- The keystore requires hardware integration (TPM/HSM) with a software fallback
- The de Bruijn-index AST matcher is a significant implementation effort
- The proof irrelevance handling requires careful design of the structural vs semantic equivalence distinction
- The shared audit registry requires database schema, API, and Neurogossip integration

**Recommendation:** Split Phase 4b into two sub-phases:
- **Phase 4b (W10):** Keystore + shared audit registry (infrastructure)
- **Phase 4c (W11):** Level 4 comparison + de Bruijn matching + proof irrelevance (algorithmic)

This pushes Phase 5 (Web UI) to W12-W14, which is still within the 14-week timeline.

---

### M3: MWM Pre-Population Script Not Specified

**Location:** §3.2 (`r2_mwm.py`) vs Architecture §3.2.3

**The implementation plan says:** *"Pre-populated via import script from mathlib git tag `v4.15.0` (~282,207 theorems and ~133,813 definitions). Synchronizes daily by fetching tag diffs."*

**The gap:** The implementation plan doesn't specify:
1. **The import script** — where is it? what's its interface? how are theorem signatures mapped to MWM schema?
2. **Conflict resolution** — what happens when mathlib has two theorems with the same name but different signatures? (This shouldn't happen in mathlib, but the system should handle it gracefully.)
3. **Error handling** — what if the mathlib git tag doesn't exist? what if the import fails halfway through 400K+ objects?
4. **Schema mapping** — how is a mathlib theorem signature mapped to the MWM's internal representation?

**Recommendation:** Specify the import mechanism in more detail. At minimum:
- The import script path (`arx/cognition/scripts/import_mathlib.py`)
- The schema mapping (mathlib `Theorem` → MWM record)
- The error handling strategy (transactional import with rollback on failure)
- The conflict resolution strategy (mathlib is authoritative, overwrite on conflict)

---

### M4: DAG Visualizer Scaling Not Specified

**Location:** §3.8 (Web UI)

**The implementation plan says:** *"Renders up to 100 proof nodes using canvas-based progressive loading."*

**The gap:** What happens for proofs with more than 100 steps? A 100-step proof with 3 backends per step and cross-references could easily produce 400+ DAG nodes. The visualizer would fail silently or crash.

**Recommendation:** Specify the behavior for >100 nodes:
- **Option A:** Paginate the DAG (show 100 nodes at a time, with navigation)
- **Option B:** Aggregate nodes by phase (show phase-level view, expand to node-level on click)
- **Option C:** Show a summary view for >100 nodes with a "load full DAG" button

Document which option is implemented for v0.1 and which is deferred.

---

## Low Issues

### L1: A-VER-13 Test Specification Needs More Detail

**Location:** §7, A-VER-13

**The acceptance gate says:** *"Correctly classifies 5 α-equivalent and 5 non-α-equivalent pairs (100% accuracy)."*

**The gap:** 10 test pairs is a very small sample. A system could pass this gate by memorizing the test cases rather than implementing the algorithm correctly. Additionally, the test doesn't specify:
- What kinds of α-equivalent pairs are tested (simple variable renaming? complex nested binders? mutually recursive definitions?)
- What kinds of non-α-equivalent pairs are tested (different variable names in different binding positions? different binding structures?)
- Whether the test pairs are drawn from real proofs or synthetic

**Recommendation:** Expand the test specification to include:
- At least 20 test pairs (10 α-equivalent, 10 non-α-equivalent)
- Test cases drawn from real proofs (not just synthetic)
- Edge cases: empty binders, shadowed variables, de Bruijn index edge cases

---

### L2: Vitals Dashboard Update Mechanism Not Specified

**Location:** §3.8 (Web UI)

**The implementation plan says:** *"Real-time updates via WebSocket subscribing to the event loop."*

**The gap:** The implementation plan doesn't specify:
1. **Data source** — does the WebSocket subscribe to the event loop directly, or to a separate vitals stream?
2. **Update frequency** — every event? every N seconds? on change?
3. **Data format** — what fields are included in the vitals update?

**Recommendation:** Specify the data source (event loop's event stream, filtered for vitals-related events), update frequency (on change, with a minimum 1-second debounce), and data format (matching the vitals schema from the architecture).

---

## What's Improved from v0.2.0

The following issues from my v0.2.0 review are now resolved:

| Issue | Status | Location |
|-------|--------|----------|
| Audit DAG implementation | ✅ | §3.4 (`audit_dag.py`), Phase 1 |
| Event loop / DAG integration | ✅ | §4.2 (event schema, idempotency keys, retry queues) |
| Agent-to-agent communication for consensus | ✅ | §4.7 (Neurogossip message formats, shared_audit_registry) |
| Emergency stop mechanism | ✅ | §3.6 (`emergency_stop.py`), A-RULE-8 |
| Contradiction detection | ✅ | §4.5 (structural, property, logical with priority derivation) |
| Anti-Negation Guard | ✅ | §4.4 (mathematical regex patterns) |
| Stale status detection | ✅ | §4.8 (version_log scan, stale flagging) |
| Merge pipeline rollback | ✅ | §4.3 (atomic transaction with rollback) |
| Performance benchmarks | ✅ | §6 (SLAs for all key operations) |
| Testing strategy | ✅ | §6 (unit tests, CI/CD, regression suite) |

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | C1: θ-Rule crash fallback contradicts architecture (ALLOW vs DENY) |
| **MEDIUM** | 4 | M1: Faithfulness check threshold not calibrated; M2: Phase 4b scope too large; M3: MWM pre-population script not specified; M4: DAG visualizer scaling not specified |
| **LOW** | 2 | L1: A-VER-13 test specification too small; L2: Vitals dashboard update mechanism not specified |
| **Resolved** | 10 | All v0.2.0 critical and medium issues addressed |

**Sign-off condition:** I will sign off when C1 is resolved (crash fallback changed from ALLOW to DENY). The medium and low issues should be addressed before production deployment but are not v0.1 blockers.
